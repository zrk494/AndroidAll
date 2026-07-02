# Android Agent 组件 — M1 实现计划(最小对话闭环)

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 搭建 Gradle 多模块骨架并实现 `agent-core` + `agent-llm`,跑通"用户输入 → 国产模型(OpenAI 兼容)→ 答案事件流"的最小对话闭环(无工具),全程可在纯 JVM 上单元测试。

**Architecture:** M1 为纯 Kotlin/JVM 多模块(`agent-core` 抽象层 + `agent-llm` OpenAI 兼容 Provider),不依赖 Android SDK、不依赖 ADK 运行时。`agent-core` 的 `MinimalAgent` 直接调用 `LlmProvider.chat()` 并产出 `AgentEvent.Answer` 流。ADK Runner/Model 适配器推迟到 M2(工具调用需要 ReAct 循环时再引入),`Agent`/`LlmProvider` 抽象层屏蔽此内部差异,后续切换零宿主改动。

**Tech Stack:** Kotlin 2.1.20、Kotlinx Coroutines/Flow、Kotlinx Serialization、OkHttp 4.12、JUnit 5、OkHttp MockWebServer。

## Global Constraints

- 模块以 `kotlin("jvm")` 插件构建(M1 纯 JVM,Android 宿主可消费 JAR;待 M3 引入 assets/Room 时相关模块再切 `com.android.library`)。
- 组件**不内置任何 API key,不预设默认模型**;`AgentBuilder.build()` 未配置 `LlmProvider` 时直接抛错,绝不静默兜底。
- 组件**不存储、不日志输出** API key;key 以明文由宿主传入,持久化由宿主负责。
- 所有公共 API 用 Kotlin + Coroutines/Flow,无 UI 耦合。
- 国产模型走 OpenAI 兼容 API(`/chat/completions`),`baseUrl` 可指向 DeepSeek/通义/GLM/Moonshot/自建。
- 包名前缀统一 `com.you.agent`。
- 每个任务结束必须提交一次,提交信息以 `feat:`/`chore:`/`test:` 前缀。
- DRY、YAGNI、TDD:先写失败测试,再写最小实现,再跑通,再提交。

---

## File Structure

```
agent-android/
├── .gitignore
├── gradle.properties
├── settings.gradle.kts
├── build.gradle.kts
├── gradle/libs.versions.toml
├── agent-core/
│   ├── build.gradle.kts
│   └── src/
│       ├── main/kotlin/com/you/agent/core/
│       │   ├── chat/ChatTypes.kt          # Message / ChatRequest / ChatResponse / ChatChunk
│       │   ├── llm/LlmProvider.kt         # LlmProvider 接口
│       │   ├── persona/Persona.kt         # Persona / PersonaStore / KnowledgeRetriever / KnowledgeChunk (骨架)
│       │   ├── tools/ToolRegistry.kt      # Tool / ToolRegistry / ToolRegistryScope (骨架)
│       │   ├── Agent.kt                   # Agent 接口 + AgentEvent + AgentConfig
│       │   ├── AgentBuilder.kt            # AgentBuilder DSL + Agent.builder() 扩展
│       │   └── internal/MinimalAgent.kt   # M1 最小实现: 调 LlmProvider 一次 → Answer
│       └── test/kotlin/com/you/agent/core/...
└── agent-llm/
    ├── build.gradle.kts
    └── src/
        ├── main/kotlin/com/you/agent/llm/
        │   ├── UserLlmConfig.kt           # BYO Key 配置载体
        │   ├── Providers.kt               # 各厂商工厂
        │   ├── LlmFactory.kt              # UserLlmConfig → LlmProvider 统一入口
        │   └── internal/
        │       ├── OpenAiDtos.kt          # 请求/响应 JSON DTO (kotlinx.serialization)
        │       ├── OpenAiClient.kt        # OkHttp 客户端(同步 chat + SSE 流)
        │       ├── OpenAiException.kt     # 非 2xx 异常
        │       └── OpenAiCompatibleProvider.kt # 实现 LlmProvider
        └── test/kotlin/com/you/agent/llm/...
```

职责边界:`agent-core` 只含抽象 + M1 最小实现,零网络零 ADK;`agent-llm` 只含 OpenAI 兼容 HTTP + BYO 配置,依赖 `agent-core`。

---

## Task 1: Gradle 多模块骨架 + git 初始化

**Files:**
- Create: `agent-android/.gitignore`
- Create: `agent-android/gradle.properties`
- Create: `agent-android/settings.gradle.kts`
- Create: `agent-android/build.gradle.kts`
- Create: `agent-android/gradle/libs.versions.toml`
- Create: `agent-android/agent-core/build.gradle.kts`
- Create: `agent-android/agent-llm/build.gradle.kts`

**Interfaces:**
- Consumes: 无(项目起点)
- Produces: 可构建的空模块 `:agent-core`、`:agent-llm`;后续任务在此之上添加源码

- [ ] **Step 1: 创建目录与配置文件**

`agent-android/gradle/libs.versions.toml`:
```toml
[versions]
kotlin = "2.1.20"
kotlinx-coroutines = "1.8.1"
kotlinx-serialization = "1.6.3"
okhttp = "4.12.0"
junit = "5.10.2"

[libraries]
kotlinx-coroutines-core = { module = "org.jetbrains.kotlinx:kotlinx-coroutines-core", version.ref = "kotlinx-coroutines" }
kotlinx-coroutines-test = { module = "org.jetbrains.kotlinx:kotlinx-coroutines-test", version.ref = "kotlinx-coroutines" }
kotlinx-serialization-json = { module = "org.jetbrains.kotlinx:kotlinx-serialization-json", version.ref = "kotlinx-serialization" }
okhttp = { module = "com.squareup.okhttp3:okhttp", version.ref = "okhttp" }
okhttp-mockwebserver = { module = "com.squareup.okhttp3:mockwebserver", version.ref = "okhttp" }
junit-jupiter = { module = "org.junit.jupiter:junit-jupiter", version.ref = "junit" }

[plugins]
kotlin-jvm = { id = "org.jetbrains.kotlin.jvm", version.ref = "kotlin" }
kotlin-serialization = { id = "org.jetbrains.kotlin.plugin.serialization", version.ref = "kotlin" }
```

`agent-android/settings.gradle.kts`:
```kotlin
rootProject.name = "agent-android"
include(":agent-core", ":agent-llm")
```

`agent-android/build.gradle.kts`:
```kotlin
plugins {
    alias(libs.plugins.kotlin.jvm) apply false
    alias(libs.plugins.kotlin.serialization) apply false
}
```

`agent-android/gradle.properties`:
```properties
org.gradle.jvmargs=-Xmx2g
kotlin.code.style=official
```

`agent-android/.gitignore`:
```
.gradle/
build/
!gradle/wrapper/gradle-wrapper.jar
*.iml
local.properties
.env
```

`agent-android/agent-core/build.gradle.kts`:
```kotlin
plugins {
    alias(libs.plugins.kotlin.jvm)
}

dependencies {
    implementation(libs.kotlinx.coroutines.core)
    testImplementation(libs.kotlinx.coroutines.test)
    testImplementation(libs.junit.jupiter)
}

tasks.test { useJUnitPlatform() }
```

`agent-android/agent-llm/build.gradle.kts`:
```kotlin
plugins {
    alias(libs.plugins.kotlin.jvm)
    alias(libs.plugins.kotlin.serialization)
}

dependencies {
    implementation(project(":agent-core"))
    implementation(libs.kotlinx.coroutines.core)
    implementation(libs.kotlinx.serialization.json)
    implementation(libs.okhttp)
    testImplementation(libs.kotlinx.coroutines.test)
    testImplementation(libs.okhttp.mockwebserver)
    testImplementation(libs.junit.jupiter)
}

tasks.test { useJUnitPlatform() }
```

- [ ] **Step 2: 生成 Gradle Wrapper**

在 `agent-android/` 下执行(若系统已装 gradle):
```bash
gradle wrapper --gradle-version 8.7 --distribution-type bin
```
若无 `gradle` 命令,改为手动放置 `gradle/wrapper/gradle-wrapper.jar` 与 `gradle/wrapper/gradle-wrapper.properties`(distributionUrl=gradle-8.7-bin.zip)及 `gradlew`、`gradlew.bat`。后续命令一律用 `./gradlew`。

- [ ] **Step 3: 验证模块结构可被识别**

Run: `./gradlew projects`
Expected: 输出包含 `Project ':agent-core'` 与 `Project ':agent-llm'`,BUILD SUCCESSFUL。

- [ ] **Step 4: 初始化 git 并首次提交**

```bash
cd agent-android
git init
git add .
git commit -m "chore: scaffold gradle multi-module (agent-core, agent-llm)"
```

---

## Task 2: agent-core 聊天数据类型 (ChatTypes)

**Files:**
- Create: `agent-android/agent-core/src/main/kotlin/com/you/agent/core/chat/ChatTypes.kt`
- Test: `agent-android/agent-core/src/test/kotlin/com/you/agent/core/chat/ChatTypesTest.kt`

**Interfaces:**
- Consumes: 无
- Produces: `Message`、`Message.Role`、`ChatRequest`、`ChatResponse`、`ChatChunk`(后续 Task 3/5/7 使用)

- [ ] **Step 1: 写失败测试**

`agent-core/src/test/kotlin/com/you/agent/core/chat/ChatTypesTest.kt`:
```kotlin
package com.you.agent.core.chat

import org.junit.jupiter.api.Assertions.assertEquals
import org.junit.jupiter.api.Test

class ChatTypesTest {
    @Test
    fun `message factory sets role`() {
        assertEquals(Message.Role.SYSTEM, Message.system("s").role)
        assertEquals(Message.Role.USER, Message.user("u").role)
        assertEquals(Message.Role.ASSISTANT, Message.assistant("a").role)
    }

    @Test
    fun `chat request carries model and messages`() {
        val req = ChatRequest(model = "m", messages = listOf(Message.user("hi")))
        assertEquals("m", req.model)
        assertEquals(1, req.messages.size)
        assertEquals(null, req.temperature)
        assertEquals(null, req.maxTokens)
    }

    @Test
    fun `chat response and chunk defaults`() {
        val resp = ChatResponse(content = "hello", model = "m")
        assertEquals("hello", resp.content)
        assertEquals(null, resp.finishReason)
        val chunk = ChatChunk(delta = "he")
        assertEquals(false, chunk.done)
    }
}
```

- [ ] **Step 2: 跑测试确认失败**

Run: `./gradlew :agent-core:test`
Expected: FAIL,编译错误(ChatTypes 未定义)。

- [ ] **Step 3: 写最小实现**

`agent-core/src/main/kotlin/com/you/agent/core/chat/ChatTypes.kt`:
```kotlin
package com.you.agent.core.chat

/** 对话中的一条消息。 */
data class Message(
    val role: Role,
    val content: String,
) {
    enum class Role { SYSTEM, USER, ASSISTANT, TOOL }

    companion object {
        fun system(text: String) = Message(Role.SYSTEM, text)
        fun user(text: String) = Message(Role.USER, text)
        fun assistant(text: String) = Message(Role.ASSISTANT, text)
    }
}

/** 发往 LLM Provider 的请求。 */
data class ChatRequest(
    val model: String,
    val messages: List<Message>,
    val temperature: Double? = null,
    val maxTokens: Int? = null,
)

/** LLM Provider 的非流式响应。 */
data class ChatResponse(
    val content: String,
    val model: String,
    val finishReason: String? = null,
)

/** 流式响应中的一个片段。 */
data class ChatChunk(
    val delta: String,
    val done: Boolean = false,
)
```

- [ ] **Step 4: 跑测试确认通过**

Run: `./gradlew :agent-core:test`
Expected: BUILD SUCCESSFUL,3 个测试通过。

- [ ] **Step 5: 提交**

```bash
git add agent-core/src
git commit -m "feat(agent-core): add chat data types (Message, ChatRequest, ChatResponse, ChatChunk)"
```

---

## Task 3: agent-core LlmProvider 接口 + Persona + 骨架接口

**Files:**
- Create: `agent-android/agent-core/src/main/kotlin/com/you/agent/core/llm/LlmProvider.kt`
- Create: `agent-android/agent-core/src/main/kotlin/com/you/agent/core/persona/Persona.kt`
- Create: `agent-android/agent-core/src/main/kotlin/com/you/agent/core/tools/ToolRegistry.kt`
- Test: `agent-android/agent-core/src/test/kotlin/com/you/agent/core/LlmProviderAndPersonaTest.kt`

**Interfaces:**
- Consumes: `ChatRequest`/`ChatResponse`/`ChatChunk`(Task 2)
- Produces: `LlmProvider` 接口(供 Task 5 MinimalAgent、Task 7 Provider 实现);`Persona`/`PersonaStore`/`KnowledgeRetriever`/`KnowledgeChunk` 骨架(供 Task 5 Builder);`Tool`/`ToolRegistry`/`ToolRegistryScope` 骨架(供 Task 5 Builder)

- [ ] **Step 1: 写失败测试**

`agent-core/src/test/kotlin/com/you/agent/core/LlmProviderAndPersonaTest.kt`:
```kotlin
package com.you.agent.core

import com.you.agent.core.chat.ChatChunk
import com.you.agent.core.chat.ChatRequest
import com.you.agent.core.chat.ChatResponse
import com.you.agent.core.chat.Message
import com.you.agent.core.llm.LlmProvider
import com.you.agent.core.persona.Persona
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.flowOf
import kotlinx.coroutines.test.runTest
import org.junit.jupiter.api.Assertions.assertEquals
import org.junit.jupiter.api.Test

class LlmProviderAndPersonaTest {
    private class FakeLlm(override val modelId: String = "fake") : LlmProvider {
        override suspend fun chat(request: ChatRequest): ChatResponse =
            ChatResponse(content = "echo:${request.messages.last().content}", model = modelId)
        override fun streamChat(request: ChatRequest): Flow<ChatChunk> = flowOf(ChatChunk("echo", done = true))
    }

    @Test
    fun `fake provider implements contract`() = runTest {
        val p = FakeLlm()
        val req = ChatRequest(model = "fake", messages = listOf(Message.user("hi")))
        assertEquals("echo:hi", p.chat(req).content)
    }

    @Test
    fun `persona defaults are empty`() {
        val persona = Persona(id = "p1", name = "N", systemPrompt = "sys")
        assertEquals(emptyMap<String, String>(), persona.traits)
        assertEquals(emptyList<String>(), persona.knowledgeSources)
        assertEquals(emptyList<String>(), persona.tools)
    }
}
```

- [ ] **Step 2: 跑测试确认失败**

Run: `./gradlew :agent-core:test`
Expected: FAIL,编译错误(LlmProvider/Persona 未定义)。

- [ ] **Step 3: 写最小实现**

`agent-core/src/main/kotlin/com/you/agent/core/llm/LlmProvider.kt`:
```kotlin
package com.you.agent.core.llm

import com.you.agent.core.chat.ChatChunk
import com.you.agent.core.chat.ChatRequest
import com.you.agent.core.chat.ChatResponse
import kotlinx.coroutines.flow.Flow

/** LLM 提供方抽象,挡在任何具体 SDK(含未来 ADK Model 适配)之前。 */
interface LlmProvider {
    val modelId: String
    suspend fun chat(request: ChatRequest): ChatResponse
    fun streamChat(request: ChatRequest): Flow<ChatChunk>
}
```

`agent-core/src/main/kotlin/com/you/agent/core/persona/Persona.kt`:
```kotlin
package com.you.agent.core.persona

/** Agent 所扮演的角色。 */
data class Persona(
    val id: String,
    val name: String,
    val systemPrompt: String,
    val traits: Map<String, String> = emptyMap(),
    val knowledgeSources: List<String> = emptyList(),
    val tools: List<String> = emptyList(),
)

/** 角色存储。M1 仅定义接口;M3 提供预设实现,M5 提供可写实现。 */
interface PersonaStore {
    suspend fun load(id: String): Persona
    fun list(): List<Persona>
    suspend fun save(persona: Persona)
}

/** 角色知识库检索(M4 实现 LangChain4j 版)。 */
interface KnowledgeRetriever {
    suspend fun retrieve(query: String, persona: Persona, topK: Int = 4): List<KnowledgeChunk>
}

/** 检索到的一条知识片段。 */
data class KnowledgeChunk(
    val text: String,
    val source: String,
    val score: Double? = null,
)
```

`agent-core/src/main/kotlin/com/you/agent/core/tools/ToolRegistry.kt`:
```kotlin
package com.you.agent.core.tools

/** Agent 可调用的工具。M1 仅定义结构;M2 接入 ADK @Tool。 */
data class Tool(
    val name: String,
    val description: String,
    val parametersJsonSchema: String,
    val handler: suspend (Map<String, Any?>) -> String,
)

interface ToolRegistry {
    fun register(tool: Tool)
    fun get(name: String): Tool?
    fun all(): List<Tool>
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
Expected: BUILD SUCCESSFUL,所有测试通过。

- [ ] **Step 5: 提交**

```bash
git add agent-core/src
git commit -m "feat(agent-core): add LlmProvider interface, Persona, and skeleton stores"
```

---

## Task 4: agent-core AgentEvent + Agent 接口 + AgentConfig

**Files:**
- Create: `agent-android/agent-core/src/main/kotlin/com/you/agent/core/Agent.kt`
- Test: `agent-android/agent-core/src/test/kotlin/com/you/agent/core/AgentEventTest.kt`

**Interfaces:**
- Consumes: 无新依赖
- Produces: `AgentEvent` 密封接口、`AgentConfig`、`Agent` 接口(含 `companion object`,供 Task 5 `Agent.builder()` 扩展)

- [ ] **Step 1: 写失败测试**

`agent-core/src/test/kotlin/com/you/agent/core/AgentEventTest.kt`:
```kotlin
package com.you.agent.core

import org.junit.jupiter.api.Assertions.assertEquals
import org.junit.jupiter.api.Assertions.assertTrue
import org.junit.jupiter.api.Test

class AgentEventTest {
    @Test
    fun `config defaults`() {
        val cfg = AgentConfig()
        assertEquals(8, cfg.maxSteps)
        assertEquals(null, cfg.temperature)
        assertEquals(null, cfg.maxTokens)
    }

    @Test
    fun `event subtypes are distinct`() {
        val events: List<AgentEvent> = listOf(
            AgentEvent.Thought("t"),
            AgentEvent.ToolCall("c", mapOf("x" to 1)),
            AgentEvent.ToolResult("c", "out"),
            AgentEvent.Answer("a"),
            AgentEvent.Error(RuntimeException("e")),
        )
        assertTrue(events[0] is AgentEvent.Thought)
        assertTrue(events[3] is AgentEvent.Answer)
        assertTrue(events[4] is AgentEvent.Error)
    }
}
```

- [ ] **Step 2: 跑测试确认失败**

Run: `./gradlew :agent-core:test`
Expected: FAIL,编译错误(AgentEvent/AgentConfig 未定义)。

- [ ] **Step 3: 写最小实现**

`agent-core/src/main/kotlin/com/you/agent/core/Agent.kt`:
```kotlin
package com.you.agent.core

import kotlinx.coroutines.flow.Flow

/** Agent 运行期间发出的事件流。 */
sealed interface AgentEvent {
    data class Thought(val text: String) : AgentEvent
    data class ToolCall(val name: String, val args: Map<String, Any?>) : AgentEvent
    data class ToolResult(val name: String, val output: String) : AgentEvent
    data class Answer(val text: String) : AgentEvent
    data class Error(val cause: Throwable) : AgentEvent
}

/** Agent 配置。 */
data class AgentConfig(
    var maxSteps: Int = 8,
    var temperature: Double? = null,
    var maxTokens: Int? = null,
)

/** Agent 门面。M1 最小循环:单次 LLM 调用 → Answer。 */
interface Agent {
    fun run(userInput: String, personaId: String? = null): Flow<AgentEvent>

    companion object
}
```

- [ ] **Step 4: 跑测试确认通过**

Run: `./gradlew :agent-core:test`
Expected: BUILD SUCCESSFUL。

- [ ] **Step 5: 提交**

```bash
git add agent-core/src
git commit -m "feat(agent-core): add AgentEvent, AgentConfig, and Agent facade interface"
```

---

## Task 5: agent-core AgentBuilder + MinimalAgent(最小对话闭环)

**Files:**
- Create: `agent-android/agent-core/src/main/kotlin/com/you/agent/core/AgentBuilder.kt`
- Create: `agent-android/agent-core/src/main/kotlin/com/you/agent/core/internal/MinimalAgent.kt`
- Test: `agent-android/agent-core/src/test/kotlin/com/you/agent/core/MinimalAgentTest.kt`

**Interfaces:**
- Consumes: `LlmProvider`(Task 3)、`Persona`/`PersonaStore`/`KnowledgeRetriever`/`ToolRegistryScope`(Task 3)、`Agent`/`AgentConfig`/`AgentEvent`(Task 4)、`ChatRequest`/`Message`(Task 2)
- Produces: `AgentBuilder`、`Agent.builder()` 扩展、`internal MinimalAgent`;M1 核心交付物——可运行的对话闭环

- [ ] **Step 1: 写失败测试**

`agent-core/src/test/kotlin/com/you/agent/core/MinimalAgentTest.kt`:
```kotlin
package com.you.agent.core

import com.you.agent.core.chat.ChatChunk
import com.you.agent.core.chat.ChatRequest
import com.you.agent.core.chat.ChatResponse
import com.you.agent.core.chat.Message
import com.you.agent.core.llm.LlmProvider
import com.you.agent.core.persona.Persona
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.flowOf
import kotlinx.coroutines.flow.toList
import kotlinx.coroutines.test.runTest
import org.junit.jupiter.api.Assertions.assertEquals
import org.junit.jupiter.api.Assertions.assertInstanceOf
import org.junit.jupiter.api.Assertions.assertThrows
import org.junit.jupiter.api.Test

class MinimalAgentTest {
    private class FakeLlm(override val modelId: String = "fake") : LlmProvider {
        var lastRequest: ChatRequest? = null
        var throwOnChat: Throwable? = null
        override suspend fun chat(request: ChatRequest): ChatResponse {
            lastRequest = request
            throwOnChat?.let { throw it }
            return ChatResponse(content = "answer:${request.messages.last().content}", model = modelId)
        }
        override fun streamChat(request: ChatRequest): Flow<ChatChunk> = flowOf(ChatChunk("x", done = true))
    }

    @Test
    fun `run emits single Answer with persona and user input`() = runTest {
        val llm = FakeLlm()
        val agent = Agent.builder()
            .llm(llm)
            .persona(Persona(id = "p", name = "P", systemPrompt = "be nice"))
            .build()

        val events = agent.run("hello").toList()

        assertEquals(1, events.size)
        assertInstanceOf(AgentEvent.Answer::class.java, events[0])
        assertEquals("answer:hello", (events[0] as AgentEvent.Answer).text)
        // persona system prompt 作为第一条消息
        val req = llm.lastRequest!!
        assertEquals(2, req.messages.size)
        assertEquals(Message.Role.SYSTEM, req.messages[0].role)
        assertEquals("be nice", req.messages[0].content)
        assertEquals(Message.Role.USER, req.messages[1].role)
    }

    @Test
    fun `run emits Error when llm throws`() = runTest {
        val llm = FakeLlm().apply { throwOnChat = RuntimeException("boom") }
        val agent = Agent.builder().llm(llm).build()

        val events = agent.run("hi").toList()

        assertEquals(1, events.size)
        assertInstanceOf(AgentEvent.Error::class.java, events[0])
    }

    @Test
    fun `build without llm throws`() {
        assertThrows(IllegalStateException::class.java) { Agent.builder().build() }
    }

    @Test
    fun `config is applied to request`() = runTest {
        val llm = FakeLlm()
        Agent.builder().llm(llm).config { temperature = 0.3; maxTokens = 100 }.build().run("q").toList()
        assertEquals(0.3, llm.lastRequest!!.temperature)
        assertEquals(100, llm.lastRequest!!.maxTokens)
    }
}
```

- [ ] **Step 2: 跑测试确认失败**

Run: `./gradlew :agent-core:test`
Expected: FAIL,编译错误(AgentBuilder/MinimalAgent 未定义)。

- [ ] **Step 3: 写最小实现**

`agent-core/src/main/kotlin/com/you/agent/core/internal/MinimalAgent.kt`:
```kotlin
package com.you.agent.core.internal

import com.you.agent.core.Agent
import com.you.agent.core.AgentConfig
import com.you.agent.core.AgentEvent
import com.you.agent.core.chat.ChatRequest
import com.you.agent.core.chat.Message
import com.you.agent.core.llm.LlmProvider
import com.you.agent.core.persona.Persona
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.flow

/** M1 最小实现: 组装 persona system prompt + 用户输入, 调 LlmProvider 一次, 产出 Answer。 */
internal class MinimalAgent(
    private val llm: LlmProvider,
    private val persona: Persona,
    private val config: AgentConfig,
) : Agent {
    override fun run(userInput: String, personaId: String?): Flow<AgentEvent> = flow {
        val messages = listOf(
            Message.system(persona.systemPrompt),
            Message.user(userInput),
        )
        val request = ChatRequest(
            model = llm.modelId,
            messages = messages,
            temperature = config.temperature,
            maxTokens = config.maxTokens,
        )
        try {
            val response = llm.chat(request)
            emit(AgentEvent.Answer(response.content))
        } catch (t: Throwable) {
            emit(AgentEvent.Error(t))
        }
    }
}
```

`agent-core/src/main/kotlin/com/you/agent/core/AgentBuilder.kt`:
```kotlin
package com.you.agent.core

import com.you.agent.core.internal.MinimalAgent
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
        return MinimalAgent(llm = provider, persona = persona, config = config)
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
Expected: BUILD SUCCESSFUL,4 个 MinimalAgent 测试通过。

- [ ] **Step 5: 提交**

```bash
git add agent-core/src
git commit -m "feat(agent-core): add AgentBuilder and MinimalAgent for minimal conversation loop"
```

---

## Task 6: agent-llm UserLlmConfig + Providers + LlmFactory

**Files:**
- Create: `agent-android/agent-llm/src/main/kotlin/com/you/agent/llm/UserLlmConfig.kt`
- Create: `agent-android/agent-llm/src/main/kotlin/com/you/agent/llm/internal/OpenAiCompatibleProvider.kt`
- Create: `agent-android/agent-llm/src/main/kotlin/com/you/agent/llm/internal/OpenAiDtos.kt`
- Create: `agent-android/agent-llm/src/main/kotlin/com/you/agent/llm/internal/OpenAiException.kt`
- Create: `agent-android/agent-llm/src/main/kotlin/com/you/agent/llm/internal/OpenAiClient.kt`
- Create: `agent-android/agent-llm/src/main/kotlin/com/you/agent/llm/Providers.kt`
- Create: `agent-android/agent-llm/src/main/kotlin/com/you/agent/llm/LlmFactory.kt`
- Test: `agent-android/agent-llm/src/test/kotlin/com/you/agent/llm/LlmFactoryTest.kt`

**Interfaces:**
- Consumes: `LlmProvider`(Task 3,来自 `agent-core`)
- Produces: `UserLlmConfig`、`Providers`、`LlmFactory`、`internal OpenAiCompatibleProvider`/`OpenAiClient`/DTO(供 Task 7/8 测试与 Task 9 集成)

注:本任务把 Provider 实现一并落地(工厂需要构造它),但其 HTTP 行为的测试在 Task 7/8 用 MockWebServer 覆盖。本任务只验证工厂分支逻辑。

- [ ] **Step 1: 写失败测试**

`agent-llm/src/test/kotlin/com/you/agent/llm/LlmFactoryTest.kt`:
```kotlin
package com.you.agent.llm

import org.junit.jupiter.api.Assertions.assertEquals
import org.junit.jupiter.api.Assertions.assertThrows
import org.junit.jupiter.api.Test

class LlmFactoryTest {
    @Test
    fun `deepseek defaults`() {
        val p = LlmFactory.from(UserLlmConfig(providerId = "deepseek", apiKey = "k"))
        assertEquals("deepseek-chat", p.modelId)
    }

    @Test
    fun `qwen defaults`() {
        assertEquals("qwen-plus", LlmFactory.from(UserLlmConfig("qwen", "k")).modelId)
    }

    @Test
    fun `glm defaults`() {
        assertEquals("glm-4", LlmFactory.from(UserLlmConfig("glm", "k")).modelId)
    }

    @Test
    fun `moonshot defaults`() {
        assertEquals("moonshot-v1-8k", LlmFactory.from(UserLlmConfig("moonshot", "k")).modelId)
    }

    @Test
    fun `custom model override`() {
        val p = LlmFactory.from(UserLlmConfig("deepseek", "k", model = "deepseek-reasoner"))
        assertEquals("deepseek-reasoner", p.modelId)
    }

    @Test
    fun `custom provider requires baseUrl`() {
        assertThrows(IllegalStateException::class.java) {
            LlmFactory.from(UserLlmConfig("custom", "k"))
        }
    }

    @Test
    fun `custom provider requires model`() {
        assertThrows(IllegalStateException::class.java) {
            LlmFactory.from(UserLlmConfig("custom", "k", baseUrl = "https://x/v1"))
        }
    }

    @Test
    fun `unknown provider throws`() {
        assertThrows(IllegalStateException::class.java) {
            LlmFactory.from(UserLlmConfig("nope", "k"))
        }
    }
}
```

- [ ] **Step 2: 跑测试确认失败**

Run: `./gradlew :agent-llm:test`
Expected: FAIL,编译错误。

- [ ] **Step 3: 写最小实现**

`agent-llm/src/main/kotlin/com/you/agent/llm/UserLlmConfig.kt`:
```kotlin
package com.you.agent.llm

/** 用户自定义 LLM 配置(BYO Key)。宿主可暴露给最终用户填写,Phase 2 接入运行时切换。 */
data class UserLlmConfig(
    val providerId: String,           // "deepseek" / "qwen" / "glm" / "moonshot" / "custom"
    val apiKey: String,               // 用户自有 key
    val baseUrl: String? = null,      // custom 模式必填;预设模式可覆盖默认 baseUrl
    val model: String? = null,        // 可选,覆盖默认模型名
    val temperature: Double? = null,
    val maxTokens: Int? = null,
)
```

`agent-llm/src/main/kotlin/com/you/agent/llm/internal/OpenAiDtos.kt`:
```kotlin
package com.you.agent.llm.internal

import kotlinx.serialization.SerialName
import kotlinx.serialization.Serializable

@Serializable
data class OpenAiChatRequest(
    val model: String,
    val messages: List<OpenAiMessage>,
    val temperature: Double? = null,
    @SerialName("max_tokens") val maxTokens: Int? = null,
    val stream: Boolean = false,
)

@Serializable
data class OpenAiMessage(
    val role: String,
    val content: String,
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

`agent-llm/src/main/kotlin/com/you/agent/llm/internal/OpenAiException.kt`:
```kotlin
package com.you.agent.llm.internal

class OpenAiException(val statusCode: Int, override val message: String) : RuntimeException(message)
```

`agent-llm/src/main/kotlin/com/you/agent/llm/internal/OpenAiClient.kt`:
```kotlin
package com.you.agent.llm.internal

import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.flow
import kotlinx.coroutines.flow.flowOn
import kotlinx.coroutines.suspendCancellableCoroutine
import kotlinx.serialization.encodeToString
import kotlinx.serialization.json.Json
import okhttp3.Call
import okhttp3.Callback
import okhttp3.MediaType.Companion.toMediaType
import okhttp3.OkHttpClient
import okhttp3.Request
import okhttp3.RequestBody.Companion.toRequestBody
import okhttp3.Response
import java.io.BufferedReader
import java.io.IOException
import java.util.concurrent.TimeUnit
import kotlin.coroutines.resume
import kotlin.coroutines.resumeWithException

class OpenAiClient(
    private val baseUrl: String,
    private val apiKey: String,
    private val client: OkHttpClient = defaultClient(),
) {
    private val json = Json { ignoreUnknownKeys = true; encodeDefaults = false }
    private val mediaJson = "application/json; charset=utf-8".toMediaType()

    suspend fun chat(dto: OpenAiChatRequest): OpenAiChatResponse {
        val body = json.encodeToString(dto).toRequestBody(mediaJson)
        val request = requestBuilder("/chat/completions").post(body).build()
        val text = executeForBodyText(request)
        return json.decodeFromString(text)
    }

    fun streamChunks(dto: OpenAiChatRequest): Flow<OpenAiStreamChunk> = flow {
        val streamDto = dto.copy(stream = true)
        val body = json.encodeToString(streamDto).toRequestBody(mediaJson)
        val request = requestBuilder("/chat/completions")
            .header("Accept", "text/event-stream")
            .post(body).build()
        val response = execute(request)
        response.use { resp ->
            if (!resp.isSuccessful) {
                val errText = resp.body?.string() ?: ""
                val err = runCatching { json.decodeFromString<OpenAiErrorResponse>(errText) }.getOrNull()
                throw OpenAiException(resp.code, err?.error?.message ?: errText)
            }
            val reader = BufferedReader(resp.body?.charStream() ?: java.io.StringReader(""))
            var line = reader.readLine()
            while (line != null) {
                if (line.startsWith("data:")) {
                    val data = line.removePrefix("data:").trim()
                    if (data == "[DONE]") break
                    if (data.isNotBlank()) {
                        runCatching { json.decodeFromString<OpenAiStreamChunk>(data) }
                            .onSuccess { emit(it) }
                    }
                }
                line = reader.readLine()
            }
        }
    }.flowOn(Dispatchers.IO)

    private fun requestBuilder(path: String): Request.Builder {
        val url = baseUrl.trimEnd('/') + if (path.startsWith("/")) path else "/$path"
        return Request.Builder().url(url)
            .header("Authorization", "Bearer $apiKey")
            .header("Accept", "application/json")
    }

    private suspend fun execute(request: Request): Response =
        suspendCancellableCoroutine { cont ->
            val call = client.newCall(request)
            cont.invokeOnCancellation { runCatching { call.cancel() } }
            call.enqueue(object : Callback {
                override fun onFailure(call: Call, e: IOException) { cont.resumeWithException(e) }
                override fun onResponse(call: Call, response: Response) { cont.resume(response) }
            })
        }

    private suspend fun executeForBodyText(request: Request): String {
        val response = execute(request)
        return response.use { resp ->
            val text = resp.body?.string() ?: ""
            if (!resp.isSuccessful) {
                val err = runCatching { json.decodeFromString<OpenAiErrorResponse>(text) }.getOrNull()
                throw OpenAiException(resp.code, err?.error?.message ?: text)
            }
            text
        }
    }

    companion object {
        fun defaultClient(): OkHttpClient = OkHttpClient.Builder()
            .connectTimeout(30, TimeUnit.SECONDS)
            .readTimeout(120, TimeUnit.SECONDS)
            .build()
    }
}
```

`agent-llm/src/main/kotlin/com/you/agent/llm/internal/OpenAiCompatibleProvider.kt`:
```kotlin
package com.you.agent.llm.internal

import com.you.agent.core.chat.ChatChunk
import com.you.agent.core.chat.ChatRequest
import com.you.agent.core.chat.ChatResponse
import com.you.agent.core.llm.LlmProvider
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.map

class OpenAiCompatibleProvider(
    private val baseUrl: String,
    private val apiKey: String,
    private val model: String,
    private val client: OpenAiClient = OpenAiClient(baseUrl, apiKey),
) : LlmProvider {
    override val modelId: String = model

    override suspend fun chat(request: ChatRequest): ChatResponse {
        val resp = client.chat(toDto(request))
        val choice = resp.choices.firstOrNull()
        val text = choice?.message?.content ?: error("OpenAI 兼容响应无内容: $resp")
        return ChatResponse(content = text, model = resp.model ?: model, finishReason = choice.finishReason)
    }

    override fun streamChat(request: ChatRequest): Flow<ChatChunk> =
        client.streamChunks(toDto(request)).map { chunk ->
            val choice = chunk.choices.firstOrNull()
            ChatChunk(delta = choice?.delta?.content ?: "", done = choice?.finishReason != null)
        }

    private fun toDto(request: ChatRequest): OpenAiChatRequest =
        OpenAiChatRequest(
            model = request.model,
            messages = request.messages.map { OpenAiMessage(role = it.role.name.lowercase(), content = it.content) },
            temperature = request.temperature,
            maxTokens = request.maxTokens,
        )
}
```

`agent-llm/src/main/kotlin/com/you/agent/llm/Providers.kt`:
```kotlin
package com.you.agent.llm

import com.you.agent.core.llm.LlmProvider
import com.you.agent.llm.internal.OpenAiCompatibleProvider

object Providers {
    fun deepSeek(key: String): LlmProvider =
        OpenAiCompatibleProvider("https://api.deepseek.com/v1", key, "deepseek-chat")
    fun qwen(key: String): LlmProvider =
        OpenAiCompatibleProvider("https://dashscope.aliyuncs.com/compatible-mode/v1", key, "qwen-plus")
    fun glm(key: String): LlmProvider =
        OpenAiCompatibleProvider("https://open.bigmodel.cn/api/paas/v4", key, "glm-4")
    fun moonshot(key: String): LlmProvider =
        OpenAiCompatibleProvider("https://api.moonshot.cn/v1", key, "moonshot-v1-8k")
    fun custom(baseUrl: String, key: String, model: String): LlmProvider =
        OpenAiCompatibleProvider(baseUrl, key, model)
}
```

`agent-llm/src/main/kotlin/com/you/agent/llm/LlmFactory.kt`:
```kotlin
package com.you.agent.llm

import com.you.agent.core.llm.LlmProvider
import com.you.agent.llm.internal.OpenAiCompatibleProvider

object LlmFactory {
    fun from(config: UserLlmConfig): LlmProvider = when (config.providerId) {
        "deepseek" -> OpenAiCompatibleProvider(
            config.baseUrl ?: "https://api.deepseek.com/v1",
            config.apiKey, config.model ?: "deepseek-chat")
        "qwen" -> OpenAiCompatibleProvider(
            config.baseUrl ?: "https://dashscope.aliyuncs.com/compatible-mode/v1",
            config.apiKey, config.model ?: "qwen-plus")
        "glm" -> OpenAiCompatibleProvider(
            config.baseUrl ?: "https://open.bigmodel.cn/api/paas/v4",
            config.apiKey, config.model ?: "glm-4")
        "moonshot" -> OpenAiCompatibleProvider(
            config.baseUrl ?: "https://api.moonshot.cn/v1",
            config.apiKey, config.model ?: "moonshot-v1-8k")
        "custom" -> OpenAiCompatibleProvider(
            requireNotNull(config.baseUrl) { "custom provider 必须提供 baseUrl" },
            config.apiKey,
            requireNotNull(config.model) { "custom provider 必须提供 model" })
        else -> error("未知 providerId: ${config.providerId}")
    }
}
```

- [ ] **Step 4: 跑测试确认通过**

Run: `./gradlew :agent-llm:test`
Expected: BUILD SUCCESSFUL,8 个工厂测试通过。

- [ ] **Step 5: 提交**

```bash
git add agent-llm/src
git commit -m "feat(agent-llm): add BYO config, Providers, LlmFactory, and OpenAI-compatible provider"
```

---

## Task 7: agent-llm 非流式 chat 的 MockWebServer 测试

**Files:**
- Test: `agent-android/agent-llm/src/test/kotlin/com/you/agent/llm/OpenAiCompatibleProviderChatTest.kt`
- Modify: 无实现改动(验证 Task 6 的实现)

**Interfaces:**
- Consumes: `OpenAiCompatibleProvider`、`OpenAiException`、`ChatRequest`/`Message`(Task 6/2)
- Produces: 非流式 chat 行为验证(连通性探测与错误传播)

- [ ] **Step 1: 写测试**

`agent-llm/src/test/kotlin/com/you/agent/llm/OpenAiCompatibleProviderChatTest.kt`:
```kotlin
package com.you.agent.llm

import com.you.agent.core.chat.ChatRequest
import com.you.agent.core.chat.Message
import com.you.agent.llm.internal.OpenAiCompatibleProvider
import com.you.agent.llm.internal.OpenAiException
import kotlinx.coroutines.test.runTest
import okhttp3.mockwebserver.MockResponse
import okhttp3.mockwebserver.MockWebServer
import org.junit.jupiter.api.Assertions.assertEquals
import org.junit.jupiter.api.Assertions.assertThrows
import org.junit.jupiter.api.Assertions.assertTrue
import org.junit.jupiter.api.Test

class OpenAiCompatibleProviderChatTest {
    private fun provider(server: MockWebServer) =
        OpenAiCompatibleProvider(server.url("/v1").toString().trimEnd('/'), "key", "deepseek-chat")

    @Test
    fun `chat parses completion content`() = runTest {
        val server = MockWebServer()
        server.enqueue(MockResponse().setBody("""
            {"id":"1","model":"deepseek-chat","choices":[{"index":0,"message":{"role":"assistant","content":"hello back"},"finish_reason":"stop"}]}
        """.trimIndent()))
        server.start()
        try {
            val resp = provider(server).chat(
                ChatRequest(model = "deepseek-chat", messages = listOf(Message.user("hi")))
            )
            assertEquals("hello back", resp.content)
            assertEquals("deepseek-chat", resp.model)
            assertEquals("stop", resp.finishReason)
        } finally { server.shutdown() }
    }

    @Test
    fun `chat sends bearer auth and json body`() = runTest {
        val server = MockWebServer()
        server.enqueue(MockResponse().setBody("""
            {"choices":[{"message":{"role":"assistant","content":"ok"}}]}
        """.trimIndent()))
        server.start()
        try {
            provider(server).chat(ChatRequest("deepseek-chat", listOf(Message.user("hi"))))
            val recorded = server.takeRequest()
            assertEquals("/v1/chat/completions", recorded.path)
            assertEquals("Bearer key", recorded.getHeader("Authorization"))
            val body = recorded.body.readUtf8()
            assertTrue(body.contains("\"model\":\"deepseek-chat\""))
            assertTrue(body.contains("\"content\":\"hi\""))
        } finally { server.shutdown() }
    }

    @Test
    fun `non-2xx throws OpenAiException with status`() = runTest {
        val server = MockWebServer()
        server.enqueue(MockResponse().setResponseCode(401).setBody("""{"error":{"message":"bad key"}}"""))
        server.start()
        try {
            val ex = assertThrows(OpenAiException::class.java) {
                kotlinx.coroutines.runBlocking {
                    provider(server).chat(ChatRequest("deepseek-chat", listOf(Message.user("hi"))))
                }
            }
            assertEquals(401, ex.statusCode)
            assertEquals("bad key", ex.message)
        } finally { server.shutdown() }
    }
}
```

- [ ] **Step 2: 跑测试**

Run: `./gradlew :agent-llm:test`
Expected: BUILD SUCCESSFUL,3 个测试通过。若 `runBlocking` 包裹与 `runTest` 冲突,改用 `runTest { assertThrows<OpenAiException> { provider(server).chat(...) } }`(第三个测试相应改写)。

- [ ] **Step 3: 提交**

```bash
git add agent-llm/src
git commit -m "test(agent-llm): cover non-streaming chat with MockWebServer"
```

---

## Task 8: agent-llm 流式 streamChat 的 MockWebServer 测试

**Files:**
- Test: `agent-android/agent-llm/src/test/kotlin/com/you/agent/llm/OpenAiCompatibleProviderStreamTest.kt`
- Modify: 无实现改动

**Interfaces:**
- Consumes: `OpenAiCompatibleProvider.streamChat`、`ChatChunk`(Task 6/2)
- Produces: 流式行为验证

- [ ] **Step 1: 写测试**

`agent-llm/src/test/kotlin/com/you/agent/llm/OpenAiCompatibleProviderStreamTest.kt`:
```kotlin
package com.you.agent.llm

import com.you.agent.core.chat.ChatRequest
import com.you.agent.core.chat.Message
import com.you.agent.llm.internal.OpenAiCompatibleProvider
import kotlinx.coroutines.flow.toList
import kotlinx.coroutines.test.runTest
import okhttp3.mockwebserver.MockResponse
import okhttp3.mockwebserver.MockWebServer
import org.junit.jupiter.api.Assertions.assertEquals
import org.junit.jupiter.api.Test

class OpenAiCompatibleProviderStreamTest {
    @Test
    fun `streamChat emits deltas and final done`() = runTest {
        val server = MockWebServer()
        val sse = """
            data: {"choices":[{"index":0,"delta":{"role":"assistant","content":"Hel"}}]}

            data: {"choices":[{"index":0,"delta":{"content":"lo"}}]}

            data: {"choices":[{"index":0,"delta":{},"finish_reason":"stop"}]}

            data: [DONE]

        """.trimIndent()
        server.enqueue(MockResponse().setHeader("Content-Type", "text/event-stream").setBody(sse))
        server.start()
        try {
            val provider = OpenAiCompatibleProvider(server.url("/v1").toString().trimEnd('/'), "key", "deepseek-chat")
            val chunks = provider.streamChat(ChatRequest("deepseek-chat", listOf(Message.user("hi")))).toList()
            assertEquals(listOf("Hel", "lo", ""), chunks.map { it.delta })
            assertEquals(listOf(false, false, true), chunks.map { it.done })
        } finally { server.shutdown() }
    }
}
```

- [ ] **Step 2: 跑测试**

Run: `./gradlew :agent-llm:test`
Expected: BUILD SUCCESSFUL。

- [ ] **Step 3: 提交**

```bash
git add agent-llm/src
git commit -m "test(agent-llm): cover streaming chat with SSE MockWebServer"
```

---

## Task 9: 端到端集成 — agent-core + agent-llm(MockWebServer)

**Files:**
- Test: `agent-android/agent-core/src/test/kotlin/com/you/agent/core/EndToEndMinimalAgentTest.kt`

**Interfaces:**
- Consumes: `Agent.builder()`、`OpenAiCompatibleProvider`(Task 5/6)、`ChatRequest`(Task 2)
- Produces: M1 验收——无真实网络下端到端对话闭环

- [ ] **Step 1: 写测试**

注:此测试放在 `agent-core`,需要 `agent-core` 测试依赖 `agent-llm`。在 `agent-core/build.gradle.kts` 的 `dependencies` 增加:
```kotlin
testImplementation(project(":agent-llm"))
testImplementation(libs.okhttp.mockwebserver)
```

`agent-core/src/test/kotlin/com/you/agent/core/EndToEndMinimalAgentTest.kt`:
```kotlin
package com.you.agent.core

import com.you.agent.core.chat.ChatRequest
import com.you.agent.core.chat.Message
import com.you.agent.llm.internal.OpenAiCompatibleProvider
import kotlinx.coroutines.flow.toList
import kotlinx.coroutines.test.runTest
import okhttp3.mockwebserver.MockResponse
import okhttp3.mockwebserver.MockWebServer
import org.junit.jupiter.api.Assertions.assertEquals
import org.junit.jupiter.api.Assertions.assertInstanceOf
import org.junit.jupiter.api.Test

class EndToEndMinimalAgentTest {
    @Test
    fun `user input flows through provider to Answer`() = runTest {
        val server = MockWebServer()
        server.enqueue(MockResponse().setBody("""
            {"model":"deepseek-chat","choices":[{"message":{"role":"assistant","content":"hi there"}}]}
        """.trimIndent()))
        server.start()
        try {
            val provider = OpenAiCompatibleProvider(
                server.url("/v1").toString().trimEnd('/'), "key", "deepseek-chat")
            val agent = Agent.builder()
                .llm(provider)
                .persona(com.you.agent.core.persona.Persona(id = "p", name = "P", systemPrompt = "sys"))
                .build()

            val events = agent.run("ping").toList()

            assertEquals(1, events.size)
            assertInstanceOf(AgentEvent.Answer::class.java, events[0])
            assertEquals("hi there", (events[0] as AgentEvent.Answer).text)
            // 验证 persona system prompt 被发送
            val recorded = server.takeRequest()
            val body = recorded.body.readUtf8()
            assert(body.contains("\"role\":\"system\""))
            assert(body.contains("\"content\":\"sys\""))
        } finally { server.shutdown() }
    }
}
```

- [ ] **Step 2: 跑全部测试**

Run: `./gradlew test`
Expected: BUILD SUCCESSFUL,所有模块测试通过(含端到端)。

- [ ] **Step 3: 提交**

```bash
git add agent-core/build.gradle.kts agent-core/src
git commit -m "test: end-to-end minimal agent conversation via MockWebServer"
```

---

## Self-Review

**1. Spec coverage(对照设计文档 M1 范围):**
- 多模块脚手架 → Task 1 ✓
- `agent-core` 门面(`Agent`/`AgentEvent`/`Builder`) → Task 2/3/4/5 ✓
- `agent-llm` 国产模型 Provider(OpenAI 兼容) → Task 6/7/8 ✓
- BYO Key 接口骨架(`UserLlmConfig` + `LlmFactory`) → Task 6 ✓
- 跑通最小对话(无工具) → Task 5(假 Provider)+ Task 9(真 HTTP via Mock)✓
- M1 用 `Providers.deepSeek(testKey)` 跑通:UI 配置入口暂不实现 ✓(符合 M1 不做 UI)
- 设计文档"M1 内部委托 ADK Runner":本计划**显式推迟**到 M2,理由(YAGNI + CI 可测性)已写入 Architecture;抽象层屏蔽差异,不破坏宿主 API。这是对设计的有意收窄,需在执行前与用户确认(见下)。

未覆盖(留给后续里程碑,符合范围):`@Tool` DSL 实装(M2)、预设角色 PersonaStore 实现(M3)、RAG(M4)、运行时 `reconfigure()`(M6)、sample App(M7)。

**2. Placeholder scan:** 无 TBD/TODO;每步含完整代码或确切命令。✓

**3. Type consistency:**
- `LlmProvider.modelId` / `chat` / `streamChat` 在 Task 3 定义,Task 5/6 一致使用 ✓
- `AgentEvent.Answer(text)` Task 4 定义,Task 5/9 断言一致 ✓
- `Agent.builder()` Task 4 定义 companion + Task 5 扩展,Task 9 使用 ✓
- `OpenAiCompatibleProvider(baseUrl, apiKey, model)` Task 6 定义,Task 7/8/9 构造签名一致 ✓
- `UserLlmConfig(providerId, apiKey, baseUrl?, model?, ...)` Task 6 定义,工厂测试一致 ✓
- `ChatRequest(model, messages, temperature?, maxTokens?)` Task 2 定义,全程一致 ✓

---

## Execution Handoff

计划已保存至 `docs/superpowers/plans/2026-07-01-android-agent-m1.md`。

**一处需用户确认:** 本计划将设计文档"agent-core 内部委托 ADK Runner"在 M1 显式收窄为"`MinimalAgent` 直接调 `LlmProvider`",ADK 依赖整体推迟到 M2。理由是 M1(无工具、单轮对话)用不到 ReAct 循环,且纯 JVM 单测在 CI 上更稳。`Agent`/`LlmProvider` 抽象层保证后续切换零宿主改动。**如认可此收窄,即可进入执行;如坚持 M1 即引入 ADK,我改写 Task 5 为 ADK InMemoryRunner 版本。**

两种执行方式:

**1. Subagent-Driven(推荐)** — 每个任务派发独立 subagent,任务间两阶段评审,迭代快。

**2. Inline Execution** — 在当前会话按 executing-plans 批量执行,带检查点评审。

请选择执行方式(并确认上述 ADK 收窄是否接受)。
