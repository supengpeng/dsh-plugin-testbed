# DSH 插件测试平台开发计划

> 状态：目标执行计划，依据[架构文档](./architecture.md)与[开发文档](./development.md)整理；不代表下述能力已经实现。可执行原型已位于 [`dsh-plugin-testbed`](./dsh-plugin-testbed/README.md)，但实现尚未满足全部架构验收条件；以下里程碑继续作为目标计划，当前状态和可复核证据见原型的 [`ACCEPTANCE.md`](./dsh-plugin-testbed/ACCEPTANCE.md)。  
> 计划基线：2026-10-16。  
> 估时：两份输入文档未提供人力和工期数据，因此先以里程碑和验收门槛安排，不虚构日历日期或人日。
>
> 当前实现关注点（2026-10-17）：仓库原型使用 ESM JavaScript + JSDoc（TypeScript 同栈化仍未完成），并已有安装入口、CLI、worker 与测试；全量测试现为 233/233 通过，双模式 CLI smoke 均按预期产出 `blocked` receipt 与只读投影，但这不代表架构验收 `pass`。适配器仍无可信、独立的用例启动证据，匹配判据的 observation 不能代替启动证明。Plan 后的 caller-supplied `admission:'rejected'` 仅模拟本地不派发分支，并非控制面独立 GatePolicy 决策。现已设置候选用例数≤256、每用例/manifest 的 attempts（`max_attempts≤3`，含首次尝试），以及 manifest 单项资源声明上限；并新增版本化、有边界的 `run-budget/v1` 全局聚合预算 schema/policy（attempts / wall-clock / 输出字节 / cpu / memory 五维，硬平台上限，用户只能收紧不能放宽），冻结前静态断言计划 attempt 总数、执行期计量 wall-clock 与序列化输出字节，超限 run 级 fail-closed 为 `blocked`；cpu/memory 聚合上限仍仅记录为 `unenforced`，相关 OS 资源限制也不是 L1 的强制能力。目标/策略/环境 provenance、可复现锁定环境等仍未完整。不得以修改测试期望绕开正向门禁。完整证据和剩余差距见 [`ACCEPTANCE.md`](./dsh-plugin-testbed/ACCEPTANCE.md)。

## 目标与交付边界

交付一个可安装进 DSH 的单一元插件，为开发者自有、受信的 PUT（被测插件）提供显式触发的本地自检闭环：固定一组 DSH/Cordis 与运行环境版本，在独立本地 worker 中执行契约、加载/卸载和 smoke 必测项，由可信控制面冻结测试计划、计算结果对账和门禁决定，并保存可复算摘要的 `receipt/v1` JSON。

首版定位为 **L1、本地、开发者自控环境**：所有 MVP receipt 固定 `trust_scope=local-only`、`market_eligible=false`。本地 `pass` 不是市场发布通过，也不证明 worker 能安全执行恶意 PUT 或恶意工具。实现重点是先跑通纵向闭环和负向验证，不以完整测试层级覆盖或 UI 优先。

## MVP 范围

**纳入：**

- TypeScript 单插件与最小内部模块接口；首版每个适配、断言及能力探测接口仅有一个内建实现。
- 一个实际锁定的 DSH/Cordis、Node/OS、依赖 lockfile、worker launcher/adapter、场景、策略及协议版本组合。
- 版本化 RunSpec、Policy、手写/规则场景、`PlanCandidate`、最终冻结 Plan，以及稳定 `test_id` 和必测基线。
- 独立本地 worker；worker 与控制面之间只传有 schema、大小限制及 `run_id` 的序列化消息，不共享可变对象。
- 两阶段能力探测、自有 PUT fixture、契约/加载/卸载/smoke 必测项、受控断言、控制面门禁聚合和完整运行对账。
- `receipt/v1` 权威 JSON、RFC 8785 规范化摘要、只读报告/CI 投影，以及显式本地调用入口。
- 有界的本地基础工具组合、manifest/schema/摘要校验和 `test-tool/v1` 调用协议。

**明确不纳入：**

市场发布门禁或市场适格性、L2/L3 隔离承诺、签名密钥及独立归档、运行时豁免、cron/周期扫描、在线建议器、baseline 回归比较、通用 SDK、第三方工具注册表、多 worker/远程 runner，以及未定义授权和预算的主动攻击能力。上述项目需要独立设计与安全验收，不作为 MVP 的隐含延期任务。

## 实施里程碑

按依赖顺序推进；每个里程碑先完成自动化测试，再进入下一阶段。A–E 对应开发文档的迭代顺序；M0 是编码前置准备，M6 是整体验收，不改变该顺序。

| 里程碑 | 主要工作 | 前置条件 | 完成证据 / 退出标准 |
|---|---|---|---|
| **M0. 工程与版本锚定** | 初始化可执行 TypeScript 工程、构建/测试流水线和本地插件骨架；选择并固定实际使用的 DSH commit、Cordis、Node/OS、lockfile、StandardSchemaV1 具体实现、协议、场景、断言、GatePolicy 及 worker/launcher 版本；准备自有 fixture PUT 与无秘密开发环境；选定显式自检入口，并为锁定版本先建立 Cordis 行为契约测试基线。 | 计划评审通过，确认目标是本地 MVP。 | 工程能构建、测试；版本清单及锁文件可复核；fixture 与目标归属明确；至少有可运行的 Cordis 生命周期契约测试；控制面和 worker 环境不含签名密钥、发布凭据或 ambient credentials。源码核对快照不得直接当成运行版本。 |
| **M1. 契约与可信内核（A）** | 定义并验证版本化 schema、RunSpec/Policy、required 候选基线、`PlanCandidate`/Plan、DAG 编译与稳定拓扑序；实现状态枚举、受控断言 DSL、GatePolicy 纯函数和对账规则。 | M0 的版本与类型边界确定。 | 单元测试证明空计划、零 required、重复 ID、环、未绑定输入、schema 不兼容、预算/权限越界及未知能力会 fail-closed；能力未知或未核验不能缩减 required；同一固定输入可离线重算策略结果。 |
| **M2. 本地 worker 与传输（B）** | 实现单一本地 adapter/launcher、worker 生命周期、版本化序列化协议、消息关联与大小限制、deadline、取消、owner 管理、进程树和临时资源幂等清理；明确 L1 可强制与 `unenforced` 的边界。 | M1 的协议/schema 草案和运行预算。 | 控制面不加载 PUT/工具代码；worker 无控制面凭据或发布权限；错配、重复、乱序、迟到、未知版本/字段、非法 schema 和超限消息均被拒绝或记为非通过；超时/取消/崩溃后能核验清理结果。普通同权限子进程只声明为故障隔离。 |
| **M3. 真实宿主纵向闭环（C）** | 扩充 M0 的 Cordis 行为契约测试并接入 worker；验证插件加载、事件/服务行为、`define/run/stop/undefine`、异常、取消及资源释放；实现 worker 内宿主/环境预检及 PUT 能力实测，并接入自有 fixture 的契约、加载/卸载和 smoke 用例。 | M2 的 launcher/消息链可运行；M0 的宿主版本锁定与行为测试基线。 | 从 RunSpec 生成候选必测基线；同一 worker 完成两阶段能力探测；控制面核验快照后、首条测试请求前冻结最终 Plan/expected；至少一个 required 用例有可核验 `started` 并通过，收据闭环在 M4 验收。冻结后增删改计划均被拒绝并留痕。 |
| **M4. 收据、聚合与入口（D）** | 在控制面核验 observation、应用断言、独立计算 `expected / executed / missing / waived` 并聚合 GateDecision；持久化正常结束、reject、受控阻断和取消的 `receipt/v1`；实现完整性复算、只读投影及版本化本地退出码。 | M1–M3 的计划、worker 和真实宿主观测闭环。 | Plan 前失败/取消有记录但不伪造 Plan、expected 或逐项 verdict；Plan 后保留所有 required 缺口和 attempt 历史；本地 receipt schema 与对账验证通过；固定 RFC 8785 测试向量通过；投影不能回写判定；本地 pass 不会成为 market pass。 |
| **M5. 受控工具组合（E）** | 实现本地 `test-tool-manifest/v1` 校验、固定真实工件/依赖摘要、限制大小/深度的 JSON Schema Draft 2020-12（拒绝网络远程 `$ref`）、路径逃逸与授权校验、有界场景图编译及 `test-tool/v1` 请求/响应核验；只允许已批准 runner 和参数数组启动，不拼接 shell 命令。 | M1 的静态编译与协议基础、M2 的 worker 边界、M4 的 observation/receipt 链。 | 开发者自有受信工具可经本地 worker 运行并进入完整 receipt；响应 correlation tuple、schema、摘要、deadline、次数、引用和大小均核验；工具不能提供 verdict、修改 GatePolicy 或改变 expected；无法由 L1 强制的权限写为 `unenforced`，不赋予市场权限。 |
| **M6. MVP 验收与发布准备** | 运行架构第 16.1 节全量验收矩阵和开发文档负向回归；演练本地显式调用、失败定位、receipt 复算、重跑和资源清理；整理锁定版本、复现输入、fixture、测试报告和已知局限。 | M0–M5 完成。 | 所有下节的 MVP 完成门槛通过，未通过项明确标记未实现/未验收；只发布本地自检能力，不宣传市场适格性或强隔离。 |

### 待创建的模块与交付物

下列是目标职责，不是现有目录，也不要求拆成多个发布包：

| 模块/测试目录 | 交付职责 | 对应里程碑 |
|---|---|---|
| `src/plugin/` | DSH 单插件安装入口和显式自检调用 | M0、M4 |
| `src/control/` | RunSpec/Policy 校验、PlanCandidate/Plan、调度、断言、对账和 GateDecision | M1、M3–M4 |
| `src/protocol/` | 双方共享的版本化类型、manifest、消息及 receipt schema；不得包含凭据或门禁执行权 | M1–M2、M4–M5 |
| `src/adapters/` | 控制面侧本地 worker adapter、launcher、传输和按 owner 回收 | M2 |
| `src/worker/` | 加载 PUT、执行自有工具及返回原始 observation；依赖包和可执行的安装/构建步骤留在该边界 | M2–M3、M5 |
| `src/reports/` | `receipt/v1` 持久化、摘要复算、人读报告和 CI 只读投影 | M4 |
| `tests/fixtures/`、`tests/unit/`、`tests/integration/` | 自有 PUT/工具与受控目标、纯函数/摘要测试、Cordis/worker/本地闭环测试 | M0–M6 |

## 跨阶段不变量与验收门槛

以下条件不是可后补的优化项，应贯穿实现和验收：

1. **先形成必测基线，再执行 PUT。** 加载 PUT 前静态编译 `PlanCandidate` 和 required 候选基线；在同一 worker 中完成环境预检与 PUT 能力实测；控制面核验后，只按事先定义且已核验的适用性谓词处理 `not-applicable`；第一条测试请求前冻结最终 Plan。未知、矛盾或未核验能力不能删减必测项。
2. **计划与结论由控制面掌权。** PUT、工具、依赖包及可能执行代码的安装/构建步骤都留在 worker 信任边界内；控制面不加载 PUT/工具代码。worker 只回传有界执行状态、observation 和工件引用；不能上报权威 verdict、GateDecision、required 或 expected，也不能修改计划。断言只解释已验证 observation，不执行 PUT，也不运行用户任意断言代码。
3. **逐层区分状态与证据。** 独立记录 run、admission、execution、invocation、verdict 和 gate decision；派发请求或 worker 自报时间戳不证明测试逻辑已开始。重试追加 attempt，不覆盖历史；无法确认启动时记 `unknown`。
4. **缺项不静默变通过。** `missing_required`、required 的 skip/error/inconclusive 或零 required 按冻结门禁规则优先阻断；前述阻断条件均不成立而有 required fail 时才 reject；只有 required 均通过且至少一项有可核验的 started+pass 才可本地 pass。optional 的异常仍须报告。
5. **receipt 是唯一权威记录。** HTML、JUnit/TAP 等仅为只读投影；正常结束、拒绝、受控阻断和取消都保留适当记录及失败阶段。摘要复算只排除 `integrity.record_digest`；本地无外部信任锚，不能承诺可防止正文与摘要同时被改写。
6. **本地 L1 不提升信任。** 仅在开发者控制、专用、无秘密环境中运行自有且信任的 PUT/工具；所有 MVP receipt 均为 local-only 且 `market_eligible=false`。网络/文件等不能由 L1 强制的约束明确标记 `unenforced`；不使用 `node:vm` 或普通子进程作为安全沙箱承诺。
7. **显式触发和有界资源。** MVP 不启用 cron；所有请求、输出、附件、运行时间、资源和重试均有预算。超时、取消、worker 崩溃与清理失败都留痕，并验证进程树及临时资源回收。CI 只可在开发者控制、专用且无秘密的 runner/job 中调用同一流程，不能提升信任等级。

## 必须纳入自动化回归的负向场景

| 分类 | 输入/故障注入 | 验收结果 |
|---|---|---|
| 计划完整性 | 空计划、零 required、候选遗漏必测项、未经核验的 `not-applicable`、冻结后增删改节点 | 不产生假 pass；基线保留；冻结后修改拒绝并留证。 |
| 图与预算 | 重复 ID、循环、未解析引用、未绑定输入、schema 不兼容、越权目标/权限、预算超限 | 编译拒绝或按规则阻断，不启动不合规执行。 |
| worker 协议 | 非法/未知 schema 或版本、伪造 verdict/启动时间、错配、重复、乱序、超时后迟到响应、超限响应 | 不采纳为有效通过；原因关联到 attempt；启动证据不足时为 `unknown`。 |
| 状态与门禁 | required fail；required skip/error/inconclusive 与 fail 同时出现；required 缺失；optional fail；Plan 前/后拒绝或取消 | 单独 required fail 为 reject，出现未豁免的 skip/error/inconclusive 时优先 blocked；optional 缺陷仍报告；Plan 前不生成虚构逐项终态，Plan 后拒绝 expected 项记录 `admission_status=rejected`、`execution_status=skipped`、`verdict=skip`、`reason_code=policy_rejected` 和 `invocation_status=not-started`。 |
| 生命周期与恢复 | worker 超时、取消、崩溃、缺输出、重试、Cordis 加载失败/卸载/取消 | 保存可关联状态与既有 attempt；无法证明有效终态时仍记 missing；清理可重复执行且无残留。 |
| receipt 与投影 | receipt 正文被改但摘要未改、导出件被回写、schema 未知、Plan 前失败 | 摘要/校验失败或投影无效，但不影响 canonical 判定；不伪造 `plan_id`、expected 或 verdict。 |
| 信任边界 | L1 不能执行请求的权限；CI 请求 market-required；工具试图读取控制数据或访问未授权目标 | 标注 `unenforced`/local-only；拒绝越权数据流；market-required 因本地 receipt 不具市场适格性返回退出码 `2`，不能返回发布成功。 |

## 协议、receipt 与入口契约

- **版本边界：**`receipt/v1` 是唯一权威 canonical JSON；`test-tool-manifest/v1` 与调用 envelope 的 `test-tool/v1` 分开演进；未知或不兼容的主版本拒绝加载，不做隐式降级。所有消息绑定 `run_id`、版本/schema 和大小上限；测试请求/响应另绑定 `plan_id`、`test_id`、`attempt`、`request_id` 与工具引用；计划冻结前的探测消息按协议显式标记无 `plan_id`，不得伪造该字段。每次响应都要回显并校验其请求关联 tuple。
- **receipt 最小内容：**schema 版本、run/plan/scenario/profile 引用、target 与 PUT/依赖/DSH/Cordis/worker/断言/Policy/环境摘要、能力快照、隔离及实际权限、冻结后的 `expected/executed/missing/waived`、逐 attempt 状态、observation/artifact 引用、verdict、GateDecision、限制说明和完整性字段。Plan 前阻断/取消使用 `plan_id=null`、`reconciliation=null`，记录阶段和原因，不补造 `expected=[]` 或逐项终态。
- **状态分离：**`run_status`、`admission_status`、`execution_status`、`invocation_status`、`verdict`、`gate_decision` 分别由控制面产生或归一化；只有可核验调用开始事件才能写 `invocation_status=started`；明确未启动时为 `not-started`，无法确认是否启动时为 `unknown`。`executed` 只表示控制面采纳了终态，不等同于工具逻辑已经启动。
- **摘要和投影：**按 RFC 8785 规范化 JSON 后，以 SHA-256 生成小写 `sha256:<hex>`；计算输入只排除 `integrity.record_digest`，保留 `signed=false`。必须覆盖架构文档 §11.2.1 的固定向量，期望值为 `sha256:c3b6ee97857bfdb5a83d46fc51aec09e9fb82e6da82a2e21d6101329d88cc2fb`，并额外做完整 receipt schema/对账校验。HTML/JUnit/TAP（以及后续其他格式）只能从 receipt 派生，不能回写门禁。
- **退出码：**本地 profile 的 `pass/reject/blocked` 分别为 `0/1/2`；控制面无法生成有效 receipt 为 `3`。请求 `market-required` 时，任何不具市场适格性的本地 receipt 均返回 `2`，即使本地 `gate_decision=pass`。

## 依赖、评审门和变更控制

- **进入 M1 前：**评审实际运行版本、操作系统与 Node 组合、依赖锁定、schema 具体实现、无秘密 fixture 环境，以及明确的自检入口。架构文档中的源码核对 commit 仅是核对快照，不能替代本次版本选择。
- **进入 M2 前：**协议 envelope、状态词汇、消息大小限制、资源预算和控制面/worker 职责边界通过评审。
- **进入 M3 前：**M1 必测基线和 M2 worker 生命周期具备可重复测试；不能因现有 Cordis runner 或 `node:vm` 而省略独立 worker/launcher 与行为契约测试。
- **进入 M4 前：**Plan 冻结、observation 校验、attempt 关联和 GatePolicy 均可独立测试；receipt schema 与摘要字段规则已冻结。
- **进入 M5 前：**先审查工具权限、固定摘要、依赖来源、schema 远程引用策略及 worker 执行入口；不因组合需求放宽控制面信任边界。
- **协议/schema/断言/Policy 的破坏性调整：**提升 schema/协议版本或提供明确迁移；不得回写旧 receipt。任何新增 runner、网络访问、主动测试、含凭据 CI 或市场资格都必须先补目标授权、资源预算和对应隔离/安全验收，并另行评审范围。

## 主要风险与缓解

| 风险 | 计划中的控制措施 |
|---|---|
| 普通子进程被误认为安全沙箱 | 文档、receipt 和入口均明确 L1/local-only；禁用秘密；无法强制的权限标为 `unenforced`；市场级执行另案验收。 |
| Cordis 静态 API 文档与实际行为不一致 | 锁定真实版本，用生命周期契约测试覆盖状态迁移、参数、错误、取消、卸载和清理。 |
| 必测项被能力判断、worker 输出或重试静默删掉 | 控制面持有基线与 Plan；expected 对账独立计算；能力快照受核验；重复与错配响应拒绝；历史 attempt 追加保存。 |
| 本地摘要被误称为不可篡改 | 明确 JCS/SHA-256 的检测范围及没有外部信任锚的限制；不在 MVP 引入市场签名或归档承诺。 |
| 网络、时钟、GPU、负载等导致结果漂移 | 固定版本和环境指纹，记录随机种子、时钟、区域和网络模式；声明比较容差；无法确定时返回 `inconclusive`。 |
| 市场或监控功能导致 MVP 失焦 | 不实现市场门禁、cron、建议器、回归 baseline 或注册生态；任何扩展作为独立阶段重新评估。 |

## 发布判定

只有 M0–M6 全部完成、架构文档第 16.1 节全部验收项有自动化测试或可复核证据，才可声明“本地自检可用”。发布说明应列出锁定的 DSH/Cordis/Node/OS 版本、已覆盖场景、L1 限制、未实现项和 receipt 的完整性局限。即使 MVP 全部通过，仍必须保持 `market_eligible=false`；市场门禁需要后续独立完成强隔离、来源治理、证据签收/归档和准入策略验收。

## 依据

- [架构文档](./architecture.md)：重点参考 §0、§4–5、§7–9、§11–13、§15–16.1。
- [开发文档](./development.md)：重点参考版本锚定、运行链、状态与 receipt 语义、§“开发迭代与交付验收”及负向用例。
