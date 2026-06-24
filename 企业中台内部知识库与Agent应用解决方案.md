# 企业中台内部知识库与Agent应用解决方案

# 企业中台内部知识库与 Agent 应用解决方案

## 1\. 方案定位

本方案面向两个主要业务域：

1. **电商售后体系**

    - 售后政策、商品知识、订单、物流、退款、补发、赔付、历史工单。

    - 重点解决客服检索效率、工单处理质量、政策执行一致性和复杂问题调查。

2. **研发体系**

    - 产品文档、API 文档、架构设计、ADR、研发规范、发布记录、Issue、CI 日志和故障复盘。

    - 只建设轻量级知识与工程工具链，不建设 Coding Agent。

    - 系统不得自动修改代码、执行 Shell、提交 Commit、创建合并请求或直接操作生产环境。

总体定位是：

> 建设一套“统一治理、分域检索、证据驱动、受控执行”的企业知识中台，并在其上提供有限自治的 Agent 应用。

---

## 2\. 核心设计原则

### 2\.1 共平台、分域索引

售后和研发共享：

- 数据接入平台

- 文档解析能力

- 身份权限体系

- 模型网关

- Agent Runtime

- 可观测与评测平台

但不共享同一套：

- 分块规则

- 查询改写策略

- 检索权重

- 重排模型配置

- 新鲜度规则

- 回答模板

售后知识侧重政策时效、订单状态、商品属性和工单案例；研发知识侧重术语、版本、API、模块、仓库、分支和变更历史。

### 2\.2 确定性工作流优先，Agent Loop 按需启用

按复杂度将任务划分为三类：

|类型|实现方式|示例|
|---|---|---|
|单次知识问答|单轮 RAG|查询退款政策、API 参数|
|固定业务流程|确定性 Workflow|工单分类、审批、通知、超时处理|
|动态调查任务|有界 Agent Loop|跨订单、物流、政策和工单调查复杂售后问题|

不使用通用 Agent 处理所有请求。

### 2\.3 所有边界使用结构化数据

以下数据均使用 JSON Schema 或等价类型定义：

- `QueryIntent`

- `RetrievalRequest`

- `RetrievedEvidence`

- `ToolRequest`

- `ToolResult`

- `ApprovalRequest`

- `AgentDecision`

- `FinalAnswer`

- `Citation`

- `ErrorObservation`

模型不得通过自由文本隐式控制业务流程。

### 2\.4 权限先于检索

用户身份、组织、角色、业务线、租户和数据密级必须在检索前注入查询过滤条件。

禁止采用：

> 先检索全部数据，再让模型判断哪些内容可以展示。

### 2\.5 证据先于答案

每个答案至少包含：

- 结论

- 证据引用

- 来源系统

- 文档版本

- 生效时间

- 数据更新时间

- 置信状态

- 无法确认时的拒答或升级建议

### 2\.6 读取默认开放，写入默认受限

工具按风险分级：

|等级|能力|策略|
|---|---|---|
|L0|检索、读取、查询|可自动执行|
|L1|摘要、分类、草稿、建议|自动执行，结果供人工使用|
|L2|路由工单、创建待办、请求补充资料|策略允许后执行并留痕|
|L3|退款、赔付、补发、权限修改、生产变更|强制人工审批|
|L4|不可逆或高风险操作|不向 Agent 开放|

---

## 3\. 总体架构

```mermaid
flowchart TB
    subgraph APP["用户与业务应用层"]
        A1["售后客服工作台"]
        A2["工单运营平台"]
        A3["企业知识搜索门户"]
        A4["研发知识助手"]
        A5["故障与发布分析助手"]
    end

    subgraph ACCESS["统一接入与安全网关"]
        GW["AI Gateway<br/>SSO / OIDC / RBAC / 限流"]
        POLICY["Policy Engine<br/>权限、风险、工具授权、审批"]
    end

    subgraph AGENT["RAG 与 Agent Runtime"]
        ROUTER["Domain Router<br/>售后 / 研发 / 通用"]
        QUERY["Query Understanding<br/>意图、实体、查询改写"]
        RETRIEVAL["Retrieval Orchestrator<br/>多路召回、过滤、融合、重排"]
        CONTEXT["Context Builder<br/>证据压缩、去重、引用"]
        LOOP["Bounded Agent Loop<br/>有限步骤、有限工具、停止条件"]
        WORKFLOW["Durable Workflow<br/>审批、重试、超时、补偿"]
        MODEL["Model Gateway<br/>模型路由、配额、降级"]
        TOOLREG["Tool Registry / MCP Gateway"]
    end

    subgraph KNOWLEDGE["知识与索引层"]
        SEARCH["OpenSearch / Elasticsearch<br/>BM25 + Vector + Metadata"]
        VECTOR["独立向量库<br/>按规模可选 Milvus"]
        GRAPH["关系知识<br/>Neo4j，可选"]
        CACHE["Redis<br/>会话、热点、短期状态"]
        META["PostgreSQL<br/>元数据、ACL、版本、任务状态"]
        OBJECT["对象存储 / 湖仓<br/>原文、解析结果、历史版本"]
    end

    subgraph PIPELINE["知识处理与治理层"]
        CONNECT["连接器与事件接入<br/>API / Webhook / CDC / Batch"]
        PARSE["文档解析<br/>版面、表格、OCR、代码块"]
        NORMALIZE["规范化<br/>去重、脱敏、分类、质量检查"]
        CHUNK["结构化切分<br/>章节、语义、父子块"]
        EMBED["Embedding / Sparse Index / Rerank 数据准备"]
        ACL["ACL 与数据分级同步"]
        VERSION["版本化发布<br/>增量索引、Alias 切换、回滚"]
    end

    subgraph SOURCE["企业数据源"]
        S1["订单 / 物流 / 退款 / 商品"]
        S2["客服工单 / 会话 / 通话转写"]
        S3["FAQ / SOP / 政策 / 培训资料"]
        S4["Wiki / API / ADR / 设计文档"]
        S5["Git 元数据 / Issue / CI / 发布记录"]
        S6["故障复盘 / 监控告警 / 运维手册"]
    end

    subgraph GOVERN["横向治理与可观测"]
        OBS["OpenTelemetry + MLflow<br/>Trace、成本、质量、评测"]
        PROMPT["Prompt / Tool / Schema Registry"]
        EVAL["离线评测集与线上反馈"]
        AUDIT["审计、DLP、数据生命周期"]
    end

    APP --> GW
    GW --> POLICY
    POLICY --> ROUTER
    ROUTER --> QUERY
    QUERY --> RETRIEVAL
    RETRIEVAL --> SEARCH
    RETRIEVAL --> VECTOR
    RETRIEVAL --> GRAPH
    RETRIEVAL --> CONTEXT
    CONTEXT --> LOOP
    LOOP --> MODEL
    LOOP --> TOOLREG
    LOOP --> WORKFLOW
    WORKFLOW --> POLICY

    SOURCE --> CONNECT
    CONNECT --> PARSE
    PARSE --> NORMALIZE
    NORMALIZE --> CHUNK
    CHUNK --> EMBED
    ACL --> EMBED
    EMBED --> VERSION
    VERSION --> SEARCH
    VERSION --> VECTOR
    NORMALIZE --> OBJECT
    VERSION --> META

    TOOLREG --> S1
    TOOLREG --> S2
    TOOLREG --> S5
    TOOLREG --> S6

    OBS -.-> AGENT
    OBS -.-> PIPELINE
    PROMPT -.-> AGENT
    EVAL -.-> AGENT
    AUDIT -.-> ACCESS
    AUDIT -.-> KNOWLEDGE```



---

## 4\. 知识处理与 RAG 设计

### 4\.1 数据接入

#### 售后数据

- 商品中心

- 订单系统

- 物流系统

- 退款和赔付系统

- CRM

- 客服工单系统

- 售后政策

- 商品说明书

- 历史客服会话

- 通话转写

#### 研发数据

- Confluence、Wiki 或内部文档平台

- API 管理平台

- ADR 和架构设计

- Git 仓库元数据

- Issue 和需求系统

- CI/CD 任务结果

- 测试报告

- 发布记录

- 故障复盘

- 运维手册

研发侧默认只接入只读 API。

### 4\.2 统一知识对象

建议定义统一的 `KnowledgeDocument`：

```Plaintext
document_id
source_system
source_uri
domain
tenant_id
business_unit
document_type
title
version
language
classification
owner
effective_from
effective_to
updated_at
checksum
acl_users
acl_groups
product_ids
order_scope
repository
branch
service_name
section_path
page_number
parent_chunk_id
content
```

索引中的每个 Chunk 必须能够反向定位到：

- 原始文档

- 原始段落或页码

- 当前版本

- 权限主体

- 文档 Owner

- 生效状态

### 4\.3 文档解析

首选：

- **Docling**：用于 PDF、Word、PPT、表格、公式和版面结构解析。

- OCR 引擎按已有企业能力选择。

备选：

- Unstructured

- 云厂商文档解析服务

- 针对特殊格式的自研解析器

不建议直接使用简单 PDF 文本提取后按固定字符数切分。

### 4\.4 分块策略

#### 售后文档

按照以下层级切分：

```Plaintext
政策
  └─ 地区
      └─ 商品类型
          └─ 售后场景
              └─ 条款
```

必须保留：

- 生效时间

- 失效时间

- 地区

- 渠道

- 商品类别

- 会员等级

- 订单状态

- 例外条件

#### 研发文档

按照以下单位切分：

- 文档章节

- API 定义

- ADR 决策

- 服务模块

- 配置项

- 错误码

- 发布版本

- 故障步骤

代码本身不作为普通自然语言文档随意分块。代码搜索以精确搜索、符号和仓库元数据为主，语义检索只作为辅助。

### 4\.5 检索流程

默认检索链路：

```Plaintext
用户身份解析
→ 领域识别
→ 查询规范化
→ 实体和约束提取
→ ACL / Tenant / Metadata 过滤
→ BM25 与 Dense Retrieval 并行
→ RRF 或加权融合
→ Reranker 精排
→ 去重与父子块扩展
→ 证据充分性判断
→ 上下文构造
→ 回答或继续检索
```

#### 售后检索权重

优先级通常为：

1. 当前有效政策

2. 订单和物流实时状态

3. 商品知识

4. 同类历史工单

5. 通用 FAQ

#### 研发检索权重

优先级通常为：

1. 精确术语、API、错误码、配置项和版本号

2. 当前版本的正式设计文档

3. ADR 和发布记录

4. Issue、CI 和故障复盘

5. 语义相似文档

### 4\.6 PageIndex 的定位

PageIndex 或文档树检索不作为企业全局主索引。

适合应用于：

- 超长产品手册

- 售后政策全集

- 合同

- 标准规范

- 长篇技术设计

- 单份复杂 PDF

推荐采用两阶段模式：

```Plaintext
企业级 Hybrid Search 定位文档
→ 文档树或章节导航定位文档内部内容
```

---

## 5\. Agent Runtime 设计

### 5\.1 Agent 不是聊天机器人外壳

Agent Runtime 至少包含：

- Turn 管理

- Session 状态

- Typed Event

- Tool Registry

- Context Builder

- Model Router

- Policy Engine

- Guardrail

- Approval

- Trace

- Evaluation Hook

- Completion Criteria

每次运行形成完整事件流：

```Plaintext
UserInput
→ IntentResolved
→ RetrievalRequested
→ EvidenceRetrieved
→ ModelRequested
→ ToolRequested
→ ToolCompleted
→ ApprovalRequested
→ ApprovalResolved
→ FinalResponse
```

每个事件记录：

- `run_id`

- `turn_id`

- `agent_id`

- `tool_id`

- `parent_event_id`

- `timestamp`

- 输入 Schema 版本

- 输出 Schema 版本

- Evidence IDs

- Policy Decision

- Token Usage

- Latency

- Error Information

### 5\.2 推荐的 Agent 拓扑

不建设复杂的多 Agent 自组织网络。

采用以下简单拓扑：

```mermaid
flowchart LR
    M["Master Domain Router"] --> S["售后知识与调查 Agent"]
    M --> R["研发知识 Agent"]
    M --> G["通用企业搜索"]

    S --> ST1["订单/物流查询工具"]
    S --> ST2["政策检索工具"]
    S --> ST3["工单检索工具"]
    S --> ST4["工单草稿与路由工具"]

    R --> RT1["文档与 API 检索"]
    R --> RT2["Git 元数据只读查询"]
    R --> RT3["Issue / CI 只读查询"]
    R --> RT4["发布与故障知识检索"]```

Master Router 尽量采用单次分类或短路由，不进行开放式长循环。

专业 Agent 只在需要多来源调查时进入有限 Tool\-Call Loop。

### 5\.3 Agent Loop 限制

建议默认设置：

- 最大检索轮次：2

- 最大工具调用次数：4

- 最大专业 Agent 交接次数：1

- 禁止递归创建 Agent

- 禁止动态发现未授权工具

- 禁止执行任意代码或 Shell

- 达到限制后必须输出部分结果、缺失信息和升级建议

### 5\.4 MCP 使用原则

MCP 用于标准化以下能力：

#### Resources

- 政策文档

- API Schema

- 商品知识

- 工单字段定义

- 服务目录

- 只读数据库视图

#### Tools

- 查询订单

- 查询物流

- 查询历史工单

- 创建工单草稿

- 查询 CI 结果

- 查询发布版本

- 查询服务 Owner

#### Prompts

- 售后调查模板

- 故障分析模板

- 发布检查模板

- 工单摘要模板

但 MCP 不替代：

- 身份认证

- 工具授权

- 审批

- 审计

- 超时与重试

- 业务幂等

- 风险策略

内部高风险工具必须经过企业 Tool Gateway 再调用。

### 5\.5 Workflow 与 Agent 的边界

以下能力使用确定性 Workflow：

- 工单状态机

- 审批

- 超时提醒

- SLA 升级

- 消息通知

- 重试

- 补偿

- 幂等控制

以下能力使用模型：

- 意图识别

- 查询改写

- 信息抽取

- 证据比较

- 摘要

- 草稿

- 动态决定下一步查询哪个只读工具

---

## 6\. Agent 应用设计

### 6\.1 售后客服 Copilot

输入：

- 客户问题

- 订单号

- 当前工单

- 客服上下文

输出：

- 问题分类

- 相关政策

- 订单和物流状态

- 建议回复

- 证据引用

- 建议下一步动作

- 风险提示

不自动执行退款或赔付。

### 6\.2 工单分类与路由 Agent

能力：

- 意图和原因分类

- 商品、订单和故障实体提取

- 优先级判断

- 重复工单识别

- 相似工单推荐

- 目标团队建议

- 草稿回复

采用“模型建议 \+ 规则校验”的双重判断。

### 6\.3 复杂售后调查 Agent

用于处理：

- 政策与订单状态冲突

- 多次退款失败

- 物流异常

- 跨系统数据不一致

- 多工单关联问题

执行流程：

```Plaintext
读取工单
→ 查询订单
→ 查询物流
→ 检索当前有效政策
→ 检索相似工单
→ 对比冲突
→ 输出调查结论和建议
```

该 Agent 以只读调查为主。

### 6\.4 售后政策合规检查

对客服回复或处理方案进行检查：

- 引用政策是否有效

- 是否适用于当前地区和渠道

- 是否满足订单状态条件

- 是否超出客服权限

- 是否需要主管审批

### 6\.5 研发知识助手

支持：

- API 查询

- 架构设计查询

- ADR 决策查询

- 研发规范查询

- 模块和服务 Owner 查询

- 发布版本查询

- 错误码和配置项查询

- 文档对比和摘要

明确不提供：

- 代码生成执行

- 自动修改仓库

- 自动创建 Commit

- 自动创建 Merge Request

- 自动运行 Shell

- 自动部署

### 6\.6 CI 与故障分析助手

读取：

- CI 失败日志

- 测试报告

- 历史 Issue

- 故障复盘

- 发布记录

- 服务依赖信息

输出：

- 失败阶段

- 关键错误

- 可能关联模块

- 相似历史问题

- 建议排查步骤

- 相关 Owner 和文档

该应用是“调查和导航助手”，不是自动修复 Agent。

### 6\.7 发布影响摘要

根据版本、Issue 和服务目录生成：

- 本次变更范围

- 受影响服务

- 相关接口

- 已知风险

- 测试覆盖

- 回滚资料

- 历史相似发布

---

## 7\. 技术选型建议

|层级|首选方案|备选方案|选型说明|
|---|---|---|---|
|原始数据存储|S3、OSS、COS 或 MinIO|企业湖仓|保留原文和版本，作为事实底座|
|元数据与控制面|PostgreSQL|企业现有关系库|ACL、版本、任务、反馈、配置|
|缓存与短期状态|Redis|PostgreSQL|会话、热点答案、幂等和临时状态|
|文档解析|Docling|Unstructured、云解析服务|结构化版面、表格、OCR|
|主搜索引擎|OpenSearch|Elasticsearch|同时支持 BM25、向量、过滤与聚合|
|小规模向量能力|OpenSearch Vector 或 pgvector|Weaviate|减少早期组件数量|
|大规模向量能力|Milvus|Weaviate|仅在规模和性能指标证明需要时引入|
|图谱|暂不默认建设|Neo4j|只用于产品层级、故障树和依赖关系|
|Embedding|Qwen3\-Embedding 0\.6B/4B|BGE\-M3|中文、多语言和私有部署|
|Reranker|Qwen3\-Reranker 0\.6B/4B|BGE Reranker|对 Top\-K 做精排|
|Agent API 层|FastAPI \+ Pydantic|Java/Spring 或 Go 自研|显式 Schema 和类型约束|
|Agent Harness|自研薄层|OpenAI Agents SDK、Google ADK、PydanticAI|不引入 LangChain|
|工具协议|JSON Schema Function Tool \+ MCP|内部 RPC|MCP 用于标准化和跨应用复用|
|长流程编排|Temporal|Camunda、现有 BPM|审批、重试、超时和补偿|
|模型网关|企业统一 Model Gateway|LiteLLM 类网关或自研|多模型路由、配额、降级|
|私有模型推理|vLLM|KServe \+ vLLM、Triton|先简单服务，规模明确后平台化|
|Trace 与评测|OpenTelemetry \+ MLflow|Phoenix、Langfuse|统一 Trace、评测和 Prompt 版本|
|部署|Kubernetes 或现有容器平台|VM 部署|根据现有基础设施决定|

### 7\.1 不推荐作为第一阶段默认组件

- LangChain

- LangGraph

- 全量 LlamaIndex Runtime

- Milvus 与 OpenSearch 双引擎同时起步

- Neo4j 全域知识图谱

- Kubeflow 全套平台

- 多 Agent 自组织网络

- Agent 长期个人记忆

- Coding Agent

- 自动退款或自动生产变更

---

## 8\. 推荐的基础技术组合

适合大多数企业中台的初始组合：

```Plaintext
对象存储
+ PostgreSQL
+ OpenSearch
+ Redis
+ Docling
+ Qwen3-Embedding
+ Qwen3-Reranker
+ FastAPI / Pydantic
+ 自研薄 Agent Runtime
+ MCP Tool Gateway
+ Temporal
+ OpenTelemetry
+ MLflow
+ 企业 IAM
+ 模型网关或 vLLM
```

该组合的主要特点：

- 组件边界明确

- 不依赖 LangChain

- 支持私有化

- 支持混合检索

- 支持 Agent 工具调用

- 支持人工审批

- 支持完整审计

- 可以逐步扩展，而不必第一阶段建设完整 AI 平台

---

## 9\. 安全与权限设计

### 9\.1 ACL 继承

索引中的权限来自源系统，不重新定义一套孤立权限。

用户查询时将以下 Claims 注入检索：

```Plaintext
user_id
department
roles
groups
tenant_id
business_unit
data_clearance
region
```

### 9\.2 脱敏

在解析和 Embedding 前处理：

- 手机号

- 姓名

- 地址

- 身份信息

- 支付信息

- 客户聊天记录

- 密钥和 Token

- 内部代码凭据

### 9\.3 Prompt Injection 防护

检索内容视为不可信数据：

- 文档内容不能修改 System Policy

- 文档中出现的工具调用指令不得执行

- 外部文本不得提升权限

- 工具参数必须重新校验

- 模型不能自行扩大检索范围

### 9\.4 工具安全

每个工具声明：

- 输入 Schema

- 输出 Schema

- 所需权限

- 数据范围

- 风险等级

- 是否需要审批

- 是否支持幂等

- 超时

- 重试规则

- 审计字段

---

## 10\. 可观测性与评测

### 10\.1 Trace 模型

一条完整 Trace 应包含：

```Plaintext
Request
  └─ Turn
      ├─ Intent Resolution
      ├─ Query Rewrite
      ├─ ACL Filtering
      ├─ BM25 Retrieval
      ├─ Vector Retrieval
      ├─ Rank Fusion
      ├─ Rerank
      ├─ Context Construction
      ├─ Model Call
      ├─ Tool Call
      ├─ Approval
      └─ Final Response
```

### 10\.2 离线评测指标

#### 检索层

- Recall@K

- Precision@K

- MRR

- nDCG

- ACL 过滤正确率

- 时效文档命中率

- 无效版本误召回率

#### 回答层

- 引用准确率

- 引用完整率

- Groundedness

- Policy Correctness

- 拒答正确率

- 事实一致性

- 格式合规率

#### Agent 层

- 工具选择正确率

- 工具参数正确率

- 无效工具调用率

- 平均工具调用次数

- Agent Loop 完成率

- 人工审批触发准确率

- 高风险动作拦截率

### 10\.3 业务指标

#### 售后

- 平均处理时长

- 首次解决率

- 转人工或升级率

- 客服建议采纳率

- 错误政策引用率

- 工单分类准确率

#### 研发

- 平均资料查找时间

- API 和架构问题一次命中率

- CI 问题定位时间

- 重复问题比例

- 故障复盘知识复用率

- 无引用回答比例

---

## 11\. 分阶段建设路径

### 阶段一：知识基础能力

完成：

- 数据源盘点

- ACL 模型

- 文档解析

- OpenSearch 混合检索

- 引用

- 基础评测集

- 企业知识搜索门户

退出条件：

- 权限过滤无已知穿透

- 核心问题检索质量达到业务基线

- 答案能够稳定返回来源和版本

### 阶段二：售后 Copilot

完成：

- 售后政策检索

- 订单和物流只读工具

- 工单摘要

- 相似案例

- 回复草稿

- 合规检查

退出条件：

- 客服可稳定使用

- 草稿具有可接受采纳率

- 错误政策引用可被评测和追踪

### 阶段三：研发知识助手

完成：

- Wiki、API、ADR 接入

- Git 元数据和 Issue 只读查询

- CI 日志摘要

- 发布和故障知识检索

退出条件：

- 研发问题能够定位到准确文档、版本或系统记录

- 无自动代码修改能力

- 研发数据权限与仓库权限一致

### 阶段四：受控 Agent 应用

完成：

- 复杂售后调查 Agent

- 工单路由

- 审批工作流

- Tool Gateway

- Temporal 长流程

- Agent Trace 和评测

退出条件：

- 所有写工具具有审批和审计

- Agent Loop 有明确步数和停止条件

- 异常流程可回滚或人工接管

### 阶段五：规模化与平台化

根据数据证明再引入：

- 独立 Milvus

- Neo4j

- KServe

- 多模型推理集群

- 多地域容灾

- 更复杂的跨域 Agent 协作

---

## 12\. 最终建议

本项目的核心不是建设一个“什么都能做的企业 Agent”，而是建设三个稳定底座：

1. **可信知识底座**

    - 原文、版本、权限、Owner、新鲜度和引用完整。

2. **可组合能力底座**

    - 检索、模型、工具、审批和工作流均具有明确接口。

3. **可治理 Agent Runtime**

    - 有界循环、结构化事件、工具授权、人工审批、完整追踪。

售后侧可以逐步从知识问答发展到工单辅助和有限动作；研发侧维持“搜索、关联、摘要、调查和导航”的轻量定位，不进入代码自动修改和自主开发。

最终架构应表现为：

> RAG 负责提供可信证据，Workflow 负责保证业务确定性，Agent 只负责处理少量无法提前写死的动态决策。

---

## 13\. 方案来源说明

本方案基于以下方面的共同结论：

- 企业知识库需要从单一向量检索升级为数据治理、混合检索、权限审计、模型服务和评测一体化体系。

- 售后和研发可以共享平台，但必须采用分域索引、独立排序和独立权限策略。

- Hybrid Search 应作为生产基线。

- PageIndex 更适合长文档内部定位，而不是替代企业全局检索。

- 高风险动作应保留人工审批。

- 应优先建设知识质量、权限、引用和评测，再逐步扩大 Agent 自动化范围。



