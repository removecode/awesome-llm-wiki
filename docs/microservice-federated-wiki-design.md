# 微服务联邦知识库设计方案

## 目标与边界

- 本文只定义知识抽取、数据结构、跨仓组织和检索协议，不包含具体实现计划。
- 将内网已有的 `app-call-graph` 结果视为稳定输入，不在本方案中修复或改造它。
- 首期选一个业务域的 20–50 个仓验证模型，再推广到 900+ 仓。
- 存储与索引可插拔：权威知识仍落在第 4 节的文件知识包；检索后端默认用文件系统索引（无 SQL），
  团队共用时可换 Meilisearch/Tantivy，900+ 仓且需要统一 ACL/审计时再评估 PostgreSQL。首期不使用
  向量搜索。
- LLM 负责把确定性事实编译成人可读知识，不负责发明实体、接口或调用关系。

## 1. 为什么不能只生成“每仓一篇说明”

单篇总结会混合事实、推断和操作经验，更新时只能整篇重写，也无法支持精确的跨仓查询。
知识库应区分三类内容：

- **事实层 Facts**：可从代码、配置、API 定义、部署文件和 `app-call-graph` 确定性获得。
- **知识层 Claims**：LLM 基于事实编译出的职责、架构、约束、流程和故障模式，每项都引用事实。
- **关系层 Relations**：连接服务、接口、资源、业务能力、流程步骤和源码位置，支持跨仓导航。

Facts 和 Claims 是知识类型，Relations 是连接它们的横向结构。为了明确知识从来源到检索结果的演进过程，
采用下面的 K0–K4 分层模型。

## 2. 知识分层模型

### 2.1 K0 原始证据层

保存知识来源的定位信息，不把大段源码或敏感配置复制到知识库：

- 源码符号和行号、Proto/OpenAPI、配置与部署文件。
- README、ADR、设计文档和经过授权的部门文档。
- `app-call-graph` 扫描证据、Git commit SHA 和抽取器版本。

K0 回答“依据在哪里”。所有上层事实和结论都必须能够回溯到这一层。

### 2.2 K1 结构化事实层

保存从 K0 确定性抽取的机器可读对象：

- 服务、接口、事件、任务、数据资源、配置、部署单元和关键代码符号。
- 服务调用、接口消费、消息生产与消费、数据读写等有类型关系。
- 每项事实的稳定 ID、来源、commit、置信度、可见范围和内容哈希。

K1 回答“系统中确定存在什么”，主要对应每仓知识包中的 `facts/`。

### 2.3 K2 单服务知识层

LLM 将 K1 事实和仓内文档编译为供人和 Claude Code 阅读的服务知识：

- 服务职责、边界、架构、接口用途和内部工作流。
- 业务规则、不变量、设计决策、故障模式和运维知识。
- 每项 claim 引用 K0 证据或 K1 实体，并区分 `provisional` 与 `verified`。

K2 回答“一个服务为何存在、如何工作”，主要对应每仓知识包中的 `wiki/`。

### 2.4 K3 跨服务业务知识层

保存不属于任何单一 repo 的业务知识：

- 业务域、跨服务流程、共享能力和服务在流程中的角色。
- 每个流程步骤引用服务、入口契约、动作、输出、下一跳条件和证据。
- 已确认流程与根据调用图生成的候选路径必须明确区分。

K3 回答“多个服务如何共同完成业务”，存放在 `domain/<domain_id>/` 和 `flow/<flow_id>/`。

### 2.5 K4 检索与上下文视图层

K4 不创造新事实，而是针对当前问题组织 K0–K3：

- 服务目录、别名与术语索引、关系图和 section 全文索引（后端见 7.0 节，可插拔）。
- MCP 根据问题返回的 section、关系、证据和新鲜度提示。
- Claude Code 使用的有界 context package。

K4 回答“当前问题应该加载哪些知识”。例如查询业务流程时优先返回 K3 和 K2，只在需要核实时加载 K1 与 K0。

### 2.6 横向关系模型

Relations 不是独立的上下层，而是贯穿 K0–K3：

```text
业务流程 -- includes --> 服务
服务 -- exposes --> 接口
接口 -- calls --> 下游接口
结论 -- supported_by --> 事实或源码证据
```

### 2.7 覆盖等级与检索深度

知识层之外还要区分两个维度：

- **覆盖等级 C0–C3**：C0 服务目录；C1 结构化事实；C2 单服务 Wiki；C3 已进入确认的跨仓流程。对
  `procedure`（SOP/预案）复用同一把尺子，但 C3 的含义是“已演练或经真实事故验证有效”，不是“存在
  即可”，见 6.4 节。
- **检索深度 D0–D3**：D0 实体目录；D1 摘要与流程；D2 契约与业务规则；D3 代码与原始证据。

覆盖等级表示一个 repo（或一个跨仓对象）已建设到哪里，检索深度表示当前问题要加载多少。问答类任务通常
从 D0–D1 开始，编码类任务通常需要 D0–D3，但仍应按需加载。SOP/预案除了覆盖等级和检索深度之外，还需要
额外两个维度（执行风险、时效复核），这是 flow/domain 不需要但 procedure 必须有的，见 6.4 节。

## 3. 每个 repo 应抽取什么

### 3.1 服务身份与边界

- 稳定 ID：`repo_name`；别名包括 `app_id`、部署服务名、模块名和历史名称。
- 所属组织、负责人、业务域、生命周期、语言、框架、仓库 URL、默认分支和 commit SHA。
- 服务职责、明确不负责的事项、主要用户或调用方、核心业务能力。
- 来源：仓库元数据、CODEOWNERS、README、部署配置和人工维护的服务目录。

### 3.2 对外契约

- HTTP 路由、RPC service/method、消息 topic 与生产/消费方向、定时任务、CLI 或批处理入口。
- 每个契约记录名称、协议、方向、输入/输出类型、认证、错误语义、幂等性和源码入口。
- 优先从 OpenAPI、Proto、路由注册、消息配置和框架 AST 抽取；LLM 只补充用途说明。

### 3.3 数据与状态

- Mongo/Redis/Kafka 等已由 `app-call-graph` 发现的资源，以及数据库表、缓存 key 模式和对象存储桶。
- 记录读/写/发布/消费等动作、环境或 DC、事务边界、数据所有权和敏感级别。
- 只索引 schema、实体和访问语义，不复制数据内容、密钥或生产配置值。

### 3.4 代码结构与扩展点

- 启动入口、核心模块、领域对象、关键调用链、配置入口、插件或策略扩展点。
- 不索引全部函数；只保留公开接口、跨模块边界和解释业务行为所需的关键符号。
- 每个符号使用 `repo@commit:path#symbol` 作为定位信息。

### 3.5 运行与运维

- 部署单元、端口、DC、依赖组件、健康检查、配置项名称、关键指标、日志事件、告警和常见故障。
- Runbook、回滚方式、数据修复工具和已知限制必须标明来源；没有证据时写 `unknown`。
- 只涉及本服务的 SOP（重启、扩容、单服务数据修复）留在本仓 `wiki/operations.md`；跨服务/跨团队的
  应急预案单独存放，见 6.4 节的 `procedure/`，不要在多个服务里重复粘贴同一份预案。

### 3.6 设计与业务语义

- README、ADR、设计文档、需求文档中的业务术语、规则、不变量、权衡和历史决策。
- LLM 将分散信息编译为“能力、业务规则、典型流程、失败模式、设计决策”，但保留冲突而不是强行合并。

### 3.7 明确不抽取

- 第三方依赖的内部实现、生成代码、vendor、测试快照、二进制、大段源码和重复文档。
- 密钥、token、真实客户数据、生产配置值和无权限文档。
- 无证据的“最佳实践”描述，以及仅靠名称猜出的业务职责。

## 4. 每仓知识包结构

```text
service/<repo_name>/
├── manifest.yaml
├── facts/
│   ├── identity.yaml
│   ├── interfaces.jsonl
│   ├── data-resources.jsonl
│   ├── dependencies.jsonl
│   ├── code-map.jsonl
│   └── runtime.yaml
└── wiki/
    ├── overview.md
    ├── architecture.md
    ├── interfaces.md
    ├── workflows.md
    ├── operations.md
    └── decisions.md
```

- `facts/` 是机器可读、可增量替换的事实；`wiki/` 是供 Claude Code 和人阅读的编译结果。
- 并非所有仓都必须生成全部页面；`manifest.yaml` 明确记录已覆盖、缺失和不适用的主题。
- 跨仓页面不放入任一服务包，而放在 `domain/<domain_id>/`、`flow/<flow_id>/`、
  `procedure/<procedure_id>/` 和 `project/<project_id>/`，避免服务之间复制内容。
- 联邦根目录另有一份与后端无关的派生索引（可随时从知识包重建），供默认文件系统检索引擎使用：

```text
indexes/
├── aliases.yaml      # 别名 → 稳定 ID，支撑 resolve_entity
├── relations.jsonl   # subject|predicate|object，支撑 traverse_relations
├── sections.jsonl    # section_id、summary、keywords、path，支撑全文排序
└── error_book.yaml   # 编译/校验错误模式与约束，支撑 Error Book 问答（见 9 节）
```

## 5. 统一数据模型

### 5.1 所有实体共用的信封

```yaml
id: "endpoint:order-svc:http:POST:/orders"
kind: endpoint
service_id: "service:order-svc"
name: "CreateOrder"
aliases: ["创建订单"]
attributes:
  protocol: http
  direction: inbound
relations:
  - predicate: writes
    target: "mongo:order@orders"
provenance:
  - repo: order-svc
    revision: "<commit-sha>"
    path: api/routes.go
    symbol: registerCreateOrder
    lines: [42, 58]
    extractor: tree-sitter-go
confidence: high
visibility: internal
content_hash: "<sha256>"
```

关键规则：

- `id` 在版本间稳定，显示名称可以变化；所有跨仓链接只使用 ID。
- `kind` 限定为受控枚举：`service`、`capability`、`endpoint`、`event`、`job`、`data_resource`、
  `code_symbol`、`config`、`deployment`、`decision`、`workflow`、`failure_mode`、`procedure`、
  `project`。`workflow` 描述系统会自动做什么（服务调用链，可从调用图和代码验证）；`procedure`
  描述人应该怎么做（SOP/预案，见 6.4 节）；`project` 描述一次可交接的工作范围（目标、参与服务、
  未决事项与会话状态，见 6.5 节），三者不合并存储。新增 kind 前先看 5.3 的判断标准，避免每次都
  重新设计字段。
- `provenance` 必须存在；`confidence` 表示证据质量，不表示模型自信。
- 关系统一为 `subject - predicate - object`，并允许附加 `protocol`、`environment`、`dc`、`confidence`
  和证据。

### 5.2 编译知识的最小单位不是文档，而是 section

```yaml
section_id: "service:order-svc:overview:responsibility"
page_type: overview
title: 服务职责
summary: 接收订单创建请求并协调库存与支付预检查。
entities:
  - "service:order-svc"
  - "service:inventory-svc"
claims:
  - text: 创建订单前同步检查库存。
    evidence:
      - "endpoint:order-svc:http:POST:/orders"
      - "relation:order-svc:calls:inventory-svc"
keywords: [订单, 创建订单, 库存预检查]
updated_at: "..."
```

以 section 建索引可避免整篇文档命中；Claude Code 仍读取 Markdown 页面，但检索返回精确 section 和引用。

### 5.3 新增 kind 时先判断要不要加字段，再决定怎么加

多数新增内容不需要新字段：5.1 的通用信封已经有 `attributes`（自由 map）承载 kind 特有的扁平属性，
大部分新知识只是“多一种 kind + attributes 里多几个 key + 几条 relations”，不用碰 schema。只有当内容
命中下面某一条测试时，才需要像 `procedure` 那样新增顶层字段甚至新 MCP 方法：

1. **结构形态测试**：内容本身是有序/有分支的步骤，不是扁平键值对。命中 → 需要类似 flow/procedure
   的 `steps` 子结构，不能塞进 `attributes`。
2. **信任语义测试**：`confidence`（证据质量）和 `visibility`（可见范围）回答不了“能不能被自动执行/
   采信”这个问题，存在一个正交维度。命中 → procedure 因此新增了 `execution_authority`。
3. **失效机制测试**：内容的正确性不会随任何 `related` 实体变化自动失效（合同到期、组织调整、厂商
   API 下线），需要主动时间轮询，而不是 9 节默认的“被引用事实变化才标陈旧”。命中 → procedure 因此
   新增了 `review_cadence_days`/`last_validated`。
4. **检索形态测试**：回答典型问题需要一次性组装出专属结构（比如带分支的步骤序列），现有
   `get_service_context`/`traverse_relations`/`search_knowledge` 组装不出来。命中 → procedure 因此
   新增了 `get_procedure`。

**大概率不需要新字段的未来内容**（一条测试都不命中，直接用通用信封）：

- 团队与 on-call 归属：`kind: team`，`attributes` 放轮值表和升级联系人，relation `owns` → service。
- SLO/性能基线：`kind: slo`，`attributes` 放 p99、错误预算、容量上限，relation → endpoint。
- 成本归属：`kind: cost_center`，`attributes` 放月度花费和成本驱动因子。
- 术语表：不需要新 kind，直接扩展现有实体的 `aliases`，或在 K4 别名索引里加一条。
- 已知缺陷/技术债清单：`kind: known_issue`，`attributes` 放状态和优先级，区别于运行时的
  `failure_mode`。

**命中测试、需要新字段，但可以复用 `procedure` 已经踩过的坑，不用重新设计的未来内容**：

- 供应商/第三方依赖合同（命中测试 3：到期不会被代码变化触发）→ 复用“主动到期”字段模式，加
  `expires_at`/`review_cadence_days`。
- 合规/安全控制项（命中测试 2 和 3：谁有权限判定“已通过审计”正交于证据质量，且需要定期复审）→
  复用 `execution_authority`（语义变成“谁有权限做认证判断”）和 `review_cadence_days`。
- 迁移/发布计划（命中测试 1：本质是有序阶段加里程碑）→ 复用 flow/procedure 的 `steps` 子结构，只
  多几个 `target_state`/`cutover_date`/`rollback_plan` 字段，不需要第三套步骤 schema。

把新字段收敛成少数几个可复用的“字段包”：基础信封（默认都有）、步骤结构（workflow/procedure 及未来
迁移计划共用）、执行权限与时效复核（procedure 及未来合规/合同类对象共用）、数值基线（未来 SLO/成本
类对象共用）。新增 kind 时先套用已有字段包组合，只有真的出现第五种结构形态时才设计新包——这样 kind
数量会随 900+ 仓增长变多，但字段设计不会跟着线性膨胀。

## 6. 900 个 repo 如何串联

### 6.1 稳定 URI 与层级

- 服务：`kb://service/<repo_name>`
- 接口：`kb://endpoint/<service>/<protocol>/<canonical-name>`
- 资源：`kb://resource/<type>/<canonical-name>`
- 业务域：`kb://domain/<domain_id>`
- 流程：`kb://flow/<flow_id>`
- 预案：`kb://procedure/<procedure_id>`
- 项目：`kb://project/<project_id>`

组织层级是“组织 → 业务域 → 服务 → 页面 → section → evidence”，关系图则横向连接任意层级。

### 6.2 业务域不是硬分类

- 一个服务可属于多个业务域，并分别记录 `primary`、`supporting`、`shared` 角色。
- `wiki-toplo/scripts/community_detection.py` 的 Leiden 结果只生成候选标签。
- 公共认证、配置、日志、SDK 等 hub 单独标记为 `shared_capability`，不让它们主导社区划分。
- 业务域名称与关键流程由领域负责人确认；算法负责发现，不负责命名为事实。

### 6.3 业务流程是显式对象

- 一个流程由有序步骤构成；每步引用服务、入口契约、动作、输出和下一跳条件。
- `app-call-graph` 提供候选路径，仓内代码与文档提供语义，人工确认关键业务流程。
- 流程不复制服务说明，只链接到对应 service section；服务更新时可精确找到受影响流程。

### 6.4 SOP / 预案与业务流程分开存储

`flow`（6.3 节）描述系统会自动做什么，步骤是服务调用，可以从调用图和代码验证；SOP/预案描述人应该
怎么做，步骤是人工判断、操作命令和升级路径，主要来源是运维文档、事故复盘和人工确认。两者语义不同，
不应该合并成同一种对象，否则“已验证的调用链”和“未经验证的应急草稿”会互相污染置信度。

**新增实体类型 `procedure`**（与 5.1 的 `workflow` 并列），步骤 schema 与 flow 不同；它是 5.3 节四条
判断标准里同时命中结构形态、信任语义、失效机制三条测试的典型例子：

```yaml
id: "procedure:payment-incident-response"
kind: procedure
name: 支付失败应急预案
coverage_level: C3          # 见下方“SOP 的分级”
trigger: "告警 payment_error_rate_high，或人工上报支付失败率异常"
steps:
  - order: 1
    actor: "on-call SRE"
    action: "检查 payment-svc 与下游 bank-gateway 的健康检查和最近部署记录"
    decision: "若 bank-gateway 超时率升高，跳转步骤 3；否则跳转步骤 2"
    execution_authority: ai_assistable        # 只读诊断，Claude Code 可自动执行/建议
  - order: 2
    actor: "payment 团队负责人"
    action: "回滚最近一次 payment-svc 发布"
    expected_outcome: "错误率回落到基线"
    execution_authority: human_only           # 破坏性操作，只能提示步骤，不能代执行
escalation:
  condition: "15 分钟内错误率未恢复"
  target: "procedure:payment-major-incident"
related:
  - "service:payment-svc"
  - "workflow:checkout-flow"
  - "failure_mode:payment-svc:gateway-timeout"
review_cadence_days: 90
last_validated: "2026-05-01"
```

**SOP 的分级：能复用的和必须新增的**

检索深度 D0–D3 可以直接复用，不需要为 procedure 单独设计一套：D0 只解析出预案存在，D1 拿 trigger 和
摘要，D2 拿完整 steps 和 escalation，D3 才带出每一步对应的代码/配置证据。

覆盖等级 C0–C3 也可以复用，但含义要按 procedure 重新解释，而不是照搬“repo 建设进度”：

- C0：预案已被识别、有名字和 trigger，但没有结构化步骤（相当于知道“这类故障有预案”）。
- C1：`procedure.yaml` 已有结构化 `steps`/`escalation`（对应结构化事实层）。
- C2：`wiki/runbook.md` 已编译成人可读操作说明，且步骤被团队负责人审阅确认过。
- C3：被真实事故或演练（game day）验证过确实有效——这是 procedure 专属的“确认”含义，对应 6.3 节
  flow 里“已确认流程与候选路径必须区分”的同一原则：没打过演练的 C0–C2 预案，本质上和未确认的候选
  路径一样，不能被当作可以直接执行的操作。

但覆盖等级和检索深度都回答不了两个 SOP 特有的问题，因此新增两个维度：

- **执行风险分级 `execution_authority`（按步骤标注，不是按整份预案）**：`ai_assistable`（只读诊断，
  查日志/指标/健康检查，可以自动建议甚至执行）、`human_approval_required`（可逆操作，如重启、扩容、
  切流量，需要人确认后才能执行）、`human_only`（破坏性操作，如回滚发布、改生产数据，Claude Code 只
  能给出步骤，禁止代为执行）。同一份预案里前几步常是诊断、后几步才是高风险操作，按步骤标注才能让
  Claude Code 在恰当的地方自动停下来，而不是整份预案一刀切成“全禁止”或“全放行”。
- **时效复核 `review_cadence_days` + `last_validated`**：flow/domain 的陈旧检测靠“被引用的事实变了
  就标记陈旧”（9 节），这对 procedure 不够，因为团队重组、工具下线、云厂商 API 变更这类现实变化不
  会体现在调用图或接口定义里，`related` 也不会触发。所以 procedure 需要主动的时间轮询：超过
  `review_cadence_days` 没有人复核过，即使没有任何关联实体变化，也要标 `stale`。

严重度/优先级（P0/P1/P2）可以作为普通属性用于 7.2 排序信号里加权，不必再建一套独立的分级体系，
否则分级会越加越多，反而增加维护成本。

**存储位置按范围决定，不是按“是不是流程”决定**：

- 只涉及单个服务、纯操作步骤（重启、扩容、某个数据修复工具的用法）→ 留在该服务的
  `wiki/operations.md` 里作为一个章节，不新建顶层对象；这类 SOP 本地读者足够，没必要跨仓索引。
- 跨多个服务/多个团队的应急预案、发布协调、故障响应、大促保障 → 新建 `procedure/<procedure_id>/`，
  与 `domain/<domain_id>/`、`flow/<flow_id>/` 同级：

```text
procedure/<procedure_id>/
├── manifest.yaml
├── procedure.yaml   # K1：结构化的 trigger/steps/escalation，机器可读
└── wiki/
    └── runbook.md   # K2：人可读的操作说明，引用 procedure.yaml 里的每一步
```

- 组织级、不挂任何具体业务域的通用规程（新服务上线检查单、安全审查清单、on-call 交接规范）同样放进
  `procedure/<procedure_id>/`，但打 `scope: org` 标签，不强行挂到某个 `domain_id` 下。

**关系与确认状态**：

- 新增谓词：`triggered_by`（procedure → event/alert）、`mitigates`（procedure → failure_mode）、
  `escalates_to`（procedure → procedure）、`involves`（procedure → service，用法与 flow 的
  `includes` 对称）。
- 和 flow 一样，SOP 要区分“已被团队审阅确认过的正式预案”与“从事故复盘草稿自动整理的候选步骤”；这
  一区分直接映射到上面的 `coverage_level`——低于 C2 视为未确认候选，不能被当作可执行 SOP 直接返回给
  Claude Code，只能在结果里显式标记为候选并提示，禁止自动执行。
- SOP 不复制服务或流程说明，只通过 `related` 链接到对应 `service`/`workflow`/`failure_mode`；服务
  接口变化时，可以精确找到哪些预案引用了它，而不用逐份预案人工核对。

### 6.5 项目维度包：为 handoff 服务，不是为单次问答服务

传统问答场景按“一个问题 → 若干 section”组装上下文；项目交接（handoff）按“一个项目 → 其边界内全部
持续状态”组装上下文。两者目标不同：前者要答对一个事实，后者要让下一个人或下一个 Claude Code 会话
在不重新摸索的前提下接上工作。

因此新增顶层对象 `project/<project_id>/`，与 `domain/`、`flow/`、`procedure/` 同级，**只链接不复制**：

```text
project/<project_id>/
├── manifest.yaml
├── handoff.yaml     # K1：目标、范围、参与服务/流程、负责人、未决事项、最近会话锚点
└── wiki/
    ├── overview.md      # 项目目标与成功标准
    ├── scope.md         # 涉及哪些服务/flow/procedure（只列 ID）
    ├── status.md        # 当前进度、阻塞、风险
    └── open-questions.md
```

`handoff.yaml` 最小字段示意：

```yaml
id: "project:checkout-latency-opt"
kind: project
name: 结算链路延迟优化
goal: "P99 从 800ms 降到 400ms"
includes:
  services: ["service:order-svc", "service:payment-svc"]
  flows: ["workflow:checkout-flow"]
  procedures: []
owners: ["team:checkout"]
open_items:
  - text: "payment-svc 超时重试是否与库存预占冲突"
    related: ["endpoint:payment-svc:http:POST:/pay"]
session_anchor:
  last_commit: "<sha>"
  last_sections: ["service:order-svc:architecture:latency"]
  notes: "已排除 DB 慢查询，下一步看下游 RPC"
updated_at: "..."
```

约束：项目包不替代 service/flow 知识；handoff 加载时按 `includes` 去拉各包的 D1–D2 摘要，再叠
`status`/`open-questions`/`session_anchor`。会话结束时应回写 `handoff.yaml`，否则下一次交接会丢状态。

## 7. 不使用向量搜索时如何有效检索

### 7.0 存储与索引后端（可插拔）

检索协议（四级漏斗、场景配方、MCP 方法）与具体存储引擎解耦。权威内容永远是第 4 节的文件知识包；
索引层可以替换，但不能改变 `kb://` ID、section 边界或关系谓词。按阶段选型：

| 阶段 | 后端 | 目录/实体 | 关系 | section 全文 | 适用 |
|---|---|---|---|---|---|
| 默认（首期） | 文件系统索引 | `indexes/aliases.yaml` + 直接读知识包 | `indexes/relations.jsonl` | `indexes/sections.jsonl` + 本地 BM25 或 ripgrep | 20–50 仓验证；无 SQL、无常驻服务 |
| 团队共用 | 文件知识包 + Meilisearch 或 Tantivy | 同上，或同步进引擎的 filter 字段 | 仍用 `relations.jsonl`（或内存邻接表） | 引擎索引 section 文档 | MCP 多人使用，仍不想上 SQL |
| 规模与治理 | 可选 PostgreSQL（或等价） | 表 + 别名 | 边表 / 递归 CTE | `tsvector`/GIN 或外挂引擎 | 900+ 仓、统一 ACL、审计、复杂并发 |

约束：

- 关系层不塞进全文引擎：`calls`/`writes`/`includes` 等必须走显式邻接结构，否则影响面查询会退化成
  关键词碰运气。
- 换后端只换 7.1 漏斗第 4 步的实现和 MCP 的读写适配；不改场景配方，也不复制知识包。
- 首期验证正确性时优先文件系统路线；过早引入 PostgreSQL 会把“模型是否成立”和“数据库运维”缠在一起。

### 7.1 四级检索漏斗

1. **实体解析**：别名表将用户输入解析为 service、endpoint、resource、domain、flow、procedure 或
   project ID。
2. **结构过滤**：按权限、业务域、语言、环境、页面类型和实体类型缩小范围。
3. **关系扩展**：根据问题沿指定谓词扩展 1–2 跳，而不是遍历全图。
4. **全文排序**：在候选 section 上做关键词检索与字段加权排序（默认读 `indexes/sections.jsonl`；团队
   共用可换 Meilisearch/Tantivy；规模阶段可换 PostgreSQL `tsvector`），再组装上下文。

900 个仓并不是全文规模瓶颈；真正的问题是先定位正确的 5–20 个服务和页面，再做精确 section 检索。

### 7.2 排序信号

排名由可解释信号组合：

- 服务名、别名、接口名或业务术语精确命中。
- 标题、summary、keywords、正文的字段权重。
- 与已解析实体的图距离；直接相邻优先于二跳关系。
- 证据置信度、commit 新鲜度和人工确认状态。
- 业务域匹配与页面类型匹配。
- 公共 hub 降权、重复 section 去重、每服务结果数量封顶。

### 7.3 场景分类与知识加载策略

编码和问答不需要两套独立知识库；K0–K4 与同一份文件知识包（及可选索引后端）是同一份数据，场景之间的
区别只是**检索配方不同**——加载哪些知识层、扩到多深、走哪些 MCP 调用。这与 2.5 节对 K4 的定义、
4 节“避免服务间复制内容”的原则一致：新增场景时应该扩展检索配方，而不是新建存储或复制知识包。

两类场景的默认深度不同：问答场景默认止步于 D0–D2，且很少沿关系跨包展开；编码场景默认要落到 D2–D3 的
代码证据，并经常需要同时打开服务包、domain 包和 flow 包。此外还有一类**协作与质量场景**：目标不是答
一个点状问题，而是交接整包项目状态，或查询/消费编译期 Error Book。下面按子场景列出典型问题、涉及的
知识层/覆盖等级、检索深度和 MCP 调用序列，作为 7.1 检索漏斗在具体场景下的落地方式。

**问答场景（不产出代码变更，目标是准确且可追溯的回答）**

- **服务概览**（“X 服务是做什么的/负责什么”）：K2 `overview`/`architecture`；C2；D0–D1；只做实体解析，
  不扩图。
- **调用关系问答**（“谁调用 X”/“X 依赖哪些服务”）：K1 `dependencies.jsonl` + K2 `architecture.md`；
  C1–C2；D1；`traverse_relations(predicates=[calls], depth=1)`。
- **接口契约问答**（“这个接口的参数/协议/幂等性是什么”）：K1 `interfaces.jsonl` + K2 `interfaces.md`；
  C1–C2；D1–D2；必要时 `get_evidence` 核实签名。
- **数据归属问答**（“这个字段/topic/表是谁写的、谁消费”）：K1 `data-resources.jsonl` +
  `writes/publishes/consumes` 反向边；C1；D1；反向 `traverse_relations`。
- **业务流程问答**（“一次下单经过哪些服务”）：K3 flow；C3，没有已确认 flow 时降级为候选路径并显式
  标记；D1–D2；`get_flow` 或有界路径搜索。
- **只读影响面问答**（“这个字段/接口变了会影响谁，先不改代码”）：K1 relations + K3 flow；C2–C3；D1–D2；
  `analyze_change_scope` 只读组装候选影响面，不落地代码。
- **代码定位问答**（“某个行为在哪里实现”）：service + 业务词 → K1 `code-map.jsonl` 相关 section + K0
  证据；C1–C2；D2–D3——比多数问答更深，因为要给出可跳转的代码位置。
- **运维/故障问答**（“线上报错 XX 可能是什么原因/怎么排查这个告警”）：K2 `operations.md` + failure
  mode，跨服务应急场景再加 K3 `procedure`；C2–C3；D1–D2，涉及日志或配置细节时到 D3；
  `get_service_context(aspects=operations)` 或 `get_procedure(procedure_id)`。
- **设计决策问答**（“为什么这么设计/有没有考虑过别的方案”）：K2 `decisions.md`；C2；D1–D2；存在冲突
  claims 时必须原样返回并各自标注来源，不强行合并。
- **组织与术语问答**（“这个服务属于哪个域/团队”/“XX 是什么缩写”）：K1 identity + 别名索引（K4）；
  C0–C1；D0；只走 `resolve_entity`，几乎不扩图。
- **Error Book / 编译质量问答**（“这个悬空链接/无证据 claim 为什么反复出现”/“上次增量编译修了哪些
  错误”/“有没有跨页矛盾还没关闭”）：读 `indexes/error_book.yaml`（及条目引用的 section/实体）；
  C1–C2；D1–D2；`search_error_book` 或 `get_error_book(filters)`。答案必须带错误类型、根因约束、
  生命周期状态（open/closed）和是否已注入编译 prompt——这是对知识库自身质量的问答，不是对业务
  代码行为的问答，与运维/故障问答分开。

**编码场景（会产出代码修改，需要更深的契约、代码结构和证据，常跨多个知识包）**

- **单服务接口新增/修改**（“给 X 服务加个接口/改现有接口的返回结构”）：K1 `interfaces.jsonl` + K0
  代码符号 + K2 `interfaces.md`/`workflows.md`；C1–C2；D2–D3；`get_service_context` 后用
  `get_evidence` 定位到 `repo@commit:path#symbol`。
- **故障修复/Bug 定位**（“订单偶尔重复创建，定位并修复”）：K1 `code-map.jsonl` + K2 `operations.md`/
  failure mode + K0 证据；C2；D3——必须落到代码符号，不能停在摘要层面。
- **按预案执行应急操作**（“按 SOP 处理这次支付故障/执行数据修复脚本”）：K3 `procedure` + 涉及服务的
  K1 `code-map.jsonl`/`runtime.yaml` + K0 证据；C2–C3；D2–D3；`get_procedure` 之后对预案涉及的每个
  服务分别 `get_service_context` + `get_evidence`；按每一步的 `execution_authority` 执行——
  `ai_assistable` 可以直接跑，`human_approval_required`/`human_only` 必须停下来交给人。
- **跨服务功能开发/联调**（“实现一个跨 3 个服务的新流程/给现有流程加一步”）：K3 flow + 涉及每个服务的
  K1/K2；C2–C3；D2–D3，跨越 domain/flow 包与多个服务包；`get_flow` 后对每个步骤服务分别调用
  `get_service_context` + `get_evidence`。
- **变更影响面实施**（“要改这个共享字段/接口，先看哪些服务要跟着改，然后真的去改”）：K1 relations +
  K3 flow + 受影响服务的 K2；C2–C3；D2–D3；`analyze_change_scope` 之后逐个受影响服务补 D2–D3，禁止
  无界传播。
- **重构/技术债处理**（“把这个模块拆出来/替换这个依赖”）：K1 `code-map.jsonl`/`dependencies.jsonl` +
  K2 `architecture.md`/`decisions.md`（先搞清楚为什么现在这样再改）；C2；D2–D3。
- **新服务接入/规范对齐**（“新建服务，要符合现有鉴权/日志/配置约定”）：K3 `shared_capability` + 同域
  参照服务的 K2 全量；C2；D1–D2；先看公共能力和参照服务约定，再决定新服务怎么写。
- **配置/部署变更**（“加个环境变量/改端口/挪 DC”）：K1 `runtime.yaml` + K2 `operations.md`；C1–C2；
  D2，通常不需要 D3 源码。
- **数据/Schema 变更**（“这个表/topic 要加字段，看看谁会受影响”）：K1 `data-resources.jsonl` + 反向
  `reads/writes/consumes` 边 + 命中的 K3 flow；C2–C3；D2–D3；对每个消费方补 `get_service_context` +
  `get_evidence` 再评估改动。

**协作与质量场景（整包交接或知识库自检，不是点状业务问答）**

- **项目维度 handoff**（“把结算延迟优化这个项目交给下一班/下一个会话接着做”）：K3 `project` 的
  `handoff.yaml` + `wiki/status.md`/`open-questions.md`，再按 `includes` 拉取相关 service/flow/
  procedure 的 D1–D2 摘要；C2–C3；默认 D1–D2，接手后编码再加深到 D3；`get_project_context
  (project_id)` 一次返回范围清单、未决事项、`session_anchor` 和陈旧警告。与传统问答的区别：入口是
  项目 ID 而不是自然语言单问；输出是可续作的工作包，而不是单个答案；会话结束应回写 handoff 状态。
- **Error Book 错误问答**（同上问答场景条目）：查询编译期结构/语义错误模式、已注入约束与修复状态；
  编码或增量编译前也可先扫 open 条目，避免重复引入同类错误。

**场景差异小结**

- 默认检索深度：问答 D0–D2；编码 D2–D3；项目 handoff 先 D1–D2 整包，再按需加深。
- 关系扩展：问答通常 0–1 跳且只读；编码常需 1–2 跳，并要求核实证据后才能改代码；handoff 按项目
  `includes` 批量拉包，不做无界扩图。
- 典型知识包：问答多数只读单服务 `wiki/` 和 domain/flow 摘要；编码常同时打开 `facts/`、`wiki/`、
  domain/flow/procedure；handoff 以 `project/` 为根再链接各包；Error Book 问答读 `indexes/
  error_book.yaml`。
- 是否落到代码符号：问答很少（代码定位、故障排查类除外）；编码几乎总是；handoff 默认不落，除非未决
  事项点名了具体符号。
- 新增子场景的方式：都是新增一条检索配方（知识层 + 深度 + MCP 调用序列），不新建知识库或复制知识包。

### 7.4 上下文组装

- 返回顺序固定为：命中的结论 → 相关关系 → 原始证据 → 新鲜度/置信度警告。
- 对每个服务和页面设置 token 预算，优先返回 summary；Claude Code 需要时再调用 `get_evidence`。
- 所有结果携带 `repo`、`commit_sha`、`section_id`、`citations`、`confidence` 和 `stale`。
- MCP 只检索和打包，不在查询时再次总结；最终推理由 Claude Code 完成。

### 7.5 缓存分层与失效

MCP 服务要承受真实查询量时，检索延迟不能每次都打到权威存储。参照 WikiKV（Ming et al., 2026，
arXiv:2606.14275）在工业场景验证过的三层缓存设计，按访问倾斜程度分层，而不是对所有内容用同一套
缓存策略：

- **L1 进程内缓存**（容量：几十条）：业务域目录（domain 列表）和每个域下的服务清单，进程启动时
  预热，常驻不过期，只在收到失效事件时刷新——这是几乎每次查询都会命中的最热路径，直接从网络往返
  里拿掉。
- **L2 共享缓存**（容量：几千条，如 Redis；单机 MCP 可省略，退化为扩大的 L1）：按 `access_count`
  （见下）识别出的热门服务 `overview`/`architecture` section 和高频 flow/procedure，LRU + TTL
  淘汰，被多个 MCP 进程共享。
- **L3 权威存储**：默认是文件知识包 + `indexes/`；换 Meilisearch/Tantivy 或 PostgreSQL 后仍以知识包
  （或由其同步出的权威副本）为真理来源，不设被动过期——陈旧检测交给 9 节的主动机制（`related`
  变化触发 + procedure 的 `review_cadence_days`）。

**失效方式**：写入管道（facts 更新、section 重编译、indexes 重建）完成后发布一条带 `kb://` 路径的
失效事件；L1/L2 异步订阅并按前缀刷新对应条目，在线查询不需要等锁或轮询检测变化。离线重建时，
在线查询最坏情况只是多打一次 L3，不会读到半更新的结果。文件系统后端下，失效也可简化为“更新
`indexes/` 后清空 L1”。

**驱动缓存策略的字段**：给信封（5.1 节）补一个 `access_count` 字段，随 MCP 调用自增，L2 用它判断
该缓存谁、该淘汰谁；这个字段不改变检索逻辑，只是给缓存层一个客观依据，不用额外接入日志系统。

## 8. MCP 的设计边界

- `resolve_entity(query)`：解析服务、接口、资源、业务域、流程、预案和项目，返回消歧候选。
- `get_service_context(service_id, aspects)`：按 `overview/interfaces/data/operations` 精确取 section。
- `search_knowledge(query, filters)`：受范围约束的全文检索，不允许默认全库宽搜。
- `traverse_relations(entity_id, predicates, direction, depth)`：有类型、有深度限制的图查询。
- `get_flow(flow_id)`：返回已确认流程；候选路径必须显式标记。
- `get_procedure(procedure_id)`：按 `coverage_level` 返回 SOP/预案；低于 C2 的候选步骤必须显式
  标记，禁止当作可执行操作直接返回；返回结果必须带每一步的 `execution_authority`，供 Claude Code
  判断哪些步骤可以自动执行、哪些必须等人确认。
- `get_project_context(project_id)`：返回项目 handoff 包（目标、范围 ID 列表、未决事项、
  `session_anchor`、陈旧警告），并按 `includes` 附带各关联包的 D1 摘要；不在此时展开到 D3 源码。
- `get_error_book(filters)` / `search_error_book(query)`：按错误类型、实体、生命周期状态查询
  Error Book 条目（现象、根因约束、是否已注入编译 prompt、open/closed）。
- `get_evidence(ids)`：返回源码或扫描证据定位，不默认发送大段源码。
- `analyze_change_scope(changed_entities)`：组合反向关系与流程，输出候选影响面而非确定结论。

## 9. 更新与质量模型

- Git commit 变化先更新 facts；只有被变更 facts 引用的 section 才重编译。
- 接口、事件、资源或调用边变化时，标记关联 flow/domain/procedure 页面陈旧，不立即全量重写。
- procedure 额外需要主动的时效巡检：超过 `review_cadence_days` 未复核就标 `stale`，不能只依赖
  `related` 实体变化触发，因为组织、工具、云厂商变化通常不会反映在调用图或接口定义里（见 6.4 节）。
- 保留矛盾 claims 及各自证据；来源优先级为机器契约/代码配置、人工确认文档、普通文档、模型推断。
- 质量检查覆盖 schema、稳定 ID、引用存在性、孤立实体、过期 section、无证据 claim、重复实体和 ACL。
- **Error Book（跨批次质量记忆）**：将重复出现的编译/校验错误沉淀为 `indexes/error_book.yaml` 条目
  （现象、根因、约束规则、验证方式、open/closed）。每批编译后：确定性代码修复结构类错误（悬空链接、
  格式错误引用、索引不一致）；周期性 LLM 修复内容类错误（无证据 claim、跨页矛盾）。open 约束注入
  下一次编译 prompt，避免同类错误反复出现。这为 7.3 的 Error Book 问答提供数据面，也改进后续
  wiki 编译质量（思路对齐 LLM-Wiki / WikiKV 的 Error Book，但落在本方案的文件索引上）。
- 采用分级覆盖：全量 900+ 仓先达到 C0；核心仓达到 C1–C2；关键跨仓流程与经过演练的关键预案达到 C3；
  进行中的重点工作用 `project/` 包维护 handoff 状态。

## 10. 设计验证方式

- 从目标业务域收集 30–50 个 Claude Code 真实问题，并按上述检索配方分类。
- 选择 20–50 个仓，人工建立服务、接口、资源、调用边和 5–10 条关键流程的黄金集。
- 验证重点不是文案质量，而是：实体是否抽全、ID 是否稳定、关系是否可追溯、正确 section 能否进入前列、
  上下文是否在预算内、过期与低置信度是否被明确暴露。
- 只有真实问题中出现稳定的同义词召回缺口，才在第二阶段评估向量搜索；向量不能替代实体解析和关系过滤。
  WikiKV（Ming et al., 2026，arXiv:2606.14275）的消融实验印证了这一点：移除词法路径路由退回逐层
  导航后，端到端正确率从 65.6% 掉到 40.2%，而向量检索在其架构中被限定只服务叶子内容、与路径路由
  正交——真实生产系统里，实体/路径路由比向量语义匹配更决定检索质量。
