# 基本信息

技术/论文标题：BreachWeave: Manager/Solver/Observer 三角色多 Agent 协作的 CTF 自动化框架

作者/团队：M-SEC 社区，中文安全社区组织。GitHub 上有 ez-ai-agent 158 star、ez-xbow-platform-mcp 94 star、wafkiller 51 star 等多个安全工具项目。BreachWeave 获腾讯云黑客松智能渗透测试挑战赛一等奖，排名 1/613，竞赛成绩是权威性的有力支撑

发表刊物/来源：GitHub 开源项目

发布时间：2026年9月

地址：https://github.com/m-sec-org/BreachWeave

类型：开源项目

标签：多 Agent 协作、CTF 自动化、渗透测试、黑板架构、旁路监督

技术成熟度：可用阶段，代码完整开源，Web UI 和 CLI 双入口，Web UI 服务已验证可启动

# 一句话总结

把多 Agent 治理拆成 Manager、Solver、Observer 三个正交角色，通过共享的 Idea 和 Memory 看板实现间接协作，用旁路监督机制解决单个 LLM Agent 在复杂安全任务中容易陷入低效循环的问题。

# 关键结论

核心结论：BreachWeave 不靠堆模型能力，靠架构治理来提升 Agent 的有效工作时长和决策质量。三个角色分工明确：Manager 是 LLM 驱动的调度器，Solver 是在容器中运行的解题 Agent，Observer 是独立的旁路监督者。三者通过 challenge 级看板间接协作，典型的共享黑板架构。

适用场景：CTF 竞赛自动化，渗透测试辅助，多 Agent 协作架构设计参考。

不适用场景：简单单步任务，不需要多 Agent 协作的场景。

是否值得关注：值得。Observer 旁路监督机制和 Idea 与 Memory 分层看板设计在多 Agent 治理方面比较新颖，不只是又一个 CTF 框架。

# 为什么值得关注

背景与问题：单个 LLM Agent 在复杂安全任务比如 CTF 和渗透测试中容易陷入低效循环、状态混杂、过早终止。这些问题的根源不是模型能力不够，而是缺乏有效的外部监督和状态管理。

现有方案存在什么不足：大多数多 Agent 框架用消息传递做直接通信，Agent 之间耦合紧密，一个 Agent 的状态污染会影响其他 Agent。监督机制通常是看门狗式的，检测死循环或超时然后杀进程，粒度太粗。

为什么现在值得关注：BreachWeave 的 Observer 是一个完整的独立 LLM Agent 会话，不是规则引擎。它通过 SDK 事件钩子被动收集 Solver 行为轨迹，异步审查并维护看板，用双重指纹防骚扰确保轻量纠偏。这种旁路监督模式比传统看门狗精细得多，而且可以推广到任何需要 Agent 行为监督的场景。

# 技术细节

核心思想：三角色正交分离。Manager 通过 5 个工具自主决策调度 Solver 的启停，Solver 消费看板但不拥有写权限，Observer 独立于 Solver 通过事件钩子被动收集行为并维护看板。三者通过 challenge 级看板耦合，看板变更通过广播同步。

## Observer 旁路监督机制

这是整个架构中最精巧的部分。Observer 通过 4 个 SDK 扩展事件监听器被动收集 Solver 行为：工具执行开始和结束、消息结束、Agent 结束。审查任务通过文件队列异步入队，不阻塞主 Solver。三种触发 reason 分别是每 6 轮的周期性审查、Solver 请求 hint 后的强制触发、以及会话结束时的强制触发。

每次审查时 Observer 创建一个独立的 Agent 会话，不读取 Solver 完整会话，而是通过构建压缩上下文来审查：最近 10 轮活动日志加 baseline 指令和最近 4 条 user 消息。这样 Observer 的审查成本可控，不会因为 Solver 上下文膨胀而爆炸。

防骚扰机制是关键。Observer 通过双重指纹避免重复打扰：消息指纹去重同一提醒文本，活动指纹检测 Solver 是否已切换到新路径，冷却窗口 6 轮内不重复打扰。只有持续低效且未改线时 Observer 才介入，保证监督不会变成噪音。

## Idea 和 Memory 分层看板

两个独立的数据结构。Idea 是候选假设，有 pending、testing、verified、failed、skipped 五种状态。Memory 是持久化的事实、证据、失败记录，按 kind 分类。并发安全通过目录锁加原子写入保证，5 秒超时加 60 秒陈旧锁清理。

角色边界通过工具权限分离实现：Observer 拥有全套写权限，主 Solver 只有只读加追加权限。这保证了 Solver 不会自行修改决策方向，所有方向性调整都经过 Observer 审查。

与现有方案的区别：传统多 Agent 框架用消息传递直接通信，BreachWeave 用共享黑板间接协作。传统监督是看门狗式检测死循环杀进程，BreachWeave 的 Observer 是独立 LLM 会话做精细旁路审查。

# 验证与证据

实验设置：Web UI 服务已验证可启动，API 端点可访问，Solver 可通过 API 程序化触发。容器内代码阅读和文件结构验证通过。

数据集：仓库自带示例任务，通过 Docker 容器运行 Solver。

核心指标：框架自身的可用性验证，未做端到端 CTF 成绩评测。

主要结果：Web UI 服务成功启动，POST /api/runtime/solvers 端点可以程序化触发 Solver 运行。

作者结论：无正式论文，项目仍在开发中。

# 复现情况

是否复现：部分复现。服务启动和 API 触发验证通过，端到端 CTF 实验未跑。

复现环境：TypeScript 加 Bun 运行时，依赖 @mariozechner/pi-coding-agent 作为 Agent SDK，Commander 做 CLI，Docker 运行 Solver 容器。

复现步骤：

```
bun apps/cli/src/main.ts web
# 然后访问 http://127.0.0.1:3000
# 或程序化触发：
# POST /api/runtime/solvers with {"promptName":"<name>","task":"<task>","env":{}}
```

复现结果：Web UI 服务成功启动，API 端点可访问，Solver 可通过 API 程序化触发。容器内 git 状态检测失败因为容器内未初始化 git 仓库，不影响代码阅读和运行。

复现成本：低。Bun 安装方便，Docker 环境 Solver 容器可自动构建。需要配置 LLM API key。

# 局限与风险

技术局限：容器内未初始化 git 仓库，无法通过 git 获取版本信息。项目仍在开发中，没有正式论文和标准评测结果。

实验局限：未实跑端到端 CTF 或渗透测试实验，仅验证了服务启动和 API 触发路径。实际 CTF 成绩和 Agent 协作效果未验证。

适用边界：需要 LLM API key 和 Docker 环境。Agent SDK 是特定实现，切换到其他 Agent 框架需要重新适配事件钩子。

工程风险：依赖 @mariozechner/pi-coding-agent 这个特定 SDK，如果该 SDK 不再维护或接口变更，迁移成本较高。Solver 运行在 Docker 容器内通过 RPC 通信，网络配置不当可能影响连通性。

# 个人思考

## 可借鉴性

Observer 旁路监督机制是这套架构里最有价值的部分。事件驱动加独立 sidecar LLM 会话加双重指纹防骚扰，这是一个可命名的架构模式，区别于传统看门狗和主 Agent 自我反思。它是第三方 Agent 通过事件被动观察、异步审查、轻量纠偏的完整模式，适合任何需要 Agent 行为监督和质量评估的场景。

Idea 和 Memory 分层看板加工具权限分离也值得借鉴。把候选假设和持久事实分成两个独立数据结构，通过工具权限控制谁有写权限谁只有读加追加，这种状态治理模式比单一 context 堆叠精细得多。目录锁加原子写入的并发安全实现可以直接移植。

可以应用到哪些现有项目：多 Agent 协作框架设计、Agent 行为监督基建、渗透测试自动化工具链。

后续是否跟进：会关注。主要看项目是否会做标准 CTF benchmark 评测，以及 Observer 机制在大规模任务下的稳定性。

下一步行动：评估 Observer 旁路监督模式能否移植到团队内部的 Agent 测试基建中，另外看看双重指纹防骚扰算法是否可以独立复用。