# Android Agent 组件 — M5 实现计划(agent-memory Phase 2 自定义角色 + 持久化)

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 在 M3 预设角色 + 内存会话、M4 RAG 检索之上,实现 agent-memory 的持久化层:(1) `WritablePersonaStore` —— 可写 PersonaStore,把自定义角色持久化到文件系统;(2) `FileConversationStore` —— 持久化 ConversationStore,把会话历史存到文件;(3) `CompositePersonaStore` —— 组合预设(只读)+ 自定义(可写)的统一入口。全程纯 JVM 可单测,用 `java.io.File` + `kotlinx.serialization` + JUnit 5 `@TempDir`,不引 Android DataStore/Room(纯 JVM 可测原则,Android 特定持久化留宿主或 M7 sample)。

**Architecture:** 沿用 M1-M4 手写、纯 JVM 可测路线。`PersonaStore`(M1 接口,M3 仅 PresetPersonaStore 只读)+ `ConversationStore`(M3 接口,仅 InMemoryConversationStore)已在 agent-core/agent-memory 落地。M5 在 agent-memory 新增三个实现,均基于文件 JSON 持久化:
- `WritablePersonaStore(baseDir)`:角色存 `<baseDir>/personas/<id>.json`,load/list/save 全实现。
- `FileConversationStore(baseDir)`:会话存 `<baseDir>/sessions/<sessionId>.json`,每会话一文件,append=load+追加+写回。
- `CompositePersonaStore(primary, fallback)`:load 先 primary 后 fallback;list 合并;save 只入 primary。

`Message` 序列化难点:`ToolCall.args: Map<String, Any?>` 的 `Any?` 不可直接 kotlinx.serialization。M5 在 agent-memory 定义 `MessageJson` DTO,`args: Map<String, JsonElement>`,配 `toJsonElement`/`toAny` 双向转换(参考 M2 `OpenAiCompatibleProvider` 的同类逻辑,但独立实现于 agent-memory)。

**关键决策(基于 M1-M4 既定方向收敛):**
1. **持久化技术选型:文件 JSON,不引 Android DataStore/Room**。设计文档 §6 写"基于 DataStore/Room",`ConversationStore` KDoc 写"M5 提供持久化实现(Room/DataStore)"。M5 偏离为 `java.io.File` + `kotlinx.serialization`,理由同 M2-M4 一贯原则:纯 JVM 可单测、不依赖 Android SDK/Context、组件作为 Gradle 库在宿主外可验证。Android 特定持久化(DataStore/Room/sqlite-vss)由宿主按需自行实现 `PersonaStore`/`ConversationStore` 接口,M5 提供文件实现作为开箱即用的纯 JVM 选项。**这是对设计文档的有意偏离,沿用 M3 `assets→resources`、`ADK Session→手写 ConversationStore` 的同一原则。**
2. **Message 序列化方案**:`MessageJson(role, content, toolCalls?, toolCallId?)` + `ToolCallJson(id, name, args: Map<String, JsonElement>)`。`Any? → JsonElement` 按 null/Boolean/Number/String/else(toString)分发;`JsonElement → Any?` 按 JsonNull/primitive(Int/Double/Boolean/String 还原)/else(toString)。完整保留 ReAct 多轮的 tool_calls/tool 消息(append 时 `messages.drop(historyEnd)` 含这些,见 M3 ReActAgent),多轮工具对话恢复后 LLM 能看到之前的工具结果。
3. **CompositePersonaStore 语义**:`primary`(自定义,WritablePersonaStore)+ `fallback`(预设,PresetPersonaStore)。`load(id)`:primary hit → 返回;primary miss → fallback;两者 miss → 抛(`预设角色不存在` 风格)。`list()`:primary.list() + fallback.list(),**同 id 去重(primary 优先)**,允许自定义角色覆盖同名预设。`save(persona)`:**只存 primary**(预设只读,save 到 fallback 会抛 UnsupportedOp,故 Composite 必须直 primary)。这给宿主"预设打底 + 用户自定义扩展"的常见形态。
4. **WritablePersonaStore 不加 delete**:PersonaStore 接口无 delete(YAGNI),M5 不扩展接口。若需删除自定义角色,宿主直接删文件或后续里程碑加(M5 不做)。`clear()` 也不在接口。
5. **FileConversationStore append 语义**:load 现有 + 追加新 + 整体写回(简单原子性:写临时文件 → rename,防写一半崩溃)。单测足够,生产并发场景(多线程同 sessionId)留宿主加锁(M5 注释说明,不实现)。
6. **范围控制(YAGNI)**:M5 只做角色 + 会话的文件持久化。**不做**:persona.knowledgeSources 过滤检索(M4 留,偏离主题)、向量库持久化(M4 InMemoryEmbeddingStore 持久化留 M6/M7)、reconfigure(M6)、Android DataStore/Room 适配(留宿主)、delete/clear 自定义角色(接口无)。

**Tech Stack:** Kotlin 2.1.20、kotlinx.serialization(已有 agent-memory 依赖)、java.io.File、JUnit 5 `@TempDir`、kotlinx.coroutines.test。

**Context — M4 已落地(本计划基线):**
- `agent-core`:`PersonaStore`(M1,load/list/save)、`ConversationStore`(M3,load/append/clear)、`Persona`(M1)、`Message`(M1,含 role/content/toolCalls/toolCallId)、`ToolCall`(M1,含 args: Map<String, Any?>)、`Agent.builder()`(M3,已传 conversation/retriever)。
- `agent-memory`:`PresetPersonaStore`(M3,只读 classpath `personas/*.json`)、`InMemoryConversationStore`(M3,Map)、`PersonaJson`(M3,序列化 DTO,有 `toPersona()` 正向,无反向)。
- 测试:`:agent-core` 32 / `:agent-llm` 17 / `:agent-tools` 6 / `:agent-memory` 10 / `:agent-rag` 6 = **71 个全绿**。
- `settings.gradle.kts`:`include(":agent-core", ":agent-llm", ":agent-tools", ":agent-memory", ":agent-rag")`。
- agent-memory build.gradle.kts:已有 `kotlin.serialization` 插件 + `kotlinx-serialization-json` 依赖,无需新增。

## Global Constraints

- 纯 Kotlin/JVM,M5 不依赖 Android SDK、不依赖 ADK、不依赖 google-genai、不依赖 Room/DataStore。
- 持久化用 `java.io.File` + `kotlinx.serialization`(agent-memory 已有依赖,不新增)。
- 所有公共 API 用 Kotlin + Coroutines/Flow(Store 接口的 suspend 方法),无 UI 耦合。
- `Message` 序列化完整保留 toolCalls/toolCallId(不丢工具调用上下文)。
- CompositePersonaStore.save 只入 primary(自定义),不污染预设。
- 测试用 JUnit 5 `@TempDir`(或 `kotlin.io.path.createTempDirectory`),不污染工作区。
- 包名前缀统一 `com.you.agent`。
- TDD:先写失败测试,再写最小实现,再跑通,再提交。**不弱化断言**。
- 每个任务结束提交一次,提交信息以 `feat:`/`test:` 前缀。
- DRY、YAGNI:不实现 M5 范围外的能力(向量库持久化 M6/M7、reconfigure M6、Android 适配留宿主、delete 角色 YAGNI)。

---

## File Structure(本计划新增/改动)

```
agent-android/
└── agent-memory/
    └── src/
        ├── main/kotlin/com/you/agent/memory/
        │   ├── MessageJson.kt               # 新增:Message 序列化 DTO + 双向转换
        │   ├── WritablePersonaStore.kt      # 新增:文件 JSON 持久化角色(可写)
        │   ├── FileConversationStore.kt     # 新增:文件 JSON 持久化会话
        │   ├── CompositePersonaStore.kt     # 新增:组合预设+自定义统一入口
        │   ├── PersonaJson.kt               # 既有(微调:加 toPersonaJson() 反向,供 WritablePersonaStore.save 用)
        │   ├── PresetPersonaStore.kt        # 既有(不改)
        │   └── InMemoryConversationStore.kt # 既有(不改)
        └── test/kotlin/com/you/agent/memory/
            ├── WritablePersonaStoreTest.kt  # 新增
            ├── FileConversationStoreTest.kt # 新增
            ├── CompositePersonaStoreTest.kt # 新增
            └── 既有测试(不改)
```

agent-core 本计划**不改源码**(接口 M1/M3 已定),仅 Task 4 加端到端测试(放 agent-core test,已有 `testImplementation(project(":agent-memory"))` 依赖)。

---

## Task 1: WritablePersonaStore(文件 JSON 持久化角色)

**Files:**
- Modify: `agent-android/agent-memory/src/main/kotlin/com/you/agent/memory/PersonaJson.kt`(加 `Persona.toPersonaJson()` 反向函数)
- Create: `agent-android/agent-memory/src/main/kotlin/com/you/agent/memory/WritablePersonaStore.kt`
- Test: `agent-android/agent-memory/src/test/kotlin/com/you/agent/memory/WritablePersonaStoreTest.kt`

**Interfaces:**
- Consumes: `PersonaStore`/`Persona`(M1,agent-core)、`PersonaJson`(M3)
- Produces: `WritablePersonaStore`(供 Task 3 CompositePersonaStore 作 primary,Task 4 端到端)

- [ ] **Step 1: 加 Persona.toPersonaJson() 反向函数**

修改 `PersonaJson.kt`,在文件末尾加(既有 `PersonaJson.toPersona()` 保留):
```kotlin
internal fun Persona.toPersonaJson(): PersonaJson = PersonaJson(
    id = id,
    name = name,
    systemPrompt = systemPrompt,
    traits = traits,
    knowledgeSources = knowledgeSources,
    tools = tools,
)
```
(注:`PersonaJson` 的 `@SerialName("systemPrompt")` 等已在 M3 定义,序列化输出与预设 JSON 格式一致,可互读。)

- [ ] **Step 2: 写失败测试**

`agent-memory/src/test/kotlin/com/you/agent/memory/WritablePersonaStoreTest.kt`:
```kotlin
package com.you.agent.memory

import com.you.agent.core.persona.Persona
import kotlinx.coroutines.test.runTest
import org.junit.jupiter.api.Assertions.assertEquals
import org.junit.jupiter.api.Assertions.assertFalse
import org.junit.jupiter.api.Assertions.assertThrows
import org.junit.jupiter.api.Assertions.assertTrue
import org.junit.jupiter.api.Test
import org.junit.jupiter.api.io.TempDir
import java.nio.file.Path

class WritablePersonaStoreTest {
    @TempDir
    lateinit var baseDir: Path

    private fun persona(id: String, name: String = id) = Persona(
        id = id, name = name, systemPrompt = "prompt for $id",
        traits = mapOf("style" to "concise"),
        knowledgeSources = listOf("kb_$id"), tools = listOf("t_$id"),
    )

    @Test
    fun `save then load roundtrip`() = runTest {
        val store = WritablePersonaStore(baseDir)
        val p = persona("custom_1", "我的助手")
        store.save(p)

        val loaded = store.load("custom_1")
        assertEquals(p, loaded)
    }

    @Test
    fun `load unknown id throws`() = runTest {
        val store = WritablePersonaStore(baseDir)
        val ex = assertThrows(IllegalStateException::class.java) {
            kotlinx.coroutines.runBlocking { store.load("not_exist") }
        }
        assertTrue(ex.message!!.contains("not_exist"))
    }

    @Test
    fun `list returns saved personas`() = runTest {
        val store = WritablePersonaStore(baseDir)
        store.save(persona("a"))
        store.save(persona("b"))

        val ids = store.list().map { it.id }
        assertTrue(ids.contains("a"))
        assertTrue(ids.contains("b"))
        assertEquals(2, store.list().size)
    }

    @Test
    fun `save overwrites existing persona`() = runTest {
        val store = WritablePersonaStore(baseDir)
        store.save(persona("p", "v1"))
        store.save(persona("p", "v2"))

        val loaded = store.load("p")
        assertEquals("v2", loaded.name)
        assertEquals(1, store.list().size)
    }

    @Test
    fun `persists across instances`() = runTest {
        val store1 = WritablePersonaStore(baseDir)
        store1.save(persona("durable", "持久角色"))

        // 新实例同目录应能看到已保存角色
        val store2 = WritablePersonaStore(baseDir)
        val loaded = store2.load("durable")
        assertEquals("持久角色", loaded.name)
        assertTrue(store2.list().map { it.id }.contains("durable"))
    }

    @Test
    fun `empty store list returns empty`() {
        val store = WritablePersonaStore(baseDir)
        assertTrue(store.list().isEmpty())
    }
}
```

- [ ] **Step 3: 跑测试确认失败**

Run: `gradle :agent-memory:test`(cwd=/workspace/agent-android)
Expected: FAIL,编译错误(`WritablePersonaStore` 未定义)。

- [ ] **Step 4: 写最小实现**

`agent-memory/src/main/kotlin/com/you/agent/memory/WritablePersonaStore.kt`:
```kotlin
package com.you.agent.memory

import com.you.agent.core.persona.Persona
import com.you.agent.core.persona.PersonaStore
import kotlinx.serialization.json.Json
import java.nio.file.Files
import java.nio.file.Path

/**
 * 可写 PersonaStore:把自定义角色持久化到文件系统 `<baseDir>/personas/<id>.json`。
 * save 覆盖同名;load 不存在抛 IllegalStateException;list 列目录所有 .json。
 * 非线程安全(单测足够,生产并发留宿主加锁)。
 *
 * @param baseDir 持久化根目录(宿主提供,如 app filesDir/personas)
 */
class WritablePersonaStore(
    private val baseDir: Path,
) : PersonaStore {
    private val json = Json { prettyPrint = true }
    private val dir: Path = baseDir.resolve("personas")

    override suspend fun load(id: String): Persona {
        val file = dir.resolve("$id.json")
        if (!Files.exists(file)) error("自定义角色不存在: $id, 可用: ${listIds()}")
        val text = Files.readString(file)
        return json.decodeFromString(PersonaJson.serializer(), text).toPersona()
    }

    override fun list(): List<Persona> {
        if (!Files.exists(dir)) return emptyList()
        return Files.list(dir).use { stream ->
            stream
                .filter { it.toString().endsWith(".json") }
                .map { parseFile(it) }
                .toList()
        }
    }

    override suspend fun save(persona: Persona) {
        Files.createDirectories(dir)
        val file = dir.resolve("${persona.id}.json")
        val text = json.encodeToString(PersonaJson.serializer(), persona.toPersonaJson())
        Files.writeString(file, text)
    }

    private fun parseFile(file: Path): Persona {
        val text = Files.readString(file)
        return json.decodeFromString(PersonaJson.serializer(), text).toPersona()
    }

    private fun listIds(): List<String> = list().map { it.id }
}
```

注:`Files.list(dir).use { ... }` 需关闭 Stream(`use` 扩展需 `kotlin.io.use` 或 java AutoCloseable.use,标准库已有)。`.map { parseFile(it) }` 在 Stream 上是 Java Stream 的 map,返回 Stream<Path>,需 `.toList()`(Java 16+)。若 JDK 17 无 Stream.toList(),改用 `.collect(Collectors.toList())`。**subagent 按编译错误调整,断言不变**。

- [ ] **Step 5: 跑测试确认通过**

Run: `gradle :agent-memory:test`(cwd=/workspace/agent-android)
Expected: BUILD SUCCESSFUL,新增 6 个 WritablePersonaStoreTest 通过,M3 既有 10 个测试不受影响,合计 16 个 `:agent-memory` 测试。

- [ ] **Step 6: 提交**

```bash
cd /workspace/agent-android && git add agent-memory/src && git commit -m "feat(agent-memory): add WritablePersonaStore with file-based JSON persistence"
```

---

## Task 2: FileConversationStore + MessageJson(文件 JSON 持久化会话)

**Files:**
- Create: `agent-android/agent-memory/src/main/kotlin/com/you/agent/memory/MessageJson.kt`
- Create: `agent-android/agent-memory/src/main/kotlin/com/you/agent/memory/FileConversationStore.kt`
- Test: `agent-android/agent-memory/src/test/kotlin/com/you/agent/memory/FileConversationStoreTest.kt`

**Interfaces:**
- Consumes: `ConversationStore`/`Message`/`ToolCall`(M1/M3,agent-core)
- Produces: `FileConversationStore`(供 Task 4 端到端)

- [ ] **Step 1: 写 MessageJson 序列化 DTO + 双向转换**

`agent-memory/src/main/kotlin/com/you/agent/memory/MessageJson.kt`:
```kotlin
package com.you.agent.memory

import com.you.agent.core.chat.Message
import com.you.agent.core.chat.ToolCall
import kotlinx.serialization.SerialName
import kotlinx.serialization.Serializable
import kotlinx.serialization.json.JsonElement
import kotlinx.serialization.json.JsonNull
import kotlinx.serialization.json.JsonPrimitive

/** Message 的 JSON 序列化形式(完整保留 toolCalls/toolCallId,支持 ReAct 多轮恢复)。 */
@Serializable
data class MessageJson(
    val role: String,
    val content: String,
    @SerialName("tool_calls") val toolCalls: List<ToolCallJson>? = null,
    @SerialName("tool_call_id") val toolCallId: String? = null,
)

@Serializable
data class ToolCallJson(
    val id: String,
    val name: String,
    val args: Map<String, JsonElement> = emptyMap(),
)

internal fun Message.toJson(): MessageJson = MessageJson(
    role = role.name.lowercase(),
    content = content,
    toolCalls = toolCalls?.map { it.toJson() },
    toolCallId = toolCallId,
)

internal fun MessageJson.toMessage(): Message = Message(
    role = Message.Role.valueOf(role.uppercase()),
    content = content,
    toolCalls = toolCalls?.map { it.toToolCall() },
    toolCallId = toolCallId,
)

private fun ToolCall.toJson(): ToolCallJson = ToolCallJson(
    id = id,
    name = name,
    args = args.mapValues { it.value.toJsonElement() },
)

private fun ToolCallJson.toToolCall(): ToolCall = ToolCall(
    id = id,
    name = name,
    args = args.mapValues { it.value.toAny() },
)

/** Any? → JsonElement(null/Boolean/Number/String/else→toString)。 */
private fun Any?.toJsonElement(): JsonElement = when (this) {
    null -> JsonNull
    is Boolean -> JsonPrimitive(this)
    is Number -> JsonPrimitive(this)
    is String -> JsonPrimitive(this)
    else -> JsonPrimitive(this.toString())
}

/** JsonElement → Any?(JsonNull→null;primitive→Int/Double/Boolean/String;else→toString)。 */
private fun JsonElement.toAny(): Any? = when (this) {
    is JsonNull -> null
    is JsonPrimitive -> {
        val c = content
        c.toIntOrNull() ?: c.toDoubleOrNull() ?: c.toBooleanStrictOrNull() ?: c
    }
    else -> toString()
}
```

注:`toBooleanStrictOrNull` 是 Kotlin 1.5+ 扩展,2.1.20 可用。`Message.Role.valueOf(role.uppercase())`:role 序列化时是 `system`/`user`/`assistant`/`tool`,uppercase 后匹配枚举名。**测试需覆盖含 toolCalls 的消息往返**。

- [ ] **Step 2: 写失败测试**

`agent-memory/src/test/kotlin/com/you/agent/memory/FileConversationStoreTest.kt`:
```kotlin
package com.you.agent.memory

import com.you.agent.core.chat.Message
import com.you.agent.core.chat.ToolCall
import kotlinx.coroutines.test.runTest
import org.junit.jupiter.api.Assertions.assertEquals
import org.junit.jupiter.api.Assertions.assertTrue
import org.junit.jupiter.api.Test
import org.junit.jupiter.api.io.TempDir
import java.nio.file.Path

class FileConversationStoreTest {
    @TempDir
    lateinit var baseDir: Path

    @Test
    fun `append then load roundtrip`() = runTest {
        val store = FileConversationStore(baseDir)
        store.append("s1", listOf(
            Message.user("hello"),
            Message.assistant("hi there"),
        ))

        val loaded = store.load("s1")
        assertEquals(2, loaded.size)
        assertEquals(Message.Role.USER, loaded[0].role)
        assertEquals("hello", loaded[0].content)
        assertEquals(Message.Role.ASSISTANT, loaded[1].role)
        assertEquals("hi there", loaded[1].content)
    }

    @Test
    fun `append preserves toolCalls with typed args`() = runTest {
        val store = FileConversationStore(baseDir)
        val toolCall = ToolCall(id = "c1", name = "search", args = mapOf(
            "query" to "Beijing",
            "limit" to 5,
            "verbose" to true,
        ))
        store.append("s2", listOf(
            Message(Message.Role.ASSISTANT, content = "", toolCalls = listOf(toolCall)),
            Message(Message.Role.TOOL, content = "result", toolCallId = "c1"),
        ))

        val loaded = store.load("s2")
        assertEquals(2, loaded.size)
        val loadedCall = loaded[0].toolCalls!!.first()
        assertEquals("c1", loadedCall.id)
        assertEquals("search", loadedCall.name)
        assertEquals("Beijing", loadedCall.args["query"])
        assertEquals(5, loadedCall.args["limit"])
        assertEquals(true, loadedCall.args["verbose"])
        assertEquals("c1", loaded[1].toolCallId)
        assertEquals("result", loaded[1].content)
    }

    @Test
    fun `load unknown session returns empty`() = runTest {
        val store = FileConversationStore(baseDir)
        assertTrue(store.load("nope").isEmpty())
    }

    @Test
    fun `append accumulates across calls`() = runTest {
        val store = FileConversationStore(baseDir)
        store.append("s3", listOf(Message.user("first")))
        store.append("s3", listOf(Message.assistant("second")))

        val loaded = store.load("s3")
        assertEquals(2, loaded.size)
        assertEquals("first", loaded[0].content)
        assertEquals("second", loaded[1].content)
    }

    @Test
    fun `clear removes session`() = runTest {
        val store = FileConversationStore(baseDir)
        store.append("s4", listOf(Message.user("x")))
        store.clear("s4")

        assertTrue(store.load("s4").isEmpty())
    }

    @Test
    fun `persists across instances`() = runTest {
        val store1 = FileConversationStore(baseDir)
        store1.append("durable", listOf(Message.user("persistent msg")))

        val store2 = FileConversationStore(baseDir)
        val loaded = store2.load("durable")
        assertEquals(1, loaded.size)
        assertEquals("persistent msg", loaded[0].content)
    }
}
```

- [ ] **Step 3: 跑测试确认失败**

Run: `gradle :agent-memory:test`(cwd=/workspace/agent-android)
Expected: FAIL,编译错误(`FileConversationStore` 未定义)。

- [ ] **Step 4: 写最小实现**

`agent-memory/src/main/kotlin/com/you/agent/memory/FileConversationStore.kt`:
```kotlin
package com.you.agent.memory

import com.you.agent.core.chat.Message
import com.you.agent.core.conversation.ConversationStore
import kotlinx.serialization.builtins.ListSerializer
import kotlinx.serialization.json.Json
import java.nio.file.Files
import java.nio.file.Path
import java.nio.file.StandardCopyOption

/**
 * 持久化 ConversationStore:每会话存 `<baseDir>/sessions/<sessionId>.json`。
 * append = load 现有 + 追加 + 原子写回(临时文件 → rename)。
 * 完整保留 toolCalls/toolCallId(支持 ReAct 多轮恢复)。
 * 非线程安全(单测足够,生产并发留宿主加锁)。
 *
 * @param baseDir 持久化根目录
 */
class FileConversationStore(
    private val baseDir: Path,
) : ConversationStore {
    private val json = Json { prettyPrint = true }
    private val serializer = ListSerializer(MessageJson.serializer())
    private val dir: Path = baseDir.resolve("sessions")

    override suspend fun load(sessionId: String): List<Message> {
        val file = dir.resolve("$sessionId.json")
        if (!Files.exists(file)) return emptyList()
        val text = Files.readString(file)
        val jsonList = json.decodeFromString(serializer, text)
        return jsonList.map { it.toMessage() }
    }

    override suspend fun append(sessionId: String, messages: List<Message>) {
        Files.createDirectories(dir)
        val existing = load(sessionId).toMutableList()
        existing.addAll(messages)
        val jsonList = existing.map { it.toJson() }
        val text = json.encodeToString(serializer, jsonList)
        // 原子写:先写临时文件再替换,防写一半崩溃
        val target = dir.resolve("$sessionId.json")
        val tmp = dir.resolve("$sessionId.json.tmp")
        Files.writeString(tmp, text)
        Files.move(tmp, target, StandardCopyOption.REPLACE_EXISTING, StandardCopyOption.ATOMIC_MOVE)
    }

    override suspend fun clear(sessionId: String) {
        val file = dir.resolve("$sessionId.json")
        Files.deleteIfExists(file)
    }
}
```

注:`StandardCopyOption.ATOMIC_MOVE` 在同文件系统内是原子操作。若跨文件系统可能抛 AtomicMoveNotSupportedException,但 `@TempDir` 与 dir 同盘,无此问题。`ListSerializer(MessageJson.serializer())` 序列化 `List<MessageJson>`。**断言基于 Message 抽象层(Step 2 测试),不直接断言 JSON 文本**。

- [ ] **Step 5: 跑测试确认通过**

Run: `gradle :agent-memory:test`(cwd=/workspace/agent-android)
Expected: BUILD SUCCESSFUL,Task 1 的 6 个 + Task 2 的 6 个 + M3 既有 10 个 = 22 个 `:agent-memory` 测试。

**若 `append preserves toolCalls with typed args` 中 `assertEquals(5, loadedCall.args["limit"])` 失败**(类型还原为 Double 而非 Int):检查 `toAny()` 的 `c.toIntOrNull() ?: c.toDoubleOrNull()` 顺序——`"5"` 应先匹配 `toIntOrNull()` 返回 Int 5。若 FakeEmbeddingModel 式的 Double 还原问题,断言改为 `assertEquals(5.0, (loadedCall.args["limit"] as Number).toDouble())`——**但优先确保 toAny 还原为 Int**(计划要求,因为原 args 存的是 Int 5)。`assertEquals(5, ...)` 期望 Int,若实际是 Double 5.0 会失败。**不弱化**:确保 toAny 对 `"5"` 返回 Int。

- [ ] **Step 6: 提交**

```bash
cd /workspace/agent-android && git add agent-memory/src && git commit -m "feat(agent-memory): add FileConversationStore with MessageJson serialization"
```

---

## Task 3: CompositePersonaStore(组合预设 + 自定义)

**Files:**
- Create: `agent-android/agent-memory/src/main/kotlin/com/you/agent/memory/CompositePersonaStore.kt`
- Test: `agent-android/agent-memory/src/test/kotlin/com/you/agent/memory/CompositePersonaStoreTest.kt`

**Interfaces:**
- Consumes: `PersonaStore`/`Persona`(M1)、`PresetPersonaStore`(M3)、`WritablePersonaStore`(Task 1)
- Produces: `CompositePersonaStore`(供 Task 4 端到端,宿主默认入口)

- [ ] **Step 1: 写失败测试**

`agent-memory/src/test/kotlin/com/you/agent/memory/CompositePersonaStoreTest.kt`:
```kotlin
package com.you.agent.memory

import com.you.agent.core.persona.Persona
import kotlinx.coroutines.runBlocking
import kotlinx.coroutines.test.runTest
import org.junit.jupiter.api.Assertions.assertEquals
import org.junit.jupiter.api.Assertions.assertThrows
import org.junit.jupiter.api.Assertions.assertTrue
import org.junit.jupiter.api.Test
import org.junit.jupiter.api.io.TempDir
import java.nio.file.Path

class CompositePersonaStoreTest {
    @TempDir
    lateinit var baseDir: Path

    private fun customPersona(id: String) = Persona(id, id, "custom prompt $id")

    @Test
    fun `load custom from primary`() = runTest {
        val primary = WritablePersonaStore(baseDir)
        primary.save(customPersona("my_role"))
        val composite = CompositePersonaStore(primary = primary, fallback = PresetPersonaStore())

        val loaded = composite.load("my_role")
        assertEquals("custom prompt my_role", loaded.systemPrompt)
    }

    @Test
    fun `load preset from fallback when primary miss`() = runTest {
        val primary = WritablePersonaStore(baseDir)
        val composite = CompositePersonaStore(primary = primary, fallback = PresetPersonaStore())

        // assistant_v1 是预设角色,M3 PresetPersonaStore 提供
        val loaded = composite.load("assistant_v1")
        assertEquals("assistant_v1", loaded.id)
        assertEquals("通用助手", loaded.name)
    }

    @Test
    fun `load unknown throws when both miss`() = runTest {
        val primary = WritablePersonaStore(baseDir)
        val composite = CompositePersonaStore(primary = primary, fallback = PresetPersonaStore())

        val ex = assertThrows(IllegalStateException::class.java) {
            runBlocking { composite.load("totally_unknown") }
        }
        assertTrue(ex.message!!.contains("totally_unknown"))
    }

    @Test
    fun `list combines primary and fallback`() = runTest {
        val primary = WritablePersonaStore(baseDir)
        primary.save(customPersona("custom_a"))
        primary.save(customPersona("custom_b"))
        val composite = CompositePersonaStore(primary = primary, fallback = PresetPersonaStore())

        val ids = composite.list().map { it.id }
        // 自定义 + 预设 3 个(assistant_v1/coder_v1/translator_v1)
        assertTrue(ids.contains("custom_a"))
        assertTrue(ids.contains("custom_b"))
        assertTrue(ids.contains("assistant_v1"))
        assertTrue(ids.contains("coder_v1"))
        assertEquals(5, composite.list().size)
    }

    @Test
    fun `save goes to primary not fallback`() = runTest {
        val primary = WritablePersonaStore(baseDir)
        val fallback = PresetPersonaStore()
        val composite = CompositePersonaStore(primary = primary, fallback = fallback)

        composite.save(customPersona("new_custom"))

        // primary 能读到
        assertEquals("new_custom", primary.load("new_custom").id)
        // fallback 仍是只读,不含 new_custom
        assertThrows(UnsupportedOperationException::class.java) {
            runBlocking { fallback.save(customPersona("should_fail")) }
        }
    }

    @Test
    fun `custom overrides preset by id`() = runTest {
        val primary = WritablePersonaStore(baseDir)
        // 用预设同 id 保存自定义版本
        primary.save(Persona(
            id = "assistant_v1",
            name = "我的覆盖版",
            systemPrompt = "overridden",
        ))
        val composite = CompositePersonaStore(primary = primary, fallback = PresetPersonaStore())

        val loaded = composite.load("assistant_v1")
        assertEquals("我的覆盖版", loaded.name)
        assertEquals("overridden", loaded.systemPrompt)
        // list 中 assistant_v1 不重复(去重,primary 优先)
        val count = composite.list().count { it.id == "assistant_v1" }
        assertEquals(1, count)
    }
}
```

- [ ] **Step 2: 跑测试确认失败**

Run: `gradle :agent-memory:test`(cwd=/workspace/agent-android)
Expected: FAIL,编译错误(`CompositePersonaStore` 未定义)。

- [ ] **Step 3: 写最小实现**

`agent-memory/src/main/kotlin/com/you/agent/memory/CompositePersonaStore.kt`:
```kotlin
package com.you.agent.memory

import com.you.agent.core.persona.Persona
import com.you.agent.core.persona.PersonaStore

/**
 * 组合 PersonaStore:primary(自定义,可写)+ fallback(预设,只读)。
 * load:先 primary 命中返回,miss 查 fallback,两者 miss 抛。
 * list:合并去重(同 id primary 优先)。
 * save:只入 primary(预设只读,不污染)。
 * 适用:预设打底 + 用户自定义扩展的常见宿主形态。
 */
class CompositePersonaStore(
    private val primary: PersonaStore,
    private val fallback: PersonaStore,
) : PersonaStore {
    override suspend fun load(id: String): Persona {
        // primary 命中?
        val fromPrimary = runCatching { primary.load(id) }.getOrNull()
        if (fromPrimary != null) return fromPrimary
        // fallback
        return fallback.load(id)
    }

    override fun list(): List<Persona> {
        val primaryList = primary.list()
        val primaryIds = primaryList.map { it.id }.toSet()
        val fallbackList = fallback.list().filter { it.id !in primaryIds }
        return primaryList + fallbackList
    }

    override suspend fun save(persona: Persona) {
        primary.save(persona)
    }
}
```

注:`runCatching { primary.load(id) }` 捕获 primary 的 "不存在" 异常(WritablePersonaStore.load 不存在抛 IllegalStateException),miss 时返回 null 走 fallback。若 primary 抛其他异常(如 IO 错误),也会被吞掉走 fallback——这是已知折中(简化实现,单测足够;生产若需区分异常类型可后续增强,M5 YAGNI)。**断言基于 Persona 抽象层**。

- [ ] **Step 4: 跑测试确认通过**

Run: `gradle :agent-memory:test`(cwd=/workspace/agent-android)
Expected: BUILD SUCCESSFUL,Task 1/2 的 12 个 + Task 3 的 6 个 + M3 既有 10 个 = 28 个 `:agent-memory` 测试。

**若 `custom overrides preset by id` 的去重断言失败**(`count == 2` 而非 1):检查 `list()` 的 `filter { it.id !in primaryIds }`——primary 已含 assistant_v1 时,fallback 的 assistant_v1 应被过滤掉。逻辑正确,应通过。

- [ ] **Step 5: 提交**

```bash
cd /workspace/agent-android && git add agent-memory/src && git commit -m "feat(agent-memory): add CompositePersonaStore combining preset and custom stores"
```

---

## Task 4: 端到端持久化集成(自定义角色 + 会话恢复)

**Files:**
- Test: `agent-android/agent-core/src/test/kotlin/com/you/agent/core/EndToEndPersistenceTest.kt`
(agent-core build.gradle.kts 已有 `testImplementation(project(":agent-memory"))`,无需改依赖)

**Interfaces:**
- Consumes: `WritablePersonaStore`/`FileConversationStore`/`CompositePersonaStore`(Task 1/2/3)、`Agent.builder()`(M3,conversation/rag/persona)、`OpenAiCompatibleProvider`/`LlmFactory`(M1,MockWebServer)
- Produces: M5 验收——自定义角色 + 持久化会话跨实例恢复

- [ ] **Step 1: 写测试**

`agent-core/src/test/kotlin/com/you/agent/core/EndToEndPersistenceTest.kt`:
```kotlin
package com.you.agent.core

import com.you.agent.core.chat.Message
import com.you.agent.core.persona.Persona
import com.you.agent.llm.LlmFactory
import com.you.agent.llm.UserLlmConfig
import com.you.agent.memory.CompositePersonaStore
import com.you.agent.memory.FileConversationStore
import com.you.agent.memory.WritablePersonaStore
import kotlinx.coroutines.flow.toList
import kotlinx.coroutines.test.runTest
import okhttp3.mockwebserver.MockResponse
import okhttp3.mockwebserver.MockWebServer
import org.junit.jupiter.api.Assertions.assertEquals
import org.junit.jupiter.api.Assertions.assertInstanceOf
import org.junit.jupiter.api.Assertions.assertTrue
import org.junit.jupiter.api.Test
import org.junit.jupiter.api.io.TempDir
import java.nio.file.Path

class EndToEndPersistenceTest {
    @TempDir
    lateinit var baseDir: Path

    @Test
    fun `custom persona persists and is usable by new agent instance`() = runTest {
        // 第一实例:保存自定义角色
        val personaStore = WritablePersonaStore(baseDir)
        personaStore.save(Persona(
            id = "my_geographer",
            name = "地理专家",
            systemPrompt = "You are a geography expert.",
        ))

        // 新实例同目录加载(模拟重启)
        val reopened = WritablePersonaStore(baseDir)
        val persona = reopened.load("my_geographer")

        val server = MockWebServer()
        server.enqueue(MockResponse().setBody(
            """{"model":"m","choices":[{"message":{"role":"assistant","content":"Paris"}}]}"""))
        server.start()
        try {
            val provider = LlmFactory.from(UserLlmConfig(
                providerId = "custom",
                apiKey = "k",
                baseUrl = server.url("/v1").toString().trimEnd('/'),
                model = "m",
            ))
            val agent = Agent.builder()
                .llm(provider)
                .persona(persona)
                .build()

            val events = agent.run("capital of France?").toList()
            assertInstanceOf(AgentEvent.Answer::class.java, events.last())
            assertEquals("Paris", (events.last() as AgentEvent.Answer).text)

            // 请求体含自定义角色的 system prompt
            val body = server.takeRequest().body.readUtf8()
            assertTrue(body.contains("You are a geography expert."))
        } finally { server.shutdown() }
    }

    @Test
    fun `conversation persists across agent instances via FileConversationStore`() = runTest {
        val conversation = FileConversationStore(baseDir)

        val server = MockWebServer()
        // 第一轮 + 第二轮响应
        server.enqueue(MockResponse().setBody(
            """{"model":"m","choices":[{"message":{"role":"assistant","content":"first reply"}}]}"""))
        server.enqueue(MockResponse().setBody(
            """{"model":"m","choices":[{"message":{"role":"assistant","content":"second reply"}}]}"""))
        server.start()
        try {
            val provider = LlmFactory.from(UserLlmConfig(
                providerId = "custom",
                apiKey = "k",
                baseUrl = server.url("/v1").toString().trimEnd('/'),
                model = "m",
            ))

            // 第一实例:对话一轮
            val agent1 = Agent.builder()
                .llm(provider)
                .persona(Persona("p", "P", "sys"))
                .conversation(conversation)
                .build()
            agent1.run("first question", sessionId = "sess_A").toList()

            // 第二实例(模拟重启):同 sessionId 继续对话,应能看到历史
            val agent2 = Agent.builder()
                .llm(provider)
                .persona(Persona("p", "P", "sys"))
                .conversation(conversation)
                .build()
            val events2 = agent2.run("second question", sessionId = "sess_A").toList()

            assertInstanceOf(AgentEvent.Answer::class.java, events2.last())
            assertEquals("second reply", (events2.last() as AgentEvent.Answer).text)

            // 第二实例请求体应含第一轮的历史(user: first question, assistant: first reply)
            val secondReq = server.takeRequest()
            // takeRequest 取的是第一个请求(第一轮),需再取一次拿第二轮
            server.takeRequest() // 消费第一轮请求
            val secondReqBody = server.takeRequest().body.readUtf8()
            assertTrue(secondReqBody.contains("first question"))
            assertTrue(secondReqBody.contains("first reply"))
        } finally { server.shutdown() }
    }

    @Test
    fun `composite store loads custom and preset in one agent`() = runTest {
        val primary = WritablePersonaStore(baseDir)
        primary.save(Persona("my_role", "我的角色", "custom sys"))
        val composite = CompositePersonaStore(primary = primary, fallback = com.you.agent.memory.PresetPersonaStore())

        // 自定义角色可用
        val custom = composite.load("my_role")
        assertEquals("custom sys", custom.systemPrompt)

        // 预设角色也可用
        val preset = composite.load("assistant_v1")
        assertEquals("assistant_v1", preset.id)

        // list 合并
        val ids = composite.list().map { it.id }
        assertTrue(ids.contains("my_role"))
        assertTrue(ids.contains("assistant_v1"))
    }
}
```

注:`conversation persists across agent instances` 测试的 `takeRequest` 顺序:MockWebServer 的 takeRequest 按请求到达顺序返回。第一轮 agent1.run 发 1 个请求,第二轮 agent2.run 发 1 个请求。代码里第一个 `server.takeRequest()` 拿到第一轮请求(消费),第二个 `server.takeRequest()` 拿到第二轮请求体验证历史。**subagent 执行时若 takeRequest 顺序错乱(如 MockWebServer 行为),调整调用顺序,但断言"第二轮请求体含第一轮历史"不变**。

- [ ] **Step 2: 跑测试确认通过**

Run: `gradle :agent-core:test`(cwd=/workspace/agent-android)
Expected: BUILD SUCCESSFUL,新增 3 个 EndToEndPersistenceTest 通过,M4 既有 32 个不受影响,合计 35 个 `:agent-core` 测试。

**若 `conversation persists` 的历史注入失败**(第二轮请求体不含 "first question"):检查 M3 ReActAgent/MinimalAgent 的 conversation 注入逻辑——`history = conversation.load(sessionId)` 拼到 system 后 user 前。FileConversationStore 的 append 在 agent1.run 收敛时被调用(`conversation.append(sessionId, messages.drop(historyEnd))`),写入文件。agent2 同 sessionId load 应读到。若失败,检查 FileConversationStore.append 是否正确写回(原子写 move 成功),或 agent1.run 是否调用了 append(收敛路径才 append,看 MinimalAgent:emit Answer 后 append)。

**若 takeRequest 顺序问题导致断言失败**:调整测试代码的 takeRequest 调用顺序(可能需先消费第一轮再读第二轮),但核心断言(第二轮请求含第一轮历史)不弱化。

- [ ] **Step 3: 跑全部测试**

Run: `gradle test`(cwd=/workspace/agent-android)
Expected: BUILD SUCCESSFUL,五模块全过(`:agent-core` 35 / `:agent-llm` 17 / `:agent-tools` 6 / `:agent-memory` 28 / `:agent-rag` 6),合计 **92 个**。

- [ ] **Step 4: 提交**

```bash
cd /workspace/agent-android && git add agent-core/src && git commit -m "test: end-to-end persistence with custom persona and cross-instance conversation recovery"
```

---

## Self-Review

**1. Spec coverage(对照设计文档 M5 范围):**
- `agent-memory` Phase 2 自定义角色 → Task 1 `WritablePersonaStore`(可写 save)✓
- 持久化 → Task 1(角色文件)+ Task 2(会话文件)✓
- `PersonaStore` 可写实现 → Task 1 ✓ + Task 3 Composite ✓
- 设计文档 §6 "Phase 2 开放 save(persona)" → Task 1/3 ✓
- 设计文档 §6 "持久化 SessionService (Phase 1.5)" → Task 2 `FileConversationStore` ✓
- CompositePersonaStore(预设打底+自定义扩展)→ Task 3 ✓(设计文档未明确,但符合"接口不动替换实现"原则)
- 未覆盖(留给后续,符合范围):Android DataStore/Room 适配(留宿主)、向量库持久化(M6/M7)、reconfigure(M6)、persona.knowledgeSources 过滤(后续)、delete 角色(YAGNI)

**2. 关键决策说明(对设计文档的有意偏离):**
- **文件 JSON 而非 DataStore/Room**:设计文档 §6 与 ConversationStore KDoc 提及 Room/DataStore。M5 改为 `java.io.File` + `kotlinx.serialization`,理由:纯 JVM 可测(M1-M4 一贯原则)、不依赖 Android SDK/Context、组件作为 Gradle 库在宿主外可验证。Android 特定持久化由宿主自行实现 Store 接口,M5 提供文件实现作纯 JVM 开箱选项。沿用 M3 `assets→resources`、`ADK Session→手写` 的同一原则。
- **MessageJson 完整序列化 toolCalls**:M5 不简化会话持久化(只存 content),而是完整保留 toolCalls/toolCallId,确保 ReAct 多轮工具对话恢复后 LLM 能看到之前的工具调用与结果。`args: Map<String, Any?>` 用 `Map<String, JsonElement>` + toAny/toJsonElement 双向转换处理(参考 M2 OpenAiCompatibleProvider 同类逻辑,独立实现于 agent-memory)。
- **CompositePersonaStore.save 只入 primary**:预设只读(M3 PresetPersonaStore.save 抛 UnsupportedOp),Composite 必须直 primary,否则 save 到 fallback 会抛。这给宿主"预设打底 + 自定义扩展"形态。

**3. Placeholder scan:** Task 1 Step 4 对 `Files.list(...).toList()`(Java 16+ Stream.toList)给了 fallback(`Collectors.toList()`)。Task 2 Step 4 对 `StandardCopyOption.ATOMIC_MOVE` 给了说明(同盘无问题)。Task 4 Step 2 对 `takeRequest` 顺序给了调整指导。每步含完整代码或确切命令。✓

**4. Type consistency:**
- `PersonaStore.load/list/save` M1 定义,Task 1/3 实现签名一致 ✓
- `ConversationStore.load/append/clear` M3 定义,Task 2 实现签名一致 ✓
- `Persona(id, name, systemPrompt, traits, knowledgeSources, tools)` M1,Task 1/3 测试用一致 ✓
- `Message(role, content, toolCalls?, toolCallId?)` + `ToolCall(id, name, args)` M1,Task 2 MessageJson 映射一致 ✓
- `Message.Role.{SYSTEM,USER,ASSISTANT,TOOL}` M1,MessageJson 用 `name.lowercase()` 序列化、`valueOf(uppercase())` 反序列化 ✓
- `PersonaJson` M3,Task 1 加 `Persona.toPersonaJson()` 反向,字段对齐 ✓

**5. 向后兼容:**
- M3 `PresetPersonaStore`/`InMemoryConversationStore` 不改,既有测试不受影响 ✓
- M3 `ConversationAgentTest`/`EndToEndConversationTest` 用 InMemoryConversationStore,行为不变 ✓
- M4 `RagAgentTest`/`EndToEndRagTest` 不涉及持久化,不受影响 ✓
- agent-core 源码零改动(Task 4 仅加测试),接口未动 ✓

**6. 测试覆盖:**
- WritablePersonaStore:6 个(save/load roundtrip、load unknown throws、list、overwrite、跨实例、empty list)✓
- FileConversationStore:6 个(append/load roundtrip、toolCalls 类型保留、unknown empty、accumulate、clear、跨实例)✓
- CompositePersonaStore:6 个(load custom、load preset fallback、unknown throws、list 合并、save 入 primary、custom override preset 去重)✓
- EndToEndPersistence:3 个(自定义角色跨实例、会话跨实例恢复、composite 混合)✓
- 共 21 个新增测试,71 → 92 ✓

---

## Execution Handoff

计划已保存至 `docs/superpowers/plans/2026-07-02-android-agent-m5.md`。

**无需用户再次确认架构偏离**——文件 JSON 而非 DataStore/Room、MessageJson 完整序列化均沿用 M1-M4 既定方向(手写、纯 JVM 可测、降低外部 API 耦合),理由记录于关键决策节。

沿用 M1-M4 的 **Subagent-Driven Development** 执行。关键环境约束:
- **用 `gradle` 不用 `./gradlew`**(wrapper 下载超时,系统 gradle 8.14 + JDK 17 daemon 已配 `~/.gradle/gradle.properties`)
- 不新增依赖(agent-memory 已有 `kotlinx-serialization-json`;JUnit 5 `@TempDir` 是 junit-jupiter 自带)
- 不弱化断言;Task 2 的 `assertEquals(5, args["limit"])` 须还原为 Int(优先确保 toAny 顺序),Task 4 的 takeRequest 顺序可调但核心断言不弱化
- Task 4 是验收关键(跨实例恢复),若会话历史注入失败,检查 FileConversationStore.append 写回 + agent 收敛时 append 调用,不弱化断言
- agent-core 源码零改动(M5 只在 agent-memory 加实现 + agent-core 加测试)
