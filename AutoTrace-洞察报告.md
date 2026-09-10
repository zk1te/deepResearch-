# 基本信息

- 技术/论文标题：AutoTrace: From Patches to Triggers via Agentic Interprocedural Exploration
- 作者/团队：Arastoo Zibaeirad（UNC Charlotte）、Marco Vieira（知名容错与安全研究学者）、Thomas Zimmermann（知名软件工程研究学者，微软研究院）。作者阵容跨软件工程和安全两个领域，但核心流水线代码未开源，仅开放评估工件
- 发表刊物/来源：arXiv
- 发布时间：2026年7月
- 地址：https://arxiv.org/abs/2607.12058v1  https://github.com/Erroristotle/AutoTrace
- 类型：
  - [x] 技术趋势
  - [x] Paper
  - [x] 开源项目
- 标签：漏洞触发定位、LLM 智能体、代码属性图 CPG、跨过程分析、AI for Security
- 技术成熟度：研究阶段，论文配套了 artifact 评估仓库，但流水线本体代码没有开源

# 一句话总结

用 LLM 智能体做向导来跨过程探索调用图，再用 CPG 数据流证据做确定性门控来把关，最终把漏洞的触发语句定位到补丁函数外面好几层调用的地方，靠图验证而非模型自报来消除幻觉。

# 关键结论

核心结论：AutoTrace 在 InterPVD 全量基准上拿到 75.0% VulnHit 和 80.8% FuncHit，比之前最好的 VulTrigger 高了五个多百分点。思路上的关键在于职责分离，智能体负责探索和判断方向，CPG 负责出证据，每个触发语句都得有数据流证据链撑着，不能模型自己说了算。

适用场景：有补丁的漏洞做触发语句精确锁定，跨过程数据流追踪，漏洞因果链构建和基准数据集制作。

不适用场景：没有补丁就没法跑，纯漏洞发现用不了；流水线本体不开源，想直接拿来跑单个 CVE 的定位做不到。

是否值得关注：值得。CPG 门控消除幻觉这个点，对做 AI for Security 的人来说有实际借鉴意义。

# 为什么值得关注

背景与问题：漏洞触发语句定位这件事比简单的漏洞检测难太多。检测只要说有没有漏洞就行，定位得说清楚到底是哪一行代码把脆弱状态变成了真实的危险操作。论文里提到，很多真实 CVE 的触发语句藏在补丁函数外面好几层调用处，静态规则够不到，模式匹配型模型也够不到，因为这本质上需要跨过程的因果推理。

现有方案存在什么不足：VulTrigger 这类方法依赖函数内模式匹配，追不到函数外面的触发语句；纯靠 LLM 推理又没有确定性验证，模型容易给出看似合理但实际错误的结论。

为什么现在值得关注：LLM 智能体的代码理解能力已经到了能做跨过程探索的程度，加上 CPG 提供确定性验证，这两者结合恰好填上了跨过程因果推理和防止幻觉之间的空白。另外作者还做了 SinkTrace-Bench，1542 个样本、771 对漏洞和安全代码匹配对，拿前沿 LLM 去跑发现就算最强的模型也很难区分这些匹配对，说明因果推理确实是当前模型的短板。

# 技术细节

核心思想：职责分离。LLM 智能体当向导，负责解释代码、提取漏洞相关的关键变量、在调用图里决定下一步去哪找；CPG 确定性准入门当裁判，设硬条件，每个被报告的触发语句都必须有一条从关键变量到触发语句的逐行数据流证据链，没有证据就不算数。

## 流程与架构

先输入漏洞修复补丁，智能体识别出关键变量，然后在调用图里跨过程探索，沿着数据流追踪到候选触发语句，CPG 门控接着验证，构建从关键变量到候选语句的逐行数据流证据链，最后输出一个 trigger 对象，里面带 dataflow_proof 证据链、sink_category、depth、call_chain、confidence 这些字段。关键模块有三个：LLM Agent 负责跨过程探索，CPG Gating 负责确定性数据流证据准入，评估流水线负责多级选优和防虚假命中。

## 核心创新点

我觉得有三个。第一是 dataflow_proof 证据链，触发器不是模型嘴上说的，是图里验证出来的，比如从 L7093 的 width 到 L7097 的 image 加 row 乘 width 加 col，再到 L7102 的 val 等于 pix，一条链清清楚楚。第二是 patch-fallback 扣留机制，如果一个候选只是靠函数名匹配命中了，而那个函数恰好是补丁函数，这个分就扣住不算，防止补丁位点冒充触发器。第三是 SinkTrace-Bench 的匹配对设计，同一个 CVE 同一个关键变量生成一对漏洞代码和安全代码，让 LLM 必须理解因果链才能区分，光看代码模式没用。

与现有方案的区别：VulTrigger 拿 69.8% 的 VulnHit 靠的是函数内模式匹配，AutoTrace 靠智能体跨过程探索加 CPG 验证拿到 75.0%，而且每个触发语句都有可审计的数据流证据链。

# 验证与证据

实验设置：实验在 InterPVD 全量基准上跑，8 个独立 RQ 脚本各自单独运行，输出 JSON 报告。评估这块设计得很细致，best_per_cve 会把同一个 CVE 的多个预测按多级元组排序挑出最佳代表，从函数匹配到行匹配到精确匹配到 5 行内匹配逐级比较，行号差越小越好；patch-fallback 扣留会挡住补丁位点的虚假命中。

数据集：用了已有的 InterPVD，加上本文新构建的 SinkTrace-Bench，后者 1542 个样本、771 对漏洞安全匹配对，每个样本包含 source-to-sink 因果链代码和 diff。

对比方法：VulTrigger，此前 SOTA，69.8% VulnHit。

核心指标：VulnHit、FuncHit、Within±5、per-CWE 分解和跨过程深度分桶。

主要结果：AutoTrace 75.0% VulnHit、80.8% FuncHit，超 VulTrigger 5.2 个百分点。

作者结论：CPG 门控优于纯 LLM 推理，而且前沿 LLM 在 SinkTrace-Bench 上即便最强也难以区分漏洞和安全匹配对，因果推理差距明显。

# 复现情况

是否复现：部分复现。评估代码能跑，流水线本体跑不了。

复现环境：Python，依赖 anthropic、google-genai、openai 这些做 LLM 基线对比，内部还有 evaluate、_common、_log_ablation 几个模块。

复现步骤：

```
git lfs install
git lfs pull
python evaluation/RQ1_1_overall_effectiveness.py --results-dir results --output evaluation/output/rq1_report.json
```

复现结果：评估脚本能独立跑通，输出 JSON 报告。但流水线本体也就是 LLM agent 加 CPG gating 那套代码不在仓库里，没法实跑定位单个 CVE，只能通过 results 目录的工件和 logs 目录的日志逆向重构它的行为。

与原结果差异：评估侧可以验证，因为 results 和 logs 都在；流水线侧没法直接复现。

复现成本：评估脚本这边成本低，直接跑就行；流水线这边成本高，得自己实现 agent 和 CPG gating。

# 局限与风险

技术局限：流水线实现不在仓库，只有 artifact 和评估代码；没有 pyproject.toml 或 requirements.txt，依赖全靠推断；evaluate 模块可能缺失。

实验局限：codex_ground 降级了，实验接地轨道没启用，宣称的评估没有在保留容器里机检复现，实验结论是基于源码深读和工件逆向重构得出的。

适用边界：只能处理有补丁的情况，没补丁就没法工作；CPG 依赖 Joern 这类工具，构建成本不低。

工程风险：依赖未声明，实际跑的时候可能缺 evaluate 等内部模块或者需要额外配置；InterPVD 的 GT 数据文件要手动放到 data/interpvd 下面。

# 个人思考

优缺点：CPG 门控消除幻觉这个设计简洁有力，dataflow_proof 证据链可审计、可追溯，SinkTrace-Bench 匹配对设计也很精巧，逼着模型理解因果链而不是看表面模式。缺点方面，流水线本体不开源严重限制了可复现性，依赖没声明也降低了开箱可用性，想拿来直接用不太现实。

## 可借鉴性

best_per_cve 的多级元组选优适合任何一个 GT 对应多个预测的定位评估场景；patch-fallback 扣留适合任何基线信息天然相关、需要排除代理作弊的评估；日志正则解析逆向重构 agent 事件做消融，适合 agent 流水线的组件归因；SinkTrace-Bench 的匹配对设计适合 LLM 因果推理评测对齐。这几条拆开看都不复杂，但组合在一起确实构成了一个严谨的评估框架，值得拿过来用。

可以应用到哪些现有项目：团队现有的漏洞定位评估流程设计、agent 流水线评估基建、漏洞安全匹配对基准构建这些方向。

后续是否跟进：会跟进，主要关注两点，流水线本体是否会开源，以及 SinkTrace-Bench 在更多 LLM 上的评测扩展情况。

下一步行动：在团队内部 AI for Sec 评估流程里引入 best_per_cve 选优和 patch-fallback 扣留机制，另外评估一下 SinkTrace-Bench 能不能用来做团队 LLM 安全能力评测。

