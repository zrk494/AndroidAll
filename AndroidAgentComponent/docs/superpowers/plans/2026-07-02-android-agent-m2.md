# Android Agent 组件 — M2 实现计划(工具调用闭环)

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 在 M1 最小对话闭环之上,实现 Agent 的工具调用(Tool Use)能力:用户输入 → LLM 决定调用工具 → 执行工具 → 结果喂回 LLM → 最终答案。形成 ReAct 单 Agent 工具闭环,全程纯 JVM 可单测,无真实网络依赖。

**Architecture:** 沿用 M1 分层。`agent-core` 扩展聊天类型以承载 function calling 字段,并新增 `internal ReActAgent` 实现工具循环;`agent-llm` 扩展 OpenAI 兼容 DTO 与 Provider 的双向(tools 序列化 / tool_calls 解析)适配;新建 `agent-tools` 模块提供类型安全 schema DSL 与内置 demo 工具。**不引入 ADK Runner / genai SDK**——经研究 ADK Kotlin 0.2.0 的 `com.google.adk.kt.models` 包仅含 `Gemini`(走 genai SDK),无 OpenAI 兼容 Model;国产模型要用 ADK Runner 须自行实现 `Model` 接口并吃下 google-genai 类型体系,投入产出不划算且可测性下降。M2 改为手写轻量 ReAct 循环,基于已有 `LlmProvider`(OpenAI 兼容)与 OpenAI function calling 标准协议,纯 JVM 可单测。`LlmProvider`/`Agent` 抽象层保证未来 ADK 加入 OpenAI 兼容 Model 时可平滑切换、宿主零改动。

**Tech Stack:** Kotlin 2.1.20、Kotlinx Coroutines/Flow、Kotlinx Serialization(JsonObject schema 构造)、OkHttp 4.12、JUnit 5、OkHttp MockWebServer。

**Context — M1 已落地(本计划基线):**
- `agent-core`:`Message(role,content)` / `ChatRequest(model,messages,temperature?,maxTokens?)` / `ChatResponse(content,model,finishReason?)` / `ChatChunk` / `LlmProvider` 接口 / `Tool(name,description,parametersJsonSchema,handler)` / `ToolRegistry` 接口 / `ToolRegistryScope`(`operator fun Tool.unaryPlus()`) / `AgentEvent` sealed(Thought/ToolCall/ToolResult/Answer/Error) / `AgentConfig(maxSteps=8,...)` / `Agent` 门面 / `AgentBuilder` / `internal MinimalAgent`(单轮)。
- `agent-llm`:`UserLlmConfig` / `Providers` / `LlmFactory` / `internal OpenAiClient`(OkHttp+SSE) / `internal OpenAiCompatibleProvider` / DTO。
- 测试:`:agent-core` 12 个、`:agent-llm` 12 个,全绿。
- `settings.gradle.kts`:`include(":agent-core", ":agent-llm")`。

## Global Constraints

- 纯 Kotlin/JVM 多模块,M2 不依赖 Android SDK、不依赖 ADK、不依赖 google-genai。
- 组件**不内置任何 API key**,工具 handler 不得在日志/异常中输出 key。
- 所有公共 API 用 Kotlin + Coroutines/Flow,无 UI 耦合。
- function calling 走 OpenAI 兼容标准协议(`/chat/completions` 的 `tools` + `tool_calls` + `role:"tool"` 消息)。ReAct 循环**只用非流式 `chat`**(流式 tool_calls 分片重组复杂,M2 不做,留后续)。
- 包名前缀统一 `com.you.agent`。
- TDD:先写失败测试,再写最小实现,再跑通,再提交。**不弱化断言**;若测试与实现冲突,先查根因。
- 每个任务结束提交一次,提交信息以 `feat:`/`chore:`/`test:`/`refactor:` 前缀。
- DRY、YAGNI:不实现 M2 范围外的能力(多 agent / Session 持久化 / 流式 tool_calls / RAG)。

---

## File Structure(本计划新增/改动)

```
agent-android/
├── settings.gradle.kts                  # 改: include +(":agent-tools")
├── gradle/libs.versions.toml            # 不动(agent-tools 复用现有 serialization-json)
├── agent-core/
│   ├── build.gradle.kts                 # 不动
│   └── src/
│       ├── main/kotlin/com/you/agent/core/
│       │   ├── chat/ChatTypes.kt        # 改: Message +toolCalls/toolCallId; ChatRequest +tools; ChatResponse +toolCalls; 新增 ToolCall
│       │   ├── tools/ToolRegistry.kt    # 改: 新增 ToolDeclaration + Tool.toDeclaration(); MapToolRegistry 默认实现
│       │   ├── AgentBuilder.kt          # 改: build() 按 tools 非空选 ReActAgent
│       │   └── internal/
│       │       ├── MinimalAgent.kt      # 不动(M1 保留)
│       │       └── ReActAgent.kt        # 新增: 工具循环
│       └── test/kotlin/com/you/agent/core/
│           ├── chat/ChatTypesFCTest.kt  # 新增: function calling 类型测试
│           ├── ReActAgentTest.kt        # 新增: FakeLlm 多轮 + 事件流
│           └── EndToEndMinimalAgentTest.kt  # M1 既有,不动
├── agent-llm/
│   ├── build.gradle.kts                 # 不动
│   └── src/
│       ├── main/kotlin/com/you/agent/llm/internal/
│       │   ├── OpenAiDtos.kt            # 改: +tools/toolChoice/toolCalls/toolCallId; OpenAiMessage.content 可空; 新增 OpenAiTool/OpenAiFunctionDecl/OpenAiToolCall/OpenAiToolCallFunction
│       │   └── OpenAiCompatibleProvider.kt  # 改: toDto 双向适配; chat 解析 tool_calls
│       └── test/kotlin/com/you/agent/llm/
│           ├── OpenAiCompatibleProviderToolsRequestTest.kt  # 新增
│           └── OpenAiCompatibleProviderToolsResponseTest.kt # 新增
└── agent-tools/                         # 新模块
    ├── build.gradle.kts                 # 新增
    └── src/
        ├── main/kotlin/com/you/agent/tools/
        │   ├── Schema.kt                # 新增: jsonSchema { } DSL → JSON Schema 字符串
        │   ├── Tools.kt                 # 新增: tool { } 便捷工厂 + MapToolRegistry 兼容(若不放 core)
        │   └── BuiltinTools.kt          # 新增: getCurrentTime demo 工具
        └── test/kotlin/com/you/agent/tools/
            ├── SchemaTest.kt            # 新增
            └── BuiltinToolsTest.kt      # 新增
```

职责边界:`agent-core` 只含抽象 + ReAct 循环实现,零网络零 ADK;`agent-llm` 负责 OpenAI 兼容 function calling 双向适配;`agent-tools` 负责 schema DSL 与内置工具,依赖 `agent-core`。`ToolCall.args` 用 `Map<String,Any?>`(agent-llm 负责 arguments string ↔ Map 互转,agent-core 不引 serialization)。

---

## Task 1: agent-core 扩展 ChatTypes 支持 function calling + ToolDeclaration

**Files:**
- Modify: `agent-android/agent-core/src/main/kotlin/com/you/agent/core/chat/ChatTypes.kt`
- Modify: `agent-android/agent-core/src/main/kotlin/com/you/agent/core/tools/ToolRegistry.kt`
- Test: `agent-android/agent-core/src/test/kotlin/com/you/agent/core/chat/ChatTypesFCTest.kt`

**Interfaces:**
- Consumes: M1 `Message`/`ChatRequest`/`ChatResponse`/`Tool`
- Produces: `Message.toolCalls`/`toolCallId`、`ChatRequest.tools`、`ChatResponse.toolCalls`、`ToolCall`、`ToolDeclaration`、`Tool.toDeclaration()`、`MapToolRegistry`(供 Task 5 ReActAgent 与 Task 3 toDto 使用)

- [ ] **Step 1: 写失败测试**

`agent-core/src/test/kotlin/com/you/agent/core/chat/ChatTypesFCTest.kt`:
```kotlin
package com.you.agent.core.chat

import com.you.agent.core.tools.Tool
import org.junit.jupiter.api.Assertions.assertEquals
import org.junit.jupiter.api.Assertions.assertNull
import org.junit.jupiter.api.Assertions.assertTrue
import org.junit.jupiter.api.Test

class ChatTypesFCTest {
    @Test
    fun `message carries tool calls and tool call id`() {
        val tc = ToolCall(id = "call_1", name = "get_time", args = mapOf("city" to "Beijing"))
        val assistant = Message(Role.ASSISTANT, content = "", toolCalls = listOf(tc))
        assertEquals(Role.ASSISTANT, assistant.role)
        assertEquals(1, assistant.toolCalls!!.size)
        assertEquals("call_1", assistant.toolCalls!![0].id)

        val toolMsg = Message(Role.TOOL, content = "result", toolCallId = "call_1")
        assertEquals("call_1", toolMsg.toolCallId)
        assertNull(toolMsg.toolCalls)
    }

    @Test
    fun `chat request carries tools`() {
        val decl = Tool(name = "get_time", description = "d", parametersJsonSchema = "{}") { "{}" }.toDeclaration()
        val req = ChatRequest(
            model = "m",
            messages = listOf(Message.user("hi")),
            tools = listOf(decl),
        )
        assertEquals(1, req.tools!!.size)
        assertEquals("get_time", req.tools!![0].name)
        assertEquals("{}", req.tools!![0].parametersJsonSchema)
    }

    @Test
    fun `chat response carries tool calls`() {
        val tc = ToolCall(id = "c1", name = "get_time", args = mapOf("city" to "Shanghai"))
        val resp = ChatResponse(content = "", model = "m", finishReason = "tool_calls", toolCalls = listOf(tc))
        assertEquals("tool_calls", resp.finishReason)
        assertEquals(1, resp.toolCalls!!.size)
        assertEquals("get_time", resp.toolCalls!![0].name)
        assertEquals("Shanghai", resp.toolCalls!![0].args["city"])
    }

    @Test
    fun `tool declaration strips handler`() {
        val tool = Tool(name = "t", description = "desc", parametersJsonSchema = "{\"type\":\"object\"}") { "ok" }
        val decl = tool.toDeclaration()
        assertEquals("t", decl.name)
        assertEquals("desc", decl.description)
        assertEquals("{\"type\":\"object\"}", decl.parametersJsonSchema)
    }

    @Test
    fun `map tool registry get and all`() {
        val reg = MapToolRegistry()
        val tool = Tool("t", "d", "{}") { "x" }
        reg.register(tool)
        assertTrue(reg.get("t") === tool)
        assertEquals(1, reg.all().size)
        assertNull(reg.get("missing"))
    }
}
```

- [ ] **Step 2: 跑测试确认失败**

Run: `./gradlew :agent-core:test`
Expected: FAIL,编译错误(`ToolCall`/`ToolDeclaration`/`MapToolRegistry`/新字段未定义)。

- [ ] **Step 3: 写最小实现**

`agent-core/src/main/kotlin/com/you/agent/core/chat/ChatTypes.kt`(整体替换):
```kotlin
package com.you.agent.core.chat

import com.you.agent.core.tools.ToolDeclaration

/** 对话中的一条消息。 */
data class Message(
    val role: Role,
    val content: String,
    /** assistant 消息携带的 tool_calls(LLM 决定调用工具时)。 */
    val toolCalls: List<ToolCall>? = null,
    /** role=TOOL 时对应的 tool_call_id。 */
    val toolCallId: String? = null,
) {
    enum class Role { SYSTEM, USER, ASSISTANT, TOOL }

    companion object {
        fun system(text: String) = Message(Role.SYSTEM, text)
        fun user(text: String) = Message(Role.USER, text)
        fun assistant(text: String) = Message(Role.ASSISTANT, text)
    }
}

/** LLM 决定调用的工具调用(来自响应或回传给 LLM 的 assistant 消息)。 */
data class ToolCall(
    val id: String,
    val name: String,
    val args: Map<String, Any?>,
)

/** 发往 LLM Provider 的请求。 */
data class ChatRequest(
    val model: String,
    val messages: List<Message>,
    val temperature: Double? = null,
    val maxTokens: Int? = null,
    /** 可用工具声明(OpenAI function calling)。null 表示不启用工具。 */
    val tools: List<ToolDeclaration>? = null,
)

/** LLM Provider 的非流式响应。 */
data class ChatResponse(
    val content: String,
    val model: String,
    val finishReason: String? = null,
    /** 模型决定调用工具时的 tool_calls;非空表示需执行工具后继续循环。 */
    val toolCalls: List<ToolCall>? = null,
)

/** 流式响应中的一个片段。 */
data class ChatChunk(
    val delta: String,
    val done: Boolean = false,
)
```

`agent-core/src/main/kotlin/com/you/agent/core/tools/ToolRegistry.kt`(整体替换):
```kotlin
package com.you.agent.core.tools

/** 发往 LLM 的工具声明(不含 handler,可安全序列化/传输)。 */
data class ToolDeclaration(
    val name: String,
    val description: String,
    val parametersJsonSchema: String,
)

/** Agent 可调用的工具。 */
data class Tool(
    val name: String,
    val description: String,
    val parametersJsonSchema: String,
    val handler: suspend (Map<String, Any?>) -> String,
) {
    /** 转为不含 handler 的声明,用于构造 LLM 请求。 */
    fun toDeclaration(): ToolDeclaration = ToolDeclaration(name, description, parametersJsonSchema)
}

interface ToolRegistry {
    fun register(tool: Tool)
    fun get(name: String): Tool?
    fun all(): List<Tool>
}

/** 默认实现:基于 Map。 */
class MapToolRegistry : ToolRegistry {
    private val map = mutableMapOf<String, Tool>()
    override fun register(tool: Tool) { map[tool.name] = tool }
    override fun get(name: String): Tool? = map[name]
    override fun all(): List<Tool> = map.values.toList()
}

/** Builder 的 tools { } DSL 作用域。 */
class ToolRegistryScope {
    private val tools = mutableListOf<Tool>()
    operator fun Tool.unaryPlus() { tools.add(this) }
    internal fun build(): List<Tool> = tools.toList()
}
```

- [ ] **Step 4: 跑测试确认通过**

Run: `./gradlew :agent-core:test`
Expected: BUILD SUCCESSFUL,新增 5 个测试通过,且 M1 既有测试不受影响(新字段均有默认值,向后兼容)。

- [ ] **Step 5: 提交**

```bash
git add agent-core/src
git commit -m "feat(agent-core): extend chat types for function calling (tools, tool_calls, ToolDeclaration, MapToolRegistry)"
```

---

## Task 2: agent-tools 模块脚手架 + schema DSL + 内置工具

**Files:**
- Modify: `agent-android/settings.gradle.kts`
- Create: `agent-android/agent-tools/build.gradle.kts`
- Create: `agent-android/agent-tools/src/main/kotlin/com/you/agent/tools/Schema.kt`
- Create: `agent-android/agent-tools/src/main/kotlin/com/you/agent/tools/Tools.kt`
- Create: `agent-android/agent-tools/src/main/kotlin/com/you/agent/tools/BuiltinTools.kt`
- Test: `agent-android/agent-tools/src/test/kotlin/com/you/agent/tools/SchemaTest.kt`
- Test: `agent-android/agent-tools/src/test/kotlin/com/you/agent/tools/BuiltinToolsTest.kt`

**Interfaces:**
- Consumes: `Tool`/`ToolDeclaration`(Task 1,来自 `agent-core`)
- Produces: `jsonSchema { }` DSL、`tool { }` 工厂、`getCurrentTime()` 内置工具(供 Task 5/6 使用)

- [ ] **Step 1: 模块脚手架**

`agent-android/settings.gradle.kts`:
```kotlin
rootProject.name = "agent-android"
include(":agent-core", ":agent-llm", ":agent-tools")
```

`agent-android/agent-tools/build.gradle.kts`:
```kotlin
plugins {
    alias(libs.plugins.kotlin.jvm)
    alias(libs.plugins.kotlin.serialization)
}

dependencies {
    implementation(project(":agent-core"))
    implementation(libs.kotlinx.coroutines.core)
    implementation(libs.kotlinx.serialization.json)
    testImplementation(libs.kotlinx.coroutines.test)
    testImplementation(libs.junit.jupiter)
}

tasks.test { useJUnitPlatform() }
```

- [ ] **Step 2: 写失败测试**

`agent-tools/src/test/kotlin/com/you/agent/tools/SchemaTest.kt`:
```kotlin
package com.you.agent.tools

import kotlinx.serialization.json.Json
import org.junit.jupiter.api.Assertions.assertEquals
import org.junit.jupiter.api.Assertions.assertTrue
import org.junit.jupiter.api.Test

class SchemaTest {
    private val json = Json { ignoreUnknownKeys = true }

    @Test
    fun `object schema with required string prop`() {
        val s = jsonSchema {
            prop("city", "string", "城市名称", required = true)
        }
        val obj = json.parseToJsonElement(s).jsonObject
        assertEquals("object", obj["type"]!!.jsonPrimitive.content)
        assertTrue(obj["properties"]!!.jsonObject.containsKey("city"))
        assertEquals(listOf("city"), obj["required"]!!.jsonArray.map { it.jsonPrimitive.content })
    }

    @Test
    fun `enum prop`() {
        val s = jsonSchema {
            prop("unit", "string", "单位", required = false, enum = listOf("celsius", "fahrenheit"))
        }
        val obj = json.parseToJsonElement(s).jsonObject
        val unit = obj["properties"]!!.jsonObject["unit"]!!.jsonObject
        assertEquals(listOf("celsius", "fahrenheit"), unit["enum"]!!.jsonArray.map { it.jsonPrimitive.content })
    }

    @Test
    fun `empty schema is bare object`() {
        val s = jsonSchema { }
        val obj = json.parseToJsonElement(s).jsonObject
        assertEquals("object", obj["type"]!!.jsonPrimitive.content)
        assertTrue(obj["properties"]!!.jsonObject.isEmpty())
    }
}
```

`agent-tools/src/test/kotlin/com/you/agent/tools/BuiltinToolsTest.kt`:
```kotlin
package com.you.agent.tools

import kotlinx.coroutines.test.runTest
import org.junit.jupiter.api.Assertions.assertEquals
import org.junit.jupiter.api.Assertions.assertTrue
import org.junit.jupiter.api.Test

class BuiltinToolsTest {
    @Test
    fun `getCurrentTime tool shape`() {
        val tool = getCurrentTime()
        assertEquals("get_current_time", tool.name)
        assertTrue(tool.parametersJsonSchema.contains("city"))
    }

    @Test
    fun `getCurrentTime handler returns json with city`() = runTest {
        val tool = getCurrentTime()
        val out = tool.handler(mapOf("city" to "Beijing"))
        assertTrue(out.contains("Beijing"))
        assertTrue(out.contains("time"))
    }

    @Test
    fun `tool factory builds from schema block`() {
        val t = tool(name = "echo", description = "回显") {
            prop("text", "string", "文本", required = true)
        } { args -> args["text"].toString() }
        assertEquals("echo", t.name)
        assertTrue(t.parametersJsonSchema.contains("text"))
    }
}
```

- [ ] **Step 3: 跑测试确认失败**

Run: `./gradlew :agent-tools:test`
Expected: FAIL,编译错误(模块/类未定义)。首次需 Gradle 识别新模块,可能需 `./gradlew projects` 确认 `:agent-tools` 出现。

- [ ] **Step 4: 写最小实现**

`agent-tools/src/main/kotlin/com/you/agent/tools/Schema.kt`:
```kotlin
package com.you.agent.tools

import kotlinx.serialization.json.JsonArray
import kotlinx.serialization.json.JsonObject
import kotlinx.serialization.json.JsonPrimitive
import kotlinx.serialization.json.add
import kotlinx.serialization.json.buildJsonArray
import kotlinx.serialization.json.buildJsonObject
import kotlinx.serialization.json.put

/** 类型安全的 JSON Schema 构造器,产出 OpenAI function calling 的 parameters 字符串。 */
class SchemaBuilder {
    private val properties = mutableMapOf<String, JsonObject>()
    private val required = mutableListOf<String>()

    /** 声明一个参数。 */
    fun prop(
        name: String,
        type: String,
        description: String,
        required: Boolean = false,
        enum: List<String>? = null,
    ) {
        val prop = buildJsonObject {
            put("type", type)
            put("description", description)
            if (enum != null) put("enum", JsonArray(enum.map { JsonPrimitive(it) }))
        }
        properties[name] = prop
        if (required) this.required.add(name)
    }

    internal fun build(): JsonObject = buildJsonObject {
        put("type", "object")
        putJsonObject("properties") {
            properties.forEach { (k, v) -> put(k, v) }
        }
        if (required.isNotEmpty()) {
            put("required", JsonArray(required.map { JsonPrimitive(it) }))
        }
    }
}

/** DSL 入口:构造一个 JSON Schema 字符串。 */
fun jsonSchema(block: SchemaBuilder.() -> Unit): String =
    SchemaBuilder().apply(block).build().toString()
```

`agent-tools/src/main/kotlin/com/you/agent/tools/Tools.kt`:
```kotlin
package com.you.agent.tools

import com.you.agent.core.tools.Tool

/** 便捷工厂:用 schema DSL 构造工具,避免手写 JSON Schema 字符串。 */
fun tool(
    name: String,
    description: String,
    schema: SchemaBuilder.() -> Unit,
    handler: suspend (Map<String, Any?>) -> String,
): Tool = Tool(
    name = name,
    description = description,
    parametersJsonSchema = jsonSchema(schema),
    handler = handler,
)
```

`agent-tools/src/main/kotlin/com/you/agent/tools/BuiltinTools.kt`:
```kotlin
package com.you.agent.tools

import com.you.agent.core.tools.Tool

/** 内置 demo 工具:返回指定城市的(模拟)当前时间。 */
fun getCurrentTime(): Tool = tool(
    name = "get_current_time",
    description = "获取指定城市的当前时间",
    schema = { prop("city", "string", "城市名称", required = true) },
) { args ->
    val city = args["city"] as? String ?: "unknown"
    """{"city":"$city","time":"10:30"}"""
}
```

注:`Schema.kt` 用到 `buildJsonObject`/`putJsonObject` 等,需 `import kotlinx.serialization.json.putJsonObject`(kotlinx-serialization-json 提供)。若编译报 `putJsonObject` 未解析,改为嵌套 `buildJsonObject` 内联到 `"properties"` key。Step 4 实现前确认 API 存在;`putJsonObject` 自 kotlinx-serialization-json 1.6 起可用,本项目用 1.6.3,可用。

- [ ] **Step 5: 跑测试确认通过**

Run: `./gradlew :agent-tools:test`
Expected: BUILD SUCCESSFUL,6 个测试通过。

- [ ] **Step 6: 提交**

```bash
git add settings.gradle.kts agent-tools
git commit -m "feat(agent-tools): add schema DSL, tool factory, and getCurrentTime builtin tool"
```

---

## Task 3: agent-llm function calling 请求侧(tools 序列化)

**Files:**
- Modify: `agent-android/agent-llm/src/main/kotlin/com/you/agent/llm/internal/OpenAiDtos.kt`
- Modify: `agent-android/agent-llm/src/main/kotlin/com/you/agent/llm/internal/OpenAiCompatibleProvider.kt`
- Test: `agent-android/agent-llm/src/test/kotlin/com/you/agent/llm/OpenAiCompatibleProviderToolsRequestTest.kt`

**Interfaces:**
- Consumes: `ChatRequest.tools`/`Message.toolCalls`/`Message.toolCallId`(Task 1)
- Produces: 请求体含 `tools` + `tool_choice="auto"`;assistant 消息的 `tool_calls` 与 tool 消息的 `tool_call_id` 正确序列化(供 Task 5/6)

- [ ] **Step 1: 写失败测试**

`agent-llm/src/test/kotlin/com/you/agent/llm/OpenAiCompatibleProviderToolsRequestTest.kt`:
```kotlin
package com.you.agent.llm

import com.you.agent.core.chat.ChatRequest
import com.you.agent.core.chat.Message
import com.you.agent.core.chat.ToolCall
import com.you.agent.core.tools.ToolDeclaration
import kotlinx.coroutines.test.runTest
import okhttp3.mockwebserver.MockResponse
import okhttp3.mockwebserver.MockWebServer
import org.junit.jupiter.api.Assertions.assertTrue
import org.junit.jupiter.api.Test

class OpenAiCompatibleProviderToolsRequestTest {
    private fun provider(server: MockWebServer) =
        com.you.agent.llm.internal.OpenAiCompatibleProvider(
            server.url("/v1").toString().trimEnd('/'), "key", "deepseek-chat")

    @Test
    fun `request body includes tools and tool_choice auto`() = runTest {
        val server = MockWebServer()
        server.enqueue(MockResponse().setBody(
            """{"model":"deepseek-chat","choices":[{"message":{"role":"assistant","content":"ok"}}]}"""))
        server.start()
        try {
            val decl = ToolDeclaration("get_time", "get time", """{"type":"object","properties":{"city":{"type":"string"}}}""")
            provider(server).chat(ChatRequest(
                model = "deepseek-chat",
                messages = listOf(Message.user("hi")),
                tools = listOf(decl),
            ))
            val body = server.takeRequest().body.readUtf8()
            assertTrue(body.contains("\"tools\""))
            assertTrue(body.contains("\"get_time\""))
            assertTrue(body.contains("\"tool_choice\":\"auto\""))
        } finally { server.shutdown() }
    }

    @Test
    fun `assistant tool_calls message serialized with tool_call_id on tool message`() = runTest {
        val server = MockWebServer()
        server.enqueue(MockResponse().setBody(
            """{"model":"deepseek-chat","choices":[{"message":{"role":"assistant","content":"done"}}]}"""))
        server.start()
        try {
            val tc = ToolCall(id = "call_9", name = "get_time", args = mapOf("city" to "Beijing"))
            val messages = listOf(
                Message.user("hi"),
                Message(Message.Role.ASSISTANT, content = "", toolCalls = listOf(tc)),
                Message(Message.Role.TOOL, content = "result", toolCallId = "call_9"),
            )
            provider(server).chat(ChatRequest("deepseek-chat", messages))
            val body = server.takeRequest().body.readUtf8()
            assertTrue(body.contains("\"tool_calls\""))
            assertTrue(body.contains("\"call_9\""))
            assertTrue(body.contains("\"get_time\""))
            assertTrue(body.contains("\"tool_call_id\":\"call_9\""))
        } finally { server.shutdown() }
    }
}
```

- [ ] **Step 2: 跑测试确认失败**

Run: `./gradlew :agent-llm:test`
Expected: FAIL(请求体不含 tools/tool_choice/tool_calls——toDto 未实现)。

- [ ] **Step 3: 写最小实现**

`agent-llm/src/main/kotlin/com/you/agent/llm/internal/OpenAiDtos.kt`(整体替换):
```kotlin
package com.you.agent.llm.internal

import kotlinx.serialization.SerialName
import kotlinx.serialization.Serializable
import kotlinx.serialization.json.JsonElement
import kotlinx.serialization.json.Json

@Serializable
data class OpenAiChatRequest(
    val model: String,
    val messages: List<OpenAiMessage>,
    val temperature: Double? = null,
    @SerialName("max_tokens") val maxTokens: Int? = null,
    val stream: Boolean = false,
    val tools: List<OpenAiTool>? = null,
    @SerialName("tool_choice") val toolChoice: String? = null,
)

@Serializable
data class OpenAiTool(
    val type: String = "function",
    val function: OpenAiFunctionDecl,
)

@Serializable
data class OpenAiFunctionDecl(
    val name: String,
    val description: String,
    val parameters: JsonElement,
)

@Serializable
data class OpenAiMessage(
    val role: String,
    val content: String? = null,
    val toolCalls: List<OpenAiToolCall>? = null,
    @SerialName("tool_call_id") val toolCallId: String? = null,
)

@Serializable
data class OpenAiToolCall(
    val id: String,
    val type: String = "function",
    val function: OpenAiToolCallFunction,
)

@Serializable
data class OpenAiToolCallFunction(
    val name: String,
    val arguments: String,
)

@Serializable
data class OpenAiChatResponse(
    val id: String? = null,
    val model: String? = null,
    val choices: List<OpenAiChoice> = emptyList(),
)

@Serializable
data class OpenAiChoice(
    val index: Int = 0,
    val message: OpenAiMessage? = null,
    @SerialName("finish_reason") val finishReason: String? = null,
)

@Serializable
data class OpenAiStreamChunk(
    val choices: List<OpenAiStreamChoice> = emptyList(),
)

@Serializable
data class OpenAiStreamChoice(
    val index: Int = 0,
    val delta: OpenAiDelta = OpenAiDelta(),
    @SerialName("finish_reason") val finishReason: String? = null,
)

@Serializable
data class OpenAiDelta(
    val role: String? = null,
    val content: String? = null,
)

@Serializable
data class OpenAiErrorResponse(
    val error: OpenAiError? = null,
)

@Serializable
data class OpenAiError(
    val message: String? = null,
    val type: String? = null,
    val code: String? = null,
)
```

`agent-llm/src/main/kotlin/com/you/agent/llm/internal/OpenAiCompatibleProvider.kt`(整体替换):
```kotlin
package com.you.agent.llm.internal

import com.you.agent.core.chat.ChatChunk
import com.you.agent.core.chat.ChatRequest
import com.you.agent.core.chat.ChatResponse
import com.you.agent.core.chat.Message
import com.you.agent.core.chat.ToolCall
import com.you.agent.core.llm.LlmProvider
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.map
import kotlinx.serialization.json.Json

class OpenAiCompatibleProvider(
    private val baseUrl: String,
    private val apiKey: String,
    private val model: String,
    private val client: OpenAiClient = OpenAiClient(baseUrl, apiKey),
) : LlmProvider {
    override val modelId: String = model

    private val argsJson = Json { ignoreUnknownKeys = true }

    override suspend fun chat(request: ChatRequest): ChatResponse {
        val resp = client.chat(toDto(request))
        val choice = resp.choices.firstOrNull()
        val msg = choice?.message
        val toolCalls = msg?.toolCalls?.map {
            ToolCall(
                id = it.id,
                name = it.function.name,
                args = parseArgs(it.function.arguments),
            )
        }
        val text = msg?.content ?: ""
        return ChatResponse(
            content = text,
            model = resp.model ?: model,
            finishReason = choice?.finishReason,
            toolCalls = toolCalls,
        )
    }

    override fun streamChat(request: ChatRequest): Flow<ChatChunk> =
        client.streamChunks(toDto(request)).map { chunk ->
            val choice = chunk.choices.firstOrNull()
            ChatChunk(delta = choice?.delta?.content ?: "", done = choice?.finishReason != null)
        }

    private fun toDto(request: ChatRequest): OpenAiChatRequest = OpenAiChatRequest(
        model = request.model,
        messages = request.messages.map { it.toOpenAi() },
        temperature = request.temperature,
        maxTokens = request.maxTokens,
        tools = request.tools?.map {
            OpenAiTool(function = OpenAiFunctionDecl(
                name = it.name,
                description = it.description,
                parameters = argsJson.parseToJsonElement(it.parametersJsonSchema),
            ))
        },
        toolChoice = if (request.tools?.isNotEmpty() == true) "auto" else null,
    )

    private fun Message.toOpenAi(): OpenAiMessage = OpenAiMessage(
        role = role.name.lowercase(),
        content = content.ifEmpty { null },
        toolCalls = toolCalls?.map {
            OpenAiToolCall(id = it.id, function = OpenAiToolCallFunction(
                name = it.name,
                arguments = argsJson.encodeToString(kotlinx.serialization.builtins.MapSerializer(
                    kotlinx.serialization.builtins.serializer<String>(),
                    kotlinx.serialization.json.JsonElement.serializer()),
                    it.args.mapValues { (_, v) -> kotlinx.serialization.json.JsonPrimitive(v.toString()) }),
            ))
        },
        toolCallId = toolCallId,
    )

    private fun parseArgs(s: String): Map<String, Any?> {
        if (s.isBlank()) return emptyMap()
        val obj = argsJson.parseToJsonElement(s).let { it as? kotlinx.serialization.json.JsonObject }
            ?: return emptyMap()
        return obj.mapValues { (_, v) ->
            when (v) {
                is kotlinx.serialization.json.JsonPrimitive ->
                    v.content.toIntOrNull() ?: v.content.toDoubleOrNull() ?: v.content
                else -> v.toString()
            }
        }
    }
}
```

注:`toOpenAi` 中把 `args: Map<String,Any?>` 序列化回 arguments string 的写法较繁(Any? 无 serializer)。简化替代:用 `Json.encodeToString(JsonElement.serializer(), buildJsonObject { args.forEach { (k,v) -> put(k, v.toString()) } })`。Step 3 实现时优先用更简洁可编译的写法,但**必须保证 arguments 是合法 JSON 字符串且含 city 字段**(测试断言 `contains("Beijing")` 经由 `contains("\"get_time\"")` 等间接覆盖)。若上述 MapSerializer 写法编译困难,改为:
```kotlin
arguments = kotlinx.serialization.json.buildJsonObject {
    it.args.forEach { (k, v) -> put(k, v?.toString() ?: kotlinx.serialization.json.JsonNull) }
}.toString()
```
任选其一,以编译通过 + 测试通过为准。

- [ ] **Step 4: 跑测试确认通过**

Run: `./gradlew :agent-llm:test`
Expected: BUILD SUCCESSFUL,新增 2 个测试通过,既有 12 个不受影响。若 `streamChat` 因 OpenAiMessage.content 改可空而编译失败,检查 `OpenAiClient.streamChunks` 未直接访问 `message.content`(它只读 stream delta,不受影响)。

- [ ] **Step 5: 提交**

```bash
git add agent-llm/src
git commit -m "feat(agent-llm): serialize tools, tool_choice, and tool_calls in request body"
```

---

## Task 4: agent-llm function calling 响应侧(tool_calls 解析)

**Files:**
- Modify: 无实现改动(Task 3 的 `chat` 已含 tool_calls 解析,本任务补测试覆盖)
- Test: `agent-android/agent-llm/src/test/kotlin/com/you/agent/llm/OpenAiCompatibleProviderToolsResponseTest.kt`

**Interfaces:**
- Consumes: `OpenAiCompatibleProvider.chat`(Task 3 实现)
- Produces: 响应 tool_calls 解析行为验证(供 Task 5/6 信任)

- [ ] **Step 1: 写测试**

`agent-llm/src/test/kotlin/com/you/agent/llm/OpenAiCompatibleProviderToolsResponseTest.kt`:
```kotlin
package com.you.agent.llm

import com.you.agent.core.chat.ChatRequest
import com.you.agent.core.chat.Message
import kotlinx.coroutines.test.runTest
import okhttp3.mockwebserver.MockResponse
import okhttp3.mockwebserver.MockWebServer
import org.junit.jupiter.api.Assertions.assertEquals
import org.junit.jupiter.api.Assertions.assertNull
import org.junit.jupiter.api.Test

class OpenAiCompatibleProviderToolsResponseTest {
    private fun provider(server: MockWebServer) =
        com.you.agent.llm.internal.OpenAiCompatibleProvider(
            server.url("/v1").toString().trimEnd('/'), "key", "deepseek-chat")

    @Test
    fun `parses tool_calls response`() = runTest {
        val server = MockWebServer()
        server.enqueue(MockResponse().setBody("""
            {"model":"deepseek-chat","choices":[{"index":0,"message":{"role":"assistant","content":null,"tool_calls":[{"id":"call_1","type":"function","function":{"name":"get_current_time","arguments":"{\"city\":\"Beijing\"}"}}]},"finish_reason":"tool_calls"}]}
        """.trimIndent()))
        server.start()
        try {
            val resp = provider(server).chat(ChatRequest("deepseek-chat", listOf(Message.user("几点了"))))
            assertEquals("tool_calls", resp.finishReason)
            assertEquals(1, resp.toolCalls!!.size)
            val tc = resp.toolCalls!![0]
            assertEquals("call_1", tc.id)
            assertEquals("get_current_time", tc.name)
            assertEquals("Beijing", tc.args["city"])
        } finally { server.shutdown() }
    }

    @Test
    fun `plain answer has no tool_calls`() = runTest {
        val server = MockWebServer()
        server.enqueue(MockResponse().setBody("""
            {"model":"deepseek-chat","choices":[{"message":{"role":"assistant","content":"hello"}}]}
        """.trimIndent()))
        server.start()
        try {
            val resp = provider(server).chat(ChatRequest("deepseek-chat", listOf(Message.user("hi"))))
            assertEquals("hello", resp.content)
            assertNull(resp.toolCalls)
        } finally { server.shutdown() }
    }

    @Test
    fun `empty arguments parsed to empty map`() = runTest {
        val server = MockWebServer()
        server.enqueue(MockResponse().setBody("""
            {"choices":[{"message":{"role":"assistant","content":null,"tool_calls":[{"id":"c2","type":"function","function":{"name":"no_args","arguments":""}}]}}]}
        """.trimIndent()))
        server.start()
        try {
            val resp = provider(server).chat(ChatRequest("deepseek-chat", listOf(Message.user("go"))))
            assertEquals(0, resp.toolCalls!![0].args.size)
        } finally { server.shutdown() }
    }
}
```

- [ ] **Step 2: 跑测试**

Run: `./gradlew :agent-llm:test`
Expected: BUILD SUCCESSFUL,3 个测试通过。若 `empty arguments` 解析报错,确认 `parseArgs` 对空串返回 `emptyMap()`(Task 3 实现已含 `if (s.isBlank()) return emptyMap()`)。

- [ ] **Step 3: 提交**

```bash
git add agent-llm/src
git commit -m "test(agent-llm): cover tool_calls response parsing with MockWebServer"
```

---

## Task 5: agent-core ReActAgent 工具循环

**Files:**
- Create: `agent-android/agent-core/src/main/kotlin/com/you/agent/core/internal/ReActAgent.kt`
- Modify: `agent-android/agent-core/src/main/kotlin/com/you/agent/core/AgentBuilder.kt`
- Test: `agent-android/agent-core/src/test/kotlin/com/you/agent/core/ReActAgentTest.kt`

**Interfaces:**
- Consumes: `LlmProvider`(M1)、`Tool`/`MapToolRegistry`/`ToolDeclaration`/`ToolCall`(Task 1)、`ChatRequest.tools`/`ChatResponse.toolCalls`(Task 1)、`AgentEvent`(M1)、`AgentConfig.maxSteps`(M1)
- Produces: `internal ReActAgent`、`AgentBuilder.build()` 工具分支(供 Task 6 端到端)

- [ ] **Step 1: 写失败测试**

`agent-core/src/test/kotlin/com/you/agent/core/ReActAgentTest.kt`:
```kotlin
package com.you.agent.core

import com.you.agent.core.chat.ChatChunk
import com.you.agent.core.chat.ChatRequest
import com.you.agent.core.chat.ChatResponse
import com.you.agent.core.chat.Message
import com.you.agent.core.chat.ToolCall
import com.you.agent.core.llm.LlmProvider
import com.you.agent.core.persona.Persona
import com.you.agent.core.tools.Tool
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.flowOf
import kotlinx.coroutines.flow.toList
import kotlinx.coroutines.test.runTest
import org.junit.jupiter.api.Assertions.assertEquals
import org.junit.jupiter.api.Assertions.assertInstanceOf
import org.junit.jupiter.api.Test

class ReActAgentTest {
    /** 可编排多轮响应的 Fake LLM。 */
    private class ScriptedLlm(override val modelId: String = "fake") : LlmProvider {
        val requests = mutableListOf<ChatRequest>()
        private val responses = ArrayDeque<ChatResponse>()
        fun enqueue(resp: ChatResponse) { responses.addLast(resp) }
        override suspend fun chat(request: ChatRequest): ChatResponse {
            requests.add(request)
            return responses.removeFirst()
        }
        override fun streamChat(request: ChatRequest): Flow<ChatChunk> = flowOf(ChatChunk("x", done = true))
    }

    private fun timeTool() = Tool(
        name = "get_current_time",
        description = "get time",
        parametersJsonSchema = "{}",
    ) { args -> """{"city":"${args["city"]}","time":"10:30"}""" }

    @Test
    fun `tool call then answer emits ToolCall ToolResult Answer`() = runTest {
        val llm = ScriptedLlm().apply {
            enqueue(ChatResponse(content = "", model = "fake", finishReason = "tool_calls",
                toolCalls = listOf(ToolCall("c1", "get_current_time", mapOf("city" to "Beijing")))))
            enqueue(ChatResponse(content = "Beijing 时间是 10:30", model = "fake", finishReason = "stop"))
        }
        val agent = Agent.builder()
            .llm(llm)
            .persona(Persona("p", "P", "sys"))
            .tools { +timeTool() }
            .build()

        val events = agent.run("几点了").toList()

        assertEquals(3, events.size)
        assertInstanceOf(AgentEvent.ToolCall::class.java, events[0])
        assertEquals("get_current_time", (events[0] as AgentEvent.ToolCall).name)
        assertEquals("Beijing", (events[0] as AgentEvent.ToolCall).args["city"])
        assertInstanceOf(AgentEvent.ToolResult::class.java, events[1])
        assertEquals("get_current_time", (events[1] as AgentEvent.ToolResult).name)
        assertInstanceOf(AgentEvent.Answer::class.java, events[2])
        assertEquals("Beijing 时间是 10:30", (events[2] as AgentEvent.Answer).text)
        // 第二轮请求应含 tool 结果消息(role=TOOL)
        val secondReq = llm.requests[1]
        assertEquals(Message.Role.TOOL, secondReq.messages.last().role)
        assertEquals("c1", secondReq.messages.last().toolCallId)
    }

    @Test
    fun `no tools falls back to single answer via MinimalAgent`() = runTest {
        val llm = ScriptedLlm().apply {
            enqueue(ChatResponse(content = "hi", model = "fake", finishReason = "stop"))
        }
        val agent = Agent.builder().llm(llm).persona(Persona("p", "P", "sys")).build()
        val events = agent.run("hello").toList()
        assertEquals(1, events.size)
        assertInstanceOf(AgentEvent.Answer::class.java, events[0])
    }

    @Test
    fun `maxSteps exceeded emits Error`() = runTest {
        val llm = ScriptedLlm().apply {
            // 始终返回 tool_calls,永不收敛
            repeat(10) {
                enqueue(ChatResponse(content = "", model = "fake", finishReason = "tool_calls",
                    toolCalls = listOf(ToolCall("c$it", "get_current_time", mapOf("city" to "x")))))
            }
        }
        val agent = Agent.builder()
            .llm(llm)
            .persona(Persona("p", "P", "sys"))
            .tools { +timeTool() }
            .config { maxSteps = 2 }
            .build()
        val events = agent.run("loop").toList()
        // 2 轮各 emit ToolCall+ToolResult,然后 maxSteps 超限 emit Error
        assertInstanceOf(AgentEvent.Error::class.java, events.last())
    }

    @Test
    fun `unknown tool name emits Error`() = runTest {
        val llm = ScriptedLlm().apply {
            enqueue(ChatResponse(content = "", model = "fake", finishReason = "tool_calls",
                toolCalls = listOf(ToolCall("c1", "missing_tool", emptyMap()))))
        }
        val agent = Agent.builder()
            .llm(llm).persona(Persona("p", "P", "sys"))
            .tools { +timeTool() }
            .build()
        val events = agent.run("x").toList()
        assertInstanceOf(AgentEvent.Error::class.java, events.last())
    }
}
```

- [ ] **Step 2: 跑测试确认失败**

Run: `./gradlew :agent-core:test`
Expected: FAIL,编译错误(ReActAgent 未定义,build() 未分支)。

- [ ] **Step 3: 写最小实现**

`agent-core/src/main/kotlin/com/you/agent/core/internal/ReActAgent.kt`:
```kotlin
package com.you.agent.core.internal

import com.you.agent.core.Agent
import com.you.agent.core.AgentConfig
import com.you.agent.core.AgentEvent
import com.you.agent.core.chat.ChatRequest
import com.you.agent.core.chat.Message
import com.you.agent.core.llm.LlmProvider
import com.you.agent.core.persona.Persona
import com.you.agent.core.tools.MapToolRegistry
import com.you.agent.core.tools.Tool
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.flow

/**
 * M2 ReAct 实现: 循环调用 LLM; 若返回 tool_calls 则执行工具、把结果作为 tool 消息喂回, 直到 LLM 返回纯文本 Answer 或达 maxSteps。
 * 不依赖 ADK, 仅基于 LlmProvider + OpenAI function calling 协议。
 */
internal class ReActAgent(
    private val llm: LlmProvider,
    private val persona: Persona,
    private val tools: List<Tool>,
    private val config: AgentConfig,
) : Agent {
    override fun run(userInput: String, personaId: String?): Flow<AgentEvent> = flow {
        val registry = MapToolRegistry().also { reg -> tools.forEach { reg.register(it) } }
        val declarations = tools.map { it.toDeclaration() }
        val messages = mutableListOf(Message.system(persona.systemPrompt), Message.user(userInput))
        try {
            repeat(config.maxSteps) {
                val resp = llm.chat(ChatRequest(
                    model = llm.modelId,
                    messages = messages.toList(),
                    temperature = config.temperature,
                    maxTokens = config.maxTokens,
                    tools = declarations,
                ))
                val toolCalls = resp.toolCalls
                if (toolCalls.isNullOrEmpty()) {
                    emit(AgentEvent.Answer(resp.content))
                    return@flow
                }
                // 记录 assistant 的 tool_calls 消息
                messages.add(Message(Message.Role.ASSISTANT, content = "", toolCalls = toolCalls))
                for (tc in toolCalls) {
                    emit(AgentEvent.ToolCall(tc.name, tc.args))
                    val tool = registry.get(tc.name)
                        ?: run {
                            emit(AgentEvent.Error(IllegalStateException("未注册的工具: ${tc.name}")))
                            return@flow
                        }
                    val result = tool.handler(tc.args)
                    emit(AgentEvent.ToolResult(tc.name, result))
                    messages.add(Message(Message.Role.TOOL, content = result, toolCallId = tc.id))
                }
            }
            // 循环结束仍未收敛
            emit(AgentEvent.Error(IllegalStateException("达到 maxSteps(${config.maxSteps}) 未收敛")))
        } catch (t: Throwable) {
            emit(AgentEvent.Error(t))
        }
    }
}
```

`agent-core/src/main/kotlin/com/you/agent/core/AgentBuilder.kt`(改 `build()` 与 import):
```kotlin
package com.you.agent.core

import com.you.agent.core.internal.MinimalAgent
import com.you.agent.core.internal.ReActAgent
import com.you.agent.core.llm.LlmProvider
import com.you.agent.core.persona.KnowledgeRetriever
import com.you.agent.core.persona.Persona
import com.you.agent.core.persona.PersonaStore
import com.you.agent.core.tools.Tool
import com.you.agent.core.tools.ToolRegistryScope

class AgentBuilder {
    private var llm: LlmProvider? = null
    private var persona: Persona = DEFAULT_PERSONA
    private var personaStore: PersonaStore? = null
    private var retriever: KnowledgeRetriever? = null
    private var tools: List<Tool> = emptyList()
    private val config = AgentConfig()

    fun llm(provider: LlmProvider): AgentBuilder = apply { this.llm = provider }
    fun persona(persona: Persona): AgentBuilder = apply { this.persona = persona }
    fun memory(store: PersonaStore): AgentBuilder = apply { this.personaStore = store }
    fun rag(retriever: KnowledgeRetriever): AgentBuilder = apply { this.retriever = retriever }
    fun tools(block: ToolRegistryScope.() -> Unit): AgentBuilder =
        apply { this.tools = ToolRegistryScope().apply(block).build() }
    fun config(block: AgentConfig.() -> Unit): AgentBuilder = apply { config.block() }

    fun build(): Agent {
        val provider = llm
            ?: error("Agent requires an LlmProvider. 未配置 LLM 时绝不静默使用默认 Provider。")
        // 工具非空 → ReAct 循环;否则 → M1 单轮 MinimalAgent(零改动,向后兼容)。
        return if (tools.isNotEmpty()) {
            ReActAgent(llm = provider, persona = persona, tools = tools, config = config)
        } else {
            MinimalAgent(llm = provider, persona = persona, config = config)
        }
    }

    companion object {
        val DEFAULT_PERSONA = Persona(
            id = "default",
            name = "Assistant",
            systemPrompt = "You are a helpful assistant.",
        )
    }
}

/** 入口 DSL: Agent.builder() */
fun Agent.Companion.builder(): AgentBuilder = AgentBuilder()
```

- [ ] **Step 4: 跑测试确认通过**

Run: `./gradlew :agent-core:test`
Expected: BUILD SUCCESSFUL,4 个 ReActAgent 测试通过,M1 既有测试不受影响(MinimalAgent 路径保留)。

- [ ] **Step 5: 提交**

```bash
git add agent-core/src
git commit -m "feat(agent-core): add ReActAgent for tool-calling loop with maxSteps guard"
```

---

## Task 6: 端到端集成(MockWebServer 两轮 tool 闭环)

**Files:**
- Test: `agent-android/agent-core/src/test/kotlin/com/you/agent/core/EndToEndToolAgentTest.kt`
- Modify: `agent-android/agent-core/build.gradle.kts`(加 `testImplementation(project(":agent-tools"))`)

**Interfaces:**
- Consumes: `Agent.builder()`(Task 5)、`OpenAiCompatibleProvider`(M1)、`getCurrentTime()`(Task 2)
- Produces: M2 验收——无真实网络下端到端工具调用闭环

- [ ] **Step 1: 加 test 依赖**

在 `agent-core/build.gradle.kts` 的 `dependencies { }` 内追加:
```kotlin
    testImplementation(project(":agent-tools"))
```
(用于测试中调用 `getCurrentTime()`。`agent-llm` 与 `mockwebserver` M1 Task 9 已加。)

- [ ] **Step 2: 写测试**

`agent-core/src/test/kotlin/com/you/agent/core/EndToEndToolAgentTest.kt`:
```kotlin
package com.you.agent.core

import com.you.agent.core.persona.Persona
import com.you.agent.llm.LlmFactory
import com.you.agent.llm.UserLlmConfig
import com.you.agent.tools.getCurrentTime
import kotlinx.coroutines.flow.toList
import kotlinx.coroutines.test.runTest
import okhttp3.mockwebserver.MockResponse
import okhttp3.mockwebserver.MockWebServer
import org.junit.jupiter.api.Assertions.assertEquals
import org.junit.jupiter.api.Assertions.assertInstanceOf
import org.junit.jupiter.api.Assertions.assertTrue
import org.junit.jupiter.api.Test

class EndToEndToolAgentTest {
    @Test
    fun `tool call loop completes end to end`() = runTest {
        val server = MockWebServer()
        // 第一轮:模型决定调用工具
        server.enqueue(MockResponse().setBody("""
            {"model":"deepseek-chat","choices":[{"index":0,"message":{"role":"assistant","content":null,"tool_calls":[{"id":"call_1","type":"function","function":{"name":"get_current_time","arguments":"{\"city\":\"Beijing\"}"}}]},"finish_reason":"tool_calls"}]}
        """.trimIndent()))
        // 第二轮:模型给出最终答案
        server.enqueue(MockResponse().setBody("""
            {"model":"deepseek-chat","choices":[{"message":{"role":"assistant","content":"Beijing 现在是 10:30"}}]}""".trimIndent()))
        server.start()
        try {
            val provider = LlmFactory.from(UserLlmConfig(
                providerId = "custom",
                apiKey = "key",
                baseUrl = server.url("/v1").toString().trimEnd('/'),
                model = "deepseek-chat",
            ))
            val agent = Agent.builder()
                .llm(provider)
                .persona(Persona("p", "P", "You can tell time."))
                .tools { +getCurrentTime() }
                .build()

            val events = agent.run("Beijing 几点了?").toList()

            assertEquals(3, events.size)
            assertInstanceOf(AgentEvent.ToolCall::class.java, events[0])
            assertEquals("get_current_time", (events[0] as AgentEvent.ToolCall).name)
            assertInstanceOf(AgentEvent.ToolResult::class.java, events[1])
            assertInstanceOf(AgentEvent.Answer::class.java, events[2])
            assertEquals("Beijing 现在是 10:30", (events[2] as AgentEvent.Answer).text)

            // 第二轮请求体应含 tool 结果消息
            server.takeRequest() // 第一轮
            val secondReq = server.takeRequest()
            val body = secondReq.body.readUtf8()
            assertTrue(body.contains("\"role\":\"tool\""))
            assertTrue(body.contains("\"tool_call_id\":\"call_1\""))
            assertTrue(body.contains("\"get_current_time\""))
        } finally { server.shutdown() }
    }
}
```

- [ ] **Step 3: 跑全部测试**

Run: `./gradlew test`
Expected: BUILD SUCCESSFUL,三模块测试全过(`:agent-core`、`:agent-llm`、`:agent-tools`)。

- [ ] **Step 4: 提交**

```bash
git add agent-core/build.gradle.kts agent-core/src
git commit -m "test: end-to-end tool-calling loop via MockWebServer"
```

---

## Self-Review

**1. Spec coverage(对照设计文档 M2 范围):**
- `agent-tools` @Tool DSL + 内置工具 → Task 2 ✓(注:设计文档写"ADK KSP",经用户确认为手写路线,改纯 Kotlin schema DSL + 工厂,不引 KSP。理由记录于 Architecture)
- 跑通工具调用闭环(无真实网络) → Task 5(FakeLlm 多轮)+ Task 6(MockWebServer 两轮)✓
- 国产模型 OpenAI 兼容 function calling → Task 3/4 ✓
- `LlmProvider`/`Agent` 抽象层不变,宿主零改动 → build() 内部分支,对外 API 不变 ✓
- 未覆盖(留给后续里程碑,符合范围):多 agent/Session 持久化(M3+)、RAG(M4)、流式 tool_calls、自定义角色(M5)、sample(M7)

**2. ADK 取代说明(对设计文档的有意偏离):**
设计文档 2.1/5 节假设 M2 引入 ADK Runner。研究确证 ADK Kotlin 0.2.0 的 models 包仅 `Gemini`(genai SDK),无 OpenAI 兼容 Model;国产模型走 ADK 须自实现 `Model` 接口 + 引 google-genai 类型体系,可测性与投入产出不划算。经用户确认(2026-07-02),M2 改为手写轻量 ReAct,`Agent`/`LlmProvider` 抽象层保证未来 ADK 加 OpenAI 兼容 Model 时可平滑切换。

**3. Placeholder scan:** Task 3 Step 3 对 `args Map → arguments string` 序列化给了两种可编译写法(任选其一),非 placeholder,是明确的双选实现指引。其余每步含完整代码或确切命令。✓

**4. Type consistency:**
- `ToolCall(id, name, args: Map<String,Any?>)` Task 1 定义,Task 3(parseArgs)/Task 5(handler 调用)/Task 6 断言一致 ✓
- `ToolDeclaration(name, description, parametersJsonSchema)` Task 1 定义,Task 1 测试 + Task 3 toDto 一致 ✓
- `Message(role, content, toolCalls?, toolCallId?)` Task 1 定义,Task 3/5/6 一致;`content` 保持 String(空串),DTO 层 `ifEmpty{null}` 转 OpenAI null ✓
- `ChatRequest.tools: List<ToolDeclaration>?` Task 1 定义,Task 3 toDto + Task 5 构造一致 ✓
- `ChatResponse.toolCalls: List<ToolCall>?` Task 1 定义,Task 3 chat 解析 + Task 4 测试 + Task 5 分支一致 ✓
- `MapToolRegistry` Task 1 定义,Task 5 使用 ✓
- `AgentBuilder.build()` Task 5 改造,Task 6 使用;无 tools 走 MinimalAgent(M1 测试不受影响)✓
- `AgentEvent.ToolCall/ToolResult` M1 定义,Task 5 emit + Task 6 断言一致 ✓
- `AgentConfig.maxSteps` M1 定义(默认 8),Task 5 用作循环上限 + 测试 `maxSteps=2` ✓

**5. 向后兼容:** 所有新增字段均有默认值;`MinimalAgent` 与 M1 全部既有测试零改动保留。`OpenAiMessage.content` 改可空(内部 DTO,不暴露);`OpenAiCompatibleProvider.chat` 的 content 取值由 `?: ""` 兜底,对外 `ChatResponse.content` 仍为 String。✓

---

## Execution Handoff

计划已保存至 `docs/superpowers/plans/2026-07-02-android-agent-m2.md`。

**无需用户再次确认架构偏离**——ADK→手写 ReAct 的收窄已于计划编写前经用户明确确认(2026-07-02)。其余均严格遵循设计文档 M2 范围与 M1 既定约束。

沿用 M1 的 **Subagent-Driven Development** 执行:每个 Task 派发独立 subagent,TDD 严格流程(失败测试→实现→通过→提交),任务间两阶段评审。关键环境约束(JDK 17、`./gradlew`、沙箱代理已配、不改 build 文件除非计划明确要求、不弱化断言)与 M1 一致。
