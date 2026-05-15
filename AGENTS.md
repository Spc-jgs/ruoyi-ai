# AGENTS.md

> 本文件是当前仓库的 AI 编码助手上下文指南。
> AI 助手在分析、修改、测试本项目时，必须优先遵守本文档。
> 如本文档与实际源码冲突，以实际源码为准，并在执行前说明冲突点。

---

## 1. Project Mission

- 当前项目基线：**RuoYi-AI tag 3.0.0**（全栈式 AI 开发平台）
- 公司 Codeup `main` 分支以 tag 3.0.0 初始化，作为二开基线
- 二开目标：基于 RuoYi-AI 构建 **AI 律师系统**（AI 法律咨询 + 律师服务撮合 + 合同模板交易 + 律所管理 + 平台运营）
- 核心约束：RBAC 权限体系和多租户是数据隔离的根基，不可绕过。AI 对话通过 SSE 流式推送，必须保持连接稳定性
- **当前阶段是 M0/M1 摸底验证，不是直接开发业务**

---

## 2. Repository Baseline

- 当前公司二开基线：RuoYi-AI tag `3.0.0`
- 当前 Codeup `main` 分支由 tag `3.0.0` 初始化
- 后续业务二开基于 Codeup `main` 或 `feature/*` 分支进行
- **不要**向原作者仓库 `upstream` 推送代码
- **不要**向 GitHub 推送公司二开代码
- 如需同步原项目更新，必须先创建 `upgrade/ruoyi-ai-x.y.z` 分支验证后再合并
- 分支命名规范：`feature/模块名-简述`、`fix/简述`、`upgrade/ruoyi-ai-x.y.z`

---

## 3. Current Phase

| 阶段 | 目标 | 允许的操作 | 禁止的操作 |
|------|------|-----------|-----------|
| **M0** | 跑通原项目、摸清架构、明确二开边界 | 配置调整、模型 Key 填写、Docker 启动中间件 | 修改业务代码、新增模块 |
| **M1** | 验证 AI 对话、知识库/RAG、文件上传解析、SSE 流式响应 | API 接口测试、知识库功能验证、RAG 链路验证 | 重构核心链路、新增业务模块 |
| **M2** | 设计 AI 律师 MVP、出详细 PRD、确认模块拆分 | 需求文档、模块设计、数据库设计 | 开始写业务代码 |
| **M3** | 新增 AI 律师业务模块 | 新建 ruoyi-lawyer 等模块、新增表和接口 | 重构 ruoyi-chat 核心链路 |
| **M4** | 接入合同模板、律师入驻、审核流程 | 合同模板 CRUD、律师资质审核、管理端菜单 | 复杂支付/分账 |
| **M5** | 再考虑订单、IM、佣金、服务商 | 支付对接、腾讯 IM、佣金分账 | — |

**当前处于 M0→M1 过渡期，不要直接开发 AI 律师业务代码。**

---

## 4. Toolchain Registry

| Intent | Command | Notes |
|--------|---------|-------|
| JDK | JDK 17（`java.version=17`，pom.xml L20） | 必须 JDK 17，不支持 8/11/21 |
| Build 全量 | `mvn clean install -DskipTests` | 根目录执行 |
| Build 入口模块 | `mvn -pl ruoyi-admin -am -DskipTests package -Pdev` | 编译后端入口及所有依赖模块 |
| Run backend | `mvn -pl ruoyi-admin -am spring-boot:run -Pdev` | 或 IDEA 直接运行 `RuoYiAIApplication`（端口 6039） |
| Test | `mvn test -Pdev` | 是否按 `@Tag("dev")` 过滤以 pom.xml surefire 配置为准 |
| API Docs | `http://localhost:6039/doc.html` | Knife4j 增强文档 |
| Lint | 无独立 lint 工具 | 依赖 IDEA 检查 + compiler `-parameters` |

### 必需中间件

| 中间件 | 端口 | 配置位置 | Docker 启动 |
|--------|------|---------|------------|
| MySQL 8.0 | 3306 | `application-dev.yml` L54-63 | `docs/docker/milvus/docker-compose.yml` 或独立 |
| Redis | 6379 | `application-dev.yml` L95-108 | `docker run -d -p 6379:6379 redis:7` |
| Milvus v2.5.7 | 19530 | `application.yml` L304-306（`vector-store.type: milvus`） | `cd docs/docker/milvus && docker-compose up -d` |
| MinIO | 9000/9090 | `sys_oss_config` 表 | 见 `docs/docker/ruoyi-ai/docker-compose.yaml` |

### 前端仓库

- 管理端：独立仓库 `ruoyi-admin`（Vue 3 + Vben Admin）
- 用户端：独立仓库 `ruoyi-web`（Vue 3 + element-plus-x）
- 前端通过 Nginx 反向代理访问后端 API

### 启动顺序

```
MySQL → Redis → Milvus → MinIO → RuoYiAIApplication → 前端
```

---

## 5. Judgment Boundaries

### NEVER

- **永远不要**绕过 `LoginHelper.getUserId()` / `LoginHelper.getTenantId()` 直接硬编码用户 ID 或租户 ID
- **永远不要**绕过 `@SaCheckPermission` 注解，权限是数据安全的底线
- **永远不要**删除或绕过 `tenant_id` 字段，多租户隔离依赖此字段
- **永远不要**在 `ruoyi-common-*` 模块中引入 `ruoyi-modules` 的依赖（依赖方向是 modules → common）
- **永远不要**在 `ChatServiceFacade` 之外直接创建 `StreamingChatModel`，所有模型调用必须经过工厂路由
- **永远不要**将真实的 API Key / 密码 / AccessKey / SecretKey 提交到代码或 SQL 中
- **永远不要**直接修改生产配置（`application-prod.yml`）
- **永远不要**破坏 SSE 流式响应链路（`SseMessageUtils` → `SseEmitterManager`）
- **永远不要**直接修改 `sys_*` 基础表结构，除非确认不影响 RuoYi 基础功能
- **永远不要**在当前阶段（M0/M1）直接重构 `ChatServiceFacade`、`ChatServiceFactory`、`VectorStoreStrategyFactory`

### ASK

- 新增模块前先确认应放在 `ruoyi-modules` 还是 `ruoyi-common`
- 新增 AI Provider 前先确认 `ChatModeType` 枚举和 `ChatServiceFactory` 的扩展方式
- 修改向量库类型前先确认 `VectorStoreStrategyFactory` 是否已支持
- 涉及 `sys_*` 前缀表的变更需确认是否影响 RuoYi 基础功能
- 新增 SSE 事件类型前先确认 `SseMessageUtils` 的事件协议
- 新增配置文件前必须先确认用途、加载方式、是否会被提交；禁止提交真实密钥
- 想要重构任何核心类（`ChatServiceFacade`、`ChatServiceFactory` 等）时，必须先说明原因和影响范围

### ALWAYS

- 新增实体类必须继承 `BaseEntity`（`ruoyi-common-mybatis`）或 `TenantEntity`（需要租户隔离时）
- 新增 Mapper 必须继承 `BaseMapperPlus<Entity, Vo>`，不要直接用 `BaseMapper`
- 新增 Controller 必须继承 `BaseController`，返回值统一用 `R<T>` 或 `TableDataInfo<T>`
- 新增 Service 接口方法时，同步更新对应的 Bo（输入）和 Vo（输出）类
- 使用 `@SaCheckPermission` 注解保护写操作接口
- 使用 `@RepeatSubmit` 注解防止重复提交
- 使用 `@Log` 注解记录关键操作日志
- 使用 `MapstructUtils.convert()` 做 Bo ↔ Entity 转换，不要手写
- 保存 AI 对话消息时调用 `chatMessageService.saveChatMessage()`，确保消息持久化
- SSE 响应完成后必须调用 `SseMessageUtils.completeConnection()` 关闭连接

---

## 6. RuoYi-AI Architecture Map

### 模块总览

| 模块 | 职责 | 关键类/包 | 二开建议 | 风险等级 |
|------|------|----------|---------|---------|
| `ruoyi-admin` | Web 入口，唯一启动模块 | `RuoYiAIApplication`、`config/`、`controller/` | 可新增 Controller | 🟢 低 |
| `ruoyi-chat` | ★ AI 核心模块 | `ChatServiceFacade`、`ChatServiceFactory`、Provider、RAG、向量库 | 不要重构核心链路，可新增 Provider | 🔴 高 |
| `ruoyi-aiflow` | AI 流程编排（Langgraph4j） | 工作流设计器、SSE 流式执行 | 可扩展节点类型 | 🟡 中 |
| `ruoyi-system` | 系统管理（用户/角色/菜单/部门/字典） | `SysUserController`、`SysRoleController` 等 | 可新增业务菜单 | 🟢 低 |
| `ruoyi-generator` | 代码生成器（Velocity 模板） | 代码生成配置 | 工具模块，不常改 | 🟢 低 |
| `ruoyi-workflow` | 审批工作流（Warm-Flow） | 流程定义、任务审批 | 可用于律师审核流程 | 🟡 中 |
| `ruoyi-common-*` | 24 个通用组件 | 认证/租户/Redis/OSS/SSE/MyBatis 等 | **不要动** | 🔴 高 |
| `ruoyi-extend/*` | 扩展模块（监控/调度） | Spring Boot Admin、SnailJob | 不常改 | 🟢 低 |

### 目录结构

```
ruoyi-ai-master/
├── ruoyi-admin/                    # Web 入口（唯一启动模块，端口 6039）
│   └── src/main/java/org/ruoyi/
│       ├── RuoYiAIApplication.java
│       ├── config/
│       └── controller/
│
├── ruoyi-modules/
│   ├── ruoyi-chat/                 # ★ AI 核心模块（对话/RAG/向量库/MCP）
│   │   └── .../org/ruoyi/
│   │       ├── controller/         # chat/ knowledge/ mcp/
│   │       ├── service/            # chat/impl/ knowledge/impl/ vector/impl/ embed/
│   │       ├── factory/            # ChatServiceFactory, VectorStoreStrategyFactory, ResourceLoaderFactory
│   │       ├── domain/             # entity/ bo/ vo/ dto/
│   │       ├── mapper/
│   │       ├── agent/              # 多智能体（Supervisor 模式）
│   │       ├── enums/              # ChatModeType, ModelType
│   │       └── config/             # VectorStoreProperties, McpSseConfig
│   ├── ruoyi-aiflow/              # AI 流程编排（Langgraph4j）
│   ├── ruoyi-system/              # 系统管理（用户/角色/菜单/部门/字典）
│   ├── ruoyi-generator/           # 代码生成器
│   └── ruoyi-workflow/            # 审批工作流（Warm-Flow）
│
├── ruoyi-common/                   # 24 个通用组件（不要动）
│   ├── ruoyi-common-chat/         # Chat 公共层（实体/DTO/VO/枚举/Service接口）
│   ├── ruoyi-common-sse/          # SSE 流式推送
│   ├── ruoyi-common-satoken/      # Sa-Token 认证
│   ├── ruoyi-common-security/     # 安全拦截
│   ├── ruoyi-common-oss/          # 对象存储（MinIO）
│   ├── ruoyi-common-tenant/       # 多租户
│   ├── ruoyi-common-mybatis/      # MyBatis-Plus（BaseEntity, BaseMapperPlus）
│   ├── ruoyi-common-redis/        # Redis + Redisson
│   ├── ruoyi-common-core/         # 核心工具（R<T>, MapstructUtils）
│   └── ... (其他 15 个)
│
├── ruoyi-extend/                   # 扩展模块
│   ├── ruoyi-monitor-admin/       # Spring Boot Admin 监控
│   └── ruoyi-snailjob-server/     # SnailJob 任务调度
│
└── docs/
    ├── ai-analysis/               # 架构分析文档
    ├── product/                   # 产品方向文档
    ├── script/sql/                # 数据库初始化 SQL
    └── docker/                    # Docker Compose 文件
```

---

## 7. Core AI Architecture

### 7.1 AI 对话链路

```
ChatController (POST /chat/send)
  → ChatServiceFacade.sseChat()
    ├── 1. chatModelService.selectModelByName()          # 查 chat_model 表获取模型配置
    ├── 2. buildContextMessages()                         # 构建上下文
    │     ├── PersistentChatMemoryStore                  # 从 DB 加载历史消息
    │     └── VectorStoreService.getQueryVector()        # 如有 knowledgeId，检索知识库
    ├── 3. handleSpecialChatModes()                       # 工作流/人机交互/思考模式
    ├── 4. ChatServiceFactory.getOriginalService()        # 按 provider_code 路由
    ├── 5. AbstractChatService.buildStreamingChatModel()  # 构建 StreamingChatModel
    └── 6. streamingChatModel.chat(messages, handler)     # 发起流式调用
          └── StreamingChatResponseHandler                # SSE 推送 + chatMessageService 持久化
```

### 7.2 模型路由链路

```
ChatModeType 枚举 → provider_code
  openai   → OpenAIServiceImpl
  qianwen  → QianWenChatServiceImpl
  zhipu    → ZhiPuChatServiceImpl
  deepseek → DeepseekServiceImpl
  ollama   → OllamaServiceImpl
  ppio     → PPIOServiceImpl

ChatServiceFactory: ApplicationContext 启动时收集所有 AbstractChatService 实现
  按 getProviderName() 注册 → 运行时按 provider_code 路由
```

### 7.3 知识库/RAG 链路

```
KnowledgeAttachController.upload()
  → KnowledgeAttachServiceImpl.upload()
    ├── OssService.uploadFile()                          # 文件存 MinIO
    ├── ResourceLoaderFactory.getLoaderByFileType()       # 选解析器（6 类）
    ├── ResourceLoader.getContent()                       # 提取文本
    ├── ResourceLoader.getChunkList()                     # 文本切片
    ├── KnowledgeFragment 批量入库                        # 切片持久化到 knowledge_fragment 表
    └── VectorStoreService.storeEmbeddings()              # 向量化 + 存向量库
          └── VectorStoreStrategyFactory.getStrategy()    # 策略模式选 Milvus/Weaviate/Qdrant
                └── EmbeddingModelFactory.getModel()      # 获取 Embedding 模型
```

### 7.4 文档上传/解析/切片/向量化链路

```
文件上传 → MinIO (OssService)
  → ResourceLoaderFactory.getLoaderByFileType(fileType)
    ├── txt/csv/xml/yml → TextFileLoader
    ├── doc/docx        → WordLoader (Apache Tika)
    ├── pdf             → PdfFileLoader (Apache Tika)
    ├── md              → MarkDownFileLoader
    ├── xls/xlsx        → ExcelFileLoader
    └── 代码文件(20+)   → CodeFileLoader
  → getContent() → getChunkList(separator, overlapChar, textBlockSize)
  → knowledge_fragment 表批量插入
  → EmbeddingModelFactory.getModel() → Embed
  → VectorStoreStrategyFactory → Milvus/Weaviate/Qdrant
```

### 7.5 SSE 流式响应链路

```
ChatController.send() → 返回 SseEmitter
  → SseEmitterManager 建立连接
  → StreamingChatResponseHandler.onPartialResponse()
    → SseMessageUtils.sendContent(userId, partialResponse)
  → StreamingChatResponseHandler.onCompleteResponse()
    → chatMessageService.saveChatMessage()  # 持久化
    → SseMessageUtils.sendDone(userId)
    → SseMessageUtils.completeConnection(userId, tokenValue)

事件类型：content(片段) / done(完成) / error(错误)
```

### 7.6 新增模型 Provider 扩展方式

1. 在 `ChatModeType` 枚举中新增 code
2. 在 `ruoyi-chat/service/chat/impl/provider/` 下新建实现类，实现 `AbstractChatService`
3. 使用 `@Service` 注册为 Spring Bean，`getProviderName()` 返回枚举 code
4. `ChatServiceFactory` 自动发现，无需修改工厂代码
5. 在 `chat_model` 表中新增模型记录，`provider_code` 填枚举 code

---

## 8. AI Lawyer Secondary Development Direction

### 五端定位

| 端 | 使用人群 | 核心诉求 | 主要功能 | 一期建议 |
|----|---------|---------|---------|---------|
| **C 端** | 个人用户 | 免费法律咨询、找律师、下载合同 | AI 对话、律师列表/详情、合同模板列表/下载 | **必须做** |
| **律师端** | 执业律师 | AI 辅助办公、服务变现 | AI Agent（案件分析/文书草拟）、服务上架、合同模板上传、收益 | **必须做** |
| **律所端** | 律所管理员 | 管理律师、抽佣、数据看板 | 律师管理、佣金配置、数据看板 | **简化做** |
| **服务商端** | 合作方/子公司 | 查看管辖范围数据 | 数据看板、客户管理 | **后续做** |
| **平台管理端** | 运营/审核/客服 | 全局管理 | 账号/客户/律师/律所/服务/订单/合同/内容/客服/权限/系统 | **必须做** |

### 当前阶段原则

- **先复用 RuoYi-AI 的 AI 对话、知识库、权限、菜单、文件上传、SSE 能力**
- **新业务模块优先新增，不直接污染 ruoyi-chat 核心链路**
- AI 律师业务模块建议独立拆分为 `ruoyi-lawyer`、`ruoyi-legal-consult` 等
- **初期不要做复杂支付、佣金、IM、分账**
- 后续再考虑腾讯 IM、订单支付、合同模板付费、律所抽佣

---

## 9. Reuse / Extension / Do Not Touch

### 可直接复用

| 能力 | RuoYi-AI 模块/类 | 说明 |
|------|-----------------|------|
| 用户/角色/菜单/权限 | `ruoyi-system`（SysUser/SysRole/SysMenu） | RBAC 完整可用，新增业务菜单即可 |
| Sa-Token 登录认证 | `ruoyi-common-satoken` + `LoginHelper` | JWT + 权限注解，直接用 |
| 多租户能力 | `ruoyi-common-tenant` + `TenantEntity` | 数据隔离自动注入 |
| AI 对话基础能力 | `ChatServiceFacade` → `ChatServiceFactory` → Provider | 6 个模型 Provider 开箱即用 |
| 知识库/RAG | `KnowledgeAttachServiceImpl` → `VectorStoreServiceImpl` | 上传→解析→切片→向量化→检索 |
| 文件上传/OSS | `ruoyi-common-oss` + `OssService`（MinIO） | 文件存取完整 |
| SSE 流式响应 | `ruoyi-common-sse` + `SseMessageUtils` | 流式推送协议完整 |
| 管理端基础能力 | `ruoyi-admin`（Vben Admin 前端） | 后台管理框架完整 |
| 审批工作流 | `ruoyi-workflow`（Warm-Flow） | 可用于律师入驻审核 |
| 代码生成器 | `ruoyi-generator`（Velocity） | 新模块 CRUD 脚手架 |

### 可轻微改造

| 能力 | 改造点 | 风险 |
|------|--------|------|
| 模型配置 | `chat_model` 表增加法律领域模型、`ChatModeType` 枚举扩展 | 🟢 低 |
| Prompt 组织 | `ChatServiceFacade.buildContextMessages()` 增加 system prompt 配置化 | 🟡 中 |
| 知识库分类 | `knowledge_info` 增加法律分类字段（民法典/刑法/合同法等） | 🟢 低 |
| 问答记录 | `chat_session`/`chat_message` 增加业务类型标识 | 🟢 低 |
| 文件解析 | `ResourceLoaderFactory` 增加法律文书格式支持 | 🟢 低 |
| 管理端菜单 | `sys_menu` 增加 AI 律师业务菜单 | 🟢 低 |

### 建议新增（独立模块）

| 模块 | 职责 | 一期需要 |
|------|------|---------|
| `ruoyi-lawyer` | 律师/律所/执业信息管理 | 是 |
| `ruoyi-legal-consult` | 法律咨询/问答记录/咨询意向 | 是 |
| `ruoyi-contract` | 合同模板/下载/审核 | 是 |
| `ruoyi-legal-order` | 订单/支付/收益（后续版本） | 否 |
| `ruoyi-service-provider` | 服务商管理（后续版本） | 否 |

### 暂时不要动

| 模块/类 | 原因 |
|---------|------|
| `ruoyi-common-*`（全部 24 个） | 公共底座，修改影响全局 |
| `ruoyi-system` 的 `sys_*` 基础表 | RuoYi 基础功能依赖 |
| `ChatServiceFacade` | AI 对话核心入口，603 行，拆分风险极高 |
| `ChatServiceFactory` | 模型路由核心，所有 Provider 依赖 |
| `VectorStoreStrategyFactory` | 向量库策略路由，3 种实现依赖 |
| `SseMessageUtils` / `SseEmitterManager` | SSE 推送核心，断连/漏推风险 |
| `ruoyi-common-tenant` 多租户插件 | 数据隔离根基，改错会越权 |
| `ruoyi-common-satoken` 认证体系 | 安全根基，改错会鉴权失败 |
| 全局返回结构 `R<T>` / `TableDataInfo<T>` | 前后端契约，不能改 |
| `BaseEntity` / `BaseMapperPlus` / `MapstructUtils` | ORM 基础设施 |
