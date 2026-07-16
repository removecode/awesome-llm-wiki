---
title: "WikiKV：面向层次化知识导航的模式演化路径索引存储"
original_title: "WikiKV: Schema-Evolving Path-Indexed Storage for Hierarchical Knowledge Navigation"
source_url: "https://arxiv.org/abs/2606.14275"
authors:
  - Feifei Li
  - Haoliang Ming
  - Zihan Li
  - Hang Liao
  - Xingyu Fan
  - Xiaoqing Wu
  - Chenggong Wang
  - Wenhui Que
---

> 本文为英文论文中文译本，仅供阅读参考。原文见 source_url。

# WikiKV：面向层次化知识导航的模式演化路径索引存储

Feifei Li\*, Haoliang Ming\*, Zihan Li\*, Hang Liao, Xingyu Fan, Xiaoqing Wu,
Chenggong Wang, Wenhui Que†  
WeChat, Tencent Inc., Beijing, China  
`{niyali, hliangming, muselli, caryliao}@tencent.com`  
`{fanxfan, xiaoqingwwu, jevanwang, victorque}@tencent.com`

\* 这些作者对本文贡献相同。  
† 通讯作者。

## 摘要

由 LLM 策划的层次化知识库，即一种树结构 wiki，其中节点对底层语料进行总结，已经成为检索增强应用的主导基底；然而，其存储层仍常被视为实现细节。此类工作负载具有层次化、查询密集且持续演化的特点，
而现有存储模型无法同时原生刻画这三种属性。

我们提出 **WIKIKV**，一种专为该工作负载构建的路径索引键值存储模型，包含三个组成部分：（i）一种数据驱动的模式，它通过意图锚定模式归纳（Intent-Anchored Schema Induction）
启动层次结构，并通过连续演化算子对其进行细化；（ii）一种用于路径索引存储模型的一致性协议，它在并发离线重写期间无需读路径加锁即可排除部分读观测；以及（iii）一种预算化导航算子，其搜索加速路由将期望的 LLM
辅助下降步数从 $d$ 降至 $O(1)$，同时保留 anytime 语义，并以逐步细化的答案形式返回结果。

我们通过微信公众账号 AI 助手的真实部署评估 WIKIKV，并在 AUTHTRACE 数据集上将其与多种基线比较。结果表明，相比关系型、图和文件系统后端，WIKIKV 在四类查询算子上实现了均衡的低单算子延迟；
在端到端答案正确率上达到 63.2%，超过多个 RAG 基线，且在低 fan-in 与高 fan-in 多文档问题上的差距进一步扩大。消融研究进一步验证了 WIKIKV 各组件的有效性。

**索引词**——层次化知识库，键值存储，路径索引，模式演化，LLM 策划 wiki，导航查询，检索增强生成。

## I. 引言

在大语言模型（LLM）[1]–[3] 与数据系统的交叉处，一种新的知识管理范式正在形成。生产应用不再只是为检索增强生成（RAG）[4], [5] 索引扁平文档集合，而是越来越多地将非结构化语料编译为**层次化知识库**：
一种树结构 wiki，其节点对底层材料进行总结和组织——这呼应了 Wikidata [6]、DBpedia [7] 和 Freebase [8] 等社区构建知识库中的策划结构。
终端用户查询随后通过从全局索引经由维度下降到单个实体页面和文档的方式来回答，通常还会在该结构上通过智能体式的推理—行动循环完成导航 [9]。这一模式已经支撑了 LLM 策划助手、企业知识门户和内容创作工具，
使存储层处于每次在线查询的关键路径上。

该工作负载有三种属性，使其区别于文档检索和通用图处理。第一，访问是**层次化**的：每次读取都会遍历固定模式

```text
Index -> Dimension -> Entity -> Digest -> Document
```

第二，工作负载具有**读写不对称**性：在线流量在严格延迟 SLA 下是只读的，而写入被批处理到离线流水线中，用于构建和演化 wiki。第三，知识库是**持续演化**的：随着语料增长和访问统计暴露陈旧内容，页面会被添加、
合并和修正。因此，面向这种范式的存储系统需要低延迟路径查找和目录列表，需要在并发离线重写下提供快照一致读取，并需要一等公民式的模式演化能力。

然而，没有现有存储模型原生支持这种访问模式。关系数据库 [10] 需要递归 `JOIN` 或 CTE 才能处理深路径，将 $O(1)$ 导航变成 $O(\text{depth})$，其固定模式也无法容纳多样化的
wiki 结构。Neo4j 等图数据库 [11] 可以建模节点和边，但缺少“目录列表”原语，与“读取页面并枚举子节点”的契约不匹配，且操作开销较高。扁平键值存储 [12]–[14] 提供 $O(1)$ 查找，
但没有层次语义，枚举时必须依赖扫描或二级索引。层次化文件系统 [15], [16] 在结构上相近，但它们针对大型顺序字节流优化，而不是针对并发随机读取下细粒度、可查询的目录元数据；其目录列表作为不透明枚举暴露，
而不是作为一等的、负载有界的查询原语。因此，实践者往往拼接向量索引 [17], [18]、关系型元数据和定制缓存等临时栈，但没有任何一层端到端地捕获层次语义。

我们引入 **WIKIKV**，一种专为层次化知识库设计的**路径索引**键值（KV）存储模型。WIKIKV 将 wiki 模式直接编码进键空间，使一次目录列表可以通过目录节点上的单次点查找完成，在存储往返上为
$O(1)$。在这一存储核心之上，WIKIKV 提供：（i）一个数据驱动模式层，从语料样本初始化层次结构，并通过合并与拆分算子持续演化，同时辅以内容级 Error Book 进行跨批次自纠错；（ii）一个一致性协议，
在读写并发流量下防止部分读；以及（iii）一个预算化导航查询算子，使用搜索加速路由将多步下降压缩为 $O(1)$ 个 LLM 辅助跳转，同时保持渐进式答案质量。

WIKIKV 是我们更广泛的 LLM 策划 wiki 计划在数据库工程侧的实现。我们在先前工作 [19] 中提出了应用层检索方法，该方法在多个公开基准上取得了最先进结果（详见 [19]）。本文聚焦于存储、部署与演化侧。
应用层检索行为和内容级 Error Book 均承接自先前工作 [19]，并在本文中重新落到存储层之上；本文的贡献是数据库层组件（路径即键存储、一致性协议、三级缓存和模式优化形式化），以及 §VI-D
中端到端研究所报告的完整流水线在 WIKIKV 上运行，以验证存储层能保留其答案质量。

WIKIKV 已成功部署到生产环境，为微信公众账号 AI 助手提供服务，将微信公众账号文章组织为个人知识库，使 AI 助手能够基于这些知识库回答关注者的问题。按每个账号包含数百到数千篇文章和知识页面估算，
该系统能够支持数百万账号作者构建个人知识库。

我们的主要贡献总结如下：

- **数据驱动的模式设计与演化。** 我们将层次化 wiki 构建形式化为一个受约束的模式优化问题，并提出用于冷启动初始化的算法，以及通过互信息驱动合并和 Architect–Critic–Arbiter
  拆分实现连续演化的算法；同时辅以内容级 Error Book，使其可跨完整与增量摄取运行持久存在。
- **路径索引存储模型。** 我们提出路径即键编码与结构化值模式，并证明在配套的“先子后父”写入协议下，每个在线查询都会观察到一致且无部分读的 wiki 视图。点查找和目录列表都以 $O(1)$ 个存储往返完成。
- **搜索加速导航查询。** 我们定义预算化导航查询 $\mathrm{NAV}(q,B)$ 及其渐进语义，并展示搜索加速路由如何将单目标查询中 LLM 驱动导航步数的期望从 $O(\text{depth})$ 降至
  $O(1)$，同时在预算提前耗尽时仍返回可用的粗粒度答案。

我们展示了 WIKIKV 在关系型、图和文件系统后端的单算子延迟上优于代表性基线，并在 AUTHTRACE 数据集上相对 RAG 基线取得最先进的端到端答案正确率。

本文其余部分组织如下。第二节形式化层次化知识库模型、查询模型和一致性要求。第三节描述数据驱动模式冷启动和连续演化。第四节给出 WIKIKV 存储模型并证明其一致性保证。第五节介绍预算化导航查询及其搜索加速执行计划。
第六节报告实验研究。第七节综述相关工作，第八节总结全文。

## II. 预备知识与问题形式化

### A. 层次化知识库模型

**层次化知识库**（下称 **wiki**）是一个有序根树 $T=(V,E)$，其顶点集被划分为五个有序层级：

$$
V = V_I \cup V_D \cup V_E \cup V_{DI} \cup V_{DO},
$$

分别对应节点类型 Index、Dimension、Entity、Digest 和 Document。唯一根节点 $r \in V_I$ 是全局索引，内部节点属于 $V_D$，叶子节点属于
$V_E \cup V_{DI} \cup V_{DO}$。每个节点 $v \in V$ 携带三类状态：路径 $\pi(v)$、内容负载 $c(v)$ 和元数据记录 $m(v)$。路径

$$
\pi(v) = /d_1/d_2/\cdots/d_n
$$

是从 $r$ 到 $v$ 的有向路径上边标签的唯一序列，编码为斜杠分隔的字符串；它既是人类可读地址，也在第四节中作为存储键。我们用深度预算 $D$ 约束每个节点，满足 $\mathrm{depth}(v)\le D$。
该模式有五种节点类型（Index、Dimension、Entity、Digest、Document）；在我们的部署中每种类型占据一层，因此 $D=5$。$D$ 可以取任意值，但实际有效性偏向较小的 $D$（通常为 5）。
我们默认取 $D=5$，因为五种节点类型已经覆盖完整的

```text
index -> dimension -> entity -> digest -> document
```

抽象链；更大的 $D$ 只会插入额外的 Dimension 路由层级，既不提升覆盖率，也不提升答案质量，却会拉长每次导航下降。存储模型本身与深度无关，但第五节中的算子依赖这一界限进行延迟分析。

### B. 查询模型

层次化知识库上的真实工作负载可归约为四种算子，它们共同捕获了 LLM 策划知识应用中的访问模式，如图 1 所示：

- **Q1：路径查找。** $\mathrm{GET}(\pi)\to v$ 返回路径 $\pi$ 指向的节点，若无此节点则返回 $\bot$。
- **Q2：目录列表。** $\mathrm{LS}(\pi)\to (v,\langle \pi_1,\ldots,\pi_k\rangle)$ 返回 $\pi$ 处的节点以及其子节点路径的有序列表，其中 $k$ 是 $v$ 的扇出。
- **Q3：导航查询。** $\mathrm{NAV}(q,B)\to\langle r_1,r_2,\ldots,r_m\rangle$ 接收自然语言查询 $q$ 和时间预算 $B$，并返回在预算内沿层次结构下降产生的一组结果记录。
- **Q4：前缀搜索。** $\mathrm{SEARCH}(p)\to\{\pi: p \text{ is a prefix of } \pi\}$ 返回文本前缀匹配 $p$ 的路径集合；它用于加速 NAV 的入口步骤。

NAV 的语义与标准树遍历有一个关键差异，我们将在第五节利用这一差异。

**性质 1（渐进式答案）。** 若 $\mathrm{NAV}(q,B)$ 在第 $i$ 步后被中断，则前缀 $\langle r_1,\ldots,r_i\rangle$ 仍是 $q$ 的一个有效答案，尽管粒度更粗。
等价地，结果按粒度单调增加的顺序发出，因此输出的任意前缀本身都是可用答案。

**图 1：层次化知识库上的四种查询算子。** 底层引擎中实际存储的物理 KV 键是路径 $\pi(v)$ 的哈希摘要 $H(\pi(v))$。

### C. 一致性要求

wiki 被在线查询流量读取，并由离线构建与演化流水线写入。实现上述模型的任何存储层都必须提供以下保证。

**R1：写后读一致性。** 一旦新页面 $v$ 被接纳进 wiki，随后在线查询返回的每个 $\mathrm{LS}(\pi(\mathrm{parent}(v)))$ 目录列表都必须包含 $\pi(v)$，使得 $v$ 可通过导航到达。

**R2：读写不对称下的并发安全。** 在线流量严格只读，构建流水线是唯一写者。存储层必须保证读者永远不会观察到部分写状态，理想情况下无需显式加锁；流水线发出的页面级增量更新必须遵守同样属性。

**R3：有界陈旧性。** 离线写入完成后，在线读者必须在有界陈旧窗口 $\Delta$ 内观察到其效果；超过该窗口后，新状态应全局可见。

这三项要求将在线读可用性与离线写成本和复杂性解耦。第四节给出的写协议和查询侧回退共同在不锁定读路径的情况下满足 R1–R3。

## III. 数据驱动的模式设计与演化

我们处理从语料中冷启动 wiki 模式，以及使其持续演化这两项挑战。我们将模式设计形式化为受约束的优化问题，提出用于初始化的意图锚定归纳算法，并引入两个带单调改进保证的模式演化算子，同时辅以内容级 Error Book
进行跨批次自纠错。这些组件被集成到离线构建与演化流水线中，以支持动态模式适配。

### A. 动机

生产知识库会超出人工策划的能力范围：即便一个设计良好的五层模式，一旦语料规模跨越一个数量级，也会失去平衡。根据我们的部署经验，缺少有原则重塑机制的 wiki 在超过 $10^4$ 个 KV 对后即难以维护。
我们观察到三种同时发生的漂移形式。**宽度漂移**会增加新的顶层维度，迫使每个导航查询在第一跳即区分大量 Index 兄弟节点，并膨胀 LLM 路由难度和索引原语常数。**深度与密度漂移**使页面和边大致随语料线性增长，
膨胀物化 KV 足迹，并把目录列表推到不可操作的扇出规模。**质量漂移**会累积陈旧内容、低置信断言和从未被读取的条目，提高检索噪声底线而不增加召回。

### B. 将模式设计视为受约束优化问题

我们将模式设计视为在深度至多为 $D$ 的有效 wiki 空间上的全局优化问题（五种节点类型：Index、Dimension、Entity、Digest、Document；在我们的系统中 $D=5$，如 §II-A
所述）。令 $S=(V,\ell,E)$ 表示候选模式，其中 $V$ 是节点集，$\ell:V\to\{I,D,E,DI,DO\}$ 将每个节点分配到五种节点类型之一，$E$ 编码父子结构。令 $\rho$
表示在线查询工作负载在 $V$ 上诱导的访问分布，令 $W$ 表示 §II-B 形式的工作负载。我们用如下代价为每个候选模式打分：

$$
C(S;W)=\alpha|V|+\beta\sum_{v\in V}\mathrm{depth}(v)\cdot\rho(v)-\gamma Q(S;W),
\tag{1}
$$

其约束包括结构约束 $\mathrm{depth}(v)\le D$，以及每节点扇出界 $|\mathrm{children}(v)|\le k_{\max}$。内容级质量由本节稍后介绍的 Error Book
修复循环单独保证，而不是作为硬模式约束。

式（1）中的每一项都在本文其他位置有直接对应。存储项 $\alpha|V|$ 衡量物化 KV 命名空间大小，其布局将在第四节固定。下降深度项 $\beta\sum_v \mathrm{depth}(v)\rho(v)$
是访问加权遍历成本，它主导严格预算查询中的在线导航延迟。质量项 $Q(S;W)$ 是工作负载在经验上暴露出的端到端答案正确率（第六节）。三个系数 $\alpha,\beta,\gamma>0$ 是部署时超参数，
用于在存储足迹、在线延迟和答案正确率之间权衡。

完整优化关于 $|D|$ 呈超指数复杂度，因此我们不尝试全局求解。相反，我们采用**贪心局部搜索**：从冷启动模式 $S_0$（§III-C）出发，反复应用 §III-D 的局部算子。我们将在定理 1 中证明，
在适当选择阈值时，每个算子都会单调降低 $C$。

### C. 冷启动：意图锚定模式归纳

冷启动问题是模式设计的零阶实例：给定一个新语料 $D$ 且没有先验结构假设，产出一个满足 §III-B 结构与约束的有效初始模式 $S_0$。我们用一种称为**意图锚定模式归纳**（Intent-Anchored
Schema Induction，IASI）的过程解决该问题。IASI 直接调用 LLM，不依赖任何统计聚类、向量检索或嵌入模型。该过程在部署时运行一次，与任何在线关键路径解耦，并消费一个样本 $S\subset D$，
其大小与 $|D|$ 无关。

**意图锚定模式归纳（IASI）过程。** 首先，IASI 消费样本 $S$ 并生成一个**语料定位描述符**

$$
P=\langle \mathrm{focus},\mathrm{audience},\mathrm{ingestion\text{-}bias}\rangle,
$$

它是一个简短结构化记录，说明语料内容是什么、面向谁写作，以及文档进入语料时存在何种偏向。随后，IASI 以 $(S,P)$ 为输入并生成目录脚手架 $T$，固定 $V_I,V_D,V_E$ 以及这些层级中的父子结构
$E$（digest 和 document 节点由摄取流水线动态填充）；该过程携带 §III-B 的结构约束，使树在首次调用时即在语法和结构上合法，而不是通过事后“生成再验证”循环被拒绝并重新采样。

作为对比，最直接的冷启动基线会把文章样本交给 LLM，并用“请生成一个目录”之类的提示解析其响应；这种方式得到的目录会锚定在前几篇样本文档中碰巧突出的显著实体上，长尾实体则被吸收到兜底桶中。与作为 LLM
消费后丢弃的临时中间字符串不同，$P$ 是一等模式对象，会与目录树一起物化到持久存储中，并由 §III-D 的后续演化算子直接读取。

**非均匀采样。** 样本 $S$ 并非从 $D$ 中均匀抽取。一个轻量级**摄取过滤器** $\Phi$ 在采样前运行，移除七类低信息文档，例如模板化节日问候、对上游内容的逐字转载和活动公告。由于 $P$ 由 LLM
对 $S$ 的观察构建，大量低信息文档会系统性地使 $P$ 偏向不合适的抽象。采样前过滤可以从源头防止这种误校准，而不是稍后再修正。

### D. 连续演化算子

随着语料增长、访问分布 $\rho$ 变化或页面质量下降，冷启动模式 $S_0$ 会立即偏离 §III-B 的最优点。我们定义两个局部算子，让 $S$ 每次沿 $C$ 的代价曲面下降一步。
每个算子只消费存储层已在每个页面上携带的统计信息（第四节），不需要外部使用日志或分析仓库。

**算子 1：DIMENSION MERGE（互信息驱动）。** 对于共享同一父节点的两个兄弟内部节点 $v_1,v_2$，定义按查询的共访问指示变量
$X_i=\mathbb{1}[\text{a query touches }v_i]$，以及互信息

$$
\mathrm{MI}(v_1,v_2)
=\sum_{x_1,x_2} p_{12}(x_1,x_2)
\log\frac{p_{12}(x_1,x_2)}{p_1(x_1)p_2(x_2)},
\tag{2}
$$

该值直接从与每个页面记录共址的 `access_count` 统计估计。当 $\mathrm{MI}(v_1,v_2)>\theta_{\mathrm{merge}}$ 时，我们将 $v_1$ 和 $v_2$
合并为单个节点 $v_{12}$：子列表取原子列表的并集，`access_count` 求和，内容为原摘要拼接。操作上的解释是：若两个兄弟节点访问模式高度互相关，则说明用户心智模型把它们视为同一概念；
继续分离只会拓宽索引并增加导航算子的路由负担。

**算子 2：PAGE SPLIT（Architect–Critic–Arbiter）。** 我们通过三个角色形式化 PAGE SPLIT 算子，具体实例化如下。**Architect**
通过基于规则的触发器提出局部候选集 $E_e$，并让 LLM 作为局部 oracle，在以下任一条件下调用：（i）$\mathrm{length}(e)>l_{\max}$；或（ii）单次 LLM 调用判定 $e$
可以拆成可分离的实体子树。

**Critic** 为每个 $e\in E_e$ 分配估计代价变化

$$
\widetilde{\Delta C}(e;W)
=\alpha\Delta|V|+\beta\Delta(\mathrm{depth}\cdot\rho)-\gamma\Delta\widetilde{Q},
\tag{3}
$$

其中 $\widetilde{Q}$ 用每页记录中共址的 `access_count` 和 `confidence` 统计近似 $Q(S;W)$（§IV-B）。

**Arbiter** 选择提交集 $C_t\subseteq E_e$：

$$
C_t=\{e\in E_e:\widetilde{\Delta C}(e;W)<0\land\mathrm{Safety}(e)\},\quad |C_t|\le K,
\tag{4}
$$

其中 $\mathrm{Safety}(e)$ 要求 $S_t$ 中可达的每个实体在 $S_{t+1}$ 中仍可达，而 $K$ 限制每轮提交数量。

与 DIMENSION MERGE 一样，拆分会降低每节点扇出，并降低原先集中在 $v$ 上的访问质量所对应的 $\beta\sum_v \mathrm{depth}(v)\rho(v)$，代价是 $|V|$
增加一个单位，并由 $\alpha\Delta|V|$ 直接计费。

**定理 1（单调改进）。** 令 $C$ 为模式代价（1），且有下界。如果一个算子通过（4）的接受测试 $\Delta C(e;W)\le 0$，则称其为**可接受**；并假设质量项 $Q$ 在节点不相交支撑上可分离，
使得不相交算子对 $\Delta C$ 的贡献可加。若每轮提交一组节点不相交的可接受算子 $C_t$，则 $C$ 沿贪心轨迹

$$
S_0\to S_1\to\cdots
$$

非增，并且收敛。

**证明。** $C$ 的三项都是节点上的求和；由可分离性，质量项在不相交支撑上分解。因此对于节点不相交的提交，有

$$
\Delta C(C_t)=\sum_{e\in C_t}\Delta C(e),
$$

每个求和项都因可接受性而不大于 0。所以 $\{C(S_t)\}$ 非增且有下界，因此收敛。□

**内容级自纠错（Error Book）。** 作为上述模式级算子的补充，我们的离线流水线运行一个内容级自纠错循环，即 Error Book，该机制承接自先前工作 [19]。DIMENSION MERGE 和 PAGE
SPLIT 作用于命名空间的结构形态，而 Error Book 作用于单个记录内容（例如悬空 wikilink、格式错误的来源引用、无支撑事实、跨页面矛盾）：检测到的错误模式被累积为约束规则，并注入后续摄取提示；
两层修复——确定性代码级修复与周期性 LLM 修复——会减少新错误和既有错误。Error Book 与 wiki 一起持久化，并在完整与增量摄取运行之间复用，因此早期运行累积的约束会在后续运行中继续生效。
本文的一个贡献是将 Error Book 重新落到存储层之上，使其约束状态和修复共享与上述模式算子相同的路径键记录和按作者构建流水线。

### E. 与离线流水线集成

冷启动过程、两个模式演化算子和 Error Book 都作为**离线后台作业**执行，并与存储层共享同一个构建与演化流水线。流水线按以下节奏运行：冷启动只执行一次；DIMENSION MERGE 和 PAGE
SPLIT 每摄取 $N$ 篇文章触发一次（在我们的部署中 $N=30$），用于吸收访问分布漂移和语料增长；Error Book 在每个摄取批次后运行（确定性代码级修复），并配合周期性 LLM 级修复循环，其状态跨完整与增量运行持久保存。

## IV. WIKIKV 存储模型

我们提出 WIKIKV，一种路径索引存储模型，它将 wiki 模式直接物化到下层 KV 存储的键命名空间中，如图 2 所示。

**图 2：WIKIKV 中的路径即键编码和值模式。** 路径以逻辑形式展示；底层引擎中实际存储的物理 KV 键是哈希摘要 $H(\pi(v))$。

### A. 路径即键编码

WIKIKV 通过逐字使用每个节点的路径 $\pi(v)$ 作为逻辑键来编码 wiki 模式。因此，五层层次结构被折叠为一个自描述命名空间，其绑定如表 I 所示。根索引绑定到字面键 `"/"`。维度索引绑定到单段键
`"/d"`。Entity 既包含在维度目录节点下，又作为页面文件包含指向原始 digest 和 document 的链接。

Digest 和 document 被刻意**不**嵌套在每个 entity 下面。由于一篇源文章通常会支撑跨多个维度的多个实体，如果将其 digest 和全文复制到每个引用实体下，就会重复存储大部分字节。因此，
我们把所有来源提升到一个共享子树中（`"/sources/digests/·"`、`"/sources/articles/·"`），无论有多少实体引用它们都只物化一次；每个实体页面**链接**到相关源路径，
而不是嵌入其内容。因此，一个被 $k$ 个实体共享的来源只物化一次，而不是 $k$ 次。相应地，语料增长只发生在持久层；内存缓存足迹由容量上限固定（§V-C），与语料规模无关。

路径 $\pi(v)$ 是全文使用的逻辑地址，但**物理** KV 键是其哈希摘要：

$$
\mathrm{key}(v)=H(\pi(v)).
$$

目录记录中公布的子路径（§IV-B）按需哈希。哈希产生固定宽度、与分隔符和字符集无关的键，规避了原始路径字符串编码陷阱，例如非 ASCII（如中文）路径段在不同后端中的字节长度和排序差异。

**表 I：路径即键绑定。** 底层引擎中实际存储的物理 KV 键是哈希摘要 $H(\pi(v))$。

| 节点 | 逻辑路径 $\pi(v)$ | 绑定到 |
| --- | --- | --- |
| Index | `"/"` | 根索引 |
| Dimension | `"/d"` | 维度索引 $d\in V_D$ |
| Entity | `"/d/e"` | 实体页面 $e\in V_E$ |
| Digest | `"/sources/digests/title"` | 文章摘要 $di\in V_{DI}$ |
| Document | `"/sources/articles/title"` | 文章文档 $do\in V_{DO}$ |

为使路径相等性无歧义，我们对路径进行规范化：无尾随斜杠，路径段大小写敏感匹配，路径段内不得包含保留分隔符，并由模式常数 $D$ 约束深度。规范化在哈希前运行，因此一条路径既是树地址，又通过 $H(\pi(v))$
成为其存储键，无需单独的转换表。

### B. 值模式

内部节点（Index、Dimension）存储为**目录记录**，叶节点（Entity、Document 和 Digest）存储为**文件记录**。

**目录记录。** 目录记录包含三部分。它通过 `type="dir"` 标识节点，并用 `name` 保存相对于父节点的路径段；它用两个并列数组枚举子节点：`sub_dirs`（子目录）和 `files`（叶文件）；
并携带包含统计的 `meta` 块：`updated_at`、`entry_count` 和 `access_count`。

**文件记录。** 文件记录采用类似布局：`type="file"` 与 `name`，一个 UTF-8 `text` 负载，以及记录以下字段的 `meta` 块：单调递增的 `version`、位于 $[0,1]$
的 `confidence` 分数、URI 组成的 `sources` 列表、`last_verified` 时间戳和 `access_count`。

**设计依据。** 有两点值得强调。第一，由于每个目录记录显式命名其可达子节点，$\mathrm{LS}(\pi)$ 通过 $\pi$ 处的单次点查找即可完成，无需前缀扫描或辅助索引；也即在常见情况下
$\mathrm{LS}(\pi)\equiv\mathrm{GET}(\pi)$，时间为 $O(1)$。第二，`meta` 计数器（`access_count`、`confidence`、
`last_verified`、`version`）不被这里的存储算子使用，但会馈送第三节中的模式演化算子。

### C. 一致性协议

存储层必须调和两项要求：新接纳页面必须能通过父目录到达（R1），同时与离线写并发运行的读者不能观察到部分写状态（R2）。如图 3 所示，WIKIKV 通过“先子后父”写协议配合宽松读取满足二者，并且不在读路径上显式加锁。

**图 3：一致性协议（无部分读）。** 写协议采用先子后父顺序，读协议采用 miss 时跳过的策略。

**写协议（先子后父）。** 为在路径 $\pi(v)=/d/e$ 接纳新实体 $v$，离线流水线按顺序执行以下两个操作：

1. **子写入。** $\mathrm{PUT}(\pi(v),c(v))$ 将实体记录写入存储。
2. **父更新。** $\mathrm{UPDATE}(\pi(\mathrm{parent}(v)))$ 将路径段 $e$ 追加到 `/d` 维度记录的 `files` 列表。

若步骤 2 失败，$\pi(v)$ 处的孤儿记录仍留在存储中，但尚未从任何目录列表链接。对既有实体进行原地重写的页面级更新遵循同样模式，只是步骤 2 通常只刷新父节点上的 `meta` 统计，因此是无操作。

**读协议（miss 时跳过）。** 执行 $\mathrm{LS}(\pi)$ 的读者首先获取 $\pi$ 处的目录记录，然后对其公布的每个子路径发出 `GET`。如果某个子 `GET` 返回 $\bot$，
读者会静默丢弃该条目。结合上述写入顺序，该规则排除了部分写观测，如下一定理所述。

**定理 2（无部分读）。** 在“先子后父”写协议和“miss 时跳过”读协议下，并假设跨键可见性单调（观察到父更新的读者也能观察到先前的子写入），对 $\pi$ 的 `GET`/`LS` 永远不会返回一个被
$\pi$ 处记录公布、但其自身记录缺失的子节点。

**证明。** 令读者观察到 $\pi$ 处的目录记录 $R$，令 $\pi(v)$ 为一个新实体。（a）若 $R$ 省略 $\pi(v)$，则步骤 2 尚未提交，所以 $v$ 至多是一个未公布孤儿，不会被获取。（b）
若 $R$ 列出 $\pi(v)$，则步骤 2 已提交；由写入顺序和跨键可见性，步骤 1 在此处已持久且可见，故 $\mathrm{GET}(\pi(v))$ 完整。唯一可能由陈旧缓存列表到达的孤儿会返回 $\bot$，
并在进入结果集前由 miss 时跳过规则丢弃。□

该保证针对我们评估的部署而表述：每个子树一个离线写者，在线层只读。它关注“已公布但缺失的子节点”，而不是完整快照隔离；多个离线写者之间的并发以及缓存竞争下的强隔离委托给底层 TABLEKV 层。

**乐观并发控制。** 页面级更新会重写既有记录，因此我们为每个记录附加单调递增的 `version` 字段，并将其作为比较并交换（compare-and-swap）令牌。发现期望版本陈旧的写者会中止，并用最新值重试。
由于工作负载在在线层只读且离线流水线是唯一写者，实践中的写写冲突很少，少量有界重试即可满足需求；因此我们不需要更强的悲观锁层。

**多进程并行构建。** 离线流水线通过按作者边界划分写负载来扩展：每个作者语料编译为独立子树，不同作者没有共享路径，因此写集合在构造上互不相交。构建由此成为**按作者并行、作者内串行**：一组 worker
每次拥有一个作者，并按顺序应用其摄取和演化步骤（在子树内保持先子后父顺序和 version-CAS），而不同 worker 可并行推进且无需跨作者协调。除已由 OCC 处理的子树内冲突外，这不会引入新的写写冲突，定理
2 继续在每个子树内成立。由于每个摄取批次在数秒内完成，一个适度规模的 worker 池即可支撑为数百万公众账号作者构建和维护 wiki 所需的吞吐。

## V. 预算化路径导航查询

基于存储层的键值原语，本节介绍预算化导航查询算子 $\mathrm{NAV}(q,B)$。该算子将多步下降编译为 $O(1)$ 个搜索加速、LLM 辅助跳转，并配合路径键三级缓存，在知识库规模变化时保持稳定尾延迟。

### A. 查询语义

导航查询算子 $\mathrm{NAV}(q,B)$ 是 §II-B 中唯一没有映射到单个存储原语的算子：给定自然语言查询 $q$ 和墙钟预算 $B$，它返回一个**有序**序列

$$
R=\langle r_1,\ldots,r_m\rangle,\quad m\le\lceil B/b\rceil,
$$

其中 $b$ 是主导的单步延迟（最坏情况下为 LLM 辅助下降，最好情况下为 `GET`）。因此，该算子是 anytime 的：增大 $B$ 可能产生更长、更细粒度的序列，而不会使较小预算下返回的前缀失效。

如图 4 所示，NAV 区别于通用 anytime 遍历之处在于 §II-B 性质 1 中的**渐进**契约；在这里，它被专门化到 wiki 模式：记录按粒度单调增加的顺序发出，并与层次结构层级对齐。

**图 4：导航查询算子的流程图，带预算有界中断契约。**

- $r_1$ 是 **Index 级摘要**，即系统可返回的最粗答案；例如“该 wiki 包含 $N$ 个维度：Personal Relationships、Writing Style、……”
- $r_2$ 是受 $q$ 所选维度限制的 **Dimension 级摘要**；例如“Personal Relationships 包含三个实体：Family、Mentors and Friends、Polemic Opponents”。
- $r_3$ 及之后是 **Entity 或 Article 级页面**；例如“The estrangement between Zhou Zuoren and Lu Xun occurred in 1923 ...”。

对于任意前缀长度 $i\le m$，$\langle r_1,\ldots,r_i\rangle$ 都是 $q$ 的有效答案（尽管更粗）。因此，该算子允许**预算有界中断**契约：预算耗尽时，已累积的部分序列按原样返回。
这也解释了执行计划中的发出顺序和预算检查保护。

### B. 搜索加速路由

字面逐层执行 $\mathrm{NAV}(q,B)$ 会在每一层调用一次 LLM 来选择下降到哪个子节点，在第一个实体级记录之前需要 $D$ 次串行 LLM 调用。路径即键命名空间可以绕过这种下降：
对路径命名空间执行词法关键词/前缀搜索 $\mathrm{SEARCH}(p)$（§II-B）会返回候选目标路径，它们已经近似定位到树的正确区域，且无需每层 LLM 调用。路由器作用于**文本路径键**，
而不是稠密向量索引；若使用向量检索，也只限于叶内容算子 Q4，并与路径路由正交。因此，该计划有两个阶段：阶段 1 通过一次搜索调用选择 $k$ 个候选路径（常数个 KV 往返，独立于 $D$）；阶段 2
执行定向点查找和可选的单层扩展。两个阶段都由预算检查保护，使算子可以在任意时刻返回一个连贯前缀。

**工程组件。** 该算法包含五个轻量组件，每个都是显式伪代码步骤，而非不透明子过程。

- `CLASSIFY(q)` 是一个混合路由器，结合用于枚举触发词（“which ...”、“list ...”）的正则表达式层，以及用于歧义查询的小型蒸馏分类器，最多增加 5 ms。其路由类别 `cls` 驱动阶段
  1：枚举查询由单次目录列表回答，其余查询转交搜索路由器。
- `SEARCH(EXTRACT(q))` 从 $q$ 中抽取候选页面名关键词，并在路径命名空间上运行前缀扫描，返回候选文件路径。
- `NEEDSDEEPER(q,v)` 将 $q$ 与候选的内容 $c(v)$ 比较，只有当其对 $q$ 的语义覆盖低于阈值 $\theta$ 时才返回 true；它可以是轻量分类器或单次 LLM 调用。
- `BUDGETEXHAUSTED(t_0,B)` 是预算执行保护，在每个潜在昂贵步骤前评估。
- 候选集上的多路径探索默认串行执行，以使预算保护保持权威；当剩余预算足以摊销调用开销时，阶段 2 中每个候选的 `GET` 调用可以并行发出。

**算法。** 算法 1 给出执行计划，其中 $t_0$ 为开始时间，`cls` 为 `CLASSIFY` 输出的路由类别，$\Pi$ 为阶段 1 候选集，$R$ 为累积结果。枚举类查询会短路为单次根列表，
绕过关键词搜索；所有其他类别进入搜索加速路由。第 8 行和第 17 行是预算保护，第 15 行在候选页面本身不能覆盖 $q$ 时执行可选单层扩展。

**算法 1：$\mathrm{NAV}(q,B)$：搜索加速导航**

```text
Require: 自然语言查询 q，时间预算 B（ms）
Ensure: 渐进结果序列 R

1:  t0 <- NOW(); R <- <>
2:  cls <- CLASSIFY(q)                    {<= 5 ms 的混合路由器}
3:  // 阶段 1：搜索加速路由
4:  if cls = ENUMERATE then
5:      return <LS("/")>                  {枚举查询由单次目录列表回答}
6:  end if
7:  Pi <- TOPK(SEARCH(EXTRACT(q)), k)     {k = 3 个候选路径}
8:  if BUDGETEXHAUSTED(t0, B) then
9:      return <LS("/")>                  {最粗回退}
10: end if
11: // 阶段 2：定向导航
12: for all pi in Pi do
13:     v <- GET(pi); R.APPEND(v)
14:     if NEEDSDEEPER(q, v) then
15:         R.APPEND(LS(pi))
16:     end if
17:     if BUDGETEXHAUSTED(t0, B) then
18:         break
19:     end if
20: end for
21: return R
```

**定理 3（步数压缩）。** 令 $D$ 为 wiki 深度，令 $h$ 表示从阶段 1 返回的候选路径到达目标文件所需的后路由跳数。在算法 1 下，单个目标路径上的 LLM 辅助下降步数从纯逐层导航中的 $D$
降为 $h$，其中对于单目标查询有 $h\in\{0,1\}$（候选已覆盖 $q$，或通过 `NEEDSDEEPER` 一次扩展即足够），当 $q$ 需要跨 $k$ 个维度聚合证据时有 $h\le k$。

**证明。** 逐层导航需要每层一次 LLM 调用，即到达深度 $D$ 需要 $D$ 次。算法 1 中，阶段 1 发出一次路由调用，随后一次 `SEARCH` 用独立于 $D$ 的常数 KV 往返替代前 $D-h$ 层。
在剩余部分中，只有 `NEEDSDEEPER` 位于关键路径上，给出 $h$ 次后路由下降调用：若候选覆盖 $q$ 则为零；若一次 `LS` 扩展足够则为一次；若 $q$ 跨越 $k$ 个维度则至多为 $k$ 次。
因此下降步数等于 $h$（即命题中数量），端到端计数为路由调用加这些调用，即 $1+h$；二者均独立于 $D$。□

### C. 缓存策略

§II-B 的查询模型把每个 `GET` 视为固定 KV 往返，但在生产中我们观察到强烈偏斜的访问分布：根索引和少数维度页面几乎每次查询都会读取，而深层实体页面读取较少，但必须为预算化下降保持可用。
这种偏斜促使我们设计一个三级缓存层次，使用与底层存储相同的路径命名空间作为键，使缓存查找与存储查找共享同一键派生。三个层级按容量递增、期望命中率递减排列。

**L1——进程内缓存（容量：数十页）。**

- **内容。** 根索引节点 `"/"` 和每个维度节点 `"/d"`。
- **策略。** 进程启动时预热，进程生命周期内永不过期；在下文所述缓存失效事件上刷新。
- **依据。** 通过将命名空间中最频繁访问的前缀移出网络路径，约束阶段 1 的最坏情况延迟。

**L2——共享 Redis 层（容量：数千页）。**

- **内容。** 完整目录节点以及热实体节点子集，热度由每个文件 `meta` 记录中的 `access_count` 统计识别（§IV-B）。
- **策略。** LRU 驱逐与一小时 TTL，使移出工作集的页面即便没有显式失效也最终被回收。
- **依据。** 吸收 L1 容纳不下的维度级和实体级长尾流量，并作为所有在线查询 worker 共享的跨进程缓存。

**L3——KV 存储（容量：完整 wiki）。**

- **内容。** 完整 WIKIKV 键空间，包括完整目录节点和文件节点。
- **策略。** 持久后备层；服务任何 L1 和 L2 未命中的请求。
- **依据。** 充当权威事实源，以及缓存失效协议的一致性锚点。

**有界内存足迹。** 只有 L1 和 L2 驻留内存，且二者都由固定页面预算限制（分别为数十页和数千页），并采用 LRU+TTL 驱逐；因此常驻工作集由访问偏斜而非语料规模约束。文章数量增长只会扩大持久 L3
层（磁盘/SSD），所以随着 wiki 扩展，内存使用保持平坦。我们有意不给 L3 附加过期：策划知识库是权威且持久的，内容陈旧性由缓存失效流和 Error Book / 模式演化算子主动处理，而不是通过被动过期已存储知识来处理。

**缓存失效。** 离线构建与演化流水线在 §IV-C 的“先子后父”协议成功完成的每次写入上发布路径键失效事件。在线层订阅该事件流，并异步刷新其键等于或是受影响路径前缀的任何 L1 或 L2 条目。
由于“先子后父”协议保证底层存储中永远不会观察到已公布但缺失的子节点（定理 2），与飞行中读取竞争的失效最多会迫使一次到 L3 的额外往返，但绝不会向应用暴露部分写状态。因此，有界陈旧性要求 R3 得到满足，其中
$\Delta$ 等于离线提交成功与对应缓存刷新回调之间的最大延迟，该延迟与 wiki 规模无关。

## VI. 实验评估

本节在一个生产层次化知识库平台上评估 WIKIKV，并将其与代表性存储后端和代表性检索增强基线比较。

### A. 设置

**数据集与工作负载。** 我们使用 AUTHTRACE [20]，这是一个近期诊断基准，用于在主题密集的单作者语料上评估证据构建。它提供引用证据、每个问题的精确 fan-in 标注，以及一个统一的 pack 级协议，
用于测量证据召回、证据精度和答案正确率。其 **fan-in 梯度**，即支撑一个答案所需的源文档数量，定义了一个主要诊断轴，使检索、记忆、图和结构化证据范式可以在同一基准上比较。我们采用其三个桶：
**single-doc**（证据在一个文档中）、**low multi-doc**（两个文档）和 **high multi-doc**（三个或更多文档）。

**平台。** 在线层服务面向用户的导航查询，而离线构建与演化流水线向存储层写入页面。除 §IV-C 的“先子后父”写协议外，读写路径不共享同步协调。在线层运行在 Redis Cluster 上，持久层运行在
TABLEKV 上；TABLEKV 是微信基于带 PaxosStore 共识的 LSM-tree 引擎构建的分布式 KV 系统，其简单 KV 抽象可水平扩展。所有 LLM 推理均使用 DEEPSEEK-V4-FLASH
[21]。[^deepseek]

[^deepseek]: https://huggingface.co/deepseek-ai/DeepSeek-V4-Flash

**基线。** 存储延迟研究（§VI-B）将 WIKIKV 在 LevelDB 引擎上的路径即键布局与三个替代后端比较：使用目录和文件原语的层次化文件系统（FS）；带 `ltree` 扩展和规范化父子模式的
PostgreSQL；以及代表原生属性图存储的 Neo4j。端到端研究（§VI-D）将 WIKIKV 与检索增强基线比较：No-RAG，即不检索证据直接查询 LLM；Dense-RAG，即在扁平向量索引上的基于嵌入检索；
GraphRAG [22]，即在构建的知识图谱上进行社区摘要检索；以及 RAPTOR [23]，即产生层次化检索树的递归抽象摘要。

**测量协议。** 所有延迟数字都是 10 次独立运行的中位数，且在每次运行前进行 200 查询预热；运行间变化始终低于 1%。为控制 LLM 随机性，每个答案正确率结果都使用贪心解码（temperature 0）
和固定随机种子，因此在给定检索证据时报告分数是确定性的。除非另有说明，下文报告的准确率差异都在按问题分数进行的配对 bootstrap 检验下具有统计显著性（$p<0.01$）。

### B. 存储后端延迟

我们首先在一个中等规模 wiki（约 2,000 个 KV 对）上比较四种存储配置中对应 §II-B 的 Q1–Q4 的单算子延迟，如表 II 所示。对每个算子，我们随机采样 100 个目标路径或前缀，并在 200
查询预热后对每个后端发出 1,000 个查询。路径即键引擎是我们的方法：为将引擎成本与服务层效应隔离，我们在本地 LevelDB 上实现它，暴露与 TABLEKV 相同的 `Put`/`Get` 接口但绕过服务栈，
只反映本地引擎读写成本。其余三个后端——FS、PostgreSQL 和 Neo4j——作为比较点。所有后端都使用受控的进程内、内存常驻设置；绝对延迟因此低于生产环境，但由于每个后端都在同一配置下测量，相对比较仍然公平。

**表 II：MEDIUM wiki 上按后端划分的单算子延迟中位数（P50，毫秒）。**

| 后端 | Q1 | Q2 | Q3 | Q4 |
| --- | ---: | ---: | ---: | ---: |
| FS | 0.021 | 0.436 | 6.480 | 0.091 |
| WIKIKV | 0.017 | 0.088 | 5.411 | 0.085 |
| PostgreSQL | 0.075 | 0.117 | 1.671 | 0.432 |
| Neo4j | 2.494 | 1.230 | 6.397 | 1.686 |

因此，我们的主张并不是路径即键引擎在每个算子上都是最快的——没有后端在全部四个算子上占优——而是它在完整 Q1–Q4 混合上实现了**均衡的低延迟**；相反，每个替代方案都只在部分算子上较快，并在其他算子上付出不成比例的代价。

**Q1（路径查找）。** WIKIKV 和 FS 实现亚毫秒 P50，因为 Q1 在二者中都直接映射到单键获取。即便 `ltree` 键有索引，PostgreSQL 仍为 SQL 解析路径付出小而稳定的开销；
Neo4j 比 WIKIKV 慢两个数量级，因为每个 Q1 都要产生 Bolt 驱动往返和 Cypher 计划编译。数据路径最接近路径即键契约的后端付出最低常数。

**Q2（目录列表）。** WIKIKV 的 P50 最快，因为值记录共址子路径段（§IV-B），使 Q2 归约为单次点查找，而不是前缀扫描。FS 受每个条目的元数据系统调用拖慢；PostgreSQL 只有在
MEDIUM 工作集适合内存且 `ltree` 索引避免递归 CTE 时才具有竞争力。Neo4j 仍需多毫秒常数，因为该操作必须遍历出边并重建 Cypher 结果行。

**Q3（沿已知路径导航）。** Q3 是四个后端差异最明显之处。PostgreSQL 在 P50 上意外最快（1.671 ms），因为模拟导航分解为少量索引路径等值查找，每个查找复用单个客户端连接；在仅约 2,000
行时，索引和基表基本都位于页缓存中，单步 SQL 常数较小。WIKIKV 和 FS 都要通过各自目录层执行多次往返（WIKIKV 需要前缀扫描和 JSON 解析，FS 需要重复元数据调用），Neo4j
还在每个下降步骤额外付出 Bolt 和 Cypher 常数。

**Q4（前缀搜索）。** WIKIKV 和 FS 的 P50 最强，因为其字典序键布局允许原生前缀范围扫描。PostgreSQL 通过 `ltree` 匹配算子引入额外开销；Neo4j 没有原生前缀搜索原语，
只能用模式匹配查询模拟 Q4，且每次调用都需要计划编译。

### C. 模式设计与演化有效性

接下来，我们隔离冷启动过程（§III-C）和两个模式演化算子（§III-D）的贡献。在 AUTHTRACE 上，将完整 WIKIKV 系统与两个内部变体比较：WIKIKV-FIXEDSCHEMA 用固定维度替换冷启动；
WIKIKV-STATIC 保留冷启动模式但禁用两个演化算子。三种配置共享同一存储层（§IV）和查询层（§V），因此答案正确率或延迟差异都可归因于模式设计与演化。整节中，**答案正确率**（AC）表示
AUTHTRACE pack 级协议下生成答案的端到端正确率；我们同时报告 AC 与在线首 token 延迟，而将**人工评分**留到 §VI-E 的生产研究中。

**冷启动与固定维度。** 用人工固定模式（FIXED）替换冷启动过程，会损失近 10 个绝对百分点的答案正确率（53.5 vs. 63.2），同时页面数量增加 12%（2672 vs. 2385），
因为固定维度会将语料过度划分到主题很薄的子树中。准确率损失会传导到在线延迟——平均首 token 时间从 11.7 s 上升到 12.7 s——因为更宽的 Index 和过度划分的主题迫使导航算子在每次查询中执行更多
`NEEDSDEEPER` 扩展（平均工具调用 3.36 vs. 3.18），并在每次下降中读取更多页面（1.56 vs. 1.48）。因此，§III-C 的数据驱动冷启动在我们报告的每个指标上都优于固定维度。

**表 III：冷启动和演化对答案正确率与在线延迟的影响。**

| 指标 | WIKIKV | FIXED | STATIC |
| --- | ---: | ---: | ---: |
| Page count | 2385 | 2672 | 2290 |
| Avg. tool calls / query | 3.18 | 3.36 | 3.12 |
| Avg. pages read / query | 1.48 | 1.56 | 1.44 |
| Avg. first-token time (s) (↓) | 11.7 | 12.7 | 11.7 |
| AC (↑) | 63.2 | 53.5 | 52.5 |

**演化算子与冻结目录。** 冷启动后冻结模式（STATIC）会保留完整 WIKIKV 的延迟特征——平均首 token 时间与完整系统相同（11.7 s）——但会损失超过 10 个绝对百分点的答案正确率（52.5
vs. 63.2）。页面数量（2290 vs. 2385）进一步揭示了机制：没有 PAGE SPLIT 和 DIMENSION MERGE，模式无法在语料密集化区域生长新主题，也无法合并访问分布视为同一概念的维度。因此，
§III-D 的两个演化算子贡献了系统大部分准确率空间，同时几乎不引入延迟开销，这与定理 1 的单调改进保证一致。

**页面数量与准确率交互。** 联合阅读表 III 中两个消融可知，约束系统表现的是模式质量，而不是模式大小。FIXED 是最大的模式，却最不准确；STATIC 是最小的模式，却仍比完整 WIKIKV 低 10
个百分点。只有冷启动加演化的组合同时实现了适当大小的命名空间和最高准确率。这是式（1）联合优化的经验对应：单独最小化 $|V|$（STATIC）或单独最大化扇出（FIXED）都不充分；二者必须通过 §III-D 的算子共同优化。

### D. 端到端检索：AuthTrace

我们将 §V 中运行在 WIKIKV 之上的继承 LLM-WIKI 检索流水线，与 AUTHTRACE 上四个代表性基线比较。所有基线共享相同生成模型和提示模板；只有检索阶段不同。指标是答案正确率，按 fan-in
桶和总体平均报告（表 IV）。

**表 IV：AUTHTRACE 上按 fan-in 桶划分的端到端答案正确率。**

| 方法 | Single-doc | Low multi-doc | High multi-doc | Overall |
| --- | ---: | ---: | ---: | ---: |
| No-RAG | 9.9 | 16.4 | 16.1 | 12.4 |
| Dense-RAG | 59.5 | 37.1 | 29.4 | 49.9 |
| GraphRAG | 52.2 | 33.8 | 27.6 | 44.3 |
| RAPTOR | 60.1 | 35.6 | 30.8 | 50.0 |
| LLM-Wiki (WIKIKV) | 67.2 | 60.9 | 47.7 | 63.2 |

**Single-doc 桶。** 在 single-doc 查询上，WIKIKV 比最强基线（RAPTOR）高 7.1 个百分点（$p<0.01$）。Single-doc 问题正是扁平向量索引预期表现最佳的场景，
因为单个近邻查找通常已经能定位支撑段落。因此，相对 Dense-RAG 和 RAPTOR 的差距可归因于导航到路径有界实体页面，而非自由漂浮 chunk 的结构优势。

**Low-multi-doc 桶。** 当 fan-in 为 2 时差距显著扩大：WIKIKV 达到 60.9，而最强基线 Dense-RAG 为 37.1。双文档证据需要在兄弟实体页面之间进行显式遍历，
或进行多层遍历。WIKIKV 的路径索引导航直接匹配这种访问模式，而扁平稠密检索和图社区检索都缺乏这种能力。

**High-multi-doc 桶。** 在最高 fan-in 桶上，WIKIKV 得分 47.7，而最佳基线为 30.8，差距 16.9 个点（配对 bootstrap，$p<0.01$），确认了 §V-B 的预测：
随着查询所需证据文档数量增加，结构导航的价值也相应增长，因为每个额外支撑文档通常都可通过层次化 wiki 节点到达。跨桶趋势显示，三个检索基线从 single-doc 到 high-multi-doc 都损失超过 20
个点，而 WIKIKV 损失不足 20 个点；这说明结构化检索在 fan-in 压力下比任何扁平索引或摘要树基线退化得更平滑。

### E. 生产部署研究

我们进一步报告 WIKIKV 作为微信公众账号 AI 助手个人知识库服务的在线部署。该部署支撑一个跨数百个作者账号、规模达数百万页面和数千万 KV 对的生产知识库，其查询流量比 §VI-D 的受控工作负载高数个数量级。
[^scale] 我们从面向作者特定知识库的真实用户查询中采样 1,000 条，测量完整在线路径（router → navigation → generation），并报告系统侧延迟和人工评估质量。

[^scale]: 由于商业机密，绝对语料和流量数字被隐去；报告的数量级足以刻画部署规模。

**质量评分协议。** 1,000 条查询从实时查询日志中均匀采样，并在评审前去标识化。每个答案按生产评审规则使用 3 分量表评分：3（exact hit）——答案直接回答用户查询，或完全支持回答用户查询，
无需额外推理或补充信息；2（related but indirect）——答案没有直接回答查询，但包含与查询主题相关、可经进一步推理使用的内容；1（irrelevant or empty）——答案与查询没有实质联系，
或为空。每项由三名对系统配置盲评的标注者评分，标注者间一致性较高（Krippendorff's $\alpha=0.71$）；报告值为逐项均值。

**表 V：微信公众账号 AI 助手部署中 1,000 条真实查询的在线延迟和人工评估质量。**

| 指标 | Avg. | P50 | P95 | P99 |
| --- | ---: | ---: | ---: | ---: |
| Wiki tool calls / query | 2.2 | 2 | 3 | 4.6 |
| Wiki tool latency (s) | 0.432 | 0.411 | 0.554 | 0.966 |
| First-token latency (s) | 6.856 | 6.65 | 8.34 | 12.32 |
| Human rating (1–3) | 2.86 |  |  |  |

**在线延迟（表 V）。** 部署系统的平均首 token 延迟为 6.856 s。wiki 工具组件本身平均只贡献 0.432 s，即便在 P99 也保持在 1 s 以下，确认 §IV 的路径即键存储路径不是生产瓶颈：
端到端时间由 LLM 生成主导，而不是检索主导。平均每查询工具调用数为 2.2（中位数 2，P99 为 4.6），进一步表明 §V-B 的搜索路由算子用一到两步导航即可解析大多数真实查询。

**答案质量。** 人工平均评分为 2.86，介于 related（2）和 exact hit（3）之间，且更接近后者；这说明在真实作者知识库查询上，WIKIKV 多数情况下返回直接可用答案，而不仅是相关材料。
结合上述延迟特征，这为受控 AUTHTRACE 评估提供了现实世界证据。

### F. 可扩展性

我们从 AUTHTRACE 中采样三个语料规模递增的嵌套区间：500-query、1000-query 和 full。对每个区间，我们记录归纳实例的结构足迹（目录和页面）以及 Avg./P50/P95/P99 的首
token 延迟。图 5(a) 绘制结构增长——目录数量基本不变，而页面数几乎与语料规模成比例增长；图 5(b) 绘制延迟特征——主体仅轻微漂移而尾部压缩。这两个视角共同分离了**模式深度**和**页面质量**对延迟的影响。

**图 5：WIKIKV 在 AUTHTRACE 上的端到端可扩展性。** (a) WIKIKV 结构增长；(b) 端到端首 token 延迟。

**端到端延迟的亚线性扩展。** 图 5(b) 的延迟曲线直观展示了亚线性扩展：将问题集翻倍使页面数量增加约 $1.4\times$，但平均端到端延迟只增加 12.2%，P99 延迟只增加 10.2%；
继续扩大到完整语料后，额外增长不到 1%。首 token Avg. 曲线仅轻微漂移（三个区间总计 +9.2%），而 P50 基本固定；高分位指标以相同比例收紧，Full 上 P95 甚至收缩，P99 被限制在 0.7
s 带宽内。因此，规模通过延迟分布主体进入系统，而不是通过尾部爆炸进入。

**为何增长近乎平坦。** 这种形态直接来自 §IV–§V 的模型。每个查询的 Q1 点查找因 §IV 的路径即键共址而在 KV 对数量上为 $O(1)$，在所有规模下都是常数；后续路径导航由模式深度 $D$ 约束，
而不是由 $|V|$ 约束，§III-C 的冷启动使 $D$ 随语料增长保持稳定。图 5(a) 中平坦的“Directories”曲线证实了这一点——目录数量保持在 34–35，而文档和页面分别增长约
$1.75\times$ 和 $1.73\times$。剩余约 10% 的增长来自叶节点处的逐页读取，以及略大的 `NEEDSDEEPER` 候选集，而不是导航深度。

### G. 消融

为隔离 WIKIKV 中两个最具特征的设计选择，即 §III-C 的采样冷启动和 §V-B 的搜索路由算子，我们在 AUTHTRACE 中主题最密集的单作者 LUXUN 语料上运行端到端消融。我们在相同查询工作负载、
生成模型和提示模板下比较三种配置：Full（完整系统）；w/o Cold-Start，将完整文档集注入模式构建而非使用采样子集；以及 w/o Search Routing，禁用阶段 1 路由器并回退到纯逐层导航。

**表 VI：LUXUN 语料（AUTHTRACE 中主题最密集子集）上的消融：移除冷启动采样或搜索路由。**

| 指标 | Full | w/o Cold-Start | w/o Search Routing |
| --- | ---: | ---: | ---: |
| Avg. tool calls / query | 3.53 | 3.75 | 5.42 |
| Avg. pages read / query | 1.56 | 1.63 | 5.40 |
| Avg. first-token latency (s) | 11.9 | 14.8 | 15.1 |
| AC (↑) | 65.6 | 54.7 | 40.2 |

**冷启动采样与完整文档注入。** 用完整文档注入替代采样冷启动（W/O COLD-START）会损失 10.9 个绝对百分点的 AC（54.7 vs. 65.6），并使每个延迟指标变差：平均首 token 延迟增加
24%（11.9 s → 14.8 s）（表 VI）。其机制与 §III-C 一致：把每个文档都喂给模式归纳会膨胀提示，使 LLM 偏向由偶然重叠导致的过细主题，并产生噪声更大、区分度更低的模式。下游工具调用（3.75
vs. 3.53）和页面读取（1.63 vs. 1.56）增加，因为导航器必须在线补偿构建时过拟合的模式。因此，采样冷启动不仅更便宜，而且在准确率和延迟上都产生严格更好的模式。

**搜索路由与纯逐层导航。** 平均首 token 延迟 11.9 s 表明系统可以高效交付正确答案。移除搜索路由会使每查询读取页面数增加三倍以上（5.40 vs. 1.56），工具调用增加 53%（5.42 vs.
3.53），反映出路由器无法剪除无关分支时需要更深导航。延迟适度增加到 15.1 s，这是额外探索的副作用，而不是主要成本。真正的惩罚是准确率：没有搜索路由时 AC 下降 25.4 个点（65.6% → 40.2%）。
因此，系统把导航预算精确分配到重要位置——以相同数量级延迟获得更少页面、更少工具调用和保留的正确性。这验证了搜索加速设计：瓶颈是路由质量，而不是原始速度。

## VII. 相关工作

我们回顾层次化数据管理、用于知识的图数据库和键值数据库，以及检索增强生成方面的既有工作。

### A. 层次化数据管理

LDAP [24] 等目录服务把条目组织在全局唯一 DN 层次结构下。GFS [15] 和 HDFS [16] 等分布式文件系统提供父子布局，但只在叶节点暴露不透明字节流，并把目录元数据作为面向批量的簿记处理，
而不是作为负载有界、可查询的列表原语。Hive [25] 等数仓式查询层位于这些文件系统之上，但其关系型、集合导向的评估模型同样不匹配 LLM 驱动导航中的“页面加枚举子节点”契约。

更接近的工作是 XML 的原生与关系型处理。TIMBER [26] 开创了通过结构 join 对树查询进行集合式评估；Tatarinov [27] 将有序 XML 编码到关系中，同时不丧失 XPath 表达能力。然而，
以上所有系统暴露的导航接口都会返回整棵子树或无界子集合，没有机制在 LLM 上下文窗口内限制单个遍历步骤的负载大小。

相比之下，WIKIKV 提供了为 LLM 驱动遍历定制的预算感知层次化导航（§V）。

### B. 面向知识的图数据库与键值存储

Neo4j 与 Cypher [11], [28], [29] 等属性图数据库擅长在异构边上进行多跳推理，并支撑 Freebase [8]、DBpedia [7] 和 Wikidata [6] 等大型知识图谱。然而，
它们都不暴露原生“目录列表”原语：枚举子节点需要 Cypher 模式或带标签遍历，其规划器开销相对于 $O(k)$ 前缀扫描而言过高。

在另一端，Dynamo [12]、Bigtable [14]、Cassandra [13] 和 Spanner [30] 等工业 KV 与宽列存储——通常建立在 LSM-tree 后端 [31] 之上——提供
$O(1)$ 查找和可扩展性，但刻意保持**扁平**：没有层次化键空间，也没有把父节点值与子节点存在性绑定的契约。基于它们构建 wiki 会迫使应用重新发明路径语义，而 WIKIKV
通过把层次结构提升进键编码来弥合这一缺口。因此，WIKIKV 并不是图或 KV 存储的替代品，而是可置于任何支持点查找和前缀扫描后端之上的薄**路径索引层**（§IV）。

### C. 检索增强生成与知识检索

用外部语料为 LLM [1]–[3] 提供 grounding 的主导范式是 RAG [4], [5]：在 HNSW [35]、FAISS [17] 或 Milvus [18] 等 ANN 索引上使用 DPR [32]、
ColBERT [33] 或 REALM [34] 等稠密检索器，并用 Fusion-in-Decoder [36] 等 reader 聚合证据。与这些方法不同，它们把语料扁平化为 chunk 袋，并迫使 LLM
重建被丢弃的结构；WIKIKV 通过行走层次结构来回答，而不是拼接 top-$k$ chunk。

近期工作在检索器内部恢复结构：RAPTOR [23] 将 chunk 聚类并摘要为平衡树；MemWalker [37] 将长上下文阅读视为摘要树上的导航；GraphRAG [22] 提取带社区摘要的实体关系图；
ReAct [9] 等智能体框架交替进行检索与推理。这些方法运行在静态内存层次结构上，并未处理模式演化；WIKIKV 增加了存储层演化算子（§III），使层次结构能在并发读写下安全演化。

## VIII. 结论

本文提出 WIKIKV，一种专为 LLM 策划层次化知识库构建的路径索引键值存储模型，并已在微信公众账号平台上规模化部署。通过把 wiki 模式直接物化到键命名空间中，每个目录列表都被归约为 $O(1)$
个存储往返中的单次点查找，而不是深度相关遍历。在此之上，数据驱动模式层通过意图锚定模式归纳进行冷启动，并通过互信息合并与 Architect–Critic–Arbiter 拆分持续演化；
“先子后父”写协议在并发离线重写下无需锁定读路径即可排除部分读；预算化导航算子则将 LLM 辅助下降从 $O(\text{depth})$ 压缩到 $O(1)$。AUTHTRACE 上的实验和微信公众账号 AI
助手中的在线部署证实，相比关系型、图和文件系统后端，WIKIKV 在单算子延迟上持续保持低延迟，并相对 RAG 基线取得更高端到端答案正确率。未来工作将面向多模态知识库、自适应深度调优、并发离线写者下更强一致性，
以及超越 AUTHTRACE 单一文学领域的跨域泛化。

## AI 生成内容声明

我们披露，在第六节报告的实验评估中，人工智能（AI）作为可替换推理组件被使用。具体而言，DEEPSEEK-V4-FLASH 被用于三个位置：（i）使用意图锚定冷启动模式归纳离线构建 WIKIKV 实例；（ii）
在线模式演化流水线；以及（iii）WIKIKV 和所有 RAG 基线的端到端检索增强问答流水线。

## REFERENCES

[1] OpenAI, J. Achiam, S. Adler, S. Agarwal, L. Ahmad, I. Akkaya, F. L.
Aleman, D. Almeida, J. Altenschmidt, S. Altman et al., “Gpt-4 technical
report,” 2024. [Online]. Available: https://arxiv.org/abs/2303.08774

[2] A. Yang, A. Li, B. Yang, B. Zhang, B. Hui, B. Zheng, B. Yu, C. Gao,
C. Huang, C. Lv et al., “Qwen3 technical report,” 2025. [Online].
Available: https://arxiv.org/abs/2505.09388

[3] A. Grattafiori, A. Dubey, A. Jauhri, A. Pandey, A. Kadian, A. Al-Dahle,
A. Letman, A. Mathur, A. Schelten, A. Vaughan et al., “The llama 3 herd
of models,” 2024. [Online]. Available: https://arxiv.org/abs/2407.21783

[4] P. Lewis, E. Perez, A. Piktus, F. Petroni, V. Karpukhin, N. Goyal,
H. Küttler, M. Lewis, W.-t. Yih, T. Rocktäschel et al., “Retrieval-
augmented generation for knowledge-intensive nlp tasks,” in Advances
in Neural Information Processing Systems, H. Larochelle, M. Ranzato,
R. Hadsell, M. Balcan, and H. Lin, Eds., vol. 33. Curran
Associates, Inc., 2020, pp. 9459–9474. [Online]. Available: https:
//proceedings.neurips.cc/paper_files/paper/2020/file/6b493230205f780e
1bc26945df7481e5-Paper.pdf

[5] Y. Gao, Y. Xiong, X. Gao, K. Jia, J. Pan, Y. Bi, Y. Dai,
J. Sun, M. Wang, and H. Wang, “Retrieval-augmented generation
for large language models: A survey,” 2024. [Online]. Available:
https://arxiv.org/abs/2312.10997

[6] D. Vrandečić and M. Krötzsch, “Wikidata: a free collaborative
knowledgebase,” Commun. ACM, vol. 57, no. 10, p. 78–85, Sep. 2014.
[Online]. Available: https://doi.org/10.1145/2629489

[7] S. Auer, C. Bizer, G. Kobilarov, J. Lehmann, R. Cyganiak, and
Z. Ives, “Dbpedia: A nucleus for a web of open data,” in The
Semantic Web: 6th International Semantic Web Conference, 2nd
Asian Semantic Web Conference, ISWC 2007 + ASWC 2007, Busan,
Korea, November 11-15, 2007. Proceedings. Berlin, Heidelberg:
Springer-Verlag, 2007, p. 722–735. [Online]. Available: https:
//doi.org/10.1007/978-3-540-76298-0_52

[8] K. Bollacker, C. Evans, P. Paritosh, T. Sturge, and J. Taylor, “Freebase:
a collaboratively created graph database for structuring human
knowledge,” in Proceedings of the 2008 ACM SIGMOD International
Conference on Management of Data, ser. SIGMOD ’08. New York,
NY, USA: Association for Computing Machinery, 2008, p. 1247–1250.
[Online]. Available: https://doi.org/10.1145/1376616.1376746

[9] S. Yao, J. Zhao, D. Yu, N. Du, I. Shafran, K. Narasimhan, and Y. Cao,
“React: Synergizing reasoning and acting in language models,” in The
Eleventh International Conference on Learning Representations, 2023.
[Online]. Available: https://arxiv.org/abs/2210.03629

[10] M. Stonebraker and L. A. Rowe, “The design of postgres,” in
Proceedings of the 1986 ACM SIGMOD International Conference on
Management of Data, ser. SIGMOD ’86. New York, NY, USA:
Association for Computing Machinery, 1986, p. 340–355. [Online].
Available: https://doi.org/10.1145/16894.16888

[11] I. Robinson, J. Webber, and E. Eifrem, Graph Databases: New Oppor-
tunities for Connected Data, 2nd ed. O’Reilly Media, Inc., 2015.

[12] G. DeCandia, D. Hastorun, M. Jampani, G. Kakulapati, A. Lakshman,
A. Pilchin, S. Sivasubramanian, P. Vosshall, and W. Vogels,
“Dynamo: amazon’s highly available key-value store,” in Proceedings
of Twenty-First ACM SIGOPS Symposium on Operating Systems
Principles, ser. SOSP ’07. New York, NY, USA: Association
for Computing Machinery, 2007, p. 205–220. [Online]. Available:
https://doi.org/10.1145/1294261.1294281

[13] A. Lakshman and P. Malik, “Cassandra: a decentralized structured
storage system,” SIGOPS Oper. Syst. Rev., vol. 44, no. 2, p. 35–40, Apr.
2010. [Online]. Available: https://doi.org/10.1145/1773912.1773922

[14] F. Chang, J. Dean, S. Ghemawat, W. C. Hsieh, D. A. Wallach,
M. Burrows, T. Chandra, A. Fikes, and R. E. Gruber, “Bigtable:
A distributed storage system for structured data,” ACM Trans.
Comput. Syst., vol. 26, no. 2, Jun. 2008. [Online]. Available:
https://doi.org/10.1145/1365815.1365816

[15] S. Ghemawat, H. Gobioff, and S.-T. Leung, “The google file system,”
in Proceedings of the Nineteenth ACM Symposium on Operating
Systems Principles, ser. SOSP ’03. New York, NY, USA: Association
for Computing Machinery, 2003, p. 29–43. [Online]. Available:
https://doi.org/10.1145/945445.945450

[16] K. Shvachko, H. Kuang, S. Radia, and R. Chansler, “The hadoop
distributed file system,” in 2010 IEEE 26th Symposium on Mass Storage
Systems and Technologies (MSST), 2010, pp. 1–10.

[17] J. Johnson, M. Douze, and H. Jégou, “Billion-scale similarity search
with gpus,” IEEE Transactions on Big Data, vol. 7, no. 3, pp. 535–547,
2021.

[18] J. Wang, X. Yi, R. Guo, H. Jin, P. Xu, S. Li, X. Wang, X. Guo,
C. Li, X. Xu et al., “Milvus: A purpose-built vector data management
system,” in Proceedings of the 2021 International Conference on
Management of Data, ser. SIGMOD ’21. New York, NY, USA:
Association for Computing Machinery, 2021, p. 2614–2627. [Online].
Available: https://doi.org/10.1145/3448016.3457550

[19] H. Ming, F. Li, X. Wu, and W. Que, “Retrieval as reasoning:
Self-evolving agent-native retrieval via llm-wiki,” 2026. [Online].
Available: https://arxiv.org/abs/2605.25480

[20] X. Wu, F. Li, H. Ming, and W. Que, “Authtrace: Diagnosing
evidence construction in thematically dense single-author corpora,”
2026. [Online]. Available: https://arxiv.org/abs/2605.25382

[21] DeepSeek-AI, “Deepseek-v4: Towards highly efficient million-token
context intelligence,” Model card, 2026. [Online]. Available: https:
//huggingface.co/deepseek-ai/DeepSeek-V4-Flash

[22] D. Edge, H. Trinh, N. Cheng, J. Bradley, A. Chao, A. Mody, S. Truitt,
D. Metropolitansky, R. O. Ness, and J. Larson, “From local to global:
A graph rag approach to query-focused summarization,” 2025. [Online].
Available: https://arxiv.org/abs/2404.16130

[23] P. Sarthi, S. Abdullah, A. Tuli, S. Khanna, A. Goldie, and
C. D. Manning, “RAPTOR: Recursive abstractive processing for
tree-organized retrieval,” in The Twelfth International Conference
on Learning Representations, 2024. [Online]. Available: https:
//openreview.net/forum?id=GN921JHCRw

[24] J. Sermersheim, “Lightweight Directory Access Protocol (LDAP):
The Protocol,” RFC 4511, Jun. 2006. [Online]. Available: https:
//www.rfc-editor.org/info/rfc4511

[25] A. Thusoo, J. S. Sarma, N. Jain, Z. Shao, P. Chakka, S. Anthony,
H. Liu, P. Wyckoff, and R. Murthy, “Hive: a warehousing
solution over a map-reduce framework,” vol. 2, no. 2. VLDB
Endowment, Aug. 2009, p. 1626–1629. [Online]. Available: https:
//doi.org/10.14778/1687553.1687609

[26] H. V. Jagadish, S. Al-Khalifa, A. Chapman, L. V. S. Lakshmanan,
A. Nierman, S. Paparizos, J. M. Patel, D. Srivastava, N. Wiwatwattana,
Y. Wu et al., “Timber: A native xml database,” The VLDB
Journal, vol. 11, no. 4, p. 274–291, Dec. 2002. [Online]. Available:
https://doi.org/10.1007/s00778-002-0081-x

[27] I. Tatarinov, S. D. Viglas, K. Beyer, J. Shanmugasundaram, E. Shekita,
and C. Zhang, “Storing and querying ordered xml using a relational
database system,” in Proceedings of the 2002 ACM SIGMOD
International Conference on Management of Data, ser. SIGMOD ’02.
New York, NY, USA: Association for Computing Machinery, 2002, p.
204–215. [Online]. Available: https://doi.org/10.1145/564691.564715

[28] R. Angles, M. Arenas, P. Barceló, A. Hogan, J. Reutter, and D. Vrgoč,
“Foundations of modern query languages for graph databases,” ACM
Comput. Surv., vol. 50, no. 5, Sep. 2017. [Online]. Available:
https://doi.org/10.1145/3104031

[29] N. Francis, A. Green, P. Guagliardo, L. Libkin, T. Lindaaker,
V. Marsault, S. Plantikow, M. Rydberg, P. Selmer, and A. Taylor,
“Cypher: An evolving query language for property graphs,” in
Proceedings of the 2018 International Conference on Management
of Data, ser. SIGMOD ’18. New York, NY, USA: Association
for Computing Machinery, 2018, p. 1433–1445. [Online]. Available:
https://doi.org/10.1145/3183713.3190657

[30] J. C. Corbett, J. Dean, M. Epstein, A. Fikes, C. Frost, J. J.
Furman, S. Ghemawat, A. Gubarev, C. Heiser, P. Hochschild
et al., “Spanner: Google’s globally distributed database,” ACM Trans.
Comput. Syst., vol. 31, no. 3, Aug. 2013. [Online]. Available:
https://doi.org/10.1145/2491245

[31] P. O’Neil, E. Cheng, D. Gawlick, and E. O’Neil, “The log-structured
merge-tree (lsm-tree),” Acta Inf., vol. 33, no. 4, p. 351–385, Jun. 1996.
[Online]. Available: https://doi.org/10.1007/s002360050048

[32] V. Karpukhin, B. Oguz, S. Min, P. Lewis, L. Wu, S. Edunov,
D. Chen, and W.-t. Yih, “Dense passage retrieval for open-domain
question answering,” in Proceedings of the 2020 Conference on
Empirical Methods in Natural Language Processing (EMNLP),
B. Webber, T. Cohn, Y. He, and Y. Liu, Eds. Online: Association
for Computational Linguistics, Nov. 2020, pp. 6769–6781. [Online].
Available: https://aclanthology.org/2020.emnlp-main.550/

[33] O. Khattab and M. Zaharia, “Colbert: Efficient and effective passage
search via contextualized late interaction over bert,” in Proceedings
of the 43rd International ACM SIGIR Conference on Research and
Development in Information Retrieval, ser. SIGIR ’20. New York, NY,
USA: Association for Computing Machinery, 2020, p. 39–48. [Online].
Available: https://doi.org/10.1145/3397271.3401075

[34] K. Guu, K. Lee, Z. Tung, P. Pasupat, and M. Chang, “Retrieval
augmented language model pre-training,” in Proceedings of the 37th
International Conference on Machine Learning, ser. Proceedings of
Machine Learning Research, H. D. III and A. Singh, Eds., vol.
119. PMLR, 13–18 Jul 2020, pp. 3929–3938. [Online]. Available:
https://proceedings.mlr.press/v119/guu20a.html

[35] Y. A. Malkov and D. A. Yashunin, “Efficient and robust approxi-
mate nearest neighbor search using hierarchical navigable small world
graphs,” IEEE Transactions on Pattern Analysis and Machine Intelli-
gence, vol. 42, no. 4, pp. 824–836, 2020.

[36] G. Izacard and E. Grave, “Leveraging passage retrieval with generative
models for open domain question answering,” in Proceedings of the
16th Conference of the European Chapter of the Association for
Computational Linguistics: Main Volume, P. Merlo, J. Tiedemann,
and R. Tsarfaty, Eds. Online: Association for Computational
Linguistics, Apr. 2021, pp. 874–880. [Online]. Available: https:
//aclanthology.org/2021.eacl-main.74/

[37] H. Chen, R. Pasunuru, J. Weston, and A. Celikyilmaz, “Walking down
the memory maze: Beyond context limit through interactive reading,”
2023. [Online]. Available: https://arxiv.org/abs/2310.05029
