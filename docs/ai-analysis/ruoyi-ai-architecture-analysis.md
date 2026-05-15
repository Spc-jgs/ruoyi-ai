# RuoYi-AI 项目摸底分析报告（Tag 3.0.0）

> 分析时间：2026-05-08
> 分析方式：只读摸底，未修改任何代码
> 项目版本：3.0.0（tag 3.0.0）
> 分析人：AI 架构分析师

---

## 1. 项目整体定位

RuoYi-AI 是一个**企业级AI助手平台**，是多个能力的组合体：

| 定位 | 说明 |
|------|------|
| 后台管理系统 | 继承 RuoYi-Vue-Plus 的完整 RBAC 权限体系、菜单/用户/角色/部门管理 |
| AI 应用平台 | 基于 LangChain4j 构建的多模型对话、知识库RAG、多智能体协同（Supervisor模式） |
| 知识库系统 | 支持文档上传→解析→切片→向量化→检索的完整RAG流水线 |
| 流程编排平台 | 可视化工作流设计器（ruoyi-aiflow），支持模型调用、邮件发送、人工审核等节点 |
| MCP工具市场 | 支持MCP协议工具的注册、发现、加载和管理 |

**核心定义**：`全栈式AI开发平台`（来自 `pom.xml` 第13行 `<description>全栈式AI开发平台</description>`）

**前端三端架构**：
- 管理后台：ruoyi-admin（Vue 3 + Vben Admin）
- 用户前端：ruoyi-web（Vue 3 + element-plus-x）
- 后端API：ruoyi-ai（Spring Boot 单体）

**开源协议**：MIT

---

## 2. 技术栈总览

| 技术名称 | 作用 | 在项目中的位置 | 需要重点学习 |
|---------|------|-------------|------------|
| Spring Boot 3.5.8 | 基础框架 | 根 `pom.xml` L17 | 必须马上懂 |
| Java 17 | 运行环境 | 根 `pom.xml` L20 | 必须马上懂 |
| LangChain4j 1.13.0 | AI 应用开发框架 | `ruoyi-chat/pom.xml` L33-36 | 必须马上懂 |
| LangChain4j Community 1.13.0-beta23 | 社区模型适配（通义/智谱/Ollama等） | `ruoyi-chat/pom.xml` L46-60 | 必须马上懂 |
| Langgraph4j 1.5.3 | AI流程编排 | 根 `pom.xml` L59 | 边做边学 |
| Sa-Token 1.44.0 + JWT | 权限认证 | `ruoyi-common-satoken` | 必须马上懂 |
| MyBatis-Plus 3.5.14 | ORM框架 | 根 `pom.xml` L27 | 必须马上懂 |
| Milvus / Weaviate / Qdrant | 向量数据库 | `ruoyi-chat/pom.xml` L82-98 | 必须马上懂 |
| Apache Tika | 文档解析（PDF/Word/Excel） | `ruoyi-chat/pom.xml` L69-79 | 边做边学 |
| MCP协议 | 工具调用协议 | `ruoyi-chat/pom.xml` L100-104 | 边做边学 |
| Redis + Redisson | 缓存/分布式锁 | `ruoyi-common-redis` | 必须马上懂 |
| MySQL 8.0 | 关系型数据库 | `application-dev.yml` L61 | 必须马上懂 |
| MinIO | 对象存储 | `ruoyi-common-oss` | 边做边学 |
| Warm-Flow | 国产工作流引擎 | 根 `pom.xml` L51 | 后期再学 |
| SnailJob | 分布式任务调度 | 根 `pom.xml` L34 | 后期再学 |
| Neo4j Driver | 知识图谱存储 | `ruoyi-chat/pom.xml` L148-150 | 边做边学 |
| Dify Java Client | Dify平台集成 | `ruoyi-chat/pom.xml` L152-156 | 后期再学 |
| SSE | 流式推送 | `ruoyi-common-sse` | 必须马上懂 |
| Hutool 5.8.40 | 工具库 | 根 `pom.xml` L29 | 边做边学 |
| MapStruct-Plus 1.5.0 | 对象映射 | 根 `pom.xml` L35 | 边做边学 |
| Knife4j 4.4.0 | API文档增强 | 根 `pom.xml` L70 | 边做边学 |

---

## 3. 模块结构说明

### 3.1 顶级模块

| 模块名 | 主要功能 | 关键包路径 | 是否建议二开时修改 |
|-------|---------|----------|-----------------|
| ruoyi-admin | Web服务入口、启动类 | `org.ruoyi` | 仅修改配置，不动核心 |
| ruoyi-common | 通用组件集合（24个子模块） | `org.ruoyi.common.*` | 按需扩展，不建议改原有代码 |
| ruoyi-modules | 业务模块集合（5个子模块） | `org.ruoyi.*` | **重点二开区域** |
| ruoyi-extend | 监控和调度扩展 | `org.ruoyi.*` | 不建议修改 |

### 3.2 ruoyi-modules 子模块

| 模块名 | 主要功能 | 关键包路径 | 关键类 | 是否建议二开时修改 |
|-------|---------|----------|-------|-----------------|
| ruoyi-chat | **AI核心模块**：对话、知识库、向量库、MCP | `org.ruoyi.service.chat`, `org.ruoyi.service.knowledge`, `org.ruoyi.service.vector` | `ChatServiceFacade`, `ChatServiceFactory`, `KnowledgeAttachServiceImpl`, `VectorStoreServiceImpl` | **重点修改** |
| ruoyi-aiflow | AI流程编排模块 | `org.ruoyi.workflow` | 工作流设计器相关类 | 按需扩展 |
| ruoyi-system | 系统管理：用户/角色/菜单/部门 | `org.ruoyi.system` | `SysUserController`, `SysRoleController` | 仅扩展，不改原有 |
| ruoyi-generator | 代码生成器 | `org.ruoyi.generator` | 代码生成模板 | 工具使用，一般不改 |
| ruoyi-workflow | 审批工作流（Warm-Flow） | `org.ruoyi.workflow` | 审批流程相关 | 按需使用 |

### 3.3 ruoyi-common 子模块（AI相关重点标注）

| 模块名 | 主要功能 | 是否建议二开时修改 |
|-------|---------|-----------------|
| **ruoyi-common-chat** | **Chat服务公共层**：实体、DTO、VO、枚举、Service接口 | 可扩展接口，不改原有 |
| ruoyi-common-sse | **SSE流式推送**：SseEmitter管理、消息工具 | 可复用，一般不改 |
| ruoyi-common-satoken | Sa-Token认证配置 | 不建议改 |
| ruoyi-common-security | 安全拦截配置 | 不建议改 |
| ruoyi-common-oss | 对象存储 | 可复用 |
| ruoyi-common-tenant | 多租户 | 可复用，需理解 |
| ruoyi-common-core | 核心工具类 | 不建议改 |
| ruoyi-common-mybatis | MyBatis-Plus配置 | 不建议改 |
| ruoyi-common-redis | Redis配置 | 不建议改 |
| 其他15个模块 | 日志/邮件/短信/加密/脱敏等 | 按需使用 |

---

## 4. 启动流程分析

### 4.1 启动哪个模块
启动 `ruoyi-admin` 模块

### 4.2 启动类在哪里
`ruoyi-admin/src/main/java/org/ruoyi/RuoYiAIApplication.java`
- 主类：`RuoYiAIApplication`
- 默认端口：6039
- 启动时自动检测并终止占用6039端口的进程（仅Windows）

### 4.3 依赖哪些中间件

| 中间件 | 用途 | 必须启动 | 配置位置 |
|-------|------|---------|---------|
| MySQL 8.0 | 主数据库 | 是 | `application-dev.yml` L61 |
| Redis | 缓存/Token存储 | 是 | `application-dev.yml` L95-108 |
| Milvus/Weaviate/Qdrant | 向量数据库 | 至少一个 | `application.yml` L294-312，默认Milvus |
| MinIO | 对象存储 | 建议启动 | `ruoyi-common-oss` |
| Neo4j | 知识图谱 | 可选 | `application.yml` L59-61 禁用了自动配置 |

### 4.4 需要哪些数据库表
- **RuoYi基础表**：sys_user, sys_role, sys_menu, sys_dept, sys_user_role, sys_role_menu 等（约60+张表）
- **AI相关表**：chat_message, chat_model, chat_provider, chat_session, knowledge_info, knowledge_attach, knowledge_fragment, knowledge_graph_instance, knowledge_graph_segment, mcp_market_info, mcp_market_tool, mcp_tool_info
- **工作流表**：flow_category, flow_definition, flow_node 等

SQL初始化脚本：`docs/script/sql/ruoyi-ai-v3_mysql8.sql`（3482行，数据库名 `ruoyi-ai`）

### 4.5 需要哪些配置项

| 配置项 | 说明 | 默认值 | 位置 |
|-------|------|-------|------|
| `spring.profiles.active` | 环境标识 | dev | `application.yml` L78 |
| `spring.datasource.dynamic.datasource.master.*` | MySQL连接 | 127.0.0.1:3306/ruoyi-ai | `application-dev.yml` L54-63 |
| `spring.data.redis.*` | Redis连接 | localhost:6379 | `application-dev.yml` L95-108 |
| `vector-store.type` | 向量库类型 | milvus | `application.yml` L297 |
| `vector-store.milvus.*` | Milvus配置 | localhost:19530 | `application.yml` L304-306 |
| `sa-token.jwt-secret-key` | JWT密钥 | abcdefghijklmnopqrstuvwxyz | `application.yml` L112 |
| `tenant.enable` | 多租户开关 | true | `application.yml` L131 |
| `sse.enabled` | SSE推送开关 | true | `application.yml` L251 |

---

## 5. RuoYi 基础架构分析

### 5.1 登录认证
- 使用 **Sa-Token + JWT** 双重机制
- 认证入口：`ruoyi-common-satoken` 模块
- Token名称：`Authorization`（`application.yml` L106）
- 登录辅助类：`org.ruoyi.common.satoken.utils.LoginHelper`
  - `LoginHelper.login()` 执行登录
  - `LoginHelper.getUserId()` 获取当前用户ID
  - `LoginHelper.getTenantId()` 获取租户ID
  - `LoginHelper.isSuperAdmin()` 判断是否超管
- 支持并发登录（`is-concurrent: true`）
- 每次登录生成新Token（`is-share: false`）

### 5.2 权限
- 拦截器配置：`org.ruoyi.common.security.config.SecurityConfig`
- 使用 `SaInterceptor` 拦截所有路径，排除静态资源和API文档
- 权限实现：`org.ruoyi.common.satoken.core.service.SaPermissionImpl`
- SSE路径自动排除认证（`SecurityConfig` L92）
- 支持客户端ID校验（clientid）

### 5.3 菜单
- 表：`sys_menu`
- 实体类在 `ruoyi-system` 模块
- 支持多级菜单和权限标识
- 菜单与角色关联：`sys_role_menu`

### 5.4 用户/角色/部门
- 用户：`sys_user` → `SysUser`
- 角色：`sys_role` → `SysRole`
- 部门：`sys_dept` → `SysDept`
- 用户-角色：`sys_user_role`
- 角色-部门：`sys_role_dept`
- 用户-岗位：`sys_user_post`
- **多租户**：`sys_tenant` → 支持租户隔离（`tenant.enable: true`）

### 5.5 Controller / Service / Mapper 分层规范
- **Controller**：接收请求，调用Service，返回 `R<T>` 统一响应
- **Service**：接口 + Impl 实现，业务逻辑
- **Mapper**：继承 `BaseMapperPlus`，支持 Vo 查询
- **Entity/Bo/Vo** 三层：
  - Entity：数据库映射（`@TableName`）
  - Bo：业务输入对象
  - Vo：业务输出对象
  - 使用 Mapstruct-Plus 做 Bo↔Entity 转换

### 5.6 统一返回格式
- 使用 `R<T>` 泛型封装
- 含 code、msg、data 字段

### 5.7 异常处理
- 全局异常处理器在 `ruoyi-common-core`
- 业务异常：`ServiceException`
- Sa-Token异常处理：`SaTokenExceptionHandler`

### 5.8 字典/参数配置
- 字典表：`sys_dict_type`, `sys_dict_data`
- 参数配置表：`sys_config`
- 均在 `ruoyi-system` 模块管理

### 5.9 代码生成器
- 存在：`ruoyi-generator` 模块
- 基于 Velocity 模板引擎
- 可自动生成 Entity/Mapper/Service/Controller

---

## 6. AI 能力分析

### 6.1 大模型调用入口
- **核心入口**：`ChatServiceFacade.sseChat()`（`ruoyi-modules/ruoyi-chat/src/main/java/org/ruoyi/service/chat/impl/ChatServiceFacade.java`）
- **对外接口**：`ChatController.sseChat()` → `POST /chat/send`
- 调用链路：`ChatController` → `ChatServiceFacade` → `ChatServiceFactory.getOriginalService(providerCode)` → 具体Provider → `StreamingChatModel.chat()`

### 6.2 模型配置
- 表：`chat_model`，实体：`ChatModel`（`ruoyi-common-chat/.../entity/chat/ChatModel.java`）
- 关键字段：`model_name`（模型名）, `provider_code`（供应商编码）, `category`（分类chat/vector/reranker等）, `api_host`, `api_key`, `model_dimension`
- 通过 `IChatModelService.selectModelByName()` 查询模型配置

### 6.3 是否支持多模型
**支持**，当前已接入6个Provider：

| Provider | 编码 | 实现类 |
|----------|------|--------|
| OpenAI | openai | `OpenAIServiceImpl` |
| 通义千问 | qianwen | `QianWenChatServiceImpl` |
| 智谱AI | zhipu | `ZhiPuChatServiceImpl` |
| DeepSeek | deepseek | `DeepseekServiceImpl` |
| Ollama | ollama | `OllamaServiceImpl` |
| PPIO | ppio | `PPIOServiceImpl` |

模型类型枚举 `ModelType`：chat(0), image(1), vector(3), reranker(4), audio(5), text(6), video(7), ppt(8), music(9)

### 6.4 对话接口
- **主接口**：`POST /chat/send` → 返回 `SseEmitter`
- 入参：`ChatRequest`（model, content, sessionId, knowledgeId, enableThinking, enableWorkFlow, isResume 等）
- 出参：SSE流式事件（content/done/error）

### 6.5 Prompt是怎么组织的
**当前为硬编码模式**，在 `ChatServiceFacade.buildContextMessages()` 中：
1. 加载历史消息（从 `PersistentChatMemoryStore` 查数据库）
2. 如果有 `knowledgeId`，从向量库查询相关内容作为上下文
3. 添加当前用户消息

**没有独立的 Prompt 模板管理模块**，system prompt 未在代码中找到配置化机制。

### 6.6 是否有知识库
**有**，完整知识库管理：
- 知识库信息：`knowledge_info` 表 → `KnowledgeInfo` 实体
- 知识库附件：`knowledge_attach` 表 → `KnowledgeAttach` 实体
- 知识片段：`knowledge_fragment` 表 → `KnowledgeFragment` 实体
- 知识图谱：`knowledge_graph_instance` / `knowledge_graph_segment` 表
- Controller：`KnowledgeInfoController`, `KnowledgeAttachController`, `KnowledgeFragmentController`, `KnowledgeGraphInstanceController`

### 6.7 是否有RAG
**有**，在 `ChatServiceFacade.buildContextMessages()` 中实现：
1. 根据 `knowledgeId` 查询知识库配置（`KnowledgeInfoVo`）
2. 查询对应的 Embedding 模型配置（`ChatModelVo`）
3. 构建 `QueryVectorBo` 调用 `VectorStoreService.getQueryVector()`
4. 向量检索结果作为 `AiMessage` 注入上下文

**注意**：当前RAG实现较简单，**没有** Rerank 重排序步骤（虽然 `ModelType` 中有 RERANKER 类型定义，但 `ChatServiceFacade` 中未使用）。

### 6.8 Embedding在哪里
- 工厂类：`EmbeddingModelFactory`（`ruoyi-modules/ruoyi-chat/src/main/java/org/ruoyi/factory/EmbeddingModelFactory.java`）
- 基类：`BaseEmbedModelService`
- 多模态：`MultiModalEmbedModelService`
- 缓存机制：`ConcurrentHashMap` 缓存已创建的模型实例
- 通过 `IChatModelService.selectModelByName()` 查找 embedding 模型配置

### 6.9 向量库怎么接入
- **策略模式**：`VectorStoreStrategyFactory`（`ruoyi-modules/ruoyi-chat/src/main/java/org/ruoyi/factory/VectorStoreStrategyFactory.java`）
- 配置：`VectorStoreProperties`（`vector-store.type` 决定使用哪个向量库）
- 三种实现：
  - `WeaviateVectorStoreStrategy`（langchain4j-weaviate）
  - `MilvusVectorStoreStrategy`（langchain4j-milvus）
  - `QdrantVectorStoreStrategy`（langchain4j-qdrant）
- 默认配置：`vector-store.type: milvus`
- 代理层：`VectorStoreServiceImpl` 通过 `getCurrentStrategy()` 委托给具体策略

### 6.10 文件上传后怎么解析
- 入口：`KnowledgeAttachServiceImpl.upload()`（`ruoyi-modules/ruoyi-chat/src/main/java/org/ruoyi/service/knowledge/impl/KnowledgeAttachServiceImpl.java` L163-219）
- 流程：
  1. `OssService.uploadFile()` 上传到对象存储
  2. `ResourceLoaderFactory.getLoaderByFileType()` 根据文件类型选择解析器
  3. `ResourceLoader.getContent()` 解析文档内容
  4. `ResourceLoader.getChunkList()` 文档切片
  5. 切片结果存入 `knowledge_fragment` 表
  6. 调用 `VectorStoreService.storeEmbeddings()` 向量化并存储

### 6.11 文档怎么切片
- 通过 `ResourceLoaderFactory` 根据文件类型分发到不同解析器：
  - `TextFileLoader` + `CharacterTextSplitter`
  - `WordLoader` + `CharacterTextSplitter`
  - `PdfFileLoader` + `CharacterTextSplitter`
  - `MarkDownFileLoader` + `MarkdownTextSplitter`
  - `CodeFileLoader` + `CodeTextSplitter`
  - `ExcelFileLoader` + `ExcelTextSplitter`
- 切片参数在 `KnowledgeInfo` 中配置：`separator`（分隔符）、`overlapChar`（重叠字符数）、`textBlockSize`（文本块大小）

### 6.12 检索怎么做
- `VectorStoreService.getQueryVector(QueryVectorBo)` 执行向量检索
- 输入：用户问题 + 知识库ID + Embedding模型信息
- 输出：`List<String>` 相关文本片段列表
- 当前是**纯向量检索**，未实现混合检索（关键词+向量）和Rerank

### 6.13 回答怎么生成
- `ChatServiceFacade.sseChat()` 中：
  1. 构建上下文消息（历史 + 知识库检索结果 + 当前问题）
  2. 通过 `ChatServiceFactory` 路由到对应Provider
  3. `AbstractChatService.buildStreamingChatModel()` 构建 `StreamingChatModel`
  4. `streamingChatModel.chat(contextMessages, handler)` 发起流式调用
  5. `StreamingChatResponseHandler` 实时将片段通过 `SseMessageUtils.sendContent()` 推送前端
  6. 完成后保存消息到 `chat_message` 表

### 6.14 多智能体（Supervisor模式）
- 实现位置：`ChatServiceFacade.handleThinkingMode()`（L212-336）
- 使用 `langchain4j-agentic` 的 `SupervisorAgent`
- 子Agent列表：
  - `WebSearchAgent`：Web搜索（通过MCP的Playwright/Bing客户端）
  - `SkillsAgent`：文档处理技能（基于Skills的docx/pdf/xlsx处理）
  - `SqlAgent`：数据库查询（QueryAllTablesTool, QueryTableSchemaTool, ExecuteSqlQueryTool）
  - `ChartGenerationAgent`：图表生成
  - `EchartsAgent`：ECharts数据可视化
- 通过 `enableThinking=true` 激活

### 6.15 MCP工具管理
- Controller：`McpMarketController`, `McpToolController`
- Service：`McpMarketServiceImpl`, `McpToolServiceImpl`
- 表：`mcp_market_info`, `mcp_market_tool`, `mcp_tool_info`
- 内置工具：edit_file, list_directory, read_file, query_all_tables, execute_sql_query, query_table_schema
- 支持远程MCP市场加载

---

## 7. 数据库表分析（AI相关）

| 表名 | 作用 | 关键字段 | 对应Java实体 | 对应Mapper | 二开风险 |
|------|------|---------|------------|-----------|---------|
| `chat_message` | 聊天消息记录 | session_id, user_id, content, role, model_name | `ChatMessage`（在common-chat） | `ChatMessageMapper` | 低，可直接扩展 |
| `chat_model` | 模型配置管理 | model_name, provider_code, category, api_host, api_key, model_dimension | `ChatModel` | `ChatModelMapper` | 中，需加字段支持更多模型参数 |
| `chat_provider` | 厂商管理 | provider_code, provider_name, api_host | `ChatProvider` | `ChatProviderMapper` | 低 |
| `chat_session` | 会话管理 | user_id, session_title, conversation_id | `ChatSession` | `ChatSessionMapper` | 中，需扩展支持律师端/用户端隔离 |
| `knowledge_info` | 知识库信息 | name, separator, overlap_char, retrieve_limit, text_block_size, vector_model, embedding_model | `KnowledgeInfo` | `KnowledgeInfoMapper` | 中，需加类型字段（法条/案例/合同） |
| `knowledge_attach` | 知识库附件 | knowledge_id, doc_id, name, type, oss_id | `KnowledgeAttach` | `KnowledgeAttachMapper` | 低 |
| `knowledge_fragment` | 知识片段 | doc_id, idx, content | `KnowledgeFragment` | `KnowledgeFragmentMapper` | 低 |
| `knowledge_graph_instance` | 知识图谱实例 | graph_uuid, knowledge_id, graph_name, graph_status, node_count, model_name, entity_types, relation_types | `KnowledgeGraphInstance` | `KnowledgeGraphInstanceMapper` | 中 |
| `knowledge_graph_segment` | 知识图谱片段 | 关联图谱和知识库 | `KnowledgeGraphSegment` | `KnowledgeGraphSegmentMapper` | 中 |
| `mcp_market_info` | MCP市场 | name, url, status | `McpMarket` | `McpMarketMapper` | 低 |
| `mcp_market_tool` | MCP市场工具 | market_id, tool_name, tool_metadata, is_loaded | `McpMarketTool` | `McpMarketToolMapper` | 低 |
| `mcp_tool_info` | MCP工具 | name, type(LOCAL/REMOTE/BUILTIN), config_json | `McpTool` | `McpToolMapper` | 低 |

---

## 8. 接口分析

### 8.1 AI对话接口

| 接口路径 | Controller | 功能 | 入参 | 出参 | 前端可复用 | AI律师需改造 |
|---------|-----------|------|------|------|----------|------------|
| `POST /chat/send` | `ChatController` | AI对话（SSE流式） | `ChatRequest`（model, content, sessionId, knowledgeId, enableThinking等） | `SseEmitter` | 是 | 需加Prompt模板参数 |
| `GET/POST /chat/session/*` | `ChatSessionController` | 会话CRUD | `ChatSessionBo` | `R<ChatSessionVo>` | 是 | 需扩展会话类型 |
| `GET/POST /chat/message/*` | `ChatMessageController` | 消息查询 | 分页查询参数 | `TableDataInfo<ChatMessageVo>` | 是 | 直接复用 |
| `GET/POST /chat/model/*` | `ChatModelController` | 模型配置CRUD | `ChatModelBo` | `R<ChatModelVo>` | 是 | 需加领域Prompt配置 |
| `GET/POST /chat/provider/*` | `ChatProviderController` | 厂商CRUD | `ChatProviderBo` | `R<ChatProviderVo>` | 是 | 直接复用 |

### 8.2 知识库接口

| 接口路径 | Controller | 功能 | 入参 | 出参 | 前端可复用 | AI律师需改造 |
|---------|-----------|------|------|------|----------|------------|
| `POST /knowledge/info/*` | `KnowledgeInfoController` | 知识库CRUD | `KnowledgeInfoBo` | `R<KnowledgeInfoVo>` | 是 | 需加知识库分类 |
| `POST /knowledge/attach/*` | `KnowledgeAttachController` | 附件上传/管理 | `KnowledgeInfoUploadBo`（含MultipartFile） | `R<Void>` | 是 | 直接复用 |
| `POST /knowledge/fragment/*` | `KnowledgeFragmentController` | 片段管理 | `KnowledgeFragmentBo` | `R<KnowledgeFragmentVo>` | 是 | 直接复用 |
| `POST /knowledge/graph/*` | `KnowledgeGraphInstanceController` | 知识图谱管理 | `KnowledgeGraphInstanceBo` | `R<KnowledgeGraphInstanceVo>` | 是 | 按需使用 |

### 8.3 MCP接口

| 接口路径 | Controller | 功能 | 入参 | 出参 | 前端可复用 | AI律师需改造 |
|---------|-----------|------|------|------|----------|------------|
| `POST /mcp/market/*` | `McpMarketController` | MCP市场管理 | `McpMarketBo` | `R<McpMarketVo>` | 是 | 后期考虑 |
| `POST /mcp/tool/*` | `McpToolController` | MCP工具管理 | `McpToolBo` | `R<McpToolVo>` | 是 | 后期考虑 |

---

## 9. 如果做 AI 律师，哪些能复用

### 9.1 可直接复用

| 能力 | 依据 | 说明 |
|------|------|------|
| 多模型对话框架 | `ChatServiceFacade` + `ChatServiceFactory` + 6个Provider | 完整的对话链路，直接用 |
| SSE流式输出 | `SseEmitterManager` + `SseMessageUtils` | 前后端对流式输出的基础设施 |
| 会话管理 | `ChatSession` + `ChatMessage` + `PersistentChatMemoryStore` | 完整的会话持久化 |
| 知识库文档上传解析 | `KnowledgeAttachServiceImpl.upload()` + `ResourceLoaderFactory` | 支持6种文件格式 |
| 向量化存储 | `VectorStoreServiceImpl` + 3种向量库策略 | 文档→切片→向量化→存储 |
| 向量检索 | `VectorStoreService.getQueryVector()` | 基础语义检索 |
| RBAC权限体系 | Sa-Token + JWT + 用户/角色/菜单/部门 | 完整的权限隔离 |
| 多租户 | `ruoyi-common-tenant` | 天然支持租户隔离 |
| 对象存储 | `ruoyi-common-oss` + MinIO | 文件上传下载 |
| 代码生成器 | `ruoyi-generator` | 快速生成CRUD代码 |

### 9.2 需要轻微改造

| 能力 | 当前状态 | 改造方向 |
|------|---------|---------|
| RAG检索 | 纯向量检索，无Rerank | 加入关键词检索+Rerank重排序，提升法律文档检索精度 |
| 知识库分类 | 无分类字段 | 在 `knowledge_info` 加 category 字段（法条/案例/合同/裁判文书） |
| Prompt管理 | 硬编码 | 新增 `prompt_template` 表，支持法律场景Prompt模板 |
| 会话类型 | 单一会话 | 扩展 `chat_session` 加 session_type（AI咨询/律师对话/案件讨论） |
| 模型配置 | 基础配置 | 增加 temperature、top_p 等参数支持 |

### 9.3 需要重写

| 能力 | 原因 |
|------|------|
| 人-人即时通讯 | 当前只有AI对话，没有律师与用户的即时通讯通道 |
| 人工律师介入 | 需要新的"转人工"机制，AI回答不满意时转接律师 |
| 法律领域专属Agent | 当前Agent是通用型（SQL/图表/搜索），需要法律专用Agent（法条查询/案例检索/合同审查） |
| 用户端/律师端双入口 | 当前只有管理端+用户端，需要新增律师端 |
| 案件管理 | 全新业务模块 |

### 9.4 不建议使用

| 能力 | 原因 |
|------|------|
| 思考模式中的MCP硬编码 | `ChatServiceFacade.handleThinkingMode()` 中硬编码了Windows路径和npx命令，不适合生产 |
| Dify集成 | 当前为可选项，AI律师场景建议自建而非依赖第三方 |
| 图谱构建（knowledge_graph） | 当前状态较新，稳定性未知，建议后期评估 |

---

## 10. 二开建议路线

### P0：先跑起来
- **目标**：项目能在本地启动，完成一次AI对话
- **要改哪些模块**：仅修改配置文件
- **要看哪些类**：`RuoYiAIApplication`, `application-dev.yml`, `application.yml`
- **要准备哪些配置**：
  1. MySQL 8.0 创建 `ruoyi-ai` 数据库并导入 `docs/script/sql/ruoyi-ai-v3_mysql8.sql`
  2. Redis 启动
  3. Milvus/Weaviate/Qdrant 至少启动一个（推荐用Docker）
  4. 在 `chat_model` 表中配置一个可用模型（如DeepSeek的API Key）
  5. 修改 `application-dev.yml` 中数据库和Redis连接信息
- **风险点**：向量库连接失败不影响启动但知识库功能不可用
- **验收标准**：访问 `http://localhost:6039` 通过Swagger调用 `POST /chat/send` 成功获得AI回复

### P1：看懂架构
- **目标**：理解项目整体架构和AI模块调用链路
- **要改哪些模块**：不改代码
- **要看哪些类**：
  - `ChatServiceFacade` → 理解对话全流程
  - `ChatServiceFactory` → 理解多模型路由
  - `OpenAIServiceImpl` → 理解Provider模式
  - `KnowledgeAttachServiceImpl` → 理解知识库上传流程
  - `VectorStoreServiceImpl` → 理解向量存储策略
  - `VectorStoreStrategyFactory` → 理解向量库切换
  - `ChatRequest` → 理解请求参数
  - `SseMessageUtils` → 理解SSE推送
- **要准备哪些配置**：无
- **风险点**：无
- **验收标准**：能画出完整的对话链路图和知识库RAG流程图

### P2：接入一个模型
- **目标**：配置并调通一个具体的AI模型
- **要改哪些模块**：仅修改数据库 `chat_model` 和 `chat_provider` 表
- **要看哪些类**：`ChatModelServiceImpl`, `ChatProviderServiceImpl`, 具体Provider实现
- **要准备哪些配置**：
  1. 获取一个模型API Key（推荐DeepSeek，性价比高）
  2. 在 `chat_provider` 表中确认厂商记录存在
  3. 在 `chat_model` 表中添加模型记录，确保 `provider_code` 与枚举 `ChatModeType` 一致
- **风险点**：API Key泄露；模型名称与Provider不匹配
- **验收标准**：通过前端或Swagger成功与指定模型对话

### P3：跑通知识库问答
- **目标**：上传文档→解析→向量化→检索→回答
- **要改哪些模块**：不改代码，仅配置
- **要看哪些类**：`KnowledgeAttachServiceImpl`, `VectorStoreServiceImpl`, `ResourceLoaderFactory`
- **要准备哪些配置**：
  1. 向量库正常运行（Milvus推荐）
  2. 配置Embedding模型（如 `baai/bge-m3`）
  3. 在 `chat_model` 中添加vector类型的模型记录
  4. 创建知识库并上传测试文档
- **风险点**：Embedding模型维度需与向量库Collection配置一致
- **验收标准**：创建知识库→上传PDF→在对话中选择该知识库→AI回答基于文档内容

### P4：改造成 AI 律师问答
- **目标**：具备法律领域的AI问答能力
- **要改哪些模块**：
  - `ruoyi-chat`：新增Prompt模板管理、法律领域RAG优化
  - `ruoyi-common-chat`：扩展 `ChatRequest` 增加promptTemplateId
- **要看哪些类**：`ChatServiceFacade.buildContextMessages()`, `KnowledgeInfo`
- **要准备哪些配置**：
  1. 设计法律知识库分类（法条/案例/合同/裁判文书）
  2. 准备法律领域Prompt模板
  3. 整理法律文档数据集
- **风险点**：法律回答准确性；RAG检索精度不够
- **验收标准**：AI能基于法律知识库给出相对准确的法律咨询回答

### P5：增加律师端
- **目标**：律师可以登录、管理知识库、查看历史咨询
- **要改哪些模块**：
  - 新增律师端前端项目（或复用管理端改造）
  - `ruoyi-system`：新增律师角色和菜单权限
  - `ruoyi-chat`：新增律师专属功能接口
- **要看哪些类**：`SysRoleController`, `SysMenuController`, `LoginHelper`
- **要准备哪些配置**：
  1. 设计律师角色权限矩阵
  2. 规划律师端菜单结构
- **风险点**：与现有权限体系冲突
- **验收标准**：律师账号登录后看到专属界面，能管理知识库和查看咨询记录

### P6：增加用户端
- **目标**：用户能注册、咨询AI律师、转人工律师
- **要改哪些模块**：
  - 用户端前端项目（ruoyi-web 改造）
  - `ruoyi-chat`：新增转人工机制、用户咨询接口
  - 新增即时通讯模块
- **要看哪些类**：`ChatController`, `ChatSessionController`
- **要准备哪些配置**：
  1. WebSocket/SSE 即时通讯方案
  2. 转人工的路由策略
- **风险点**：即时通讯复杂度高；消息安全性
- **验收标准**：用户能发起咨询→AI回答→不满意→转人工律师→律师回复

### P7：生产化改造
- **目标**：可上线运营
- **要改哪些模块**：所有模块
- **要看哪些类**：全部AI核心类
- **要准备哪些配置**：
  1. HTTPS证书
  2. 生产环境数据库/Redis/向量库集群
  3. 模型API高可用方案
  4. 消息审计日志
  5. 内容安全审查
- **风险点**：法律合规性；AI回答责任的界定；数据安全
- **验收标准**：通过安全审计，支持100+并发，AI回答合规率>95%

---

## 11. 我作为 Java 后端，需要优先学习什么

### 11.1 必须马上懂

| 内容 | 学习路径 | 项目中的位置 |
|------|---------|------------|
| RuoYi-Vue-Plus 架构 | 官方文档 + 项目代码 | `ruoyi-admin`, `ruoyi-system`, `ruoyi-common` |
| Sa-Token认证 | Sa-Token官方文档 | `ruoyi-common-satoken`, `ruoyi-common-security` |
| LangChain4j 基础 | LangChain4j官方文档 + 示例 | `ruoyi-chat` 全模块 |
| SSE流式输出 | Spring SSE + 项目中实现 | `ruoyi-common-sse`, `ChatServiceFacade` |
| 向量数据库概念 | Milvus/Weaviate文档 | `VectorStoreStrategyFactory`, `MilvusVectorStoreStrategy` |

### 11.2 可以边做边学

| 内容 | 学习路径 | 项目中的位置 |
|------|---------|------------|
| RAG全流程 | 理论 + 项目中走通 | `KnowledgeAttachServiceImpl`, `ChatServiceFacade.buildContextMessages()` |
| Embedding模型 | 理解文本向量化原理 | `EmbeddingModelFactory`, `BaseEmbedModelService` |
| Prompt Engineering | 实践中优化 | `ChatServiceFacade`（当前硬编码，需要改进） |
| 文档解析（Tika） | Apache Tika文档 | `ResourceLoaderFactory`, 各种Loader |
| MCP协议 | MCP规范文档 | `McpToolServiceImpl`, `LangChain4jMcpToolProviderService` |
| 多智能体（Supervisor） | LangChain4j Agentic文档 | `ChatServiceFacade.handleThinkingMode()` |
| 知识图谱 | Neo4j + LightRAG | `KnowledgeGraphInstanceServiceImpl` |

### 11.3 后期再学

| 内容 | 说明 |
|------|------|
| 多租户深入 | `ruoyi-common-tenant`，基本机制已理解即可 |
| Warm-Flow工作流 | 审批流引擎，与AI律师核心业务关联不大 |
| 分布式任务调度 | SnailJob，生产化时再学 |
| Dify平台集成 | 第三方AI平台集成，按需 |
| 文生图/文生视频 | 非法律场景核心需求 |
| 代码生成器自定义 | 提效工具，不急 |

---

## 12. 风险点

### 12.1 项目是否适合直接二开
**适合**。项目基于成熟的RuoYi-Vue-Plus框架，AI模块与业务模块分离度较好。MIT协议允许商用。

### 12.2 哪些地方不建议动
- `ruoyi-common-core`：核心工具类，稳定不动
- `ruoyi-common-satoken`：认证体系，牵一发动全身
- `ruoyi-common-security`：安全拦截，改错则全站裸奔
- `ruoyi-system`：系统管理，仅扩展不改原有
- `ruoyi-common-mybatis`：ORM配置，不建议动

### 12.3 哪些地方必须封装
- **Prompt模板管理**：当前硬编码，必须抽象为独立的PromptService
- **RAG检索策略**：当前只有纯向量检索，必须封装为可插拔的检索策略
- **对话模式路由**：`ChatServiceFacade.handleSpecialChatModes()` 中的if-else需重构为策略模式
- **思考模式中的MCP硬编码**：Windows路径和npx命令必须配置化

### 12.4 AI模块是否和业务耦合严重
**中等程度耦合**。
- `ChatServiceFacade` 同时承担了对话路由、知识库检索、SSE推送、消息持久化4个职责，需要拆分
- `KnowledgeAttachServiceImpl.upload()` 方法做了文件上传+解析+切片+向量化，职责过多
- 向量库策略模式设计良好，可独立扩展

### 12.5 是否适合做三端：管理端/律师端/用户端
**适合**，原因：
1. 已有多租户支持（`tenant_id`字段遍布全表）
2. Sa-Token支持多端登录（`is-concurrent: true`）
3. 已有管理端+用户端双前端项目
4. 需新增律师端前端和律师角色权限配置

### 12.6 是否适合商用
**需注意**：
1. **MIT协议**：允许商用，但需保留版权声明
2. **AI回答法律问题**：存在合规风险，必须有免责声明+人工审核
3. **数据安全**：用户咨询涉及隐私，需加密存储
4. **API Key管理**：当前 `chat_model` 表中 api_key 明文存储，需加密
5. **内容审查**：AI输出必须经过内容安全审查

### 12.7 后续维护成本
- **中等**。项目活跃度较高（GitHub star数多），社区支持较好
- LangChain4j 版本更新快，可能有API不兼容风险
- 向量数据库（Milvus/Weaviate）运维有一定成本
- 法律领域知识库需要持续维护和更新

---

## 13. 最终结论

### 13.1 这个项目适不适合做 AI 律师的底座
**适合**。理由：
1. 具备完整的AI对话链路（多模型工厂 + SSE流式 + 消息持久化）
2. 具备完整的知识库RAG链路（上传→解析→切片→向量化→检索→增强对话）
3. 成熟的RBAC权限体系和多租户支持
4. MIT开源协议，可商用
5. 项目结构清晰，模块化程度好，二开门槛适中

### 13.2 适合复用哪些能力
1. **ChatServiceFacade + ChatServiceFactory**：多模型路由框架，核心复用
2. **KnowledgeAttachServiceImpl**：文档上传解析切片全流程
3. **VectorStoreStrategyFactory + 3种向量库策略**：向量存储检索
4. **SSE流式推送体系**：`SseEmitterManager` + `SseMessageUtils`
5. **Sa-Token认证体系**：登录鉴权 + 权限控制
6. **多租户体系**：天然支持数据隔离
7. **对象存储**：MinIO/OSS文件管理
8. **代码生成器**：快速生成新业务模块

### 13.3 不适合依赖哪些能力
1. **思考模式（Supervisor Agent）**：硬编码严重，需重构后才能用于生产
2. **RAG检索精度**：纯向量检索无Rerank，法律场景精度不足
3. **Prompt管理**：无独立管理模块，需新建
4. **即时通讯**：只有AI对话，无人与人聊天能力
5. **知识图谱**：模块较新，稳定性待验证

### 13.4 我下一步应该先做什么
1. **P0**：用Docker一键启动MySQL+Redis+Milvus，导入SQL，配置一个DeepSeek API Key，把项目跑起来
2. 通过Swagger（`http://localhost:6039/doc.html`）调用 `POST /chat/send` 完成一次AI对话
3. 在管理端创建知识库，上传一份法律文档，测试知识库问答
4. 通读 `ChatServiceFacade` 全部代码，画出调用链路图
5. 设计法律领域的Prompt模板和知识库分类方案

---

> 本报告所有判断均基于 tag 3.0.0 版本的实际代码分析，引用了具体的文件路径、类名和方法名。
> 如有信息不足之处，已明确标注"未在当前项目中找到依据"。
