# RuoYi-AI M0 启动试运行准备清单

> 基于项目 Tag 3.0.0 源码分析
> 分析时间：2026-05-08
> 本文档仅做启动准备分析，不修改任何代码、配置、SQL

---

## 1. 本次启动目标

本次 M0 的唯一目标是：**跑通原项目 + 验证最小 AI 链路**。

具体包括：
1. 后端服务能成功启动，无报错
2. 管理端前端能登录
3. 能通过管理端配置一个 AI 模型
4. 能完成一次 AI 对话（SSE 流式响应）
5. 能上传文档到知识库并完成一次知识库问答

**不做的事情**：
- 不做 AI 律师业务二开
- 不改 SQL、不改配置文件、不改业务代码
- 不做性能优化
- 不做生产部署

---

## 2. 推荐启动路线

### 路线对比

| 路线 | 优点 | 缺点 | 适合场景 |
|------|------|------|---------|
| **Docker 一键启动** | 最快，5分钟出结果；数据库/Redis/向量库自动就绪 | 镜像是预构建的，无法调试后端代码；修改模型Key需改Docker环境变量 | 只想看效果，不关心代码 |
| **源码启动后端** | 能断点调试、看日志、理解代码；修改配置方便 | 需要自己装MySQL/Redis/向量库 | **M0 推荐**，后续二开必需 |
| **源码启动前端** | 能修改前端、理解前后端交互 | 前端代码在独立仓库，需单独克隆 | 验证前后端联调 |

### 当前最推荐路线

**源码启动后端 + Docker 启动中间件（MySQL/Redis/向量库） + 管理端前端用预构建 Docker 镜像**

理由：
1. 后端源码启动才能看到完整日志、断点调试 AI 链路
2. 中间件用 Docker 启动最快最稳，避免本地安装版本冲突
3. 管理端前端 M0 阶段不需要改代码，用 Docker 镜像即可

---

## 3. 本机需要准备的软件

| 软件 | 版本要求 | 依据文件路径 | M0 是否必须 | 安装建议 |
|------|---------|------------|-----------|---------|
| **JDK** | **17** | `pom.xml` L20 `<java.version>17</java.version>`；`Dockerfile.backend` L4 `maven:3.9-eclipse-temurin-17` | 是 | 推荐 Eclipse Temurin 17 |
| **Maven** | **3.9+** | `Dockerfile.backend` L4 `maven:3.9-eclipse-temurin-17` | 是 | 推荐 Maven 3.9.x |
| **Node.js** | 项目后端不需要Node.js；前端独立仓库 | README.md L54 前端仓库 `ruoyi-admin`、`ruoyi-web` 是独立项目 | 否（前端用Docker则不需要） | 如需源码启动前端，建议 18+ |
| **pnpm/npm/yarn** | 前端项目决定 | 前端仓库独立，本后端仓库不含 package.json | 否 | 如需源码启动前端再确认 |
| **Docker** | 需要 Docker + Docker Compose | `docs/docker/ruoyi-ai/docker-compose.yaml`；`docs/docker/milvus/docker-compose.yml` | **强烈推荐** | Docker Desktop for Windows |
| **MySQL** | **8.0+** | `docs/script/sql/ruoyi-ai-v3_mysql8.sql` L7 注释 `80045 (8.0.45)`；`docker-compose.yaml` L14 `mysql:8.0.33` | 是（本地安装或Docker） | 推荐用Docker启动 |
| **Redis** | **6.2+** | `docker-compose-all.yaml` L43 `redis:6.2`；`docker-compose.yaml` L46 `redis:6.2` | 是（本地安装或Docker） | 推荐用Docker启动 |
| **向量库** | 取决于 `vector-store.type` 配置 | `application.yml` L294-312；默认 `type: milvus` | 是（至少一种） | 推荐Milvus，用Docker启动 |
| **Milvus** | **v2.5.7** | `docs/docker/milvus/docker-compose.yml` L41 `milvusdb/milvus:v2.5.7` | 推荐（当前默认） | Docker启动，含etcd+minio |
| **Weaviate** | **1.19.7~1.30.0** | `docs/docker/weaviate/docker-compose.yml` L11 `weaviate:1.19.7`；`docker-compose-all.yaml` L61 `weaviate:1.30.0` | 可选 | Docker启动 |
| **Qdrant** | **latest** | `docs/docker/qdrant/docker-compose.yml` L4 `qdrant/qdrant:latest` | 可选 | Docker启动 |
| **MinIO** | latest | `docker-compose.yaml` L82 `minio/minio` | 推荐（OSS存储依赖） | Docker启动 |

---

## 4. 需要启动的服务

### 4.1 MySQL

| 项目 | 内容 |
|------|------|
| 作用 | 主数据库，存储用户/角色/菜单/会话/消息/知识库/模型配置等所有业务数据 |
| 配置文件位置 | `ruoyi-admin/src/main/resources/application-dev.yml` L54-63 |
| 默认端口 | 3306（Docker模式映射到23306） |
| 默认库名 | `ruoyi-ai`（源码模式）或 `ruoyi-ai-agent`（Docker模式） |
| 默认账号 | root / root |
| M0 是否必须 | **是** |
| Docker启动命令 | 见第8节 |

### 4.2 Redis

| 项目 | 内容 |
|------|------|
| 作用 | Token存储、缓存、分布式锁、Sa-Token会话存储 |
| 配置文件位置 | `ruoyi-admin/src/main/resources/application-dev.yml` L95-108 |
| 默认端口 | 6379（Docker模式映射到26379或6379） |
| 默认密码 | 无（`application-dev.yml` L104 注释了password） |
| M0 是否必须 | **是** |
| Docker启动命令 | 见第8节 |

### 4.3 向量库（Milvus）

| 项目 | 内容 |
|------|------|
| 作用 | 存储文档向量，支持语义检索 |
| 配置文件位置 | `application.yml` L304-306（`vector-store.milvus.url: http://localhost:19530`） |
| 默认端口 | 19530（gRPC），9091（健康检查） |
| Collection名称 | `LocalKnowledge`（`application.yml` L306） |
| M0 是否必须 | **是**（知识库问答依赖向量库） |
| Docker启动命令 | 见第8节（含etcd+minio依赖） |

### 4.4 MinIO

| 项目 | 内容 |
|------|------|
| 作用 | 对象存储，存储上传的文件/文档/图片 |
| 配置位置 | `sys_oss_config` 表（SQL L2724：accessKey=ruoyi, secretKey=ruoyi123, endpoint=127.0.0.1:9000） |
| 默认端口 | 9000（API），9090（Console） |
| 默认账号 | ruoyi / ruoyi123 |
| M0 是否必须 | **是**（知识库文件上传依赖OSS） |
| Docker启动命令 | 见第8节 |

### 4.5 后端服务

| 项目 | 内容 |
|------|------|
| 作用 | Spring Boot 主服务，提供所有 API |
| 启动类 | `ruoyi-admin/src/main/java/org/ruoyi/RuoYiAIApplication.java` |
| 配置文件 | `ruoyi-admin/src/main/resources/application.yml` + `application-dev.yml` |
| 默认端口 | 6039 |
| M0 是否必须 | **是** |
| 启动方式 | IDEA 直接运行 或 `mvn spring-boot:run` |

### 4.6 管理端前端

| 项目 | 内容 |
|------|------|
| 作用 | 管理后台界面，配置模型/知识库/用户等 |
| 源码仓库 | 独立仓库 `ruoyi-admin`（https://github.com/ageerle/ruoyi-admin） |
| Docker镜像 | `crpi-31mraxd99y2gqdgr.cn-beijing.personal.cr.aliyuncs.com/ruoyi_ai/ruoyi-ai-admin:latest` |
| 默认端口 | 5666（源码）或 25666（Docker一键启动） |
| 默认账号 | admin / admin123 |
| M0 是否必须 | **是**（配置模型和知识库依赖管理端） |
| Docker启动命令 | 见第8节 |

### 4.7 用户端前端

| 项目 | 内容 |
|------|------|
| 作用 | 用户聊天界面 |
| 源码仓库 | 独立仓库 `ruoyi-web`（https://github.com/ageerle/ruoyi-web） |
| Docker镜像 | `crpi-31mraxd99y2gqdgr.cn-beijing.personal.cr.aliyuncs.com/ruoyi_ai/ruoyi-ai-web:latest` |
| 默认端口 | 5137（源码）或 25137（Docker一键启动） |
| M0 是否必须 | **可选**（管理端也可测试对话） |

### 服务启动顺序

```
1. MySQL → 等待健康检查通过
2. Redis
3. Milvus（含 etcd + minio）
4. MinIO
5. 后端服务 → 等待启动成功日志
6. 管理端前端
7. 用户端前端（可选）
```

---

## 5. 数据库准备

### 5.1 初始化 SQL 在哪里
`docs/script/sql/ruoyi-ai-v3_mysql8.sql`（3482行）

### 5.2 默认库名
- **源码模式**：`ruoyi-ai`（`application-dev.yml` L61：`jdbc:mysql://127.0.0.1:3306/ruoyi-ai`）
- **Docker模式**：`ruoyi-ai-agent`（`docker-compose.yaml` L21：`MYSQL_DATABASE: ruoyi-ai-agent`）

> **注意**：库名不一致！如果用Docker启动MySQL但源码启动后端，需要确保后端 `application-dev.yml` 中的数据库名与Docker创建的库名一致。

### 5.3 默认账号密码
- 用户名：`root`
- 密码：`root`
- 依据：`application-dev.yml` L62-63

### 5.4 是否会自动初始化
- **Docker模式**：会。`docker-compose.yaml` L24-25 挂载了初始化脚本：
  - `/docker-entrypoint-initdb.d/init-db.sh`（`docs/script/docker/mysql/init/init-db.sh`）
  - `/docker-entrypoint-initdb.d/ruoyi-ai-v3_mysql8.sql`（`docs/script/sql/ruoyi-ai-v3_mysql8.sql`）
  - `init-db.sh` 内容：`mysql -uroot -proot ruoyi-ai-agent --force < ruoyi-ai-v3_mysql8.sql`
- **本地MySQL**：不会自动初始化，需要手动导入 SQL

### 5.5 是否需要手动执行 SQL
- **Docker模式**：不需要，容器首次启动时自动执行
- **本地MySQL模式**：**需要**手动执行：
  ```sql
  CREATE DATABASE `ruoyi-ai` DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;
  USE `ruoyi-ai`;
  SOURCE D:/projects/ruoyi-ai-master/docs/script/sql/ruoyi-ai-v3_mysql8.sql;
  ```

### 5.6 哪些表和 AI 模块相关

| 表名 | 作用 | 是否M0必需 |
|------|------|----------|
| `chat_message` | 聊天消息记录 | 是 |
| `chat_model` | 模型配置（API Key、模型名等） | **是**（必须配置模型才能对话） |
| `chat_provider` | 厂商信息 | 是（SQL已预置5个厂商） |
| `chat_session` | 会话管理 | 是 |
| `knowledge_info` | 知识库信息 | 是（知识库问答必需） |
| `knowledge_attach` | 知识库附件 | 是 |
| `knowledge_fragment` | 知识片段 | 是 |
| `knowledge_graph_instance` | 知识图谱实例 | 否（M0可跳过） |
| `knowledge_graph_segment` | 知识图谱片段 | 否（M0可跳过） |
| `mcp_market_info` | MCP市场 | 否（M0可跳过） |
| `mcp_market_tool` | MCP市场工具 | 否（M0可跳过） |
| `mcp_tool_info` | MCP工具 | 否（M0可跳过） |
| `sys_oss_config` | OSS存储配置 | 是（文件上传依赖） |

SQL 中 `chat_model` 表预置了2条记录（L75-76）：
- `deepseek/deepseek-v3.2`（PPIO，chat类型，Key 为 `sk_xx` 占位符）
- `baai/bge-m3`（PPIO，vector类型，Key 为 `sk_xx` 占位符）

**需要将 `sk_xx` 替换为真实的 API Key**，或者通过管理端新增模型配置。

---

## 6. 模型 Key 准备

### 6.1 项目支持哪些模型供应商

依据：`ruoyi-modules/ruoyi-chat/src/main/java/org/ruoyi/enums/ChatModeType.java` 和 SQL `chat_provider` 预置数据

| 供应商 | 编码（provider_code） | 实现类 | API格式 |
|--------|---------------------|--------|---------|
| OpenAI | openai | `OpenAIServiceImpl` | OpenAI 兼容 |
| 阿里云百炼 | qianwen | `QianWenChatServiceImpl` | DashScope |
| 智谱AI | zhipu | `ZhiPuChatServiceImpl` | 智谱 |
| DeepSeek | deepseek | `DeepseekServiceImpl` | OpenAI 兼容 |
| Ollama | ollama | `OllamaServiceImpl` | Ollama 本地 |
| PPIO | ppio | `PPIOServiceImpl` | OpenAI 兼容 |

### 6.2 模型配置在哪里
- **数据库**：`chat_model` 表（`ChatModelServiceImpl.selectModelByName()` 查询）
- **管理端界面**：`ChatModelController` → `POST /system/model`（权限标识：`system:model:add`）
- **关键字段**：
  - `model_name`：模型名称（如 `deepseek/deepseek-v3.2`），**对话时用这个名称匹配**
  - `provider_code`：供应商编码（必须与 `ChatModeType` 枚举的 code 一致）
  - `category`：模型分类（`chat`=对话，`vector`=向量，`reranker`=重排序 等）
  - `api_host`：API 地址
  - `api_key`：API 密钥

### 6.3 M0 最推荐用哪个模型供应商跑通

**推荐 DeepSeek**，理由：
1. DeepSeek API 兼容 OpenAI 格式，直接用 `openai` 或 `deepseek` 的 provider_code
2. 价格极低（约 ¥1/百万Token），适合试运行
3. 国内网络可直接访问（`https://api.deepseek.com`）
4. SQL预置的 PPIO 厂商也是 DeepSeek 系，但 PPIO 需要额外注册

**备选**：PPIO（`https://api.ppinfra.com/openai`），SQL 已预置配置，只需填 Key

### 6.4 需要配置哪些 Key

M0 最小配置需要 **2个模型 Key**：

| Key 类型 | 用途 | category | 示例模型名 |
|----------|------|----------|----------|
| **聊天模型 Key** | AI 对话 | chat | `deepseek-chat` |
| **向量模型 Key** | 文档向量化/检索 | vector | `bge-m3` 或 DeepSeek 的 embedding |

> 如果使用 DeepSeek，它目前不提供独立 embedding API，向量模型建议用 PPIO 的 `baai/bge-m3` 或阿里云百炼的 `text-embedding-v3`。

### 6.5 Key 应该放在哪里
- **方式1**：通过管理端界面配置（`系统管理 → 模型管理 → 新增`）
  - 前端路径：管理端登录后 → AI管理 → 模型管理
  - 后端接口：`POST /system/model`
- **方式2**：直接修改数据库 `chat_model` 表的 `api_key` 字段
  - SQL 中预置的 `sk_xx` 需要替换为真实 Key

**不要把真实 Key 写进文档或代码！**

---

## 7. 知识库/RAG 试运行准备

### 7.1 上传文件入口在哪里
- **接口**：`POST /system/attach/upload`
- **Controller**：`KnowledgeAttachController.upload()`（`ruoyi-modules/ruoyi-chat/src/main/java/org/ruoyi/controller/knowledge/KnowledgeAttachController.java` L110-114）
- **入参**：`KnowledgeInfoUploadBo`（含 `knowledgeId` + `MultipartFile file`）
- **管理端界面**：AI管理 → 知识库 → 选择知识库 → 上传文档

### 7.2 支持哪些文件类型
依据：`FileTypeConstants.java`（`ruoyi-modules/ruoyi-chat/src/main/java/org/ruoyi/constant/FileTypeConstants.java`）

| 文件类型 | 扩展名 | 解析器 |
|---------|--------|--------|
| 纯文本 | txt, csv, log, xml, properties, ini, yaml, yml | `TextFileLoader` + `CharacterTextSplitter` |
| Word | doc, docx | `WordLoader` + `CharacterTextSplitter` |
| PDF | pdf | `PdfFileLoader` + `CharacterTextSplitter` |
| Markdown | md | `MarkDownFileLoader` + `MarkdownTextSplitter` |
| Excel | xls, xlsx | `ExcelFileLoader` + `ExcelTextSplitter` |
| 代码文件 | java, py, js, html, css, sql, ts, cpp 等20+种 | `CodeFileLoader` + `CodeTextSplitter` |

**M0 推荐用 txt 或 pdf 测试**，最简单不容易出错。

### 7.3 文档解析链路在哪里
`KnowledgeAttachServiceImpl.upload()`（L163-219）

```
文件上传 → OssService.uploadFile() → 存到MinIO
         → ResourceLoaderFactory.getLoaderByFileType() → 选解析器
         → ResourceLoader.getContent() → 提取文本
         → ResourceLoader.getChunkList() → 文本切片
         → KnowledgeFragment 批量入库
         → VectorStoreService.storeEmbeddings() → 向量化+存入Milvus
```

### 7.4 向量化入口在哪里
`VectorStoreService.storeEmbeddings(StoreEmbeddingBo)` → 由 `KnowledgeAttachServiceImpl.upload()` L218 调用
- 实际委托给 `VectorStoreStrategyFactory.getStrategy()` → `MilvusVectorStoreStrategy.storeEmbeddings()`

### 7.5 检索入口在哪里
`VectorStoreService.getQueryVector(QueryVectorBo)` → 由 `ChatServiceFacade.buildContextMessages()` L456 调用

### 7.6 问答入口在哪里
- **接口**：`POST /chat/send`
- **Controller**：`ChatController.sseChat()`（L32-36）
- **核心逻辑**：`ChatServiceFacade.sseChat()`
  - 如果 `ChatRequest.knowledgeId` 不为空，自动走知识库检索增强

### 7.7 最小验证步骤

1. 确保向量库（Milvus）正常运行
2. 确保向量模型（`category=vector`）已在 `chat_model` 表中配置且 Key 有效
3. 在管理端创建一个知识库（必须选择向量库类型和向量模型）
4. 上传一个简单的 txt 文件（如一段法律条文）
5. 等待上传完成（后端日志会显示切片和向量化进度）
6. 新建对话，选择该知识库
7. 输入与文档内容相关的问题
8. 验证 AI 回答中包含了文档内容

---

## 8. 启动命令草案

### 8.1 方案A：Docker 启动中间件 + 源码启动后端（推荐）

#### Step 1：启动 MySQL（Docker）

```powershell
docker run -d --name ruoyi-ai-mysql ^
  -p 3306:3306 ^
  -e MYSQL_ROOT_PASSWORD=root ^
  -e MYSQL_DATABASE=ruoyi-ai ^
  -e TZ=Asia/Shanghai ^
  -v ruoyi-mysql-data:/var/lib/mysql ^
  mysql:8.0.33 ^
  --default-authentication-plugin=mysql_native_password ^
  --character-set-server=utf8mb4 ^
  --collation-server=utf8mb4_general_ci ^
  --lower_case_table_names=1
```

等待MySQL启动后，手动导入SQL：
```powershell
# 等待30秒让MySQL完全启动
timeout /t 30

# 导入SQL（需要将SQL文件复制到容器中或使用客户端工具）
docker exec -i ruoyi-ai-mysql mysql -uroot -proot ruoyi-ai < "D:\projects\ruoyi-ai-master\docs\script\sql\ruoyi-ai-v3_mysql8.sql"
```

#### Step 2：启动 Redis（Docker）

```powershell
docker run -d --name ruoyi-ai-redis ^
  -p 6379:6379 ^
  redis:6.2 ^
  redis-server --appendonly yes
```

#### Step 3：启动 Milvus（Docker Compose）

```powershell
cd D:\projects\ruoyi-ai-master\docs\docker\milvus
docker-compose up -d
```

这会启动3个容器：etcd、milvus-minio（Milvus内部用）、milvus-standalone
- Milvus端口：19530
- Attu管理界面：http://localhost:19500

#### Step 4：启动 MinIO（Docker）

```powershell
docker run -d --name ruoyi-ai-minio ^
  -p 9000:9000 ^
  -p 9090:9090 ^
  -e MINIO_ROOT_USER=ruoyi ^
  -e MINIO_ROOT_PASSWORD=ruoyi123 ^
  -v ruoyi-minio-data:/data ^
  minio/minio ^
  server /data --console-address ":9090"
```

#### Step 5：源码启动后端

方式1 - IDEA 启动（推荐）：
- 打开 IDEA，导入项目
- 找到 `ruoyi-admin/src/main/java/org/ruoyi/RuoYiAIApplication.java`
- 直接运行 main 方法
- 确保 Maven Profile 选中 `dev`

方式2 - 命令行启动：
```powershell
cd D:\projects\ruoyi-ai-master
mvn clean install -DskipTests
cd ruoyi-admin
mvn spring-boot:run -Pdev
```

#### Step 6：管理端前端（Docker 镜像）

```powershell
docker run -d --name ruoyi-ai-admin ^
  -p 5666:5666 ^
  -e UPSTREAM_HOST=host.docker.internal:6039 ^
  crpi-31mraxd99y2gqdgr.cn-beijing.personal.cr.aliyuncs.com/ruoyi_ai/ruoyi-ai-admin:latest
```

> `host.docker.internal` 是 Docker Desktop for Windows 提供的宿主机域名，用于容器访问宿主机上的后端服务。

### 8.2 方案B：全 Docker 一键启动（最省事但无法调试代码）

```powershell
cd D:\projects\ruoyi-ai-master\docs\docker\ruoyi-ai
docker-compose -f docker-compose-all.yaml up -d
```

这会启动7个容器：MySQL、Redis、Weaviate、MinIO、后端、管理端、用户端

> **注意**：一键启动使用 Weaviate 作为向量库，但 `application.yml` 默认配置是 `milvus`。Docker 镜像内部已处理此差异，但源码启动需注意向量库类型匹配。

### 8.3 方案C：全源码启动（最灵活但最复杂）

中间件全部本地安装（略），后端和前端都从源码启动。前端需单独克隆：
- 管理端：`git clone https://github.com/ageerle/ruoyi-admin.git`
- 用户端：`git clone https://github.com/ageerle/ruoyi-web.git`

---

## 9. 启动后验证步骤

### 9.1 管理端能登录

1. 打开 `http://localhost:5666`（Docker）或 `http://localhost:25666`（一键启动）
2. 输入账号 `admin`，密码 `admin123`
3. 成功进入管理后台首页

**失败排查**：
- 前端无法访问 → 检查 Docker 容器是否启动：`docker ps`
- 登录失败 → 检查后端是否启动、MySQL是否正常

### 9.2 用户端能打开

1. 打开 `http://localhost:5137`（Docker源码）或 `http://localhost:25137`（一键启动）
2. 页面正常加载

### 9.3 后端接口能访问

1. 打开 Swagger 文档：`http://localhost:6039/doc.html`
2. 能看到接口文档页面
3. 健康检查：`http://localhost:6039/actuator/health` 返回 `{"status":"UP"}`

### 9.4 模型配置成功

1. 登录管理端 → AI管理 → 模型管理
2. 新增一个聊天模型：
   - 模型名称：`deepseek-chat`（或你想用的模型名）
   - 供应商编码：`deepseek`
   - 模型分类：`chat`
   - API地址：`https://api.deepseek.com`
   - API Key：填入真实 Key
3. 新增一个向量模型：
   - 模型名称：`bge-m3`
   - 供应商编码：`ppio`（或你用的供应商）
   - 模型分类：`vector`
   - API地址：对应供应商的 embedding 接口地址
   - API Key：填入真实 Key
4. 在模型列表中能看到新增的记录

### 9.5 普通对话成功

1. 在用户端或管理端打开对话界面
2. 选择刚配置的聊天模型（如 `deepseek-chat`）
3. 输入：`你好，请自我介绍`
4. 看到 SSE 流式响应，AI 回复正常
5. 后端日志无异常

**验证关键日志**（后端控制台）：
```
路由到服务提供商: deepseek, 模型: deepseek-chat
收到消息片段: ...
消息结束，已保存到数据库
```

### 9.6 上传一个测试文档

1. 准备一个简单的 txt 文件，内容如：
   ```
   中华人民共和国民法典第一千一百六十五条：行为人因过错侵害他人民事权益造成损害的，应当承担侵权责任。
   ```
2. 管理端 → AI管理 → 知识库 → 新增知识库
   - 名称：测试法律知识库
   - 向量库：选择配置的向量库
   - 向量模型：选择配置的向量模型
3. 选择知识库 → 上传文档
4. 等待上传完成（后端日志显示切片和向量化）

**验证关键日志**：
```
存储向量数据: kid=X, docId=X, 数据条数=N
```

### 9.7 知识库问答成功

1. 新建对话，选择刚创建的知识库
2. 输入：`民法典中关于侵权责任是怎么规定的？`
3. AI 回答中包含了上传文档中的内容
4. 后端日志显示向量检索结果

**验证关键日志**：
```
查询向量数据: kid=X, query=..., maxResults=N
```

### 9.8 查看日志无明显异常

1. 后端控制台无 ERROR 级别日志
2. 无 `Connection refused` 类连接错误
3. 无 `模型不存在` 类业务错误
4. Docker 容器全部运行中：`docker ps` 显示所有容器 STATUS 为 Up

---

## 10. 常见失败点

### 10.1 JDK 版本不对

| 现象 | 原因 | 解决 |
|------|------|------|
| `UnsupportedClassVersionError` | JDK 版本低于17 | 安装 JDK 17，设置 JAVA_HOME |
| `mvn` 命令提示找不到 | Maven 未安装或未配置 PATH | 安装 Maven 3.9+，配置 PATH |
| 编译报错 `java: 错误: 不支持发行版本 17` | IDEA 编译器设置不对 | IDEA → Settings → Build → Compiler → Java Compiler → Target bytecode version 设为 17 |

### 10.2 端口冲突

| 端口 | 服务 | 排查命令 |
|------|------|---------|
| 6039 | 后端服务 | `netstat -ano | findstr 6039` |
| 3306 | MySQL | `netstat -ano | findstr 3306` |
| 6379 | Redis | `netstat -ano | findstr 6379` |
| 19530 | Milvus | `netstat -ano | findstr 19530` |
| 9000 | MinIO | `netstat -ano | findstr 9000` |

> `RuoYiAIApplication.java` L19 会自动终止占用 6039 端口的进程（仅 Windows），但其他端口需手动处理。

### 10.3 MySQL 连接失败

| 现象 | 原因 | 解决 |
|------|------|------|
| `Communications link failure` | MySQL 未启动或端口不对 | 确认 Docker 容器运行：`docker ps` |
| `Access denied for user` | 密码不对 | 检查 `application-dev.yml` L62-63 |
| `Unknown database 'ruoyi-ai'` | 库名不匹配或未导入SQL | 创建数据库并导入SQL |
| `Public Key Retrieval is not allowed` | MySQL 8.0 SSL 问题 | URL 中已含 `allowPublicKeyRetrieval=true`（L61），确认未被删除 |

### 10.4 Redis 连接失败

| 现象 | 原因 | 解决 |
|------|------|------|
| `Unable to connect to Redis` | Redis 未启动 | `docker ps` 确认 Redis 容器运行 |
| `NOAUTH Authentication required` | Redis 设了密码但配置没配 | `application-dev.yml` L104 密码默认注释，如Redis设了密码需取消注释 |
| 超时 | Redisson 配置的 timeout 太短 | 默认3000ms，一般够用 |

### 10.5 向量库没启动

| 现象 | 原因 | 解决 |
|------|------|------|
| 知识库上传报错 `Connection refused` | Milvus 未启动 | `docker-compose -f docs/docker/milvus/docker-compose.yml up -d` |
| 向量化报错 | Milvus 的 etcd 未就绪 | 等待90秒后再试（Milvus start_period: 90s） |
| `vector-store.type` 与实际不匹配 | 配置了 milvus 但启动了 weaviate | 检查 `application.yml` L297 的值 |
| Milvus Collection 不存在 | 首次使用需自动创建 | 项目代码 `VectorStoreServiceImpl.createSchema()` 会自动创建 |

### 10.6 MinIO 配置错

| 现象 | 原因 | 解决 |
|------|------|------|
| 文件上传失败 | MinIO 未启动 | `docker ps` 确认 |
| `Access Denied` | `sys_oss_config` 表中配置的 accessKey/secretKey 与 MinIO 不一致 | SQL L2724 预置：ruoyi/ruoyi123，MinIO 启动参数必须匹配 |
| Bucket 不存在 | MinIO 未自动创建 bucket | 首次上传时项目会自动创建 bucket（`ruoyi`） |

### 10.7 模型 Key 错误

| 现象 | 原因 | 解决 |
|------|------|------|
| `模型不存在: xxx` | `chat_model` 表中没有对应的 `model_name` | 通过管理端新增模型，或在数据库插入记录 |
| `401 Unauthorized` | API Key 无效 | 确认 Key 正确，检查是否过期 |
| `provider_code` 不匹配 | 数据库中的 `provider_code` 与 `ChatModeType` 枚举不一致 | `provider_code` 必须是：openai/qianwen/zhipu/deepseek/ollama/ppio 之一 |
| `api_host` 格式错 | URL 末尾多了 `/v1` 或少了 `/` | DeepSeek: `https://api.deepseek.com`；OpenAI: `https://api.openai.com` |

### 10.8 前端代理错

| 现象 | 原因 | 解决 |
|------|------|------|
| 前端登录接口404 | 前端代理未指向后端6039 | Docker 模式检查 `UPSTREAM_HOST` 环境变量 |
| 前端白屏 | 前端构建失败或路径不对 | 检查 Docker 容器日志：`docker logs ruoyi-ai-admin` |

### 10.9 跨域问题

| 现象 | 原因 | 解决 |
|------|------|------|
| 浏览器控制台 CORS 报错 | 前端和后端端口不同，跨域配置未生效 | 项目已配置跨域（`ruoyi-common-web`），一般不会出现；Docker 模式通过 Nginx 反向代理解决 |

### 10.10 SSE 流式响应问题

| 现象 | 原因 | 解决 |
|------|------|------|
| 对话请求发出后无响应 | SSE 连接未建立 | 确认 `sse.enabled: true`（`application.yml` L251） |
| 响应一次性返回而非流式 | Nginx/代理缓冲了 SSE 响应 | Docker 模式 Nginx 已配置 `proxy_buffering off`；如用其他代理需手动配置 |
| SSE 连接超时断开 | 代理超时设置太短 | 确保 Nginx `proxy_read_timeout` 大于60s |
| `SseEmitter` 报错 `AsyncRequestTimeoutException` | Spring MVC 异步超时 | 项目已配置 Undertow 服务器，一般不会出现 |

---

## 11. M0 验收标准

以下**全部通过**，即认为 M0 启动试运行通过：

| 序号 | 验收项 | 验收方法 | 通过标准 |
|------|--------|---------|---------|
| 1 | MySQL 正常运行 | `docker exec -it ruoyi-ai-mysql mysql -uroot -proot -e "SHOW DATABASES"` | 能看到 `ruoyi-ai` 库 |
| 2 | Redis 正常运行 | `docker exec -it ruoyi-ai-redis redis-cli ping` | 返回 `PONG` |
| 3 | Milvus 正常运行 | 访问 `http://localhost:19500`（Attu界面） | 能打开 Attu 管理界面 |
| 4 | MinIO 正常运行 | 访问 `http://localhost:9090` | 能打开 MinIO Console，用 ruoyi/ruoyi123 登录 |
| 5 | 后端服务启动 | 查看控制台日志 | 出现 `RuoYi-AI启动成功`，无 ERROR |
| 6 | Swagger 文档可访问 | `http://localhost:6039/doc.html` | 能打开接口文档页面 |
| 7 | 管理端能登录 | `http://localhost:5666` → admin/admin123 | 成功进入管理后台 |
| 8 | 模型配置成功 | 管理端 → 模型管理 → 新增 | 能新增模型且列表可见 |
| 9 | 普通对话成功 | 选择模型 → 发送消息 | AI 流式回复正常，消息保存到数据库 |
| 10 | 知识库创建成功 | 管理端 → 知识库 → 新增 | 知识库记录可见 |
| 11 | 文档上传成功 | 知识库 → 上传 txt/pdf | 上传成功，`knowledge_fragment` 表有切片记录 |
| 12 | 知识库问答成功 | 新对话 → 选择知识库 → 提问 | AI 回答包含文档内容 |

---

## 12. 暂不处理事项

M0 阶段**明确不做**以下事情：

| 事项 | 原因 |
|------|------|
| AI 律师业务二开 | M0 只验证原项目能跑通，二开是后续阶段 |
| 接入腾讯 IM / 即时通讯 | 当前项目无人-人聊天能力，需单独设计 |
| 支付/订单模块 | 与M0验证无关 |
| 生产部署 | M0 是本地开发验证，不做生产化 |
| 本地大模型部署（Ollama） | M0 用云端 API 即可，本地部署增加复杂度 |
| 重构 RAG 检索 | 当前纯向量检索够 M0 验证，优化是后续阶段 |
| 知识图谱验证 | 模块较新且非M0必需，跳过 |
| MCP 工具市场 | 非M0必需，跳过 |
| 多智能体（Supervisor模式） | 需要额外配置 MCP 客户端且代码硬编码了Windows路径，跳过 |
| 前端源码修改 | M0 用 Docker 镜像或原始代码即可，不改前端 |
| 性能测试 | M0 只验证功能可用性 |
| 安全加固 | M0 不考虑安全审计 |

---

> 本文档所有结论均基于 tag 3.0.0 版本源码分析，引用了具体的文件路径和行号。
> 本文档不包含任何需要执行的命令的执行动作，仅作为启动前准备参考。
