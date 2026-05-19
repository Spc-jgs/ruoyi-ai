# MCP 工具加载与 LangChain4j Agent 调用机制

> 版本：v1.0
> 日期：2026-05-19
> 适用阶段：M0/M1 架构摸底
> 目的：帮助 AI 开发初学者理解 `ruoyi-chat` 模块中 MCP、`@Tool`、`ToolProvider`、Agent 的关系

---

## 1. 一句话理解

`ruoyi-chat` 里的 MCP 代码，本质是在做一件事：

```
把数据库里的 MCP 工具配置
  → 转成 LangChain4j 能识别的工具集合
  → 让 Agent 在和模型对话时可以动态调用这些工具
```

它不是 AI 对话模型本身，而是 **给 Agent/Chat 挂工具的适配层**。

---

## 2. 整体链路

```
管理端配置工具
  → mcp_tool_info 表保存工具信息
  → LangChain4jMcpToolProviderService 读取 ENABLED 工具
  → 按工具类型创建 McpClient
  → McpToolProvider 包装 McpClient
  → AgenticServices.agentBuilder(...).toolProvider(...)
  → Agent 调用模型时携带工具 schema
  → 模型返回 tool call
  → LangChain4j 执行对应 ToolExecutor
  → 工具结果回填给模型
  → 模型生成最终回答
```

核心类：

| 类 | 位置 | 作用 |
|----|------|------|
| `McpTool` | `domain/entity/mcp` | 对应 `mcp_tool_info`，保存本地工具配置 |
| `McpToolController` | `controller/mcp` | MCP 工具管理接口 |
| `McpToolServiceImpl` | `service/mcp/impl` | 工具增删改查、状态切换、测试连接 |
| `LangChain4jMcpToolProviderService` | `mcp/service/core` | 从数据库配置创建 `McpClient` 和 `ToolProvider` |
| `BuiltinToolRegistry` | `mcp/service/core` | 扫描项目内置 Java 工具 |
| `ToolProviderFactory` | `mcp/service/core` | 对外提供统一工具入口 |

---

## 3. 工具类型

`McpTool.type` 目前有三类：

| 类型 | 含义 | 配置方式 | 运行方式 |
|------|------|----------|----------|
| `BUILTIN` | 项目内置 Java 工具 | Java 类 + `@Tool` | 直接反射调用 Java 方法 |
| `LOCAL` | 本地 MCP Server | `command` + `args` | 启动本地子进程，通过 stdin/stdout 通信 |
| `REMOTE` | 远程 MCP Server | `baseUrl` | 通过 HTTP/SSE 连接远程服务 |

### 3.1 BUILTIN

内置工具是当前 Java 项目里的方法，例如：

```java
@Tool("Execute a SELECT SQL query and return the results. Example: SELECT * FROM sys_user")
public String executeSql(String sql) {
    ...
}
```

运行时，LangChain4j 会反射扫描 `@Tool` 方法，生成：

```
ToolSpecification  工具说明：名称、描述、参数 schema
ToolExecutor       工具执行器：真正调用 Java 方法
```

项目中的例子：

| 工具类 | 作用 |
|--------|------|
| `ReadFileTool` | 读取工作区文件 |
| `ListDirectoryTool` | 列出目录 |
| `EditFileTool` | 编辑文件 |
| `QueryAllTablesTool` | 查询所有数据表 |
| `QueryTableSchemaTool` | 查询表结构 |
| `ExecuteSqlQueryTool` | 执行 SELECT 查询 |

### 3.2 LOCAL

`LOCAL` 类型配置的不是 URL，而是 **启动本地 MCP Server 的命令**。

示例配置：

```json
{
  "command": "npx",
  "args": ["-y", "@modelcontextprotocol/server-filesystem", "/tmp/workspace"]
}
```

等价于在终端执行：

```bash
npx -y @modelcontextprotocol/server-filesystem /tmp/workspace
```

这个命令启动后，它本身就是一个 MCP Server。Java 进程通过标准输入输出和它通信：

```
RuoYi-AI Java 进程
  stdin/stdout JSON-RPC
本地 MCP Server 子进程
```

底层逻辑在 LangChain4j 的 `StdioMcpTransport` 中：

```java
ProcessBuilder processBuilder = new ProcessBuilder(command);
Process process = processBuilder.start();

InputStream inputStream = process.getInputStream();
OutputStream outputStream = process.getOutputStream();
```

本项目创建 STDIO MCP Client 的代码在：

```java
McpTransport transport = StdioMcpTransport.builder()
    .command(fullCommand)
    .logEvents(true)
    .build();

return new DefaultMcpClient.Builder()
    .transport(transport)
    .build();
```

对应类：`LangChain4jMcpToolProviderService#createStdioClient`

### 3.3 REMOTE

`REMOTE` 类型表示 MCP Server 已经在远程或本机某个端口运行好了，Java 只需要连接它。

示例配置：

```json
{
  "baseUrl": "http://localhost:8080/mcp"
}
```

运行链路：

```
RuoYi-AI Java 进程
  HTTP/SSE
远程 MCP Server
```

对应代码：

```java
McpTransport transport = StreamableHttpMcpTransport.builder()
    .url(baseUrl)
    .logRequests(true)
    .build();
```

---

## 4. `McpClient` 是什么

`McpClient` 可以理解成 **连接某个 MCP Server 的客户端对象**。

它不等于工具本身，而是负责：

| 能力 | 说明 |
|------|------|
| `listTools()` | 向 MCP Server 查询有哪些工具 |
| `executeTool(...)` | 模型决定调用工具后，把调用请求发给 MCP Server |
| `listResources()` | 查询 MCP Resource |
| `readResource(...)` | 读取 MCP Resource |
| `checkHealth()` | 检查连接是否健康 |

所以：

```
McpClient = 和某个 MCP Server 通信的连接器
```

一个 `McpToolProvider` 可以持有多个 `McpClient`：

```java
ToolProvider toolProvider = McpToolProvider.builder()
    .mcpClients(List.of(client1, client2))
    .build();
```

---

## 5. `@Tool`、`ToolProvider`、`McpToolProvider` 的关系

### 5.1 最终统一模型

LangChain4j 最终管理工具时，核心是两个对象：

| 对象 | 含义 |
|------|------|
| `ToolSpecification` | 给模型看的工具说明，包括名称、描述、参数 schema |
| `ToolExecutor` | 给程序用的工具执行器，负责真正执行工具 |

不管工具来自哪里，最后都要变成：

```
ToolSpecification + ToolExecutor
```

### 5.2 `@Tool`

`@Tool` 是把 Java 方法声明成工具。

```
Java 对象
  → 扫描 @Tool 方法
  → 生成 ToolSpecification
  → 生成 DefaultToolExecutor
  → 调用时反射执行 Java 方法
```

适合项目内置工具。

### 5.3 `ToolProvider`

`ToolProvider` 是 LangChain4j 的工具供应接口：

```java
public interface ToolProvider {
    ToolProviderResult provideTools(ToolProviderRequest request);
}
```

它返回的是：

```java
Map<ToolSpecification, ToolExecutor>
```

也就是说：

```
ToolProvider = 一个工具集合供应商
```

### 5.4 `McpToolProvider`

`McpToolProvider` 是 LangChain4j 提供的 `ToolProvider` 实现，专门用于 MCP。

它内部大致逻辑：

```java
for (McpClient client : mcpClients) {
    for (ToolSpecification spec : client.listTools()) {
        result.add(spec, new McpToolExecutor(client, spec.name()));
    }
}
```

所以：

```
McpClient
  → 负责连接 MCP Server

McpToolProvider
  → 负责把 MCP Server 暴露的工具转成 LangChain4j 工具

ToolProvider
  → LangChain4j Agent 统一使用的工具入口
```

---

## 6. Agent 是什么

本项目里的 Agent 主要是 LangChain4j Agentic 的声明式接口，例如：

```java
public interface SqlAgent {

    @SystemMessage("...")
    @UserMessage("Answer the following question: {{query}}")
    @Agent("Intelligent database query assistant")
    String getData(@V("query") String query);
}
```

`@Agent` 表示这个接口方法是一个 Agent 动作。

真正生成可调用实现的是：

```java
SqlAgent sqlAgent = AgenticServices.agentBuilder(SqlAgent.class)
    .chatModel(plannerModel)
    .tools(new QueryAllTablesTool(), new QueryTableSchemaTool(), new ExecuteSqlQueryTool())
    .build();
```

这个 `build()` 会创建代理对象。调用：

```java
sqlAgent.getData("查一下用户数量");
```

并不是执行普通 Java 业务逻辑，而是进入 LangChain4j 的 Agent 调用链：

```
接口方法调用
  → 拼接 SystemMessage / UserMessage
  → 收集工具
  → 调用模型
  → 处理 tool call
  → 返回最终结果
```

---

## 7. Agent 是否带着所有工具和模型对话

不是全系统所有工具，而是 **当前 Agent 配置了哪些工具，就带哪些工具**。

项目中示例：

```java
WebSearchAgent searchAgent = AgenticServices.agentBuilder(WebSearchAgent.class)
    .chatModel(plannerModel)
    .toolProvider(toolProvider)
    .build();
```

`WebSearchAgent` 看到的是 `toolProvider` 提供的工具。

```java
SqlAgent sqlAgent = AgenticServices.agentBuilder(SqlAgent.class)
    .chatModel(plannerModel)
    .tools(new QueryAllTablesTool(), new QueryTableSchemaTool(), new ExecuteSqlQueryTool())
    .build();
```

`SqlAgent` 看到的是这三个 SQL 工具。

如果一个 Agent 配了 20 个工具，模型请求里通常就会携带这 20 个工具的 schema。模型根据用户问题决定是否调用工具。

模型不会直接执行工具，它只会返回类似：

```json
{
  "name": "executeSql",
  "arguments": {
    "sql": "select count(*) from sys_user"
  }
}
```

LangChain4j 收到后，根据工具名找到对应 `ToolExecutor`，在 Java 侧执行。

---

## 8. LangChain4j 工具调用循环

简化后的底层循环：

```
1. 构建 Agent / AiService
2. 收集工具
   - tools(new XxxTool()) 扫描 @Tool
   - toolProvider(...) 调 provideTools()
3. 得到：
   - List<ToolSpecification>
   - Map<toolName, ToolExecutor>
4. 发请求给模型时，把 ToolSpecification 带上
5. 模型返回：
   - 普通文本：直接结束
   - tool call：进入工具执行
6. LangChain4j 根据 toolName 找 ToolExecutor
7. 执行工具
   - @Tool 工具：DefaultToolExecutor 反射调用 Java 方法
   - MCP 工具：McpToolExecutor 调 McpClient.executeTool()
8. 工具结果包装成 ToolExecutionResultMessage
9. 再次请求模型
10. 模型基于工具结果生成最终回答
```

可以用一个比喻帮助理解：

```
ToolSpecification = 菜单
ToolExecutor      = 厨房
模型              = 点菜的人
LangChain4j       = 服务员，负责把点菜单交给厨房，再把菜端回来
```

---

## 9. 和本项目代码的对应关系

### 9.1 外部 MCP 工具加载

入口：

```java
ToolProviderFactory#getAllEnabledMcpToolsProvider()
```

调用：

```java
LangChain4jMcpToolProviderService#getAllEnabledToolsProvider()
```

流程：

```
查询 mcp_tool_info 中 ENABLED 工具
  → getToolProvider(toolIds)
  → getOrCreateClient(toolId)
  → createMcpClient(tool)
  → LOCAL: createStdioClient(tool)
  → REMOTE: createRemoteClient(tool)
  → McpToolProvider.builder().mcpClients(clients).build()
```

### 9.2 内置工具注册

启动时：

```java
BuiltinToolRegistry#init()
```

会扫描所有实现 `BuiltinToolProvider` 的 Spring Bean。

然后：

```java
SystemToolInitializer#run()
```

会把这些内置工具同步到 `mcp_tool_info` 表，便于统一管理启用/禁用状态。

注意：`BUILTIN` 类型不会创建 `McpClient`。在 `LangChain4jMcpToolProviderService#getOrCreateClient` 中会跳过：

```java
if ("BUILTIN".equals(tool.getType())) {
    return null;
}
```

### 9.3 思考模式中的示例 Agent

`ChatServiceFacade#handleThinkingMode` 中创建了多个子 Agent：

| Agent | 工具来源 |
|-------|----------|
| `WebSearchAgent` | `toolProvider(toolProvider)`，来自 MCP |
| `SkillsAgent` | `skills.toolProvider()` |
| `SqlAgent` | `.tools(new QueryAllTablesTool(), ...)` |
| `ChartGenerationAgent` | 无工具，仅生成图表配置 |
| `EchartsAgent` | SQL 工具 |

最后由：

```java
AgenticServices.supervisorBuilder()
    .subAgents(skillsAgent, searchAgent, sqlAgent, chartGenerationAgent, echartsAgent)
    .build();
```

组成 Supervisor Agent。

---

## 10. 当前代码里的注意点

### 10.1 `env` 配置暂未传入 STDIO Transport

`McpTool.configJson` 注释中支持：

```json
{
  "command": "npx",
  "args": ["-y", "@example/mcp-server"],
  "env": {}
}
```

但当前 `createStdioClient` 只解析了 `command` 和 `args`，没有把 `env` 传给：

```java
StdioMcpTransport.builder().environment(...)
```

如果后续 MCP Server 需要 API Key 等环境变量，这里可能需要补。

### 10.2 思考模式中存在 Windows 路径硬编码

`ChatServiceFacade#handleThinkingMode` 中有：

```java
"C:\\Program Files\\nodejs\\npx.cmd"
```

在 macOS 环境下这段无法直接运行。当前阶段可以先作为源码理解，不建议在 M0/M1 直接重构核心链路。

### 10.3 工具越多不一定越好

模型请求会携带工具 schema。工具太多会带来：

| 风险 | 说明 |
|------|------|
| Token 增加 | 工具名称、描述、参数 schema 会进入模型上下文 |
| 选择困难 | 模型可能误选工具 |
| 权限风险 | 文件、SQL、浏览器类工具需要特别谨慎 |
| 调试复杂 | 工具调用链变长，问题定位更难 |

建议后续按 Agent 职责拆分工具，而不是给每个 Agent 挂全量工具。

---

## 11. 初学者阅读顺序

建议按下面顺序读源码：

1. `McpTool`：先理解数据库保存了哪些工具字段
2. `McpToolController`：看管理端暴露了哪些接口
3. `McpToolServiceImpl`：看工具 CRUD、测试、状态更新
4. `LangChain4jMcpToolProviderService`：看数据库配置如何变成 `McpClient`
5. `McpToolProvider`：理解 MCP 工具如何变成 `ToolProvider`
6. `BuiltinToolRegistry`：理解内置 `@Tool` 如何注册
7. `ChatServiceFacade#handleThinkingMode`：看 Agent 如何使用这些工具

读源码时抓住这条主线即可：

```
配置在哪里
  → 谁读取配置
  → 谁创建连接
  → 谁生成工具说明
  → 谁执行工具
  → 结果怎么回到模型
```

