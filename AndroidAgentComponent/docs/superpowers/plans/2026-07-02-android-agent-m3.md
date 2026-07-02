# Android Agent 组件 — M3 实现计划(预设角色 + 会话记忆)

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 在 M2 工具调用闭环之上,实现 (1) `agent-memory` 模块提供预设角色(JSON classpath 加载,只读 `PresetPersonaStore`);(2) 跨轮会话记忆(`ConversationStore` 接口 + `InMemoryConversationStore` 实现),让 Agent 能在多轮对话中保留上下文。全程纯 JVM 可单测,无真实网络、无 Android SDK 依赖。

**Architecture:** 沿用 M1/M2 分层与手写路线,**不引入 ADK Session / genai SDK**。`agent-core` 新增 `ConversationStore` 接口(与 `PersonaStore` 同级,供 ReActAgent/MinimalAgent 注入);新建 `agent-memory` 模块提供两个实现:`PresetPersonaStore`(从 classpath `personas/*.json` 加载,Phase 1 只读,`save()` 抛 `UnsupportedOperationException` 为 M5 预留)与 `InMemoryConversationStore`(`Map<sessionId, MutableList<Message>>`,Phase 1 内存,持久化留 M5)。`AgentBuilder` 加 `conversation(store)` 方法;`Agent.run()` 签名加可选 `sessionId: String? = null`(向后兼容 M2);`ReActAgent`/`MinimalAgent` 在 run 开始时按 sessionId `load` 历史拼到 system 之后、user 之前,run 结束时 `append` 本轮新增消息(user + assistant + tool)。M3 上下文注入用**全量历史**(最简,滑窗/摘要留 M5 token 预算时做)。

**关键决策(基于 M2 既定方向收敛):**
1. **预设角色加载机制**:用 classpath resources(`src/main/resources/personas/*.json` + `index.txt`),`ClassLoader.getResourceAsStream` 读取。理由:纯 JVM 可单测(Android 宿主打包时 resources 自动进 APK classpath,无需 `Context`);比 `assets/` 更跨平台;比 Kotlin 对象更灵活(用户可在自己的 agent-memory 扩展里加 JSON 不改代码)。设计文档 §6 写 `assets/`,本计划改为 `resources/`,功能等价且可测性更好。
2. **会话记忆架构**:手写 `ConversationStore` + `InMemoryConversationStore`,不引 ADK `InMemorySessionService`。理由同 M2(ADK Kotlin 0.2.0 无 OpenAI 兼容 Model,且 Session 体系绑定 genai 类型);手写轻量 Map 实现纯 JVM 可测,`ConversationStore` 接口保证未来可换持久化实现(M5 Room/DataStore)。
3. **`PersonaStore.save()` Phase 1 行为**:`PresetPersonaStore` 只读,`save()` 抛 `UnsupportedOperationException("PresetPersonaStore 只读,M5 提供可写实现")`。接口保留 `save()` 不动(M1 落地 + 设计文档),M5 提供可写实现时只需新增 `WritablePersonaStore` 类,不改接口。
4. **`Agent.run()` 签名**:加 `sessionId: String? = null`。null 表示无状态单轮(向后兼容 M2 全部测试);非空 + 注入 `ConversationStore` 时启用多轮。`personaId` 参数 M3 仍不接入运行时(留 M5 自定义角色按 ID 加载)。
5. **上下文注入策略**:全量历史(M3 最简)。`load(sessionId)` 返回 `List<Message>`,拼到 `[system]` 之后、`[user]` 之前;本轮新增消息(user + 后续 assistant/tool)在 run 结束时 `append`。滑窗/摘要留 M5(token 预算 + 持久化一起做)。

**Tech Stack:** Kotlin 2.1.20、Kotlinx Coroutines/Flow、Kotlinx Serialization(JsonObject DTO)、JUnit 5、OkHttp MockWebServer。

**Context — M2 已落地(本计划基线):**
- `agent-core`:`Message(role, content, toolCalls?, toolCallId?)` / `ChatRequest(model, messages, temperature?, maxTokens?, tools?)` / `ChatResponse(content, model, finishReason?, toolCalls?)` / `ChatChunk` / `ToolCall(id, name, args)` / `ToolDeclaration` / `Tool` / `MapToolRegistry` / `ToolRegistryScope` / `LlmProvider` / `Persona` / `PersonaStore`(接口,M3 实现在 agent-memory)/ `KnowledgeRetriever`/`KnowledgeChunk`(M4)/ `AgentEvent` sealed / `AgentConfig(maxSteps=8, temperature?, maxTokens?)` / `Agent` 门面 / `AgentBuilder`(`llm/persona/memory/rag/tools/config/build`,`memory/rag` M3 仍预留) / `internal MinimalAgent`(单轮)/ `internal ReActAgent`(工具循环,maxSteps 守卫)。
- `agent-llm`:`UserLlmConfig` / `Providers` / `LlmFactory` / `internal OpenAiClient` / `internal OpenAiCompatibleProvider`(双向 function calling 适配)/ DTO。
- `agent-tools`:`jsonSchema { }` DSL / `tool()` 工厂 / `getCurrentTime()` 内置工具。
- 测试:`:agent-core` 22 / `:agent-llm` 17 / `:agent-tools` 6,共 45 个全绿。
- `settings.gradle.kts`:`include(":agent-core", ":agent-llm", ":agent-tools")`。

## Global Constraints

- 纯 Kotlin/JVM 多模块,M3 不依赖 Android SDK、不依赖 ADK、不依赖 google-genai、不依赖 Room/DataStore。
- 组件**不内置任何 API key**;工具 handler 不得在日志/异常中输出 key。
- 所有公共 API 用 Kotlin + Coroutines/Flow,无 UI 耦合。
- 预设角色 JSON 走 classpath resources(`src/main/resources/personas/`),不读 Android `assets/`、不要 `Context`。
- 会话记忆走 `ConversationStore` 接口,默认 `InMemoryConversationStore`(Map),不引 ADK Session。
- `Agent.run()` 加 `sessionId: String? = null`,**向后兼容 M2**(M2 全部测试不传 sessionId 仍通过)。
- 上下文注入用全量历史(不滑窗、不摘要),`append` 本轮新增消息(不含 system、不含已 load 的 history)。
- 包名前缀统一 `com.you.agent`。
- TDD:先写失败测试,再写最小实现,再跑通,再提交。**不弱化断言**;若测试与实现冲突,先查根因。
- 每个任务结束提交一次,提交信息以 `feat:`/`chore:`/`test:`/`refactor:` 前缀。
- DRY、YAGNI:不实现 M3 范围外的能力(角色可写 M5 / RAG M4 / 滑窗摘要 M5 / 持久化 M5)。

---

## File Structure(本计划新增/改动)

```
agent-android/
├── settings.gradle.kts                  # 改: include +(":agent-memory")
├── gradle/libs.versions.toml            # 不动(agent-memory 复用现有 serialization-json)
├── agent-core/
│   ├── build.gradle.kts                 # 不动
│   └── src/
│       ├── main/kotlin/com/you/agent/core/
│       │   ├── Agent.kt                 # 改: Agent.run() 加 sessionId: String? = null
│       │   ├── AgentBuilder.kt          # 改: +conversation(store) 字段与方法;build() 传入
│       │   ├── conversation/
│       │   │   └── ConversationStore.kt # 新增: 接口(load/append/clear)
│       │   └── internal/
│       │       ├── MinimalAgent.kt      # 改: +conversation 注入;run 历史拼装 + append
│       │       └── ReActAgent.kt        # 改: +conversation 注入;run 历史拼装 + append
│       └── test/kotlin/com/you/agent/core/
│           ├── ConversationAgentTest.kt # 新增: FakeLlm 多轮历史闭环
│           └── EndToEndConversationTest.kt # 新增: MockWebServer 多轮带历史
└── agent-memory/                        # 新模块
    ├── build.gradle.kts                 # 新增
    └── src/
        ├── main/kotlin/com/you/agent/memory/
        │   ├── PersonaJson.kt           # 新增: @Serializable DTO + toPersona()
        │   ├── PresetPersonaStore.kt    # 新增: classpath JSON 只读 PersonaStore
        │   └── InMemoryConversationStore.kt # 新增: Map 实现 ConversationStore
        ├── main/resources/personas/
        │   ├── index.txt                # 新增: 每行一个 persona id
        │   ├── assistant_v1.json        # 新增: 通用助手
        │   ├── coder_v1.json            # 新增: 编程助手
        │   └── translator_v1.json       # 新增: 翻译助手
        └── test/kotlin/com/you/agent/memory/
            ├── PresetPersonaStoreTest.kt # 新增
            └── InMemoryConversationStoreTest.kt # 新增
```

职责边界:`agent-core` 只含 `ConversationStore` 抽象(ReActAgent 依赖它),零 IO;`agent-memory` 提供 `PresetPersonaStore`(读 classpath JSON)+ `InMemoryConversationStore`(Map)+ 预设角色 JSON 资源,依赖 `agent-core`。

---

## Task 1: agent-memory 模块 + PresetPersonaStore + 预设角色 JSON

**Files:**
- Modify: `agent-android/settings.gradle.kts`
- Create: `agent-android/agent-memory/build.gradle.kts`
- Create: `agent-android/agent-memory/src/main/kotlin/com/you/agent/memory/PersonaJson.kt`
- Create: `agent-android/agent-memory/src/main/kotlin/com/you/agent/memory/PresetPersonaStore.kt`
- Create: `agent-android/agent-memory/src/main/resources/personas/index.txt`
- Create: `agent-android/agent-memory/src/main/resources/personas/assistant_v1.json`
- Create: `agent-android/agent-memory/src/main/resources/personas/coder_v1.json`
- Create: `agent-android/agent-memory/src/main/resources/personas/translator_v1.json`
- Test: `agent-android/agent-memory/src/test/kotlin/com/you/agent/memory/PresetPersonaStoreTest.kt`

**Interfaces:**
- Consumes: `Persona`/`PersonaStore`(M1,agent-core)
- Produces: `PresetPersonaStore`(供宿主 `.persona(store.load("assistant_v1"))` 或后续 M5 按 ID 加载)、3 个预设角色 JSON

- [ ] **Step 1: 模块脚手架**

`agent-android/settings.gradle.kts`:
```kotlin
rootProject.name = "agent-android"
include(":agent-core", ":agent-llm", ":agent-tools", ":agent-memory")
```

`agent-android/agent-memory/build.gradle.kts`:
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

`agent-memory/src/test/kotlin/com/you/agent/memory/PresetPersonaStoreTest.kt`:
```kotlin
package com.you.agent.memory

import kotlinx.coroutines.test.runTest
import org.junit.jupiter.api.Assertions.assertEquals
import org.junit.jupiter.api.Assertions.assertThrows
import org.junit.jupiter.api.Assertions.assertTrue
import org.junit.jupiter.api.Test

class PresetPersonaStoreTest {
    private val store = PresetPersonaStore()

    @Test
    fun `load assistant_v1 returns persona with system prompt`() = runTest {
        val p = store.load("assistant_v1")
        assertEquals("assistant_v1", p.id)
        assertEquals("通用助手", p.name)
        assertTrue(p.systemPrompt.isNotBlank())
    }

    @Test
    fun `list returns all preset personas from index`() {
        val list = store.list()
        val ids = list.map { it.id }
        assertTrue(ids.contains("assistant_v1"))
        assertTrue(ids.contains("coder_v1"))
        assertTrue(ids.contains("translator_v1"))
        assertEquals(3, list.size)
    }

    @Test
    fun `load unknown id throws`() = runTest {
        val ex = assertThrows(IllegalStateException::class.java) { store.load("not_exist") }
        assertTrue(ex.message!!.contains("not_exist"))
    }

    @Test
    fun `save throws UnsupportedOperationException`() = runTest {
        val p = store.load("assistant_v1")
        assertThrows(UnsupportedOperationException::class.java) { store.save(p) }
    }

    @Test
    fun `load is idempotent (cached)`() = runTest {
        val a = store.load("coder_v1")
        val b = store.load("coder_v1")
        assertEquals(a, b)
        assertEquals("编程助手", a.name)
    }
}
```

- [ ] **Step 3: 跑测试确认失败**

Run: `./gradlew :agent-memory:test`
Expected: FAIL,编译错误(模块/类未定义,首次需 Gradle 识别新模块,可先 `./gradlew projects` 确认 `:agent-memory` 出现)。

- [ ] **Step 4: 写最小实现**

`agent-memory/src/main/kotlin/com/you/agent/memory/PersonaJson.kt`:
```kotlin
package com.you.agent.memory

import com.you.agent.core.persona.Persona
import kotlinx.serialization.SerialName
import kotlinx.serialization.Serializable

/** 预设角色的 JSON 序列化形式(与 Persona 字段对齐)。 */
@Serializable
data class PersonaJson(
    val id: String,
    val name: String,
    @SerialName("systemPrompt") val systemPrompt: String,
    val traits: Map<String, String> = emptyMap(),
    @SerialName("knowledgeSources") val knowledgeSources: List<String> = emptyList(),
    val tools: List<String> = emptyList(),
)

internal fun PersonaJson.toPersona(): Persona = Persona(
    id = id,
    name = name,
    systemPrompt = systemPrompt,
    traits = traits,
    knowledgeSources = knowledgeSources,
    tools = tools,
)
```

`agent-memory/src/main/kotlin/com/you/agent/memory/PresetPersonaStore.kt`:
```kotlin
package com.you.agent.memory

import com.you.agent.core.persona.Persona
import com.you.agent.core.persona.PersonaStore
import kotlinx.serialization.json.Json

/**
 * Phase 1 预设角色存储:从 classpath `personas/*.json` 加载,只读。
 * 构造时一次性加载所有角色到内存(避免 list() 的 runBlocking)。
 * save() 抛 UnsupportedOperationException(M5 提供可写实现)。
 */
class PresetPersonaStore(
    private val classLoader: ClassLoader = Thread.currentThread().contextClassLoader
        ?: PresetPersonaStore::class.java.classLoader,
) : PersonaStore {
    private val json = Json { ignoreUnknownKeys = true }
    private val personas: Map<String, Persona> = loadAll()

    override suspend fun load(id: String): Persona =
        personas[id] ?: error("预设角色不存在: $id, 可用: ${personas.keys}")

    override fun list(): List<Persona> = personas.values.toList()

    override suspend fun save(persona: Persona): Unit =
        throw UnsupportedOperationException("PresetPersonaStore 只读,M5 提供可写实现")

    private fun loadAll(): Map<String, Persona> {
        val index = classLoader.getResourceAsStream("personas/index.txt")
            ?.bufferedReader()?.use { it.readLines().map(String::trim).filter(String::isNotBlank) }
            ?: emptyList()
        return index.associateWith { id ->
            val stream = classLoader.getResourceAsStream("personas/$id.json")
                ?: error("预设角色文件缺失: personas/$id.json")
            val text = stream.bufferedReader().use { it.readText() }
            json.decodeFromString(PersonaJson.serializer(), text).toPersona()
        }
    }
}
```

`agent-memory/src/main/resources/personas/index.txt`:
```
assistant_v1
coder_v1
translator_v1
```

`agent-memory/src/main/resources/personas/assistant_v1.json`:
```json
{
  "id": "assistant_v1",
  "name": "通用助手",
  "systemPrompt": "你是一个乐于助人的通用助手。请用简洁清晰的中文回答用户问题。",
  "traits": {},
  "knowledgeSources": [],
  "tools": []
}
```

`agent-memory/src/main/resources/personas/coder_v1.json`:
```json
{
  "id": "coder_v1",
  "name": "编程助手",
  "systemPrompt": "你是一个专业的编程助手。请给出准确、可运行的代码,并解释关键点。优先使用 Kotlin/Java。",
  "traits": { "style": "concise" },
  "knowledgeSources": [],
  "tools": []
}
```

`agent-memory/src/main/resources/personas/translator_v1.json`:
```json
{
  "id": "translator_v1",
  "name": "翻译助手",
  "systemPrompt": "你是一个专业翻译。用户输入中文时翻译为英文,输入英文时翻译为中文,保持语义与语气。",
  "traits": { "bilingual": "true" },
  "knowledgeSources": [],
  "tools": []
}
```

- [ ] **Step 5: 跑测试确认通过**

Run: `./gradlew :agent-memory:test`
Expected: BUILD SUCCESSFUL,5 个测试通过。

- [ ] **Step 6: 提交**

```bash
git add settings.gradle.kts agent-memory
git commit -m "feat(agent-memory): add PresetPersonaStore with 3 preset personas from classpath JSON"
```

---

## Task 2: agent-core ConversationStore 接口 + agent-memory InMemoryConversationStore

**Files:**
- Create: `agent-android/agent-core/src/main/kotlin/com/you/agent/core/conversation/ConversationStore.kt`
- Create: `agent-android/agent-memory/src/main/kotlin/com/you/agent/memory/InMemoryConversationStore.kt`
- Test: `agent-android/agent-memory/src/test/kotlin/com/you/agent/memory/InMemoryConversationStoreTest.kt`

**Interfaces:**
- Consumes: `Message`(M2,agent-core)
- Produces: `ConversationStore` 接口(agent-core,供 Task 3 ReActAgent/MinimalAgent 注入)、`InMemoryConversationStore`(agent-memory,Map 实现)

- [ ] **Step 1: 写失败测试**

`agent-memory/src/test/kotlin/com/you/agent/memory/InMemoryConversationStoreTest.kt`:
```kotlin
package com.you.agent.memory

import com.you.agent.core.chat.Message
import kotlinx.coroutines.test.runTest
import org.junit.jupiter.api.Assertions.assertEquals
import org.junit.jupiter.api.Assertions.assertTrue
import org.junit.jupiter.api.Test

class InMemoryConversationStoreTest {
    @Test
    fun `load unknown session returns empty list`() = runTest {
        val store = InMemoryConversationStore()
        assertTrue(store.load("s1").isEmpty())
    }

    @Test
    fun `append then load returns appended messages`() = runTest {
        val store = InMemoryConversationStore()
        val msgs = listOf(Message.user("hi"), Message.assistant("hello"))
        store.append("s1", msgs)
        assertEquals(msgs, store.load("s1"))
    }

    @Test
    fun `append twice accumulates in order`() = runTest {
        val store = InMemoryConversationStore()
        store.append("s1", listOf(Message.user("first")))
        store.append("s1", listOf(Message.assistant("resp1")))
        val loaded = store.load("s1")
        assertEquals(2, loaded.size)
        assertEquals("first", loaded[0].content)
        assertEquals("resp1", loaded[1].content)
    }

    @Test
    fun `sessions are isolated by id`() = runTest {
        val store = InMemoryConversationStore()
        store.append("s1", listOf(Message.user("a")))
        store.append("s2", listOf(Message.user("b")))
        assertEquals("a", store.load("s1")[0].content)
        assertEquals("b", store.load("s2")[0].content)
    }

    @Test
    fun `clear removes session`() = runTest {
        val store = InMemoryConversationStore()
        store.append("s1", listOf(Message.user("x")))
        store.clear("s1")
        assertTrue(store.load("s1").isEmpty())
    }
}
```

- [ ] **Step 2: 跑测试确认失败**

Run: `./gradlew :agent-memory:test`
Expected: FAIL,编译错误(`ConversationStore`/`InMemoryConversationStore` 未定义)。

- [ ] **Step 3: 写最小实现**

`agent-core/src/main/kotlin/com/you/agent/core/conversation/ConversationStore.kt`:
```kotlin
package com.you.agent.core.conversation

import com.you.agent.core.chat.Message

/**
 * 会话记忆存储:按 sessionId 存取对话历史。
 * Phase 1 用 InMemoryConversationStore(Map);M5 提供持久化实现(Room/DataStore)。
 */
interface ConversationStore {
    /** 加载指定会话的历史消息(不含 system,system 由 persona 每轮重新注入)。 */
    suspend fun load(sessionId: String): List<Message>

    /** 追加本轮新增消息到指定会话。 */
    suspend fun append(sessionId: String, messages: List<Message>)

    /** 清空指定会话历史。 */
    suspend fun clear(sessionId: String)
}
```

`agent-memory/src/main/kotlin/com/you/agent/memory/InMemoryConversationStore.kt`:
```kotlin
package com.you.agent.memory

import com.you.agent.core.chat.Message
import com.you.agent.core.conversation.ConversationStore

/** Phase 1 内存会话存储:Map<sessionId, MutableList<Message>>。非线程安全(M3 单测足够,M5 持久化时再处理并发)。 */
class InMemoryConversationStore : ConversationStore {
    private val sessions = mutableMapOf<String, MutableList<Message>>()

    override suspend fun load(sessionId: String): List<Message> =
        sessions[sessionId]?.toList() ?: emptyList()

    override suspend fun append(sessionId: String, messages: List<Message>) {
        sessions.getOrPut(sessionId) { mutableListOf() }.addAll(messages)
    }

    override suspend fun clear(sessionId: String) {
        sessions.remove(sessionId)
    }
}
```

- [ ] **Step 4: 跑测试确认通过**

Run: `./gradlew :agent-memory:test`
Expected: BUILD SUCCESSFUL,Task 1 的 5 个 + Task 2 的 5 个 = 10 个测试通过。

- [ ] **Step 5: 提交**

```bash
git add agent-core/src agent-memory/src
git commit -m "feat(agent-core,agent-memory): add ConversationStore interface and InMemoryConversationStore"
```

---

## Task 3: AgentBuilder.conversation() + ReActAgent/MinimalAgent 历史注入 + Agent.run() sessionId

**Files:**
- Modify: `agent-android/agent-core/src/main/kotlin/com/you/agent/core/Agent.kt`
- Modify: `agent-android/agent-core/src/main/kotlin/com/you/agent/core/AgentBuilder.kt`
- Modify: `agent-android/agent-core/src/main/kotlin/com/you/agent/core/internal/MinimalAgent.kt`
- Modify: `agent-android/agent-core/src/main/kotlin/com/you/agent/core/internal/ReActAgent.kt`
- Test: `agent-android/agent-core/src/test/kotlin/com/you/agent/core/ConversationAgentTest.kt`

**Interfaces:**
- Consumes: `ConversationStore`(Task 2)、`Message`/`ChatRequest`(M2)、`Agent`/`AgentBuilder`/`AgentEvent`/`AgentConfig`(M1/M2)
- Produces: `Agent.run(sessionId)` 多轮、`AgentBuilder.conversation(store)`、ReActAgent/MinimalAgent 历史注入(供 Task 4/5)

- [ ] **Step 1: 写失败测试**

`agent-core/src/test/kotlin/com/you/agent/core/ConversationAgentTest.kt`:
```kotlin
package com.you.agent.core

import com.you.agent.core.chat.ChatChunk
import com.you.agent.core.chat.ChatRequest
import com.you.agent.core.chat.ChatResponse
import com.you.agent.core.chat.Message
import com.you.agent.core.conversation.ConversationStore
import com.you.agent.core.conversation.InMemoryConversationStoreShim
import com.you.agent.core.llm.LlmProvider
import com.you.agent.core.persona.Persona
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.flowOf
import kotlinx.coroutines.flow.toList
import kotlinx.coroutines.test.runTest
import org.junit.jupiter.api.Assertions.assertEquals
import org.junit.jupiter.api.Assertions.assertInstanceOf
import org.junit.jupiter.api.Test

class ConversationAgentTest {
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

    @Test
    fun `second turn request includes first turn history`() = runTest {
        val llm = ScriptedLlm().apply {
            enqueue(ChatResponse(content = "你好,我是助手", model = "fake", finishReason = "stop"))
            enqueue(ChatResponse(content = "你刚才说了你好", model = "fake", finishReason = "stop"))
        }
        val store = InMemoryConversationStoreShim()
        val agent = Agent.builder()
            .llm(llm)
            .persona(Persona("p", "P", "sys"))
            .conversation(store)
            .build()

        agent.run("你好", sessionId = "s1").toList()
        agent.run("我刚才说了什么", sessionId = "s1").toList()

        // 第二轮请求的 messages 应为 [system, user("你好"), assistant("你好,我是助手"), user("我刚才说了什么")]
        val secondReq = llm.requests[1]
        assertEquals(4, secondReq.messages.size)
        assertEquals(Message.Role.SYSTEM, secondReq.messages[0].role)
        assertEquals("你好", secondReq.messages[1].content)
        assertEquals(Message.Role.ASSISTANT, secondReq.messages[2].role)
        assertEquals("你好,我是助手", secondReq.messages[2].content)
        assertEquals(Message.Role.USER, secondReq.messages[3].role)
        assertEquals("我刚才说了什么", secondReq.messages[3].content)
    }

    @Test
    fun `no conversation store means stateless single turn`() = runTest {
        val llm = ScriptedLlm().apply {
            enqueue(ChatResponse(content = "r1", model = "fake", finishReason = "stop"))
            enqueue(ChatResponse(content = "r2", model = "fake", finishReason = "stop"))
        }
        val agent = Agent.builder()
            .llm(llm)
            .persona(Persona("p", "P", "sys"))
            .build()

        agent.run("a").toList()
        agent.run("b").toList()

        // 第二轮请求只有 [system, user("b")],无历史
        val secondReq = llm.requests[1]
        assertEquals(2, secondReq.messages.size)
        assertEquals("b", secondReq.messages[1].content)
    }

    @Test
    fun `sessionId null with store means stateless`() = runTest {
        val llm = ScriptedLlm().apply {
            enqueue(ChatResponse(content = "r1", model = "fake", finishReason = "stop"))
            enqueue(ChatResponse(content = "r2", model = "fake", finishReason = "stop"))
        }
        val store = InMemoryConversationStoreShim()
        val agent = Agent.builder()
            .llm(llm)
            .persona(Persona("p", "P", "sys"))
            .conversation(store)
            .build()

        agent.run("a").toList()  // sessionId = null
        agent.run("b").toList()  // sessionId = null

        val secondReq = llm.requests[1]
        assertEquals(2, secondReq.messages.size)  // 无历史
        assertEquals("b", secondReq.messages[1].content)
        // store 不应被写入(sessionId null 时不 load/append)
        assertEquals(0, store.sessionCount())
    }

    @Test
    fun `ReAct agent with conversation appends tool messages to history`() = runTest {
        val llm = ScriptedLlm().apply {
            enqueue(ChatResponse(content = "", model = "fake", finishReason = "tool_calls",
                toolCalls = listOf(com.you.agent.core.chat.ToolCall("c1", "echo", mapOf("text" to "hi")))))
            enqueue(ChatResponse(content = "done", model = "fake", finishReason = "stop"))
        }
        val store = InMemoryConversationStoreShim()
        val agent = Agent.builder()
            .llm(llm)
            .persona(Persona("p", "P", "sys"))
            .conversation(store)
            .tools {
                +com.you.agent.core.tools.Tool("echo", "d", "{}") { args -> args["text"].toString() }
            }
            .build()

        val events = agent.run("go", sessionId = "s1").toList()
        assertInstanceOf(AgentEvent.Answer::class.java, events.last())

        // store 应存 [user("go"), assistant(toolCalls), tool(result), assistant("done")]
        val history = store.load("s1")
        assertEquals(4, history.size)
        assertEquals(Message.Role.USER, history[0].role)
        assertEquals(Message.Role.ASSISTANT, history[1].role)
        assertEquals(Message.Role.TOOL, history[2].role)
        assertEquals(Message.Role.ASSISTANT, history[3].role)
        assertEquals("done", history[3].content)
    }
}
```

注:测试用 `InMemoryConversationStoreShim` 是 agent-core test 内的简单假实现(agent-core test 不依赖 agent-memory,避免循环依赖;真实 `InMemoryConversationStore` 在 agent-memory,Task 5 端到端用)。Step 3 会一并创建该 Shim。

- [ ] **Step 2: 跑测试确认失败**

Run: `./gradlew :agent-core:test`
Expected: FAIL,编译错误(`Agent.run` 无 sessionId 参数、`AgentBuilder.conversation` 未定义、`InMemoryConversationStoreShim` 未定义、ReActAgent/MinimalAgent 无 conversation 注入)。

- [ ] **Step 3: 写最小实现**

`agent-core/src/main/kotlin/com/you/agent/core/Agent.kt`(整体替换,仅 run 签名加 sessionId):
```kotlin
package com.you.agent.core

import kotlinx.coroutines.flow.Flow

sealed interface AgentEvent {
    data class Thought(val text: String) : AgentEvent
    data class ToolCall(val name: String, val args: Map<String, Any?>) : AgentEvent
    data class ToolResult(val name: String, val output: String) : AgentEvent
    data class Answer(val text: String) : AgentEvent
    data class Error(val cause: Throwable) : AgentEvent
}

data class AgentConfig(
    var maxSteps: Int = 8,
    var temperature: Double? = null,
    var maxTokens: Int? = null,
)

interface Agent {
    /**
     * 运行一轮对话。
     * @param userInput 用户输入
     * @param personaId 角色 ID(M5 按 ID 加载;M3 仍用 Builder 已设 persona)
     * @param sessionId 会话 ID;非空 + Builder 注入了 ConversationStore 时启用多轮历史。null 表示无状态单轮。
     */
    fun run(userInput: String, personaId: String? = null, sessionId: String? = null): Flow<AgentEvent>

    companion object
}
```

`agent-core/src/main/kotlin/com/you/agent/core/AgentBuilder.kt`(整体替换):
```kotlin
package com.you.agent.core

import com.you.agent.core.conversation.ConversationStore
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
    private var conversation: ConversationStore? = null
    private val config = AgentConfig()

    fun llm(provider: LlmProvider): AgentBuilder = apply { this.llm = provider }
    fun persona(persona: Persona): AgentBuilder = apply { this.persona = persona }
    fun memory(store: PersonaStore): AgentBuilder = apply { this.personaStore = store }
    fun rag(retriever: KnowledgeRetriever): AgentBuilder = apply { this.retriever = retriever }
    fun tools(block: ToolRegistryScope.() -> Unit): AgentBuilder =
        apply { this.tools = ToolRegistryScope().apply(block).build() }
    fun conversation(store: ConversationStore): AgentBuilder = apply { this.conversation = store }
    fun config(block: AgentConfig.() -> Unit): AgentBuilder = apply { config.block() }

    fun build(): Agent {
        val provider = llm
            ?: error("Agent requires an LlmProvider. 未配置 LLM 时绝不静默使用默认 Provider。")
        return if (tools.isNotEmpty()) {
            ReActAgent(llm = provider, persona = persona, tools = tools, config = config, conversation = conversation)
        } else {
            MinimalAgent(llm = provider, persona = persona, config = config, conversation = conversation)
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

fun Agent.Companion.builder(): AgentBuilder = AgentBuilder()
```

`agent-core/src/main/kotlin/com/you/agent/core/internal/MinimalAgent.kt`(整体替换):
```kotlin
package com.you.agent.core.internal

import com.you.agent.core.Agent
import com.you.agent.core.AgentConfig
import com.you.agent.core.AgentEvent
import com.you.agent.core.chat.ChatRequest
import com.you.agent.core.chat.Message
import com.you.agent.core.conversation.ConversationStore
import com.you.agent.core.llm.LlmProvider
import com.you.agent.core.persona.Persona
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.flow

/** M1 最小实现: 组装 persona system prompt + 历史(可选) + 用户输入, 调 LlmProvider 一次, 产出 Answer。 */
internal class MinimalAgent(
    private val llm: LlmProvider,
    private val persona: Persona,
    private val config: AgentConfig,
    private val conversation: ConversationStore? = null,
) : Agent {
    override fun run(userInput: String, personaId: String?, sessionId: String?): Flow<AgentEvent> = flow {
        val history = conversation?.let { store -> sessionId?.let { store.load(it) } } ?: emptyList()
        val messages = mutableListOf<Message>()
        messages.add(Message.system(persona.systemPrompt))
        messages.addAll(history)
        val historyEnd = messages.size
        messages.add(Message.user(userInput))
        try {
            val response = llm.chat(ChatRequest(
                model = llm.modelId,
                messages = messages.toList(),
                temperature = config.temperature,
                maxTokens = config.maxTokens,
            ))
            emit(AgentEvent.Answer(response.content))
            messages.add(Message.assistant(response.content))
            conversation?.let { store -> sessionId?.let { store.append(it, messages.drop(historyEnd)) } }
        } catch (t: Throwable) {
            emit(AgentEvent.Error(t))
        }
    }
}
```

`agent-core/src/main/kotlin/com/you/agent/core/internal/ReActAgent.kt`(整体替换):
```kotlin
package com.you.agent.core.internal

import com.you.agent.core.Agent
import com.you.agent.core.AgentConfig
import com.you.agent.core.AgentEvent
import com.you.agent.core.chat.ChatRequest
import com.you.agent.core.chat.Message
import com.you.agent.core.conversation.ConversationStore
import com.you.agent.core.llm.LlmProvider
import com.you.agent.core.persona.Persona
import com.you.agent.core.tools.MapToolRegistry
import com.you.agent.core.tools.Tool
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.flow

/**
 * M2/M3 ReAct 实现: 循环调用 LLM; 若返回 tool_calls 则执行工具、把结果作为 tool 消息喂回,
 * 直到 LLM 返回纯文本 Answer 或达 maxSteps。
 * M3: 可选注入 ConversationStore,按 sessionId 加载历史拼到 system 后、user 前;run 结束 append 本轮新增。
 */
internal class ReActAgent(
    private val llm: LlmProvider,
    private val persona: Persona,
    private val tools: List<Tool>,
    private val config: AgentConfig,
    private val conversation: ConversationStore? = null,
) : Agent {
    override fun run(userInput: String, personaId: String?, sessionId: String?): Flow<AgentEvent> = flow {
        val history = conversation?.let { store -> sessionId?.let { store.load(it) } } ?: emptyList()
        val registry = MapToolRegistry().also { reg -> tools.forEach { reg.register(it) } }
        val declarations = tools.map { it.toDeclaration() }
        val messages = mutableListOf<Message>()
        messages.add(Message.system(persona.systemPrompt))
        messages.addAll(history)
        val historyEnd = messages.size
        messages.add(Message.user(userInput))
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
                    messages.add(Message.assistant(resp.content))
                    conversation?.let { store -> sessionId?.let { store.append(it, messages.drop(historyEnd)) } }
                    return@flow
                }
                messages.add(Message(Message.Role.ASSISTANT, content = "", toolCalls = toolCalls))
                for (tc in toolCalls) {
                    emit(AgentEvent.ToolCall(tc.name, tc.args))
                    val tool = registry.get(tc.name)
                        ?: run {
                            emit(AgentEvent.Error(IllegalStateException("未注册的工具: ${tc.name}")))
                            conversation?.let { store -> sessionId?.let { store.append(it, messages.drop(historyEnd)) } }
                            return@flow
                        }
                    val result = tool.handler(tc.args)
                    emit(AgentEvent.ToolResult(tc.name, result))
                    messages.add(Message(Message.Role.TOOL, content = result, toolCallId = tc.id))
                }
            }
            emit(AgentEvent.Error(IllegalStateException("达到 maxSteps(${config.maxSteps}) 未收敛")))
            conversation?.let { store -> sessionId?.let { store.append(it, messages.drop(historyEnd)) } }
        } catch (t: Throwable) {
            emit(AgentEvent.Error(t))
        }
    }
}
```

测试用 Shim(agent-core test 内,简单实现 ConversationStore + sessionCount 便于断言):
`agent-core/src/test/kotlin/com/you/agent/core/conversation/InMemoryConversationStoreShim.kt`:
```kotlin
package com.you.agent.core.conversation

import com.you.agent.core.chat.Message

/** agent-core test 内的简单 InMemory 实现(避免 test 依赖 agent-memory 模块)。 */
class InMemoryConversationStoreShim : ConversationStore {
    private val sessions = mutableMapOf<String, MutableList<Message>>()
    override suspend fun load(sessionId: String): List<Message> = sessions[sessionId]?.toList() ?: emptyList()
    override suspend fun append(sessionId: String, messages: List<Message>) {
        sessions.getOrPut(sessionId) { mutableListOf() }.addAll(messages)
    }
    override suspend fun clear(sessionId: String) { sessions.remove(sessionId) }
    fun sessionCount(): Int = sessions.size
}
```

- [ ] **Step 4: 跑测试确认通过**

Run: `./gradlew :agent-core:test`
Expected: BUILD SUCCESSFUL,新增 4 个 ConversationAgentTest 通过,M2 既有 22 个测试不受影响(sessionId 默认 null,向后兼容),合计 26 个 `:agent-core` 测试。

**若 M2 既有测试失败**(例如 ReActAgentTest/EndToEndToolAgentTest 因 run 签名或 conversation 注入编译失败),停止并报告——不要弱化断言,不要改 M2 测试。检查是否签名变更破坏了二进制兼容(本计划 run 加的是带默认值的参数,Kotlin 调用 `agent.run("x")` 仍合法,不应破坏)。

- [ ] **Step 5: 提交**

```bash
git add agent-core/src
git commit -m "feat(agent-core): inject ConversationStore into agents, add sessionId to Agent.run"
```

---

## Task 4: 端到端多轮对话(MockWebServer + agent-memory 真实 store)

**Files:**
- Modify: `agent-android/agent-core/build.gradle.kts`(加 `testImplementation(project(":agent-memory"))`)
- Test: `agent-android/agent-core/src/test/kotlin/com/you/agent/core/EndToEndConversationTest.kt`

**Interfaces:**
- Consumes: `Agent.builder().conversation()`(Task 3)、`InMemoryConversationStore`(Task 2)、`OpenAiCompatibleProvider`(M1)、`PresetPersonaStore`(Task 1,可选)
- Produces: M3 验收——无真实网络下端到端多轮对话带历史闭环

- [ ] **Step 1: 加 test 依赖**

在 `agent-core/build.gradle.kts` 的 `dependencies { }` 内追加:
```kotlin
    testImplementation(project(":agent-memory"))
```
(M1 Task 9 已加 `:agent-llm` + mockwebserver,M2 Task 6 已加 `:agent-tools`。)

- [ ] **Step 2: 写测试**

`agent-core/src/test/kotlin/com/you/agent/core/EndToEndConversationTest.kt`:
```kotlin
package com.you.agent.core

import com.you.agent.core.persona.Persona
import com.you.agent.llm.LlmFactory
import com.you.agent.llm.UserLlmConfig
import com.you.agent.memory.InMemoryConversationStore
import com.you.agent.memory.PresetPersonaStore
import kotlinx.coroutines.flow.toList
import kotlinx.coroutines.test.runTest
import okhttp3.mockwebserver.MockResponse
import okhttp3.mockwebserver.MockWebServer
import org.junit.jupiter.api.Assertions.assertEquals
import org.junit.jupiter.api.Assertions.assertInstanceOf
import org.junit.jupiter.api.Assertions.assertTrue
import org.junit.jupiter.api.Test

class EndToEndConversationTest {
    @Test
    fun `multi-turn conversation carries history across runs`() = runTest {
        val server = MockWebServer()
        server.enqueue(MockResponse().setBody(
            """{"model":"deepseek-chat","choices":[{"message":{"role":"assistant","content":"你好,我是助手"}}]}"""))
        server.enqueue(MockResponse().setBody(
            """{"model":"deepseek-chat","choices":[{"message":{"role":"assistant","content":"你刚才说了你好"}}]}"""))
        server.start()
        try {
            val provider = LlmFactory.from(UserLlmConfig(
                providerId = "custom",
                apiKey = "key",
                baseUrl = server.url("/v1").toString().trimEnd('/'),
                model = "deepseek-chat",
            ))
            val persona = PresetPersonaStore().load("assistant_v1")
            val store = InMemoryConversationStore()
            val agent = Agent.builder()
                .llm(provider)
                .persona(persona)
                .conversation(store)
                .build()

            val e1 = agent.run("你好", sessionId = "s1").toList()
            val e2 = agent.run("我刚才说了什么", sessionId = "s1").toList()

            assertInstanceOf(AgentEvent.Answer::class.java, e1.last())
            assertEquals("你好,我是助手", (e1.last() as AgentEvent.Answer).text)
            assertInstanceOf(AgentEvent.Answer::class.java, e2.last())
            assertEquals("你刚才说了你好", (e2.last() as AgentEvent.Answer).text)

            // 第二轮请求体应含第一轮的历史(user+assistant)
            server.takeRequest() // 第一轮
            val secondBody = server.takeRequest().body.readUtf8()
            assertTrue(secondBody.contains("\"role\":\"user\""))
            assertTrue(secondBody.contains("你好"))
            assertTrue(secondBody.contains("\"role\":\"assistant\""))
            assertTrue(secondBody.contains("你好,我是助手"))
            assertTrue(secondBody.contains("我刚才说了什么"))
        } finally { server.shutdown() }
    }
}
```

- [ ] **Step 3: 跑全部测试**

Run: `./gradlew test`
Expected: BUILD SUCCESSFUL,四模块测试全过(`:agent-core` 27 / `:agent-llm` 17 / `:agent-tools` 6 / `:agent-memory` 10),合计 60 个。

- [ ] **Step 4: 提交**

```bash
git add agent-core/build.gradle.kts agent-core/src
git commit -m "test: end-to-end multi-turn conversation with history via MockWebServer"
```

---

## Self-Review

**1. Spec coverage(对照设计文档 M3 范围):**
- `agent-memory` 预设角色 JSON 只读加载 → Task 1 ✓(用 classpath resources 替代 assets,理由记录于 Architecture)
- `PersonaStore` 默认实现 → Task 1 `PresetPersonaStore` ✓
- 会话记忆(设计文档 §6 写 ADK InMemorySessionService)→ Task 2/3 `ConversationStore` + `InMemoryConversationStore` ✓(手写替代 ADK,理由同 M2)
- Phase 1 InMemory → Task 2 ✓;持久化留 M5 ✓
- 接口不动,仅替换实现 → `PersonaStore`/`ConversationStore` 接口稳定,M5 加 `WritablePersonaStore`/持久化 ConversationStore 不改接口 ✓
- 未覆盖(留给后续里程碑,符合范围):角色可写 M5 / RAG M4 / 滑窗摘要 M5 / 持久化 M5 / personaId 按 ID 加载 M5 / sample M7

**2. 关键决策说明(对设计文档的有意偏离):**
- 设计文档 §6 写预设角色放 `assets/personas/*.json` + ADK `InMemorySessionService`。经 M2 确认不引 ADK,M3 改为 classpath `resources/personas/*.json`(纯 JVM 可测,Android 打包自动进 classpath)+ 手写 `ConversationStore`。功能等价,可测性更好。`PersonaStore`/`Agent`/`LlmProvider` 抽象层不变。

**3. Placeholder scan:** 每步含完整代码或确切命令,无 placeholder。Task 3 测试用 `InMemoryConversationStoreShim`(agent-core test 内)而非 agent-memory 的实现,是为避免 agent-core test 依赖 agent-memory(模块依赖方向:agent-memory → agent-core,反向会循环);Task 4 端到端用真实 `InMemoryConversationStore`(agent-core test 已加 `:agent-memory` test 依赖)。✓

**4. Type consistency:**
- `ConversationStore.load/append/clear` Task 2 定义,Task 3 ReActAgent/MinimalAgent + Shim + Task 4 一致 ✓
- `Message(role, content, toolCalls?, toolCallId?)` M2 定义,Task 2/3/4 一致 ✓
- `Agent.run(userInput, personaId?, sessionId?)` Task 3 定义,Task 3/4 调用一致;sessionId 默认 null 向后兼容 M2 ✓
- `AgentBuilder.conversation(store)` Task 3 定义,Task 3/4 调用一致 ✓
- `PresetPersonaStore.load/list/save` Task 1 实现 `PersonaStore`(M1 定义),Task 4 调用 `load` 一致 ✓
- `Persona(id, name, systemPrompt, traits, knowledgeSources, tools)` M1 定义,Task 1 PersonaJson.toPersona() 字段对齐 ✓
- 历史注入位置:`[system] + history + [user] + 后续`,Task 3 ReActAgent/MinimalAgent 一致;append 用 `messages.drop(historyEnd)` 排除 system 与已 load 的 history ✓

**5. 向后兼容:**
- `Agent.run()` 加 `sessionId: String? = null`,M2 全部测试(`agent.run("x")` / `agent.run("x").toList()`)零改动通过 ✓
- `AgentBuilder` 加 `conversation(store)` 是新方法,不影响既有 builder 链 ✓
- `MinimalAgent`/`ReActAgent` 构造加 `conversation: ConversationStore? = null` 带默认值,AgentBuilder.build() 传 `conversation`(可能 null),M2 路径(tools 非空 → ReActAgent / tools 空 → MinimalAgent)行为不变 ✓
- M2 ReActAgentTest 的 `secondReq.messages.last().role == TOOL` 断言:无 conversation 时 history=空,messages=[system, user, assistant(toolCalls), tool],last 仍是 TOOL ✓
- M2 EndToEndToolAgentTest:无 sessionId,行为同 M2 ✓

---

## Execution Handoff

计划已保存至 `docs/superpowers/plans/2026-07-02-android-agent-m3.md`。

**无需用户再次确认架构偏离**——`assets/`→`resources/` 与 ADK Session→手写 ConversationStore 的收窄均沿用 M2 已确认的"手写、不引 ADK、纯 JVM 可测"既定方向,理由记录于 Architecture 节。

沿用 M1/M2 的 **Subagent-Driven Development** 执行:每个 Task 派发独立 subagent,TDD 严格流程(失败测试→实现→通过→提交),任务间两阶段评审。关键环境约束(JDK 17 已写入 `~/.gradle/gradle.properties` 的 `org.gradle.java.home`、`./gradlew`、沙箱代理、不弱化断言)与 M2 一致。

Task 3 是关键改动点(改 M2 稳定的 ReActAgent/MinimalAgent + Agent.run 签名),派发时须特别叮嘱:若 M2 既有测试因签名变更编译失败,停止报告不弱化断言(预期不破坏,因 sessionId 带默认值)。
