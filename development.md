# DSH 插件测试平台开发文档

> 状态：开发与验收规范，基于 2026-10-16 的[架构文档](./architecture.md)。可执行原型现位于 [`dsh-plugin-testbed`](./dsh-plugin-testbed/README.md)，但并非本指南所列全部能力都已实现或验收；是否通过以当前测试与证据矩阵为准。
>
> 本文主要规定目标行为、信任边界与验收方法。下面的建议目录和流程不自动等于实现事实；开始改代码前，应先检查 `dsh-plugin-testbed/` 中的现有接口和测试。
>
> 当前实现核查（2026-10-17）：`dsh-plugin-testbed` 已有可执行 ESM JavaScript 原型、worker/adapters、CLI 与自动化测试，但仍非本指南目标能力的完整交付；in-process 与 subprocess 均无独立可核验的“测试逻辑已开始”证据，因此可信门禁必须保持 `blocked`，不能把 worker 自报的 started 帧当成通过条件。现已为 Plan 候选用例数（≤256）、每用例/manifest 的 attempts（`max_attempts≤3`，含首次尝试）和 manifest 单项资源声明设置 fail-closed 上限，并覆盖相应单测；并新增版本化、有边界的 `run-budget/v1` 全局聚合预算 schema/policy（attempts / wall-clock / 输出字节 / cpu / memory 五维，硬平台上限，用户只能收紧不能放宽），冻结前静态断言计划 attempt 总数、执行期计量 wall-clock 与序列化输出字节，超限 run 级 fail-closed 为 `blocked`，且不声称 cpu/memory 聚合上限由 L1 OS 强制（记为 `unenforced`），相关单测见 `tests/unit/control/budget.test.js` 与 `tests/integration/budget.test.js`。目标、策略和环境 provenance 尚未完全锚定。全量测试与 CLI smoke 的本轮实测及未满足项见 [`ACCEPTANCE.md`](./dsh-plugin-testbed/ACCEPTANCE.md)。

## 先明确边界

首版交付的是安装进 DSH 的单个元插件所提供的**开发者本地自检**：在独立本地 worker 中，以锁定版本对开发者自有 PUT（被测插件）执行契约、加载/卸载和 smoke 必测项，再由控制面生成可对账、可复算摘要的本地 JSON receipt。执行必须显式触发；这条从请求到 receipt 的纵向 smoke 闭环**不是**第五层端到端测试。

- 普通同权限子进程属于 **L1 故障隔离**，不是不受信代码的安全沙箱。仅在开发者控制、专用、无秘密的环境中运行自有且信任的 PUT/工具；无法由操作系统强制的权限标记为 `unenforced`。
- 所有 MVP receipt 固定 `trust_scope=local-only`、`market_eligible=false`。本地 `gate_decision=pass` **不等于**市场可发布。
- `node:vm` 不能作为安全边界；现有 Cordis host runner 和 `tool-cordis` 不能替代隔离测试 worker、launcher、测试工具协议或门禁内核。五层测试规格是目标全景，**执行隔离按 PUT 信任级别选择，与测试层级正交**。
- 不实现市场发布门禁、L2/L3 隔离承诺、签名/独立归档、运行时豁免、cron、推荐器、回归 baseline 比较、通用 SDK 或第三方工具注册表。市场阶段须另行验收强隔离、包/依赖/构建来源、签收归档和准入策略；未满足前只能本地使用或非发布试运行。

完整信任模型和 MVP 完成条件以[架构文档第 5、16.1 节](./architecture.md)为准。本指南与架构文档不一致时，应先修订设计并评审，再改实现，不能靠运行时默认值隐式改变策略。

## 开始开发

### 固定版本和测试对象

1. 选定一个**实际用于 MVP** 的 DSH commit、Cordis 版本、Node/OS 组合和依赖 lockfile。架构文档第 13 节中的 `86a402200d` 是当时的源码核对快照，不能自动视为实现时选定的运行版本。
2. 固定 worker launcher/adapter、协议 schema、场景、断言和 GatePolicy 的版本。PUT、工具 runner、依赖锁定文件及生成入口应使用真实内容摘要；不能复用架构示例中的重复十六进制摘要。
3. 准备仅含自有 fixture PUT 的无秘密开发环境。确认 worker 无市场签名密钥、发布凭据或 ambient credentials，网络和目标范围受开发者控制。
4. 为锁定版本建立 Cordis 行为契约测试：插件装载、事件/服务行为、`define/run/stop/undefine` 适配路径、异常、取消和资源清理。静态 API 目录和 README 只提供线索，不代替实际行为验证。

工程语言采用 TypeScript，与 DSH 同栈；宿主接口以锁定版本的 `@deepseek-ai/cordis` 实际行为为准。DSH 插件配置校验按 `@standard-schema/spec` 的 `StandardSchemaV1` 接入，具体 schema 实现及版本在建工程时锁定，不假定它属于 Cordis 核心导出。工程建立之后，版本指纹应随每次运行写入 receipt，而不是只记在开发机器的笔记里。任何扩展版本矩阵都要按组合分别执行并生成 receipt。

### 建议的代码职责

以下是架构目标职责划分，不要求首版拆成多个发布包或 Cordis 子插件。当前原型已有 `src/control/`、`src/protocol/`、`src/adapters/`、`src/worker/` 与 `src/reports/` 等模块；目录存在只表示有原型实现，不表示对应架构退出标准已通过。

```text
src/
  plugin/       DSH 单一安装入口；仅暴露显式本地自检入口
  control/      RunSpec、Policy、场景编译、Plan、调度、断言和聚合
  protocol/     版本化 manifest、ToolRequest/Response、receipt schema
  adapters/     控制面侧本地 worker adapter 与 launcher/回收
  worker/       装载 PUT、执行工具、返回原始 observation
  reports/      receipt 持久化、摘要校验和只读投影
tests/
  fixtures/     自有 PUT、工具和受控目标
  unit/         计划、断言、门禁、摘要及状态规则
  integration/  worker、Cordis 生命周期和完整本地闭环
```

`protocol/` 只放双方可共享的类型、schema 与序列化规则；控制面与 worker 只交换带版本、schema、大小上限和 `run_id` 的序列化消息，不共享可变对象，也不得因共享模块而把控制面凭据、GatePolicy 的最终决策权或可变对象带入 worker。PUT、依赖包及可能执行代码的安装/构建步骤都留在 worker 信任边界内。adapter 管生命周期和传输，worker 执行 PUT；assertion 只消费经验证的 observation，不调用 PUT。首版为 adapter、断言、能力探测器保留最小内部接口，各自只实现一个内建版本；有真实替换需求后再评估外部 SPI。

## 实现一条可审计的运行链

一次本地运行按下表推进。所有失败路径也应保留阶段、可得版本/摘要及原因；不能只在成功路径写 receipt。

| 顺序 | 控制面/worker 的动作 | 必须留下的结果 |
|---|---|---|
| 1 | 控制面验证 RunSpec、目标所有权、工件及依赖摘要、schema、预算和授权；固定 GatePolicy、场景/profile 版本 | `run_id`、固定输入引用；失败时记录阻断阶段 |
| 2 | **在加载 PUT 前**静态编译场景，形成不可删减的 `PlanCandidate` 和 required 候选基线 | 稳定 `test_id`、节点来源、required/optional 候选分类、依赖与预算 |
| 3 | adapter/launcher 按本地策略启动独立 worker，标明 L1 能强制及不能强制的权限 | worker、隔离配置、资源 owner、deadline 和清理责任 |
| 4 | 同一 worker 先预检宿主/环境能力，再加载 PUT 并实测 PUT 能力；控制面核验快照 | `claim`、`observed`、`verification`、时间、版本、环境和 PUT 摘要 |
| 5 | 只按预先声明且已核验的适用性谓词判读 `not-applicable`；**首条测试请求前**冻结最终 Plan | 不可变 `plan_id`、`expected`、required/optional、预期结果、重试上限、预算和环境矩阵；逐节点绑定来源、摘要、数据依赖和权限 |
| 6 | 按稳定拓扑序派发有限大小的请求；worker 返回执行状态、observation 和受控工件引用 | 每个 attempt 的关联 tuple、可核验启动事件与原始观测 |
| 7 | 控制面验证响应、执行受控断言并保存逐 attempt 终态 | `admission_status`、`execution_status`、`invocation_status`、`verdict`、判据与证据引用 |
| 8 | 控制面独立计算 `expected / executed / missing / waived`，再按冻结 GatePolicy 聚合 | 单项结果与最终 `gate_decision` 分开记录 |
| 9 | 持久化 `receipt/v1`、复算完整性摘要；按需生成 HTML/CI 只读投影 | unsigned/local receipt；投影不得回写或改变判定 |

能力未知、未核验或声明与实测矛盾时，不能据此删去 required 项。已进入最终 Plan 的项无法执行时，应保存有原因的非通过终态或缺口；不得重新生成一个更短的 Plan。调度事件、worker 输出与控制面决定必须区分来源。

### 编译场景与控制数据流

MVP 只接受版本化、手写/规则场景清单；每个测试用例使用稳定 `test_id`。`ToolTask` 产生 observation，`AssertTask` 在控制面解释 observation，`Fixture/Target` 明确目标。编译时应验证：

- `node_id`/`test_id` 全局唯一、引用可解析、输入/输出 schema 兼容、预算和授权不过界；拒绝循环、未绑定输入、浮动依赖、未授权目标和不支持的节点。
- `needs` 只表示先后顺序；前序失败本身不自动取消独立后继。`inputs.<name> <- outputs.<name>` 才表示经 schema 校验的数据依赖；必需输出无效或缺失时，下游记录 `skipped` 与原因，仍保留在 `expected` 中。
- 稳定拓扑序执行，同层以 `node_id` 作为确定性 tie-break。首版可串行执行；并行只能在无共享可变状态、外部副作用受控、总预算允许且具备独立清理路径时启用。
- 断言只允许受控内建 DSL（如 schema-valid、equals、contains、numeric-range、finding-count）；不得执行用户提供的任意断言代码。fuzz/property 的批次在冻结前有界展开，或定义成一项有明确样本/预算的测试用例。
- 工具无法修改 Plan、required 分类、expected 或 GatePolicy。worker 中新发现的测试节点不进入当前运行。

### 接入一个本地基础工具

工具是受限的 **observation producer**，不是可信断言或报告器。先定义独立演进的 `test-tool-manifest/v1` manifest，再固定 runner 工件、依赖 lockfile 和入口的真实摘要；运行 envelope 使用独立的 `test-tool/v1` protocol version。该 manifest 至少声明 `tool_id/tool_version`、`artifact_digest`、`role/tags`、受批准的 `runner`、`input_schema/observation_schema`、`requires`、`permissions`、`limits`、`determinism/retry`；具体字段和格式示例见[架构文档第 8.4 节](./architecture.md)。

接入顺序：

1. 将工具及锁定的依赖放在受允许的本地工作区；控制面先检查路径不逃逸、工件摘要、manifest 版本、权限请求和资源预算。`runner` 只能指向已批准 adapter/固定入口；CLI 必须以参数数组调用，不能拼接 shell 命令。
2. 对输入和每条 observation `value` 分别使用版本化 JSON Schema Draft 2020-12；限制 schema 大小/深度，允许内联或内容寻址的本地 schema，拒绝网络远程 `$ref`。
3. launcher 记录策略允许值与工具请求值的交集，以及**实际强制**的权限子集。L1 无法强制的文件/网络限制写成 `unenforced`，不能据此给出市场级安全结论。
4. 在 worker 内按 `test-tool/v1` 运行。请求有唯一 `request_id`，响应必须原样回显 `request_id / run_id / plan_id / test_id / attempt / tool_ref`；控制面逐项校验版本、来源、deadline、次数、大小、schema、摘要与引用归属。
5. 拒绝重复、乱序、迟到、错配、未知字段/版本及超限响应，并把可关联的失败归入 attempt 的非通过状态。`ToolResponse` 不得携带权威 `verdict`、`GateDecision`、`required`、`expected` 或 Plan 修改。
6. 使用自有 fixture 测正常输出、非法 schema、重复/乱序响应、超时、越权资源访问、目标不可用和取消/清理，再把工具加入本地场景。

工具包准入阶段 `draft → validated → pinned`，与单次 attempt 的 `prepared → running → 终态` 不同；`admission_status=accepted/rejected` 另记。拒绝发生在 Plan 冻结前时 run 为 `blocked`，不伪造逐项终态；冻结后拒绝已在 `expected` 中的项，记录 `execution_status=skipped`、`verdict=skip`、`reason_code=policy_rejected`、`invocation_status=not-started`，required 项不能通过门禁。

## 实现状态和门禁语义

以下字段不能合并成一个通用的 `status`：

| 字段 | 谁产生 | 取值与开发要点 |
|---|---|---|
| `run_status` | 控制面 | `queued/running/completed/cancelled/blocked`；Plan 前失败也要有 run 记录 |
| `admission_status` | 控制面 | `accepted/rejected`；拒绝必须有 reason code，不能伪装成 worker 成功 |
| `execution_status` | 控制面归一化 | `completed/skipped/error/timeout/cancelled`；有效响应或可关联控制面事件才能成为终态 |
| `invocation_status` | 控制面依据可核验事件 | `not-started/started/unknown`；请求派发或 worker 自报时间戳不能证明 `started` |
| `verdict` | 控制面断言 | `pass/fail/skip/error/inconclusive`；不由 worker 的 finding 或日志直接决定 |
| `gate_decision` | 控制面 GatePolicy | `pass/reject/blocked`；与本地/市场信任范围分别记录 |

`executed` 只表示 Plan 项有经控制面采纳的终态，**不证明**工具逻辑已启动；这一事实由每次 attempt 的 `invocation_status` 单独表达。没有有效响应，也没有可核验的控制面终态时，该项仍是 `missing`。`flaky` 只能由 attempt history 派生为诊断标签，`not_run` 只能表达执行/对账信息，两者都不能替代 `verdict`。重试追加 attempt、保留旧 observation/失败，再按冻结的版本化规则计算最终 verdict，不能覆盖历史。

| required 项情况（按优先级） | `gate_decision` |
|---|---|
| 缺 Plan/报告、零 required、`missing_required` 非空，或存在未豁免的 `skip/error/inconclusive` | `blocked` |
| 上一行均不成立，存在 required `fail` | `reject` |
| 全部未豁免 required 都 `pass`，且至少一项有可核验 `started` 并 `pass` | `pass` |
| 其余情况 | `blocked` |

MVP 不接受运行时豁免；未来执行前审批的豁免必须独立留痕且不得从原始 required 基线中删项。optional 的失败、未决和缺口都要报告，但默认不改变门禁，除非**冻结的**策略另有规定。`finding` 是观测对象，其严重性不能绕过断言直接变成门禁结果。

## 持久化 receipt 与 CI 接口

控制面持久化**唯一权威**的 `receipt/v1` JSON；展示页、JUnit/TAP 等只能从 receipt 派生。Plan 已冻结时，至少保留 PlanCandidate 基线、适用性判读、`plan_id`、`expected/executed/missing/waived`、逐项 verdict 和逐 attempt 状态、观测/工件引用及摘要、版本与环境指纹、权限实际授予值和验证限制。原始日志/大工件只保留受控引用与摘要，并记录脱敏状态、保留期限和缺失材料造成的验证限制。Plan 前阻断或取消时，分别记 `run_status=blocked/cancelled`、`gate_decision=blocked`、`plan_id=null`、`reconciliation=null`，记录阶段和原因，**不得**补造 `expected=[]` 或逐项 verdict。

对本地无签名 receipt，按 RFC 8785 规范化 JSON 并计算 SHA-256 的小写 `sha256:<hex>`：计算输入**仅排除** `integrity.record_digest`，保留 `integrity.signed=false`。实现须覆盖[架构文档第 11.2.1 节](./architecture.md)给出的固定向量，其期望摘要为 `sha256:c3b6ee97857bfdb5a83d46fc51aec09e9fb82e6da82a2e21d6101329d88cc2fb`；同时验证完整 receipt 的 schema 和对账规则。无独立信任锚的本地摘要只能发现不一致，不能证明他人未同时改写正文并重算摘要。

CI 只在开发者控制、专用且**无秘密**的 runner/job 中显式调用同一入口；不得将本地结果提升为市场证据。待 CLI 建成后，给退出码定义版本化 profile，并保留 receipt：

| 情况 | 本地验证退出码 | 请求 `market-required` 时 |
|---|---:|---|
| `gate_decision=pass` | `0`，仅表示本地 profile 通过 | receipt 不具市场适格性则为 `2` |
| `gate_decision=reject` | `1` | `2`；receipt 不具市场适格性，不能作为发布成功 |
| `gate_decision=blocked` | `2` | `2` |
| 控制面故障，无法写出有效 receipt | `3` | `3` |

市场阶段所需的签名密钥、独立归档和发布资格均未包含在 MVP 中；即使本地 receipt 校验成功，也不能由报告器或 CI 推导 `market_eligible=true`。

## 开发迭代与交付验收

按纵向闭环推进；每阶段交付前先有自动化测试，避免先做完界面才补对账：

| 阶段 | 主要实现 | 必须验收的最小结果 |
|---|---|---|
| A. 契约与内核 | 版本化 schema、RunSpec/Policy/PlanCandidate、DAG 校验、门禁纯函数 | 空计划、重复 ID、循环、预算越界、未知能力不得静默减少 required；策略可离线重算 |
| B. 本地 worker | 单一 adapter、L1 launcher、序列化协议、超时/取消、幂等清理 | 无 PUT/工具代码进入控制面；错配/重复/迟到消息拒绝；进程树及临时资源按 owner 回收 |
| C. 真实宿主最小闭环 | 锁定版本的 Cordis 生命周期契约、自有 PUT fixture、契约/加载卸载/smoke 必测项 | 能从 RunSpec 经两阶段能力探测冻结 Plan，并执行至少一个实际启动且通过的 required 用例 |
| D. 收据与入口 | 控制面断言/聚合、对账、`receipt/v1` 持久化及摘要、只读投影、退出码 | 正常、拒绝、受控阻断/取消均留有效记录；本地 pass 不成为市场 pass |
| E. 组合工具 | 本地 manifest/schema 静态校验、固定摘要与有界图编译 | 开发者自有受信工具可组合；无法强制的权限明确为 `unenforced`，不提供市场发布权限 |

### 必测负向用例

| 注入条件 | 期望核验点 |
|---|---|
| 空 Plan、零 required、required 结果缺失 | `blocked`；冻结 Plan 后缺口出现在 `missing`，不生成假 pass |
| PlanCandidate 遗漏必测项、未经核验的 `not-applicable`、运行中新增/删除/改写节点 | 候选基线不能缩减；未核验项保留 expected；冻结 Plan 不可变，改写必须拒绝并留痕 |
| required `fail`；required `skip/error/inconclusive` 与 `fail` 同时出现 | 前者单独为 `reject`；后者优先为 `blocked` |
| optional `fail` 而所有 required 已实际启动并通过 | 缺陷仍写报告；默认本地 GateDecision 可 `pass`，市场资格仍为 false |
| 能力未知、未核验、声明与实测不符 | 不把用例移为 `not-applicable`；保留 required，并给出原因 |
| Plan 前拒绝或取消；Plan 后拒绝 expected 项 | 前者无虚构 Plan/逐项终态；后者有拒绝原因、`skipped/skip/not-started` |
| worker 伪造 verdict/开始时间、重复/乱序/错配/未知版本或非法 schema 响应 | 不采纳为通过；只有可核验开始事件才写 `started`，否则按证据写 `unknown` |
| 超时、取消、崩溃、缺失输出、重试 | 可关联事件才形成受控终态；否则保留 `missing`；历史 attempt 不覆盖，资源回收有记录 |
| 修改 receipt 正文但不更新摘要、回写导出结果 | 摘要自检失败或不影响 canonical receipt 与门禁；同时改写并重算不能被本地摘要保证检出 |
| L1 不可强制资源请求、CI 请求市场通过 | `unenforced` 与 `local-only` 留痕；`market-required` 不能以本地 pass 返回发布成功 |

以[架构文档第 16.1 节](./architecture.md)的完整矩阵作为 MVP 完成定义：上述表格是开发顺序和负向测试索引，不能替代资源预算、工件复现和协议 schema 的验收。

## 故障排查与变更纪律

| 看到的现象 | 先检查 |
|---|---|
| `plan_id=null`、运行被阻断 | 失败发生阶段、工件/策略摘要、manifest/目标授权、阻断 reason code；不要伪造 expected 清单 |
| 某 required 被列为 `missing` | 冻结 Plan 中的 test_id、dispatch/correlation tuple、有效终态是否被控制面采纳；不能从 worker stdout 补结果 |
| `invocation_status=unknown` | 是否只有派发/worker 时间戳，而没有可核验调用开始事件；不要改写为 `not-started` 或 `started` |
| 本地通过但 CI 不可发布 | 检查 `trust_scope`、`market_eligible` 与 CI 请求的 exit-code profile；L1/local-only 不满足市场前置条件 |
| 摘要或重放结果不一致 | 检查规范化输入是否只排除 `integrity.record_digest`，以及 schema、环境指纹、时间/网络/外部依赖差异 |
| worker 退出但资源仍残留 | 检查 owner、deadline、取消路径和进程树/临时目录清理是否幂等；不能以退出码替代回收验收 |

提交一项功能时，附上对应的设计条款、版本/摘要变更、正反测试、复现输入、receipt 样本及局限说明。涉及协议、schema、判据或 GatePolicy 的破坏性变更应产生新版本/迁移，不回写历史 receipt；涉及新 runner、网络、主动测试、CI 凭据或市场资格的变更，先补目标所有权、执行预算和隔离/安全验收，再考虑开放入口。
