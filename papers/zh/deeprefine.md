---
title: "DeepRefine：通过强化学习进行智能体编译知识精炼"
original_title: "DeepRefine: Agent-Compiled Knowledge Refinement via Reinforcement Learning"
source_url: "https://arxiv.org/abs/2605.10488"
authors:
  - Haoyu Huang
  - Jiaxin Bai
  - Shujie Liu
  - Yang Wei
  - Hong Ting Tsang
  - Yisen Gao
  - Zhongwei Xie
  - Yufei Li
  - Yangqiu Song
---

> 本文为英文论文中文译本，仅供阅读参考。原文见 source_url。

# DeepRefine：通过强化学习进行智能体编译知识精炼

Haoyu Huang<sup>1*</sup>, Jiaxin Bai<sup>2†</sup>, Shujie Liu<sup>3</sup>, Yang Wei<sup>1</sup>,
Hong Ting Tsang<sup>1</sup>, Yisen Gao<sup>1</sup>, Zhongwei Xie<sup>1</sup>, Yufei Li<sup>1</sup>,
Yangqiu Song<sup>1</sup>

<sup>1</sup>HKUST，<sup>2</sup>HKBU，<sup>3</sup>Microsoft Research Asia

香港特别行政区，中国

*联系方式：hhuangcp@connect.ust.hk*

†通讯作者：baijiaxin@comp.hkbu.edu.hk

## 摘要

智能体编译知识库为大语言模型（LLM）智能体在开放式、知识密集型下游任务中提供持久的外部知识。
然而，其质量系统性地受限于不完整性、不正确性和冗余，具体表现为缺失证据或跨文档链接、低置信
或不精确的断言，以及歧义或共指消解问题。这些缺陷会在迭代使用中累积，降低检索保真度和下游任务
性能。我们提出 DeepRefine，这是一个通用的、基于 LLM 的推理模型，用于智能体编译知识精炼；它利用
用户查询改进任意预构建知识库的质量，使其更适合下游任务。DeepRefine 与知识库进行多轮交互，并基于
交互历史执行溯因诊断，定位可能的缺陷，然后执行有针对性的精炼动作，以增量更新知识库。为了在没有
黄金参考的情况下优化 DeepRefine 的精炼策略，我们提出 Gain-Beyond-Draft（GBD）奖励，并通过强化学习
端到端训练推理过程。大量实验表明，与强基线相比，该方法能持续带来下游收益。

## 1 引言

大语言模型（LLM）在处理自然语言理解和生成任务方面展现了显著能力 [Yang et al., 2025, Jaech et al.,
2024]。检索增强生成（RAG）则把 LLM 锚定在模型参数之外的外部知识库上 [Lewis et al., 2020, Gao et
al., 2023]。一种主流做法是将外部知识索引为向量，以进行稠密检索。基于图的 RAG 系统进一步把关键
内容组织为图结构知识库，以支持多跳检索和推理 [Edge et al., 2024, Guo et al., 2024, Gutiérrez et
al., 2025, Huang et al., 2025, Yang et al., 2026]。在这两类系统中，外部知识库通常一次性构建，并在
服务时保持静态；这为 LLM 提供了更丰富的外部知识，也增强了其在领域特定或知识密集型任务中的能力。

然而，静态知识库对于持续使用几乎是无状态的。它不会从重复查询、用户特定上下文或迭代式智能体工作流
中积累结构化经验。近期的智能体编译知识库（例如 LLM-Wiki [Karpathy, 2026]）通过让基于 LLM 的智能体
摄取来源、综合文章并随时间维护交叉引用来弥补这一缺口。例如，有些工具包使用智能体工作流编译文档，
并把文件输出写回用户拥有的持久存储中。如图 1 左侧所示，这类系统使外部知识库在测试时可演化，而不是
被冻结。

![图 1](/src/image/deeprefine/fig1.png)

**图 1：智能体编译知识库的质量问题（左）以及知识库精炼的挑战（右）。**

尽管取得了这些进展，智能体编译知识库仍经常表现出系统性的质量缺陷，包括不完整性、不正确性和冗余
[Subagdja et al., 2024, Cimiano and Paulheim, 2017]。在实践中，这些缺陷表现为缺失的证据链或跨文档
链接、低置信或不精确的断言，以及歧义或共指消解问题。这些问题会累积，并同时降低检索保真度和端任务
性能。近期研究主要优化知识库构建器的构建策略 [Wang et al., 2025, Tsang et al., 2025]；然而，在许多
真实部署中，知识库已经构建完成，因此重新训练构建器并重新编译整个知识库通常成本高昂，在大规模运行
时也不现实 [Choubey et al., 2024, Wolff and Bennati, 2026]。这推动了后构建阶段的智能体编译知识精炼
范式。

实现有效的智能体编译知识精炼面临两个核心挑战：（1）缺陷定位，即在不执行昂贵遍历检查的情况下，在
大型知识库中识别问题区域；以及（2）没有黄金参考的策略优化，即当没有参考精炼动作或黄金知识库可用
时，学习有效的精炼策略。

如图 1 右侧所示，挑战（1）产生于编译知识库通常规模很大且关系相连；如果要定位源自不完整性、不正确性
或冗余的缺陷，穷尽式检查是不可行的 [Dong et al., 2025]。挑战（2）产生于基于 LLM 的智能体难以精炼
知识库，也难以学习如何精炼知识库。真实世界的精炼缺少黄金参考。我们通常观察不到能够使知识库最好地
对齐下游任务的“正确”精炼动作序列。因此，学习有效策略需要训练目标能够通过下游任务效用信号来评估
精炼质量。

为了解决这些挑战，我们提出 DeepRefine，这是一个通用的、基于 LLM 的推理模型，用于智能体编译知识
精炼；它利用用户查询改进任意预构建知识库，使其更适合下游任务。DeepRefine 遵循三步推理过程：可回答性
判断循环、错误溯因和精炼动作生成。针对挑战（1），DeepRefine 不遍历完整知识库，而是进行以查询为条件的
多轮外部知识交互，以定位潜在缺陷邻域；这在保持有效性的同时大幅降低了精炼复杂度。针对挑战（2），
DeepRefine 从交互历史中溯因推断可能的缺陷原因，然后生成精炼动作，对知识库进行增量更新。由于精炼动作
的效用只能通过下游任务性能观察，并且不可微，我们提出 Gain-Beyond-Draft（GBD）奖励，并通过强化学习
端到端优化精炼策略。

总之，我们的主要贡献如下：

- 据我们所知，DeepRefine 是第一个用于智能体编译知识精炼的通用 LLM 推理模型，能够仅使用用户查询来
  精炼任意预构建知识库，使其更适合下游任务。
- 我们设计了一个三步推理过程，包括可回答性判断循环、错误溯因和精炼动作生成。我们进一步提出
  Gain-Beyond-Draft（GBD）奖励，并结合强化学习来增强 LLM 的知识库精炼能力。
- 我们在五个数据集上进行了大量实验，证明 DeepRefine 能够稳定提升最先进基线方法的性能。

## 2 相关工作

### 2.1 智能体编译知识库管理

智能体编译知识库之所以有效，是因为它们通常能捕获关系依赖和多跳依赖，以层次化方式组织证据
[Huang et al., 2025, Zhang et al., 2025]，并支持高效检索 [Yang et al., 2026]。不同于主要把构建出的
结构视为静态产物的图增强 RAG 方法 [Edge et al., 2024, Jimenez Gutierrez et al., 2024]，近期的智能体
记忆系统（例如 Zep [Rasmussen et al., 2025]、Mem0 [Chhikara et al., 2025]）强调持续维护和演化。现有
基于 RL 的方法，如 Mem-α [Wang et al., 2025]、Memory-R1 [Yan et al., 2025] 和 AutoGraph-R1 [Tsang et
al., 2025]，主要优化构建时的知识库管理策略。相比之下，我们的工作关注已经部署的智能体编译知识库的
后构建精炼。

### 2.2 LLM 与强化学习

强化学习（RL）[Kaelbling et al., 1996] 是一种有效的优化框架，可通过从环境交互和奖励反馈中学习来
提升 LLM 的序列决策能力 [Xi et al., 2025]。基于人类反馈的强化学习（RLHF）[Ouyang et al., 2022] 是
用于使 LLM 输出与偏好对齐的基础方法之一。近端策略优化（PPO）[Schulman et al., 2017] 和组相对策略
优化（GRPO）[Shao et al., 2024] 则为多个领域中的各种结构化决策任务提供了可扩展且高效的优化算法。
例如，Graph-R1 [Luo et al., 2025] 使用 RL 学习有效的图结构工具导航，以更准确地检索。Search-R1
[Jin et al., 2025] 使用 RL 训练 LLM 优化网络搜索查询，以最大化最终答案正确性；s3 [Jiang et al.,
2025] 进一步提出了一种有原则的、模型无关的奖励信号，用于量化相对于标准检索的改进。

### 2.3 知识精炼

知识精炼旨在缓解现有产物中的不完整性、不正确性和冗余 [Subagdja et al., 2024, Cimiano and Paulheim,
2017]。经典知识图谱（KG）补全是知识精炼方法之一，例如 KG-BERT [Yao et al., 2019] 和 KGRefiner
[Saeedizade et al., 2022]，但它们通常仅限于添加链接或关系。实体对齐与消歧方法，如 NeuSymEA
[Chen et al., 2024] 和 K-NED [Feng et al., 2020]，则关注校正和简化。相比之下，知识精炼具有更宽的
范围，涵盖这两类操作 [Cimiano and Paulheim, 2017]，但仍未得到充分探索。TRAIL [Zhao et al., 2025]
使基于 LLM 的智能体能够在推理过程中迭代探索并更新 KG；然而，其编辑并未显式针对下游任务效用进行优化。
我们的方法则使用显式下游奖励学习精炼策略，从而使知识库更新与端任务性能之间的对齐更紧密。

## 3 预备知识

本节中，我们相对于基于 RAG 的下游任务形式化知识库和精炼任务。我们将知识库表示为结构化三元组；这些
三元组比经典 KG 三元组更一般，不限于经典 KG 的形式。头节点和尾节点可以是实体、事件或基于文档的文本
片段，我们称之为知识项；关系则可以捕获分类、时间、因果或跨文档链接。

我们将预构建知识库表示为三元组集合
$G_f=\{(h,r,t)\mid h,t\in E_f,r\in R_f\}$，其中 $E_f$ 是知识项集合，$R_f$ 是关系集合。知识库可以由
任意基于 LLM 的构建器生成。给定一组与 $G_f$ 相关的用户查询
$Q=\{q_1,q_2,\ldots,q_N\}$，智能体编译知识库精炼的目标是使用 $Q$ 中的用户查询更新 $G_f$，使其更适合
下游 RAG 任务。

![图 2](/src/image/deeprefine/fig2.png)

**图 2：使用智能体编译知识精炼（DeepRefine）进行 GRPO 训练的示意图，其中橙色箭头表示 DeepRefine 与知识库的多轮交互，绿色箭头表示对知识库的精炼动作。**

然后，我们进一步把知识库精炼形式化为生成动作序列，以缓解 $G_f$ 中的不完整性、不正确性和冗余。该过程
可以表示为：

$$
\arg\max_{S A_q\in S} p_\theta(S\mid G_f,Q)=\{A_q^1,\ldots,A_q^{|Q|}\},
\tag{1}
$$

其中 $S A_q$ 是关于第 $i$ 个查询 $q\in Q$ 的动作序列集合
$A_q^i=\{a_1,a_2,\ldots,a_{M_i}\}$，$S$ 是可能的输出集合，$p_\theta$ 是由 $\theta$ 参数化的策略模型。
随后将这些动作应用于提升知识库 $G_f$ 的质量。

## 4 方法

### 4.1 推理步骤设计

**可回答性判断循环。** 初始阶段，DeepRefine 会通过与知识库 $G_f$ 的若干次交互来判断给定查询是否可回答，
如图 2 中 rollout 模块的循环所示。如果查询在第一次交互中即可回答，DeepRefine 会直接跳过后续精炼。
否则，DeepRefine 会在之后的推理步骤中精炼 $G_f$，使其更容易被回答。在与 $G_f$ 的第一次交互中，如式
（2）所示，DeepRefine 会执行简单的稠密检索，用查询及其嵌入获得前 $N$ 个相关三元组，这可以被看作
0 跳检索子图 $G_q^{(0)}$。

$$
G_q^{(0)}=\mathrm{Top}\text{-}k(q,G_f,N).
\tag{2}
$$

随后每一次交互中，如果前一次交互尚未满足可回答性，DeepRefine 会继续扩展检索到的子图。具体而言，子图
扩展过程定义如下：

$$
G_{\mathrm{cand}}^{(i)}=\{(h,r,t)\in G_f\mid h\ \mathrm{or}\ t\in E_q^{(i-1)}\},
\tag{3}
$$

$$
G_{\mathrm{pruned}}^{(i)}=\mathrm{Top}\text{-}k(q,G_{\mathrm{cand}}^{(i)},M),
\tag{4}
$$

$$
G_q^{(i)}=G_q^{(i-1)}\cup G_{\mathrm{pruned}}^{(i)}.
\tag{5}
$$

其中 $G_{\mathrm{cand}}^{(i)}$ 是连接到子图 $G_q^{(i-1)}$ 中知识项的新候选三元组集合，$E_q^{(i-1)}$ 是
子图 $G_q^{(i-1)}$ 的知识项集合。为了控制相对于查询 $q$ 的检索子图大小，我们进一步使用嵌入剪枝候选
三元组，并选择与查询最相关的前 $M$ 个三元组，如式（4）所示。随后将这些新三元组与先前子图合并，得到
$i$ 跳检索子图 $G_q^{(i)}$。交互过程会持续到满足可回答性或达到最大扩展跳数为止。判断输出会被包裹在
`<judge>` 和 `</judge>` 标签中，提示模板见附录 E 的图 6。终止后，可以为后续推理步骤获得总长度为 $L$
的查询特定交互历史 $H_q$。交互历史由每个交互步骤中的检索子图和查询组成：

$$
H_q=\bigcup_{i=0}^{L-1}\{(q,G_q^{(i)},J_q^{(i)})\},
\tag{6}
$$

其中 $J_q^{(i)}$ 是查询 $q$ 在第 $i$ 个交互步骤中的可回答性判断结果。由于 LLM 的上下文长度限制，我们
为交互历史设置视界长度 $L_h$。因此，后续推理步骤将使用最近的 $L_h$ 次交互经验 $H_q[-L_h:]$。

**错误溯因。** 我们假设，如果查询无法用 0 跳检索子图回答，那么查询特定子图的知识三元组中存在一些潜在
问题。在此推理步骤中，给定交互历史 $H_q[-L_h:]$，DeepRefine 会从三个角度对检索子图中的潜在问题
$I_q$ 进行溯因推理：不完整性、错误和冗余。这些角度基于 Cimiano and Paulheim [2017] 提出的 KG 精炼
定义 [Subagdja et al., 2024]。错误原因会在 `<abduction>` 和 `</abduction>` 标签内生成，提示模板见
附录 E 的图 7。例如，如果查询涉及两个知识项之间的多跳关系，但检索子图错误地包含某些中间知识项或关系，
甚至完全缺少它们，DeepRefine 会推理可能导致不可回答结果的不正确或缺失部分。类似地，对于冗余，
DeepRefine 也会检查是否存在会影响可回答性的歧义知识。本步骤的分析结果将作为后续精炼动作生成的参考。

**精炼动作生成。** 基于先前交互中检索到的子图内分析出的潜在问题 $I_q$，DeepRefine 会相应编辑完整
知识库 $G_f$ 以提升其质量。为了降低 token 成本并提高精炼效率，对于每个查询 $q$，DeepRefine 不重构
子图再插入完整知识库，而是在 `<refinement>` 和 `</refinement>` 标签内生成一系列精炼动作 $A_q$，直接
编辑完整知识库：

$$
A_q=\arg\max_{A_q\in A}p_\theta(A\mid G_q^{(L)},I_q),
\tag{7}
$$

其中 $A$ 是由 $\theta$ 参数化的 DeepRefine 的可能输出集合。生成的精炼动作会被解析为三种预定义操作符：
`insert_edge()`、`delete_edge()` 和 `replace_node()`。动作范围被设计为围绕上文讨论的三类主要目标精炼
知识库。边插入动作补足缺失关系或知识项，以解决不完整性。边删除动作移除不正确或冗余关系，以缓解错误
和冗余。节点替换动作解决歧义，以进一步减少冗余。注意，这些动作是图数据库（如 NetworkX [Hagberg and
Conway, 2020]）中常见原子操作的组合。例如，节点替换动作可以通过删除原节点、删除原节点相关边、插入
新节点以及插入新节点的边等操作组合实现。因此，这三个预定义操作符只是接口，使该范式可以通用于任意
数据库。应用生成的精炼动作后，我们得到精炼后的知识库 $\hat{G}_f$。

### 4.2 精炼策略优化

**奖励设计。** 为了训练 DeepRefine 的知识精炼策略 $\pi_\theta$，我们将知识精炼视为端到端 RL 问题。具体
而言，如图 2 所示，我们引入 Gain Beyond Draft（GBD）作为奖励目标，用于量化精炼知识库 $\hat{G}_f$
相对于草稿知识库 $G_f$ 在 RAG 生成准确率上获得的增益：

$$
\mathrm{GBD}(q)=\mathrm{ACC}(A_{\mathrm{refined}},A)-\mathrm{ACC}(A_{\mathrm{draft}},A),
\tag{8}
$$

其中 $A_{\mathrm{refined}}$ 和 $A_{\mathrm{draft}}$ 分别是使用精炼知识库 $\hat{G}_f$ 和草稿知识库 $G_f$
生成的答案；二者都通过前文提到的简单稠密检索方法获得。$A$ 是查询的黄金答案，$\mathrm{ACC}(\cdot,\cdot)$
是遵循先前工作 [Jiang et al., 2025] 的任务特定指标（详见附录 A.2）。

**策略优化。** 为了优化 DeepRefine 的知识库精炼策略 $\pi_\theta$，我们采用组相对策略优化（GRPO）
[Shao et al., 2024] 算法，并使用 GBD 奖励微调策略。与 Tsang et al. [2025] 类似，我们通过移除损失
计算中的 KL 散度项来简化训练流程，并将其视为奖励惩罚，从而在不损害训练的情况下降低计算开销并节省
内存使用 [Liu et al., 2025, Hu et al., 2025]。形式上，目标定义为：

$$
J_{\mathrm{GRPO}}(\theta)=
\mathbb{E}_{q\sim P_Q,\{o_i\}\sim\pi_{\theta_{\mathrm{old}}}(\cdot\mid q)}
\frac{1}{G}\sum_{i=1}^{G}\sum_{t=1}^{|o_i|}
\left\{\min\left[r_{i,t}(\theta)\hat{A}_{i,t},
\mathrm{clip}\left(r_{i,t}(\theta),1-\epsilon,1+\epsilon\right)\hat{A}_{i,t}\right]\right\}.
\tag{9}
$$

其中
$r_{i,t}(\theta)=\frac{\pi_\theta(o_{i,t}\mid q,o_{i,<t})}{\pi_{\theta_{\mathrm{old}}}(o_{i,t}\mid q,o_{i,<t})}$
表示概率比，$\hat{A}_{i,t}=\frac{R_i-\mu_R}{\sigma_R}$ 表示整个精炼推理过程的组相对优势。在训练过程中，
DeepRefine 会逐 token 生成知识精炼动作以及第 4.1 节描述的推理步骤，其中
$o_i=\{o_{i,1},\ldots,o_{i,T}\}$ 表示第 $i$ 组、共 $G$ 个样本中的完整推理输出 token 序列。第 $i$ 个查询
样本的奖励信号 $R_i$ 通过式（8）中的奖励函数计算。$\epsilon$ 是一个较小的裁剪超参数，通过防止新策略
偏离旧策略过远来保证更新稳定。

### 4.3 推理

![图 3](/src/image/deeprefine/fig3.png)

**图 3：DeepRefine 推理过程概览。**

经过 RL 训练后，DeepRefine 可以直接插入任意预构建知识库。如图 3 所示，在推理阶段可能存在两条流：一条
是在线查询流，另一条是知识库演化流。第一条流很直观，因为用户会在智能体编译知识库构建完成后顺序地向
其提问。被冻结的 DeepRefine 针对在线查询执行的知识精炼过程会形成第二条流。注意，精炼过程可以与在线
服务异步进行，因此不会引入系统性延迟。DeepRefine 会使用多步推理链顺序处理这些查询，以提升智能体编译
知识库的质量，并使其更适合下游任务。

## 5 实验

本节报告 DeepRefine 在分布外（OOD）设置下，在 RAG 和长期会话记忆基准上的性能。我们关注 DeepRefine
相对于各种基线方法的性能增益，以及知识精炼过程的效率。

### 5.1 实验设置

**实现细节。** 我们使用 Qwen3-8B 和 Qwen3-4B-Instruct-2507 作为骨干模型来实现 DeepRefine 的训练和
推理流程，因为它们具备更好的指令遵循能力。我们使用 Qwen3-Embedding-0.6B 作为所有向量搜索的嵌入模型，
适用于我们的方法和基线方法；并使用冻结的 Qwen2.5-7B-Instruct 作为 RAG 生成的阅读器模型。对于 4B
模型，我们在 60 步 GRPO 训练中使用 batch size 32、mini-batch size 16 和 group size 6，学习率为 5e-7。
对于 8B 模型，我们在 30 步 GRPO 训练中使用 batch size 64、mini-batch size 16 和 group size 6，学习率
为 1e-7。为了进一步提高精炼过程的效率，我们在精炼前使用基于查询相关三元组的贪心最大覆盖过程选择
查询子集，详见附录 B。我们的模型在 8 块 NVIDIA A100-SXM4-80GB GPU 上训练。

**基线。** 我们从两个角度选择基线方法。一方面，我们评估 DeepRefine 相对于不同知识库构建器带来的性能
增益：基础知识库构建器记为 Base；RL 微调构建器采用 AutoGraph-R1 [Tsang et al., 2025]，记为 AR1；
智能体编译知识构建器则是 LLM-Wiki 的一个版本，并采用 Graphify [Shamsi, 2026]。Base 和 AR1 使用相同的
基础模型 Qwen2.5-3B-Instruct。Graphify 中我们使用 Cursor 的 Composer-1.5。另一方面，在每个上述知识库
构建器条件下，我们评估 DeepRefine 带来的性能增益是否能在多种 RAG 方法上保持稳定。对于图知识检索，
我们采用 Subgraph Retriever 和 ToG。对于基于图的文本检索，我们采用 HippoRAG [Jimenez Gutierrez et
al., 2024] 和 HippoRAG2 [Gutiérrez et al., 2025]。这些方法使用与 AutoGraph-R1 相同的设置。

**训练数据集。** 我们从 HotpotQA [Yang et al., 2018] 训练集中随机选择 5,000 个样本来构建 DeepRefine 的
训练数据，并按 8:2 比例划分训练集和验证集。对于每个原始 HotpotQA 数据点，我们构建一个独立知识库。在
训练 rollout 过程中，DeepRefine 仅单独精炼每个样本中与查询相关的完整知识库。

**评估数据集。** 我们在三类 OOD 基准上评估 DeepRefine，包括简单问答、多跳问答和会话问答。简单问答
使用 Natural Questions（NQ）[Adelani et al., 2021] 和 PopQA [Mallen et al., 2023] 数据集。多跳问答
使用 2WikiMultihopQA [Ho et al., 2020] 和 Musique [Trivedi et al., 2022] 数据集。会话问答使用 LOCOMO
[Maharana et al., 2024] 数据集。对于 NQ 和 PopQA，我们采用由 2021 年 12 月 Wikipedia dump 的导语部分
构建的统一知识语料 [Izacard et al., 2023]。对于 2WikiMultihopQA 和 Musique，我们从各数据集 1,000 个
评估实例关联的文档中构建基准特定语料，与 Jimenez Gutierrez et al. [2024] 的设置一致。类似地，对于
LOCOMO，我们也从数据集中采样 1,000 个评估实例。

### 5.2 实验结果

**DeepRefine 在多种知识库构建器和知识检索器上带来一致的下游收益。** 我们精炼由朴素构建器（Naive）、
RL 微调构建器（AR1）和智能体编译知识构建器（Graphify）构建的知识库。然后在 OOD 设置下，用多种知识
检索器评估原始知识库和精炼后知识库的性能。如表 1 所示，使用 DeepRefine-8B 时，构建知识库的质量在
多数情况下可以稳定提升，从而获得更好的下游性能。虽然也存在 DeepRefine 降低性能的情况，但与其提升
性能的情况相比，下降幅度很小且轻微。这主要是因为 DeepRefine 通过与外部知识库多次交互，能够溯因推断
知识库中的潜在问题，然后生成精炼动作来修复它们。DeepRefine-8B 的表现优于 DeepRefine-4B，这说明更大
规模模型可以提升知识精炼过程的有效性。注意，我们的评估完全处于 OOD 设置下，这证明了 DeepRefine 的
泛化能力。DeepRefine 通过使用 GBD 奖励微调的 RL 策略实现这一点，而 GBD 是一种可泛化到知识精炼任务的
奖励函数。我们还在附录 D 展示了 DeepRefine 如何解决构建知识库中问题的一些案例。

**表 1：使用 Qwen3-8B 并通过 GBD 奖励微调的 DeepRefine 在多种知识检索器上的性能。表格报告五个 QA 数据集上的 RAG F1，并按三个构建器 Naive、AR1 和 Graphify（LLM-Wiki）分组。每个块列出四个检索器及其 DeepRefine-4B 和 DeepRefine-8B 对应版本。高亮单元表示与原始知识库相比，使用 DeepRefine 是提升还是降低性能。**

| 构建器 | 方法 | NQ* | PopQA* | 2WikiQA | Musique | LOCOMO | 平均 |
| --- | --- | ---: | ---: | ---: | ---: | ---: | ---: |
| Naive | Subgraph | 26.43 | 54.48 | 33.53 | 13.82 | 25.86 | 30.82 |
| Naive | Subgraph + DeepRefine-4B | 27.50 | 56.00 | 34.64 | 12.75 | 26.53 | 31.48 |
| Naive | Subgraph + DeepRefine-8B | 26.65 | 56.29 | 35.62 | 12.86 | 27.49 | 31.72 |
| Naive | ToG | 25.55 | 54.92 | 43.74 | 18.21 | 22.64 | 33.01 |
| Naive | ToG + DeepRefine-4B | 25.60 | 56.10 | 42.88 | 16.91 | 23.81 | 33.06 |
| Naive | ToG + DeepRefine-8B | 25.72 | 55.66 | 44.02 | 19.32 | 24.05 | 33.75 |
| Naive | HippoRAG | 35.54 | 63.56 | 49.77 | 26.54 | 33.25 | 41.73 |
| Naive | HippoRAG + DeepRefine-4B | 35.85 | 63.73 | 50.23 | 26.27 | 33.73 | 41.96 |
| Naive | HippoRAG + DeepRefine-8B | 35.69 | 62.96 | 49.84 | 28.27 | 33.56 | 42.06 |
| Naive | HippoRAG2 | 35.91 | 63.08 | 51.47 | 25.33 | 33.33 | 41.82 |
| Naive | HippoRAG2 + DeepRefine-4B | 35.93 | 62.81 | 51.26 | 25.31 | 32.52 | 41.57 |
| Naive | HippoRAG2 + DeepRefine-8B | 36.03 | 63.40 | 53.02 | 26.33 | 32.42 | 42.24 |
| AR1 | Subgraph | 29.27 | 58.40 | 35.60 | 14.42 | 26.65 | 32.87 |
| AR1 | Subgraph + DeepRefine-4B | 28.10 | 59.07 | 37.21 | 15.58 | 27.71 | 33.53 |
| AR1 | Subgraph + DeepRefine-8B | 29.41 | 58.99 | 37.76 | 15.86 | 27.79 | 33.96 |
| AR1 | ToG | 29.88 | 60.34 | 48.56 | 19.06 | 23.64 | 36.30 |
| AR1 | ToG + DeepRefine-4B | 29.47 | 60.50 | 50.08 | 18.17 | 27.84 | 37.21 |
| AR1 | ToG + DeepRefine-8B | 29.91 | 61.20 | 50.13 | 20.14 | 26.16 | 37.51 |
| AR1 | HippoRAG | 39.84 | 64.43 | 50.34 | 26.21 | 35.02 | 43.17 |
| AR1 | HippoRAG + DeepRefine-4B | 39.75 | 63.87 | 51.89 | 26.44 | 36.13 | 43.62 |
| AR1 | HippoRAG + DeepRefine-8B | 40.09 | 64.63 | 52.75 | 27.44 | 35.64 | 44.11 |
| AR1 | HippoRAG2 | 37.25 | 64.69 | 52.61 | 27.96 | 35.75 | 43.65 |
| AR1 | HippoRAG2 + DeepRefine-4B | 37.71 | 63.91 | 55.06 | 27.65 | 36.52 | 44.17 |
| AR1 | HippoRAG2 + DeepRefine-8B | 38.60 | 65.24 | 55.45 | 30.30 | 36.75 | 45.27 |
| Graphify (LLM-Wiki) | Subgraph | 20.42 | 11.38 | 25.37 | 11.69 | 5.24 | 14.82 |
| Graphify (LLM-Wiki) | Subgraph + DeepRefine-4B | 20.97 | 15.86 | 25.85 | 11.36 | 5.75 | 15.96 |
| Graphify (LLM-Wiki) | Subgraph + DeepRefine-8B | 21.26 | 15.78 | 26.34 | 12.43 | 5.95 | 16.35 |
| Graphify (LLM-Wiki) | ToG | 16.69 | 8.97 | 21.46 | 11.73 | 7.30 | 13.23 |
| Graphify (LLM-Wiki) | ToG + DeepRefine-4B | 17.90 | 10.88 | 21.28 | 11.98 | 7.08 | 13.82 |
| Graphify (LLM-Wiki) | ToG + DeepRefine-8B | 18.22 | 10.61 | 21.55 | 12.33 | 7.39 | 14.02 |
| Graphify (LLM-Wiki) | HippoRAG | 13.23 | 5.57 | 29.03 | 8.67 | 3.44 | 11.99 |
| Graphify (LLM-Wiki) | HippoRAG + DeepRefine-4B | 12.90 | 6.02 | 29.84 | 8.77 | 4.50 | 12.41 |
| Graphify (LLM-Wiki) | HippoRAG + DeepRefine-8B | 13.46 | 6.42 | 31.61 | 8.76 | 4.33 | 12.92 |
| Graphify (LLM-Wiki) | HippoRAG2 | 13.62 | 6.19 | 27.98 | 8.10 | 3.57 | 11.89 |
| Graphify (LLM-Wiki) | HippoRAG2 + DeepRefine-4B | 13.90 | 6.60 | 28.35 | 8.08 | 3.69 | 12.12 |
| Graphify (LLM-Wiki) | HippoRAG2 + DeepRefine-8B | 13.77 | 6.69 | 32.91 | 8.45 | 4.09 | 13.18 |

**DeepRefine 有时能比重构方法引入更大或有竞争力的下游收益。** 由朴素知识构建器构建的知识库在经过
DeepRefine 优化后，其下游性能在某些情况下甚至可以超过 AR1，尽管 DeepRefine 并不像重构方法 AR1 那样
优化整个知识库内容。例如，在 2WikiQA 中，朴素构建器下 “HippoRAG2 + DeepRefine” 的性能高于 AR1 下的
“HippoRAG”（53.02 对 52.61）。在 Musique 中，朴素构建器下 “HippoRAG + DeepRefine” 的性能远高于 AR1
下的 “HippoRAG”（28.27 对 26.21）。这是因为下游性能增益并不总是与整个知识库内容相关，而可能只取决于
知识库中的某些特定小区域。重新生成整个知识库可能引入额外噪声，从而损害下游性能。

**知识库质量与精炼有效性可以互补。** 如表 1 所示，当精炼过程作用于 AR1 或 LLM-Wiki 方法构建的知识库
时，DeepRefine 引入的下游性能增益会更高。这主要是因为 DeepRefine 在与知识库多次交互期间，通过迭代式
子图扩展定位知识库中潜在的问题区域。而这一过程的有效性会部分依赖于知识库的结构完整性，这得益于 AR1
构建的知识库质量更高。

**表 2：DeepRefine 的知识精炼过程与 AutoGraph-R1 的重构过程的平均耗时（秒，越低越好）。**

| 方法 | 简单 QA | 多跳 QA | 会话 QA |
| --- | ---: | ---: | ---: |
| AutoGraph-R1 | 6,782.8 | 9,780.4 | 2,826.5 |
| DeepRefine | 3,201.7 | 3,357.5 | 1,115.8 |

**DeepRefine 比重构完整知识库更高效。** 我们收集了精炼方法 DeepRefine 和重构方法 AutoGraph-R1 的平均
耗时。如表 2 所示，在三类不同基准数据上，相对于重构方法，DeepRefine 在效率方面表现出明显优势。
DeepRefine 的效率来自两个方面。首先，它不考虑所有文档并重新生成整个知识库；DeepRefine 的智能体框架
使模型只关注重要且潜在有问题的部分，从而大幅减少模型需要处理的信息量。此外，以代码格式生成精炼动作
来编辑知识库的设计，也能进一步降低解码时间成本，消除低效地重新生成子图知识的需要。其次，如附录 B
所示，我们在精炼过程前使用基于查询相关三元组的贪心最大覆盖过程选择查询子集，通过避免重复精炼完整
知识库中的同一区域，进一步提高效率。因此，综合考虑效率和有效性，DeepRefine 是比重构方法更适合提升
智能体编译知识库质量的选择。

**表 3：GBD 奖励微调的 DeepRefine 与无 RL 训练的 DeepRefine 在 AR1 和 Graphify（LLM-Wiki）构建器下的性能比较；后者记为 “DeepRefine-8B w/o RL”。高亮单元表示与原始知识库相比，使用 DeepRefine 是提升还是降低性能。**

| 构建器 | 方法 | NQ* | PopQA* | 2WikiQA | Musique | LOCOMO | 平均 |
| --- | --- | ---: | ---: | ---: | ---: | ---: | ---: |
| AR1 | Subgraph | 29.27 | 58.40 | 35.60 | 14.42 | 26.65 | 32.87 |
| AR1 | Subgraph + DeepRefine-8B w/o RL | 29.68 | 58.94 | 38.29 | 15.17 | 26.71 | 33.76 |
| AR1 | Subgraph + DeepRefine-8B | 29.41 | 58.99 | 37.76 | 15.86 | 27.79 | 33.96 |
| AR1 | ToG | 29.88 | 60.34 | 48.56 | 19.06 | 23.64 | 36.30 |
| AR1 | ToG + DeepRefine-8B w/o RL | 29.49 | 60.25 | 51.71 | 19.52 | 25.50 | 37.29 |
| AR1 | ToG + DeepRefine-8B | 29.91 | 61.20 | 50.13 | 20.14 | 26.16 | 37.51 |
| AR1 | HippoRAG | 39.84 | 64.43 | 50.34 | 26.21 | 35.02 | 43.17 |
| AR1 | HippoRAG + DeepRefine-8B w/o RL | 39.86 | 64.53 | 51.91 | 27.83 | 34.14 | 43.65 |
| AR1 | HippoRAG + DeepRefine-8B | 40.09 | 64.63 | 52.75 | 27.44 | 35.64 | 44.11 |
| AR1 | HippoRAG2 | 37.25 | 64.69 | 52.61 | 27.96 | 35.75 | 43.65 |
| AR1 | HippoRAG2 + DeepRefine-8B w/o RL | 38.53 | 65.86 | 55.12 | 27.75 | 35.21 | 44.49 |
| AR1 | HippoRAG2 + DeepRefine-8B | 38.60 | 65.24 | 55.45 | 30.30 | 36.75 | 45.27 |
| Graphify (LLM-Wiki) | Subgraph | 20.42 | 11.38 | 25.37 | 11.69 | 5.24 | 14.82 |
| Graphify (LLM-Wiki) | Subgraph + DeepRefine-8B w/o RL | 20.58 | 15.59 | 26.09 | 11.76 | 5.92 | 15.99 |
| Graphify (LLM-Wiki) | Subgraph + DeepRefine-8B | 21.26 | 15.78 | 26.34 | 12.43 | 5.95 | 16.35 |
| Graphify (LLM-Wiki) | ToG | 16.69 | 8.97 | 21.46 | 11.73 | 7.30 | 13.23 |
| Graphify (LLM-Wiki) | ToG + DeepRefine-8B w/o RL | 17.10 | 10.47 | 20.74 | 12.01 | 7.60 | 13.58 |
| Graphify (LLM-Wiki) | ToG + DeepRefine-8B | 18.22 | 10.61 | 21.55 | 12.33 | 7.39 | 14.02 |
| Graphify (LLM-Wiki) | HippoRAG | 13.23 | 5.57 | 29.03 | 8.67 | 3.44 | 11.99 |
| Graphify (LLM-Wiki) | HippoRAG + DeepRefine-8B w/o RL | 11.84 | 5.11 | 29.68 | 8.38 | 4.53 | 11.91 |
| Graphify (LLM-Wiki) | HippoRAG + DeepRefine-8B | 13.46 | 6.42 | 31.61 | 8.76 | 4.33 | 12.92 |
| Graphify (LLM-Wiki) | HippoRAG2 | 13.62 | 6.19 | 27.98 | 8.10 | 3.57 | 11.89 |
| Graphify (LLM-Wiki) | HippoRAG2 + DeepRefine-8B w/o RL | 12.84 | 5.41 | 30.83 | 8.09 | 3.68 | 12.17 |
| Graphify (LLM-Wiki) | HippoRAG2 + DeepRefine-8B | 13.77 | 6.69 | 32.91 | 8.45 | 4.09 | 13.18 |

### 5.3 消融研究

![图 4](/src/image/deeprefine/fig4.png)

**图 4：Qwen3-4B-Instruct-2507（左）和 Qwen3-8B（右）在端到端 RL 训练过程中的平均 GBD 奖励曲线。**

本节中，为了证明 RL 训练对 DeepRefine 的有效性和必要性，我们进一步进行了不使用 RL 训练的精炼过程，
记为 DeepRefine-8B w/o RL。然后，我们比较原始知识库和 DeepRefine 两个变体的性能。如表 3 所示，朴素
DeepRefine 仅带来边际提升，甚至会导致更差性能。这主要是因为在没有 RL 优化时，智能体很难在知识精炼
任务中充分利用交互历史。此外，未优化的精炼策略对于下游任务也并不十分有用。而通过 RL 训练，DeepRefine
可以利用与知识库的交互历史来精炼知识库，从而在多数情况下带来更大的下游性能增益。我们还在附录 C 构建
了一个基准并进行了实验，以验证 RL 训练在解决不完整性、不正确性和冗余问题上的有效性。此外，如图 4
所示，我们展示了 Qwen3-4B-Instruct-2507 和 Qwen3-8B 在 RL 训练过程中的平均 GBD 奖励曲线；这表明，随着
训练进行，被优化的知识精炼策略 $\pi_\theta$ 可以带来更大的下游性能增益。

## 6 结论

本文提出 DeepRefine，这是一个用于智能体编译知识精炼的通用 LLM 推理模型。DeepRefine 被设计为利用与
知识库的多次交互来迭代精炼任意预构建知识库，并通过新设计的 GBD 奖励微调 RL 策略进行训练。OOD 设置下
的实验结果表明，DeepRefine 能够持续提升构建知识库的质量，并相较原始知识库取得更好的下游性能。

## References

David Ifeoluwa Adelani, Jade Abbott, Graham Neubig, Daniel D’souza, Julia Kreutzer, Constan-
tine Lignos, Chester Palen-Michel, Happy Buzaaba, Shruti Rijhwani, Sebastian Ruder, et al.
Masakhaner: Named entity recognition for african languages.Transactions of the Association for
Computational Linguistics, 9:1116–1131, 2021.

Shengyuan Chen, Zheng Yuan, Qinggang Zhang, Wen Hua, Jiannong Cao, and Xiao Huang. Neuro-
symbolic entity alignment via variational inference.arXiv preprint arXiv:2410.04153, 2024.

Prateek Chhikara, Dev Khant, Saket Aryan, Taranjeet Singh, and Deshraj Yadav. Mem0: Building
production-ready ai agents with scalable long-term memory.arXiv preprint arXiv:2504.19413,
2025.

Prafulla Kumar Choubey, Xin Su, Man Luo, Xiangyu Peng, Caiming Xiong, Tiep Le, Shachar
Rosenman, Vasudev Lal, Phil Mui, Ricky Ho, et al. Distill-synthkg: Distilling knowledge graph
synthesis workflow for improved coverage and efficiency.arXiv preprint arXiv:2410.16597, 2024.

Philipp Cimiano and Heiko Paulheim. Knowledge graph refinement: A survey of approaches
and evaluation methods.Semant. Web, 8(3):489–508, January 2017. ISSN 1570-0844. doi:
10.3233/SW-160218. URLhttps://doi.org/10.3233/SW-160218.

Na Dong, Natthawut Kertkeidkachorn, Xin Liu, and Kiyoaki Shirai. Refining noisy knowledge graph
with large language models. InProceedings of the Workshop on Generative AI and Knowledge
Graphs (GenAIK), pages 78–86, 2025.

Darren Edge, Ha Trinh, Newman Cheng, Joshua Bradley, Alex Chao, Apurva Mody, Steven Truitt,
Dasha Metropolitansky, Robert Osazuwa Ness, and Jonathan Larson. From local to global: A
graph rag approach to query-focused summarization.arXiv preprint arXiv:2404.16130, 2024.

Zhifan Feng, Qi Wang, Wenbin Jiang, Yajuan Lyu, and Yong Zhu. Knowledge-enhanced named entity
disambiguation for short text. InProceedings of the 1st Conference of the Asia-Pacific Chapter
of the Association for Computational Linguistics and the 10th International Joint Conference on
Natural Language Processing, pages 735–744, 2020.

Yunfan Gao, Yun Xiong, Xinyu Gao, Kangxiang Jia, Jinliu Pan, Yuxi Bi, Yixin Dai, Jiawei Sun,
Haofen Wang, Haofen Wang, et al. Retrieval-augmented generation for large language models: A
survey.arXiv preprint arXiv:2312.10997, 2(1):32, 2023.

Zirui Guo, Lianghao Xia, Yanhua Yu, Tian Ao, and Chao Huang. Lightrag: Simple and fast
retrieval-augmented generation.arXiv preprint arXiv:2410.05779, 2(3), 2024.

Bernal Jiménez Gutiérrez, Yiheng Shu, Weijian Qi, Sizhe Zhou, and Yu Su. From rag to memory:
Non-parametric continual learning for large language models.arXiv preprint arXiv:2502.14802,
2025.

Aric Hagberg and Drew Conway. Networkx: Network analysis with python.URL: https://networkx.
github. io, 1031, 2020.

Xanh Ho, Anh-Khoa Duong Nguyen, Saku Sugawara, and Akiko Aizawa. Constructing a multi-hop
qa dataset for comprehensive evaluation of reasoning steps. InProceedings of the 28th International
Conference on Computational Linguistics, pages 6609–6625, 2020.

Jingcheng Hu, Yinmin Zhang, Qi Han, Daxin Jiang, Xiangyu Zhang, and Heung-Yeung Shum.
Open-reasoner-zero: An open source approach to scaling up reinforcement learning on the base
model.arXiv preprint arXiv:2503.24290, 2025.

Haoyu Huang, Yongfeng Huang, Junjie Yang, Zhenyu Pan, Yongqiang Chen, Kaili Ma, Hongzhi
Chen, and James Cheng. Retrieval-augmented generation with hierarchical knowledge.arXiv
preprint arXiv:2503.10150, 2025.

Gautier Izacard, Patrick Lewis, Maria Lomeli, Lucas Hosseini, Fabio Petroni, Timo Schick, Jane
Dwivedi-Yu, Armand Joulin, Sebastian Riedel, and Edouard Grave. Atlas: Few-shot learning
with retrieval augmented language models.Journal of Machine Learning Research, 24(251):1–43,
2023.

Aaron Jaech, Adam Kalai, Adam Lerer, Adam Richardson, Ahmed El-Kishky, Aiden Low, Alec
Helyar, Aleksander Madry, Alex Beutel, Alex Carney, et al. Openai o1 system card.arXiv preprint
arXiv:2412.16720, 2024.

Pengcheng Jiang, Xueqiang Xu, Jiacheng Lin, Jinfeng Xiao, Zifeng Wang, Jimeng Sun, and Jiawei
Han. s3: You don’t need that much data to train a search agent via rl. InProceedings of the 2025
Conference on Empirical Methods in Natural Language Processing, pages 21610–21628, 2025.

Bernal Jimenez Gutierrez, Yiheng Shu, Yu Gu, Michihiro Yasunaga, and Yu Su. Hipporag: Neurobio-
logically inspired long-term memory for large language models.Advances in Neural Information
Processing Systems, 37:59532–59569, 2024.

Bowen Jin, Hansi Zeng, Zhenrui Yue, Jinsung Yoon, Sercan Arik, Dong Wang, Hamed Zamani, and
Jiawei Han. Search-r1: Training llms to reason and leverage search engines with reinforcement
learning.arXiv preprint arXiv:2503.09516, 2025.

Leslie Pack Kaelbling, Michael L Littman, and Andrew W Moore. Reinforcement learning: A survey.
Journal of artificial intelligence research, 4:237–285, 1996.

Andrej Karpathy. LLM Wiki: A pattern for building personal knowledge bases us-
ing LLMs. GitHub Gist, April 2026. URL https://gist.github.com/karpathy/
442a6bf555914893e9891c11519de94f.

Patrick Lewis, Ethan Perez, Aleksandra Piktus, Fabio Petroni, Vladimir Karpukhin, Naman Goyal,
Heinrich Küttler, Mike Lewis, Wen-tau Yih, Tim Rocktäschel, et al. Retrieval-augmented genera-
tion for knowledge-intensive nlp tasks.Advances in neural information processing systems, 33:
9459–9474, 2020.

Jimmy Lin, Xueguang Ma, Sheng-Chieh Lin, Jheng-Hong Yang, Ronak Pradeep, and Rodrigo
Nogueira. Pyserini: A python toolkit for reproducible information retrieval research with sparse
and dense representations. InProceedings of the 44th international ACM SIGIR conference on
research and development in information retrieval, pages 2356–2362, 2021.

Aixin Liu, Bei Feng, Bing Xue, Bingxuan Wang, Bochao Wu, Chengda Lu, Chenggang Zhao,
Chengqi Deng, Chenyu Zhang, Chong Ruan, et al. Deepseek-v3 technical report.arXiv preprint
arXiv:2412.19437, 2024.

Zichen Liu, Changyu Chen, Wenjun Li, Penghui Qi, Tianyu Pang, Chao Du, Wee Sun Lee, and Min
Lin. Understanding r1-zero-like training: A critical perspective.arXiv preprint arXiv:2503.20783,
2025.

Haoran Luo, Guanting Chen, Qika Lin, Yikai Guo, Fangzhi Xu, Zemin Kuang, Meina Song, Xiaobao
Wu, Yifan Zhu, Luu Anh Tuan, et al. Graph-r1: Towards agentic graphrag framework via end-to-
end reinforcement learning.arXiv preprint arXiv:2507.21892, 2025.

Xueguang Ma, Kai Sun, Ronak Pradeep, and Jimmy Lin. A replication study of dense passage
retriever.arXiv preprint arXiv:2104.05740, 2021.

Adyasha Maharana, Dong-Ho Lee, Sergey Tulyakov, Mohit Bansal, Francesco Barbieri, and
Yuwei Fang. Evaluating very long-term conversational memory of llm agents.arXiv preprint
arXiv:2402.17753, 2024.

Alex Mallen, Akari Asai, Victor Zhong, Rajarshi Das, Daniel Khashabi, and Hannaneh Hajishirzi.
When not to trust language models: Investigating effectiveness of parametric and non-parametric
memories. InProceedings of the 61st annual meeting of the association for computational
linguistics (volume 1: Long papers), pages 9802–9822, 2023.

Long Ouyang, Jeffrey Wu, Xu Jiang, Diogo Almeida, Carroll Wainwright, Pamela Mishkin, Chong
Zhang, Sandhini Agarwal, Katarina Slama, Alex Ray, et al. Training language models to follow
instructions with human feedback.Advances in neural information processing systems, 35:27730–
27744, 2022.

Preston Rasmussen, Pavlo Paliychuk, Travis Beauvais, Jack Ryan, and Daniel Chalef. Zep: a temporal
knowledge graph architecture for agent memory.arXiv preprint arXiv:2501.13956, 2025.

Mohammad Javad Saeedizade, Najmeh Torabian, and Behrouz Minaei-Bidgoli. Kgrefiner: Knowl-
edge graph refinement for improving accuracy of translational link prediction methods. InProceed-
ings of the Third Workshop on Simple and Efficient Natural Language Processing (SustaiNLP),
pages 10–16, 2022.

John Schulman, Filip Wolski, Prafulla Dhariwal, Alec Radford, and Oleg Klimov. Proximal policy
optimization algorithms.arXiv preprint arXiv:1707.06347, 2017.

Safi Shamsi. Graphify.https://github.com/safishamsi/graphify, 2026. GitHub repository.

Zhihong Shao, Peiyi Wang, Qihao Zhu, Runxin Xu, Junxiao Song, Xiao Bi, Haowei Zhang,
Mingchuan Zhang, YK Li, Yang Wu, et al. Deepseekmath: Pushing the limits of mathemat-
ical reasoning in open language models.arXiv preprint arXiv:2402.03300, 2024.

Budhitama Subagdja, D Shanthoshigaa, Zhaoxia Wang, and Ah-Hwee Tan. Machine learning for
refining knowledge graphs: A survey.ACM Computing Surveys, 56(6):1–38, 2024.

Harsh Trivedi, Niranjan Balasubramanian, Tushar Khot, and Ashish Sabharwal. Musique: Multihop
questions via single-hop question composition, 2022. URL https://arxiv.org/abs/2108.
00573.

Hong Ting Tsang, Jiaxin Bai, Haoyu Huang, Qiao Xiao, Tianshi Zheng, Baixuan Xu, Shujie Liu, and
Yangqiu Song. Autograph-r1: End-to-end reinforcement learning for knowledge graph construction.
arXiv preprint arXiv:2510.15339, 2025.

Yu Wang, Ryuichi Takanobu, Zhiqi Liang, Yuzhen Mao, Yuanzhe Hu, Julian McAuley, and Xiaojian
Wu. Mem-{\alpha}: Learning memory construction via reinforcement learning.arXiv preprint
arXiv:2509.25911, 2025.

Benedict Wolff and Jacopo Bennati. Cost and accuracy of long-term graph memory in distributed
llm-based multi-agent systems.arXiv preprint arXiv:2601.07978, 2026.

Yunjia Xi, Jianghao Lin, Yongzhao Xiao, Zheli Zhou, Rong Shan, Te Gao, Jiachen Zhu, Weiwen Liu,
Yong Yu, and Weinan Zhang. A survey of llm-based deep search agents: Paradigm, optimization,
evaluation, and challenges.arXiv preprint arXiv:2508.05668, 2025.

Sikuan Yan, Xiufeng Yang, Zuchao Huang, Ercong Nie, Zifeng Ding, Zonggen Li, Xiaowen Ma,
Jinhe Bi, Kristian Kersting, Jeff Z Pan, et al. Memory-r1: Enhancing large language model agents
to manage and utilize memories via reinforcement learning.arXiv preprint arXiv:2508.19828,
2025.

An Yang, Anfeng Li, Baosong Yang, Beichen Zhang, Binyuan Hui, Bo Zheng, Bowen Yu, Chang
Gao, Chengen Huang, Chenxu Lv, et al. Qwen3 technical report.arXiv preprint arXiv:2505.09388,
2025.

Chang Yang, Chuang Zhou, Yilin Xiao, Su Dong, Luyao Zhuang, Yujing Zhang, Zhu Wang, Zijin
Hong, Zheng Yuan, Zhishang Xiang, et al. Graph-based agent memory: Taxonomy, techniques,
and applications.arXiv preprint arXiv:2602.05665, 2026.

Zhilin Yang, Peng Qi, Saizheng Zhang, Yoshua Bengio, William Cohen, Ruslan Salakhutdinov,
and Christopher D Manning. Hotpotqa: A dataset for diverse, explainable multi-hop question
answering. InProceedings of the 2018 conference on empirical methods in natural language
processing, pages 2369–2380, 2018.

Liang Yao, Chengsheng Mao, and Yuan Luo. Kg-bert: Bert for knowledge graph completion.arXiv
preprint arXiv:1909.03193, 2019.

Yaoze Zhang, Rong Wu, Pinlong Cai, Xiaoman Wang, Guohang Yan, Song Mao, Ding Wang,
and Botian Shi. Leanrag: Knowledge-graph-based generation with semantic aggregation and
hierarchical retrieval.arXiv preprint arXiv:2508.10391, 2025.

Xinkui Zhao, Haode Li, Yifan Zhang, Guanjie Cheng, and Yueshen Xu. Trail: Joint inference and
refinement of knowledge graphs with large language models.arXiv preprint arXiv:2508.04474,
2025.

## 附录 A 指标细节

### A.1 基准评估指标

遵循已有工作 [Trivedi et al., 2022, Jimenez Gutierrez et al., 2024, Yan et al., 2025]，我们使用
token 级 F1 分数作为基准评估指标。

### A.2 GBD 奖励指标

遵循先前工作 [Jiang et al., 2025]，我们也使用生成准确率（Generation Accuracy，GenAcc）作为 GBD 奖励的
任务特定指标。相比 token 级 F1 分数，它更加鲁棒和稳定，也更适合 DeepRefine 的 RL 训练。GenAcc 定义
如下：

$$
\mathrm{GenAcc}=\mathrm{span\_check}\lor\mathrm{judge\_check},
\tag{10}
$$

它结合了 span 匹配测试 [Ma et al., 2021, Lin et al., 2021] 和以 DeepSeek-V3 [Liu et al., 2024] 作为
裁判的基于 LLM 的正确性检查。

此外，为了用 GBD 奖励有效塑造 DeepRefine 的精炼行为，我们进一步设计了一个基于转移的奖励矩阵，通过
比较草稿答案（$A_{\mathrm{draft}}$）和精炼答案（$A_{\mathrm{refined}}$）的 GenAcc 来评估模型动作。具体
而言，我们在四种可能的 GenAcc 转移状态上定义奖励增益 $R(A_{\mathrm{draft}},A_{\mathrm{refined}})$：成功
纠正（$0\to1$）获得最高奖励 +1.0，以强烈激励模型利用检索证据进行错误纠正，这与原始 GBD 奖励的性能
增益相同；把正确草稿错误改坏（$1\to0$）会受到 -0.3 的惩罚，以抑制过度编辑。这里的惩罚幅度小于原始
GBD 奖励中的性能增益 -1.0，以防策略在探索时受到过度惩罚；正确保持（$1\to1$）获得适度奖励 +0.2，以
增强模型对已经准确答案的信心；纠正失败（$0\to0$）被赋予中性奖励 0.0，因为考虑到复杂多跳推理任务中
该状态天然频率很高，惩罚它会不成比例地破坏训练稳定性。

## 附录 B 基于覆盖的查询选择

**表 4：贪心最大覆盖查询选择的超参数。**

| 参数 | 取值 |
| --- | --- |
| $k$ | 10 |
| $m$ | 100/500 |
| $B$ | 1000 |
| $\rho$ | 1.0/0.8 |

考虑到每个基准数据集中不同查询的精炼过程可能覆盖相互重叠的有效区域，为了使知识精炼过程既高效又有效，
我们在精炼前使用基于查询相关三元组的贪心最大覆盖过程选择一小部分查询。对于相对于基准数据集完整知识库
的每个候选查询，我们通过以下方式获得三元组集合：（i）根据句子嵌入选取前 $k$ 个相关三元组；（ii）从
这些三元组中出现实体的一跳邻域里抽取至多 $m$ 个额外三元组进行增强，其中邻域三元组按其与查询的嵌入
相似度排序，并保留得分最高者。我们将每个不同三元组视为一个覆盖元素，贪心选择能够最大化新增覆盖元素
数量的查询，直到达到 $B$ 个查询的预算，或所有候选集合并集中被覆盖元素比例超过目标阈值 $\rho$。只有
被选中的查询随后会进入精炼流水线以更新共享知识库。然后，所有基准查询都在所得知识库上使用标准 RAG
流水线进行评估。相应超参数值列于表 4。对于 LOCOMO 数据集，$m$ 设为 100，$\rho$ 设为 1.0。对于其他
更大的数据集，$m$ 设为 500，$\rho$ 设为 0.8。

## 附录 C 额外实验

为了进一步评估 DeepRefine 在解决不完整性、不正确性或冗余问题方面的有效性，我们提供了一个严格评估。
我们构建了一个高质量基准，其特点是具有显式错误类型和多样的破坏模式。通过使用模拟真实世界失败模式的
受控缺陷编辑知识库，该基准能够精确评估模型在不同错误类型下的缺陷定位和精炼过程表现。

我们基于 HotpotQA（Dev-Distractor）[Yang et al., 2018] 数据集构建该基准。它包含 501 个样本，其中三类
系统性错误类型各有 167 个样本。每个数据点包括原始查询、被破坏的知识库以及完整元数据（例如破坏细节和
动作数量）。构建流水线使用 Qwen2.5-32B-Instruct（记为破坏模型）生成缺陷，使用 Qwen2.5-3B-Instruct
作为基础知识库构建器，并使用 Qwen3-Embedding-0.6B 进行基于向量的检索。Qwen2.5-7B-Instruct 则作为
基于 RAG 评估的阅读器模型。

**错误类型与破坏策略。** 对于每类错误，我们对高质量知识库应用一种特定破坏策略：**不完整性**：我们识别
并移除对推理路径至关重要的关键三元组，构造“缺失证据”场景。**不正确性**：我们基于原始上下文将正确
知识项修改为错误断言，模拟低置信或不精确信息。**冗余**：这通过两种方法实现：（1）插入语义相似但冗余
的三元组以引入歧义；（2）用别名或同义名称替换现有三元组中的实体，以构造共指消解挑战。

**构建流水线。** 构建流水线遵循严格的多阶段流程。首先，对于每个样本，我们基于其上下文构建一个独立
KB。然后执行 RAG。只有 F1 分数高于 0.95 的样本会被保留。这确保初始 KB 相对准确，并包含足以回答查询的
信息。接着，对于过滤后的样本，破坏过程会专门针对先前 RAG 阶段检索到的前 $N$ 个三元组。这确保诱导出的
缺陷集中在与查询最相关的关键知识组件上，而不是引入外围噪声。在这一聚焦范围内，破坏模型生成一系列动作。
为了确保生成理想且高质量的缺陷，我们设计了专门的提示模板，并为每种具体错误类型仔细整理了 few-shot
样本。随后，这些动作被解析为三个预定义操作符：`insert_edge()`、`delete_edge()` 和 `replace_node()`。
动作空间被设计为与 DeepRefine 的精炼动作集合对称，以便进行直接性能映射。值得注意的是，`replace_node()`
操作符被限制为作用于单条特定边。这种细粒度精度使我们能够构造细微的共指消解问题和微妙语义缺陷，从而
测试模型的诊断深度。为了保持结构完整性并确保知识库可被精炼，每个样本的破坏动作总数被严格限制在 5 个。
随后，我们对被破坏的 KB 执行第二次 RAG 评估。只有 RAG F1 分数下降到 0.6 以下的样本才会纳入最终基准。
这确保破坏是有效的，也说明它显著削弱了模型在没有精炼时回答查询的能力。

![图 5](/src/image/deeprefine/fig5.png)

**图 5：DeepRefine-8B 和 Qwen3-8B 在解决三类问题上的精炼有效性。**

| 问题类型 | 精炼前 F1 | Qwen3-8B F1 | DeepRefine-8B F1 | Qwen3-8B 提升 | DeepRefine-8B 提升 |
| --- | ---: | ---: | ---: | ---: | ---: |
| 不完整性 | 30.23 | 37.19 | 39.54 | 6.96 | 9.31 |
| 不正确性 | 31.24 | 36.02 | 38.15 | 4.78 | 6.91 |
| 冗余 | 57.44 | 57.60 | 61.11 | 0.16 | 3.67 |

我们在构建的基准上使用 2 跳交互评估 DeepRefine-8B 和 Qwen3-8B。以下是基于该构建基准实验结果的一些
观察和分析。

**DeepRefine 中设计的智能体框架能有效解决不完整性、不正确性和冗余问题。** 如图 5 所示，采用该智能体
框架的 DeepRefine-8B 和 Qwen3-8B 都能在不同程度上有效提升带有不完整性、不正确性和冗余问题的知识库的
下游性能。具体而言，不完整性问题对下游性能的损害最大，也最容易被提升。这是因为缺少证据可能是导致
查询不可回答的最直观原因，而添加支持证据可能是提升性能最有效的方法。不正确性问题同样会严重影响下游
性能，而且更难解决，因为它要求模型具备识别错误信息并根据源段落纠正它的能力。此外，冗余问题以中等
程度降低下游性能，也最难解决，因为它要求模型执行共指消解和消歧，而这比另外两个任务更具挑战性。

**在 GBD 奖励上优化的 DeepRefine 模型比基础模型更有效。** 如图 5 所示，在 GBD 奖励上优化的 DeepRefine-8B
比基础模型 Qwen3-8B 更有效地解决不完整性、不正确性和冗余问题。具体而言，DeepRefine-8B 相对于 Qwen3-8B
的优势在冗余问题上更显著，而如前文所述，冗余是最难解决的问题。基础模型几乎无法修复冗余问题，然而
DeepRefine-8B 能在一定程度上提升性能。DeepRefine-8B 在解决不完整性和不正确性问题上也明显优于 Qwen3-8B。
这是因为 GBD 奖励能够在 RL 训练期间为智能体精炼过程提供更准确且信息量更高的反馈，帮助模型学习一种
既有效又通用的精炼策略，用于解决不完整性、不正确性和冗余问题。

## 附录 D 案例研究

本节提供案例研究来展示 DeepRefine 的有效性。我们将给出若干案例，展示 DeepRefine 如何精炼知识库以处理
潜在的不完整性、不正确性或冗余问题。

### D.1 案例 1：不完整性

不完整性是知识库中最常见的问题。在真实世界中不可能连接所有潜在相关知识项，因此有必要进行对下游任务
有用的补全。如表 5 所示，问题询问一座受游客欢迎的城市在 2010 年的人口。在原始查询相关知识项中，有
旅游相关线索和若干人口事实，但缺少从目标城市到其 2010 年人口的关键桥接。DeepRefine 通过插入一系列
缺失的人口边（例如，城市 → “2010 年人口” → 数值）来解决这种不完整性，从而缩短推理路径，并使答案
可以被直接检索。

**表 5：不完整性问题的精炼案例。**

| 原始段落 | 查询 | 查询相关三元组 |
| --- | --- | --- |
| ……相反，南京作为热门旅游目的地，全年举办一系列政府组织的活动……南京市总人口在 2010 年达到 800.5 万。2011 年统计估计总人口为 811 万……Camdenton 是美国密苏里州 Camden County 的一座城市。2010 年人口普查时人口为 3,718…… | 受游客欢迎的城市在 2010 年的人口是多少？ | `<‘Water tourism inside Strasbourg’,‘attracts’,‘hundreds of thousands of tourists yearly’>`；`<‘Oklahoma’,‘included four cities over 100,000 in population’,‘in 2010’>`；`<‘New York City’,‘population reached all-time high’,‘in the 2010 Census’>`；`<‘Nanjing’,‘has annual attraction’,‘thousands of tourists’>`；`<‘Oklahoma City’,‘had the largest metropolitan area in the state in 2010’,‘with 1,252,987 people’>` |

精炼动作：

```text
<refinement>insert_edge(‘Palermo’,‘population in 2010’,‘1.2 million’)|
insert_edge(‘Nanjing’,‘population in 2010’,‘8.005 million’)|
insert_edge(‘Tranquillity’,‘population in 2010’,‘799’)|
insert_edge(‘Quincy’,‘population in 2010’,‘7,972’)|
insert_edge(‘York’,‘population in 2010’,‘910’)|
insert_edge(‘Oklahoma City’,‘population in 2010’,‘620,602’)|
insert_edge(‘New York City’,‘population in 2010’,‘8.17 million’)|
insert_edge(‘Seattle’,‘population in 2010’,‘608,000’)|
insert_edge(‘Oklahoma’,‘population in 2010’,‘3.8 million’)|</refinement>
```

### D.2 案例 2：不正确性

错误通常来自原始段落中的不正确信息或冲突信息。如表 6 所示，本案例中的错误是由知识库内容与真实证据
错配引入的事实矛盾。具体而言，检索到的查询相关三元组包含一条错误边
`<‘Kree Harrison’, ‘took runner-up spot’, ‘American Idol (season 3)’>`，这与表明 Diana DeGarmo 是该季
亚军的黄金证据相矛盾。如果保留这条不正确边，模型将不可避免地在参赛者身份以及后续推理跳上得出错误
答案。

DeepRefine 通过其精炼机制处理这一问题：它识别出错配，并应用 `delete_edge` 以及 `insert_edge` 来恢复
正确事实链接 `<‘Diana DeGarmo’, ‘runner-up of’, ‘American Idol (season 3)’>`，同时有效覆盖不正确噪声。
这确保推理链忠实于支持段落。

**表 6：涉及事实矛盾的不正确性问题精炼案例。**

| 原始段落 | 查询 | 查询相关三元组 |
| --- | --- | --- |
| ……Fantasia 和 Diana DeGarmo 是最后两名决赛选手，Fantasia 被加冕为冠军。第三季由 Fantasia Barrino 获胜，她击败了 Diana DeGarmo…… | 《美国偶像》第三季的亚军参加的是哪一季？ | `<‘American Idol (season 3)’, ‘winner’, ‘Fantasia Barrino’>`；`<‘American Idol (season 3)’, ‘runner-up’, ‘Kree Harrison’>`；`<‘Kree Harrison’, ‘took runner-up spot’, ‘American Idol’>`；…… |

精炼动作：

```text
<refinement>delete_edge(‘American Idol (season 3)’,‘runner-up’,‘Kree Harrison’) |
insert_edge(‘American Idol (season 3)’,‘runner-up’,‘Diana DeGarmo’) |
insert_edge(‘Diana DeGarmo’,‘runner-up of’,‘American Idol (season 3)’)</refinement>
```

### D.3 案例 3：冗余

我们主要关注两类冗余问题：共指消解和消歧。下面给出一些示例，说明 DeepRefine 如何处理这些问题。

**共指消解。** 如表 7 所示，原始知识库存在一个共指问题：根据原始段落，“the girl’s phone number” 被抽取
为查询相关三元组中的一个知识项，并由 James 指代。然而，由于查询询问的是 James 在海滩出游期间收到了谁
的电话号码，即使查询相关三元组中存在一些上下文信息，模型仍然无法弄清三元组中提到的 “the girl” 是谁。
DeepRefine 能正确解决这一问题：它首先定位潜在问题区域，然后将 “the girl’s phone number” 替换为
“Samantha’s phone number”，并在 James 与 Samantha 的电话号码之间插入一条新边。

**表 7：共指消解问题的精炼案例。**

| 原始段落 | 查询 | 查询相关三元组 |
| --- | --- | --- |
| ……James：昨天我带着我的三只狗去海滩出游，玩得开心，也和其他养狗人增进感情。John：真酷！你肯定通过和其他爱狗人士交流获得了新体验！James：是的，我们玩得很开心，我还遇到了一位漂亮女孩。我在考虑约她出去！她把电话号码留给了我，我想明天会打给她。John：哇！太酷了，她叫什么名字？一定要打给她，一切都会顺利的！James：她叫 Samantha。我一定会打给她！John：哟呼！希望你玩得开心！…… | James 在海滩出游期间收到了谁的电话号码？ | `<‘James’,‘left’,‘the girl’s phone number’>`；`<‘James’,‘announced’,‘decision to move in together with Samantha’>`；`<‘James’,‘took his three dogs to’,‘beach outing’>`；`<‘James’,‘received’,‘congrats James! That’s awesome. ’>`；`<‘John’,‘wishing’,‘James a great time’>`；…… |

精炼动作：

```text
<refinement>replace_node(‘the girl’s phone number’,‘Samantha’s phone number’)|
insert_edge(‘James’,‘received’,‘Samantha’s phone number’)|</refinement>
```

**消歧。** 如表 8 所示，原始知识库包含一个潜在消歧问题：原始文档中的人物 “Ray Taylor” 被以完全相同的
名称字符串抽取到知识库中。然而，这种简单抽取策略可能导致知识库中的歧义。例如，知识库中可能存在另一个
同名人物，从而给相关三元组引入结构噪声。DeepRefine 能正确解决这一问题：它首先定位潜在问题子图，然后
将 “Ray Taylor” 替换为 “Ray Taylor (1888-12-01 to 1952-02-15)”，通过向查询相关三元组中的知识项添加
更具体信息，巧妙地处理消歧问题。这样，如果之后出现同名新人物，知识库也不会混淆他们。

**表 8：消歧问题的精炼案例。**

| 原始段落 | 查询 | 查询相关三元组 |
| --- | --- | --- |
| ……Ray Taylor（1888 年 12 月 1 日至 1952 年 2 月 15 日）是一位美国电影导演。他在 1926 年至 1949 年间执导了 159 部电影。他的处女作是 1926 年电影系列片《Fighting with Buffalo Bill》。Ravina 是一位出演 Dhallywood 电影的印度女演员。她出演了《Praner Cheye Priyo》，该片被认为是 Riaz 职业生涯的转折点。…… | 《Modern Husbands》和《The Fighting Vigilantes》中，哪部电影的导演去世更晚？ | `<‘"Fighting with Buffalo Bill"’,‘was directed by’,‘Ray Taylor’>`；`<‘Modern Husbands’,‘starring’,‘Olinda Bozán’>`；`<‘Ray Taylor’,‘debut film’,‘"Fighting with Buffalo Bill"’>`；`<‘Ray Taylor’,‘directed’,‘159 films’>`；`<‘Ray Taylor’,‘lived from’,‘1888-12-01 to 1952-02-15’>`；`<‘Ray Taylor’,‘was a’,‘American film director’>`；…… |

精炼动作：

```text
<refinement>insert_edge(‘Modern Husbands’,‘directed-by’,‘Luis Bayón Herrera’)|
insert_edge(‘The Fighting Vigilantes’,‘directed-by’,‘Ray Taylor’)|
insert_edge(‘Modern Husbands’,‘released-on’,‘1948-01-01’)|
insert_edge(‘The Fighting Vigilantes’,‘released-on’,‘1947-11-15’)|
replace_node(‘Ray Taylor’,‘Ray Taylor (1888-12-01 to 1952-02-15)’)|</refinement>
```

## 附录 E 提示模板

本节提供 DeepRefine 模型的提示模板。我们在图 6、图 7 和图 8 中展示 DeepRefine 每个推理步骤的提示模板。

### 可回答性判断提示模板

**System：**

```text
作为高级判断助手，你的任务是根据给定 KG 上下文判断给定问题是否可回答。

根据提供的 KG 上下文评估给定问题是否可回答。请按如下格式输出你的判断：
<judge>Yes</judge> 或 <judge>No</judge>

**重要：** 在做出判断前，你必须仔细思考问题和 KG 上下文。并直接以指定格式输出判断结果。
```

**User：**

```text
Question:{question}
Knowledge Graph (KG) context:{triples_string}
Output:
<judge>Yes</judge> or <judge>No</judge>
```

![图 6](/src/image/deeprefine/fig6.png)

**图 6：DeepRefine 中可回答性判断的提示模板。**

### 错误溯因提示模板

**System：**

```text
作为高级错误溯因助手，你的任务是基于给定交互历史分析错误原因。

根据给定交互历史分析问题不可回答的原因。请按如下格式输出你的分析：
<abduction>...</abduction>

**重要：** 在做出分析前，你必须仔细思考交互历史。并直接以指定格式输出分析结果。
```

**User：**

```text
Interaction history:{interaction_history}
Output:
<abduction>...</abduction>
```

![图 7](/src/image/deeprefine/fig7.png)

**图 7：DeepRefine 中错误溯因的提示模板。**

### 精炼动作生成提示模板

**System：**

```text
作为高级知识图谱精炼助手，你的任务是生成一系列动作来精炼给定 KG，使其更适合回答给定问题。

基于给定 KG 和已分析的错误原因，精炼给定 KG，使其更容易被检索并用于回答给定问题。你可以执行以下三类动作：
- insert_edge(subject, relation, object)：向 KG 插入一条新边，以补全缺失信息。
- delete_edge(subject, relation, object)：从 KG 删除一条边，以移除冗余信息或冲突信息。
- replace_node(old_entity, new_entity)：替换 KG 中的一个实体，以纠正错误或处理消歧。

请按如下格式输出一系列动作：
<refinement>insert_edge("...", "...", "...")|delete_edge("...", "...", "...")|
replace_node("...", "...")|...</refinement>

**重要：** 在做出精炼前，你必须仔细思考给定 KG 和已分析的错误原因。不要从原始 KG 中删除任何无关三元组。
尽量保留原始 KG。并直接以指定格式输出精炼结果。
```

**User：**

```text
Original Text:{original_text}
KG:{triples_string}
Question:{question}
Error reasons:{error_reasons}
Output:
<refinement>insert_edge("...", "...", "...")|delete_edge("...", "...", "...")|
replace_node("...", "...")|...</refinement>
```

![图 8](/src/image/deeprefine/fig8.png)

**图 8：DeepRefine 中精炼动作生成的提示模板。**

## 附录 F 局限性

在本工作中，我们提出了一个用于智能体编译知识库精炼的通用 LLM 推理模型。尽管 DeepRefine 在实验中展现了
有前景的性能，但该模型仍有一些潜在改进空间。例如，我们仅从 HotpotQA 中使用简单过滤标准构建训练数据，
这使得训练数据质量还不够高。寻找更高质量、更多样化的训练数据是未来工作的一个有前景方向。此外，如何
将 DeepRefine 与稳健的系统化精炼框架结合，以使性能更好、更稳定，也仍然探索不足。
