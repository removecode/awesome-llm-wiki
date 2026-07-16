---
title: "向量 RAG 与 LLM 编译 Wiki：小型多领域研究语料上的预注册比较"
original_title: "Vector RAG vs LLM-Compiled Wiki: A Preregistered Comparison on a Small Multi-Domain Research Corpus"
source_url: "https://arxiv.org/abs/2605.18490"
authors:
  - "Theodore O. Cochran"
---

> 本文为英文论文中文译本，仅供阅读参考。原文见 source_url。

# 向量 RAG 与 LLM 编译 Wiki：小型多领域研究语料上的预注册比较

Theodore O. Cochran  
AI for Altruism (A4A)  
theo@ai4altruism.org

## 摘要

我们预注册了一项比较，考察两种帮助 LLM 在小型研究语料上回答问题的方法：单轮向量 RAG
系统与由 LLM 编译的 Markdown Wiki。两个系统使用同一个答案生成模型，在 24 篇论文上回答
相同的 13 个问题，并由盲法 LLM 评审对答案打分。

Wiki 在跨论文连接发现方面得分高得多，但在评审调整后，它在答案组织方面的优势并不强。
RAG 达到了针对单一事实查找问题的预注册检验。清晰的查询侧成本结果与预期的 Wiki 优势
相反：在被测试的设置下，Wiki 使用的查询 token 远多于 RAG，因此无法通过更便宜的查询来
收回任何前期构建成本。

两项探索性分析改变了我们对结果的解释。第一，声明级引用核查更支持 Wiki：即使 RAG 在整体
`groundedness` 评分量表上得分更好，Wiki 所引用的页面也更常支持正在提出的精确声明。
第二，一种基于分解的 RAG 变体，以较低的 LLM-token 成本追回了 Wiki 在跨论文综合上的大部分
优势，但没有追回 Wiki 在逐声明引用支持方面的优势。

主要结论是，有根据的研究综合并不是一种单一能力。系统在组织证据的能力、其引用对每个声明
的支持程度以及运行成本上都可能不同。在本研究中，没有一种架构在三者上都最好。

## 1 引言

“LLM Wiki” 在摄取时把研究语料转换为一个交叉链接的 Wiki；在提问时，模型浏览这个 Wiki，
而不是从原始论文中检索原始 chunk。这一架构在非正式评论中得到普及，尤其是 Andrej Karpathy
提出的“agentic markdown wiki”表述。这个想法暗示了三种可能的权衡：Wiki 可能有助于多论文
综合；把论文重写为 Wiki 页面可能损失来源保真度；Wiki 可能把成本从查询时转移到摄取时，
并在足够多问题之后达到盈亏平衡。据我们所知，目前尚无已发表的、与向量 RAG 进行定量正面
比较的研究。[^1]

本文报告了一项预注册、盲法、双评审比较：在固定的 24 篇论文语料上，向量 RAG 与 LLM Wiki
回答跨越六个难度层级的 13 个评估问题。两个系统使用相同的查询模型（Claude Opus 4.7，
`xhigh`）；跨模型家族评审为 GPT-5.4，Gemini 2.5 Pro 用于评审间可靠性。语料、问题集、
评分量表、判定规则和贝叶斯模型都在任何评审运行之前，锁定在 OSF 预注册标签中。

[^1]: 我们于 2026-05-08 在 Google Scholar、Semantic Scholar、arXiv、ACL Anthology
    和 OpenReview 中搜索了 “LLM wiki”、“agentic markdown wiki”、“compiled wiki”、
    “RAG comparison” 和 “retrieval augmented generation wiki” 的组合。我们发现了相关的
    层级式、图式和摘要记忆工作（第 2 节），但没有发现关于 LLM 编译 Markdown Wiki 与
    chunk-向量 RAG 的预注册正面比较。

**贡献。**

- **预注册比较。** Wiki 在连接论文方面好得多；它在答案组织上的优势在评审调整后较弱，
  虽为正但低于注册的 H1 $+2.0$ 阈值（评分量表准则为 `inter_paper_mapping` 和
  `structural_integrity`）。RAG 达到了针对单一事实来源扎根性的预注册检验，即
  `groundedness`（在 IRR 调整下属于小效应且脆弱）。预期中的 Wiki 成本优势以相反方向失败：
  Wiki 每次查询约花费 RAG 的 $21\times$。
- **评估方法发现。** 两个 LLM 评审的行为不同。Gemini 2.5 Pro 在整体性准则上经常给出接近
  满分的分数，而 GPT-5.4 使用了更多评分刻度。评审在最具体的准则 `inter_paper_mapping`
  上一致性最高。一项事后声明级扎根性分析与评分量表的 `groundedness` 分数在方向上不一致，
  表明整体性扎根指标与逐引用扎根指标测量的是相关但不同的属性。
- **探索性 RAG 消融（单评审）。** 一种分解检索 RAG 变体追回了 Wiki 在跨论文综合上的大部分
  优势，在 H1 子集上弥合了约 $88\%$ 的差距，并把剩余的 Wiki 优势降到注册的 $+2.0$ 阈值
  以下。它**没有**追回 Wiki 在逐声明引用支持方面的优势。结果显示，单轮 RAG、decomp-RAG
  和 Wiki 在综合结构、声明-引用对齐以及成本之间存在三方权衡。

五个预注册运行产物、四个事后产物以及复现实验脚本分别存放在 OSF / GitHub 上。

## 2 相关工作

**RAG 与多跳检索。** 检索增强语言建模由 REALM [Guu et al., 2020] 和 Lewis et al.
[Lewis et al., 2020] 引入，并结合了密集段落检索 [Karpukhin et al., 2020]。我们的向量 RAG
是一个单轮“检索后生成”实例，包含多查询扩展、混合检索、Cohere 重排序以及受 CRAG
[Yan et al., 2024] 启发的纠正式验证；它不执行生成过程中的检索（参见 IRCoT
[Trivedi et al., 2023]、FLARE [Jiang et al., 2023]）。多跳基准（HotpotQA
[Yang et al., 2018]、MuSiQue [Trivedi et al., 2022]、MultiHop-RAG [Tang and Yang, 2024]）
表明，单轮检索不足以完成跨文档的关联推理。

**层级式与抽象式记忆。** LLM Wiki 架构最接近“编译式抽象记忆”：离线构建一个由 LLM 生成的
表示，并用它替代（或辅助）原始 chunk 进行查询。RAPTOR [Sarthi et al., 2024] 构建递归的
聚类摘要树；RECOMP [Xu et al., 2024] 压缩检索到的文档；GraphRAG [Edge et al., 2024]
构建带有社区摘要的实体图；知识图谱变体包括 REANO [Fang et al., 2024] 和 GNN-RAG
[Mavromatis and Karypis, 2025]。从概念上看，我们的 Wiki 是这一家族的 LLM 构建版本，具有
Markdown 渲染的中间表示。OpenScholar [Asai et al., 2026] 和 PaperQA2 [Skarlinski et al.,
2024] 在更大规模上应用了类似原语。

**引用忠实性与 LLM 作为评审。** ALCE [Gao et al., 2023] 为长篇答案引入了引用精确率/召回率
基准；Liu et al. [2023a] 发现商业生成式搜索系统经常产生流畅但无支持的答案；我们的声明级
扎根性分析遵循这一协议家族。Zheng et al. [2023] 记录了 LLM 评审中的位置、冗长和自我增强
偏差；G-Eval [Liu et al., 2023b] 报告了结构化表单填写下更好的人类一致性，但指出可能偏向
LLM 生成文本；Wang et al. [2024] 表明排名会随回答顺序改变。我们的 IRR 天花板评审发现
（第 5.3 节）与这些文献一致。

## 3 方法

**语料与问题。** 24 篇跨三个领域的同行评审论文（每个领域 8 篇：AI 伦理与法律、气候科学、
精准医学），时间范围为 2017--2026 年。13 个评估问题跨越六个层级：时间顺序、冲突、
多跳、涌现、政策（每类 $n = 2$，预注册为偏向 Wiki），以及偏差检查（$n = 3$，
预注册为偏向 RAG，并为 H2 提供额外功效）。纳入标准：同行评审或 arXiv 预印本研究论文；
排除标准：仅综述式摘要、书籍、非研究产物。带 DOI 的语料列表和完整问题文本见 OSF 存档。

**向量 RAG。** RAG 一次性检索段落，对其重排序并验证，然后撰写答案。实现包括：文档感知的
Markdown 标题 chunking，并进行 token 限制的二次切分（目标 512 token、50-token 重叠）；
逐 chunk 上下文增强；多查询扩展；密集+稀疏混合检索；Cohere 重排序到 top-5；受 CRAG
[Yan et al., 2024] 启发的纠正式验证；以及单轮答案生成。“单轮”意味着没有生成过程中的检索
（参见 [Trivedi et al., 2023, Jiang et al., 2023]）；除此之外，该流水线仍是多阶段的。

**LLM Wiki。** Wiki 系统首先离线把论文编译成持久的交叉链接 Markdown Wiki（实体、概念、
来源和分析，并带有交叉引用）。在提问时，一个使用工具的 agent 会列出页面、读取所选页面，
并通过三个工具提交答案（`list_pages`、`read_page`、`submit_answer`；`MAX_TURNS=30`，
查询侧无提示缓存）。两个较小的遥测注意事项（$\le 7\%$ 的 thinking-token 过计数；
查询侧放弃的缓存节省上界 $\le 6.5\%$）不会实质影响逐查询比较。

两个系统使用相同的查询模型（Claude Opus 4.7，`xhigh`）、语料和评估集；比较是完全交叉、
按问题配对的。这是对两种实用架构的比较，而不是对“知识组织”的干净因果隔离：系统在多个
轴线上不同（离线 LLM 重写；编译 Markdown 与向量库；工具循环与单轮检索；多调用上下文与
单调用上下文）。

**评审。** 主评审：GPT-5.4，中等推理，且与答案模型跨模型家族。IRR：Gemini 2.5 Pro，
相同提示，固定种子偏移。每个问题的 {RAG, Wiki} $\rightarrow$ {A, B} 盲法使用
`random.Random(seed=42)`（主评审）/ `(seed=43)`（次评审）。评审提示要求模型在每个准则上
独立给两个系统打分。评分量表：四个 1--10 的锚定准则，明确锚点为 1/5/10：
`groundedness`（声明可追溯到来源材料）、`structural_integrity`（统一叙事而非拼接片段）、
`conflict_awareness`（指出相互矛盾的发现）、`inter_paper_mapping`（跨 $\ge 2$ 篇论文的
多跳综合）。

**预注册与判定规则（转述）。**

- **H1（综合）：** 在 multi-hop + emergence 层级（$n = 4$）上，Wiki $-$ RAG 在
  `inter_paper_mapping` 和 `structural_integrity` 上均 $\ge +2.0$。如果二者都清晰达到，
  则支持；如果二者为正但至少一个低于阈值，则弱支持；如果任一为负，则驳斥。
- **H2（点来源）：** 在 bias-check 层级（$n = 3$）上，RAG $-$ Wiki 在 `groundedness`
  上 $\ge 0$。
- **H3（成本）：** 同时满足
  $T_{\mathrm{ingest}}[\mathrm{wiki}] > T_{\mathrm{ingest}}[\mathrm{rag}]$
  且
  $T_{\mathrm{query}}[\mathrm{rag}] > T_{\mathrm{query}}[\mathrm{wiki}]$。

预注册明确指出，主要分析是均值相对于幅度阈值的方向性比较，而不是频率学检验，因为 $n$ 很小，
且 LLM 分数具有非 i.i.d. 的误差结构。还预注册了两个稳健性检查（bootstrap 百分位 CI；
贝叶斯模型，$\mu \sim N(0, 4)$、$\sigma \sim \mathrm{HalfNormal}(4)$、NUTS）。

**IRR 触发调整。** 如果任何准则上的主评审--次评审最大分数差超过 2，则在 H1/H2 检验中，
该准则的 `mean_tier` 值会用两个评审逐问题分数的均值重新计算，而不是只使用主评审均值。

## 4 实验设置

**产物。** 五个预注册运行产物加四个事后产物（原始运行 + decomp 消融运行、评审和扎根性的
声明级扎根性分析）带 SHA-256 哈希存放在 OSF；复现实验脚本位于 GitHub。

**操纵检查。** 全部 26 个（问题、系统）单元格都使用 `claude-opus-4-7`，`xhigh`；零排除，
零摄取/查询/评审失败。

**成本核算范围。** H3 遵循预注册的 token 核算方式（摄取时为 prompt + completion 之和；
查询时为 prompt + completion + thinking），并排除 embedding 花费、reranker 花费、向量库
存储以及美元归一化。

**提示缓存注意事项（影响关键结论）。** Wiki 摄取产物把 `uncached`、`cache_creation` 和
`cache_read` 汇总到一个字段中。Anthropic 对 cache read 按约基础输入费率的 $10\%$ 收费，
因此，在缓存密集型工作负载上，粗略数值会把可计费成本高估一个数量级。因此，我们只报告
逐查询（H3b）上的 H3；它在两侧均以未缓存方式捕获。H3a 不依据存档产物裁定。

**稳健性检查（预注册）。** Bootstrap：95% 百分位 CI，每个“准则 $\times$ 层级”10,000 次重采样。
贝叶斯：每个“准则 $\times$ 层级”，
`score_diff` $\sim N(\mu, \sigma^2)$，$\mu \sim N(0, 4)$，
$\sigma \sim \mathrm{HalfNormal}(4)$，NUTS（4 条链 $\times$ 2,000 次迭代
$\times$ 2,000 次 warmup，`random_seed=42`）；所有拟合都满足预注册的收敛标准
（$\hat{R} \le 1.003$，最小 ESS $\ge 1,907$）。贝叶斯“强佐证”阈值为
$P(\mu \ge \theta) \ge 0.95$（H1 的 $\theta = +2$，H2 的 $\theta = 0$）。

**IRR 调整。** 预注册指定了 3 个问题的 IRR 子集；执行期间，我们把次评审扩展到所有 13 个问题，
这是一个已披露的、增加严谨性的偏离。在预注册的 $n = 3$ 和扩展后的 $n = 13$ 覆盖范围下，
所有四个准则都会触发最大差值规则；调整规则不受该偏离影响。

**事后声明级扎根性。** 对 26 个答案单元格中的每一个，Claude Opus 4.7（中等自适应 thinking）
把答案原子化为带有 `cited_source_idx` 的原子声明；对于每个有引用的声明，GPT-5.4
（中等、跨模型家族）基于被引用 chunk 打分为 `supported` / `partial` / `contradicted` /
`unsupported`。未引用声明自动记为 `unsupported`。完整提示见 OSF 补充材料。

## 5 结果

我们在**仅主评审**解读（IRR 前基线）和 **IRR 调整后**解读（评审均值重算；见第 3 节）下报告
每个假设。触发条件和调整规则均已预注册。IRR 调整后的解读更保守，也是我们标题性结论的依据。

### 5.1 验证性分析

**H1：综合优势（$n = 4$）。** 在 multi-hop + emergence 子集上，逐问题配对差值
（Wiki $-$ RAG）如下：

| 问题 | `structural_integrity` 主评审 | 次评审 | 均值 | `inter_paper_mapping` 主评审 | 次评审 | 均值 |
| --- | ---: | ---: | ---: | ---: | ---: | ---: |
| T3-mia-as-copyright-evidence | +2 | 0 | +1.0 | +8 | +7 | +7.5 |
| T3-rwd-validity-for-side-effects | +1 | 0 | +0.5 | +2 | +4 | +3.0 |
| T4-cross-domain-audit | +1 | +2 | +1.5 | +6 | +8 | +7.0 |
| T4-eelgrass-mrv-gaps | +4 | +3 | +3.5 | +9 | +9 | +9.0 |
| **均值** | **+2.000** | **+1.250** | **+1.625** | **+6.250** | **+7.000** | **+6.625** |

仅主评审：$\Delta_{\mathrm{struct}} = +2.000$（正好达到阈值），
$\Delta_{\mathrm{mapping}} = +6.250$（超过阈值）。两个准则均 $\ge +2.0$，因此支持。
IRR 调整后（标题性结论）：$\Delta_{\mathrm{struct}} = +1.625$（差 0.375 未达到），
$\Delta_{\mathrm{mapping}} = +6.625$（超过阈值），因此根据预注册的三分判定规则为**弱支持**。
`inter_paper_mapping` 优势很大，从未接近阈值；`structural_integrity` 优势对阈值敏感。

**H2：点来源平价（$n = 3$）。** 在 bias-check 层级上，`groundedness` 的逐问题配对差值
（RAG $-$ Wiki）如下：

| 问题 | 主评审 | 次评审 | 均值 |
| --- | ---: | ---: | ---: |
| B1-devote3-confidence-interval | +1 | +3 | +2.0 |
| B2-he-tyka-equilibration-ratios | +3 | -6 | -1.5 |
| B3-mia-attack-success-rate | +2 | +1 | +1.5 |
| **均值** | **+2.000** | **-0.667** | **+0.667** |

B2 的次评审条目是本实验中最大的单次评审分歧：GPT-5.4 给 RAG=9 / Wiki=6，
Gemini 2.5 Pro 给 RAG=4 / Wiki=10。尽管如此，两个解读都返回 $\Delta \ge 0$，
满足预注册判定规则。仅主评审：$\Delta = +2.000$，因此支持。IRR 调整后：
$\Delta = +0.667$，因此支持。H2 在注册的评分量表指标上得到支持；第 5.4 节报告的事后分析
细化了其底层机制。

**H3：成本不对称。** 预注册预期 Wiki 构建昂贵但查询便宜。清晰的查询侧数据表现为相反结果：
Wiki 的每次查询成本更高，因此摊销叙事无法成立。查询侧检验（H3b）可直接由逐查询遥测裁定；
摄取侧检验（H3a）由于我们事后发现的提示缓存核算问题，**不依据存档产物裁定**（见第 7 节）。

| 数量（13 个问题求和） | 值 |
| --- | ---: |
| $T_{\mathrm{query}}[\mathrm{rag}]$（prompt + completion + thinking） | 78,093 |
| $T_{\mathrm{query}}[\mathrm{wiki}]$（prompt + completion + thinking） | 1,651,357 |
| H3b 支持 $(T_{\mathrm{query}}[\mathrm{rag}] > T_{\mathrm{query}}[\mathrm{wiki}])$ | False（相反方向 $21\times$） |

Wiki 查询侧 harness 明确禁用提示缓存（伴随材料 §4.3）；上面的逐查询数值是未缓存输入 +
completion + thinking，且等价于可计费量。RAG 的逐查询同样未缓存。$21\times$ 的逐查询差距
没有歧义。

H3 作为合取假设要求 H3a 和 H3b 同时得到支持，因此 H3b 的相反方向结果本身就足以驳斥它。
Wiki 成本直觉是：预编译支付大额前期费用，以换取更便宜的查询，并在某个 $N$ 处盈亏平衡。
实际情况相反：用户每次查询支付**更多**，而非更少，因此在观察到的逐查询成本下，预注册的
Wiki 摊销机制不可能成立。预注册的交叉查询公式分母
$(T_{\mathrm{query}}[\mathrm{rag}] - T_{\mathrm{query}}[\mathrm{wiki}]) / 13 = -121,020$
为负；公式返回负的 $N$，在数学上编码为“没有正查询数交叉点”。

我们不引用摄取成本比率。Wiki 侧摄取遥测把 `uncached + cache_creation + cache_read` token
聚合到单个 `tokens_in` 字段；Anthropic 对 cache read 按约基础输入费率的 $10\%$ 收费，因此
按表面值相加会在缓存密集型工作负载上把可计费成本高估一个数量级或更多。没有四列分解
（`uncached` / `cache_create` / `cache_read` / `output`），我们无法重建可计费成本。**H3a
不依据存档产物裁定。** H3 结论仅依靠 H3b，而 H3b 在合取上已经足够。

### 5.2 稳健性检查（预注册）

**表 1：** Bootstrap 百分位 CI（10,000 次重采样）与每个 H1/H2 单元格的预注册贝叶斯后验
（$\mu \sim N(0, 4)$、$\sigma \sim \mathrm{HalfNormal}(4)$、NUTS）。“强”表示按照预注册的
贝叶斯阈值 $P(\mu \ge \theta) \ge 0.95$（H1 的 $\theta = +2$；H2 的 $\theta = 0$）。

| 检验 | 解读 | 均值 | bootstrap 95% CI | 后验 $\mu$ | $P(\mu \ge \theta)$ | 强？ |
| --- | --- | ---: | --- | ---: | ---: | --- |
| H1 struct | 评审均值 | +1.625 | [+0.75, +2.88] | +1.500 | 0.283 | 否 |
| H1 mapping | 评审均值 | +6.625 | [+4.13, +8.50] | +5.589 | 0.967 | 是 |
| H1 struct | 主评审 | +2.000 | [+1.00, +3.25] | +1.819 | 0.440 | 否 |
| H1 mapping | 主评审 | +6.250 | [+3.50, +8.50] | +5.060 | 0.938 | 接近 |
| H2 ground | 评审均值 | +0.667 | [-1.50, +2.00] | +0.522 | 0.660 | 否 |
| H2 ground | 主评审 | +2.000 | [+1.00, +3.00] | +1.808 | 0.937 | 接近 |

只有 H1 `inter_paper_mapping`（评审均值）超过预注册的“强佐证”门槛
$P(\mu \ge \theta) \ge 0.95$。贝叶斯后验把点估计向零收缩（例如 H1 主评审
`structural_integrity` 的样本均值 $+2.000$ 收缩为后验 $+1.819$），反映了小 $n$ 下正则化先验
的作用。两个稳健性检查在方向上一致，但在阈值**多久/多稳健地**被超过上不一致。

### 5.3 评审间可靠性：天花板评审发现

预注册的 IRR 规则在全部四个准则上触发；在扩展后的 $n = 13$ 次评审覆盖中，各准则最大差值为
6（`groundedness`）、4（`structural_integrity`）、8（`conflict_awareness`）、6
（`inter_paper_mapping`）。若仅在预注册的 $n = 3$ IRR 子集上计算，最大差值为 6 / 4 / 4 / 3，
仍在全部四个准则上触发。

分歧主要关乎**水平**，而非**排序**。两个评审在 H1 验证性子集上每个准则级差距的符号都一致，
并且在全部 13 个问题的四个准则级均值中有三个一致。`groundedness` 是例外：在全部 13 个问题
上，GPT-5.4 给 RAG 更高分（7.00 vs 6.31），而 Gemini 2.5 Pro 给 Wiki 更高分（8.15 vs 9.69）。
H2 子集（bias-check `groundedness`）表现出同样的水平脆弱性：主评审 $\Delta = +2.000$，
次评审 $\Delta = -0.667$，评审均值 $\Delta = +0.667$，在注册判定规则下为支持，但仅次评审
会驳斥。这强化了如下解释：整体性 `groundedness` 是对评审校准最敏感的准则。

**表 2：** 全部 13 个问题上，每个评审、每个系统的均值。Gemini 2.5 Pro 对 Wiki 的评分除
`inter_paper_mapping` 外接近 10/10；后者是唯一具有具体操作定义的准则。

| 评审 | RAG g | Wiki g | RAG s | Wiki s | RAG c | Wiki c | RAG m | Wiki m |
| --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| GPT-5.4（主评审） | 7.00 | 6.31 | 6.85 | 8.92 | 6.77 | 8.38 | 5.38 | 9.46 |
| Gemini 2.5 Pro（次评审） | 8.15 | 9.69 | 8.15 | 9.85 | 8.54 | 9.62 | 5.23 | 10.00 |

Gemini 2.5 Pro 对 Wiki 输出的评分在四个准则中的三个上接近 10/10 饱和。GPT-5.4 的评分分布在
6 到 9 之间，方差更大。例外是 `inter_paper_mapping`：两个评审的 RAG 均值相差 0.15
（5.38 vs 5.23），两个评审的 Wiki 均值相差 0.54（9.46 vs 10.00）。两个评审在操作定义最具体
的准则上最接近。我们把它作为可迁移的方法学发现提出：**具有具体操作定义的评分准则会表现出
评审间收敛；依赖整体风格评估的准则会表现出天花板评审校准漂移**，这与既有 LLM-as-judge 文献
[Zheng et al., 2023, Liu et al., 2023b, Wang et al., 2024] 一致。

### 5.4 事后声明级扎根性

我们对全部 466 个原子声明运行了事后声明级扎根性分析。**该分析没有预注册**，作为 H2 机制的
探索性稳健性分析报告。

**重要解释注意事项。** 对 RAG 声明而言，被引用证据是检索系统给出的原始 PDF chunk；对 Wiki
声明而言，被引用证据是 Wiki 页面摘录，而该页面本身是来源 PDF 的编译物，并非原始 PDF 段落。
因此，下面的分析测试的是**证据产物--声明对齐**（每个声明是否可由系统所引用的内容推出？），
而不是**原始来源保真度**（每个声明是否可由底层 PDF 推出？）。想要了解 Wiki 来源保真度的读者
还必须追问：Wiki 页面本身是否准确反映来源 PDF？我们没有测量这一点；第 7 节把它标为首要
后续工作。

**按系统聚合。** 总声明数：RAG 150，Wiki 316。引用率（具有非空 `cited_source_idx` 的声明）：
RAG 88.0%，Wiki 76.3%。Wiki 每个答案产生约两倍的声明（24.3 vs 11.5），并引用其中更小比例。

把未引用声明视为 `unsupported`（**全声明**视角）时，标题性比率为：RAG 16.7% supported /
42.0% unsupported；Wiki 30.7% / 28.5%。以“有引用为条件”的视角（引用精确率意义）如下：

| 判定（以有引用为条件） | RAG（$n = 132$） | Wiki（$n = 241$） |
| --- | ---: | ---: |
| supported | 18.9%（macro 18.4%） | 40.2%（macro 37.3%） |
| partial | 44.7% | 53.1% |
| contradicted | 2.3% | 0.4% |
| unsupported | 34.1%（macro 35.8%） | 6.2%（macro 8.0%） |

同时报告按问题的宏平均（13 个答案各自等权，不受声明数影响）和微平均，以回应长 Wiki 答案可能
抬高 Wiki 逐声明比率的担忧。定性结论不变：Wiki 的有引用声明获得支持的频率约为 RAG 的
$2\times$，无支持的频率约低 $4$--$5\times$。

实验中 4 个直接矛盾声明里，3 个来自 RAG（B1：被引用表格与答案之间的 HR/CI 不匹配；
T4-cross-domain：特征漂移方向反转；T5-glp1：对论文内容的错误描述），1 个来自 Wiki
（T2-oae：把 “in 95%” 表述为 “over 95%”，属于单词级转述伪影）。

**按层级。** 最大差距出现在 **bias-check** 层级，也就是 H2 的主场：评分量表
$\Delta_{\mathrm{groundedness}} = -2.00$（RAG 领先，注册指标），但声明级 supported% 为
RAG 5.3% 对 Wiki 51.9%。两个指标在同一层级上指向相反方向（表 3）。

**表 3：** 按层级的声明级扎根率，以有引用为条件。†最右列：评分量表 `groundedness` 差值，
按 Wiki $-$ RAG 计算（正值 = Wiki 在评分量表上领先）。

| 层级 | RAG sup% | Wiki sup% | RAG unsup% | Wiki unsup% | 评分量表 $\Delta$ ground† |
| --- | ---: | ---: | ---: | ---: | ---: |
| bias-check | 5.3% | 51.9% | 78.9% | 7.4% | -2.00 |
| chronological | 8.0% | 50.8% | 60.0% | 1.6% | -0.50 |
| conflict | 22.7% | 42.9% | 9.1% | 8.6% | +2.00 |
| emergence | 25.0% | 37.5% | 30.0% | 0.0% | -1.00 |
| multi-hop | 32.0% | 35.6% | 20.0% | 11.1% | -3.50 |
| policy | 19.0% | 22.0% | 9.5% | 9.8% | +1.50 |

**绝对比率注意事项。** 这些比率不应被解读为高绝对引用可靠性：即便是 Wiki 的有引用声明，
也更常是 `partial` 而非严格 `supported`（全部 13 个问题为 53.1% vs 40.2%）。该结果是在严格
supported/partial 边界下的相对架构比较，而不是关于高引用精确率的基准级声明。

**解读。** 两个指标测量的是相关但不同的属性。评分量表评审评估整体性的答案--证据痕迹，并奖励
RAG 简短、引用密集、锚定于逐字 chunk 文本的答案。声明级分析评估**严格的逐引用匹配**，并捕捉
到 RAG 检索一个 chunk 后又综合/外推超过该 chunk 的倾向，包括 B1 的置信区间错误：chunk 报告
$\mathrm{HR}=1.37$，但答案写为 $\mathrm{HR}=1.38$。观察到的模式与这样一种解释一致：Wiki
编译把证据预先放置在声明形态的产物中（因此，当答案引用某个 Wiki 页面时，被引用页面通常直接
包含支持句）；不过，我们并没有把标题/声明对齐作为独立变量来测量；这并不等同于来源保真度。

这并不推翻预注册的 H2 结论。H2 是关于评分量表 `groundedness` 的注册检验，该结论
（RAG $\ge$ Wiki）在注册指标上成立。但它确实实质性地细化了底层**机制**：预注册推测 Wiki 的
有损摘要会损失点来源保真度；在声明级审查下，RAG 的 chunk-检索-再综合流水线显示出比 Wiki 的
编译-遍历流水线更差的**证据产物对齐**。Wiki 的编译步骤是否以高于 RAG 检索步骤的比率保留
来源保真度，是一个独立问题，本分析不予裁定。

方法学教训是：**评分量表式的整体扎根性分数与声明级引用对齐可能在方向上相互矛盾，而不只是
幅度不同。** RAG/Wiki 评估应同时报告二者。

### 5.5 其他预注册探索性分析

预注册列出了七项计划中的次要分析，但它们不具裁定性。我们在此总结；完整数据见 OSF 补充材料。

- **偏差标签分层。** 问题分类成立：`bias=rag` 问题显示 RAG 在 `groundedness` 上赢 2.0；
  `bias=wiki` 问题显示 Wiki 在全部四个准则上获胜（最大为 `inter_paper_mapping` 的 +5.75）。
- **延迟。** RAG 总计 3.3 分钟，Wiki 总计 22.0 分钟（$6.6\times$）；墙钟时间比率低于 token
  比率，因为自适应 thinking 与输入处理重叠。
- **自适应 thinking 参与。** RAG 在 2/13 个问题上触发非零 `thinking_tokens`；Wiki 在 13/13
  个问题上触发。二者使用相同的 `xhigh` 标志：任务形态，而非标志，驱动参与。
- **成本--质量散点。** RAG 聚类（$\le 8k$ token，分数 3.75--8.25）和 Wiki 聚类
  （约 30k--210k token，分数 6.75--9.25）并不存在全局支配关系。在 13 个问题中的 4 个上，
  RAG 以低一个数量级的成本在评分量表均值上追平或击败 Wiki（T1-cardio、T3-rwd、B2、B3）。
  帕累托前沿依赖任务。

## 6 探索性 decomp-RAG 消融

一位评审提出了这样的假设：单轮检索是多跳综合中的已知弱点
[Trivedi et al., 2023, Jiang et al., 2023]，因此 Wiki 的 H1 优势可能部分是注册 RAG 配置的
产物。我们用一项事后的分解检索消融测试这一点。

**方法。** 对每个问题：Claude Opus 4.7 把它分解为 2--5 个子问题；现有单轮 RAG 流水线
（第 3 节）针对每个子问题运行，只保留有引用来源；引用 chunk 去重；Claude Opus 4.7 使用与
单轮基线**相同**的答案生成系统提示（逐字来自代码库）生成最终答案。查询模型、语料、向量库和
reranker 相同；唯一变化是“问题到检索上下文”这一步。在运行前提交了三个预测：**P1** decomp
在 `inter_paper_mapping` 上弥合 Wiki--单轮 RAG 差距的 $\ge 50\%$；**P2** 逐查询 token 成本
介于单轮的 $5$--$10\times$；**P3** 声明级 supported-rate 在 multi-hop 上提高。GPT-5.4 使用
相同提示、评分量表和种子（42）在（decomp, wiki）配对上重新调用。**decomp 消融只由 GPT-5.4
评分；我们没有重新运行 Gemini 2.5 Pro。** 鉴于原始分析因评审校准漂移触发了完整 IRR 调整
（第 5.3 节），下面的 decomp 结果应被视为探索性且对评审敏感，而不是预注册的重新检验。

**三方结果。成本（P2）。** Decomp 的逐查询 LLM-token 成本（按 H3 token 核算范围，排除 reranker
和 embedding 花费）是单轮的 $6.3\times$，并比 Wiki 便宜 $3.4\times$，满足 P2。墙钟时间却比
Wiki **更慢**，因为 decomp 对每个子问题顺序执行检索、重排序和验证调用；LLM-token 图景对
decomp 有利，墙钟图景不利，而 embedding/reranker 成本图景未在本研究中测量。

**表 4：** 三方比较摘要。成本行对 13 个问题求和。综合形态行使用 H1 验证性子集（$n = 4$）；
“解读”列报告从原始评审运行到 decomp 评审运行的评审内收缩。声明级行使用全部 13 个问题中
有引用的声明。a 单轮 RAG，在原始评审运行中评分。b Decomp-RAG，在事后 decomp 评审运行中
用相同提示、评分量表和种子评分。c Wiki：两次评审运行范围（相同答案，评分略有不同）。

| 指标 | 单轮 a | Decomp b | Wiki c | 解读 |
| --- | ---: | ---: | ---: | --- |
| 总查询 token | 78k | 491k | 1,651k | decomp 在 LLM token 上比 Wiki 便宜 $3.4\times$ |
| 总墙钟时间（分钟） | 3.3 | 26.8 | 22.0 | decomp 比 Wiki 慢（顺序执行） |
| H1 `inter_paper_mapping` 均值 | 3.50 | 9.00 | 9.00--9.75 | Wiki 差距从 +6.25 收缩到 +0.75 |
| H1 `structural_integrity` 均值 | 7.00 | 8.75 | 8.92--9.00 | Wiki 差距从 +2.00 收缩到 +0.25 |
| 有引用声明 supported % | 18.9 | 19.2 | 40.2 | decomp **没有**弥合引用差距 |

**综合结构（P1）。** 在 H1 验证性子集上，decomp 运行支持 P1。Wiki 在
`inter_paper_mapping` 上的优势从 $+6.25$ 收缩到 $+0.75$（弥合约 $88\%$ 差距），在
`structural_integrity` 上从 $+2.00$ 收缩到 $+0.25$（约 $87.5\%$）。相对于 decomp 基线，
两个准则都低于预注册的 $+2.0$ 阈值；在这一比较中，Wiki 不会超过注册的 H1 阈值。**这并不改变
相对于单轮 RAG 的预注册 H1 结论。** 它表明，综合结构优势对 RAG 基线敏感。Decomp 在整体性
`groundedness` 上也优于 Wiki（全部 13 个问题为 +1.15；multi-hop 层级特别为 +3.00），并在
bias-check `groundedness` 上满足 H2 平价检验（+2.33）。

**声明级扎根性（P3 部分）。** 我们在 232 个 decomp 声明上重新运行了 §5.4 的原子化与打分流水线
（94.4% 有引用，相比单轮 88.0%、Wiki 76.3%）。Decomp 的整体有引用声明 supported-rate 为
19.2%，基本等于单轮（18.9%），约为 Wiki（40.2%）的一半。Decomp 相对于单轮的声明级优势集中在
bias-check（5.3 $\rightarrow$ 43.5%，修复了 B1 置信区间错误等 chunk 重组错误），在 multi-hop
上较小（32.0 $\rightarrow$ 37.0%）。在 chronological / conflict / emergence / policy 上，
decomp 与单轮相似或略低。

**解读。** 这一模式与 Wiki 优势背后的两个可分离机制一致。**检索覆盖**在该消融中被分解检索
大幅缓解：当 RAG 把每个问题拆成子问题时，它追回了 Wiki 在跨论文综合上的大部分优势。
**表示对齐**并未被更广检索弥合：Wiki 在引用可直接支持具体声明的页面方面仍然更好。一个合理
解释是，Wiki 页面预先把证据放置在声明形态的产物中，而 RAG chunk 是与**查询**对齐，并不与
每个下游答案声明对齐。我们没有直接测量这一机制。未来结合迭代检索和基于声明的事后引用修复
工作，可以测试是否也能弥合对齐差距。

**注意事项总结。** 单评审（消融无 IRR）；跨运行 Wiki 绝对分数在任何准则上相差 $\le 0.6$
（表 4 的评审内差值对此稳健）；H1 子集 $n = 4$。因为消融是探索性的且小 $n$，我们只报告点估计，
并把它们视为机制证据而非验证性推断；一次分解提示在运行前固定。复现脚本：
`experiments/run_decomp_rag.py` + `run_claim_grounding_decomp.py`。

## 7 讨论与局限

**发现意味着什么。** Wiki 优势似乎可分解为两个可分离机制。Wiki 综合形态优势中的**检索覆盖**
部分，在我们的消融中被分解检索大幅缓解（第 6 节）。**表示对齐**部分（声明级引用精确率准则）
并未被更广检索弥合。观察到的模式与这样一种解释一致：Wiki 编译把证据预先放在声明形态的产物中
（因此，当答案引用 Wiki 页面时，被引用页面通常直接包含支持句），而 RAG 检索使 chunk 与
**查询**对齐，而不是与答案最终提出的每个具体声明对齐。我们没有把标题/声明对齐作为独立变量
来测量；这是与数据一致的机制假设，而非已证明的因果解释。

Karpathy 的三条直觉部分得到印证。（i）预编译有助于综合（在注册基线上得到支持；可被 decomp
大幅缓解）。（ii）预编译可能牺牲点来源保真度（在评分量表上得到支持；在声明级审查下反转）。
其中相关比较是证据产物对齐，而不是原始 PDF 保真度；后者我们没有测量。（iii）预编译把成本从
查询转移到摄取，并存在有限 $N$ 的摊销（被驳斥：逐查询成本**上升**，而非下降）。

**何时选择哪一种。** 单轮 RAG 适合成本敏感的点来源检索。Decomp-RAG 适合多论文综合中结构
很重要、且预算能承受约 $6\times$ 单轮 LLM-token 成本的场景（不含多子问题检索带来的额外
reranker/embedding 花费）；本研究只在一个操作点上测试。Wiki 适合证据产物的声明--引用对齐，
尽管其逐查询 token 溢价约为 $21\times$，且仍待来源 PDF 验证。H3 的逐查询结果从逐查询成本
角度排除了 Wiki 的规模化使用，无论 $N$ 为何。

**这些发现没有说明什么。** **没有人类评估**（最关键缺口）：全部评分量表打分、声明级打分和
decomp 评审都基于 LLM；IRR 发现（第 5.3 节）表明，LLM 评审校准漂移很大，并会在整体性准则上
造成方向翻转。人类验证是最高优先级的未来工作；在此之前，声明级分析应被解读为 LLM 打分的
证据产物对齐，而非人类验证的来源保真度。§6 的 decomp 消融同样应被视为探索性且对评审敏感
（单评审、无 IRR）。**小 $n$**（H1 为 $n = 4$，H2 为 $n = 3$）：标准误很大，decomp 消融只报告
点估计。**单一语料与单一查询模型**：24 篇论文、前沿 LLM；向大语料或不同模型家族迁移未经验证。
**风格混杂**：Wiki 产出连续散文，RAG 产出项目符号加引用；未控制。**一个分解器提示**在运行前
固定，未迭代。**提示缓存下的成本遥测**：Wiki 摄取的 `tokens_in` 字段把 uncached +
cache_creation + cache_read 按表面值聚合；cache read 按约基础输入费率的 $10\%$ 计费，因此
粗略数值把可计费成本高估了一个数量级。H3b 结论不受影响（查询侧缓存两边均禁用）。**给复现者的
方法注记**：缓存与非缓存成本比较研究必须为每次 LLM 调用捕获四列（uncached / cache_creation /
cache_read / output），而不是汇总的 `tokens_in`。

**超出本研究。** 未来工作应把声明级扎根性与原始 PDF 段落比较（测试 Wiki 的来源保真度），
而不仅仅与被引用证据产物比较；在 decomp 配对上运行 Gemini 2.5 Pro 以增加 IRR 覆盖；测试
生成过程中的检索方法 [Trivedi et al., 2023, Jiang et al., 2023]，它们可能弥合更多 Wiki 优势；
并测试混合 Wiki+source-RAG 架构，后者可以用来源 chunk 验证来补强 Wiki 偶发转述伪影
（4 个矛盾声明中的 1 个）。

## 8 结论

这项预注册比较最站得住脚的结论，并不是编译式 Wiki 记忆胜过 RAG，也不是相反。扎根的研究综合
并不是一种单一能力：系统可以很好地组织证据、很好地为每个具体声明引用证据，或便宜地运行；
在本实验中，没有任何架构在三者上都最好。单轮 RAG 最小化成本；decomp-RAG 以约 $3.4\times$
更低的逐查询 LLM-token 成本，在综合形态评分准则上接近 Wiki；Wiki 保持最强的 LLM 打分证据
产物声明--引用对齐，但逐查询成本高，并仍待来源 PDF 验证。未来的小语料 RAG/Wiki 评估应分别
报告综合结构、声明--引用对齐和成本，而不是把它们折叠成单一的扎根综合结论。

## References

Akari Asai, Jacqueline He, Rulin Shao, Weijia Shi, et al. Synthesizing scientific literature with
retrieval-augmented language models.Nature, 2026.

Darren Edge, Ha Trinh, Newman Cheng, Joshua Bradley, Alex Chao, Apurva Mody, Steven
Truitt, and Jonathan Larson. From local to global: A Graph RAG approach to query-focused
summarization, 2024. arXiv preprint, Microsoft Research technical report.

Jinyuan Fang, Zaiqiao Meng, and Craig Macdonald. REANO: Optimising retrieval-augmented
reader models through knowledge graph generation. InProceedings of the 62nd Annual Meeting
of the Association for Computational Linguistics (ACL), 2024.

Tianyu Gao, Howard Yen, Jiatong Yu, and Danqi Chen. Enabling large language models to generate
text with citations. InProceedings of the 2023 Conference on Empirical Methods in Natural
Language Processing (EMNLP), 2023.

Kelvin Guu, Kenton Lee, Zora Tung, Panupong Pasupat, and Ming-Wei Chang. REALM: Retrieval-
augmented language model pre-training. InProceedings of the 37th International Conference on
Machine Learning (ICML), 2020.

Zhengbao Jiang, Frank F. Xu, Luyu Gao, Zhiqing Sun, Qian Liu, Jane Dwivedi-Yu, Yiming Yang,
Jamie Callan, and Graham Neubig. Active retrieval augmented generation. InProceedings of the
2023 Conference on Empirical Methods in Natural Language Processing (EMNLP), 2023.

Vladimir Karpukhin, Barlas Oğuz, Sewon Min, Patrick Lewis, Ledell Wu, Sergey Edunov, Danqi
Chen, and Wen tau Yih. Dense passage retrieval for open-domain question answering. InProceed-
ings of the 2020 Conference on Empirical Methods in Natural Language Processing (EMNLP),
2020.

Patrick Lewis, Ethan Perez, Aleksandra Piktus, Fabio Petroni, Vladimir Karpukhin, Naman Goyal,
Heinrich Küttler, Mike Lewis, Wen tau Yih, Tim Rocktäschel, Sebastian Riedel, and Douwe
Kiela. Retrieval-augmented generation for knowledge-intensive NLP tasks. InAdvances in Neural
Information Processing Systems (NeurIPS), 2020.

Nelson F. Liu, Tianyi Zhang, and Percy Liang. Evaluating verifiability in generative search engines.
InFindings of the Association for Computational Linguistics: EMNLP, 2023a.

Yang Liu, Dan Iter, Yichong Xu, Shuohang Wang, Ruochen Xu, and Chenguang Zhu. G-Eval: NLG
evaluation using GPT-4 with better human alignment. InProceedings of the 2023 Conference on
Empirical Methods in Natural Language Processing (EMNLP), 2023b.

Costas Mavromatis and George Karypis. GNN-RAG: Graph neural retrieval for efficient large
language model reasoning on knowledge graphs. InFindings of the Association for Computational
Linguistics (ACL Findings), 2025.

Parth Sarthi, Salman Abdullah, Aditi Tuli, Shubh Khanna, Anna Goldie, and Christopher D.
Manning. RAPTOR: Recursive abstractive processing for tree-organized retrieval. InInternational
Conference on Learning Representations (ICLR), 2024.

Michael D. Skarlinski, Sam Cox, Jon M. Laurent, James D. Braza, Michaela Hinks, Michael J.
Hammerling, Manvitha Ponnapati, Samuel G. Rodriques, and Andrew D. White. Language
agents achieve superhuman synthesis of scientific knowledge (PaperQA2), 2024. arXiv preprint,
FutureHouse technical report.

Yixuan Tang and Yi Yang. MultiHop-RAG: Benchmarking retrieval-augmented generation for
multi-hop queries. InConference on Language Modeling (COLM), 2024.

Harsh Trivedi, Niranjan Balasubramanian, Tushar Khot, and Ashish Sabharwal. MuSiQue: Multihop
questions via single-hop question composition.Transactions of the Association for Computational
Linguistics, 2022.

HarshTrivedi, NiranjanBalasubramanian, TusharKhot, andAshishSabharwal. Interleavingretrieval
with chain-of-thought reasoning for knowledge-intensive multi-step questions. InProceedings of
the 61st Annual Meeting of the Association for Computational Linguistics (ACL), 2023.

Peiyi Wang, Lei Li, Liang Chen, Zefan Cai, Dawei Zhu, Binghuai Lin, Yunbo Cao, Qi Liu, Tianyu
Liu, and Zhifang Sui. Large language models are not fair evaluators. InProceedings of the 62nd
Annual Meeting of the Association for Computational Linguistics (ACL), 2024.

Fangyuan Xu, Weijia Shi, and Eunsol Choi. RECOMP: Improving retrieval-augmented LMs
with context compression and selective augmentation. InInternational Conference on Learning
Representations (ICLR), 2024.

Shi-Qi Yan, Jia-Chen Gu, Yun Zhu, and Zhen-Hua Ling. Corrective retrieval augmented generation
(CRAG), 2024. arXiv preprint.

Zhilin Yang, Peng Qi, Saizheng Zhang, Yoshua Bengio, William W. Cohen, Ruslan Salakhutdinov,
and Christopher D. Manning. HotpotQA: A dataset for diverse, explainable multi-hop question
answering. InProceedings of the 2018 Conference on Empirical Methods in Natural Language
Processing (EMNLP), 2018.

Lianmin Zheng, Wei-Lin Chiang, Ying Sheng, Siyuan Zhuang, Zhanghao Wu, Yonghao Zhuang,
Zi Lin, Zhuohan Li, Dacheng Li, Eric P. Xing, Hao Zhang, Joseph E. Gonzalez, and Ion Stoica.
Judging LLM-as-a-Judge with MT-Bench and Chatbot Arena. InAdvances in Neural Information
Processing Systems (NeurIPS), 2023.
