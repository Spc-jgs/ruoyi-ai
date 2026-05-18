# StreamingAgent 流式输出机制详解

> 源码入口：`ruoyi-chat/src/test/java/org/ruoyi/agent/StreamingAgentIntegrationTest.java`
>
> 核心依赖类：`OutputChannel`、`StreamingOutputWrapper`、`SupervisorStreamListener`

---

## 1. 整体思路

本测试验证的是 **LangChain4j Agentic 框架中 Agent 的流式输出能力**，核心流程：

```
用户调用 Agent 方法
  → LangChain4j 动态代理把注解拼成 Prompt
  → 发送 HTTP 请求到 OpenRouter API（SSE 流式）
  → AI 逐 token 返回
  → StreamingOutputWrapper 拦截每个 token，推入 OutputChannel
  → 主线程通过 channel.drain() 实时消费并打印
```

---

## 2. 关键类职责

### 2.1 Agent 接口定义（声明式编程）

```java
public interface MathAgent {
    @SystemMessage("你是一个数学计算助手...")    // 角色/系统提示词
    @UserMessage("计算：{{query}}")              // 用户消息模板
    @dev.langchain4j.agentic.Agent("数学计算助手") // 标记为 Agent
    String calculate(@V("query") String query);   // 调用方法 = 调 AI
}
```

| 注解 | 作用 |
|------|------|
| `@SystemMessage` | 对应 OpenAI API 的 `system` 角色，定义 AI 的行为准则 |
| `@UserMessage` | 对应 `user` 角色，`{{query}}` 会被方法参数替换 |
| `@Agent` | LangChain4j Agentic 标记，使接口可被 `AgenticServices` 识别 |
| `@V` | 绑定方法参数到模板中的占位符 |

**无需手写实现类**，`AgenticServices.agentBuilder(MathAgent.class).build()` 会通过 JDK 动态代理生成实现。

### 2.2 OutputChannel — 线程间消息管道

```
生产者线程（AI 调用）  ──send()──▶  BlockingQueue  ──drain()──▶  消费者线程（打印/SSE推送）
```

| 方法 | 作用 |
|------|------|
| `create(requestId)` | 创建并注册到全局 `REGISTRY`（ConcurrentHashMap） |
| `send(text)` | 非阻塞写入队列（队列满则丢弃，超时 100ms） |
| `drain(consumer)` | 阻塞消费队列，逐条回调 consumer，收到 `__DONE__` 终止 |
| `complete()` | 标记完成，往队列塞入 `__DONE__` 信号 |
| `completeWithError(t)` | 标记错误完成，记录异常 + 塞入 `__DONE__` |
| `remove(requestId)` | 从全局注册表移除，防止内存泄漏 |

### 2.3 StreamingOutputWrapper — 流式拦截器

实现了 `StreamingChatModel` 和 `ChatModel` 两个接口，本质是一个 **装饰器**：

```
streamingModel（原始模型）
    ↓ 包装
StreamingOutputWrapper
    ↓ 效果
每次收到 token → ① 推入 OutputChannel  ② 传给原始 handler
```

关键方法：
- **`chat(ChatRequest)`（同步接口）**：内部调用 `streamingDelegate.chat(request, handler)`，用 `CompletableFuture` 等待流式完成，最后返回完整 `ChatResponse`。Agent 通过 `.chatModel(wrappedModel)` 走的就是这条路。
- **`chat(ChatRequest, StreamingChatResponseHandler)`（流式接口）**：用 `wrapHandler()` 包装原始 handler，在每个回调中额外 `channel.send()`。

### 2.4 SupervisorStreamListener — Agent 生命周期监听

实现 `AgentListener` 接口，用于监控 Supervisor 模式下的 Agent 调用：

| 回调 | 作用 |
|------|------|
| `beforeAgentInvocation` | Agent 开始执行前（仅日志） |
| `afterAgentInvocation` | Agent 执行完成后（仅日志） |
| `onAgentInvocationError` | Agent 执行出错时 → 推送到 OutputChannel |
| `inheritedBySubagents` | 返回 `true`，监听器自动继承给所有子 Agent |

---

## 3. 四个测试场景详解

### 3.1 测试1：基础流式输出 — 单个 Agent

```
testBasicStreamingAgent()
│
├── 1. 创建 OutputChannel + CountDownLatch
│
├── 2. 用 StreamingOutputWrapper 包装 streamingModel
│      → wrappedModel = new StreamingOutputWrapper(streamingModel, channel)
│
├── 3. 构建 Agent
│      → AgenticServices.agentBuilder(MathAgent.class)
│          .chatModel(wrappedModel)    // 关键：用包装后的模型
│          .build()
│
├── 4. CompletableFuture.runAsync() — 异步线程执行
│      → mathAgent.calculate("计算 123 * 456 + 789 的值")
│         │
│         ├── 代理对象将 @SystemMessage + @UserMessage 拼成请求
│         ├── wrappedModel.chat(request) 被调用
│         ├── 内部 streamingModel.chat(request, handler) 发 SSE 请求
│         ├── 每收到 token → onPartialResponse → channel.send(token)
│         └── 完成 → onCompleteResponse → channel.complete()
│
└── 5. channel.drain(text -> System.out.print(text)) — 主线程消费
       → 阻塞等待，逐 token 打印，收到 DONE 退出
```

**线程交互时序图**：

```
异步线程                                    主线程
  │                                          │
  ├─ calculate("计算...")                    │
  ├─ HTTP SSE 请求 → OpenRouter              │
  ├─ 收到 token "1" → channel.send ────────▶ │ drain 取出 → 打印 "1"
  ├─ 收到 token "2" → channel.send ────────▶ │ drain 取出 → 打印 "2"
  ├─ ...                                     ├─ ...
  ├─ 完成 → channel.complete() ─────────────▶ │ drain 收到 DONE → 退出
  ├─ completed.countDown()                   │
  │                                          ├─ completed.await() 返回
  │                                          ├─ OutputChannel.remove()
```

### 3.2 测试2：Supervisor + 单子 Agent

```
testSupervisorWithSingleSubAgent()
│
├── 1. 构建子 Agent（用 streamingChatModel，不包装）
│      → AgenticServices.agentBuilder(MathAgent.class)
│          .streamingChatModel(streamingModel)
│          .build()
│
├── 2. 构建 Supervisor（用 syncModel 决策）
│      → AgenticServices.supervisorBuilder()
│          .chatModel(syncModel)                     // Supervisor 决策用同步模型
│          .subAgents(mathAgent)                      // 注册子 Agent
│          .responseStrategy(SupervisorResponseStrategy.LAST)
│          .build()
│
├── 3. 异步调用 supervisor.invoke("帮我计算 999 除以 3")
│      │
│      ├── Supervisor 用 syncModel 判断：这是数学问题 → 路由给 MathAgent
│      ├── MathAgent 用 streamingModel 流式生成
│      └── 返回最终结果
│
└── 4. channel.drain() 消费输出
```

**Supervisor 模式说明**：Supervisor 本身用同步模型做"调度决策"（判断该路由给哪个子 Agent），子 Agent 用流式模型做"内容生成"。

### 3.3 测试3：Supervisor + 多子 Agent

与测试2 类似，区别是注册了 3 个子 Agent（MathAgent、TextAgent、WeatherAgent），并启用了 `SupervisorStreamListener` 来监控 Agent 生命周期。

Supervisor 会根据用户提问自动路由到合适的 Agent。

### 3.4 测试4：直接使用 StreamingChatModel

最简单的用法，不经过 Agent 封装，直接调用 `streamingModel.chat()` 并传入 `StreamingChatResponseHandler`：

```java
streamingModel.chat("你好，请自我介绍", new StreamingChatResponseHandler() {
    void onPartialResponse(String token)  { /* 每个 token */ }
    void onCompleteResponse(ChatResponse) { /* 完成 */ }
    void onError(Throwable error)         { /* 出错 */ }
});
```

---

## 4. 模型初始化要点

```java
// 流式模型 — 必须指定 modelName！
streamingModel = OpenAiStreamingChatModel.builder()
    .baseUrl(BASE_URL)       // OpenRouter 兼容 OpenAI 协议
    .apiKey(API_KEY)
    .modelName(MODEL_NAME)   // ⚠️ 必填，否则报 "No models provided" 400 错误
    .listeners(...)          // 可选：请求/响应/错误监听
    .build();

// 同步模型
syncModel = OpenAiChatModel.builder()
    .baseUrl(BASE_URL)
    .apiKey(API_KEY)
    .modelName(MODEL_NAME)
    .build();
```

> **常见坑**：OpenRouter API 要求请求体必须包含 `model` 字段。如果 `OpenAiStreamingChatModel.builder()` 漏掉 `.modelName()`，请求不会携带 model 参数，API 返回 `{"error":{"message":"No models provided","code":400}}`。

---

## 5. 关键设计模式总结

| 设计模式 | 在代码中的体现 |
|---------|--------------|
| **装饰器模式** | `StreamingOutputWrapper` 包装 `StreamingChatModel`，增加 channel 推送能力 |
| **代理模式** | LangChain4j 用 JDK 动态代理为 Agent 接口生成实现类 |
| **生产者-消费者** | `OutputChannel` 的 BlockingQueue 连接 AI 线程和主线程 |
| **观察者模式** | `ChatModelListener` 和 `AgentListener` 监听模型/Agent 生命周期事件 |
| **Builder 模式** | 所有 LangChain4j 模型类都通过 Builder 构建配置 |

---

## 6. 类关系图

```
StreamingAgentIntegrationTest
  │
  ├── StreamingChatModel (OpenAiStreamingChatModel)
  │     └── 被 StreamingOutputWrapper 装饰
  │           ├── 实现 StreamingChatModel 接口（流式）
  │           └── 实现 ChatModel 接口（同步包装流式）
  │
  ├── OpenAiChatModel (syncModel)
  │     └── Supervisor 用，做调度决策
  │
  ├── OutputChannel
  │     ├── BlockingQueue<String>  ── 线程间通信
  │     └── static REGISTRY        ── 全局注册表
  │
  ├── AgenticServices
  │     ├── .agentBuilder()   ── 构建单个 Agent
  │     └── .supervisorBuilder() ── 构建 Supervisor Agent
  │
  └── SupervisorStreamListener
        └── 实现 AgentListener ── 监控 Supervisor 下 Agent 生命周期
```
