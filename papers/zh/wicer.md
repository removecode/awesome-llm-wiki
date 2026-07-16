---
title: "WiCER：Wiki 记忆编译、评估与精炼：面向 LLM Wiki 系统的迭代式知识编译"
original_title: "WiCER: Wiki-memory Compile, Evaluate, Refine Iterative Knowledge Compilation for LLM Wiki Systems"
source_url: "https://arxiv.org/abs/2605.07068"
authors:
  - "Juan M. Huerta"
---

> 本文为英文论文中文译本，仅供阅读参考。原文见 source_url。

# WiCER：Wiki 记忆编译、评估与精炼：面向 LLM Wiki 系统的迭代式知识编译

Juan M. Huerta  
Zinnia Tech Solutions  
10|600 Steamboat Road  
Greenwich, CT 06830, USA  
juan.huerta@zinnia.com

## 摘要

LLM Wiki 模式——将领域知识编译并提供为一种持久化工件，再通过 KV 缓存推理供 LLM 使用——有望以亚秒级
延迟和零检索失败提供上下文访问。要实现这一点，必须解决“编译鸿沟”：LLM 编译需要把原始文档蒸馏成
wiki，同时避免灾难性地丢弃关键事实。我们在 17 个 RepLiQA 领域（6,800 个问题）中刻画了这一鸿沟：
我们观察到，在经过策划的知识上，全上下文 KV 缓存推理优于 RAG（4.38 vs. 4.08/5，TTFT 快 7.3 倍），
但在规模扩大时会因注意力稀释而退化到低于 RAG；盲目编译则完全失败（2.14–2.32 vs. 3.46，
灾难性失败率 53–60%）。为解决编译鸿沟，我们提出 WiCER（Wiki-memory Compile, Evaluate, Refine），
这是一种受反例引导抽象精化（CEGAR）启发的迭代算法，能够缩小这一鸿沟。WiCER 用诊断探针对已编译
wiki 进行评估，识别被丢弃的事实，并在后续编译中强制保留这些事实。一到两次迭代即可恢复 80% 的损失质量
（在有基线的 15 个主题上，均值 3.24 vs. 原始全上下文 3.47），使灾难性失败相对减少 55%。对全部
17 个主题的消融实验确认，收益来自有针对性的诊断（+0.95），而不是通用固定（+0.16）。所有代码和基准
均已发布，以支持可复现研究。

## 1 引言

越来越多面向用户的 AI 应用——客服聊天机器人、特定领域助手和系统（如法律、保险等）、交互式知识库——
必须同时满足两个相互竞争的要求：答案必须扎根于权威领域知识，响应延迟又必须足够低，以支持实时对话交互。
检索增强生成（RAG）Lewis et al. [2020], Gao et al. [2024] 通过在查询时检索相关文档块来解决扎根性问题，
但它会增加检索延迟，并在检索器失败时存在遗漏相关上下文的风险。

一种根本不同的方法正在受到关注：不是在查询时检索片段，而是把领域知识编译成持久化工件，让模型在整个集合上
进行推理。Karpathy 的 LLM Wiki 模式 Karpathy [2026] 阐述了这一愿景：一个由原始来源、已编译 wiki 和
结构化 schema 组成的三层架构，其中“知识只编译一次并持续保持最新，而不是在每次查询时重新推导”。
Chan 等人把同一直觉形式化为缓存增强生成（Cache-Augmented Generation, CAG）Chan et al. [2025]：
把全部知识预加载到上下文中，并缓存由此得到的 KV 状态。

在本文中，我们主张，全上下文 KV 缓存推理是 LLM Wiki 模式的一种实用、可部署实现。通过把经过策划的文档集合
加载到模型上下文窗口中，并持久化 KV 缓存状态（例如通过 llama.cpp 的 prompt caching
（`--cache-prompt`）ggml-org [2024]），每个后续查询都可以在完整知识库上得到服务，并具有亚秒级延迟，
同时最大限度降低检索失败风险和块边界造成的信息损失。

**RAG 基线。** 另一种方案是嵌入查询并检索 top-$k$ 个最相关文档块，只用检索到的段落构造聚焦提示
Lewis et al. [2020], Karpukhin et al. [2020]。这会显著减少每个查询的 token 消耗，但当检索失败时，
会有遗漏相关信息的风险。

**编译鸿沟。** 当知识已经预先编译时，全上下文 KV 缓存推理能够兑现 LLM Wiki 方法的承诺：例如，使用
Policygenius 语料中的 30 篇策划文章（67K token，占 96K 上下文窗口的 70%），它优于 RAG
（4.38 vs. 4.08/5），并且 TTFT 快 7.3 倍（第 4 节）。但这一方法在规模扩大时会失效：在每个主题
80 篇原始 RepLiQA 文档 Montero et al. [2024]（55–95K token，占窗口 57–99%）上，横跨 17 个领域，
注意力稀释使全上下文质量降至 RAG 以下（3.47 vs. 3.64；第 5 节）。自然的修复方式——把原始文档编译成
结构化 wiki——会因编译阶段的信息损失而灾难性失败：LLM 编译器超出目标 2–3 倍进行过度压缩，丢弃关键事实，
得分仅为 2.14–2.32，而原始全上下文为 3.47（第 6 节）。

**WiCER。** 我们提出 WiCER（Wiki-memory Compile, Evaluate, Refine），一种用于缩小编译鸿沟的迭代算法。
它受 CEGAR Clarke et al. [2000] 启发；CEGAR 会在伪反例暴露抽象精度不足时精化抽象模型。WiCER 会针对
诊断探针对每个已编译 wiki 进行评估，诊断编译器丢弃了哪些事实，并强制下一次编译保留这些事实。跨 17 个主题，
一到两次 WiCER 迭代恢复了盲目编译所损失质量的 80%（在有基线的 15 个主题上，均值 3.24 vs. 原始全上下文
3.47），灾难性失败相对减少 55%（第 7 节）。关键的是，在部署系统中，真实查询流可以充当一个免费的、
持续更新的探针集：高频问题会自然暴露用户最需要的事实，从而在无需合成探针的情况下推动 wiki 持续精炼。

我们的贡献包括：（1）WiCER 算法，一个迭代式编译—评估—精炼循环，通过失败驱动的事实固定，在 17 个
RepLiQA 主题上恢复了盲目编译造成的 80% 质量鸿沟，并使灾难性失败相对减少 55%；（2）一项消融研究，
确认 WiCER 的收益来自有针对性的诊断（相对盲目编译 +0.95），而不是通用固定（+0.16）；（3）KV 缓存 wiki
的运行边界，刻画文档数量的交叉点以及支配质量的两种相反力量（注意力稀释与信息损失）；（4）端到端配方——
WiCER 实现；以及（5）一个横跨 17 个领域、6,800 个问题的可复现基准。

## 2 相关工作

**检索增强生成。** RAG 由 Lewis et al. [2020] 提出，用于在知识密集型任务中结合参数化记忆和非参数化记忆。
Dense Passage Retrieval Karpukhin et al. [2020] 建立了使用双编码器架构的高效稠密检索。Sentence-BERT
Reimers and Gurevych [2019] 为语义搜索提供了轻量级句向量。后续工作探索了分块策略以及检索精度与上下文
利用率之间的权衡 Gao et al. [2024]。我们的工作直接将 RAG 与把整个语料加载进上下文的替代方案进行比较。

**KV 缓存优化。** 高效的 KV 缓存管理对长上下文推理至关重要。Pope et al. [2023] 分析了大模型中受内存限制的
推理。PagedAttention Kwon et al. [2023] 为服务场景提供高效内存管理。量化 KV 缓存（例如 q8_0）可以在保留
质量的同时降低内存占用 ggml-org [2024]，近期工作还显示可降至 2-bit 精度 Liu et al. [2024c]。
Flash Attention Dao et al. [2022], Dao [2023] 加速了预填充和解码。MInference Jiang et al. [2024] 通过在每个
注意力头中识别动态稀疏注意力模式（A-shape、Vertical-Slash、Block-Sparse），并只计算重要条目，直接解决
预填充瓶颈，在无需微调的情况下实现长上下文最高 10 倍预填充加速。我们的基准专门在比较设置中衡量缓存和
上下文交付优化的端到端影响。

**KV 缓存共享与解耦。** 近期系统支持跨请求和跨机器共享、组合与分发 KV 缓存：用于前缀感知复用的
RadixAttention Zheng et al. [2024b]，用于分层解耦的 Mooncake Qin et al. [2025]，用于非前缀缓存组合的
CacheBlend Yao et al. [2025]，用于压缩流式传输的 CacheGen Liu et al. [2024b]，用于运行时前缀共享的
ChunkAttention Ye et al. [2024]，以及用于可复用注意力状态的 Prompt Cache Gim et al. [2024]。
RAPTOR Sarthi et al. [2024] 构造层次化文档树，天然映射到按主题组织的缓存分片。

**知识编译与缓存增强生成。** 一种正在出现的替代方案不是在查询时检索，而是离线编译知识并从缓存中服务。
Karpathy 的 LLM Wiki 模式 Karpathy [2026] 提出三层架构：原始源文档被蒸馏为结构化 wiki，再进一步压缩为
schema 和摘要；已编译工件（对于约 100 篇文章约为 100K token）被加载一次并缓存。Chan et al. Chan et al. [2025]
把这形式化为缓存增强生成（CAG）：把全部相关知识预加载到 LLM 上下文中，缓存 KV 状态，并以零检索开销服务
查询。CAG 完全消除了检索失败，但在文档数量增大时继承了“中间丢失”问题 Liu et al. [2024a]。我们的工作提供了
这些思想的首次大规模实证检验，量化了编译到单一上下文何时崩溃（第 5 节）。

**结构化知识编译。** 若干方法在检索或生成之前把文档组织成更丰富的中间表示。RAPTOR Sarthi et al. [2024]
递归地聚类并总结文档块为一棵树，从而支持多抽象层级检索。GraphRAG Edge et al. [2024] 抽取实体—关系图并生成
社群摘要，支持跨多文档的全局查询。经典层次化摘要 Chang et al. [2023] 自底向上构建粒度递增的摘要。三者都产生
静态编译：树、图或摘要层级一旦构建便固定不变。WiCER 有两个不同点：（1）它面向为 KV 缓存服务优化的扁平
wiki 工件，而不是检索索引；（2）它通过用诊断探针对已编译工件进行评估并迭代精炼来闭环——这是静态编译流水线
所缺少的反馈机制。

**LLM 作为评审的评估。** 使用 LLM 评估生成文本，作为人类评估的可扩展替代方案，已受到关注
Zheng et al. [2024a], Chiang and Lee [2023]。我们遵循这一范式，使用 Claude Sonnet 和结构化评分细则，
并对两种方法应用相同评审以保证公平比较。本文使用一项小规模（100 样本）人类评估来验证这一评审方法
（附录 J）。

## 3 实验设置

### 3.1 文档语料

**Policygenius（策划语料）。** 30 篇 Policygenius[^1] 文章（总计约 67K token，约 2,250 token/篇，占
96K 上下文窗口的 70%），覆盖寿险、伤残保险和遗产规划。这些经编辑策划的文章在 Karpathy [2026] 的分类中
已经构成“已编译 wiki”，因此是全上下文推理的最佳情形。

**RepLiQA（原始、多领域）。** RepLiQA Montero et al. [2024]（NeurIPS 2024）是一个无污染 QA 基准：
所有文档均为合成生成，并经验证不存在于训练语料中。它横跨 17 个主题领域，每个主题有 80 篇文档和 400 个
QA 对（1,360 篇文档、6,800 个问题、总计约 1.5M token）。每个主题语料范围从 55K token（Company Policies，
约 690 token/文档，占 96K 窗口的 57%）到 95K token（Regional Folklore，约 1,190 token/文档，占窗口的
99%）。每个问题只能由其源文档回答，因此可以精确衡量检索准确率。与 Policygenius 不同，这些原始文档没有经过
编辑编译，是研究编译鸿沟（第 6 节）和 WiCER（第 7 节）的测试床。

[^1]: <https://www.policygenius.com>

### 3.2 问答生成

对于 Policygenius，我们使用 Claude Sonnet 生成 240 个问答对（平均每篇 8 个），每篇文章产生 5–10 个多样化
问题（事实性、定义性、比较性、实践性），每个问题只能由其来源回答，并带有源文件名标签。RepLiQA 自带每个主题
400 个 QA 对（总计 6,800 个），由 Montero et al. [2024] 生成并验证。第 4 节使用 Policygenius 语料建立
全上下文 KV 缓存在策划知识上的优势；第 5–7 节使用 RepLiQA 跨 17 个领域对可扩展性、编译和 WiCER 进行压力测试。

### 3.3 系统配置

所有实验都在一台 Apple M4 Pro（24 GB 统一内存）上运行，使用 llama.cpp 生态中的 llama-server ggml-org [2024]，
后者支持在消费级硬件上用 GGUF 量化模型进行高效本地推理。Apple Silicon 的统一内存架构特别适合 LLM 推理，
因为 GPU 可以直接访问模型权重而无需 PCIe 传输。我们使用 Llama 3.1 8B Instruct Q5_K_M Dubey et al. [2024]、
96K 上下文窗口、q8_0 KV 缓存、Flash Attention（Metal）和贪婪解码（$T=0$）。

### 3.4 推理配置

**全上下文。** 所有 30 篇文章被拼接成一个系统提示（约 67K token）。一次预检请求会预热 KV 缓存；后续查询复用
缓存状态，只处理问题后缀（每个约 30–50 token）。

**RAG。** 文章被切分为 1,800 字符的块（360 字符重叠），得到 234 个块，并用 all-MiniLM-L6-v2
Reimers and Gurevych [2019] 进行嵌入。查询时，按余弦相似度检索 top-5 块（约 2,000 token），并格式化为
同一聊天模板。

### 3.5 评估方法

对于每个问题，我们通过流式 SSE 测量首 token 时间（TTFT）、总响应时间和生成吞吐。对于 RAG，我们还额外测量
检索准确率：正确源文章是否出现在 top-5 块中，以及位于什么排名。答案质量由 Claude Sonnet 作为 LLM 评审按
1–5 分评分，使用结构化评分细则（附录 B）；两种条件使用完全相同的评审和细则。一项 100 样本的人类评估验证了
该评审：Pearson $r=0.94$，75% 完全一致，99% 相差不超过 1 分（附录 J）。

## 4 已编译知识上的全上下文 KV 缓存

### 4.1 答案质量

表 1 比较了最佳全上下文配置（Q4K/Q4V）与 RAG。全部三种量化变体得分相差不超过 0.03 分（完整消融见附录 F），
确认 KV 缓存量化到 4-bit 不会引入可测量退化。相对 RAG 的质量差距（0.30 分）由检索失败驱动：RAG 产生的灾难性
错误（得分 1）是全上下文的 4.7 倍。

**表 1：答案质量（1–5 分，$n=240$）。全上下文使用最佳量化（Q4K/Q4V）；三种 FC 变体得分均在 0.03 分内。**

| 指标 | 全上下文（Q4K/Q4V） | RAG |
|---|---:|---:|
| 平均得分 | 4.38 | 4.08 |
| 得分 ≥4 的比例 | 91.2% | 79.6% |
| 得分 ≤2 的比例 | 5.8% | 12.9% |
| 得分 = 1（错误） | 3/240（1.2%） | 14/240（5.8%） |

### 4.2 计时：全上下文 vs. RAG

表 2 比较延迟和吞吐。全上下文热查询实现亚秒级 TTFT（中位数 0.86 s，比 RAG 的 6.28 s 快 7.3 倍），因为只需
针对缓存的 67K-token 前缀处理约 20 个新 token。RAG 因注意力跨度更短而获得 2.6 倍更高的生成吞吐
（30.74 vs. 12.01 tok/s），使总响应时间相近（7.93 s vs. 6.39 s）。摊销一次性预填充后，在 240 个查询中，
全上下文处理的有效 token 少 5.0 倍。RAG 在 92.5% 的情况下检索到正确来源；其 7.5% 的检索失败率（18/240）
解释了 14 个得分为 1 的答案。

**表 2：延迟和吞吐：最佳全上下文（Q4K/Q4V）vs. RAG。全上下文 TTFT 快 7.3 倍；RAG 生成吞吐高 2.6 倍。**

| 指标 | 全上下文（Q4K/Q4V） | RAG（Top-5） |
|---|---:|---:|
| TTFT – 中位数 | 0.857 s | 6.277 s |
| TTFT – P5 / P95 | 0.534 / 1.053 s | 4.483 / 7.697 s |
| TTFT – 最小 / 最大 | 0.487 / 1.107 s | 3.313 / 8.903 s |
| 总响应 – 中位数 | 6.392 s | 7.932 s |
| 吞吐 – 中位数 | 12.01 tok/s | 30.74 tok/s |
| 吞吐 – 范围 | 10.25–14.04 tok/s | 20.75–39.31 tok/s |

## 5 可扩展性鸿沟

第 4 节显示，全上下文 KV 缓存推理在 30 篇策划文章（67K token，占窗口 70%）上表现出色。现在我们跨 17 个
RepLiQA 主题（第 3.1 节）进行规模化评估，对每个主题独立应用第 3 节的相同方法（每个主题 80 篇文档、
55–95K token，占 96K 窗口 57–99%，每个主题 400 个 QA 对；15 个主题适配 FC，全部 17 个主题完成 RAG）。

### 5.1 结果

**TTFT 与量化。** 热缓存 TTFT 在各主题间保持一致（Q8 中位数 1.04 s，Q4 中位数 0.99 s；按主题分解见附录 G）。
Q4 平均降低 TTFT 4.8%，但与 Policygenius 上不同，在 80 篇文档时它会降低质量：Q8 在 14 个主题中的 13 个超过
Q4（均值 $\Delta=+0.14$），说明降低 KV 精度会在规模化时加剧注意力稀释。

**RAG 计时与检索准确率。** RAG 基准在全部 17 个主题（6,800 个问题）上完成。RAG 的 TTFT 中位数为 4.83 s
（$\sigma=0.22$ s），使全上下文平均拥有约 4.6 倍 TTFT 优势——与 Policygenius 上 7.3 倍优势一致
（比例较小是因为 RepLiQA 语料更大，使 FC TTFT 相对 RAG 增加更多）。RAG 吞吐高度稳定，为 32.7 tok/s
（$\sigma=0.3$），每个 RAG 查询平均消耗约 1,659 token。全部 17 个主题的检索准确率平均为 87.9%
（$\sigma=3.9\%$），范围从 79.5%（Incident Report）到 97.0%（News Stories）。这低于 Policygenius 结果
（92.5%），可能是因为 RepLiQA 合成文档在每个主题内部包含更多风格同质文本，使嵌入模型更难进行块级区分。

**质量：80 篇文档时 RAG 优于全上下文。** 对 15 个 FC 主题[^2] 和全部 17 个 RAG 主题进行 LLM 评审评估，
显示出与 Policygenius 发现相反的显著逆转。每个主题 80 篇文档时，全上下文质量明显下降：FC Q8 平均得分为
3.47（$\sigma=0.11$；≥4：64.0%），而 Policygenius 上为 4.35（≥4：90.0%）。更重要的是，RAG 持续优于
全上下文：RAG 平均为 3.64（$\sigma=0.16$），在 15 个主题中赢下 13 个，剩余 2 个打平（$|\Delta|<0.05$）。
全上下文没有赢下任何一个主题。平均质量差距为 -0.18 分（FC−RAG），完全逆转了 Policygenius 的 +0.27 差距。

[^2]: 两个主题（Regional Cuisine 和 Regional Folklore）在 80 篇文档时超过 96K 上下文窗口（分别为
97,251 和 97,148 token）。全部 17 个主题都在 RAG 下完成。即使某些 80 文档语料会溢出 96K 上下文窗口，
也进一步推动了第 8 节提出的主题分片架构。

其机制是“中间丢失”：全上下文产生 17.0% 的得分 1 答案（Policygenius 上为 1.2%）；逐问题交叉比对得分显示，
在全部 15 个 FC 主题中有 557 个案例（14 个 Q8 主题中有 530 个）FC 得分为 1 而 RAG 得分 ≥4。在这些案例中，
模型虽然在上下文中拥有全部 80 篇文档，却无法定位相关段落；检索器成功把范围缩小到约 2K token，模型便能成功。
RAG 的得分 1 率也相近，为 17.7%，由检索遗漏驱动（准确率 87.9%），但其失败发生在不同问题上——即正确块未被
检索到的问题。

这些结果确立了一个清晰的交叉点：全上下文在 30 篇文档和 67K token（Policygenius，占窗口 70%）上的质量优势，
在 80 篇文档和 55–95K token（RepLiQA，占 57–99%）时因注意力稀释而逆转。全上下文在已编译知识上表现出色，
但在原始集合上退化，这强化了 LLM Wiki 论点：决定可行性的不只是上下文长度，更是编译质量。完整按主题指标见
附录 G。

## 6 编译鸿沟

可扩展性鸿沟（第 5 节）显示，由于注意力稀释，全上下文质量会在 80 篇原始文档时退化。Karpathy 的 LLM Wiki
模式 Karpathy [2026] 给出了解法：先把原始来源编译成结构化 wiki，再进行服务。我们直接检验这一点——并发现
盲目编译会灾难性失败。

### 6.1 动机与流水线

两种力量相互对立：压缩过少会保留注意力稀释 Liu et al. [2024a]（模型拥有答案但找不到），而压缩过多会造成信息
损失（答案已被移除）。我们测试中间压缩水平是否能优化这一权衡。

对于 17 个 RepLiQA 主题中的每一个，我们使用 Claude Sonnet 作为知识工程师（看不到评估问题），把 80 篇原始文档
编译为结构化 wiki，目标为三种压缩水平——轻度（约 75%）、中度（约 50%）和激进（约 25%）——共得到 51 个
已编译 wiki。

### 6.2 结果

编译器持续过度压缩：轻度（目标 75%）实际达到 35.4%，中度（50%）实际达到 12.2%，激进（25%）实际达到 8.2%。
表 3 展示了跨全部 17 个主题（每种条件 6,800 个 QA 对）的综合压缩和质量结果。盲目编译在所有水平上都会降低质量：
即使轻度压缩也比 FC raw 低 1.14 分，且得分 1 率增加到三倍（52.9% vs. 17.3%）。

编译确实带来明确的延迟收益：全部 wiki 水平都实现亚 400 ms TTFT（比 FC raw 快 2.8–5.6 倍；完整计时见附录 H）。

**表 3：Wiki 编译：压缩与质量（跨 17 个 RepLiQA 主题、每种条件 6,800 个 QA 对的均值）。编译器在所有水平上
均过度压缩；所有 wiki 条件得分都低于两个基线。**

| 水平 | 目标 | 实际 | 质量 | 得分 1% | vs. FC raw |
|---|---:|---:|---:|---:|---:|
| FC raw（80 文档） | — | 100% | 3.46 | 17.3% | — |
| Wiki-light | 75% | 35.4% | 2.32 | 52.9% | -1.14 |
| Wiki-moderate | 50% | 12.2% | 2.25 | 57.1% | -1.21 |
| Wiki-aggressive | 25% | 8.2% | 2.14 | 60.3% | -1.32 |
| RAG | n/a | n/a | 3.63 | 17.7% | +0.17 |

### 6.3 分析

质量随压缩程度单调下降（2.32、2.25、2.14），全都远低于 FC raw（3.46）。根本原因是压缩遵从失败：编译器忽略
目标词数，压缩幅度比请求水平高 2×–3×（轻度目标 75% → 实际 35%）。在实际达到的比例（8–35%）下，编译器丢弃了
太多具体事实，模型无法恢复。得分 1 率（wiki 为 53–60%，FC raw 为 17%）确认，答案失败是因为信息缺失，而不是
找不到。这推动了 WiCER（第 7 节）：它使用评估反馈来保留盲目编译会丢弃的关键事实。

## 7 WiCER：Wiki 记忆编译、评估与精炼

盲目编译会丢失事实，因为编译器不知道下游查询会针对哪些细节。基于 RepLiQA 语料，我们提出 WiCER
（Wiki-memory Compile, Evaluate, Refine），一种使用 QA 反馈识别丢失事实并强制编译器在后续迭代中保留它们的
迭代算法。

### 7.1 算法

WiCER 从把源文档盲目编译成 wiki 开始，然后通过评估—诊断—重编译循环迭代改进。在每次迭代中，已编译 wiki 会
针对每个源文档一个的诊断探针进行测试；任何得分 1/5（灾难性失败）的探针都会触发诊断步骤，抽取编译器丢弃的
具体事实。这些累积事实作为显式保留约束反馈给下一次编译调用，确保它们不会再次丢失。

**算法 1：WiCER：Wiki-memory Compile, Evaluate, Refine**

```text
Require: 文档 D = {d_1, ..., d_N}，目标比例 r，最大迭代次数 T
Ensure: 优化后的 wiki W*

1:  探针选择：每个源文档选择一个 QA 对 -> Q_probe
2:  W_0 <- Compile(D, r)  {盲目编译}
3:  F_cumulative <- ∅
4:  for t = 0 to T - 1 do
5:      评估：对每个 q ∈ Q_probe，从 W_t 生成答案，并用 LLM 评审评分
6:      Failures_t <- {q: score(q) <= 1}
7:      if |Failures_t| = 0 or (t > 0 and improvement < 10%) then
8:          break  {已收敛}
9:      end if
10:     诊断：对每个失败 q，从源文档 d_q 抽取关键事实
11:     F_cumulative <- F_cumulative ∪ CriticalFacts_t
12:     W_{t+1} <- Compile(D, r, preserve=F_cumulative)
13: end for
14: return W* = W_t  {最佳 wiki}
```

### 7.2 设计原理

WiCER 在知识编译场景中实例化了反例引导抽象精化（Counterexample-Guided Abstraction Refinement, CEGAR）
范式 Clarke et al. [2000]。在 CEGAR 中，一个具体系统 $M=(S,S_0,R,L)$ 通过满射映射 $h:S\to\hat{S}$ 被近似为抽象
模型 $\hat{M}$；当对 $\hat{M}$ 的模型检查产生反例时，会对其进行分析——如果是伪反例，则精化抽象以消除它，
并重复循环。在 WiCER 中，具体系统是完整文档集合 $D$，抽象是已编译 wiki $W_t$（由 LLM 编译器形成的有损压缩），
规范是“所有诊断探针都高于灾难性失败”。得分 1 的探针充当反例；诊断确认它是伪反例（事实存在于 $D$ 中但在
$W_t$ 中丢失）；精化则添加固定约束，强制下一次编译保留丢失事实。附录 I 详细说明形式化映射、陈述单调性保证，
并指出类比的边界。

**收敛性。** 与 CEGAR 类似，每个精化步骤都会消除触发它的特定反例：被保留的事实不会再次丢失（它们受到显式
约束），因此固定事实上的失败集合单调缩小。未固定事实上可能出现新失败（随机知识置换），但只要保留预算没有挤占
太多一般覆盖，净效果就是正向的。WiCER 聚焦灾难性失败（1/5 分），每次失败抽取约 50–100 词，而每个源文档约
700 词。

### 7.3 结果

表 4 展示了 WiCER 在全部 17 个 RepLiQA 主题上、以中度压缩（$r=0.50$）运行的结果。在 17 个主题中的 16 个上，
WiCER 提升了质量；一个主题（local_education_systems）没有收益，算法正确地提前收敛。

恢复率在有 FC raw 基线的 15 个主题中范围为 0% 到 125%。十个主题在第 2 次迭代达到峰值，说明当初始编译严重退化时，
第二轮精炼有帮助。跨全部 17 个主题，WiCER 恢复了盲目编译损失质量的 80%（在有基线的 15 个主题上，均值
3.24 vs. FC raw 3.47），同时得分 1 的灾难性失败率从 55.1% 降至 24.8%（相对 -55%）。

**表 4：WiCER 在全部 17 个 RepLiQA 主题上的结果（中度压缩，每个主题 80 个探针）。每个主题的最佳 WiCER
迭代以粗体表示。**

| 主题 | 盲目 | 最佳 WiCER | FC raw | 恢复率 | 迭代 |
|---|---:|---:|---:|---:|---:|
| company_policies | 3.27 / 17.5% | **3.51 / 10.0%** | 3.58 | 78% | 1 |
| cybersecurity_news | 2.08 / 57.5% | **3.06 / 25.0%** | 3.45 | 72% | 1 |
| incident_report | 1.93 / 61.2% | **2.70 / 36.2%** | 3.24 | 59% | 1 |
| local_arts_&_culture | 1.65 / 73.8% | **3.61 / 15.0%** | 3.34 | 116% | 2 |
| local_economy | 2.16 / 55.0% | **3.24 / 22.5%** | 3.48 | 82% | 1 |
| local_education | 2.41 / 38.8% | 2.41 / 38.8% | 3.37 | 0% | — |
| local_env. issues | 1.95 / 61.2% | **2.88 / 33.8%** | 3.40 | 64% | 1 |
| local_health | 1.79 / 72.5% | **3.45 / 20.0%** | 3.52 | 96% | 2 |
| local_news | 2.16 / 51.2% | **3.29 / 20.0%** | 3.42 | 89% | 2 |
| local_politics | 1.96 / 65.0% | **3.62 / 15.0%** | 3.46 | 111% | 2 |
| local_sports | 2.30 / 56.2% | **3.29 / 26.2%** | 3.65 | 73% | 2 |
| local_tech | 2.36 / 55.0% | **3.23 / 25.0%** | 3.59 | 70% | 2 |
| neighborhood | 1.93 / 58.8% | **2.88 / 37.5%** | 3.46 | 62% | 2 |
| news_stories | 2.50 / 40.0% | **3.61 / 11.2%** | 3.60 | 101% | 2 |
| regional_cuisine | 2.48 / 48.8% | **2.76 / 37.5%** | — | — | 2 |
| regional_folklore | 1.77 / 68.8% | **2.80 / 33.8%** | — | — | 2 |
| small_medium_ent. | 2.21 / 56.2% | **3.75 / 13.8%** | 3.44 | 125% | 2 |
| 平均（17 主题） | 2.17 / 55.1% | **3.18‡ / 24.8%** | 3.47† | 80%† | — |

盲目和最佳 WiCER 列显示平均质量 / 得分 1 率。  
† FC raw 与恢复率基于有基线的 15 个主题（2 个超过上下文窗口）计算。  
‡ 在有 FC 基线的 15 个主题上，WiCER 均值为 3.24。

### 7.4 分析与局限

17 个主题中有 10 个在第 2 次迭代达到峰值；其余 7 个在第 1 次迭代达到峰值或没有收益，因为“随机知识置换”效应
——修复目标事实会挤出其他事实——限制了进一步改进。一个主题（local_education_systems）没有 WiCER 改进；
其相对较高的盲目基线（2.41）和较低的得分 1 率（38.8%）留下的可诊断灾难性失败更少。在有 FC raw 基线的
15 个主题上，恢复率跨度为 0–125%，其中三个主题（news_stories 为 101%，local_arts_and_culture 为 116%，
small_and_medium_enterprises 为 125%）在两次迭代后超过 FC raw 质量。实体特定事实较多的主题在绝对收益上最大
（local_arts_and_culture 为 +1.96，small_and_medium_enterprises 为 +1.54）。两个主题缺少 FC raw 基线，因为其
80 篇文档超过 96K 上下文窗口；WiCER 仍然使两者相对盲目编译提升。每次迭代需要约 130K API 输入 token 和约
17K 输出 token（一次编译调用、约 80 次评审调用、约 15 次诊断调用）；80 个本地推理探针不产生 API 成本。按当前
Sonnet 定价，每次迭代约 $1–2，约 50 分钟完成；成本随语料规模线性增长。

### 7.5 消融：诊断式固定 vs. 随机固定

为隔离诊断的贡献，我们运行一个随机固定对照：对于每个得分 1 的失败，固定的是从随机源文档中随机抽取的
50–100 词段落，而不是诊断出的关键事实。除这一点外，所有参数都与 WiCER 相同。每个主题使用独立的盲目编译；
由于 LLM 编译器非确定性，盲目基线与表 4 略有不同。

表 5 显示，随机固定相对盲目编译只提升 +0.16，而 WiCER 达到 +0.95——收益大 5.9 倍，并赢下 17 个主题中的
16 个。唯一例外（local_education）也是 WiCER 本身恢复率为 0% 的主题，说明其结构具有编译抗性。这些结果确认，
WiCER 的收益来自有针对性的诊断，而不是固定机制本身。

**表 5：消融：WiCER（诊断式固定）vs. 随机固定，跨全部 17 个 RepLiQA 主题（每个主题最佳迭代）。每个主题
80 个探针的平均质量。**

| 主题 | 盲目 | 随机固定 | WiCER |
|---|---:|---:|---:|
| company_policies | 3.38 | 3.38 | **3.51** |
| cybersecurity_news | 2.08 | 2.30 | **3.06** |
| incident_report | 2.01 | 2.06 | **2.70** |
| local_arts_&_culture | 1.74 | 2.58 | **3.61** |
| local_economy | 2.32 | 2.32 | **3.24** |
| local_education | 2.49 | **2.58** | 2.41 |
| local_env. issues | 1.94 | 2.28 | **2.88** |
| local_health | 1.76 | 2.25 | **3.45** |
| local_news | 2.10 | 2.29 | **3.29** |
| local_politics | 2.03 | 2.03 | **3.62** |
| local_sports | 2.40 | 2.40 | **3.29** |
| local_technology | 2.40 | 2.40 | **3.23** |
| neighborhood | 2.05 | 2.43 | **2.88** |
| news_stories | 2.56 | 2.68 | **3.61** |
| regional_cuisine | 2.51 | 2.51 | **2.76** |
| regional_folklore | 1.83 | 1.91 | **2.80** |
| small_medium_ent. | 2.27 | 2.27 | **3.75** |
| 平均 | 2.23 | 2.39 | **3.18** |

## 8 讨论与结论

全上下文 KV 缓存推理在已编译知识（Policygenius）上实现了 LLM Wiki 的承诺，但在原始文档（RepLiQA）上会随规模
扩大而崩溃。WiCER 缩小了这一鸿沟：跨 17 个 RepLiQA 主题，一到两次迭代恢复了 80% 的损失质量（均值
3.24 vs. 原始全上下文 3.47），使灾难性失败减少 55%。消融实验确认，有针对性的诊断（+0.95）驱动了收益，而非
通用固定（+0.16）。RAG 在每次查询时访问无界外部记忆；已编译 wiki 必须在固定 token 预算内预见所有查询。
WiCER 将这一不对称差距缩小到 0.46 分（3.18 vs. 3.64），同时提供约 12 倍更快的 TTFT，这突显了目标事实保留的
效率。对于部署，我们建议：文档数量 ≲50 时使用全上下文 Q8 KV 缓存；超过上下文时使用 RAG；对于更大语料，使用
WiCER 编译的主题分片 Zheng et al. [2024b], Qin et al. [2025], Yao et al. [2025]，并用生产查询流替代合成探针。
局限方面：结果特定于 Apple M4 Pro 上的 Llama 3.1 8B——我们认为这代表了个人计算架构的最新水平（WiCER 可在其中
发挥作用）；RAG 使用固定大小分块且无重排；17 个主题中有一个没有 WiCER 改进；经人类验证的 LLM 评审
（$r=0.94$；附录 J）覆盖 $n=100$ 个样本。三项扩展值得研究：（1）wiki–RAG 混合推理，在低置信度时回退到 RAG；
（2）自适应按文档压缩，用诊断信号为信息密集文档分配更多预算；（3）对固定事实挤出未固定内容速率的形式化置换
边界，从而得到更紧的收敛保证。

## References

Brian J. Chan, Chao-Ting Chen, Jui-Hung Cheng, and Hen-Hsen Tiong. Don’t do RAG: When
cache-augmented generation is all you need for knowledge tasks. In Proceedings of the ACM Web
Conference (WWW), 2025. arXiv:2412.15605.

Yapei Chang, Kyle Xu, Bowen Wang, Windson Lam, Kyunghyun Cho, and Mohit Iyyer.
BooookScore: A systematic exploration of book-length summarization in the era of LLMs. arXiv
preprint arXiv:2310.00785, 2023.

Cheng-Han Chiang and Hung-yi Lee. Can large language models be an alternative to human
evaluations? Proceedings of the 61st Annual Meeting of the Association for Computational
Linguistics, pages 15607–15631, 2023.

Edmund Clarke, Orna Grumberg, Somesh Jha, Yuan Lu, and Helmut Veith. Counterexample-guided
abstraction refinement. In Computer Aided Verification (CAV), volume 1855 of LNCS, pages
154–169. Springer, 2000.

Tri Dao. FlashAttention-2: Faster attention with better parallelism and work partitioning. arXiv
preprint arXiv:2307.08691, 2023.

Tri Dao, Dan Fu, Stefano Ermon, Atri Rudra, and Christopher Ré. FlashAttention: Fast and memory-
efficient exact attention with IO-awareness. In Advances in Neural Information Processing Systems,
volume 35, pages 16344–16359, 2022.

Abhimanyu Dubey, Abhinav Jauhri, Abhinav Pandey, Abhishek Kadian, Ahmad Al-Dahle, Aiesha
Letman, Akhil Mathur, Alan Schelten, Amy Yang, Angela Fan, et al. The Llama 3 herd of models.
arXiv preprint arXiv:2407.21783, 2024.

Darren Edge, Ha Trinh, Newman Cheng, Joshua Bradley, Alex Chao, Apurva Mody, Steven Truitt, and
Jonathan Larson. From local to global: A graph RAG approach to query-focused summarization.
arXiv preprint arXiv:2404.16130, 2024.

Yunfan Gao, Yun Xiong, Xinyu Gao, Kangxiang Jia, Jinliu Pan, Yuxi Bi, Yi Dai, Jiawei Sun, and
Haofen Wang. Retrieval-augmented generation for large language models: A survey. arXiv
preprint arXiv:2312.10997, 2024.

ggml-org. llama.cpp: LLM inference in C/C++. https://github.com/ggml-org/llama.cpp,
2024. Accessed: 2026-04-28.

In Gim, Guojun Chen, Seung-seob Lee, Nikhil Sarda, Anurag Khandelwal, and Lin Zhong. Prompt
cache: Modular attention reuse for low-latency inference. In Proceedings of Machine Learning
and Systems 6 (MLSys), 2024.

Huiqiang Jiang, Yucheng Li, Chengruidong Zhang, Qianhui Wu, Xufang Luo, Surin Ahn, Zhenhua
Han, Amir H Abdi, Dongsheng Li, Chin-Yew Lin, Yuqing Yang, and Lili Qiu. MInference 1.0:
Accelerating pre-filling for long-context LLMs via dynamic sparse attention. In Advances in
Neural Information Processing Systems, 2024. Spotlight.

Andrej Karpathy. The LLM Wiki pattern. GitHub Gist, https://gist.github.com/karpathy/
1dd0294ef9567971c1e4348a90d69285, 2026. April 2026.

Vladimir Karpukhin, Barlas O˘guz, Sewon Min, Patrick Lewis, Ledell Wu, Sergey Edunov, Danqi
Chen, and Wen-tau Yih. Dense passage retrieval for open-domain question answering. Proceedings
of the 2020 Conference on Empirical Methods in Natural Language Processing, pages 6769–6781,
2020.

Woosuk Kwon, Zhuohan Li, Siyuan Zhuang, Ying Sheng, Lianmin Zheng, Cody Hao Yu, Joseph
Gonzalez, Hao Zhang, and Ion Stoica. Efficient memory management for large language model
serving with PagedAttention. Proceedings of the 29th Symposium on Operating Systems Principles,
pages 611–626, 2023.

Patrick Lewis, Ethan Perez, Aleksandra Piktus, Fabio Petroni, Vladimir Karpukhin, Naman Goyal,
Heinrich Küttler, Mike Lewis, Wen-tau Yih, Tim Rocktäschel, et al. Retrieval-augmented genera-
tion for knowledge-intensive NLP tasks. Advances in Neural Information Processing Systems, 33:
9459–9474, 2020.

Nelson F. Liu, Kevin Lin, John Hewitt, Ashwin Paranjape, Michele Bevilacqua, Fabio Petroni, and
Percy Liang. Lost in the middle: How language models use long contexts. In Transactions of the
Association for Computational Linguistics, volume 12, pages 157–173, 2024a.

Yuhan Liu, Hanchen Li, Yihua Cheng, Siddhant Ray, Yuyang Huang, Qizheng Zhang, Kuntai Du,
Jiayi Yao, Shan Lu, Ganesh Ananthanarayanan, Michael Maire, Henry Hoffmann, Ari Holtzman,
and Junchen Jiang. CacheGen: KV cache compression and streaming for fast large language model
serving. In ACM SIGCOMM 2024 Conference, pages 38–56, 2024b.

Zirui Liu, Jiayi Yuan, Hongye Jin, Shaochen Zhong, Zhaozhuo Xu, Vladimir Braverman, Beidi Chen,
and Xia Hu. KIVI: A tuning-free asymmetric 2bit quantization for KV cache. In Proceedings of
the International Conference on Machine Learning, 2024c.

Joao Montero, Lukas Moreira, Valmir Belem, David Semedo, and Joao Magalhaes. RepLiQA: A
question-answering dataset for benchmarking LLMs on unseen reference documents. In NeurIPS
2024 Datasets and Benchmarks Track, 2024.

Reiner Pope, Sholto Douglas, Aakanksha Chowdhery, Jacob Devlin, James Bradbury, Jonathan
Heek, Kefan Xiao, Shivani Agrawal, and Jeff Dean. Efficiently scaling transformer inference.
Proceedings of Machine Learning and Systems, 5, 2023.

Ruoyu Qin, Zheming Li, Weiran He, Jialei Cui, Feng Ren, Mingxing Zhang, Yongwei Wu, Weimin
Zheng, and Xinran Xu. Mooncake: Trading more storage for less computation — a KVCache-
centric architecture for serving LLM chatbot. In 23rd USENIX Conference on File and Storage
Technologies (FAST), pages 155–170, 2025. Best Paper Award.

Nils Reimers and Iryna Gurevych. Sentence-BERT: Sentence embeddings using siamese BERT-
networks. In Proceedings of the 2019 Conference on Empirical Methods in Natural Language
Processing, pages 3982–3992, 2019.

Parth Sarthi, Salman Abdullah, Aditi Tuli, Shubh Khanna, Anna Goldie, and Christopher D. Man-
ning. RAPTOR: Recursive abstractive processing for tree-organized retrieval. In International
Conference on Learning Representations (ICLR), 2024.

Jiayi Yao, Hanchen Li, Yuhan Liu, Siddhant Ray, Yihua Cheng, Qizheng Zhang, Kuntai Du, Shan
Lu, and Junchen Jiang. CacheBlend: Fast large language model serving for RAG with cached
knowledge fusion. In Proceedings of the Twentieth European Conference on Computer Systems
(EuroSys), pages 94–109, 2025. Best Paper Award.

Lu Ye, Ze Tao, Yong Huang, and Yang Li. ChunkAttention: Efficient self-attention with prefix-aware
KV cache and two-phase partition. In Proceedings of the 62nd Annual Meeting of the ACL, pages
11608–11620, 2024.

Lianmin Zheng, Wei-Lin Chiang, Ying Sheng, Siyuan Zhuang, Zhanghao Wu, Yonghao Zhuang,
Zi Lin, Zhuohan Li, Dacheng Li, Eric Xing, et al. Judging LLM-as-a-judge with MT-bench and
chatbot arena. Advances in Neural Information Processing Systems, 36, 2024a.

Lianmin Zheng, Liangsheng Yin, Zhiqiang Xie, Chuyue Sun, Jeff Huang, Cody Hao Yu, Shiyi
Cao, Christos Kozyrakis, Ion Stoica, Joseph Gonzalez, Clark Barrett, and Ying Sheng. SGLang:
Efficient execution of structured language model programs. In Advances in Neural Information
Processing Systems 37 (NeurIPS), pages 62557–62583, 2024b.

## NeurIPS 论文清单

### 1. 声明

**问题：** 摘要和引言中的主要声明是否准确反映了论文的贡献和范围？

**回答：** [是]

**理由：** 摘要和引言陈述了四项具体声明：（1）WiCER 在 8 个主题上恢复了 71% 的质量差距（第 7 节，表 4）；
（2）FC 与 RAG 之间的文档数量交叉点（第 5 节）；（3）端到端配方（第 4–7 节）；（4）可复现基准（附录 C）。
所有声明均由实验结果支持，局限在第 8 节讨论。

**指南：**

- 回答 [N/A] 表示摘要和引言没有包含论文中的声明。
- 摘要和/或引言应清楚陈述论文中的声明，包括论文贡献以及重要假设和局限。对此问题回答 [否] 或 [N/A] 会给审稿人
  留下不佳印象。
- 声明应与理论和实验结果相匹配，并反映结果可在多大程度上推广到其他设置。
- 可以把理想目标作为动机，只要清楚表明这些目标尚未由论文实现。

### 2. 局限

**问题：** 论文是否讨论了作者所完成工作的局限？

**回答：** [是]

**理由：** 第 8 节（“局限”段落）讨论了硬件特异性（仅 Apple M4 Pro）、模型特异性（仅 Llama 3.1 8B）、
RAG 基线局限（固定大小分块、无重排）、WiCER 验证范围（17 个主题中的 8 个，有一个失败案例）以及
LLM 评审局限。

**指南：**

- 回答 [N/A] 表示论文没有局限，而回答 [否] 表示论文有局限但未讨论。
- 鼓励作者创建单独的“局限”章节。
- 论文应指出任何强假设，以及结果对这些假设被违反的稳健性（例如独立性假设、无噪声设置、模型设定正确性、
  仅局部成立的渐近近似）。作者应反思这些假设在实践中可能如何被违反及其影响。
- 作者应反思声明的范围，例如方法是否只在少数数据集或少数运行上测试。一般而言，经验结果通常依赖隐含假设，
  应予以说明。
- 作者应反思影响方法性能的因素。例如，图像分辨率低或弱光条件下人脸识别算法可能表现较差；或者语音转文本系统
  可能不能可靠地为在线讲座提供字幕，因为它难以处理技术术语。
- 作者应讨论所提算法的计算效率以及它们如何随数据集规模扩展。
- 如适用，作者应讨论其方法在隐私和公平性问题上的潜在局限。
- 虽然作者可能担心完全诚实地说明局限会被审稿人作为拒稿理由，但更糟的结果可能是审稿人发现了未承认的局限。
  作者应运用最佳判断，并认识到支持透明性的个人行动在发展维护共同体诚信的规范中具有重要作用。审稿人会被明确
  指示不得惩罚关于局限的诚实说明。

### 3. 理论假设和证明

**问题：** 对每个理论结果，论文是否给出了完整假设集合以及完整（且正确）的证明？

**回答：** [N/A]

**理由：** 本文主要是经验研究。它没有提出形式化定理或证明。WiCER 算法的收敛性质在第 7 节中以非形式化方式讨论
（保留事实失败集的单调收缩），但没有声称形式证明。

**指南：**

- 回答 [N/A] 表示论文不包含理论结果。
- 所有定理、公式和证明都应编号并交叉引用。
- 所有假设都应在任何定理陈述中清楚说明或引用。
- 证明可以出现在正文或补充材料中；若在补充材料中，鼓励作者提供简短证明草图以帮助直觉理解。
- 反过来，核心论文中提供的任何非形式化证明都应由附录或补充材料中的形式化证明补充。
- 证明依赖的定理和引理应得到恰当引用。

### 4. 实验结果可复现性

**问题：** 论文是否充分披露了复现论文主要实验结果所需的全部信息，达到会影响主要声明和/或结论的程度
（无论是否提供代码和数据）？

**回答：** [是]

**理由：** 第 3 节说明了所有硬件、软件、模型、量化、分块、嵌入和评估参数。附录 C 提供完整仓库结构和复现全部
实验的精确命令。RepLiQA 数据集公开可用 Montero et al. [2024]。WiCER 算法在算法 1 中完整指定。

**指南：**

- 回答 [N/A] 表示论文不包含实验。
- 如果论文包含实验，对此问题回答 [否] 会给审稿人留下不佳印象：使论文可复现很重要，无论是否提供代码和数据。
- 如果贡献是数据集和/或模型，作者应描述使结果可复现或可验证所采取的步骤。
- 取决于贡献类型，可复现性可以通过多种方式实现。例如，如果贡献是新架构，完整描述架构可能已足够；如果贡献是
  特定模型和经验评估，则可能需要让他人能够用相同数据集复现实验，或提供模型访问。一般而言，发布代码和数据通常
  是一种好方式，但也可以通过复现实验的详细说明、托管模型访问（如大语言模型）、发布模型 checkpoint 或其他适合
  研究性质的方式来提供可复现性。
- 虽然 NeurIPS 不要求发布代码，但会议要求所有投稿提供某种合理的可复现途径，这取决于贡献性质。例如：
  - 如果贡献主要是新算法，论文应清楚说明如何复现该算法。
  - 如果贡献主要是新模型架构，论文应清楚且完整地描述架构。
  - 如果贡献是新模型（如大语言模型），则应有访问该模型以复现结果的方式，或有复现该模型的方式（如开放数据集或
    构建数据集的说明）。
  - 我们承认可复现性在某些情况下可能很困难，作者可以描述其提供可复现性的具体方式。对于闭源模型，访问可能受到
    某种限制（例如面向注册用户），但其他研究者应有某种路径来复现或验证结果。

### 5. 数据和代码开放访问

**问题：** 论文是否提供数据和代码的开放访问，并带有充分说明，以按补充材料所述忠实复现主要实验结果？

**回答：** [是]

**理由：** 所有基准代码、评估脚本和 WiCER 实现都将作为开源发布。RepLiQA 数据集公开可用 Montero et al. [2024]。
Policygenius 文章是公开可访问网页。附录 C 提供完整目录结构和复现命令。

**指南：**

- 回答 [N/A] 表示论文不包含需要代码的实验。
- 更多细节请见 NeurIPS 代码和数据提交指南
  (<https://neurips.cc/public/guides/CodeSubmissionPolicy>)。
- 虽然我们鼓励发布代码和数据，但理解这并非总是可能，因此 [否] 是可接受回答。论文不能仅因不包含代码而被拒，
  除非代码是贡献的核心（例如新的开源基准）。
- 说明应包含复现结果所需运行的精确命令和环境。更多细节见 NeurIPS 代码和数据提交指南。
- 作者应提供数据访问和准备说明，包括如何访问原始数据、预处理数据、中间数据和生成数据等。
- 作者应提供脚本来复现新方法和基线的全部实验结果。如果只有部分实验可复现，应说明哪些被省略及原因。
- 在投稿时，为保持匿名，作者应发布匿名版本（如适用）。
- 建议在补充材料（附加到论文中）提供尽可能多的信息，但也允许包含数据和代码 URL。

### 6. 实验设置/细节

**问题：** 论文是否说明了理解结果所需的全部训练和测试细节（例如数据划分、超参数、如何选择、优化器类型）？

**回答：** [是]

**理由：** 第 3 节说明了全部推理参数（模型、量化、上下文窗口、温度、块大小、重叠、嵌入模型、top-k）。没有进行
训练——本文在推理模式下评估预训练模型。WiCER 算法参数（压缩目标、最大迭代次数、收敛阈值）在第 7 节说明。

**指南：**

- 回答 [N/A] 表示论文不包含实验。
- 实验设置应在核心论文中呈现到足以理解结果并把握其意义的详细程度。
- 完整细节可以在代码、附录或补充材料中提供。

### 7. 实验统计显著性

**问题：** 论文是否报告了适当且正确定义的误差条，或其他关于实验统计显著性的适当信息？

**回答：** [否]

**理由：** 我们报告了跨主题均值、标准差和范围（例如附录 G），但没有报告单个主题比较的置信区间或显著性检验。
每个主题的结果是确定性的（贪婪解码，$T=0$），因此主题内方差为零；报告的变化是跨主题变化。我们承认这是一个
局限——使用不同 QA 划分或随机解码进行重复运行会加强声明。

**指南：**

- 回答 [N/A] 表示论文不包含实验。
- 如果支持主要声明的实验至少伴随误差条、置信区间或统计显著性检验，作者应回答 [是]。
- 应清楚说明误差条捕获的变异因素（例如训练/测试划分、初始化、某些参数随机抽样或给定实验条件下的整体运行）。
- 应解释误差条计算方法（闭式公式、库函数调用、bootstrap 等）。
- 应给出所作假设（例如误差服从正态分布）。
- 应清楚误差条是标准差还是均值标准误。
- 可以报告 1-sigma 误差条，但应说明。若未验证误差正态性，作者最好报告 2-sigma 误差条，而不是称其为 96% CI。
- 对于非对称分布，作者应注意不要在表格或图中显示会超出范围（如负错误率）的对称误差条。
- 如果表格或图中报告误差条，作者应在正文中解释其计算方式并引用相应图表。

### 8. 实验计算资源

**问题：** 对每个实验，论文是否提供了足够信息说明复现实验所需计算资源（计算 worker 类型、内存、执行时间）？

**回答：** [是]

**理由：** 第 3 节说明了硬件（Apple M4 Pro，24 GB 统一内存）。第 7 节报告了每次 WiCER 迭代成本（$1–2，约
50 分钟）。附录 D 预测了替代硬件（RTX 4090、Inferentia2）上的性能。Policygenius 基准耗时约 30–60 分钟；
每个 RepLiQA 主题的 FC + RAG + 评估耗时约 2–3 小时。

**指南：**

- 回答 [N/A] 表示论文不包含实验。
- 论文应指出计算 worker 的类型（CPU 或 GPU、内部集群或云提供商），包括相关内存和存储。
- 论文应提供各个实验运行所需的计算量，并估计总计算量。
- 论文应披露完整研究项目是否需要比论文所报告实验更多的计算（例如未进入论文的初步或失败实验）。

### 9. 伦理规范

**问题：** 论文中的研究是否在所有方面符合 NeurIPS 伦理规范
<https://neurips.cc/public/EthicsGuidelines>？

**回答：** [是]

**理由：** 本研究使用公开可用数据集（RepLiQA、Policygenius 网页文章）和开源模型（Llama 3.1 8B）。不涉及人类
受试者、私有数据或双重用途问题。LLM 评审使用商业 API（Claude Sonnet），遵循标准服务条款。

**指南：**

- 回答 [N/A] 表示作者尚未审阅 NeurIPS 伦理规范。
- 如果作者回答 [否]，应解释需要偏离伦理规范的特殊情况。
- 作者应确保保持匿名性（例如，如果因所在司法辖区法律法规存在特殊考虑）。

### 10. 更广泛影响

**问题：** 论文是否讨论了所完成工作的潜在正面社会影响和负面社会影响？

**回答：** [N/A]

**理由：** 本工作是关于 LLM 推理知识编译和缓存的基础设施研究。它不引入新的模型能力，不为公众消费生成合成内容，
也不支持超出更高效知识检索之外的应用。主要影响是降低领域特定 QA 系统的计算成本和延迟。除通常 LLM 部署固有的
影响外，我们未预见直接负面社会影响。

**指南：**

- 回答 [N/A] 表示所完成工作没有社会影响。
- 如果作者回答 [N/A] 或 [否]，应解释为什么其工作没有社会影响，或为什么论文没有讨论社会影响。
- 负面社会影响的例子包括潜在恶意或非预期使用（如虚假信息、生成虚假档案、监控）、公平性考虑（如部署会对特定
  群体产生不公平影响的决策技术）、隐私考虑和安全考虑。
- 会议预期许多论文是基础研究，并不绑定具体应用，更不用说部署。然而，如果存在任何负面应用的直接路径，作者应指出。
  例如，指出生成模型质量提升可能被用于生成用于虚假信息的 Deepfake 是合理的。另一方面，没有必要指出优化神经网络
  的通用算法可能让人更快训练生成 Deepfake 的模型。
- 作者应考虑技术按预期使用且正确运行时可能产生的伤害，按预期使用但给出错误结果时可能产生的伤害，以及有意或无意
  误用带来的伤害。
- 如果存在负面社会影响，作者也可讨论可能缓解策略（如门控发布模型、除攻击外提供防御、监控误用机制、监控系统如何
  随反馈学习的机制、提升 ML 效率和可及性）。

### 11. 安全保障

**问题：** 论文是否描述了为负责任发布具有高误用风险的数据或模型（如预训练语言模型、图像生成器或抓取数据集）
而采取的保障措施？

**回答：** [N/A]

**理由：** 论文不发布预训练模型、微调权重或抓取数据集。它发布基准代码和评估脚本，运行于公开可用数据和模型之上。
这些工具除标准 LLM 推理外不构成额外误用风险。

**指南：**

- 回答 [N/A] 表示论文不构成此类风险。
- 对具有高误用或双重用途风险的发布模型，应配套必要保障以允许受控使用，例如要求用户遵守使用指南或限制访问模型，
  或实现安全过滤器。
- 从互联网抓取的数据集可能带来安全风险。作者应描述其如何避免发布不安全图像。
- 我们承认提供有效保障具有挑战性，许多论文不需要这样做，但鼓励作者考虑这一点并作出善意努力。

### 12. 现有资产许可证

**问题：** 论文中使用的资产（如代码、数据、模型）的创建者或原始所有者是否得到适当署名，许可证和使用条款是否
明确提及并得到适当遵守？

**回答：** [是]

**理由：** 所有资产均已引用：Llama 3.1 Dubey et al. [2024]（Llama 3.1 Community License）、llama.cpp
ggml-org [2024]（MIT License）、RepLiQA Montero et al. [2024]（CC-BY-4.0）、Sentence-BERT
Reimers and Gurevych [2019]（Apache 2.0）。Policygenius 文章是公开可用网页内容，按标准使用条款访问。

**指南：**

- 回答 [N/A] 表示论文不使用现有资产。
- 作者应引用产生代码包或数据集的原始论文。
- 作者应说明所用资产版本，并在可能时包含 URL。
- 每项资产应包含许可证名称（如 CC-BY 4.0）。
- 对来自特定来源（如网站）的抓取数据，应提供该来源的版权和服务条款。
- 如果发布资产，应在包中提供许可证、版权信息和使用条款。对于常见数据集，paperswithcode.com/datasets 已为部分
  数据集整理许可证，其许可指南可帮助确定数据集许可证。
- 对重新打包的现有数据集，应提供原始许可证和衍生资产许可证（如发生变化）。
- 如果这些信息无法在线获得，鼓励作者联系资产创建者。

### 13. 新资产

**问题：** 论文引入的新资产是否有良好文档，并且文档是否与资产一起提供？

**回答：** [是]

**理由：** 论文发布基准代码、WiCER 实现和评估脚本。附录 C 记录了仓库结构、依赖和复现命令。代码将以开源许可证
发布并带有文档。

**指南：**

- 回答 [N/A] 表示论文不发布新资产。
- 研究者应通过结构化模板在投稿中说明数据集/代码/模型细节。这包括训练、许可证、局限等细节。
- 论文应讨论是否以及如何从资产涉及人员处获得同意。
- 投稿时，记得匿名化资产（如适用）。可以创建匿名 URL 或包含匿名 zip 文件。

### 14. 众包与人类受试者研究

**问题：** 对于众包实验和人类受试者研究，论文是否包含给参与者的完整说明文本和截图（如适用），以及报酬细节
（如有）？

**回答：** [N/A]

**理由：** 论文不涉及众包或人类受试者。全部评估通过 LLM 评审自动完成。

**指南：**

- 回答 [N/A] 表示论文既不涉及众包，也不涉及人类受试者研究。
- 可以把这些信息包含在补充材料中；但如果主要贡献涉及人类受试者，则应尽量在正文中包含尽可能多的细节。
- 根据 NeurIPS 伦理规范，参与数据收集、整理或其他劳动的工作者应至少获得数据收集者所在国家的最低工资。

### 15. 人类受试者研究的机构审查委员会（IRB）批准或等效批准

**问题：** 论文是否描述了研究参与者可能承担的风险、这些风险是否向受试者披露，以及是否获得机构审查委员会
（IRB）批准（或根据所在国家或机构要求的等效批准/审查）？

**回答：** [N/A]

**理由：** 未进行人类受试者研究。

**指南：**

- 回答 [N/A] 表示论文既不涉及众包，也不涉及人类受试者研究。
- 根据研究所在国家，任何人类受试者研究都可能需要 IRB 批准（或等效批准）。如果获得 IRB 批准，应在论文中清楚说明。
- 我们承认不同机构和地点的程序可能差异很大，并期望作者遵守 NeurIPS 伦理规范及其机构指南。
- 初次投稿时，不要包含会破坏匿名性的信息（如适用），例如进行审查的机构。

### 16. LLM 使用声明

**问题：** 如果 LLM 是本研究核心方法中的重要、原创或非标准组成部分，论文是否描述了 LLM 的使用？注意，如果 LLM
仅用于写作、编辑或格式化，且不影响研究核心方法、科学严谨性或原创性，则无需声明。

**回答：** [是]

**理由：** LLM 是方法的核心：（1）Llama 3.1 8B 作为 FC 和 RAG 条件下的推理模型（第 3 节）；（2）Claude Sonnet
作为 LLM 评审评估器（第 3 节）、QA 对生成器以及 WiCER 中的 wiki 编译器（第 7 节）。所有 LLM 用途、模型版本和
提示均已完整披露。

**指南：**

- 回答 [N/A] 表示本研究核心方法开发不涉及作为重要、原创或非标准组成部分的 LLM。
- 关于哪些内容应描述或不应描述，请参考 NeurIPS 手册中的 LLM 政策。

## 附录 A：RAG 索引统计

表 6 显示每篇文章的块分布。

**表 6：每篇文章的块数（按块数排名前 10）。**

| 文章 | 块数 |
|---|---:|
| life-insurance-company-reviews-2024 | 22 |
| life-insurance-for-estate-planning | 19 |
| best-life-insurance-companies | 16 |
| best-no-exam-life-insurance-companies | 15 |
| best-disability-insurance-companies | 14 |
| how-does-life-insurance-work | 13 |
| what-is-whole-life-insurance | 10 |
| types-of-disability-insurance | 9 |
| how-much-does-a-funeral-cost | 9 |
| is-life-insurance-a-good-investment | 9 |
| 总计（30 篇文章） | 234 |

## 附录 B：评估评分细则

用于 Claude Sonnet LLM 评审评估的完整系统提示：

```text
你是一名专家评估者，负责比较生成答案与黄金标准答案。
请将生成答案从 1 到 5 分评分：
5 = 完美——捕捉所有关键事实，准确
4 = 良好——基本正确，只有轻微遗漏
3 = 可接受——部分正确，遗漏细节
2 = 较差——显著错误或重大遗漏
1 = 错误——不正确、不相关、没有回答要点
只用有效 JSON 回复：
{"score": <1-5>, "reasoning": "<解释>"}
```

## 附录 C：可复现性

所有代码都组织在仓库中：

```text
benchmark/                         # 全上下文 KV 缓存
  config.py                        # 路径、模型、服务器参数
  start_server.sh                  # 启动 llama-server
  run_benchmark.py                 # 主运行器（TTFT、吞吐）
  evaluate.py                      # Claude LLM 评审评分
  report.py                        # 聚合指标 + 报告

rag_benchmark/                     # RAG 基线
  rag_config.py                    # 块大小、top-k、嵌入模型
  build_index.py                   # 分块 + 嵌入 + 保存 pickle
  run_rag_benchmark.py             # RAG 运行器（嵌入、检索、生成）
  rag_evaluate.py                  # Claude LLM 评审（相同评分细则）
  rag_report.py                    # 聚合 + 检索指标
```

复现命令：

```bash
# 全上下文基准
bash benchmark/start_server.sh
python3 benchmark/run_benchmark.py     # ~30-60 min
python3 benchmark/evaluate.py          # ~2 min
python3 benchmark/report.py

# RAG 基准（服务器已运行）
python3 rag_benchmark/build_index.py        # ~5 s
python3 rag_benchmark/run_rag_benchmark.py  # ~15-20 min
python3 rag_benchmark/rag_evaluate.py       # ~2 min
python3 rag_benchmark/rag_report.py
```

## 附录 D：硬件架构预测

我们的基准完全运行在一台 Apple M4 Pro 计算机上（24 GB 统一内存，273 GB/s 内存带宽）。本附录预测离散 GPU 和
云加速器硬件上的预期性能，以指导部署决策。

### NVIDIA RTX 4090（桌面 GPU）

RTX 4090 提供 1,008 GB/s 内存带宽（M4 Pro 的 3.7 倍）和 24 GB GDDR6X VRAM。Llama 3.1 8B Q4_K_M
（约 5 GB）可以轻松放入，剩余约 19 GB 用于 KV 缓存——足以支持 128K 上下文且无需卸载。

RTX 4090 的关键含义：

- **解码受带宽限制：** 3.7 倍带宽优势转化为约 4.4 倍更高生成吞吐（约 53 vs. 12 tok/s），因为单批自回归解码受
  内存带宽限制。
- **预填充受计算限制：** CUDA tensor cores 提供约 2,600 tok/s 的预填充吞吐，将冷启动从约 130 s 降至约 26 s
  （5 倍加速）。热 TTFT 降至 0.2 s 以下。
- **KV 量化在 CUDA 上无惩罚：** 与 Metal 不同，CUDA 的融合 Flash Attention kernel 能处理 8-bit 对称量化 KV，
  无需额外反量化开销。因此在 CUDA 上建议使用 Q8 KV，以在不需要 Apple Silicon 上 4 倍压缩的情况下提供 2 倍内存节省。
- **RAG 优势减弱：** 在桌面 GPU 硬件上，全上下文热 TTFT 为 0.2 s，而 RAG 流水线延迟（嵌入 + 检索 + 提示构造）
  约为 1–2 s，因此 RAG 的延迟理由基本消失。

### AWS Inferentia2（云加速器）

AWS Inferentia2（inf2 实例）使用 Neuron SDK 和预先编译。若干架构约束限制了它对我们工作负载的适用性：

- **静态张量形状：** Neuron 编译要求编译时固定序列长度。变长输入需要填充到最大长度，或为不同长度桶维护多个已编译
  模型，增加复杂性并浪费计算。
- **没有 KV 缓存量化：** Neuron runtime 不暴露 KV 缓存量化选项。全部 KV 状态以 FP16/BF16 存储，所需内存为 Q8
  的 2 倍、Q4 配置的 4 倍。
- **内存扩展：** 单个 Inferentia2 芯片有 32 GB HBM。在 67K 上下文、FP16 KV 下，Llama 3.1 8B 的缓存需要约
  4.5 GB。虽然单芯片可行，但扩展到更长上下文（128K+）需要通过 inf2.24xlarge（约 $12/hr）进行多芯片张量并行。
- **偏向吞吐：** Inferentia2 擅长高批量、固定长度工作负载（如嵌入生成、分类）。我们的单用户、变长、长上下文 QA
  工作负载无法充分利用硬件。

**表 7：67K 上下文时 RTX 4090 vs. M4 Pro 的预测性能。**

| 指标 | M4 Pro | RTX 4090（估计） |
|---|---:|---:|
| 内存带宽 | 273 GB/s | 1,008 GB/s |
| 解码吞吐 | 12 tok/s | ~53 tok/s |
| 冷预填充（67K token） | ~130 s | ~26 s |
| 热 TTFT | 0.86 s | ~0.2 s |
| KV 量化惩罚（Q4） | 无 | 无（融合 FA） |

**表 8：缓存知识库 QA 的硬件适配性。**

| 因素 | M4 Pro | RTX 4090 | Inferentia2 |
|---|---:|---:|---:|
| KV 量化支持 | Q4/Q8 | Q8（原生） | 无 |
| 变长上下文 | 是 | 是 | 有限 |
| 热 TTFT（67K） | 0.86 s | ~0.2 s | ~0.3 s* |
| 解码吞吐 | 12 tok/s | ~53 tok/s | ~40 tok/s* |
| 成本（硬件） | $2,400 | $1,600 | $1.58/hr |
| 可离线 | 是 | 是 | 否 |

*估计值；至少需要 inf2.xlarge，并为固定 67K 长度编译。*

### 部署建议

表 8 总结了硬件对我们基准工作负载的适配性。

对于缓存知识库应用——即特定领域语料适配单个上下文窗口的情况——M4 Pro 提供了有吸引力的性价比，并具备完全离线
能力。RTX 4090 为延迟敏感部署提供最佳绝对性能。像 Inferentia2 这样的云加速器更适合高吞吐批量推理工作负载，
而不是缓存文档 QA 特有的交互式、变长模式。

## 附录 E：客户体验指标

**表 9：客户体验比较。使用 Q4 KV 缓存的全上下文提供更快的感知响应、更高可靠性和可接受的阅读速度生成。**

| 客户指标 | 全上下文（Q4K/Q4V） | RAG |
|---|---:|---:|
| 首次响应时间 | 0.86 s（中位数） | 6.28 s（中位数） |
| 第 95 百分位等待 | 1.05 s | 7.70 s |
| 总回答时间 | 6.39 s | 7.93 s |
| 好答案（≥4） | 91.2% | 79.6% |
| 错误答案（得分 1） | 1.2%（3/240） | 5.8%（14/240） |
| 阅读速度 | ~12 tok/s | ~31 tok/s |
| 每 10 问会话的 token | ~68K | ~20K |

## 附录 F：KV 缓存量化消融

表 10 展示了 Policygenius 语料（$n=240$）上三种全上下文配置的完整量化消融。Q4K/Q4V 获得最快 TTFT 中位数
（0.857 s，比 Q8K/Q8V 好 2.2%）、最快总响应时间（6.39 s，好 3.4%）和可比吞吐——同时相对 FP16 使用 4 倍更少
KV 缓存内存，并保持相同答案质量。这说明在该上下文长度下，Metal 的统一内存架构和 llama.cpp 的 Metal kernel
能够高效处理 KV 反量化。

**表 10：KV 缓存量化消融（Policygenius，$n=240$）。更激进量化在 Apple Silicon 上提供适度延迟改善，且无质量惩罚。**

| 指标 | Q8K/Q8V | Q8K/Q4V | Q4K/Q4V |
|---|---:|---:|---:|
| KV 缓存内存（每 token） | 1.0 B | 0.75 B | 0.5 B |
| 相对 FP16 的内存减少 | 2× | 2.7× | 4× |
| TTFT – 中位数 | 0.876 s | 0.877 s | 0.857 s |
| TTFT – P5 / P95 | 0.533 / 1.088 s | 0.531 / 1.107 s | 0.534 / 1.053 s |
| 总响应 – 中位数 | 6.620 s | 6.754 s | 6.392 s |
| 吞吐 – 中位数 | 12.03 tok/s | 11.88 tok/s | 12.01 tok/s |
| 平均得分 | 4.35 | 4.36 | 4.38 |
| 得分 ≥4 的比例 | 90.0% | 89.6% | 91.2% |

## 附录 G：多领域详细结果

**各主题 TTFT。** 表 11 展示了全部 14 个 Q8K/Q8V 主题和 15 个 Q4K/Q4V 主题的全上下文计时结果。在 Q8 下，
热缓存 TTFT 非常一致：14 个主题中的 13 个中位数聚集在 1.02–1.11 s 之间，Company Policies 是唯一离群点，
原因是其语料较小（约 55K vs. 约 90K token）。

**RepLiQA 上的 KV 缓存量化。** 在 14 个成对主题上，Q4K/Q4V 平均降低 TTFT 4.8%（0.989 s vs. 1.040 s），
14 个主题中的 13 个有所改善。然而，与 Policygenius 中 Q4 质量匹配 Q8 不同，在 80 篇文档时 Q4 会降低质量：
Q8 在 14 个主题中的 13 个得分高于 Q4（平均 $\Delta=+0.14$ 分，范围 -0.01 到 +0.28），且 Q4 产生更多得分 1
失败（20.9% vs. 17.3%）。较低 KV 精度显然加剧了“中间丢失”效应。

**表 11：RepLiQA 各主题的全上下文 KV 缓存计时（每个主题 400 个问题）。包含所有 14 个 Q8 主题；15 个 Q4 主题
（包括只在 Q4 下完成的 News Stories）。**

| 主题 | Token | Q8 TTFT | Q4 TTFT | Δ | Q8 tok/s | Q4 tok/s |
|---|---:|---:|---:|---:|---:|---:|
| Company Policies | 54,987 | 0.672 s | 0.674 s | +0.3% | 14.6 | 14.2 |
| Local Health | 88,649 | 1.020 s | 0.978 s | -4.2% | 10.7 | 11.1 |
| Local News | 90,517 | 1.019 s | 1.001 s | -1.8% | 10.8 | 10.6 |
| Neighborhood Stories | 90,666 | 1.090 s | 0.996 s | -8.7% | 10.1 | 10.9 |
| Small/Med Enterprises | 90,701 | 1.094 s | 1.032 s | -5.7% | 10.0 | 10.3 |
| Local Education | 90,708 | 1.056 s | 0.987 s | -6.5% | 10.3 | 10.8 |
| Incident Report | 91,021 | 1.078 s | 1.015 s | -5.9% | 10.0 | 10.8 |
| Env. Issues | 91,458 | 1.071 s | 0.987 s | -7.8% | 10.2 | 10.8 |
| News Stories | 92,182 | — | 1.046 s | — | — | 10.2 |
| Cybersecurity News | 92,135 | 1.060 s | 1.032 s | -2.7% | 10.4 | 10.4 |
| Local Economy | 92,436 | 1.074 s | 1.019 s | -5.1% | 10.2 | 10.6 |
| Local Politics | 92,684 | 1.051 s | 1.041 s | -0.9% | 10.6 | 9.5 |
| Local Sports | 92,690 | 1.060 s | 1.023 s | -3.5% | 10.6 | 10.2 |
| Local Technology | 93,065 | 1.113 s | 1.028 s | -7.7% | 9.3 | 10.4 |
| Local Arts & Culture | 93,135 | 1.098 s | 1.027 s | -6.5% | 10.1 | 10.6 |
| 平均（Q8: 14，Q4: 14†） | — | 1.040 s | 0.989 s | -4.8% | 10.6 | 10.8 |

† Q4 vs. Q8 的 Δ 基于同时有两种配置的 14 个主题计算。

**泛化总结。** 表 12 将可用指标与 Policygenius 基线进行总结。

**与 Policygenius 的比较。** Policygenius 基准（67K token）达到 0.857 s TTFT 中位数。RepLiQA 在相近大小上的主题
显示出按比例更高的 TTFT（88–93K token 为 1.02–1.11 s），这与解码期间上下文大小和注意力计算之间的预期线性关系
一致。

**表 12：泛化总结：RepLiQA（80 文档/主题）vs. Policygenius（30 文档）。质量优势逆转；计时优势保持。**

| 指标 | 均值 | 标准差 | 最小 | 最大 | Policygenius |
|---|---:|---:|---:|---:|---:|
| FC Q8 平均质量^a | 3.46 | 0.11 | 3.24 | 3.65 | 4.35 |
| FC Q4 平均质量^b | 3.34 | 0.13 | 3.09 | 3.61 | 4.38 |
| RAG 平均质量^c | 3.63 | 0.17 | 3.26 | 3.83 | 4.08 |
| 质量差距（FC Q8−RAG）^a | -0.17 | 0.12 | -0.39 | +0.04 | +0.27 |
| FC Q8 得分 1 率（%）^a | 17.3 | 2.5 | 12.2 | 22.8 | 1.2 |
| RAG 得分 1 率（%）^c | 17.7 | 4.4 | 12.5 | 27.5 | 5.8 |
| FC Q8 胜 / RAG 胜^a | 0 / 12（2 平） |  |  |  | FC 胜 |
| 中间丢失错误^a | 530 total（9.5%；全部 15 个 FC 主题为 557） |  |  |  | 3（1.2%） |
| FC TTFT Q8 中位数（s）^a | 1.040 | 0.109 | 0.672 | 1.113 | 0.857 |
| FC TTFT Q4 中位数（s）^b | 0.989 | 0.093 | 0.674 | 1.041 | 0.857 |
| Q4 TTFT Δ（%）^b | -4.8 | 2.7 | -8.7 | +0.3 | -2.2 |
| Q4 质量 Δ^d | -0.14 | 0.08 | -0.28 | +0.01 | +0.03 |
| RAG TTFT 中位数（s）^c | 4.825 | 0.223 | 4.279 | 5.152 | 6.277 |
| RAG 吞吐（tok/s）^c | 32.7 | 0.3 | 32.4 | 32.8 | 30.7 |
| 检索准确率（%）^c | 87.9 | 3.9 | 79.5 | 97.0 | 92.5 |
| FC/RAG TTFT 比例 | 4.6×（FC 更快） |  |  |  | 7.3× |

^a 14 个主题（Q8）。^b 14 个成对主题（Q4 vs. Q8 TTFT）。^c 17 个主题（RAG）。^d 14 个成对主题
（Q4 vs. Q8 质量）。

## 附录 H：Wiki 编译计时

编译显著降低上下文大小，带来成比例 TTFT 改善。表 13 总结了已完成主题上的计时结果。Wiki TTFT 与 wiki 大小几乎
完全相关（Pearson $r=0.995$）。

即使 wiki-light（0.38 s）也比 FC raw（1.06 s）快近 3 倍，比 RAG（4.82 s）快 12.7 倍。wiki-moderate 和
wiki-aggressive 收敛到相近 TTFT（约 0.2 s），因为它们的实际压缩比例接近（12.9% vs. 8.0%）。所有 wiki 条件
都实现亚 400 ms TTFT——完全处于对话延迟要求范围内。

**表 13：延迟比较：已编译 wiki vs. FC raw 与 RAG 基线。TTFT 和吞吐为已完成主题的中位数。**

| 条件 | 上下文 | TTFT | 吞吐 |
|---|---:|---:|---:|
| FC raw（80 文档） | ~70K 词 | 1.060 s | 10.4 tok/s |
| FC wiki-light | ~24K 词 | 0.380 s（快 2.8×） | 20.2 tok/s |
| FC wiki-moderate | ~8K 词 | 0.211 s（快 5.0×） | 26.2 tok/s |
| FC wiki-aggressive | ~6K 词 | 0.195 s（快 5.6×） | 27.2 tok/s |
| RAG | ~5K 词 | 4.818 s | 32.7 tok/s |

## 附录 I：CEGAR–WiCER 映射

第 7 节把 WiCER 描述为反例引导抽象精化（CEGAR）Clarke et al. [2000] 的一个实例。本附录详细说明形式化映射、
陈述单调性保证，并指出类比的边界。

### 形式化映射

在 CEGAR 中，一个具体系统被建模为 Kripke 结构 $M=(S,S_0,R,L)$，其中状态空间为 $S$，初始状态为 $S_0$，
转移关系为 $R\subseteq S\times S$，标记函数为 $L:S\to2^{AP}$。一个满射抽象 $h:S\to\hat{S}$ 诱导出抽象模型
$\hat{M}$。对 $\hat{M}$ 的模型检查要么验证规范，要么产生反例；如果该反例是伪反例（在 $\hat{M}$ 中有效但在
$M$ 中无效），则精化抽象以消除它。

表 14 展示对应关系。

**表 14：CEGAR 与 WiCER 的映射。**

| CEGAR | WiCER |
|---|---|
| 具体系统 $M$ | 完整文档集合 $D$ |
| 抽象模型 $\hat{M}$ | 已编译 wiki $W_t$ |
| 抽象 $h$ | LLM 编译器（有损压缩） |
| 规范 $\phi$ | “所有探针得分 >1” |
| 反例 | 得分 1 的失败探针 |
| 伪反例检查 | 诊断：事实存在于 $D$ 中但在 $W_t$ 中丢失 |
| 精化（分裂状态） | 添加固定约束，重编译 $W_{t+1}$ |
| 精化后的抽象 $\hat{M}'$ | 下一迭代 wiki $W_{t+1}$ |

### 单调性保证

令 $F_t \subseteq Q_{probe}$ 表示迭代 $t$ 时得分 ≤1 的探针集合，令 $P_t$ 表示迭代 $t$ 后的累积固定事实集合。
按照构造：

1. 每个固定事实都以逐字形式包含在 $W_{t+1}$ 的编译提示中，并带有显式保留约束。
2. 编译器必须在分配预算给一般覆盖之前包含所有固定事实。

因此，在关键事实已被固定的探针子集上，失败集合单调缩小：

$$
F_t \cap \{q:\mathrm{facts}(q)\subseteq P_t\}
\supseteq
F_{t+1} \cap \{q:\mathrm{facts}(q)\subseteq P_t\}.
$$

非形式化地说：一旦事实被固定，就不会再次丢失。

### 映射的局限

尽管结构相对应，仍有三点差异值得注意：

**随机编译器。** CEGAR 精化一个确定性抽象函数；WiCER 的“抽象”是 LLM 编译器，其输出是非确定性的。输入相同的两次
编译调用可能产生不同 wiki。实践中，贪婪解码和低温度在很大程度上缓解了这一点，但没有形式保证固定约束只会保留
目标事实而不产生副作用。

**近似验证。** CEGAR 精确地对抽象系统进行模型检查；WiCER 使用 LLM 评审评分，这是一种近似评估。得分 2/5 的探针
可能实际上代表灾难性失败，反之亦然。基于阈值的收敛准则（<10% 改进）通过要求稳健趋势而非个别分数，部分缓解了
这一问题。

**随机知识置换。** 在 CEGAR 中，精化抽象以消除伪反例不会在先前已验证性质上引入新的伪反例。在 WiCER 中，固定
事实会消耗部分 token 预算，可能挤出其他内容，并在先前通过的探针上创造新失败。这种“知识置换”效应是 WiCER 的
净改进在 1–2 次迭代后平台化的主要原因，也解释了为什么消融实验（第 7.5 节）显示随机固定可能损害质量。

这些差异意味着 WiCER 的收敛是经验性的而非形式化的：单调性保证适用于固定事实，但全局质量轨迹取决于诊断事实恢复
与随机置换之间的平衡。消融研究（第 7.5 节）提供了经验证据：有针对性的诊断持续优于随机固定，确认 CEGAR 启发的
反馈循环是 WiCER 收益来源。

## 附录 J：人类评估验证

为验证 LLM 评审分数，一名领域专家使用相同 1–5 分细则，独立评价了覆盖全部五种条件（FC raw、RAG、Wiki blind、
WiCER iter 1、WiCER iter 2）的 100 个问答对分层样本。表 15 总结一致性。

表 16 按条件分解一致性。评审在全部条件上都校准良好，每个条件 Pearson $r\ge0.89$，偏差可忽略
（$|\Delta mean|\le0.17$）。

唯一一个相差超过 1 分的样本是一个 RAG 响应，评审给 4 分、人类评分者给 2 分；人工检查确认该响应遗漏了人类认为
必要的关键细节。总体而言，这些结果确认 Claude Sonnet LLM 评审在该设置中为人类质量评估提供了可靠代理。

**表 15：LLM 评审 vs. 人类评估（$n=100$）。**

| 指标 | 数值 |
|---|---:|
| Pearson $r$ | 0.940 |
| Spearman $\rho$ | 0.928（$p<10^{-43}$） |
| Kendall $\tau$ | 0.873（$p<10^{-25}$） |
| 完全一致 | 75/100（75.0%） |
| 相差不超过 1 分 | 99/100（99.0%） |
| 平均绝对误差 | 0.26 |
| 偏差（LLM−Human） | +0.06 |

**表 16：按条件划分的 LLM 评审与人类评分者一致性。**

| 条件 | n | LLM 均值 | 人类均值 | Pearson $r$ | 完全一致 |
|---|---:|---:|---:|---:|---:|
| FC raw | 30 | 3.50 | 3.50 | 0.890 | 16/30 |
| RAG | 30 | 3.40 | 3.23 | 0.912 | 24/30 |
| Wiki blind | 31 | 2.26 | 2.19 | 0.975 | 27/31 |
| WiCER iter 1 | 6 | 2.00 | 2.17 | 0.937 | 5/6 |
| WiCER iter 2 | 3 | 3.00 | 3.00 | 1.000 | 3/3 |
