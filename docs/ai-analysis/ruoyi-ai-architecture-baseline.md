# RuoYi-AI 3.0.0 架构基线分析

> 版本：v1.0
> 基线：RuoYi-AI tag 3.0.0
> 日期：2026-05-08
> 用途：作为 AI 律师二开的架构参考基线，明确哪些可复用、哪些不能动

---

## 1. 项目定位

RuoYi-AI 3.0.0 不是单纯的 AI 问答系统，而是**四合一**的企业级 AI 平台：

| 定位 | 说明 | 核心能力 |
|------|------|---------|
| **RuoYi-Vue-Plus 后台管理底座** | 完整的 RBAC + 多租户 + 代码生成 + 审批工作流 | 用户/角色/菜单/部门/字典/租户 |
| **AI 对话平台** | 多模型接入 + SSE 流式响应 + 会话管理 | 6 个 Provider、ChatServiceFacade、SSE |
| **知识库/RAG 平台** | 文档上传→解析→切片→向量化→检索→回答 | ResourceLoaderFactory、VectorStoreStrategyFactory |
| **多模型接入平台** | OpenAI/通义/智谱/DeepSeek/Ollama/PPIO | ChatServiceFactory 自动发现机制 |

二开 AI 律师系统时，需要**全部复用**上述四层能力，在此基础上新增法律业务模块。

---

## 2. 技术栈总览

| 技术 | 作用 | 项目中的位置 | 二开时是否必须理解 | 学习优先级 |
|------|------|-------------|-------------------|-----------|
| Spring Boot 3.5.8 | Web 框架 | `pom.xml` L17 | 是 | ★★★★★ |
| JDK 17 | 运行时 | `pom.xml` L20 | 是 | ★★★★★ |
| Sa-Token 1.44.0 | 认证授权 | `ruoyi-common-satoken` | 是 | ★★★★★ |
| MyBatis-Plus 3.5.14 | ORM | `ruoyi-common-mybatis` | 是 | ★★★★★ |
| RuoYi-Vue-Plus | 管理底座 | `ruoyi-system` | 是 | ★★★★ |
| LangChain4j 1.13.0 | AI 编排 | `ruoyi-chat` | 是 | ★★★★ |
| SSE（SseEmitter） | 流式推送 | `ruoyi-common-sse` | 是 | ★★★★ |
| Redis + Redisson | 缓存/分布式锁 | `ruoyi-common-redis` | 是 | ★★★ |
| MySQL 8.0 | 关系数据库 | `application-dev.yml` | 是 | ★★★ |
| MinIO | 对象存储 | `ruoyi-common-oss` | 是 | ★★★ |
| 向量库（Milvus/Weaviate/Qdrant） | 向量存储和检索 | `ruoyi-chat/vector/` | 是 | ★★★★ |
| Apache Tika | 文档解析 | `ruoyi-chat` pom.xml | 是 | ★★★ |
| MapStruct-Plus | 对象映射 | `ruoyi-common-core` | 是 | ★★★ |
| Warm-Flow 1.8.2 | 审批工作流 | `ruoyi-workflow` | 律师审核需要 | ★★ |
| Langgraph4j 1.5.3 | AI 流程编排 | `ruoyi-aiflow` | 后续可能需要 | ★★ |
| MCP 协议 | 工具市场 | `ruoyi-chat/mcp/` | 后续可能需要 | ★★ |
| Neo4j | 知识图谱 | `pom.xml`（已排除自动配置） | 暂不需要 | ★ |
| Undertow | Web 服务器 | `application.yml` | 不需要深入 | ★ |

---

## 3. 模块结构

| 模块 | 职责 | 关键类 | 关键配置 | 二开建议 | 风险等级 |
|------|------|--------|---------|---------|---------|
| `ruoyi-admin` | Web 入口，端口 6039 | `RuoYiAIApplication` | `application.yml`、`application-dev.yml` | 可新增 Controller | 🟢 低 |
| `ruoyi-chat` | AI 核心模块 | `ChatServiceFacade`、`ChatServiceFactory`、`OpenAIServiceImpl` 等 6 个 Provider、`KnowledgeAttachServiceImpl`、`VectorStoreServiceImpl`、`VectorStoreStrategyFactory`、`ResourceLoaderFactory`、`EmbeddingModelFactory` | `VectorStoreProperties`（`vector-store.type`） | 不要重构核心链路；可新增 Provider | 🔴 高 |
| `ruoyi-aiflow` | AI 流程编排 | Langgraph4j 节点定义 | `application.yml` MCP/SSE 配置 | 可扩展节点 | 🟡 中 |
| `ruoyi-system` | 系统管理 | `SysUserController`、`SysRoleController`、`SysMenuController`、`SysDeptController` | `sys_menu`、`sys_role` 等表 | 可新增业务菜单和角色 | 🟢 低 |
| `ruoyi-generator` | 代码生成 | Velocity 模板 | `gen_table` 配置 | 工具，不常改 | 🟢 低 |
| `ruoyi-workflow` | 审批工作流 | Warm-Flow 流程定义 | `flow_definition` 表 | 可用于律师审核 | 🟡 中 |
| `ruoyi-common-chat` | Chat 公共层 | 实体/DTO/VO/枚举/Service 接口 | — | 可新增 DTO/VO | 🟡 中 |
| `ruoyi-common-sse` | SSE 推送 | `SseMessageUtils`、`SseEmitterManager` | — | **不要动** | 🔴 高 |
| `ruoyi-common-satoken` | 认证 | `LoginHelper` | `sa-token` 配置 | **不要动** | 🔴 高 |
| `ruoyi-common-security` | 安全拦截 | `SecurityConfig` | `security.excludes` | **不要动** | 🔴 高 |
| `ruoyi-common-tenant` | 多租户 | MyBatis-Plus 插件 | `tenant.excludes` | **不要动** | 🔴 高 |
| `ruoyi-common-mybatis` | ORM 基础 | `BaseEntity`、`BaseMapperPlus` | `mybatis-plus` 全局配置 | **不要动** | 🔴 高 |
| `ruoyi-common-oss` | 对象存储 | `OssService` | `sys_oss_config` 表 | 可复用 | 🟢 低 |
| `ruoyi-common-redis` | Redis | `RedisUtils` | Redisson 配置 | 可复用 | 🟢 低 |
| `ruoyi-common-core` | 核心工具 | `R<T>`、`MapstructUtils`、`StringUtils` | — | **不要动** | 🔴 高 |
| `ruoyi-extend/monitor` | Spring Boot Admin | 监控面板 | `application.yml` | 不常改 | 🟢 低 |
| `ruoyi-extend/snailjob` | 任务调度 | SnailJob | `application.yml` | 不常改 | 🟢 低 |

---

## 4. 启动链路

### 启动类

- 类：`org.ruoyi.RuoYiAIApplication`（`ruoyi-admin` 模块）
- 端口：6039（`application.yml` L4）
- 特殊逻辑：启动前自动杀掉占用 6039 端口的进程（Windows `taskkill`）

### 必需中间件

| 中间件 | 版本 | 默认端口 | 配置位置 | Docker 参考 |
|--------|------|---------|---------|------------|
| MySQL | 8.0 | 3306 | `application-dev.yml` L54-63 | `docs/docker/ruoyi-ai/docker-compose.yaml` |
| Redis | 6.2+ | 6379 | `application-dev.yml` L95-108 | `docker run -d -p 6379:6379 redis:7` |
| Milvus | v2.5.7 | 19530 | `application.yml` L304-306 | `docs/docker/milvus/docker-compose.yml` |
| MinIO | latest | 9000/9090 | `sys_oss_config` 表 | `docs/docker/ruoyi-ai/docker-compose.yaml` |

### 配置文件

| 文件 | 用途 | 环境 |
|------|------|------|
| `application.yml` | 主配置（端口/Sa-Token/租户/MyBatis-Plus/向量库） | 公共 |
| `application-dev.yml` | 开发环境（MySQL/Redis/MCP 配置） | dev |
| `application-prod.yml` | 生产环境 | prod |

### 初始化 SQL

- 位置：`docs/script/sql/ruoyi-ai-v3_mysql8.sql`（3482 行）
- 包含：RuoYi 基础表 + AI 业务表（chat_model/chat_provider/chat_session/chat_message/knowledge_*/mcp_*）
- Docker 自动导入：`docs/script/docker/mysql/init/init-db.sh`

### 前端仓库关系

- 管理端前端：独立仓库 [ruoyi-admin](https://github.com/ageerle/ruoyi-admin)（Vue 3 + Vben Admin）
- 用户端前端：独立仓库 [ruoyi-web](https://github.com/ageerle/ruoyi-web)（Vue 3 + element-plus-x）
- 前端通过 Nginx 反向代理后端 API
- SSE 连接路径：`/resource/sse`（`application.yml` L252）

---

## 5. 权限与租户

### 登录认证

- 框架：Sa-Token + JWT
- Token Header：`Authorization`
- 登录入口：`SysLoginService.login()` → `LoginHelper.login()`
- 获取用户：`LoginHelper.getUserId()` / `LoginHelper.getUsername()`
- 获取租户：`LoginHelper.getTenantId()`
- 超管判断：`LoginHelper.isSuperAdmin()`

### 权限注解

- 注解：`@SaCheckPermission("模块:功能:操作")`
- 示例：`@SaCheckPermission("system:model:add")`
- 前端菜单权限：`sys_menu` 表的 `perms` 字段
- 数据权限：`sys_role` 表的 `dataScope` 字段

### 用户/角色/菜单

- 用户表：`sys_user`（关联 `sys_dept` 部门）
- 角色表：`sys_role`（关联 `sys_menu` 权限）
- 菜单表：`sys_menu`（树形结构，含按钮级权限）
- 部门表：`sys_dept`（树形结构）

### 多租户数据隔离

- 开关：`tenant.enable: true`（`application.yml` L131）
- 实现方式：MyBatis-Plus 插件自动注入 `WHERE tenant_id = ?`
- 排除表：`tenant.excludes`（`application.yml` L133-143），排除 `sys_menu`、`sys_tenant` 等平台级表
- 租户表：`sys_tenant`、`sys_tenant_package`
- **不能绕过的原因**：所有业务查询自动带 `tenant_id` 条件，直接删字段会导致 SQL 报错或数据越权

---

## 6. AI 对话链路

```
用户发起对话（POST /chat/send）
  │
  ├─ ChatController.send()
  │   └─ 返回 SseEmitter
  │
  └─ ChatServiceFacade.sseChat()
      │
      ├─ 1. chatModelService.selectModelByName(modelName)
      │     从 chat_model 表查询模型配置（apiHost/apiKey/providerCode/category）
      │
      ├─ 2. buildContextMessages(chatRequest)
      │     ├─ PersistentChatMemoryStore：从 chat_message 表加载历史对话
      │     └─ VectorStoreService.getQueryVector()：如有 knowledgeId，从向量库检索相关文档
      │
      ├─ 3. handleSpecialChatModes()
      │     ├─ enableWorkFlow → 工作流模式
      │     ├─ isResume → 人机交互恢复
      │     └─ enableThinking → Supervisor Agent 模式（5 个子 Agent）
      │
      ├─ 4. ChatServiceFactory.getOriginalService(providerCode)
      │     按 provider_code 路由到具体 Provider 实现
      │
      ├─ 5. provider.buildStreamingChatModel(chatModel)
      │     构建 StreamingChatModel 实例（配置 baseUrl/apiKey/modelName/listeners）
      │
      └─ 6. streamingChatModel.chat(messages, handler)
            │
            ├─ onPartialResponse → SseMessageUtils.sendContent()
            ├─ onCompleteResponse → chatMessageService.saveChatMessage() + SseMessageUtils.sendDone()
            └─ onError → SseMessageUtils.sendError() + SseMessageUtils.completeConnection()
```

### 关键类与包路径

| 类 | 包路径 | 职责 |
|----|--------|------|
| `ChatController` | `org.ruoyi.controller.chat` | 对话入口，返回 SseEmitter |
| `ChatServiceFacade` | `org.ruoyi.service.chat.impl` | 对话统一入口，603 行 |
| `ChatServiceFactory` | `org.ruoyi.factory` | 模型路由工厂，自动发现 |
| `AbstractChatService` | `org.ruoyi.service.chat` | Provider 接口 |
| `OpenAIServiceImpl` 等 6 个 | `org.ruoyi.service.chat.impl.provider` | 具体模型实现 |
| `ChatModeType` | `org.ruoyi.enums` | 模型供应商枚举 |
| `ModelType` | `org.ruoyi.enums` | 模型类型枚举（chat/image/vector/reranker 等） |
| `ChatRequest` | `org.ruoyi.domain.dto` | 对话请求 DTO |
| `PersistentChatMemoryStore` | `org.ruoyi.service.chat.impl` | 历史消息持久化 |

---

## 7. 知识库/RAG 链路

```
用户上传文档
  │
  ├─ KnowledgeAttachController.upload()
  │
  └─ KnowledgeAttachServiceImpl.upload()
      │
      ├─ 1. OssService.uploadFile()                         # 文件存 MinIO，返回 URL
      │
      ├─ 2. ResourceLoaderFactory.getLoaderByFileType()      # 按文件类型选解析器
      │     ├── txt/csv/xml/yml → TextFileLoader
      │     ├── doc/docx        → WordLoader (Tika)
      │     ├── pdf             → PdfFileLoader (Tika)
      │     ├── md              → MarkDownFileLoader
      │     ├── xls/xlsx        → ExcelFileLoader
      │     └── java/py/go 等  → CodeFileLoader
      │
      ├─ 3. ResourceLoader.getContent()                      # 提取纯文本
      │
      ├─ 4. ResourceLoader.getChunkList()                    # 按配置参数切片
      │     参数来源：knowledge_info 表的 separator/overlapChar/textBlockSize
      │
      ├─ 5. KnowledgeFragment 批量入库                       # 切片存入 knowledge_fragment 表
      │
      └─ 6. VectorStoreService.storeEmbeddings()             # 向量化 + 存储
            │
            ├─ EmbeddingModelFactory.getModel()               # 获取 Embedding 模型
            │
            └─ VectorStoreStrategyFactory.getStrategy()       # 选择向量库
                  ├── milvus  → MilvusVectorStoreStrategy
                  ├── weaviate → WeaviateVectorStoreStrategy
                  └── qdrant  → QdrantVectorStoreStrategy

用户提问时检索：
  │
  └─ VectorStoreService.getQueryVector(query, knowledgeId)
      ├─ EmbeddingModelFactory.getModel()  # 查询向量化
      └─ strategy.query()                   # 向量相似度检索
            → 返回 TopN 相关文档片段
            → 注入到 ChatServiceFacade.buildContextMessages() 的上下文中
```

### 关键类与包路径

| 类 | 包路径 | 职责 |
|----|--------|------|
| `KnowledgeAttachController` | `org.ruoyi.controller.knowledge` | 文档上传入口 |
| `KnowledgeAttachServiceImpl` | `org.ruoyi.service.knowledge.impl` | 上传→解析→切片→向量化 全流程 |
| `KnowledgeInfoServiceImpl` | `org.ruoyi.service.knowledge.impl` | 知识库 CRUD |
| `KnowledgeFragmentServiceImpl` | `org.ruoyi.service.knowledge.impl` | 切片 CRUD |
| `VectorStoreServiceImpl` | `org.ruoyi.service.vector.impl` | 向量库代理层 |
| `VectorStoreStrategyFactory` | `org.ruoyi.factory` | 向量库策略工厂 |
| `ResourceLoaderFactory` | `org.ruoyi.factory` | 文档解析器工厂 |
| `EmbeddingModelFactory` | `org.ruoyi.factory` | Embedding 模型工厂（ConcurrentHashMap 缓存） |
| `FileTypeConstants` | `org.ruoyi.constant` | 支持的文件类型常量 |

### AI 相关核心表

| 表 | 用途 |
|----|------|
| `chat_model` | 模型配置（apiHost/apiKey/modelName/providerCode/category） |
| `chat_provider` | 模型供应商 |
| `chat_session` | 对话会话 |
| `chat_message` | 对话消息 |
| `knowledge_info` | 知识库信息 |
| `knowledge_attach` | 知识库附件 |
| `knowledge_fragment` | 知识库切片 |
| `knowledge_graph_instance` | 知识图谱实例 |
| `knowledge_graph_segment` | 知识图谱分段 |
| `mcp_market_info` | MCP 市场信息 |
| `mcp_market_tool` | MCP 市场工具 |
| `mcp_tool_info` | MCP 工具信息 |

---

## 8. 模型 Provider 扩展方式

> 本节说明新增模型供应商的步骤，但**本轮不要实现**。

### 扩展步骤

1. **在 `ChatModeType` 枚举中新增 code**
   - 位置：`org.ruoyi.enums.ChatModeType`
   - 示例：`GEMINI("gemini", "Gemini")`

2. **新建 Provider 实现类**
   - 位置：`ruoyi-chat/service/chat/impl/provider/`
   - 继承 `AbstractChatService`
   - 实现 `buildStreamingChatModel()` 方法
   - 使用 `@Service` 注册为 Spring Bean
   - `getProviderName()` 返回枚举 code

3. **`ChatServiceFactory` 自动发现**
   - 基于 `ApplicationContextAware`，启动时自动收集所有 `AbstractChatService` Bean
   - 无需修改工厂代码

4. **在 `chat_model` 表中新增模型记录**
   - `provider_code` 填枚举 code
   - `category` 填 `ModelType` 枚举值
   - `api_host` / `api_key` 填对应值

### 注意事项

- 新增 Provider 不需要修改 `ChatServiceFacade`
- 新增 Provider 不需要修改 `ChatServiceFactory`
- 如果 Provider 需要 LangChain4j 社区版支持，需在 `ruoyi-chat/pom.xml` 中添加依赖
- `langchain4j.community.version` 当前为 `1.13.0-beta23`

---

## 9. 当前已知风险点

| # | 风险 | 位置 | 说明 | 当前阶段是否处理 |
|---|------|------|------|----------------|
| 1 | API Key 明文存储 | `chat_model.api_key` | 数据库中 API Key 未加密，生产环境需加密 | 不处理，M4 评估 |
| 2 | RAG 无 Rerank | `ChatServiceFacade.buildContextMessages()` | 纯向量检索，`ModelType.RERANKER` 已定义但未使用 | 不处理，法律领域 RAG 精度不足时评估 |
| 3 | Prompt 未配置化 | `ChatServiceFacade.buildContextMessages()` | system prompt 硬编码，无 `prompt_template` 机制 | 不处理，AI Lawyer 二开需要时新增模块 |
| 4 | ChatServiceFacade 过重 | 单类 603 行 | 同时承担路由、上下文构建、SSE 管理、消息持久化 | 不处理，M3 评估拆分 |
| 5 | 数据库名不一致 | `application-dev.yml` vs `docker-compose.yaml` | 源码默认 `ruoyi-ai`，Docker 默认 `ruoyi-ai-agent` | 仅记录，Docker 部署时手动对齐 |
| 6 | Windows 路径硬编码 | `ChatServiceFacade.handleThinkingMode()` L222-248 | npx 路径写死 `C:\Program Files\nodejs\npx.cmd` | 不处理，跨平台部署时评估 |
| 7 | 商用合规风险 | 全局 | RuoYi-AI 采用 MIT 协议，可商用但需保留版权声明；AI 回答法律问题有执业风险 | **必须在 M2 前确认** |
| 8 | AI 法律回答准确性 | 全局 | LLM 幻觉可能导致错误法律建议，需要免责声明 + 人工审核机制 | **必须在 M2 前设计** |
