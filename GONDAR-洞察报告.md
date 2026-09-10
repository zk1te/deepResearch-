# 基本信息

- 技术/论文标题：GONDAR / atlantis-java: Contextualizing Sink Knowledge for Java Vulnerability Discovery
- 作者/团队：Fabian Fleischer、Cen Zhang、Joonun Jang、Jeongin Cho、Meng Xu、Taesoo Kim。Taesoo Kim 是 Georgia Tech 知名安全学者，团队是 DARPA AI 网络挑战赛 AIxCC 的参赛队伍 Team Atlanta，早期版本助力拿下第一名
- 发表刊物/来源：IEEE S&P 2026，arXiv
- 发布时间：2026年4月
- 地址：https://arxiv.org/abs/2604.01645v3  https://github.com/Team-Atlanta/atlantis-java
- 类型：
  - [ ] 技术趋势
  - [x] Paper
  - [ ] 开源项目
- 标签：Java 漏洞挖掘、LLM 代理、模糊测试、Sink 知识、覆盖率引导、DARPA AIxCC
- 技术成熟度：工业部署阶段，已集成到 Linux 基金会 OpenSSF 的 OSS-CRS 项目，兼容 OSS-Fuzz 基础设施，已发现零日漏洞

# 一句话总结

把安全敏感 API 调用点的漏洞语义知识系统性注入 LLM 推理上下文，让 LLM 代理与覆盖率引导模糊测试器 Jazzer 双向闭环协同，探索代理负责生成到达 sink 的输入，利用代理负责把输入升级为触发漏洞的 PoC，在 54 个真实漏洞上比 SOTA Jazzer 多发现 4 倍漏洞。

# 关键结论

核心结论：Jazzer 在 54 个漏洞基准上仅发现 8 个，GONDAR 发现的数量是它的 4 倍。核心创新是把漏洞发现分解为探索和利用两个子任务，由两个专门化的 LLM 代理分别负责，再与覆盖率引导的模糊测试器并发运行、持续交换种子和运行时反馈。

适用场景：Java 应用的漏洞挖掘，覆盖率引导模糊测试器的增强，安全敏感 API 的可利用性评估。

不适用场景：非 Java 程序的漏洞挖掘需要重新适配，没有 sink 概念的场景不适用。

是否值得关注：很值得。论文发在 IEEE S&P，团队是 DARPA AIxCC 第一名，而且已经实际部署到 OSS-Fuzz 基础设施并发现零日漏洞，不是停留在论文阶段的框架。

# 为什么值得关注

背景与问题：Java 应用大量漏洞源于对安全敏感 API 的不安全使用，比如文件操作导致路径遍历、反序列化导致远程代码执行。这些 sink API 本身编码了两类关键信息，到达它所需的程序特定约束和触发安全缺陷所需的利用条件。然而传统模糊测试器包括 SOTA 的 Jazzer 几乎完全忽略这类漏洞特定知识，只做覆盖率驱动的盲目变异。即便 fuzzer 的执行碰巧经过了某个 sink 调用点，也极难凑出满足触发条件的输入，比如绕过 sanitizer、构造特定反序列化 gadget 链。

现有方案存在什么不足：纯覆盖率引导的 fuzzer 没有 sink 语义知识，到达 sink 和触发漏洞是两个完全不同难度的问题，覆盖率不能区分。纯 LLM 推理虽然能理解语义但缺乏运行时反馈，生成的输入往往无法满足具体的路径约束。

为什么现在值得关注：GONDAR 把这两条线结合起来。LLM 负责理解 sink 语义和推理路径约束，fuzzer 负责覆盖率驱动的变异和运行时验证，两者双向闭环。探索代理生成的输入丰富 fuzzer 语料库供变异，fuzzer 发现的到达 sink 的输入为利用代理提供起点。这种协作模式比单一方法强得多，而且已经有实际零日漏洞发现和工业部署作为背书。

# 技术细节

核心思想：上下文化 Sink 知识。将三类知识系统性注入 LLM 的推理上下文。第一类是漏洞语义知识，每种 CWE 类型的漏洞描述、可利用性判断因素、Jazzer sanitizer 的精确触发条件。第二类是程序结构知识，从 fuzzerTestOneInput 入口到 sink 的调用路径、代码上下文、def/use 数据流。第三类是运行时状态，beepseed 的 hexdump、stacktrace、路径覆盖率评分。

## Sink 检测流水线

从 CodeQL 提取的大量候选 sink 调用点中缩减到真正有漏洞潜力的目标。流水线分四步：CodeQL CWE 特定提取、有效性过滤排除常量参数和测试代码、可达性过滤在 Joern 构建的调用图上从入口做 BFS、LLM 可利用性评估用两阶段 LangGraph 做研究探索加判定。可达性分析会把每个 sink 的坐标匹配到调用图节点，路径规范化处理 repo 和 oss-fuzz 前缀，匹配条件为文件名相同且行号在方法范围内。LLM 可利用性评估是核心，为 12 种 CWE 各自定义可利用性因素清单，把什么样的输入算成功利用从模糊的自然语言精确化为可判定的 sanitizer 触发条件。

## 双代理协同与 BeepSeed 闭环

探索代理负责生成能到达目标 sink 调用点的输入。它推理输入格式要求、分支条件约束和 API 语义，通过 JDB 断点追踪实际执行路径，量化偏离程度，LLM 定位偏离点加 Joern 查询外部调用，组合反馈注入下一轮生成。利用代理接收已到达 sink 但未触发 sanitizer 的输入即 beepseed，基于执行上下文包括堆栈、程序状态、sink 利用细节合成 PoC，通过运行时反馈迭代改进。

两个代理与覆盖率引导的模糊测试器并发运行，持续交换种子和运行时反馈。ExpKit 监控 Jazzer 输出目录，读取到达 sink 但未触发 sanitizer 的输入，经两层调度送入利用代理。利用成功后 PoC 与语料库文件广播回所有 fuzzer 实例，丰富 fuzzer 语料库供变异。这套 fuzzer 发现到达输入、agent 升级为 PoC、PoC 回传 fuzzer 语料库的双向协议是整个框架的协作核心。

## 调试器路径偏离评分与反馈闭环

探索代理用 DebuggerVerifier 在目标路径每个 CodePoint 设 JDB 断点，解析 Breakpoint hit 获取实际访问行号，计算 score 等于 last_visit 加 1 除以 path_length 来量化输入沿目标路径走了多远。偏离时用 LLM 定位偏离位置，用 Joern 查询偏离点附近的外部函数调用，组合成自然语言反馈注入下一轮输入生成。这套调试器量化路径偏离、LLM 定位偏离点、结构化查询补全外部调用、自然语言反馈的闭环，适合任何 LLM 生成输入加调试器路径追踪的可达性求解场景。

## FDP 字节布局抽象

LLM 只需推理 FuzzedDataProvider 方法调用序列，无需理解底层字节布局。FDPBlobGenFactory 维护 FuzzedDataProvider 方法到 libfdp encoder 方法的完整映射表，将 LLM 生成的 JSON 格式方法调用序列验证后转换为 libfdp encoder 脚本执行，生成符合 FDP 字节布局的 blob。这让 LLM 摆脱了底层字节布局的复杂性，专注于高层语义推理。

# 验证与证据

实验设置：在 54 个真实漏洞基准上评测，GONDAR 与覆盖率引导的 Jazzer 并发运行，双代理协同，持续交换种子和运行时反馈。

数据集：54 个真实 Java 漏洞，覆盖多种 CWE 类型。

对比方法：Jazzer，当前 Java 模糊测试 SOTA。

核心指标：发现的漏洞数量，sink 触发率，PoC 有效性。

主要结果：Jazzer 仅发现 8 个漏洞，GONDAR 发现数量是它的 4 倍。已发现零日漏洞并已集成到 OSS-Fuzz 基础设施。

作者结论：将 sink 漏洞语义知识注入 LLM 推理上下文，配合双代理与 fuzzer 的双向闭环，可以显著提升 Java 漏洞发现能力。

# 复现情况

是否复现：部分复现。代码完整开源，但完整实验需要 OSS-Fuzz 基础设施环境。

复现环境：Java，依赖 Jazzer、CodeQL、Joern、Redis、JDB，LLM 部分依赖 LangGraph。已集成到 OSS-Fuzz 基础设施。

复现步骤：

```
git clone https://github.com/Team-Atlanta/atlantis-java
cd atlantis-java
# 参考 crs/ 目录下各模块的运行脚本
```

复现结果：仓库是 OSS-Fuzz 集成的 snapshot，代码结构完整，核心模块包括 filtering-agent、llm-poc-gen、expkit、javacrs_modules 均可阅读和调试。完整漏洞挖掘实验需要配置 OSS-Fuzz 沙箱环境和多个工具链。

复现成本：中等。单模块可以独立阅读和调试，完整实验需要 Jazzer 加 CodeQL 加 Joern 加 Redis 加 LLM API 的完整工具链。

# 局限与风险

技术局限：仅适用于 Java 程序的漏洞挖掘，sink 检测依赖 CodeQL 和 Joern 的 Java 支持。LLM 推理成本不低，完整 sweep 需要大量 API 调用。

实验局限：54 个漏洞基准规模有限，虽然比 Jazzer 强 4 倍但绝对数量不大。实际零日漏洞发现的细节未完全公开。

适用边界：需要 sink API 的概念，非 sink 驱动的漏洞类型比如内存安全漏洞不适用。依赖 Jazzer 的 sanitizer 机制，其他 fuzzer 需要适配。

工程风险：工具链复杂，CodeQL、Joern、Jazzer、Redis、JDB 的版本兼容性和配置门槛高。SinkManager 的 Redis 跨 pod 状态同步在非 Kubernetes 环境下需要适配。

# 个人思考

## 可借鉴性

这套框架里有几个机制可以直接拿来用。结构化 CWE 语义知识库是最直接的，为 12 种 CWE 各自定义可利用性因素清单和 Jazzer sanitizer 精确触发条件，把什么样的输入算成功利用从模糊自然语言精确化为可判定条件，适合任何 LLM 加 Jazzer 的 Java 漏洞挖掘场景。BeepSeed 双向反馈协议，fuzzer 发现到达输入、agent 升级为 PoC、PoC 回传 fuzzer 语料库，适合任何 LLM 代理加覆盖率引导 fuzzer 协同的 harness 基建。调试器路径偏离评分加 localize/external 反馈闭环，适合任何 LLM 生成输入加调试器路径追踪的可达性求解场景。FDP 字节布局抽象让 LLM 只推理方法调用序列、自动翻译为字节布局正确的 blob，适合任何 LLM 生成 Jazzer FDP 输入的 harness。渐进式代码上下文扩展用 LLM 判断关键行、Joern 补全到方法或成员边界，适合给 LLM 提供聚焦且完整代码上下文的代码审计场景。

## 局限与风险

54 个漏洞基准的规模偏小，4 倍提升的绝对值需要谨慎看待。工具链复杂度高，CodeQL 加 Joern 加 Jazzer 加 Redis 加 JDB 的完整配置门槛不低，团队内部落地需要评估工具链维护成本。LLM 推理成本在完整 sweep 下不可忽略。

## 下一步行动

评估 BeepSeed 双向反馈协议能否移植到团队内部的模糊测试流水线中，看 LLM 代理加 fuzzer 协同的模式是否适用于我们现有的测试基建。另外试跑一下 filtering-agent 模块，看 sink 检测流水线在我们的 Java 项目上的效果。
