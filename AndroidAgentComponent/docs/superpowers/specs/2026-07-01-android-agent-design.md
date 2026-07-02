# Android Agent 组件设计文档

- 日期: 2026-07-01
- 状态: 已批准(设计阶段)
- 技能: brainstorming

## 1. 概述

构建一个基于 Android 的 Agent 组件库,以 Gradle 多模块形式发布,供宿主项目按需引入。核心能力:

- **工具调用 (Tool Use)**: Agent 可调用注册的工具执行任务
- **记忆与 RAG**: 用于实现用户自定义角色(预设 → 自定义演进),角色知识库检索
- **云端 LLM**: 首选国产模型(OpenAI 兼容 API)
- **无 UI 耦合**: 纯逻辑组件,宿主可在任意作用域收集事件流

## 2. 技术选型决策

| 决策项 | 选择 | 理由 |
|--------|------|------|
| 框架组合 | **ADK for Android (主) + LangChain4j (RAG 数据层)** | ADK 是 Android 原生一等公民,KSP 零反射工具、Session/Runner 现成;LangChain4j RAG 全链路最成熟,作数据层不冲突 |
| LLM | 国产模型 (DeepSeek/通义/GLM/Moonshot) 走 OpenAI 兼容 API | 用户首选国产;ADK 支持 OpenAI 兼容 model 接入 |
| **LLM 接入** | **用户自定义 (BYO Key)** | LLM 由用户自行接入自己的 API key,组件不绑定任何提供商;Phase 1 先用内置预设 Provider 跑通,Phase 2 开放用户配置入口 |
| 语言 | Kotlin + Coroutines/Flow | 现代安卓首选,异步天然契合 |
| 集成形式 | Gradle Library 多模块 | 按需引入、可替换实现 |
| APK 体积 | 不敏感 | 优先开发效率 |

### 2.1 国产模型 + ADK 适配路径

ADK 原生首选 Gemini,而国产模型通过 OpenAI 兼容 API 接入。路径:

1. `agent-llm` 模块封装 OpenAI 兼容 Provider,`baseUrl` 指向国产模型端点 (如 `https://api.deepseek.com/v1`)
2. 对上适配 ADK Kotlin 的 `Model` 接口,使 `LlmAgent` 可透明使用
3. 若 ADK 0.1.0-beta 对 OpenAI 兼容支持不完善,则 `agent-llm` 基于 okhttp 自实现轻量客户端,实现 ADK `Model` 接口
4. 抽象层 `LlmProvider` 接口挡在前面,未来可替换为 Gemini 或端侧

## 3. 模块划分

```
agent-android/
├── settings.gradle.kts
├── build.gradle.kts
├── gradle/libs.versions.toml
├── agent-core/          # 门面 API: Agent、AgentEvent、Builder (封装 ADK Runner/Session)
├── agent-llm/           # LLM Provider 适配: 国产模型 OpenAI 兼容 → ADK Model
├── agent-tools/         # @Tool DSL + 内置工具集 (基于 ADK KSP)
├── agent-memory/        # PersonaStore 预设角色 + 会话记忆 (基于 ADK Session)
├── agent-rag/           # LangChain4j EmbeddingStore + Document 管道 (角色知识库)
└── sample/              # Demo App 验证集成 (可选)
```

宿主集成: `implementation("com.you.agent:agent-core:0.1")` + 按需加 `agent-llm`/`agent-rag` 等。宿主**只依赖 `agent-core` 接口**,不直接依赖 ADK/LangChain4j。

## 4. 核心抽象 (agent-core, Kotlin)

```kotlin
/** LLM 提供方抽象,挡在 ADK Model 之前 */
interface LlmProvider {
    val modelId: String
    suspend fun chat(req: ChatRequest): ChatResponse
    fun streamChat(req: ChatRequest): Flow<ChatChunk>
}

/** 角色 */
data class Persona(
    val id: String,
    val name: String,
    val systemPrompt: String,
    val traits: Map<String, String> = emptyMap(),
    val knowledgeSources: List<String> = emptyList(),  // RAG 检索源
    val tools: List<String> = emptyList()              // 启用的工具名
)
interface PersonaStore {
    suspend fun load(id: String): Persona
    fun list(): List<Persona>
    suspend fun save(persona: Persona)  // Phase 2 自定义角色
}

/** Agent 事件流 */
sealed interface AgentEvent {
    data class Thought(val text: String) : AgentEvent
    data class ToolCall(val name: String, val args: Map<String, Any?>) : AgentEvent
    data class ToolResult(val name: String, val output: String) : AgentEvent
    data class Answer(val text: String) : AgentEvent
    data class Error(val cause: Throwable) : AgentEvent
}

/** Agent 门面,内部委托 ADK Runner */
interface Agent {
    fun run(userInput: String, personaId: String? = null): Flow<AgentEvent>
}

/** 构建 DSL */
class AgentBuilder {
    fun llm(provider: LlmProvider): AgentBuilder
    fun persona(persona: Persona): AgentBuilder
    fun memory(store: PersonaStore): AgentBuilder
    fun rag(retriever: KnowledgeRetriever): AgentBuilder
    fun tools(block: ToolRegistryScope.() -> Unit): AgentBuilder
    fun config(block: AgentConfig.() -> Unit): AgentBuilder
    fun build(): Agent
}
```

## 5. Agent 运行循环

内部委托 ADK `InMemoryRunner` + `LlmAgent`,ReAct 循环由 ADK 提供。`agent-core` 负责:

1. 组装上下文: `persona.systemPrompt` + RAG 检索(`persona.knowledgeSources`) + 会话记忆
2. 构造 ADK `LlmAgent`(model = 适配后的国产模型, instruction = persona, tools = 注册工具)
3. 调用 `runner.runAsync()`,将 ADK 事件转换为 `AgentEvent` Flow
4. 最终答案落盘到会话记忆

全异步基于 Coroutines/Flow,无 UI 耦合。

## 6. 角色与记忆: Preset → Custom 演进

- **Phase 1 (预设)**: `agent-memory` 内置 JSON 预设角色(`assets/personas/*.json`),`PersonaStore` 默认实现只读加载;会话记忆用 ADK `InMemorySessionService` (Phase 1) → 持久化 SessionService (Phase 1.5)
- **Phase 2 (自定义)**: `PersonaStore` 增加可写实现(基于 DataStore/Room),开放 `save(persona)`;`agent-rag` 接入 Embedding API + EmbeddingStore 做角色知识检索
- 接口不动,仅替换实现 → 宿主代码零改动升级

## 7. RAG 设计 (agent-rag, LangChain4j)

```kotlin
interface KnowledgeRetriever {
    suspend fun retrieve(query: String, persona: Persona, topK: Int = 4): List<KnowledgeChunk>
}

/** 默认实现: LangChain4j EmbeddingStoreContentRetriever 封装 */
class LangChain4jRetriever(
    private val embeddingModel: EmbeddingModel,        // 国产模型 OpenAI 兼容 embedding
    private val store: EmbeddingStore<TextSegment>,    // 默认 InMemoryEmbeddingStore, 可换
    private val ingestor: EmbeddingStoreIngestor
) : KnowledgeRetriever
```

- 文档加载/分割用 LangChain4j `DocumentSplitters.recursive`
- 向量库默认 `InMemoryEmbeddingStore`(Phase 1),可换 sqlite-vss/远程(Phase 2)
- 检索结果作为 context 注入 persona system prompt

## 8. LLM Provider 适配 (agent-llm)

### 8.1 设计原则:BYO Key (用户自带 Key)

LLM 完全由用户配置,**组件本身不绑定任何 LLM 提供商**。`agent-llm` 只提供:
- 一套 `LlmProvider` 抽象接口(已定义于 agent-core)
- 一批开箱即用的 OpenAI 兼容 Provider 工厂(用户填 key 即可)
- 一个 `UserLlmConfig` 入口,供宿主项目暴露给最终用户做运行时配置(Phase 2)

组件**不内置任何 API key,不预设默认模型**。没有用户配置时,Agent 构建阶段即报错,绝不静默使用默认 Provider。

### 8.2 内置 Provider 工厂 (Phase 1, 用户填 key)

```kotlin
/** 用户自定义 LLM 配置,宿主可暴露给最终用户填写 */
data class UserLlmConfig(
    val providerId: String,           // "deepseek" / "qwen" / "glm" / "moonshot" / "custom"
    val apiKey: String,               // 用户自有 key
    val baseUrl: String? = null,      // custom 模式必填;预设模式可覆盖默认 baseUrl
    val model: String? = null,        // 可选,覆盖默认模型名
    val temperature: Double? = null,
    val maxTokens: Int? = null,
)

/** 国产模型 OpenAI 兼容 Provider */
class OpenAiCompatibleProvider(
    private val baseUrl: String,     // https://api.deepseek.com/v1
    private val apiKey: String,
    private val model: String,       // deepseek-chat / qwen-plus / glm-4 / moonshot-v1-8k
) : LlmProvider {
    // 1. 优先尝试 ADK 的 OpenAI 兼容 Model 适配
    // 2. 不完善则 okhttp 自实现, 实现 ADK Model 接口
}

object Providers {
    fun deepSeek(key: String) = OpenAiCompatibleProvider("https://api.deepseek.com/v1", key, "deepseek-chat")
    fun qwen(key: String)     = OpenAiCompatibleProvider("https://dashscope.aliyuncs.com/compatible-mode/v1", key, "qwen-plus")
    fun glm(key: String)      = OpenAiCompatibleProvider("https://open.bigmodel.cn/api/paas/v4", key, "glm-4")
    fun moonshot(key: String) = OpenAiCompatibleProvider("https://api.moonshot.cn/v1", key, "moonshot-v1-8k")
    /** 完全自定义端点(用户自建/其他兼容服务) */
    fun custom(baseUrl: String, key: String, model: String) = OpenAiCompatibleProvider(baseUrl, key, model)
}

/** 从用户配置构造 Provider 的统一入口 */
object LlmFactory {
    fun from(config: UserLlmConfig): LlmProvider = when (config.providerId) {
        "deepseek" -> OpenAiCompatibleProvider(
            config.baseUrl ?: "https://api.deepseek.com/v1",
            config.apiKey, config.model ?: "deepseek-chat")
        "qwen"     -> OpenAiCompatibleProvider(
            config.baseUrl ?: "https://dashscope.aliyuncs.com/compatible-mode/v1",
            config.apiKey, config.model ?: "qwen-plus")
        "glm"      -> OpenAiCompatibleProvider(
            config.baseUrl ?: "https://open.bigmodel.cn/api/paas/v4",
            config.apiKey, config.model ?: "glm-4")
        "moonshot" -> OpenAiCompatibleProvider(
            config.baseUrl ?: "https://api.moonshot.cn/v1",
            config.apiKey, config.model ?: "moonshot-v1-8k")
        "custom"   -> OpenAiCompatibleProvider(
            requireNotNull(config.baseUrl) { "custom provider 必须提供 baseUrl" },
            config.apiKey, requireNotNull(config.model) { "custom provider 必须提供 model" })
        else -> error("未知 providerId: ${config.providerId}")
    }
}
```

### 8.3 Phase 2: 运行时动态切换

- 宿主可暴露 `UserLlmConfig` 的录入 UI(组件不提供)
- 用户可在运行时切换 Provider,无需重建 Agent: `agent.reconfigure(LlmFactory.from(newConfig))`
- key 持久化由宿主决定存储位置(推荐 EncryptedSharedPreferences),组件只接收明文 key,不负责加密

### 8.4 预留扩展点

- `LlmProvider` 接口对用户开放,用户可实现非 OpenAI 兼容的 Provider(如直接调 Gemini 原生 SDK)
- 端侧模型(Phase 3): 通过 `OnDeviceProvider` 实现 `LlmProvider`,接入 Gemini Nano

## 9. 集成方式 (宿主视角)

```kotlin
// 用户的 LLM 配置 (宿主可从 EncryptedSharedPreferences 读取)
val userConfig = UserLlmConfig(
    providerId = "deepseek",
    apiKey = readUserKey(),        // 用户填入的 key
    // baseUrl / model 留空则用默认
)

// Application 初始化一次
val agent = Agent.builder()
    .llm(LlmFactory.from(userConfig))   // BYO Key 入口
    .persona(PersonaStore.preset("assistant_v1"))
    .rag(LangChain4jRetriever(embeddingModel, store, ingestor))
    .tools {
        +tool("get_weather") { city -> ... }
        +tool("send_sms") { to, text -> ... }
    }
    .config { maxSteps = 8 }
    .build()

// Phase 2: 用户运行时切换 Provider,无需重建 Agent
// agent.reconfigure(LlmFactory.from(newUserConfig))

// 使用
agent.run("今天北京天气怎样?").collect { event ->
    when (event) {
        is AgentEvent.Answer -> show(event.text)
        is AgentEvent.ToolCall -> log("calling ${event.name}")
        else -> {}
    }
}
```

## 10. 开发里程碑

1. **M1**: 多模块脚手架 + `agent-core` 门面 + `agent-llm` 国产模型 Provider,跑通最小对话(无工具)
   - 包含 `UserLlmConfig` + `LlmFactory` 接口骨架(BYO Key 预留)
   - M1 阶段用 `Providers.deepSeek(testKey)` 跑通,UI 配置入口暂不实现
2. **M2**: `agent-tools` @Tool DSL + ADK KSP,跑通工具调用闭环
3. **M3**: `agent-memory` Phase 1 预设角色 + 会话记忆
4. **M4**: `agent-rag` LangChain4j 检索 + 角色知识库注入
5. **M5**: `agent-memory` Phase 2 自定义角色 + 持久化
6. **M6**: `agent-llm` Phase 2 BYO Key 完整实现: `UserLlmConfig` 运行时切换 + `agent.reconfigure()`,宿主接入录入 UI
7. **M7**: `sample` Demo 验证集成体验(含用户配置 key 的完整流程)

## 11. 风险与对策

| 风险 | 对策 |
|------|------|
| ADK 0.1.0-beta API 变动 | `agent-core` 抽象层隔离,ADK 升级时只改内部 |
| ADK 对国产模型 OpenAI 兼容支持不完善 | `agent-llm` 预留 okhttp 自实现路径 |
| LangChain4j 与 ADK 双依赖体积 | 用户已声明体积不敏感;RAG 模块可按需引入 |
| 端侧能力(Phase 3)需 Gemini Nano | 当前云端优先,端侧作为后续扩展点 |
| BYO Key 安全性 | 组件不存 key,只接收明文;key 持久化由宿主负责(推荐 EncryptedSharedPreferences);组件不写日志输出 key |
| 用户误填错 key/端点 | `LlmProvider` 首次调用做连通性探测,失败时 `AgentEvent.Error` 携带清晰错误信息 |
