> 基线版本：RuoYi-AI tag 3.0.0  
分析日期：2026-05-18  
分析范围：全部 POM 依赖声明 + 源码使用位置
>

---

## 依赖全景概览
项目采用 Maven 多模块 + `dependencyManagement` 统一版本管理模式，总共涉及 **~60+ 个外部第三方依赖** + **24 个内部 common 模块** + **5 个内部业务模块**。

```plain
ruoyi-ai (根 POM — dependencyManagement 统一版本)
├── Spring Boot 3.5.8 (spring-boot-dependencies BOM)  ← 基础底座
├── ruoyi-common-bom ← 24 个内部 common 模块版本
├── 无 BOM 的独立依赖（40+ 个） ← 逐个声明版本
└── gRPC BOM ← 解决 Milvus SDK 依赖冲突
```

---

## 一、基础设施层依赖（通过 spring-boot-dependencies BOM）
这些依赖版本由 Spring Boot 3.5.8 统一管理：

| 依赖 | 作用 | 使用位置 |
| --- | --- | --- |
| **spring-boot-starter-web** | Spring MVC Web 框架 | `ruoyi-admin`（入口）、`ruoyi-monitor-admin` |
| **spring-boot-starter-undertow** | Web 容器（替代 Tomcat，性能更强） | `ruoyi-monitor-admin` |
| **spring-boot-starter-aop** | AOP 切面支持 | `ruoyi-common-core`（供全部模块使用） |
| **spring-boot-starter-validation** | Bean Validation 参数校验 | `ruoyi-common-core`（供全部模块使用） |
| **spring-boot-starter-security** | Spring Security 安全认证 | `ruoyi-monitor-admin`（仅监控模块） |
| **spring-boot-configuration-processor** | 生成配置元数据 JSON，IDE 提示 | `ruoyi-common-core` |
| **spring-boot-properties-migrator** | 配置属性迁移辅助 | `ruoyi-common-core`（runtime scope） |
| **spring-boot-starter-test** | 测试框架 | `ruoyi-admin`、`ruoyi-chat`（test scope） |
| **spring-context-support** | Spring 上下文扩展 | `ruoyi-common-core` |
| **spring-web / spring-webmvc** | Spring Web MVC 核心 | `ruoyi-common-core` |
| **jakarta.servlet-api** | Servlet API | `ruoyi-common-core` |
| **commons-lang3** | Apache 通用工具类（StringUtils 等） | `ruoyi-common-core` |
| **jackson-databind** | JSON 序列化/反序列化 | `ruoyi-aiflow` |
| **jackson-dataformat-xml** | Jackson XML 格式支持 | 根 POM 声明 |
| **jackson-datatype-jsr310** | Jackson Java 8 时间类型支持 | `ruoyi-common-redis` |
| **caffeine** | 本地内存缓存 | `ruoyi-common-redis`、`ruoyi-common-satoken` |
| **HikariCP** | 数据库连接池 | `ruoyi-chat` |
| **mysql-connector-j** | MySQL JDBC 驱动 | `ruoyi-admin`、`ruoyi-chat` |


---

## 二、ORM / 数据库层依赖
| 依赖 | 版本 | 作用 | 使用位置 |
| --- | --- | --- | --- |
| **mybatis-plus-spring-boot3-starter** | 3.5.14 | MyBatis-Plus ORM 框架（Spring Boot 3） | `ruoyi-common-mybatis` |
| **mybatis-plus-jsqlparser** | 3.5.14 | MyBatis-Plus JSqlParser（SQL 解析） | `ruoyi-common-mybatis` |
| **mybatis-plus-annotation** | 3.5.14 | MyBatis-Plus 注解 | 根 POM dependencyManagement 声明 |
| **mybatis** | 3.5.16 | MyBatis 核心 | 根 POM dependencyManagement 声明 |
| **dynamic-datasource-spring-boot3-starter** | 4.3.1 | 多数据源动态切换 | `ruoyi-common-mybatis` |
| **p6spy** | 3.9.1 | SQL 性能分析 / 日志打印 | `ruoyi-common-mybatis` |


**代码使用**：

+ [BaseMapperPlus](file:///D:/projects/ruoyi-ai-master/ruoyi-common/ruoyi-common-mybatis/src/main/java/org/ruoyi/common/mybatis/core/mapper/BaseMapperPlus.java) — 全项目 Mapper 统一基类
+ [BaseEntity](file:///D:/projects/ruoyi-ai-master/ruoyi-common/ruoyi-common-mybatis/src/main/java/org/ruoyi/common/mybatis/core/domain/BaseEntity.java) — 全项目实体统一基类
+ `@DS("agent")` 注解 — [TableSchemaManager](file:///D:/projects/ruoyi-ai-master/ruoyi-modules/ruoyi-chat/src/main/java/org/ruoyi/agent/manager/TableSchemaManager.java) 中的多数据源切换

---

## 三、缓存 / Redis 层依赖
| 依赖 | 版本 | 作用 | 使用位置 |
| --- | --- | --- | --- |
| **redisson-spring-boot-starter** | 3.51.0 | 分布式 Redis 客户端（锁、队列、缓存） | `ruoyi-common-redis` |
| **lock4j-redisson-spring-boot-starter** | 2.2.7 | 分布式锁注解支持（`@Lock4j`） | `ruoyi-common-redis` |


**代码使用**：

+ [RedisUtils](file:///D:/projects/ruoyi-ai-master/ruoyi-common/ruoyi-common-redis/src/main/java/org/ruoyi/common/redis/utils/RedisUtils.java) — Redis 操作工具类
+ 分布式锁用于防止并发冲突

---

## 四、认证 / 权限层依赖
| 依赖 | 版本 | 作用 | 使用位置 |
| --- | --- | --- | --- |
| **sa-token-spring-boot3-starter** | 1.44.0 | Sa-Token 权限认证框架（Spring Boot 3） | `ruoyi-common-satoken` |
| **sa-token-jwt** | 1.44.0 | Sa-Token + JWT 整合 | `ruoyi-common-satoken` |
| **sa-token-core** | 1.44.0 | Sa-Token 核心 | 根 POM dependencyManagement 声明 |
| **JustAuth** | 1.16.7 | 第三方登录（微信、QQ、微博等） | `ruoyi-common-social` |


**代码使用**：

+ [LoginHelper](file:///D:/projects/ruoyi-ai-master/ruoyi-common/ruoyi-common-satoken/src/main/java/org/ruoyi/common/satoken/utils/LoginHelper.java) — 全局登录用户获取（`getUserId()` / `getTenantId()`）
+ `@SaCheckPermission` 注解 — 全项目 Controller 权限控制
+ `StpUtil.getTokenValue()` — ChatServiceFacade 中获取 SSE 连接的 token

---

## 五、AI 核心依赖（LangChain4j 生态）
### 5.1 LangChain4j 核心
| 依赖 | 版本 | 作用 | 使用位置 |
| --- | --- | --- | --- |
| **langchain4j** | 1.13.0 | LangChain4j 核心 API（ChatModel, EmbeddingModel, Memory 等） | `ruoyi-common-chat` |
| **langchain4j-core** | 1.13.0 | LangChain4j 底层核心 | `ruoyi-aiflow` |
| **langchain4j-open-ai** | 1.13.0 | OpenAI 协议 Provider（兼容 DeepSeek/千问/智谱等） | `ruoyi-chat` |
| **langchain4j-ollama** | 1.13.0 | Ollama 本地模型 Provider | `ruoyi-chat` |


**代码使用**：

+ [ChatServiceFacade](file:///D:/projects/ruoyi-ai-master/ruoyi-modules/ruoyi-chat/src/main/java/org/ruoyi/service/chat/impl/ChatServiceFacade.java) — 全局 AI 对话入口
+ [AbstractChatService](file:///D:/projects/ruoyi-ai-master/ruoyi-modules/ruoyi-chat/src/main/java/org/ruoyi/service/chat/AbstractChatService.java) — 模型 Provider 抽象基类
+ `StreamingChatModel` — 流式对话模型接口
+ `EmbeddingModel` — 向量化模型接口
+ `ChatMemory` / `MessageWindowChatMemory` — 对话记忆管理

### 5.2 LangChain4j Community 扩展
| 依赖 | 版本 | 作用 | 使用位置 |
| --- | --- | --- | --- |
| **langchain4j-community-dashscope** | 1.13.0-beta23 | 阿里百炼（通义千问）Provider | `ruoyi-chat` |
| **langchain4j-community-zhipu-ai** | 1.13.0-beta23 | 智谱 AI Provider | `ruoyi-chat` |
| **langchain4j-document-parser-apache-tika** | 1.13.0-beta23 | 文档解析（PDF/Word/Excel/TXT 等） | `ruoyi-chat` → [ResourceLoaderFactory](file:///D:/projects/ruoyi-ai-master/ruoyi-modules/ruoyi-chat/src/main/java/org/ruoyi/factory/ResourceLoaderFactory.java) |
| **langchain4j-milvus** | 1.13.0-beta23 | Milvus 向量库集成 | `ruoyi-chat` → [MilvusVectorStoreStrategy](file:///D:/projects/ruoyi-ai-master/ruoyi-modules/ruoyi-chat/src/main/java/org/ruoyi/service/vector/impl/MilvusVectorStoreStrategy.java) |
| **langchain4j-weaviate** | 1.13.0-beta23 | Weaviate 向量库集成 | `ruoyi-chat` → [WeaviateVectorStoreStrategy](file:///D:/projects/ruoyi-ai-master/ruoyi-modules/ruoyi-chat/src/main/java/org/ruoyi/service/vector/impl/WeaviateVectorStoreStrategy.java) |
| **langchain4j-qdrant** | 1.13.0-beta23 | Qdrant 向量库集成 | `ruoyi-chat` → QdrantVectorStoreStrategy |
| **langchain4j-mcp** | 1.13.0-beta23 | MCP 协议支持（Model Context Protocol） | `ruoyi-chat` → [LangChain4jMcpToolProviderService](file:///D:/projects/ruoyi-ai-master/ruoyi-modules/ruoyi-chat/src/main/java/org/ruoyi/mcp/service/core/LangChain4jMcpToolProviderService.java) |
| **langchain4j-agentic** | 1.13.0-beta23 | 多智能体框架（Agentic Supervisor） | `ruoyi-chat` → [ChatServiceFacade.handleThinkingMode()](file:///D:/projects/ruoyi-ai-master/ruoyi-modules/ruoyi-chat/src/main/java/org/ruoyi/service/chat/impl/ChatServiceFacade.java#L212) |
| **langchain4j-skills** | 1.13.0-beta23 | Skills 技能模块（SKILL.md） | `ruoyi-chat` → SkillsAgent |
| **langchain4j-experimental-skills-shell** | 1.13.0-beta23 | Skills Shell 执行器 | `ruoyi-chat` → SkillsAgent 的 `ShellSkills.from()` |


### 5.3 AI 流程编排依赖
| 依赖 | 版本 | 作用 | 使用位置 |
| --- | --- | --- | --- |
| **langgraph4j-core** | 1.5.3 | LangGraph4j 核心（图状态机工作流引擎） | `ruoyi-aiflow` → [WorkflowEngine](file:///D:/projects/ruoyi-ai-master/ruoyi-modules/ruoyi-aiflow/src/main/java/org/ruoyi/workflow/workflow/WorkflowEngine.java) |
| **langgraph4j-langchain4j** | 1.5.3 | LangGraph4j + LangChain4j 桥接 | `ruoyi-aiflow` |


**代码使用**：`WorkflowEngine` 使用 `StateGraph.compile()` → `app.stream()` → `app.getState()` 实现 AI 流程编排的 Docker 式流水线执行。

### 5.4 其他 AI 相关依赖
| 依赖 | 版本 | 作用 | 使用位置 |
| --- | --- | --- | --- |
| **openai-java** | 4.8.0 | OpenAI Java SDK（直接 HTTP 调用） | `ruoyi-common-oss` → 文件解析后可能用于图片识别等 |
| **okhttp** | 4.12.0 | HTTP 客户端（openai-java 依赖） | `ruoyi-common-oss` |
| **dify-java-client** | 1.0.7 | Dify 平台 Java SDK | `ruoyi-chat`（声明但暂无源码使用） |
| **neo4j-java-driver** | (Spring BOM) | Neo4j 图数据库驱动 | `ruoyi-chat`（声明但暂无源码使用） |
| **weaviate** (testcontainers) | 1.19.6 | Weaviate 测试容器 | `ruoyi-chat` test scope |
| **google-api-client** | 2.6.0 | Google API 客户端 | `ruoyi-aiflow`（可能用于 Google Search 等） |
| **jsoup** | 1.21.2 | HTML 解析器 | `ruoyi-aiflow` |
| **avatar-generator** | 1.1.0 | 头像生成器 | `ruoyi-aiflow` |
| **avatar-generator-cat** | 1.1.0 | 猫咪风格头像生成器 | `ruoyi-aiflow` |


**待确认**：`dify-java-client` 和 `neo4j-java-driver` 在当前源码中未找到直接使用，可能为预留扩展依赖。

---

## 六、存储 / 文件层依赖
| 依赖 | 版本 | 作用 | 使用位置 |
| --- | --- | --- | --- |
| **aws-sdk-s3** | 2.28.22 | AWS S3 协议 SDK（MinIO 兼容） | `ruoyi-common-oss` |
| **aws-sdk-s3-transfer-manager** | 2.28.22 | S3 传输管理器（大文件分片上传） | `ruoyi-common-oss` |
| **aws-sdk-netty-nio-client** | 2.28.22 | AWS SDK Netty HTTP 客户端 | `ruoyi-common-oss` |
| **commons-compress** | 1.27.1 | Apache 压缩库（ZIP/GZIP 等） | 根 POM（解决 POI 导出 Excel 时的 NoSuchMethodError） |


---

## 七、API 文档 / 接口依赖
| 依赖 | 版本 | 作用 | 使用位置 |
| --- | --- | --- | --- |
| **knife4j-openapi3-jakarta-spring-boot-starter** | 4.4.0 | Knife4j API 文档增强（Swagger UI） | `ruoyi-aiflow` |
| **springdoc-openapi-starter-webmvc-ui** | 2.8.13 | SpringDoc OpenAPI 3 规范实现 | `ruoyi-aiflow` |
| **springdoc-openapi-starter-webmvc-api** | 2.8.13 | SpringDoc API 模块 | 根 POM dependencyManagement |
| **swagger-annotations** | 2.2.8 | Swagger 注解（`@Schema` 等） | `ruoyi-common-chat` |


**前端访问**：`http://localhost:6039/doc.html` (Knife4j)

---

## 八、工具 / 辅助类依赖
| 依赖 | 版本 | 作用 | 使用位置 |
| --- | --- | --- | --- |
| **hutool-core** | 5.8.40 | 国产 Java 工具类库核心 | `ruoyi-common-core` |
| **hutool-http** | 5.8.40 | Hutool HTTP 客户端 | `ruoyi-common-core` |
| **hutool-extra** | 5.8.40 | Hutool 扩展模块 | `ruoyi-common-core` |
| **hutool-crypto** | 5.8.40 | Hutool 加密工具 | `ruoyi-common-encrypt` |
| **mapstruct-plus-spring-boot-starter** | 1.5.0 | 对象转换框架（MapStruct 增强版） | `ruoyi-common-core` |
| **mapstruct-plus-processor** | 1.5.0 | MapStruct Plus 编译期注解处理器 | 根 POM compiler annotationProcessorPaths |
| **lombok** | 1.18.40 | 简化 Java 代码（`@Data`/`@Slf4j` 等） | **全项目** |
| **lombok-mapstruct-binding** | 0.2.0 | Lombok + MapStruct 兼容绑定 | 根 POM compiler annotationProcessorPaths |
| **therapi-runtime-javadoc** | 0.15.0 | 运行时读取 Javadoc（供 SpringDoc 使用） | 根 POM dependencyManagement |
| **therapi-runtime-javadoc-scribe** | 0.15.0 | Javadoc 编译期写入 | 根 POM compiler annotationProcessorPaths |
| **fastexcel** | 1.3.0 | 高性能 Excel 读写 | `ruoyi-common-excel` |
| **fastjson** | 1.2.83 | 阿里巴巴 JSON 序列化库 | `ruoyi-common-chat` |
| **ip2region** | 2.7.0 | 离线 IP 地址定位库 | `ruoyi-common-core` → `ip2region.xdb` 资源文件 |
| **bouncycastle bcprov-jdk15to18** | 1.80 | 加密算法库（SM2/SM3/SM4 国密等） | `ruoyi-common-encrypt` |


**代码使用**：

+ `MapstructUtils.convert()` — 全项目 Bo ↔ Entity ↔ Vo 转换
+ `ExcelUtil.exportExcel()` — 管理端数据导出
+ Hutool 在 `StringUtils`、`DateUtil` 等场景广泛使用

---

## 九、业务功能依赖
### 9.1 工作流引擎
| 依赖 | 版本 | 作用 | 使用位置 |
| --- | --- | --- | --- |
| **warm-flow-mybatis-plus-sb3-starter** | 1.8.2 | Warm-Flow 国产工作流引擎 | `ruoyi-workflow` |
| **warm-flow-plugin-ui-sb-web** | 1.8.2 | Warm-Flow 管理端 UI 插件 | `ruoyi-workflow` |


**代码使用**：Warm-Flow 提供审批流程定义、任务审批等功能，未来可用于律师入驻审核流程。

### 9.2 代码生成器
| 依赖 | 版本 | 作用 | 使用位置 |
| --- | --- | --- | --- |
| **velocity-engine-core** | 2.3 | Apache Velocity 模板引擎 | `ruoyi-generator` |
| **anyline-environment-spring-data-jdbc** | 8.7.2-20250603 | AnyLine 运行时 ORM（代码生成用） | `ruoyi-generator` |
| **anyline-data-jdbc-mysql** | 8.7.2-20250603 | AnyLine MySQL 适配 | `ruoyi-generator` |


**代码使用**：`ruoyi-generator` 模块的 Velocity 模板代码生成，从数据库表结构生成 Controller/Service/Mapper/Entity/Vue 页面。

### 9.3 任务调度
| 依赖 | 版本 | 作用 | 使用位置 |
| --- | --- | --- | --- |
| **snail-job-client-starter** | 1.8.0 | SnailJob 客户端（分布式任务调度） | 根 POM dependencyManagement |
| **snail-job-client-job-core** | 1.8.0 | SnailJob 任务核心 | 根 POM dependencyManagement |
| **snail-job-server-starter** | 1.8.0 | SnailJob 服务端 | `ruoyi-snailjob-server`（独立部署） |


### 9.4 监控管理
| 依赖 | 版本 | 作用 | 使用位置 |
| --- | --- | --- | --- |
| **spring-boot-admin-starter-server** | 3.5.5 | Spring Boot Admin 服务端 | `ruoyi-monitor-admin` |
| **spring-boot-admin-starter-client** | 3.5.5 | Spring Boot Admin 客户端 | `ruoyi-admin`、`ruoyi-monitor-admin`、`ruoyi-snailjob-server` |


### 9.5 短信 / 邮件
| 依赖 | 版本 | 作用 | 使用位置 |
| --- | --- | --- | --- |
| **sms4j-spring-boot-starter** | 3.3.5 | 短信发送聚合框架（阿里云/腾讯云等） | `ruoyi-common-sms` |


邮件由 Spring Boot 自带的 `spring-boot-starter-mail` 支持（`ruoyi-common-mail` 模块）。

### 9.6 企业微信
| 依赖 | 版本 | 作用 | 使用位置 |
| --- | --- | --- | --- |
| **weixin-java-cp** | 4.6.0 | 企业微信 Java SDK | 根 POM 声明（暂无模块直接使用） |


---

## 十、内部模块依赖关系图
```plain
ruoyi-admin (唯一 Web 入口)
├── ruoyi-system          → 系统管理（用户/角色/菜单)
│   ├── ruoyi-common-core       ← Spring / Hutool / MapStruct
│   ├── ruoyi-common-mybatis    ← MyBatis-Plus / 多数据源 / p6spy
│   ├── ruoyi-common-satoken    ← Sa-Token / JWT / Redis
│   ├── ruoyi-common-redis      ← Redisson / Caffeine / lock4j
│   ├── ruoyi-common-tenant     ← 多租户数据隔离
│   ├── ruoyi-common-security   ← 安全拦截
│   ├── ruoyi-common-oss        ← MinIO S3 / OpenAI SDK
│   ├── ruoyi-common-sms        ← sms4j
│   ├── ruoyi-common-excel      ← fastexcel
│   ├── ruoyi-common-encrypt    ← BouncyCastle / Hutool-crypto
│   ├── ruoyi-common-websocket  ← WebSocket
│   └── ruoyi-common-sse        ← SSE 流式推送
│
├── ruoyi-chat             → AI 核心模块
│   ├── ruoyi-common-chat       ← langchain4j 核心 / fastjson
│   │   ├── langchain4j 1.13.0
│   │   └── ruoyi-common-core
│   ├── langchain4j-open-ai     ← OpenAI Provider
│   ├── langchain4j-ollama      ← Ollama Provider
│   ├── langchain4j-community-dashscope ← 通义千问 Provider
│   ├── langchain4j-community-zhipu-ai  ← 智谱 Provider
│   ├── langchain4j-agentic     ← 多智能体框架
│   ├── langchain4j-mcp         ← MCP 协议
│   ├── langchain4j-skills      ← Skills 技能模块
│   ├── langchain4j-milvus      ← Milvus 向量库
│   ├── langchain4j-weaviate    ← Weaviate 向量库
│   ├── langchain4j-qdrant      ← Qdrant 向量库
│   ├── langchain4j-document-parser-apache-tika ← 文档解析
│   ├── dify-java-client        ← Dify 平台（预留）
│   └── neo4j-java-driver       ← Neo4j 图数据库（预留）
│
├── ruoyi-aiflow           → AI 流程编排
│   ├── langgraph4j-core         ← 图状态机引擎
│   ├── langgraph4j-langchain4j  ← 与 LangChain4j 桥接
│   ├── google-api-client        ← Google API
│   ├── jsoup                    ← HTML 解析
│   └── avatar-generator         ← 头像生成
│
├── ruoyi-workflow         → 审批工作流
│   └── warm-flow-mybatis-plus-sb3-starter ← Warm-Flow 引擎
│
├── ruoyi-generator        → 代码生成器
│   ├── velocity-engine-core     ← 模板引擎
│   └── anyline-*                ← 动态 ORM
│
└── ruoyi-common-* (24 个) → 公共底座
```

---

## 十一、编译期注解处理器
根 POM 的 `maven-compiler-plugin` 配置了 5 个注解处理器：

| 处理器 | 作用 |
| --- | --- |
| **therapi-runtime-javadoc-scribe** | 将 Javadoc 写入字节码，供运行时 SpringDoc 读取 |
| **lombok** | 生成 getter/setter/builder/log 等 |
| **spring-boot-configuration-processor** | 生成 `spring-configuration-metadata.json` |
| **mapstruct-plus-processor** | 生成 Bo ↔ Entity ↔ Vo 转换实现类 |
| **lombok-mapstruct-binding** | 解决 Lombok 与 MapStruct 的编译顺序冲突 |


---

## 十二、独立部署模块（可单独打包）
| 模块 | 打包方式 | 依赖亮点 |
| --- | --- | --- |
| **ruoyi-admin** | Spring Boot jar（`repackage`） | 聚合所有业务模块 |
| **ruoyi-monitor-admin** | Spring Boot jar | Spring Security + Undertow |
| **ruoyi-snailjob-server** | Spring Boot jar | Scala 2.13.9 + SnailJob Server |


---

## 十三、预留 / 暂未使用的依赖
以下依赖已声明但当前源码中无直接使用痕迹，可能为后续版本预留：

| 依赖 | 声明位置 | 用途推测 |
| --- | --- | --- |
| `dify-java-client` 1.0.7 | `ruoyi-chat` | 集成 Dify AI 平台 API |
| `neo4j-java-driver` | `ruoyi-chat` | Neo4j 图数据库知识图谱 |
| `weixin-java-cp` 4.6.0 | 根 POM | 企业微信推送/通知 |
| `weaviate` (testcontainers) 1.19.6 | `ruoyi-chat` test | Weaviate 集成测试 |


---

## 十四、依赖冲突处理
| 冲突场景 | 解决方式 |
| --- | --- |
| Milvus SDK 与 LangChain4j 的 gRPC 版本不一致 | 通过 `grpc-bom` 1.62.2 统一强制版本 |
| Sa-Token 自带 Hutool 与项目 Hutool 版本冲突 | `sa-token-jwt` 中排除 `hutool-all` |
| LangChain4j Tika 的 commons-compress 版本过旧 | `langchain4j-document-parser-apache-tika` 排除 `commons-compress`，统一使用 1.27.1 |
| AWS SDK 多个 HTTP Client 冲突 | `s3` 中排除 `aws-crt-client`、`apache-client`、`url-connection-client`，只用 `netty-nio-client` |


