---
title: "作为推理的检索：通过 LLM-Wiki 实现自演化的智能体原生检索"
original_title: "Retrieval as Reasoning: Self-Evolving Agent-Native Retrieval via LLM-Wiki"
source_url: "https://arxiv.org/abs/2605.25480"
authors:
  - Haoliang Ming
  - Feifei Li
  - Xiaoqing Wu
  - Wenhui Que
---

> 本文为英文论文中文译本，仅供阅读参考。原文见 source_url。

# 作为推理的检索：通过 LLM-Wiki 实现自演化的智能体原生检索

Haoliang Ming, Feifei Li, Xiaoqing Wu, Wenhui Que∗

微信，腾讯公司，中国北京

{hliangming, niyali, xiaoqingwwu, victorque}@tencent.com

## 摘要

LLM 智能体需要检索表现得不再像一次性的上下文获取，而更像推理：搜索、阅读、遍历，并判断证据
何时已经充分。然而，当前的检索增强生成（Retrieval-Augmented Generation, RAG）系统将外部知识组织
为由嵌入相似度检索的扁平片段，暴露出一种“检索即查找”的接口，并不适合迭代式推理智能体。我们
提出 LLM-Wiki，这是一种智能体原生检索系统，它通过把外部知识视为可编译、可组合且自演化的结构，
而不是静态检索索引，来操作化“作为推理的检索”（Retrieval-as-Reasoning）范式。LLM-Wiki 将文档
编译为带有双向链接的结构化 Wiki 页面，通过标准工具调用接口暴露搜索、阅读和链接跟随操作，并引入
Error Book，用于持久的结构与语义自我纠错。LLM-Wiki 在 HotpotQA、MuSiQue 和 2WikiMultiHopQA 上
取得最先进结果，比 HippoRAG 2、LightRAG 和 GraphRAG 高出 2.0--8.1 个 F1 点。在 AuthTrace 上，
LLM-Wiki 取得最佳总体准确率，尤其在多文档结构化查询上增益显著，证实了基于编译的检索能够泛化到
链式多跳推理之外。

## 1 引言

考虑 2WikiMultiHopQA 中一个 4 跳问题：“哪部电影的导演年龄更大，*The Gamecock* 还是 *Monster A
Go-Go*？”密集检索器可能检索到两部电影的页面，却漏掉包含出生日期的导演传记页面，因为这些页面在
语义上与原始查询相距较远。没有这种中间证据，系统会回答错误。然而，能够访问结构化链接的智能体
可以阅读电影页面，沿着显式指针进入导演传记，并通过组合式遍历给出答案。这个例子揭示了 RAG
（Lewis et al., 2020）的一个根本限制：瓶颈不仅在检索算法，更在知识如何被组织并暴露给智能体。
随着基于 LLM 的智能体越来越多采用 ReAct 风格（Yao et al., 2023）的工具调用循环，它们需要能够在
推理展开过程中被搜索、阅读和遍历的知识。

我们将当前占主导地位的模式称为**检索即查找**（retrieval-as-lookup）：检索模块选择一组固定段落，
推理模块消费这些段落，而检索本身不会根据智能体的中间观察进行自适应。这促使我们转向**作为推理的
检索**（retrieval-as-reasoning）：智能体会搜索、检查、跟随中间实体、修订检索计划，并且只有在收集
到的证据充分时才停止。

核心瓶颈是知识组织，而不只是检索。当前 RAG 系统把文档切分为扁平片段并存储在向量数据库中。这带来
三项仅靠更好的检索算法无法解决的限制。

第一，**扁平片段把检索降格为匹配，而不是推理。** 选择最接近查询嵌入的 \(top-k\) 片段足以支持局部
事实查找，但当智能体必须跟随关系、比较属性或汇总跨文档证据时，这种方式会变得脆弱。

第二，**检索仍然是一次性的黑盒，而不是由智能体控制的推理过程。** 系统在推理开始前检索固定上下文，
因此智能体无法决定下一步检查哪个实体、跟随新发现的关系，或根据部分证据修订检索。

第三，**结构化知识库如果没有自我纠错就会退化。** 由 LLM 编译的知识库可能引入悬空链接、索引不一致、
无支撑事实和跨页面矛盾，但现有流水线缺乏用于检测、修复并减少此类错误复发的持久循环。

已有方法解决了这一图景中的部分问题，但尚未弥合差距。RAPTOR 和 GraphRAG 生成压缩摘要，HippoRAG 2
暴露 KG 三元组，LightRAG 依赖实体与关系索引；这些产物改进了检索，但并没有提供一个可自我纠错、
人类可读、并可供智能体规划和遍历的知识表面。附录 A 的表 4 给出了详细比较。

这些限制引出三个研究问题：**(Q1)** 基于编译的知识组织能否相较于扁平 RAG 与图增强检索基线，改进
知识密集型问答？**(Q2)** 智能体控制的组合式遍历如何通过搜索、阅读、链接跟随和充分性检查来利用
Wiki 结构？**(Q3)** 系统性的结构与语义编译错误能否通过持久 Error Book 被检测、修复和缓解？

在这些问题指引下，我们提出 LLM-Wiki。该系统操作化了 Retrieval-as-Reasoning 范式，其中智能体的
检索过程本身就是一种组合式推理活动：规划要访问的知识单元、阅读证据、跟随已编译链接、在证据不足
时发起修订后的搜索，并决定何时回答。LLM-Wiki 通过三种机制实例化这一范式。第一，它把原始文档编译
为结构化、互相链接的 Wiki 页面，而不只是分块并嵌入。第二，它通过标准工具调用接口暴露这些页面，
使智能体能够组合搜索、阅读、链接跟随与充分性检查操作。第三，它通过 Error Book 支持自演化：记录
系统性编译错误、归因根因、将其转化为可复用约束，并驱动确定性代码修复和周期性 LLM 修复。

据我们所知，LLM-Wiki 是第一个操作化 Retrieval-as-Reasoning 范式的智能体原生检索系统。我们的贡献
总结如下：

1. **通过知识编译与组合式遍历实现 Retrieval-as-Reasoning 范式。** 我们提出 LLM-Wiki，这是一种
   智能体原生检索系统，它将原始文档编译为结构化、互相链接的 Wiki 页面，并把检索从固定 \(top-k\)
   上下文选择转化为由智能体规划的组合式遍历。
2. **通过 Error Book 实现自演化 Wiki 知识。** 我们引入一种持久自我纠错机制，用于检测结构与语义
   编译错误、归因根因、积累可复用约束，并在摄取批次之间驱动双层修复。
3. **证明基于编译的检索会随推理深度扩展的经验证据。** 我们在三个公开多跳问答基准和 AuthTrace 上
   评估 LLM-Wiki，展示其相较七个基线的一致增益、随跳数增加而扩大的改进，以及在多文档结构化知识
   查询上的最强总体性能。消融实验进一步表明，Wiki 结构、渐进式遍历和 Error Book 都贡献了增益。

## 2 相关工作

**检索增强生成与层次化检索。** RAG（Lewis et al., 2020）用检索到的段落增强语言模型，而 DPR
等密集检索器（Karpukhin et al., 2020）改进了段落选择。对于多跳推理，IRCoT（Trivedi et al.,
2023）和 Self-RAG（Asai et al., 2024）将检索与推理或自我反思交错进行，而 RAPTOR（Sarthi et al.,
2024）和 MemWalker（Chen et al., 2023）把文档组织成摘要树。然而，这些方法并未提供一种显式链接的
结构，使智能体能够在其上规划组合式遍历路径。

**图增强 RAG 与知识编译。** GraphRAG（Edge et al., 2024）、HippoRAG（Gutiérrez et al., 2024）、
HippoRAG 2（Gutiérrez et al., 2025）和 LightRAG（Guo et al., 2025）等基于图的系统，用实体、关系
或基于图的索引来丰富检索。虽然它们与我们一样都希望把知识组织到扁平片段之外，但其输出通常是摘要、
三元组或向量化图结构，而不是人类可读且可被智能体遍历的页面。相比之下，LLM-Wiki 将文档编译为带有
显式双向链接的 Wiki 页面。表 4 提供了详细比较。

**LLM 自我纠错与基于智能体的检索。** LLM 自我纠错已在 Reflexion（Shinn et al., 2023）、Self-Refine
（Madaan et al., 2023）和 Self-RAG（Asai et al., 2024）中得到研究。这些方法在单次推理回合内或在
查询时运行，而 Error Book 针对知识构建中跨批次持久存在的错误模式，通过约束积累和修复来处理。更
广义地说，Karpathy（2026）提出的 LLM Wiki 范式主张把非结构化文档编译为持久的结构化知识存储，但
它仍是一种抽象设计模式。我们通过编译、质量保证和组合式检索将这一范式操作化。

## 3 方法

图 1 展示了 LLM-Wiki 的索引时构建过程。原始文档被编译为链接的 Wiki 页面，同时 Error Book 记录验证
失败、注入约束并驱动修复。经过验证的 Wiki 构成一个人类可读且机器可遍历的底座，用于下游遍历。
图 2 概览了两条遍历路线。

**Retrieval-as-Reasoning 原则。** 我们通过三项原则形式化 Retrieval-as-Reasoning 范式，认为一个
智能体原生检索系统应当满足这些原则：(1) **可编译性** —— 原始文档被转换为结构化、显式链接的单元，
并能作为持久知识库维护；(2) **可组合性** —— 检索被分解为搜索、阅读和链接跟随等原子操作，由智能体
在其推理循环中组合；(3) **可演化性** —— 知识结构会随时间自我纠错，而不是悄然退化。LLM-Wiki 分别
通过 Wiki 结构化知识、组合式检索和 Error Book 实例化这些原则。合在一起，这些原则把检索从隐藏的
排序步骤转化为在受维护知识结构上进行的显式推理过程。

### 3.1 Wiki 结构化知识

LLM-Wiki 通过把知识表示为持久 Wiki 而不是独立片段来实现**可编译性**。Wiki 由目录索引、结构化
Markdown 页面和源文档归档组成。每个页面暴露元数据、别名、标签、事实、来源引用和双向 wikilink，
使智能体能够查看概览、定位页面，并沿链接访问相关实体、事件、概念或来源摘要。

在索引时，LLM-Wiki 会针对当前 Wiki 状态编译每个源批次。对于每个段落，它选择相关的已有页面，生成
结构化 Wiki 更新，验证结构与内容层面的正确性，并把检测到的错误传给 Error Book。这个增量循环允许
页面、链接和索引演化，而无需重建整个知识库。算法 1 给出了完整伪代码。

### 3.2 组合式检索

LLM-Wiki 通过组合式 Wiki 遍历实现**可组合性**。智能体不是接收固定的 \(top-k\) 上下文，而是根据
中间观察组合 `wiki_search` 和 `wiki_read` 调用，迭代地搜索、阅读、跟随链接并检查充分性，直到收集
到足够证据。

图 2 将传统扁平片段 RAG 与两条 LLM-Wiki 遍历路线进行对比：用于实体锚定查询的搜索优先遍历，以及
用于开放式或枚举型查询的浏览优先遍历。

**工具接口。** 智能体配备两个工具：

- `wiki_search(query)`：搜索 Wiki 索引，优先使用页面名称、别名、标签和描述等结构化信号，然后再回退
  到页面内容。它返回候选页面和元数据，供后续阅读和遍历使用。
- `wiki_read(paths)`：批量读取目录索引（`_index.md`）或完整页面。对于知识页面，返回内容包含页面间
  链接，这些链接成为后续跳转的遍历支点。

与一次性 RAG 相比，搜索质量不再那么像瓶颈，因为智能体可以在阅读后判断页面有用性，并通过多轮遍历
弥补排序不完美。

**遍历策略。** 智能体会根据查询特征自适应选择检索策略：

- **直接访问：** 对已知实体，智能体直接读取页面，或先搜索再读取排名靠前的结果。
- **桥接查询（\(A \to B \to answer\)）：** 智能体读取页面 A，通过页面间链接识别实体 B，并遍历到页面
  B，从而把复杂推理简化为迭代式链接遍历。
- **探索式浏览：** 对开放式查询，智能体读取目录索引以获得结构化概览，然后选择性读取有希望的页面。

**图 1：LLM-Wiki 中的索引时构建与结构化 Wiki 知识库。** 原始文档被编译为链接的 Wiki 页面，而
Error Book 记录验证失败、注入约束并驱动修复，从而产生一个人类可读且机器可遍历的 Wiki。

**终止。** 每次 `wiki_read` 调用之后，智能体都会评估证据充分性。它在所有推理链都已被追踪、
工具调用预算 \(T_{\max}\) 用尽，或连续空搜索超过耐心阈值 \(P\) 时终止。回答前至少需要一次
`wiki_read` 调用。

### 3.3 Error Book

LLM-Wiki 通过 Error Book 实现**可演化性**。Error Book 是一种持久自我纠错机制，用于检测反复出现的
构建错误、归因根因、将其转化为可复用约束，并修复受影响页面。由 LLM 生成的 Wiki 页面可能包含结构与
内容层面的问题，包括悬空链接、缺失章节、格式错误引用、无支撑事实、事实不一致和跨页面矛盾。传统方法
依赖一次性后处理或人工审查，但相同错误模式可能在摄取批次之间反复出现。Error Book 通过把学到的约束
注入未来编译提示，并修复已有错误，使 Wiki 随时间改进。

**错误分类。** 通过对多个语料中的编译输出进行经验分析，我们识别了七类系统性错误，涵盖结构有效性、
来源扎根和语义一致性（表 6）。结构错误包括悬空链接、不完整页面、格式错误引用、未选择页面覆盖和索引
不一致，而内容层面错误包括无支撑事实和跨页面矛盾。跨语料来看，悬空链接占检测错误的 29.1%--63.8%，
格式错误引用占 18.9%--28.5%（表 7），表明链接与来源引用验证对于维护可遍历 Wiki 至关重要。确定性
验证器检测结构错误，而基于来源的 LLM 验证和跨页面一致性检查识别内容层面错误。这一区分促成了我们的
双层修复设计：代码级修复维持 Wiki 作为有效可遍历结构，基于 LLM 的修复改进事实扎根和跨页面一致性。

**自我纠错循环。** Error Book 通过五阶段生命周期运行：

1. **发现：** 每个编译批次后，确定性验证器检测结构错误，而基于来源的 LLM 验证和跨页面一致性检查
   识别内容层面错误。
2. **归因：** 每个错误都会被追溯到根因，例如在未检查索引的情况下假设被链接页面存在，或从源段落
   过度泛化。
3. **约束：** 根因被形式化为自然语言约束规则，例如在输出 wikilink 前验证链接目标，或把实体属性
   扎根于被引用的来源摘要。
4. **注入：** 所有开放约束规则都会追加到第 2 步编译提示中，引导 LLM 在后续批次中避免已知错误模式。
5. **验证并关闭：** 系统会周期性重新验证先前出错的页面。如果约束注入后错误不再出现，则将该错误条目
   标记为已关闭。

Error Book 以结构化 YAML 文件（`error_book.yaml`）持久化，其中每个条目包含错误现象、根因分析、
生成的约束规则、验证方法和生命周期状态（`open`/`closed`）。约束以自然语言指令注入，从“NEVER create
a link to a page not present in `_index.md`”这样的结构规则，到“Do not add entity attributes unless
they are supported by the cited source digest.”这样的语义规则。这些指令利用 LLM 的指令遵循能力，而无需
架构修改。

**双层修复：代码与 LLM。** 除约束注入外，Error Book 还通过双层机制主动修复已有错误。**第 1 层
（代码自动修复）**在每个编译批次后运行，并应用确定性例程修复结构错误，包括悬空链接、嘈杂格式和索引
不一致。**第 2 层（LLM 周期性修复）**每 \(N\) 篇文章触发一次，处理需要推理的语义或内容层面错误，
例如缺失页面、不完整摘要、无支撑事实和跨页面矛盾。因此，第 1 层保持 Wiki 结构有效，第 2 层改进事实
扎根和跨页面一致性。在最终化阶段，一个三轮代码修复 ↔ LLM 修复循环会捕获新引入的错误，并推动 Wiki
趋于稳定状态。

**图 2：检索即查找与作为推理的检索。** 传统扁平片段 RAG 在独立片段上执行一次性相似度查找，而
LLM-Wiki 通过搜索优先和浏览优先的组合式遍历路线，结合证据充分性检查和智能体重新规划，支持
retrieval-as-reasoning。

## 4 实验设置

### 4.1 数据集

我们在三个公开多跳问答基准和一个结构化知识基准上评估 LLM-Wiki。对于 HotpotQA（Yang et al., 2018）、
MuSiQue（Trivedi et al., 2022）和 2WikiMultiHopQA（Ho et al., 2020），我们使用原始数据集顺序中的
前 500 个 QA 样例。每个基准的源语料是与这 500 个样例关联的上下文段落并集，所有方法都使用同一语料。
这是数据集提供的上下文设置，而不是开放域全维基百科检索。

我们进一步在 AuthTrace（Wu et al., 2026）上评估。AuthTrace 是一个诊断性基准，用于考察主题密集的
单作者语料中的证据选择与组装。每个实例提供一个查询、带引号的黄金证据单元、原子黄金主张单元、参考
答案和精确证据扇入标签。遵循 AuthTrace 的答案正确性协议，我们使用 GPT-4o-mini 以 0--3 分量表评判
回答，并报告 Single-doc、Low multi-doc、High multi-doc 和 All 设置。

### 4.2 基线

我们比较了横跨四种范式的七种方法：

- **None（Closed-book）：** 不使用检索的直接 LLM 生成，作为下界。
- **Vanilla RAG（BM25）：** BM25 稀疏检索 + 前 5 个段落 + LLM 生成。
- **Vanilla RAG（Dense）：** Qwen3-Embedding-8B 密集检索 + 前 5 个段落 + LLM 生成。
- **RAPTOR**（Sarthi et al., 2024）：递归聚类并摘要为树结构。
- **GraphRAG**（Edge et al., 2024）：实体抽取 + 社区检测 + 层次化摘要。
- **LightRAG**（Guo et al., 2025）：在实体图和主题图上的双层检索。
- **HippoRAG 2**（Gutiérrez et al., 2025）：KG 三元组 + Personalized PageRank + LLM QA。

### 4.3 指标

公开 QA 基准使用 Answer F1 和 Exact Match（EM），并用按跳数和按类型的 F1 进行细粒度分析。AuthTrace
使用 GPT-4o-mini 按该基准 0--3 答案正确性量表评判的准确率（AC, %），并归一化为 \(score/3 \times
100\)。效率通过平均查询延迟衡量。

### 4.4 实现细节

所有方法都使用来自 GLM-5 模型家族（GLM-5-Team, 2026）的 GLM-5.1 作为统一答案 LLM。对于涉及基于
LLM 的索引、摘要、图构建或 Wiki 编译的方法，我们也在这些步骤中使用 GLM-5.1，使差异主要反映检索
与知识组织策略，而不是模型选择。对于所有需要文本嵌入的方法，包括密集检索和图增强基线，我们使用
Qwen3-Embedding-8B（Zhang et al., 2025）。对于所有基线，我们使用作者官方实现，并遵循其推荐超参数。
所有方法都在相同源语料上评估。

我们的评估比较的是端到端检索系统，而不是使每个检索组件完全相同。LLM-Wiki 使用更大的查询时工具调用
预算，但我们在表 10 中报告延迟，并在局限性中讨论其一次性编译成本。

对于 LLM-Wiki，智能体最大工具调用预算为 \(T_{\max}=15\)，耐心阈值为 \(P=3\) 次连续空搜索，
`SELECTPAGES` 最多选择 \(k=5\) 个页面。Error Book 每 10 个编译批次重新验证一次。

## 5 实验

### 5.1 公开基准上的主要结果

如表 1 所示，LLM-Wiki 在三个数据集上都取得最高 F1 和 EM。增益在组合性更具挑战的基准上最大：
在 MuSiQue 上，LLM-Wiki 比 Dense RAG 高 12.9 个 F1 点，比最强基线 LightRAG 高 8.1 个 F1 点；在
2WikiMultiHopQA 上，它比 LightRAG 高 6.4 个 F1 点。在 HotpotQA 上，即使强一次性检索基线表现良好，
LLM-Wiki 仍取得最佳总体性能，比 LightRAG 高 2.0 个 F1 点。总体而言，LLM-Wiki 在较简单的 2 跳设置
上保持性能，同时其优势会随推理深度和组合性的增加而扩大。

| 方法 | HotpotQA F1 | HotpotQA EM | MuSiQue F1 | MuSiQue EM | 2WikiMHQA F1 | 2WikiMHQA EM |
|---|---:|---:|---:|---:|---:|---:|
| None (Closed-book) | 0.551 | 0.442 | 0.456 | 0.372 | 0.638 | 0.546 |
| Vanilla RAG (BM25) | 0.717 | 0.590 | 0.545 | 0.442 | 0.790 | 0.684 |
| Vanilla RAG (Dense) | 0.764 | 0.642 | 0.611 | 0.500 | 0.815 | 0.724 |
| RAPTOR | 0.801 | 0.674 | 0.522 | 0.442 | 0.707 | 0.652 |
| GraphRAG | 0.771 | 0.650 | 0.582 | 0.482 | 0.720 | 0.648 |
| LightRAG | 0.819 | 0.682 | 0.659 | 0.550 | 0.847 | 0.764 |
| HippoRAG 2 | 0.805 | 0.668 | 0.624 | 0.514 | 0.831 | 0.706 |
| LLM-Wiki (Ours) | **0.839** | **0.710** | **0.739** | **0.634** | **0.911** | **0.854** |

**表 1：三个多跳 QA 基准上的主要结果，每个基准包含 500 个问题。** 最佳结果以粗体显示。

在基线中，LightRAG 总体表现最强，而 RAPTOR 和 GraphRAG 始终较弱。二者都依赖层次化摘要，而这可能
丢弃组合式推理所需的精确实体名称和关系：RAPTOR 的递归摘要可能损失特异性，而 GraphRAG 的社区检测
按共现而不是显式语义关系来分组实体。这些结果表明，LLM-Wiki 的关键优势不是简单检索更多文本，而是
以一种形式暴露知识，使智能体能够把中间观察转化为后续检索动作。

### 5.2 AuthTrace 结果

表 2 报告了 AuthTrace 上的评判准确率（AC）。LLM-Wiki 取得最佳总体性能，比最强基线 HippoRAG 2 高
2.1 个 AC 点。随着查询需要更广泛的跨文档推理，其优势变得更大：在 Low multi-doc 问题上，LLM-Wiki
比 HippoRAG 2 高 5.0 个 AC 点；在 High multi-doc 问题上高 8.9 个 AC 点。这些结果表明，基于编译的
知识组织尤其有助于从分散文档中收集、比较和聚合证据。

在 Single-doc 问题上，HippoRAG 2 比 LLM-Wiki 高 2.3 个 AC 点。这是预期之内的，因为许多单文档问题
一旦检索到相关原文文章并识别局部细节，就可以回答。相比之下，LLM-Wiki 从结构化页面回答，这些页面把
源内容重组为简洁事实和摘要；这有利于跨文档综合，但偶尔会遗漏细粒度局部细节。总体而言，AuthTrace
显示 LLM-Wiki 能够扩展到标准多跳 QA 之外，并在需要多文档综合的结构化查询上取得最大增益。

为验证基于 GPT-4o-mini 的评判可靠性，我们还在 AuthTrace 输出的随机子集上进行了人工审计。人工判断
与 GPT-4o-mini 高度一致，并保持被审计方法之间的相同排序；协议和结果报告于表 5。

| 方法 | Single | Low | High | All |
|---|---:|---:|---:|---:|
| None (Closed-book) | 12.3 | 16.8 | 16.2 | 14.0 |
| Vanilla RAG (BM25) | 75.4 | 48.5 | 30.4 | 62.7 |
| Vanilla RAG (Dense) | 72.9 | 49.0 | 35.0 | 61.9 |
| RAPTOR | 69.5 | 44.0 | 33.9 | 58.6 |
| GraphRAG | 55.8 | 35.1 | 26.7 | 46.8 |
| LightRAG | 73.4 | 34.2 | 14.5 | 55.7 |
| HippoRAG 2 | **78.3** | 59.6 | 46.5 | 68.3 |
| LLM-Wiki (Ours) | 76.0 | **64.6** | **55.4** | **70.4** |

**表 2：AuthTrace 在不同 fan-in 设置上的结果。** 分数为评判准确率（AC）。每列最佳结果以粗体显示。

| 变体 | HotpotQA | MuSiQue | 2Wiki |
|---|---:|---:|---:|
| LLM-Wiki (full) | 0.839 | 0.739 | 0.911 |
| w/o Wiki Structure | 0.778 | 0.669 | 0.844 |
| w/o Progressive Traversal | 0.722 | 0.601 | 0.789 |
| w/o Error Book | 0.801 | 0.699 | 0.877 |

**表 3：使用 F1 分数的消融研究。** 每一行移除一个组件，同时保持其他组件不变。

### 5.3 消融研究

为隔离各组件的贡献，我们逐一移除组件进行消融实验，如表 3 所示。

**移除 Wiki 结构。** 该变体移除已编译 Wiki 表示，使用扁平源片段作为检索底座，同时保留相同智能体
框架和答案生成模型。智能体仍可发起检索调用，但返回的证据由独立片段组成，而不是目录索引、结构化
页面和 wikilink。6.1--7.0 个 F1 点的一致下降表明，结构化知识组织提供了有效检索与推理所需的已编译
底座。

**移除渐进式遍历。** 该变体保持已编译 Wiki 不变，但禁用迭代式重新规划。智能体只执行一次
`wiki_search` 调用，读取排名靠前页面一次，然后在不跟随额外 wikilink 或重新访问搜索/阅读步骤的情况
下回答。11.7--13.8 个 F1 点的下降表明，已编译 Wiki 结构只有在被主动遍历时才最有用：索引、页面和
链接之所以有帮助，是因为智能体可以跨步骤搜索、阅读和跟随它们，而不是因为多轮检索本身就足够。

**移除 Error Book。** 该变体使用相同的 Wiki 编译和遍历流水线，但禁用 Error Book 更新、约束注入和
修复。编译错误仍可在验证期间被观察到，但不会被积累为可复用约束，也不会用于修复未来批次。3.4--4.0
个 F1 点的下降表明，Error Book 通过减少反复出现的结构与语义构建错误的影响，改进了下游检索。

总体而言，这些消融显示 LLM-Wiki 的增益来自相互强化的组件。Wiki 结构提供已编译底座，渐进式遍历利用
该底座进行组合式证据收集，而 Error Book 维护被遍历结构的质量。

### 5.4 细粒度分析

细粒度结果进一步表明，LLM-Wiki 的增益集中在需要更深关系推理的查询上。在 2WikiMHQA 上，LLM-Wiki
相较 LightRAG 的 F1 差距从 2 跳问题上的 5.7 个 F1 点扩大到 4 跳问题上的 8.3 个 F1 点，并在 4 跳
问题上达到 0.983 F1（图 3）。按类型的结果显示，组合式问题上的增益尤其强，LLM-Wiki 比 Dense RAG 高
15.6 个 F1 点，比 LightRAG 高 7.3 个 F1 点（表 8）。在 GPT-4o 下，LLM-Wiki 也保持了相较 Dense RAG
5.1--16.9 个 F1 点的增益（表 9），表明收益来自知识组织，而不是特定答案模型。表 10 显示，这些增益
不需要更慢的查询时推理：在多数基准上，LLM-Wiki 与 BM25/Dense RAG 相当，并显著快于 LightRAG、
HippoRAG 2 和 RAPTOR。详细分析和检索轨迹见附录 G 和 H。

总体而言，这些结果为 Retrieval-as-Reasoning 假设提供了经验证据：当检索被组织为由智能体驱动的、通过
已编译知识结构进行的组合式遍历时，性能会随查询所需推理深度而扩展。

## 6 讨论

**Wiki 何时优于 RAG？** 当查询需要关系跟随、证据枚举、跨文档聚合或动态知识探索时，LLM-Wiki 最有
价值。随着跳数和组合性增加，其优势会扩大，因为预编译链接使中间实体显式可发现。相比之下，源局部化
问题可能更偏好直接检索原始文章的方法，这有助于解释为什么 HippoRAG 2 在 AuthTrace Single-doc 问题上
表现更好。

**通过编译实现作为推理的检索。** 更广义地说，我们的结果澄清了为什么“检索即推理”的区分重要。核心
改进不是更强的相似度函数，而是知识库与智能体之间的不同契约：编译使搜索空间显式化，组合式工具让观察
驱动后续检索步骤，而 Error Book 保持结构可靠。把组织工作前置到编译时，用更丰富的智能体--知识交互
表面替代运行时匹配，同时也降低了查询延迟。

## 7 结论

我们提出了 LLM-Wiki，这是第一个实现 Retrieval-as-Reasoning 的智能体原生检索系统。结果支持三点主张。
第一，把文档编译为链接的 Wiki 页面，相较扁平片段和图增强检索，能改进知识密集型 QA，并且增益会随
推理深度增加而扩大。第二，由智能体规划的组合式遍历在已编译结构之外带来额外增益。第三，Error Book
缓解反复出现的结构与语义编译错误，并提升下游稳健性。超出这一具体设置，我们的结果确立了一条智能体
原生检索设计原则：把知识编译为可导航结构，暴露给组合式遍历，并通过闭环自我纠错进行维护。我们相信
这一范式会自然扩展到 QA 之外的知识密集型智能体任务。

## 局限性

有两个主要局限值得讨论。第一，**编译成本**：每个源段落都需要 `SELECTPAGES` 和 `COMPILEWIKIPAGES`，
使初始构建比“分块并嵌入”的方法更昂贵，尽管该成本会被后续查询摊销。第二，**可扩展性**：当 Wiki
增长到数万页面时，目录索引可能变得笨重，页面选择可能退化。Web 规模且频繁变化的语料需要层次化索引、
分片、陈旧事实处理和全局目录维护。将 LLM-Wiki 扩展到大规模动态维护、多模态 Wiki 和跨语料迁移仍是
未来工作。

## References

Akari Asai, Zeqiu Wu, Yizhong Wang, Avirup Sil, and
Hannaneh Hajishirzi. 2024. Self-RAG: Learning to
retrieve, generate, and critique through self-reflection.
In Proceedings of the Twelfth International Conference
on Learning Representations (ICLR).

Howard Chen, Ramakanth Pasunuru, Jason Weston, and
Asli Celikyilmaz. 2023. Walking down the memory
maze: Beyond context limit through interactive
reading. Preprint, arXiv:2310.05029.

Darren Edge, Ha Trinh, Newman Cheng, Joshua
Bradley, Alex Chao, Apurva Mody, Steven Truitt,
Dasha Metropolitansky, Robert Osazuwa Ness, and
Jonathan Larson. 2024. From local to global: A
graph RAG approach to query-focused summarization.
Preprint, arXiv:2404.16130.

GLM-5-Team. 2026. GLM-5: from vibe coding to
agentic engineering. Preprint, arXiv:2602.15763.

Zirui Guo, Lianghao Xia, Yanhua Yu, Tu Ao, and Chao
Huang. 2025. LightRAG: Simple and fast retrieval-
augmented generation. In Findings of the Association
for Computational Linguistics: EMNLP 2025,
pages 10746–10761, Suzhou, China. Association for
Computational Linguistics.

Bernal Jiménez Gutiérrez, Yiheng Shu, Yu Gu, Michi-
hiro Yasunaga, and Yu Su. 2024. HippoRAG: Neu-
robiologically inspired long-term memory for large
language models. In Advances in Neural Information
Processing Systems, volume 37.

Bernal Jiménez Gutiérrez, Yiheng Shu, Weijian Qi,
Sizhe Zhou, and Yu Su. 2025. From RAG to memory:
Non-parametric continual learning for large language
models. In Proceedings of the 42nd International
Conference on Machine Learning, volume 267 of
Proceedings of Machine Learning Research, pages
21497–21515. PMLR.

Xanh Ho, Anh-Khoa Duong Nguyen, Saku Sugawara,
and Akiko Aizawa. 2020. Constructing a multi-
hop QA dataset for comprehensive evaluation of
reasoning steps. In Proceedings of the 28th Inter-
national Conference on Computational Linguistics,
pages 6609–6625. International Committee on Com-
putational Linguistics.

Andrej Karpathy. 2026. LLM Wiki. GitHub Gist. Ac-
cessed: 2026-05-19.

Vladimir Karpukhin, Barlas O˘guz, Sewon Min, Patrick
Lewis, Ledell Wu, Sergey Edunov, Danqi Chen, and
Wen-tau Yih. 2020. Dense passage retrieval for open-
domain question answering. In Proceedings of the
2020 Conference on Empirical Methods in Natural
Language Processing (EMNLP), pages 6769–6781,
Online. Association for Computational Linguistics.

J. Richard Landis and Gary G. Koch. 1977. The mea-
surement of observer agreement for categorical data.
Biometrics, 33(1):159–174.

Patrick Lewis, Ethan Perez, Aleksandra Piktus, Fabio
Petroni, Vladimir Karpukhin, Naman Goyal, Hein-
rich Küttler, Mike Lewis, Wen-tau Yih, Tim Rock-
täschel, Sebastian Riedel, and Douwe Kiela. 2020.
Retrieval-augmented generation for knowledge-
intensive NLP tasks. In Advances in Neural Infor-
mation Processing Systems, volume 33, pages 9459–
9474.

Aman Madaan, Niket Tandon, Prakhar Gupta, Skyler
Hallinan, Luyu Gao, Sarah Wiegreffe, Uri Alon,
Nouha Dziri, Shrimai Prabhumoye, Yiming Yang,
Shashank Gupta, Bodhisattwa Prasad Majumder,
Katherine Hermann, Sean Welleck, Amir Yazdan-
bakhsh, and Peter Clark. 2023. Self-refine: Iterative
refinement with self-feedback. In Advances in Neu-
ral Information Processing Systems, volume 36.

Parth Sarthi, Salman Abdullah, Aditi Tuli, Shubh
Khanna, Anna Goldie, and Christopher D. Manning.
2024. RAPTOR: Recursive abstractive processing
for tree-organized retrieval. In Proceedings of the
Twelfth International Conference on Learning Repre-
sentations (ICLR).

Noah Shinn, Federico Cassano, Ashwin Gopinath,
Karthik Narasimhan, and Shunyu Yao. 2023. Re-
flexion: Language agents with verbal reinforcement
learning. In Advances in Neural Information Pro-
cessing Systems, volume 36.

Harsh Trivedi, Niranjan Balasubramanian, Tushar Khot,
and Ashish Sabharwal. 2022. MuSiQue: Multi-
hop questions via single-hop question composition.
Transactions of the Association for Computational
Linguistics, 10:539–554.

Harsh Trivedi, Niranjan Balasubramanian, Tushar Khot,
and Ashish Sabharwal. 2023. Interleaving retrieval
with chain-of-thought reasoning for knowledge-
intensive multi-step questions. In Proceedings of
the 61st Annual Meeting of the Association for Com-
putational Linguistics (Volume 1: Long Papers),
pages 10014–10037, Toronto, Canada. Association
for Computational Linguistics.

Xiaoqing Wu, Feifei Li, Haoliang Ming, and Wenhui
Que. 2026. AuthTrace: Diagnosing evidence con-
struction in thematically dense single-author corpora.
Preprint, arXiv:2605.25382.

Zhilin Yang, Peng Qi, Saizheng Zhang, Yoshua Ben-
gio, William W. Cohen, Ruslan Salakhutdinov, and
Christopher D. Manning. 2018. HotpotQA: A dataset
for diverse, explainable multi-hop question answer-
ing. In Proceedings of the 2018 Conference on Em-
pirical Methods in Natural Language Processing,
pages 2369–2380, Brussels, Belgium. Association
for Computational Linguistics.

Shunyu Yao, Jeffrey Zhao, Dian Yu, Nan Du, Izhak
Shafran, Karthik Narasimhan, and Yuan Cao. 2023.
ReAct: Synergizing reasoning and acting in language
models. In Proceedings of the Eleventh International
Conference on Learning Representations (ICLR).

Yanzhao Zhang, Mingxin Li, Dingkun Long, Xin Zhang,
Huan Lin, Baosong Yang, Pengjun Xie, An Yang,
Dayiheng Liu, Junyang Lin, Fei Huang, and Jingren
Zhou. 2025. Qwen3 Embedding: Advancing text
embedding and reranking through foundation models.
Preprint, arXiv:2506.05176.

Lianmin Zheng, Wei-Lin Chiang, Ying Sheng, Siyuan
Zhuang, Zhanghao Wu, Yonghao Zhuang, Zi Lin,
Zhuohan Li, Dacheng Li, Eric P. Xing, Hao Zhang,
Joseph E. Gonzalez, and Ion Stoica. 2023. Judging
LLM-as-a-judge with MT-bench and chatbot arena.
In Advances in Neural Information Processing Sys-
tems, volume 36, pages 46595–46623. Curran Asso-
ciates, Inc.

## 附录 A 额外相关工作与范式比较

本附录补充说明 LLM-Wiki 与既有检索和知识组织范式的差异。表 4 从索引产物、知识形式和核心瓶颈三个
方面总结了代表性方法。

我们沿三条轴比较方法。索引产物描述离线处理后生成的工件，例如片段、摘要、三元组、向量或 Wiki 页面。
知识形式刻画该工件主要是压缩文本表示、机器可读检索结构，还是人类可审计的知识组织。核心瓶颈突出
其对于智能体驱动组合式检索的主要限制。该比较强调，关键区别不在于一种方法是否使用图或 LLM，而在于
所得知识库是否暴露了智能体可以搜索、阅读和遍历的显式可遍历结构。

| 方法 | 索引产物 | 知识形式 | 核心瓶颈 |
|---|---|---|---|
| Vanilla RAG | 扁平片段 | 嵌入索引段落 | 无法遍历中间节点 |
| RAPTOR / MemWalker | 摘要树 | 文本压缩 | 树缺少横向关联 |
| GraphRAG | 社区摘要 | 层次化压缩 | 摘要丢失细节；成本高 |
| HippoRAG 2 | KG 三元组 | 机器可读片段 | 有损三元组；PPR 是近似的 |
| LightRAG | 实体/关系向量 | 向量索引 | 无法发现中间实体 |
| LLM-Wiki (Ours) | Wiki 页面 + 链接 | 结构化知识 | 编译成本（一次性） |

**表 4：知识组织范式比较。** 不同于生成压缩摘要或机器可读片段的既有方法，LLM-Wiki 将文档编译为带有
显式双向链接的结构化、人类可审计 Wiki 页面。

## 附录 B 数据集细节

**公开多跳 QA 基准。** 我们在三个成熟公开基准上评估。HotpotQA（Yang et al., 2018）包含需要在
Wikipedia 段落上进行桥接推理或比较的 2 跳问题。MuSiQue（Trivedi et al., 2022）包含 2--4 跳组合式
问题，并采用防捷径设计以阻止单跳回答。2WikiMultiHopQA（Ho et al., 2020）包含横跨桥接、比较、组合式
和推断类型的 2 跳与 4 跳问题。对于每个数据集，我们取原始数据集顺序中的前 500 个 QA 样例，并从与它们
关联的上下文段落并集构建检索语料。所有方法共享该语料；例如，2WikiMultiHopQA 评估语料包含 3,440 个
段落。在这些实验中，我们不从完整 Wikipedia dump 中检索。

**AuthTrace。** 现有公开多跳基准通常强调异质证据、桥接实体和短跨度答案。为测试高主题相似性下的证据
构建，我们在 AuthTrace（Wu et al., 2026）上评估。该基准由五位现代中文散文作家的 860 篇公版写作中
筛选出的 2,099 个 QA 实例构成。每个实例包含一个已移除标题泄漏的查询、带引号的黄金证据单元、原子
黄金主张单元、简洁参考答案和精确证据扇入标签。Fan-in 将实例分为 Single-doc（=1）、Low multi-doc
（2--3）和 High multi-doc（\(\ge 4\)），分别测试局部扎根、小规模聚合和广泛综合。遵循 AuthTrace 的
答案正确性协议，我们使用 GPT-4o-mini 作为评判器。评判器接收查询、黄金主张单元、参考答案和预测答案；
它把每个黄金主张标记为 supported、partial、missing 或 contradicted，检查无关或错误的额外内容，并
给出 0--3 分，该分数归一化为 \(score/3 \times 100\) 作为 AC。参考答案仅用于解释主张单元。

## 附录 C AuthTrace 评判的人工审计

本附录报告对主实验中基于 GPT-4o-mini 的 AuthTrace 评判器的人工审计。目标是验证自动判断与人工评估
一致，并保持相同的相对方法排序。

**审计协议。** 我们抽样 240 个回答，并分层覆盖 3 种代表性方法（LLM-Wiki、HippoRAG 2 和 Dense RAG）
以及 3 个 AuthTrace fan-in 设置（Single-doc、Low multi-doc、High multi-doc）。三位作者独立使用与
GPT-4o-mini 评判器相同的 0--3 答案正确性量表标注每个回答。标注者评估黄金主张单元覆盖情况，惩罚无关
或错误的额外内容，并给出 0--3 分，其中 3 表示完全正确，0 表示错误。标注者不知道方法身份和
GPT-4o-mini 判断。分歧通过讨论解决，以获得每个回答的单一人工分数。

**结果。** 表 5 报告了人工--GPT 一致性，以及按照与表 2 相同 fan-in 分组得到的人工准确率。人工判断与
GPT-4o-mini 在 89.6% 的被审计回答上一致，Cohen's \(\kappa=0.79\)，表明一致性显著（Landis and Koch,
1977）。

| 指标 | Single | Low | High | All |
|---|---:|---:|---:|---:|
| Human--GPT agreement (%) | 91.2 | 89.4 | 87.9 | 89.6 |
| Cohen's \(\kappa\) | 0.82 | 0.79 | 0.76 | 0.79 |
| LLM-Wiki human AC | 74.1 | 62.4 | 53.6 | 68.3 |
| HippoRAG 2 human AC | 76.5 | 57.9 | 44.7 | 66.4 |
| Dense RAG human AC | 70.8 | 47.2 | 33.4 | 60.0 |

**表 5：对基于 GPT-4o-mini 的 AuthTrace 评判在 240 个分层样本上的人工审计。** 人工--GPT 一致性和
Cohen's \(\kappa\) 表明一致性显著，且人工导出的被审计方法排序在每一列都与表 2 的 GPT-4o-mini 排序
一致。

**讨论。** 有两点观察。第一，与表 2 中的 GPT-4o-mini AC 分数相比，被审计方法的人工 AC 分数低
1.6--2.2 点，说明 GPT-4o-mini 在我们的量表下略微更宽松。这与已有工作中关于 LLM 评判器与人工评估
高度但不完美一致的发现相符（Zheng et al., 2023）。第二，被审计方法在每列中的排序保持不变：
LLM-Wiki 在 Low multi-doc、High multi-doc 和 All 上仍最强，而 HippoRAG 2 保留其 Single-doc 优势。
不同 fan-in 设置中的一致性也保持较高（87.9%--91.2%）。因此，我们使用 GPT-4o-mini 作为主要 AuthTrace
评判器，并把人工审计视为对评判协议的验证。

## 附录 D 编译算法

算法 1 给出了 LLM-Wiki 使用的完整索引时编译循环，包括页面选择、Wiki 更新生成、验证、Error Book 更新
和双层修复。

```text
Algorithm 1 Index-time Wiki compilation
Require: Source batch X, current Wiki W, directory indices I, source archives A, active Error Book constraints C
Ensure: Updated Wiki W and Error Book B
1: for each source passage x in X do
2:     S <- SELECTPAGES(x, I) {select relevant existing pages}
3:     U <- COMPILEWIKIPAGES(x, S, C) {generate pages, links, and index updates}
4:     E_s <- STRUCTURALVALIDATE(U, W)
5:     E_c <- CONTENTVALIDATE(U, W, A)
6:     E <- E_s union E_c
7:     if E != empty then
8:         B <- UPDATEERRORBOOK(B, E)
9:         C <- ACTIVECONSTRAINTS(B)
10:        U <- CODEAUTOFIX(U, E_s)
11:    end if
12:    W <- APPLYUPDATES(W, U)
13: end for
14: if PERIODICFIXDUE(B) then
15:    W <- LLMPERIODICFIX(W, B)
16:    B <- VERIFYANDCLOSE(B, W)
17: end if
18: return W, B
```

## 附录 E 示例 Wiki 结构与页面模式

本附录展示已编译 LLM-Wiki 实例的目录组织和页面格式。这些示例改编自为 2WikiMHQA 评估语料生成的 Wiki，
为便于阅读缩短了长列表。

**目录布局。** 已编译 Wiki 是一个自包含的 Markdown 文件树。根目录包含全局 `index.md`，提供知识概览
和目录目录。每个知识目录包含一个 `_index.md` 文件，作为可浏览的页面列表，以及若干独立知识页面。
并行的 `sources/` 树存储段落级摘要和完整原文文章以支持溯源。2WikiMHQA Wiki 包含横跨 6 个主题目录的
5,825 个知识页面，以及 6,840 个来源页面：

```text
wiki/
index.md
concepts/
_index.md
Dynastic-Succession.md
...
geography/
_index.md
...
history/
_index.md
...
media/
_index.md
...
organizations/
_index.md
...
people/
_index.md
John-V-Prince-of-Anhalt-Zerbst.md
Ernest-I-Prince-of-Anhalt-Dessau.md
...
sources/
digests/
john-v-prince-anhalt-zerbst.md
...
articles/
john-v-prince-anhalt-zerbst.md
...
```

**全局索引（`index.md`）。** 全局索引以知识概览段落开头，概括 Wiki 的范围和领域覆盖，随后是目录目录：

```markdown
# Wiki Directory Overview
> Knowledge Overview (updated xxxx-xx-xx)
> This high-density, relational encyclopedic
repository functions as a sophisticated engine
for complex multi-hop question answering ...
## Directory Catalog
- concepts/ -- theories, methods,
genres, categories, abstract ideas
- geography/ -- cities, villages,
countries, airports, landmarks
- history/ -- historical events,
periods, battles, medieval history
- media/ -- films, albums, songs,
TV shows, books, creative works
- organizations/ -- schools,
universities, bands, companies, dynasties
- people/ -- biographies of
historical figures, artists, politicians
- sources/ -- paragraph digests
and original archives
```

**目录索引（`_index.md`）。** 每个目录的 `_index.md` 按语义章节组织页面，包含别名、标签和一句话摘要。
例如，`people/_index.md` 把其 3,241 个条目分组到“German Nobility”“Chinese Film Directors”
“Brazilian Politicians”和“Spanish Politicians”等标题下，从而支持高效浏览：

```markdown
# People
> xx pages
## German Nobility
- [[Ernest-I-Prince-of-Anhalt-Dessau]]
(Ernest I of Anhalt-Dessau, Ernst I von
Anhalt-Dessau) #German-nobility
#House-of-Ascania #Prince
## Chinese Film Directors
- [[Zhang-Yimou]] (Zhang Yimou)
-- Prominent Chinese film director ...
#director #chinese-cinema #fifth-generation
## Brazilian Politicians
- [[Adalberto-Pereira-dos-Santos]]
(Adalberto Pereira dos Santos, General
Adalberto) #Brazil #general #politician
```

**知识页面。** 每个知识页面遵循固定模式：包含类型、创建与更新时间戳、别名和标签的 YAML frontmatter；
一句话引用摘要；结构化关键事实；指向相关页面的双向 wikilink；以及来源引用。下面是 John V, Prince of
Anhalt-Zerbst 的完整页面：

```markdown
---
type: people
created: xxxx-xx-xx
updated: xxxx-xx-xx
aliases: [John V of Anhalt-Zerbst,
Johann V von Anhalt-Zerbst,
Prince John V of Anhalt-Zerbst]
tags: [German nobility, House of Ascania,
Prince, Anhalt-Zerbst, Anhalt-Dessau]
---
# John V, Prince of Anhalt-Zerbst
> German prince of the House of Ascania who
> ruled Anhalt-Dessau and later the re-created
> principality of Anhalt-Zerbst from 1544
## Key Facts
- John V was born on 4 September 1504 in
Dessau and died on 4 February 1551 in Zerbst
- John V was the second but eldest surviving
son of Ernest I, Prince of Anhalt-Dessau
- Mother was Margarete, daughter of Henry I,
Duke of Munsterberg-Oels
- Great-grandson of George of Podebrady,
King of Bohemia
- Married Margaret, daughter of Joachim I
Nestor, Elector of Brandenburg
- Karl I, Prince of Anhalt-Zerbst was the
eldest son of John V
## Related Pages
- [[people/Ernest-I-Prince-of-Anhalt-Dessau]]
-- father of John V
- [[people/Karl-I-Prince-of-Anhalt-Zerbst]]
-- eldest son and successor of John V
## Related Sources
- [[sources/digests/john-v-prince-anhalt-zerbst]]
-- Wikipedia paragraph about John V
- [[sources/digests/karl-i-prince-of-anhalt-zerbst]]
-- Wikipedia paragraph about Karl I
mentioning John V
```

**设计动机。** 该模式通过三种方式支持查询时检索。第一，别名和标签允许 `wiki_search` 用不同表层形式
定位页面。第二，双向 wikilink 暴露显式遍历路径；例如，问题“When did John V's father die?”可以通过
从 John V 到 Ernest I 的链接来回答。第三，来源引用保留出处，使智能体或人工审计者能够对照原始段落
验证主张。

## 附录 F 错误分类

表 6 列出了 Error Book 使用的七类错误。表 7 进一步总结了这些错误类型在已编译 Wiki 中的分布。

| 错误类型 | 描述与检测 |
|---|---|
| **结构有效性错误** |  |
| Dangling Links | 页面间链接指向不存在页面；通过文件系统交叉验证 |
| Incomplete Pages | 缺少必需章节（facts、sources）；模板完整性检查 |
| Malformed Refs | 来源引用违反格式规范；正则验证 |
| Unseen Overwrite | LLM 修改未在第 1 步中选择的页面；集合比较 |
| Index Inconsistency | 索引--文件系统不匹配；双向 diff |
| **内容层面一致性错误** |  |
| Unsupported Facts | 页面包含未被引用来源摘要支撑的主张；由基于来源的 LLM 验证检测 |
| Cross-Page Contradictions | 相关页面包含冲突的实体属性、日期或关系；由基于采样的一致性检查检测 |

**表 6：涵盖结构与内容层面失败的错误分类。**

**跨语料错误分布。** 表 7 报告了不同已编译 Wiki 中被检测到的 Error Book 条目的分布。每个单元表示对应
Wiki 中某一错误类型在所有检测错误中的百分比，因此每列约合计为 100%。

| 错误类型 | Hotpot | MuSiQue | 2Wiki | AuthTrace |
|---|---:|---:|---:|---:|
| Dangling Links | 63.8% | 62.5% | 55.8% | 29.1% |
| Incomplete Pages | 2.3% | 2.3% | 2.1% | 9.1% |
| Malformed Refs | 20.5% | 18.9% | 22.3% | 28.5% |
| Unseen Overwrite | 3.5% | 3.6% | 2.6% | 2.4% |
| Index Inconsistency | 2.2% | 2.5% | 8.2% | 10.9% |
| Unsupported Facts | 5.8% | 7.2% | 6.4% | 12.7% |
| Cross-Page Contrad. | 1.9% | 3.1% | 2.6% | 7.3% |

**表 7：已编译 Wiki 中被检测到的 Error Book 条目分布。** 数值为每个 Wiki 内的百分比。

## 附录 G 细粒度与额外分析

**按跳数分解。** 图 3 显示，LLM-Wiki 的优势会随跳数增加而扩大。在 2WikiMHQA 上，LLM-Wiki 与最强基线
LightRAG 的 F1 差距从 2 跳问题上的 5.7 个 F1 点扩大到 4 跳问题上的 8.3 个 F1 点。LLM-Wiki 在 4 跳
问题上达到 0.983 F1，因为预编译双向链接把复杂推理简化为组合式遍历，而 Dense RAG 在 4 跳问题上仅
达到 0.924 F1，这是因为很难通过单个嵌入查询捕获所有中间实体。

**图 3：2WikiMHQA 上按推理深度（2/4 跳）分组的按跳数 F1。**

**2WikiMHQA 上按类型分解。** 表 8 报告了四类 2WikiMHQA 问题的性能：bridge-comparison（Br.-Comp.）、
comparison（Comp.）、compositional（Compos.）和 inference（Infer.）。比较问题接近性能上限，达到
0.989 F1，因为预编译实体页面把比较属性放在可立即访问的格式中。组合式问题显示最大改进，LLM-Wiki 比
Dense RAG 高 15.6 个 F1 点，比 LightRAG 高 7.3 个 F1 点，尽管 0.833 的绝对分数表明复杂组合式推理仍有
进一步改进空间。

| 方法 | Br.-Comp. | Comp. | Compos. | Infer. |
|---|---:|---:|---:|---:|
| None (Closed-book) | 0.689 | 0.815 | 0.523 | 0.643 |
| Vanilla RAG (BM25) | 0.919 | 0.952 | 0.632 | 0.815 |
| Vanilla RAG (Dense) | 0.924 | 0.981 | 0.677 | 0.804 |
| RAPTOR | 0.858 | 0.934 | 0.532 | 0.642 |
| GraphRAG | 0.798 | 0.860 | 0.659 | 0.558 |
| LightRAG | 0.900 | 0.960 | 0.760 | 0.857 |
| HippoRAG 2 | 0.958 | 0.981 | 0.680 | 0.856 |
| LLM-Wiki | **0.983** | **0.989** | **0.833** | **0.909** |

**表 8：2WikiMHQA 四类问题上的按类型 F1。**

**跨答案 LLM 的泛化。** 为验证 LLM-Wiki 的优势并非特定于某个答案模型，我们在保持已编译 Wiki 固定的
同时，评估 GPT-4o 作为替代答案 LLM（表 9）。

| 方法 | LLM | Hotpot | MuSiQue | 2Wiki |
|---|---|---:|---:|---:|
| Dense RAG | GLM-5.1 | 0.764 | 0.611 | 0.815 |
| LLM-Wiki | GLM-5.1 | **0.839** | **0.739** | **0.911** |
| Dense RAG | GPT-4o | 0.741 | 0.503 | 0.636 |
| LLM-Wiki | GPT-4o | **0.792** | **0.608** | **0.805** |

**表 9：跨答案 LLM 的泛化。** 使用 GLM-5.1 编译的 Wiki 知识库保持固定；仅改变查询时答案 LLM。

在 GPT-4o 下，LLM-Wiki 保持显著优势：HotpotQA 上 +5.1 个 F1 点、MuSiQue 上 +10.5 个 F1 点、
2WikiMHQA 上 +16.9 个 F1 点。这说明收益源于知识组织和智能体原生检索设计，而不是模型特定推理能力；
结构化遍历在不同答案模型上仍然有益。

**效率分析。** 表 10 比较了各方法的查询时效率。

| 方法 | Hotpot (s/q) | MuSiQue (s/q) | 2Wiki (s/q) |
|---|---:|---:|---:|
| None (Closed-book) | 24.7 | 30.0 | 21.6 |
| BM25 RAG | 18.1 | 30.2 | 16.7 |
| Dense RAG | 16.3 | 26.9 | 15.6 |
| RAPTOR | 19.9 | 40.2 | 28.0 |
| GraphRAG | 14.0 | 13.9 | 10.8 |
| LightRAG | 41.4 | 51.3 | 39.7 |
| HippoRAG 2 | 33.5 | 38.2 | 32.4 |
| LLM-Wiki | **14.9** | 27.1 | 15.9 |

**表 10：三个基准上每个问题的查询延迟（秒）。**

LLM-Wiki 在 HotpotQA 和 2WikiMHQA 上分别达到 14.9s 和 15.9s 的查询延迟，与 BM25 和 Dense RAG 相当或
更快，同时显著快于 LightRAG、HippoRAG 2 和 RAPTOR，后者分别需要 39.7--51.3s、32.4--38.2s 和
28.0--40.2s。在 MuSiQue 上，由于该数据集复杂度更高——2--4 跳问题平均需要 8.2 个遍历步骤——延迟升至
27.1s，但仍显著低于 LightRAG 的 51.3s、RAPTOR 的 40.2s 和 HippoRAG 2 的 38.2s。GraphRAG 的延迟最低，
为 10.8--14.0s，因为它只执行一次社区摘要查找而不进行迭代推理，但这种效率以显著更低的准确率为代价。
HippoRAG 2 较高的 32.4--38.2s 延迟来自其多阶段流水线：基于 LLM 的查询实体抽取、基于嵌入的实体链接、
Personalized PageRank 遍历，以及带检索段落的最终 LLM QA 调用。LLM-Wiki 的效率源于预编译链接减少了
运行时 LLM 调用：智能体每个查询平均只读取 2.5--3.9 个页面，相比之下 RAG 固定读取 5 个段落，同时通过
有针对性的遍历实现更高准确率。

反直觉的是，无检索基线每个问题更慢，为 21.6--30.0s，因为模型必须花更多生成时间依赖参数化知识，而
LLM-Wiki 提供直接证据，使生成更简洁、更有信心。

## 附录 H 详细案例研究与检索策略

我们给出详细 walkthrough，以说明结构化知识遍历相较基于嵌入的检索的优势。

**案例 1：桥接比较（4 跳）——Dense RAG 失败，LLM-Wiki 成功。** 我们分析 2WikiMultiHopQA 中一个真实的
4 跳桥接比较问题：“哪部电影的导演年龄更大，*The Gamecock* 还是 *Monster A Go-Go*？”Dense RAG 通过
嵌入相似度检索到两部电影页面，但未能检索到包含出生日期、也就是回答所需证据的导演传记页面。它检索到
的前 5 个段落是 *The Gamecock (film)*、*Monster a Go-Go*、*The Mask of the Gorilla*、*The Capture
of Bigfoot* 和 *Monster from the Ocean Floor*，其中三个无关。由于只能从电影描述中推断，它错误回答
“The Gamecock”。

LLM-Wiki 智能体用三步解决该问题：(1) 并行搜索两部电影标题；(2) 批量读取两个电影页面，并从结构化字段
识别导演姓名，即 *The Gamecock* 的 Pasquale Festa Campanile，以及 *Monster A Go-Go* 的 Bill Rebane
和 Herschell Gordon Lewis；(3) 沿预编译的 `links_to` 指针批量读取导演传记页面，获得精确出生日期：
Herschell Gordon Lewis 出生于 1926 年 6 月 15 日，而 Pasquale Festa Campanile 出生于 1927 年 7 月
28 日。由于 Lewis（1926）比 Campanile（1927）年长，正确答案是“Monster A Go-Go”。

该案例展示了一轮嵌入检索的核心限制：它无法可靠桥接电影 \(\to\) 导演 \(\to\) 出生日期这一推理链，因为
中间实体与原始查询在语义上相距较远。相比之下，LLM-Wiki 通过结构化链接使这条链显式可遍历。

**案例 2：链接跟随遍历——通过结构指针进行多跳推理。** 问题：“John V, Prince of Anhalt-Zerbst 的父亲
何时去世？”（来自 2WikiMHQA）

Dense RAG 必须检索到一个同时提到 John V 和其父亲去世日期的段落。实践中，嵌入查询“John V Prince of
Anhalt-Zerbst father death”会检索 John V 自身传记或其他 Anhalt 王子的段落，但父亲的去世日期位于关于
Ernest I 的独立段落中，该段落可能并不提及 John V。

LLM-Wiki 智能体用三步解决该问题：(1) 搜索“John V, Prince of Anhalt-Zerbst”并定位实体页面；(2) 读取
该页面并发现一个指向 Ernest I 页面、标注为“father of John V”的结构化链接；(3) 跟随该链接并直接读取
到“Ernest I died on 12 June 1516 in Dessau.”检索通过三次工具调用完成且没有歧义。这展示了预编译双向
链接如何把多跳推理转化为组合式遍历：每一跳都由显式结构指针引导，而不是由概率相似度匹配引导。

**检索策略分类。** 除这些具体案例外，我们观察到智能体会自适应选择三种不同检索策略：

- **直接检索：** 对单实体事实查询，智能体发起一次 `wiki_search`，读取目标页面并直接抽取答案，通常在
  一到两次工具调用内完成任务。
- **链接跟随遍历：** 对多跳桥接查询，智能体读取实体页面并跟随显式 `links_to` 指针。每一跳都由结构化
  链接引导，而不只依赖概率检索。
- **浏览与聚合：** 对开放式或比较型查询，智能体读取目录索引页面（`_index.md`）以获得结构化概览，然后
  选择性批量读取相关页面。这种策略类似人类先浏览目录、再聚焦相关页面的方式。
