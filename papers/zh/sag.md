---
title: "SAG：基于 SQL 检索增强生成与查询时动态超边"
original_title: "SAG: SQL-Retrieval Augmented Generation with Query-Time Dynamic Hyperedges"
source_url: "https://arxiv.org/abs/2606.15971"
authors:
  - Yuchao Wu
  - Junqin Li
  - XingCheng Liang
  - Yongjie Chen
  - Yinghao Liang
  - Linyuan Mo
  - Guanxian Li
---

> 本文为英文论文中文译本，仅供阅读参考。原文见 source_url。

# SAG：基于 SQL 检索增强生成与查询时动态超边

Yuchao Wu∗, Junqin Li, XingCheng Liang, Yongjie Chen, Yinghao Liang  
Linyuan Mo, Guanxian Li  
Zleap AI  
{jomy,junqing,lensen,jinzhoulawen,leo}@zleap.com  
{mo-linyuan,li guanxian}@foxmail.com

∗ 通信作者：jomy@zleap.com。

## 摘要

检索增强生成（Retrieval-Augmented Generation, RAG）为大语言模型访问外部知识提供了一种有效方法。
然而，现有方法依赖稠密相似度检索，在处理结构化约束和多跳推理时存在固有限制。引入知识图谱可以
部分缓解这些问题，但代价是语义碎片化、维护开销高，以及增量更新困难。本文提出 SAG
（SQL-Retrieval Augmented Generation），一种面向检索与智能体系统的结构化架构。SAG 不预先构建
全局静态图，而是把每个块转换为一个语义完整的事件和一组用于索引的实体，然后使用 SQL 连接查询，将
共享实体的事件动态链接为局部超边，从而在查询时构建一个动态实例化的局部索引结构。这一设计避免了
全局图重建和持续维护的需求；由于依赖标准数据库基础设施，系统天然支持增量写入、并发处理和持续扩展。
在 HotpotQA、2WikiMultiHop 和 MuSiQue 三个标准多跳基准上，SAG 在 9 项 Recall@K 指标中的 8 项取得
最佳结果，并在多跳推理需求最高的 MuSiQue 上达到 80.0% 的 Recall@5。SAG 也已经以数亿条数据项的生产
规模部署，在线检索延迟保持在秒级以内。项目站点和代码见
https://github.com/Zleap-AI/SAG-Benchmark。

## 1 引言

随着大语言模型能力持续进步，智能体系统的瓶颈正从模型能力转向数据基础设施。面对不断增长的语料、
跨系统关联和持续演化的状态，智能体需要的不是一次性的静态检索，而是一种能够持续摄取增量数据并支持
多步关联查询的检索基础设施。当今检索增强生成（RAG）的主流做法是把文档切分成块，将其映射到向量空间，
并在查询时检索最相似的块（Lewis et al., 2020; Karpukhin et al., 2020）。这种方法在开放域问答等任务上
表现稳健。然而，智能体往往需要多步顺序检索，每一步的错误都会沿推理链累积并放大。因此，检索基础设施
需要的不仅是更高的单次召回率，还需要在多跳查询中可靠组织证据的能力。

现有方法主要沿两个方向应对这一挑战，而二者各有局限。第一类是稠密检索，其核心是语义相似度匹配；
它擅长检索语义接近的段落，但难以恢复实体之间显式的关联链，更难把散落在多篇文档中的证据组织成结构化
证据链（Yang et al., 2018; Trivedi et al., 2022; Mavi et al., 2024）。当查询涉及时间约束、实体角色或多步
依赖时，这一限制尤其明显。第二类是结构增强方法，它们从文档中离线构建知识图谱或层级摘要，以显式表示
实体关系（Edge et al., 2024; Gutiérrez et al., 2025）。

但显式结构也有代价。三元组抽取、实体合并和关系规范化会在连续阶段分别引入错误，构建成本也很高。
随着数据演化，维护全局图的成本甚至可能超过构建它的成本。更关键的是，这些精心构建的离线结构在查询时
常常退化为节点或摘要层面的扁平相似度匹配，形成离线结构与在线召回之间的系统性脱耦（详见第 2 节）。

**图 1：三种 RAG 范式的流程与性能比较。** NaiveRAG 通过稠密向量相似度检索 top-$k$ 个块，具有良好的
可扩展性、较高速度和低成本，但精度受浅层语义匹配限制。GraphRAG 使用 LLM 离线抽取三元组并构建知识图谱；
它增强了证据组织能力，但构建成本高，并且难以进行增量更新。SAG 使用 LLM 抽取事件和实体，并在查询时通过
SQL 激活动态超边结构；它在检索质量、结构能力和系统开销之间取得平衡，同时天然支持仅追加式增量更新。

我们的核心主张是：对于涉及结构化约束和多跳关联的查询，检索既不应完全由稠密相似度支配，也不应由离线
预构建的静态图支配。SAG 将文档转换为事件—实体索引，其中每个块对应一个保留完整语义的事件，以及一组
发挥索引作用的实体，二者共同定义一条潜在超边（图 1 对比了三种范式）。在查询时，SQL 驱动确定性的
事件—实体关联和局部超边激活；这一路结构化路径与向量检索结合为统一管线，LLM 只在压缩后的候选集上执行
最终选择。由于超边不是预先构建的，而是围绕当前查询动态实例化，系统不依赖静态图结构，也不需要全局重计算。

总之，本文的主要贡献如下：（1）我们提出 SAG，一种结构化检索架构，用条目—实体索引取代离线静态图，并以
SQL 驱动检索为核心。它在单一管线中统一了结构化过滤、语义扩展和 LLM 精排三种能力；（2）我们设计了一种
查询时动态超边组织机制，使高阶关系能够在当前查询周围的局部候选空间内动态激活，无需事先枚举，并能通过
SQL 连接确定性地跨多跳扩展；（3）我们在三个多跳基准上系统评估 SAG，并通过消融研究分离事件级语义保留、
动态扩展、LLM 使用模式和候选预算各自的贡献；（4）我们已在数亿条记录规模的生产环境中部署 SAG，证明了
该框架在持续增量写入和在线成本约束下的工程可行性。

## 2 相关工作

### 2.1 检索增强生成

检索增强生成（RAG）通过把知识访问重构为检索问题，将 LLM 与外部知识连接起来（Lewis et al., 2020;
Karpukhin et al., 2020）。但默认的“块—向量—top-$k$”管线引入了经典的分块困境，并将检索固定为推理前的
一个步骤：候选项在模型理解查询之前就已经确定。后续工作从两个维度引入自适应检索。触发策略允许模型自主
决定何时检索（Asai et al., 2024; Jiang et al., 2023b; Jeong et al., 2024），而迭代推理则把多步检索与推理
交错进行（Trivedi et al., 2023; Zhuang et al., 2024）。这些进展关注的是何时检索以及用什么查询去检索；
但一旦候选项被识别，组织阶段仍然只依赖语义相似度，无法在候选项之间执行结构化过滤或显式实体关联。

### 2.2 结构增强检索与基于图的 RAG

为弥补向量检索在多跳证据组织上的不足，结构增强 RAG 主要沿三条路线发展。第一条路线从文档中离线构建
知识图谱，然后通过图遍历或排序激活多跳子图；代表性工作包括 GraphRAG（Edge et al., 2024）、HippoRAG
及其后续工作（Gutiérrez et al., 2024; 2025），以及探索提示融合、图神经网络集成和在现有知识图谱上进行
交互式遍历的方法（Wang et al., 2024; He et al., 2024; Sun et al., 2024; Liu et al., 2025; Mavromatis et al.,
2025）。第二条路线以层级方式组织文档（Sarthi et al., 2024; Huang et al., 2025; Guo et al., 2025; Zhang et
al., 2025），其中 SiReRAG 在本文使用的同三个基准上验证了结构化索引对召回的价值。第三条路线执行运行时
动态结构化，例如 StructRAG（Li et al., 2025）在推理时把文档转换为混合知识表示。三条路线都面临相同的
权衡：离线或推理时构建的结构成本高昂，而查询时检索却常常退化为对节点或摘要的扁平相似度匹配，形成精心
设计的离线结构与在线召回之间的系统性脱耦。SAG 通过 SQL 在查询时激活局部超边，从机制层面消除这种脱耦，
将结构化组织嵌入检索执行本身。

### 2.3 SQL、表格扎根问答与结构化检索接口

关于结构化数据问答的研究表明，结构化接口能够提升检索精度，但这些方法都预设底层结构已经存在。表格问答
验证了语言模型在表格上的推理能力，并提出跨表格—文本的多跳任务（Herzig et al., 2020; Liu et al., 2022;
Chen et al., 2020）；text-to-SQL 将这一方向扩展到查询侧，构建了大规模跨领域基准，并持续提升复杂 SQL
生成能力（Yu et al., 2018; Li et al., 2023; Pourreza & Rafiei, 2023）；StructGPT（Jiang et al., 2023a）和
ChatDB（Hu et al., 2023）分别将结构化接口纳入 LLM 系统，用于迭代式接口推理和关系数据库式长期记忆。
这些方法要么把 SQL 当作生成目标，要么直接预设结构化数据已经就绪；它们回答的是如何查询已有结构的问题。
SAG 处理的是更早一步的问题：它先通过离线事件抽取，从非结构化文档构建可查询的事件—实体表，然后以 SQL
的精确过滤驱动开放域 RAG 的主检索路径，并将其与向量检索集成为单一管线。

### 2.4 超边与高阶关系表示

传统知识图谱将二元关系表示为三元组 $(h, r, t)$，但现实世界事件往往涉及多个参与方或维度；强行将其分解为
二元关系会破坏整体语义。超图学习和 $n$ 元关系建模方面的工作表明，高阶表示比三元组分解更能保留原始语义
结构（Zhou et al., 2006; Feng et al., 2019; Fatemi et al., 2020; Galkin et al., 2020）。然而，这一思想在
RAG 中仍未得到充分利用；主流图增强方法仍然把实体建模为节点、把二元边建模为关系（Edge et al., 2024;
Gutiérrez et al., 2025）。近期，HGRAG（Wang et al., 2026）和 Graph-R1（Luo et al., 2026）引入超图，
但二者都离线预构建超图结构，并在查询时依赖基于嵌入的近似匹配进行激活；因此，它们仍然承担静态结构的
维护成本，并遭受匹配误差累积。SAG 将每个事件及其关联实体视为一条潜在超边，在查询时使用 SQL 识别共享
实体的一组事件，从而动态实例化局部超边，既避免了离线预构建超图的成本，又在 $n$ 元事件场景中保留了
高阶表示的表达能力。

## 3 SAG

### 3.1 框架概览

图 2 展示了 SAG 的完整架构，分为离线阶段和在线阶段。在离线阶段，每个文档块被转换为一个事件和一组实体，
并同步写入 SQL 数据库、向量索引和全文索引。在线阶段随后按顺序运行三个步骤，即种子检索、查询时扩展和
最终选择；每一步的详细设计在后续小节中给出。

这三个阶段建立在各模块清晰分工之上。SQL 负责确定性过滤和连接；向量检索负责别名、近义表达和改写的语义
扩展；LLM 则保留给少量高价值语义决策点。从流程角度看，SAG 检索包含三个顺序步骤。首先，通过实体定位
相关事件的入口点（种子检索）；然后，沿共享实体在不同事件之间跳转以扩展候选池（查询时扩展）；最后，在
压缩后的候选空间上执行细粒度选择（LLM 重排序）。

### 3.2 事件—实体索引

SAG 构建的是事件—实体索引，而不是全局知识图谱。给定一个块，索引阶段会生成一个事件 $e$ 和一组实体
$U(e)$，二者共同定义一条潜在超边。

**事件** 是对块核心内容的简洁表述，采用“一个块对应一个事件”的映射。事件保留完整语义，不再进一步分解为
多个独立三元组，从而避免三元组抽取固有的语义碎片化问题，同时为查询时动态超边激活提供必要的数据基础。

**实体** 覆盖时间、地点、人物、组织、群体、主题、作品、产品、动作、指标和标签，共 11 种类型。实体不承载
完整语义；它们仅作为连接不同事件的索引点和扩展点。

事件和实体不是由一串级联的顺序抽取步骤产生的；它们是同一块的两个并行结构化输出。抽取结果写入 SQL，
建立事件与实体之间的多对多连接；事件文本和实体文本同时写入向量索引和全文索引。一个事件连接多个实体，
即定义一条潜在超边。

该索引有意避免引入完整的实体消歧系统。真正承载语义的是事件；实体仅作为索引点和扩展点。SAG 对实体处理
采用务实策略，依赖简单的字符串规范化和 SQL 去重；稳定运行不需要完整的实体合并机制。按设计，SAG 的索引层
不是重量级知识图谱，而是一个面向非结构化文档的轻量、可追加语义索引。

**图 2：SAG 架构概览。** 在离线阶段，每个块被转换为一个事件和一组实体，并写入 SQL、向量和全文索引。
在线阶段，系统先执行初始召回，再执行查询时扩展，最终在压缩后的候选集上完成选择。

### 3.3 种子检索

给定查询 $q$，SAG 通过两条并行路径构建初始候选事件集 $E_R$。

**路径 A：实体引导的结构化召回。** LLM 从查询文本中识别关键实体，生成种子实体集 $U_q=\{u_i\}$。系统使用
每个种子实体作为查询，在实体向量索引上执行相似度检索（默认阈值为 0.9），召回语义相近的扩展实体，得到
增强实体集 $\hat{U}_q \supseteq U_q$。随后，SQL 连接查询检索与这些实体关联的所有事件：

$$
E_R^{entity}=\{e \mid \exists u\in \hat{U}_q:\mathrm{SQL\text{-}Join}(e,u)\}.
$$

**路径 B：通过查询向量直接召回事件。** 系统同时使用查询向量在事件索引上执行相似度检索，保留相似度超过
阈值 $\tau$（默认 0.4）的事件，得到直接候选 $E_R^{direct}$。两条路径合并形成初始候选事件集：

$$
E_R=E_R^{entity}\cup E_R^{direct}.
$$

路径 A 通过实体关联覆盖结构化多跳线索；路径 B 通过语义匹配覆盖直接相关事件。二者共同构成初始候选召回阶段。

### 3.4 查询时扩展与选择

查询时扩展将种子检索结果拓宽为更全面的候选池。超边不会在离线阶段显式枚举，而是围绕当前查询的候选项动态
实例化。

**扩展。** 从 $E_R$ 出发，系统执行反向 SQL 连接查询，抽取关联实体，形成实体前沿（即新近连接到种子事件但
尚未探索的实体集合），然后使用这些前沿实体作为桥接点发现新事件，逐跳扩展候选集。该过程完全依赖 SQL 连接；
多跳扩展等价于数据库中的关系连接，而不是 PageRank 或图推理。扩展最多运行 $H$ 跳（默认 $H=1$），每一轮
只引入此前未见过的实体和事件。将扩展过程中新增的事件集合记为 $E_E$，完整候选池为：

$$
E_{cand}=E_R\cup E_E.
$$

**粗排序。** SAG 按候选事件与查询向量的相似度过滤 $E_{cand}$，保留 top-$K_{cand}$（默认 100），记为
$\hat{E}$。

**双路径输出。** 系统并行执行两条输出路径。路径 A（结构路径）：LLM 在 $\hat{E}$ 上执行最终重排序，选择
top-$K_{event}$ 个事件，得到 $E^\*=\mathrm{Rerank}(\hat{E}, q, f_{LLM})$，再将它们映射回原始块，得到
$C_{event}=\mathrm{Map}(E^\*)$。路径 B（语义路径）：使用查询向量在块索引上直接检索 top-$K_{direct}$ 个
块，得到 $C_{direct}=\mathrm{Embed}_{top\text{-}K}(q)$。两个集合合并、去重，并返回 top-$K_{out}$ 个块作为
最终证据：

$$
C_{out}=\mathrm{TopK}_{K_{out}}\left(C_{event}\cup C_{direct}\right).
$$

### 3.5 可解释性

这一管线天然产生完全可审计的轨迹：

$$
q\rightarrow U_q\rightarrow \hat{U}_q\rightarrow E_R\rightarrow E_{cand}\rightarrow \hat{E}\rightarrow C_{out}.
$$

链条中的每一步都可检查。对于查询“收购 B 公司的那家公司 CTO 后来加入了哪个项目？”，链条会逐步展开：
从查询中识别出实体 $\{\text{Company B}, \text{CTO}\}$，通过实体向量搜索进行别名扩展，SQL 连接查询关联到
“Company A 收购 Company B”和“某人加入 Project C”等事件，然后沿共享实体进行基于跳数的聚合，最终把对应
原始块排序到输出中。任一环节为空结果都会直接指出失败位置：是实体未被识别、扩展没有产生新候选，还是
SQL 连接没有返回结果。这种逐步可检查性使检索失败能够定位到具体阶段，而不是只表现为一个不可分解的端到端
分数。

## 4 实验

### 4.1 数据集

我们选择 HotpotQA、2WikiMultiHopQA 和 MuSiQue 三个多跳基准，它们对推理链深度和不可跳过性的要求逐步
提高，适合评估不同难度下的跨文档实体链接能力。

**HotpotQA**（Yang et al., 2018）主要包含桥接类和比较类问题，需要跨两篇维基百科文档进行推理。我们采用
fullwiki 设置，要求系统从完整维基百科语料中检索支持段落，而不是从预过滤候选集中选择。该数据集为每个问题
提供段落级支持事实标注，我们直接以黄金段落评估召回。该数据集主要评估二跳推理场景中的跨文档实体链接能力。

**2WikiMultiHopQA**（Ho et al., 2020）包含跨多篇维基百科文档的多跳问题，覆盖桥接、比较和推断类型，并提供
显式推理路径标注，覆盖每一步推理所依赖的中间段落。该数据集要求系统同时定位桥接段落和最终答案段落，对
跨多条路径推理提出更高要求。

**MuSiQue**（Trivedi et al., 2022）通过逐步组合单跳问题来构造多跳问题，包含最多四步推理的样本。通过严格
的反事实过滤，该数据集确保每个推理步骤都不可跳过，防止系统通过单步语义匹配绕开完整推理链。这一特性使
MuSiQue 成为检验 SAG 超边扩展机制的核心测试平台；任何仅依赖向量相似度的方法在这里都会面临最大挑战。
我们使用其可回答子集进行评估。

三个数据集都提供段落级支持事实标注，因此 Recall 直接衡量系统是否把支持证据排在靠前位置，与后续生成模型
选择什么无关。所有实验均在各数据集的开发集上进行。表 1 总结了三个数据集的基本统计信息。

**表 1：本文使用的评估子集统计。** 对每个数据集，我们从官方开发划分中随机采样 1,000 个问题。“Passages”
指由采样问题的支持文档构建的局部语料中的段落数量。

| 数据集 | 查询数 | 段落数 |
| --- | ---: | ---: |
| MuSiQue | 1,000 | 11,656 |
| 2Wiki | 1,000 | 6,119 |
| HotpotQA | 1,000 | 9,811 |

### 4.2 对比方法

我们将 SAG 与 HippoRAG 2（Gutiérrez et al., 2025）进行比较。HippoRAG 2 首先使用 LLM 从文档中离线抽取
实体和关系并构建知识图谱，然后在查询时通过 Personalized PageRank 执行多跳图检索，代表了“离线图构建 +
全局图排序”范式的领先实现，并与 SAG 的“查询时动态超边”方法形成直接机制对比。我们在与 SAG 相同的嵌入
模型和 LLM 配置下重新运行 HippoRAG 2，以隔离底层模型选择的影响。此外，为了将 SAG 放置在检索质量谱系中，
表 2 还纳入简单检索器和 70 亿参数大型嵌入检索器作为背景参考。

### 4.3 指标

我们采用 Recall@K 作为主要检索指标，衡量系统返回的 top-$K$ 段落中至少包含一条支持证据的问题比例
（any-hit 标准）。该指标衡量的是单步证据可访问性，而非完整多跳证据链覆盖，因此在严格多跳评估下往往会
高估性能；我们将其视为与实际检索效用一致的近似度量，而不是对推理链完整性的严格保证。

尽管如此，在智能体顺序检索场景中，靠前排序的 Recall@K 具有直接工程价值。当返回候选更少且排序更精确时，
下游 LLM 处理所需的上下文窗口更小，调用成本更低，并为后续步骤保留更多检索预算。我们主要报告 Recall@2
和 Recall@5，并在消融中纳入 Recall@1，以刻画靠前命中的稳定性。

对于 HotpotQA 和 2WikiMultiHopQA，支持段落为数据集提供的黄金段落；对于 MuSiQue，支持段落为标注推理路径
覆盖的一组段落。如果至少一条支持证据出现在返回的 top-$K$ 段落中，则认为检索成功。

### 4.4 实现细节

**LLM 配置。** 我们使用 Qwen3.6-Flash（Qwen Team, 2025）进行离线阶段的事件摘要和实体抽取，以及在线阶段
的候选重排序。离线抽取以批处理模式运行；在线重排序只作用于压缩后的候选集，因此 LLM 调用次数保持较少。
查询实体抽取模型 $f_{entity}$ 同样由 Qwen3.6-Flash 通过 few-shot 提示驱动，负责从查询中识别实体线索；
这是在线阶段唯一一次轻量 LLM 调用。事件摘要模型 $f_{event}$ 以批处理方式执行块到事件的转换和实体抽取。

**嵌入配置。** 默认嵌入模型为 BGE-Large-EN-v1.5（Xiao et al., 2023）。我们还在 MuSiQue 上使用 NV-Embed-v2
（Lee et al., 2024）进行对比实验。SAG 使用 MySQL 作为结构化存储后端，使用 Elasticsearch 作为全文搜索
引擎；事件和实体的向量相似度检索也通过 Elasticsearch 的稠密向量能力实现，而没有引入单独的专用向量数据库。

**超参数设置。** 除非另有说明，SAG 采用以下默认配置。扩展跳数 $H=1$，初始种子召回预算 $K_{seed}=50$，
实体前沿剪枝预算为 50，传给 LLM 的候选事件数 $K_{cand}=100$。系统最终返回 $K_{out}=10$ 个块，其中
$K_{event}=5$ 个来自 LLM 对候选事件的重排序，$K_{direct}=5$ 个来自直接查询向量检索；两个流在输出前合并
并去重。实体类型覆盖时间、地点、人物、组织、群体、主题、作品、产品、动作、指标和标签，共 11 种，并可
按领域扩展。

## 5 结果与分析

### 5.1 主要结果

表 2 报告了三个多跳基准上的 Recall@2/5。在统一底层配置下，SAG 达到 79.3%/88.2% 的平均 Recall@2/5，
比 HippoRAG 2 的 68.2%/83.3% 高出 11.1/4.9 个百分点。除 2WikiMultiHop 的 Recall@5 外，SAG 在其余所有
指标上都取得最佳结果。

SAG 的优势在 MuSiQue 上最为明显，该数据集具有最长的推理链。MuSiQue 包含最多 4 跳推理，并且每个推理步骤
都不可跳过，防止系统通过单步语义匹配绕开完整推理链。SAG 的 Recall@5 达到 80.0%，而 HippoRAG 2 为 65.1%；
在 Recall@2 上差距更大（64.1% vs. 49.5%）。这一模式与两种方法的机制差异一致。SAG 使用 SQL 连接沿共享
实体确定性扩展，每一跳的扩展路径都显式且可追踪。HippoRAG 2 通过 Personalized PageRank 在全局图上传播
分数；远距离节点的分数会在阻尼因子下衰减，噪声节点的影响也会随着跳数增加通过间接路径叠加，这一效应在
更长推理链场景中更加重要。

**表 2：统一底层配置（BGE-Large-EN-v1.5 + Qwen3.6-Flash）下 SAG 与各类基线在三个数据集上的 Recall@2/5。**
标注 * 的数值来自文献报告，仅作为背景参考。

| 方法 | MuSiQue | 2Wiki | HotpotQA | 平均 |
| --- | ---: | ---: | ---: | ---: |
| **简单基线** |  |  |  |  |
| Contriever* | 34.8 / 46.6 | 46.6 / 57.5 | 58.4 / 75.3 | 46.6 / 59.8 |
| BM25* | 32.4 / 43.5 | 55.3 / 65.3 | 57.3 / 74.8 | 48.3 / 61.2 |
| GTR (T5-based)* | 37.4 / 49.1 | 60.2 / 67.9 | 59.3 / 73.9 | 52.3 / 63.6 |
| **大型嵌入模型** |  |  |  |  |
| BGE-Large-EN-v1.5 | 41.6 / 56.2 | 61.6 / 69.0 | 76.0 / 88.8 | 59.7 / 71.3 |
| GTE-Qwen2-7B-Instruct* | 48.1 / 63.6 | 66.7 / 74.8 | 75.8 / 89.1 | 63.5 / 75.8 |
| GritLM-7B* | 49.7 / 65.9 | 67.3 / 76.0 | 79.2 / 92.4 | 65.4 / 78.1 |
| NV-Embed-v2 (7B)* | 52.7 / 69.7 | 67.1 / 76.5 | 84.1 / 94.5 | 68.0 / 80.2 |
| **基于图的方法** |  |  |  |  |
| HippoRAG 2 (BGE-Large-EN-v1.5) | 49.5 / 65.1 | 76.6 / 90.4 | 78.4 / 94.4 | 68.2 / 83.3 |
| SAG（本文） | **64.1 / 80.0** | **82.3** / 88.0 | **91.6 / 96.5** | **79.3 / 88.2** |

在 HotpotQA 上，推理链更短，主要是 2 跳推理，两种方法的 Recall@5 差距收窄（96.5% vs. 94.4%），但
Recall@2 差距仍达到 13.2 个百分点（91.6% vs. 78.4%）。Recall@5 差距较小与上述机制一致：当推理链只有
2 跳时，全局图传播的衰减效应尚未充分显现，两种方法都能在更深排名处实现覆盖。SAG 在靠前排名上仍保持优势，
说明超边结构为排序质量带来了稳定改进。在多步智能体检索中，单步错误会沿推理链累积并放大，因此靠前命中的
质量直接决定后续步骤可用的检索空间和误差余量。

在 2WikiMultiHop 上，SAG 的 Recall@5 为 88.0%，略低于 HippoRAG 2 的 90.4%，但 Recall@2 仍以 82.3%
领先（HippoRAG 2 为 76.6%），这一结果需要直接分析。2WikiMultiHop 中一些问题涉及极长的实体链。当某个
桥接实体在语料中出现频率很低时，SAG 固定的剪枝预算（实体前沿剪枝预算为 50）可能会在扩展早期截断它，
导致依赖该实体的远端事件无法进入候选池。相比之下，HippoRAG 2 的全局 PageRank 可以通过全图传播到达这些
低频节点。这表明 SAG 当前的剪枝策略对低频桥接实体存在系统性盲点，这是一个需要未来改进的具体弱点。
在这一设置下 SAG 的 Recall@2 仍然领先，说明该弱点主要影响尾部召回，而非头部命中。

### 5.2 消融研究

我们在 MuSiQue 上对 SAG 的关键设计选择进行消融，因为该数据集由于强制中间步骤和反事实过滤，对多跳推理
提供了最严格的测试，最能区分设计差异。所有消融均保留 SAG 默认设置，只改变被考察的单一变量。

**超边 vs. 三元组。** 在 SAG 的三元组变体中，每个事件被分解为一组 $(subject, predicate, object)$ 三元组，
每个三元组独立建索引并参与 SQL 连接；管线其余部分保持不变。如表 3 所示，三元组版 SAG 的 Recall@5 为
77.1%，超边版为 80.0%，二者都高于 HippoRAG 2 的 65.1%。HippoRAG 2 的核心关系单元是“实体—关系—实体”
三元组，因此在本比较中与三元组版 SAG 属于同一表示类别。在表示条件相同的情况下，SAG 管线仍领先 12 个
百分点，说明差距来自架构而非表示。SAG 使用 SQL 连接沿实体路径逐跳确定性扩展，每一步覆盖清晰且可控；
HippoRAG 2 则通过 Personalized PageRank 在全图上叠加传播，因此远距离节点的分数衰减和噪声节点干扰都会
随跳数增加而增强。在此基础上，超边结构又贡献了 2.9 个百分点。超边将单条记录内的多个实体压缩在一起，
使一次 SQL 连接即可同时激活所有关联路径；三元组表示中每条记录只覆盖两个端点，达到等价覆盖需要更多跳，
而每一跳都会引入额外剪枝损失。合起来看，SAG 的收益有两个组成部分：管线架构相对于可比方法打开了系统性
差距，而超边表示进一步收紧了召回边界。

**表 3：事件级索引与三元组分解索引的消融比较（MuSiQue）。**

| 配置 | R@1 | R@2 | R@5 | R@10 |
| --- | ---: | ---: | ---: | ---: |
| Triple | 35.6 | 61.5 | 77.1 | 81.2 |
| Hyperedge | 36.2 | 64.1 | 80.0 | 83.4 |

**查询时扩展的贡献。** 将扩展跳数 $H$ 设为 0 会禁用扩展，同时保持其他设置不变。如表 4 所示，禁用扩展会使
Recall@5 从 80.0% 降至 69.4%，而 Recall@1 几乎不变（36.2% vs. 35.7%）。这一模式揭示了扩展真正贡献的
内容：它加入了向量召回本身无法浮现的证据，而不是改善候选池中已有候选项的排序。扩展引入的事件通过共享
实体路径访问，和查询缺乏直接语义重叠，因此向量相似度无法可靠地把它们排到靠前位置；所以 Recall@1 基本
不受影响。但这些候选恰恰是多跳推理链中的关键中间证据，缺少它们会导致 Recall@5 显著下降。

**表 4：动态超边构建中扩展跳数的消融研究（MuSiQue）。**

| 跳数 | R@1 | R@2 | R@5 | R@10 |
| --- | ---: | ---: | ---: | ---: |
| 无扩展 | 35.7 | 57.3 | 69.4 | 74.3 |
| 有扩展（基线） | 36.2 | 64.1 | 80.0 | 83.4 |

**候选事件数量权衡。** 为检验 SAG 的收益是否仅仅来自扩大候选池，我们考察了改变 $K_{cand}$ 的影响（表 5）。
当候选事件数从 50 增至 100 时，Recall@5 从 76.1% 提升到 80.0%，收益显著。继续扩展到 200 或 500 时，边际
收益急剧递减：每百万额外输入 token 带来的 Recall@5 边际收益低于 0.089 个百分点。因此，我们将
$K_{cand}=100$ 设为默认运行点，在召回质量和 LLM 调用成本之间取得平衡。

**表 5：候选事件数量与 token 成本之间的权衡。**

| 事件数量 | R@1 | R@2 | R@5 | R@10 | Tokens (M) |
| --- | ---: | ---: | ---: | ---: | ---: |
| 50 | 36.1 | 61.6 | 76.1 | 79.7 | 12.0 |
| 100（基线） | 36.2 | 64.1 | 80.0 | 83.4 | 20.0 |
| 200 | 36.5 | 65.0 | 80.9 | 84.4 | 35.5 |
| 500 | 36.3 | 64.3 | 81.8 | 86.1 | 76.4 |

**图 3：Token 边际收益分析。** 候选事件数从 50 增至 100 时，Recall@5 从 76.12% 提升到 80.04%；继续增至
200 和 500 时，Recall@5 分别为 80.92% 和 81.82%，但 token 成本明显上升。

**LLM 用于重排序的必要性。** 将 LLM 替换为轻量级 Qwen3-Reranker-0.6B，会使 MuSiQue 上的 Recall@5 从
80.0% 降至 62.2%，下降 17.8 个百分点（表 6）。轻量重排序器会独立给每个候选打分，无法联合评估哪些事件
子集共同构成完整推理链。在压缩后的候选集上，LLM 能够发现哪些事件共享实体，以及它们在逻辑上如何相互依赖，
因而能更准确地识别对当前查询至关重要的事件子集。

**表 6：最终选择阶段中轻量重排序器与 LLM 过滤的比较（MuSiQue）。**

| 方法 | R@1 | R@2 | R@5 | R@10 |
| --- | ---: | ---: | ---: | ---: |
| Qwen3-Reranker-0.6B | 32.5 | 46.7 | 62.2 | 70.8 |
| Qwen3.6-Flash | 36.2 | 64.1 | 80.0 | 83.4 |

**双路径候选池的互补性。** 候选池结合了结构路径（由 LLM 重排序并映射到块的事件）和语义路径（由查询向量
直接检索的块）。我们在总候选数固定为 $K_{out}=10$ 的情况下改变两条路径之间的分配，如表 7 所示。当
$K_{event}=0$ 时，系统退化为纯语义检索，Recall@5 只有 56.2%；当 $K_{event}=2$ 时，Recall@5 升至 73.3%；
在 $K_{event}=5$（基线）时达到最优的 80.0%。即使只有少量事件派生候选也能带来显著增益，说明结构路径提供
了语义路径遗漏的跨实体推理证据，并且两条路径在证据覆盖上高度互补，而非冗余。

**表 7：基于 LLM 的事件选择与基于直接块的选择对比（MuSiQue）。**

| LLM Event | Query chunk | R@1 | R@2 | R@5 | R@10 |
| ---: | ---: | ---: | ---: | ---: | ---: |
| 0 | 10 | 29.4 | 41.6 | 56.2 | 63.8 |
| 2 | 8 | 36.6 | 64.5 | 73.3 | 77.6 |
| 4 | 6 | 36.6 | 63.6 | 79.6 | 82.6 |
| 5（基线） | 5 | 36.2 | 64.1 | 80.0 | 83.4 |

### 5.3 对嵌入模型的鲁棒性

为验证 SAG 的检索收益来自结构设计而非特定嵌入模型，我们将底层嵌入模型从 BGE-Large-EN-v1.5 替换为更强的
NV-Embed-v2，并在 MuSiQue 上重新运行 SAG 和 HippoRAG 2。如表 8 所示，在 NV-Embed-v2 下，SAG 的 Recall@5
为 81.7%，HippoRAG 2 为 74.6%；当切回 BGE-Large-EN-v1.5 时，SAG 仍稳定在 80.0%，而 HippoRAG 2 降至
65.1%。HippoRAG 2 损失近 10 个百分点；SAG 几乎不受影响。这种非对称影响直接反映了两种方法的机制差异。
HippoRAG 2 的多跳检索以 Personalized PageRank 传播为中心，初始分数由嵌入相似度给定；嵌入质量的变化会
沿传播路径逐跳放大，最终对检索结果产生较大影响。SAG 的结构性收益主要来自 SQL 连接，而 SQL 连接基于精确
字符串匹配，与嵌入质量无关。即使使用较弱的 BGE-Large-EN-v1.5，SAG 仍以 80.0% 对 65.1% 领先 HippoRAG 2，
证明 SAG 的核心优势并不依赖更强嵌入模型，而是来自架构设计本身。

**表 8：NV-Embed-v2 配置下 SAG 与 HippoRAG 2 在 MuSiQue 上的检索结果。**

| 方法 | R@1 | R@2 | R@5 | R@10 |
| --- | ---: | ---: | ---: | ---: |
| SAG (NV-Embed-v2) | 36.4 | 64.6 | 81.7 | 86.6 |
| HippoRAG 2 (NV-Embed-v2) | 33.7 | 56.0 | 74.6 | 83.2 |

## 6 讨论

SAG 的检索收益主要来自事件级语义保留以及共享实体提供的结构化连通性，而不是把某个单一组件做得更强。
当前方法仍有若干局限，讨论如下。

**剪枝机制。** 当前的实体前沿剪枝预算和相似度阈值是在开发集上经验设定的。低频但关键的桥接实体可能在扩展
早期被截断；这一风险在长实体链场景中尤其明显，也与 SAG 在 2WikiMultiHop 上的 Recall@5 略低于 HippoRAG 2
这一现象一致。未来工作可以按语料频率对实体分层，为低频实体分配更大的扩展预算，或引入失败案例分析来自动
校准阈值。

**实体合并。** SAG 采用轻量实体策略，在摄取前只执行字符串规范化和 SQL 去重，使块处理能够独立并发进行，
从而支持增量写入。其代价是，同一实体的不同字符串形式可能被视为不同索引点，削弱跨文档连接密度。引入轻量
实体别名表是一个可行方向；它可以在不牺牲写入独立性的前提下合并同义形式。

**面向智能体记忆的事件级更新。** 目前，SAG 可以将记忆组织为可追加、可检索的事件索引，支持持续摄取新事件。
在智能体记忆场景中，偏好覆盖、任务状态变更等情况还要求系统更新、失效或保留现有事件的历史版本，而不仅仅
是追加。将 SAG 索引扩展为带版本的智能体记忆基底，是自然的下一步，也是我们留给未来工作的主要方向。

## 7 结论

本文提出 SAG，一种面向检索与智能体系统的结构化检索框架。SAG 以 SQL 作为结构引擎，以向量数据库作为语义
引擎，并以事件—实体索引和查询时动态超边作为核心机制。其设计原则不是用更强模块替换标准 RAG，而是在检索
管线中重新分配职责。SQL 划定候选边界并强制执行结构化组织；事件—实体索引承载完整语义单元；动态超边在查询时
恢复局部高阶关系。向量模型和 LLM 只在最高价值的位置介入，而不是支配整个管线。在统一底层配置下，SAG 在
三个多跳基准的 9 项 Recall@K 指标中的 8 项取得最佳结果。消融分析进一步表明，这一优势并非来自更强组件，
而是根植于结构化组织本身。因此，SAG 改变的不是标准 RAG 的某个孤立组件，而是数据进入检索与智能体系统时
的组织范式。在此基础上，将事件—实体索引扩展为带版本、具备时间感知能力的智能体记忆基底，是未来工作的
自然方向。

## 参考文献

Akari Asai, Zeqiu Wu, Yizhong Wang, Avirup Sil, and Hannaneh Hajishirzi. Self-RAG: Learning to
retrieve, generate, and critique through self-reflection. In International Conference on Learning
Representations (ICLR), 2024. URL https://openreview.net/forum?id=hSyW5go0v8.

Wenhu Chen, Hanwen Zha, Zhiyu Chen, Wenhan Xiong, Hong Wang, and William Yang Wang.
HybridQA: A dataset of multi-hop question answering over tabular and textual data. In Findings
of the Association for Computational Linguistics: EMNLP 2020, 2020. URL https://arxiv.org/abs/2004.07347.

Darren Edge, Ha Trinh, Newman Cheng, Joshua Bradley, Alex Chao, Apurva Mody, Steven Truitt,
and Jonathan Larson. From local to global: A graph RAG approach to query-focused summarization.
arXiv preprint arXiv:2404.16130, 2024. URL https://arxiv.org/abs/2404.16130.

Bahare Fatemi, Perouz Taslakian, David Vazquez, and David Poole. Knowledge hypergraphs:
Prediction beyond binary relations. In Proceedings of the Twenty-Ninth International Joint
Conference on Artificial Intelligence (IJCAI), 2020. URL https://arxiv.org/abs/1906.00137.

Yifan Feng, Haoxuan You, Zizhao Zhang, Rongrong Ji, and Yue Gao. Hypergraph neural networks.
In Proceedings of the AAAI Conference on Artificial Intelligence, 2019. URL https://arxiv.org/abs/1809.09401.

Mikhail Galkin, Priyansh Trivedi, Gaurav Maheshwari, Ricardo Usbeck, and Jens Lehmann. Message
passing for hyper-relational knowledge graphs. In Proceedings of the 2020 Conference on Empirical
Methods in Natural Language Processing (EMNLP), 2020. URL https://arxiv.org/abs/2009.10847.

Zirui Guo, Lianghao Xia, Yanhua Yu, Tu Ao, and Chao Huang. LightRAG: Simple and fast
retrieval-augmented generation. In Proceedings of the 2025 Conference on Empirical Methods in
Natural Language Processing (EMNLP), 2025. URL https://arxiv.org/abs/2410.05779.

Bernal Jiménez Gutiérrez, Yiheng Shu, Yu Gu, Michihiro Yasunaga, and Yu Su. HippoRAG:
Neurobiologically inspired long-term memory for large language models. In Advances in Neural
Information Processing Systems (NeurIPS), 2024. URL https://arxiv.org/abs/2405.14831.

Bernal Jiménez Gutiérrez, Yiheng Shu, Weijian Qi, Sizhe Zhou, and Yu Su. From RAG to memory:
Non-parametric continual learning for large language models. In International Conference on
Machine Learning (ICML), 2025. URL https://arxiv.org/abs/2502.14802.

Xiaoxin He, Yijun Tian, Yifei Sun, Nitesh V. Chawla, Thomas Laurent, Yann LeCun, Xavier Bresson,
and Bryan Hooi. G-retriever: Retrieval-augmented generation for textual graph understanding and
question answering. In Advances in Neural Information Processing Systems (NeurIPS), 2024.
URL https://arxiv.org/abs/2402.07630.

Jonathan Herzig, Pawel Krzysztof Nowak, Thomas Müller, Francesco Piccinno, and Julian
Eisenschlos. TaPas: Weakly supervised table parsing via pre-training. In Proceedings of the
58th Annual Meeting of the Association for Computational Linguistics (ACL), 2020.
URL https://aclanthology.org/2020.acl-main.398/.

Xanh Ho, Anh-Khoa Duong Nguyen, Saku Sugawara, and Akiko Aizawa. Constructing a multi-hop
QA dataset for comprehensive evaluation of reasoning steps. In Proceedings of the 28th International
Conference on Computational Linguistics (COLING), 2020. URL https://aclanthology.org/2020.coling-main.580/.

Chenxu Hu, Jie Fu, Chenzhuang Du, Simian Luo, Junbo Zhao, and Hang Zhao. ChatDB: Augmenting
LLMs with databases as their symbolic memory. arXiv preprint arXiv:2306.03901, 2023.
URL https://arxiv.org/abs/2306.03901.

Haoyu Huang, Yongfeng Huang, Junjie Yang, Zhenyu Pan, Yongqiang Chen, Kaili Ma, Hongzhi
Chen, and James Cheng. Retrieval-augmented generation with hierarchical knowledge. In Findings
of the Association for Computational Linguistics: EMNLP 2025, 2025. URL https://arxiv.org/abs/2503.10150.

Soyeong Jeong, Jinheon Baek, Sukmin Cho, Sung Ju Hwang, and Jong Park. Adaptive-RAG:
Learning to adapt retrieval-augmented large language models through question complexity.
In Proceedings of the 2024 Conference of the North American Chapter of the Association for
Computational Linguistics (NAACL), 2024. URL https://aclanthology.org/2024.naacl-long.389/.

Jinhao Jiang, Kun Zhou, Zican Dong, Keming Ye, Wayne Xin Zhao, and Ji-Rong Wen. StructGPT:
A general framework for large language model to reason on structured data. In Proceedings of the
2023 Conference on Empirical Methods in Natural Language Processing (EMNLP), 2023a.
URL https://arxiv.org/abs/2305.09645.

Zhengbao Jiang, Frank F. Xu, Luyu Gao, Zhiqing Sun, Qian Liu, Jane Dwivedi-Yu, Yiming Yang,
Jamie Callan, and Graham Neubig. Active retrieval augmented generation. In Proceedings of the
2023 Conference on Empirical Methods in Natural Language Processing (EMNLP), 2023b.
URL https://aclanthology.org/2023.emnlp-main.495/.

Vladimir Karpukhin, Barlas Oğuz, Sewon Min, Patrick Lewis, Ledell Wu, Sergey Edunov, Danqi
Chen, and Wen tau Yih. Dense passage retrieval for open-domain question answering. In Proceedings
of the 2020 Conference on Empirical Methods in Natural Language Processing (EMNLP), 2020.
URL https://aclanthology.org/2020.emnlp-main.550/.

Chankyu Lee, Rajarshi Roy, Mengyao Xu, Jonathan Raiman, Mohammad Shoeybi, Bryan Catanzaro,
and Wei Ping. NV-Embed: Improved techniques for training LLMs as generalist embedding models.
arXiv preprint arXiv:2405.17428, 2024. URL https://arxiv.org/abs/2405.17428.

Patrick Lewis, Ethan Perez, Aleksandra Piktus, Fabio Petroni, Vladimir Karpukhin, Naman Goyal,
Heinrich Küttler, Mike Lewis, Wen tau Yih, Tim Rocktäschel, Sebastian Riedel, and Douwe Kiela.
Retrieval-augmented generation for knowledge-intensive NLP tasks. In Advances in Neural
Information Processing Systems (NeurIPS), 2020. URL https://arxiv.org/abs/2005.11401.

Jinyang Li, Binyuan Hui, Ge Qu, Jiaxi Yang, Binhua Li, Bowen Li, Bailin Wang, Bowen Qin,
Ruiying Geng, Nan Huo, Xuanhe Zhou, Chenhao Ma, Guoliang Li, Kevin C.C. Chang, Fei Huang,
Reynold Cheng, and Yongbin Li. Can LLM already serve as a database interface? A BIg bench
for large-scale database grounded text-to-SQLs. In Advances in Neural Information Processing
Systems (NeurIPS), 2023. URL https://arxiv.org/abs/2305.03111.

Zhuoshi Li, Xin-Cheng Chen, Huiqian Yu, Haiming Lin, Jingbo Shang, Qiang Tang, Furu Wei,
Xuancheng Ren, Longtao Huang, and Chao Li. StructRAG: Boosting knowledge intensive reasoning
of LLMs via inference-time hybrid information structurization. In International Conference on
Learning Representations (ICLR), 2025. URL https://arxiv.org/abs/2410.08815.

Hao Liu, Zhengren Wang, Xi Chen, Zhiyu Li, Feiyu Xiong, Qinhan Yu, and Wentao Zhang. HopRAG:
Multi-hop reasoning for logic-aware retrieval-augmented generation. In Findings of the Association
for Computational Linguistics: ACL 2025, 2025. URL https://arxiv.org/abs/2502.12442.

Qian Liu, Bei Chen, Jiaqi Guo, Morteza Ziyadi, Zeqi Lin, Weizhu Chen, and Jian-Guang Lou.
TAPEX: Table pre-training via learning a neural SQL executor. In International Conference on
Learning Representations (ICLR), 2022. URL https://arxiv.org/abs/2107.07653.

Haoran Luo, Haihong E, Guanting Chen, Qika Lin, Yikai Guo, Fangzhi Xu, Zemin Kuang, et al.
Graph-R1: Towards agentic GraphRAG framework via end-to-end reinforcement learning. In
International Conference on Machine Learning (ICML), 2026. URL https://arxiv.org/abs/2507.21892.

Vaibhav Mavi, Anubhav Jangra, and Adam Jatowt. Multi-hop question answering. Foundations
and Trends in Information Retrieval, 17(5):457-586, 2024. URL https://arxiv.org/abs/2204.09140.

Costas Mavromatis, Soji Adeshina, Vassilis N. Ioannidis, Zhen Han, Qi Zhu, Ian Robinson, Bryan
Thompson, Huzefa Rangwala, and George Karypis. BYOKG-RAG: Multi-strategy graph retrieval
for knowledge graph question answering. In Proceedings of the 2025 Conference on Empirical
Methods in Natural Language Processing (EMNLP), 2025. URL https://arxiv.org/abs/2507.04127.

Mohammadreza Pourreza and Davood Rafiei. DIN-SQL: Decomposed in-context learning of
text-to-SQL with self-correction. In Advances in Neural Information Processing Systems (NeurIPS),
2023. URL https://proceedings.neurips.cc/paper_files/paper/2023/hash/72223cc66f63ca1aa59edaec1b3670e6-Abstract-Conference.html.

Qwen Team. Qwen3 technical report. Technical report, Alibaba Group, 2025.
URL https://arxiv.org/abs/2505.09388.

Parth Sarthi, Salman Abdullah, Aditi Tuli, Shubh Khanna, Anna Goldie, and Christopher D. Manning.
RAPTOR: Recursive abstractive processing for tree-organized retrieval. In International Conference
on Learning Representations (ICLR), 2024. URL https://arxiv.org/abs/2401.18059.

Jiashuo Sun, Chengjin Xu, Lumingyuan Tang, Saizhuo Wang, Chen Lin, Yeyun Gong, Lionel M. Ni,
Heung-Yeung Shum, and Jian Guo. Think-on-graph: Deep and responsible reasoning of large
language model on knowledge graph. In International Conference on Learning Representations
(ICLR), 2024. URL https://arxiv.org/abs/2307.07697.

Harsh Trivedi, Niranjan Balasubramanian, Tushar Khot, and Ashish Sabharwal. Musique: Multihop
questions via single-hop question composition. Transactions of the Association for Computational
Linguistics (TACL), 10, 2022. URL https://aclanthology.org/2022.tacl-1.31/.

Harsh Trivedi, Niranjan Balasubramanian, Tushar Khot, and Ashish Sabharwal. Interleaving retrieval
with chain-of-thought reasoning for knowledge-intensive multi-step questions. In Proceedings of the
61st Annual Meeting of the Association for Computational Linguistics (ACL), 2023.
URL https://aclanthology.org/2023.acl-long.557/.

Changjian Wang, Weihong Deng, Weili Guan, Quan Lu, and Ning Jiang. Cross-granularity hypergraph
retrieval-augmented generation for multi-hop question answering. In Proceedings of the AAAI
Conference on Artificial Intelligence, 2026. URL https://arxiv.org/abs/2508.11247.

Yu Wang, Nedim Lipka, Ryan A. Rossi, Alexa Siu, Ruiyi Zhang, and Tyler Derr. Knowledge graph
prompting for multi-document question answering. In Proceedings of the AAAI Conference on
Artificial Intelligence, 2024. URL https://arxiv.org/abs/2308.11730.

Shitao Xiao, Zheng Liu, Peitian Zhang, Niklas Muennighoff, Defu Lian, and Jian-Yun Nie. C-pack:
Packaged resources to advance general chinese embedding. arXiv preprint arXiv:2309.07597,
2023. URL https://arxiv.org/abs/2309.07597.

Zhilin Yang, Peng Qi, Saizheng Zhang, Yoshua Bengio, William W. Cohen, Ruslan Salakhutdinov,
and Christopher D. Manning. HotpotQA: A dataset for diverse, explainable multi-hop question
answering. In Proceedings of the 2018 Conference on Empirical Methods in Natural Language
Processing (EMNLP), 2018. URL https://aclanthology.org/D18-1259/.

Tao Yu, Rui Zhang, Kai Yang, Michihiro Yasunaga, Dongxu Wang, Zifan Li, James Ma, Irene Li,
Qingning Yao, Shanelle Roman, Zilin Zhang, and Dragomir Radev. Spider: A large-scale
human-labeled dataset for complex and cross-domain semantic parsing and text-to-SQL task.
In Proceedings of the 2018 Conference on Empirical Methods in Natural Language Processing
(EMNLP), 2018. URL https://aclanthology.org/D18-1425/.

Nan Zhang, Prafulla Kumar Choubey, Alexander Fabbri, Gabriel Bernadett-Shapiro, Rui Zhang,
Prasenjit Mitra, Caiming Xiong, and Chien-Sheng Wu. SiReRAG: Indexing similar and related
information for multihop reasoning. In International Conference on Learning Representations
(ICLR), 2025. URL https://openreview.net/forum?id=yp95goUAT1.

Dengyong Zhou, Jiayuan Huang, and Bernhard Schölkopf. Learning with hypergraphs: Clustering,
classification, and embedding. In Advances in Neural Information Processing Systems (NIPS), 2006.

Ziyuan Zhuang, Zhiyang Zhang, Sitao Cheng, Fangkai Yang, Jia Liu, Shujian Huang, Qingwei Lin,
Saravan Rajmohan, Dongmei Zhang, and Qi Zhang. EfficientRAG: Efficient retriever for multi-hop
question answering. In Proceedings of the 2024 Conference on Empirical Methods in Natural
Language Processing (EMNLP), pp. 3392-3411, 2024. URL https://arxiv.org/abs/2408.04259.
