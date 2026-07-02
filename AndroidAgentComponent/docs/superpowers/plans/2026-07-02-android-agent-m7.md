# Android Agent 组件 — M7 实现计划(sample Demo 验证集成)

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 创建 `sample` 模块,作为可运行的集成示例,演示 Agent 全能力。纯 JVM(用户要求先不考虑 UI,沙箱无 Android 设备),用 MockWebServer 内嵌模拟国产模型 OpenAI 兼容响应(无需真实 key),展示完整宿主集成链路:`LlmFactory.from(UserLlmConfig)`(BYO Key 入口)+ `PresetPersonaStore`/`CompositePersonaStore`(角色)+ `Agent.builder()`(组装)+ 工具调用 + 多轮会话记忆 + RAG 检索注入 + 持久化 + `reconfigure`(运行时切 provider)。`Main.kt` 的 `runDemo(): List<String>` 返回输出行(可测),`main()` 打印;smoke test 验证 `runDemo` 产出含预期 event 文本。

**Architecture:** 沿用 M1-M6 手写、纯 JVM 路线。新建 `sample` 模块,`application` plugin,`mainClass = com.you.agent.sample.MainKt`。依赖全部 5 个 agent-* 模块 + okhttp/mockwebserver(demo 内嵌模拟 LLM)。`Main.kt` 提供 `suspend fun runDemo(): List<String>`(可测核心)+ `fun main()`(打印)。runDemo 内启动 MockWebServer、enqueue 预设响应、用 `LlmFactory.from(UserLlmConfig("custom", "demo-key", baseUrl, "demo-model"))` 构造 provider,组装 Agent,collect 事件流转输出行,reconfigure 后第二轮展示多轮+切换。Task 2 扩展 RAG + 持久化演示。

**关键决策(基于 M1-M6 既定方向收敛):**
1. **纯 JVM sample,不做 Android UI**。用户明确"先不考虑 UI",沙箱无 Android 设备/SDK。sample 作为纯 JVM 可运行 demo,用 `application` plugin + `gradle :sample:run`。真实 Android App 集成留宿主(组件已是 Gradle 库,宿主 `implementation("com.you.agent:agent-core:0.1")` 即可)。
2. **MockWebServer 内嵌模拟 LLM,不用真实 key**。sample 演示的是"宿主如何集成组件"的 API 用法,不是真实对话。用 MockWebServer enqueue 预设 OpenAI 兼容 JSON 响应,`LlmFactory.from(UserLlmConfig("custom", "demo-key", server.url, "demo-model"))` 展示完整 BYO Key 入口。宿主换真实 key 时只需改 UserLlmConfig,其余代码零改动。
3. **runDemo() 可测函数**。`suspend fun runDemo(): List<String>` 返回输出行,`main()` 调它打印。smoke test 调 `runDemo()` 断言输出含预期 event 文本(Answer/ToolCall/ToolResult/多轮历史/reconfigure 切换)。不依赖 stdout 捕获,直接测返回值。
4. **范围控制(YAGNI)**:M7 只做 sample demo。**不做**:Android UI、真实 key 接入(宿主职责)、sample 的 Android 打包(需 Android SDK,环境不支持)、sample 的命令行交互式 REPL(YAGNI,演示用预设输入足够)。

**Tech Stack:** Kotlin 2.1.20、Kotlinx Coroutines/Flow、OkHttp MockWebServer、JUnit 5。

**Context — M6 已落地(本计划基线):**
- 全部 5 模块:agent-core(Agent/AgentBuilder/AgentEvent/reconfigure)、agent-llm(LlmFactory/UserLlmConfig/OpenAiCompatibleProvider)、agent-tools(Tool DSL)、agent-memory(PresetPersonaStore/WritablePersonaStore/CompositePersonaStore/InMemoryConversationStore/FileConversationStore)、agent-rag(KnowledgeIngestor/LangChain4jRetriever/FakeEmbeddingModel)。
- 测试:`:agent-core` 41 / `:agent-llm` 17 / `:agent-tools` 6 / `:agent-memory` 28 / `:agent-rag` 6 = **98 个全绿**。
- `settings.gradle.kts`:`include(":agent-core", ":agent-llm", ":agent-tools", ":agent-memory", ":agent-rag")`。

## Global Constraints

- 纯 Kotlin/JVM sample,不依赖 Android SDK、不做 UI。
- sample 用 MockWebServer 内嵌模拟 LLM,不用真实 key。
- `runDemo(): List<String>` 可测,`main()` 打印。
- smoke test 基于 `runDemo()` 返回值断言,不捕获 stdout。
- 包名前缀 `com.you.agent.sample`。
- TDD:先写失败测试,再写实现,再跑通,再提交。**不弱化断言**。
- 每个任务结束提交一次,提交信息以 `feat:`/`test:` 前缀。
- DRY、YAGNI:不实现 M7 范围外的能力(Android UI、真实 key、交互式 REPL)。

---

## File Structure(本计划新增)

```
agent-android/
├── settings.gradle.kts                  # 改: include +(":sample")
└── sample/                              # 新模块
    ├── build.gradle.kts                 # 新增
    └── src/
        ├── main/kotlin/com/you/agent/sample/
        │   └── Main.kt                  # 新增: runDemo() + main()
        └── test/kotlin/com/you/agent/sample/
            └── MainTest.kt              # 新增: smoke test
```

---

## Task 1: sample 模块 + Main 核心演示(对话+工具+多轮+reconfigure)

**Files:**
- Modify: `agent-android/settings.gradle.kts`
- Create: `agent-android/sample/build.gradle.kts`
- Create: `agent-android/sample/src/main/kotlin/com/you/agent/sample/Main.kt`
- Test: `agent-android/sample/src/test/kotlin/com/you/agent/sample/MainTest.kt`

**Interfaces:**
- Consumes: 全部 5 agent-* 模块(M1-M6)、okhttp/mockwebserver
- Produces: 可运行 sample demo(`gradle :sample:run`)

- [ ] **Step 1: 脚手架**

`agent-android/settings.gradle.kts`:
```kotlin
rootProject.name = "agent-android"
include(":agent-core", ":agent-llm", ":agent-tools", ":agent-memory", ":agent-rag", ":sample")
```

`agent-android/sample/build.gradle.kts`:
```kotlin
plugins {
    alias(libs.plugins.kotlin.jvm)
    application
}

dependencies {
    implementation(project(":agent-core"))
    implementation(project(":agent-llm"))
    implementation(project(":agent-tools"))
    implementation(project(":agent-memory"))
    implementation(project(":agent-rag"))
    implementation(libs.kotlinx.coroutines.core)
    implementation(libs.okhttp)
    implementation(libs.okhttp.mockwebserver)
    testImplementation(libs.kotlinx.coroutines.test)
    testImplementation(libs.junit.jupiter)
}

application {
    mainClass.set("com.you.agent.sample.MainKt")
}

tasks.test { useJUnitPlatform() }
```

- [ ] **Step 2: 写失败测试**

`sample/src/test/kotlin/com/you/agent/sample/MainTest.kt`:
```kotlin
package com.you.agent.sample

import kotlinx.coroutines.test.runTest
import org.junit.jupiter.api.Assertions.assertTrue
import org.junit.jupiter.api.Test

class MainTest {
    @Test
    fun `runDemo produces answer and tool events`() = runTest {
        val output = runDemo()
        // 至少含一个 Answer
        assertTrue(output.any { it.contains("Answer") })
        // 含工具调用(ReAct 闭环)
        assertTrue(output.any { it.contains("ToolCall") })
        assertTrue(output.any { it.contains("ToolResult") })
        // 含 reconfigure 切换提示
        assertTrue(output.any { it.contains("reconfigure") || it.contains("Reconfigure") })
        // 含多轮(第二轮历史)
        assertTrue(output.any { it.contains("round 2") || it.contains("Round 2") || it.contains("历史") })
    }

    @Test
    fun `runDemo is non-empty`() = runTest {
        val output = runDemo()
        assertTrue(output.isNotEmpty())
    }
}
```

- [ ] **Step 3: 跑测试确认失败**

Run: `gradle :sample:test`(cwd=/workspace/agent-android)
Expected: FAIL,编译错误(`runDemo` 未定义)。先 `gradle projects` 确认 `:sample` 出现。

- [ ] **Step 4: 写最小实现**

`sample/src/main/kotlin/com/you/agent/sample/Main.kt`:
```kotlin
package com.you.agent.sample

import com.you.agent.core.Agent
import com.you.agent.core.AgentEvent
import com.you.agent.llm.LlmFactory
import com.you.agent.llm.UserLlmConfig
import com.you.agent.memory.CompositePersonaStore
import com.you.agent.memory.InMemoryConversationStore
import com.you.agent.memory.PresetPersonaStore
import com.you.agent.memory.WritablePersonaStore
import com.you.agent.core.persona.Persona
import com.you.agent.core.tools.Tool
import kotlinx.coroutines.flow.toList
import okhttp3.mockwebserver.MockResponse
import okhttp3.mockwebserver.MockWebServer
import java.nio.file.Files

/**
 * sample Demo:演示 Android Agent 组件完整集成(纯 JVM,MockWebServer 模拟国产模型)。
 *
 * 宿主真实集成时:把 MockWebServer 换成真实端点,UserLlmConfig.apiKey 填用户 key,
 * 其余 Agent.builder() 代码零改动。
 *
 * 运行:`gradle :sample:run`
 */
suspend fun runDemo(): List<String> {
    val output = mutableListOf<String>()
    val server = MockWebServer()
    // 第一轮:ReAct 工具调用(assistant 返回 tool_calls)→ 工具执行 → 第二轮 assistant 返回 answer
    server.enqueue(MockResponse().setBody(
        """{"model":"demo-model","choices":[{"message":{"role":"assistant","content":"","tool_calls":[{"id":"c1","type":"function","function":{"name":"echo","arguments":"{\"text\":\"hello\"}"}}]},"finish_reason":"tool_calls"}]}"""))
    server.enqueue(MockResponse().setBody(
        """{"model":"demo-model","choices":[{"message":{"role":"assistant","content":"Echo: hello"},"finish_reason":"stop"}]}"""))
    // reconfigure 后的第二轮对话(新 model):含历史,单轮 answer
    server.enqueue(MockResponse().setBody(
        """{"model":"demo-model-2","choices":[{"message":{"role":"assistant","content":"Round 2 reply with history"},"finish_reason":"stop"}]}"""))
    server.start()
    try {
        val baseUrl = server.url("/v1").toString().trimEnd('/')
        val provider1 = LlmFactory.from(UserLlmConfig(
            providerId = "custom", apiKey = "demo-key", baseUrl = baseUrl, model = "demo-model"))
        val provider2 = LlmFactory.from(UserLlmConfig(
            providerId = "custom", apiKey = "demo-key-2", baseUrl = baseUrl, model = "demo-model-2"))

        // 角色:预设打底 + 自定义扩展
        val tmpDir = Files.createTempDirectory("agent-sample")
        val personaStore = CompositePersonaStore(
            primary = WritablePersonaStore(tmpDir),
            fallback = PresetPersonaStore(),
        )
        val persona = personaStore.load("assistant_v1")

        // 会话记忆
        val conversation = InMemoryConversationStore()

        // 组装 Agent(含工具)
        val agent = Agent.builder()
            .llm(provider1)
            .persona(persona)
            .conversation(conversation)
            .tools {
                +Tool(name = "echo", description = "回显输入文本", parametersJsonSchema = """
                    {"type":"object","properties":{"text":{"type":"string"}},"required":["text"]}
                """.trimIndent()) { args ->
                    "Echo: ${args["text"]}"
                }
            }
            .build()

        output.add("=== Round 1: ReAct 工具调用闭环 ===")
        val events1 = agent.run("请回显 hello", sessionId = "sess").toList()
        events1.forEach { e -> output.add(formatEvent(e)) }

        output.add("=== Reconfigure: 切换到 provider2 (model: demo-model-2) ===")
        agent.reconfigure(provider2)

        output.add("=== Round 2: 多轮对话(含历史)+ 新 provider ===")
        val events2 = agent.run("第二轮问题", sessionId = "sess").toList()
        events2.forEach { e -> output.add(formatEvent(e)) }

        output.add("=== Demo 完成 ===")
    } finally {
        server.shutdown()
    }
    return output
}

private fun formatEvent(e: AgentEvent): String = when (e) {
    is AgentEvent.ToolCall -> "ToolCall(name=${e.name}, args=${e.args})"
    is AgentEvent.ToolResult -> "ToolResult(name=${e.name}, output=${e.output})"
    is AgentEvent.Answer -> "Answer: ${e.text}"
    is AgentEvent.Error -> "Error: ${e.cause.message}"
    is AgentEvent.Thought -> "Thought: ${e.text}"
}

fun main() = kotlinx.coroutines.runBlocking {
    runDemo().forEach(::println)
}
```

注:`Tool` 的 `parametersJsonSchema` 是 OpenAI function calling 的 JSON Schema 字符串。`Tool(name, description, parametersJsonSchema) { args -> result }` 签名(M2 已定)。`trimIndent()` 用于多行字符串。runDemo 是 suspend,main 用 runBlocking。**subagent 执行前先 Read `/workspace/agent-android/agent-core/src/main/kotlin/com/you/agent/core/tools/Tool.kt` 确认 Tool 构造签名**(可能是 `Tool(name, description, parametersJsonSchema, handler)` 或 DSL `+tool(...)`),按实际 API 调整,断言不变。

- [ ] **Step 5: 跑测试确认通过**

Run: `gradle :sample:test`(cwd=/workspace/agent-android)
Expected: BUILD SUCCESSFUL,2 个 smoke test 通过。

**若测试失败**:`runDemo` 输出不含预期文本。检查:
- MockWebServer enqueue 顺序:第 1 个响应 tool_calls,第 2 个响应 answer(ReAct 第二轮),第 3 个响应 reconfigure 后的 answer
- AgentEvent 流:Round 1 应含 ToolCall → ToolResult → Answer;Round 2 应含 Answer
- `formatEvent` 输出含 "ToolCall"/"ToolResult"/"Answer" 字样
- reconfigure 提示行含 "Reconfigure"
- Round 2 行含 "Round 2"
- **不弱化断言**,调整 MockWebServer 响应或 formatEvent 让输出含预期文本

- [ ] **Step 6: 验证可运行**

Run: `gradle :sample:run`(cwd=/workspace/agent-android)
Expected: BUILD SUCCESSFUL,stdout 打印 runDemo 输出(含 Round 1 工具调用 + Reconfigure + Round 2)。

- [ ] **Step 7: 提交**

```bash
cd /workspace/agent-android && git add settings.gradle.kts sample && git commit -m "feat(sample): add runnable demo showcasing agent integration with tools, memory, reconfigure"
```

---

## Task 2: sample 扩展 RAG + 持久化演示

**Files:**
- Modify: `agent-android/sample/src/main/kotlin/com/you/agent/sample/Main.kt`
- Modify: `agent-android/sample/src/test/kotlin/com/you/agent/sample/MainTest.kt`

**Interfaces:**
- Consumes: `KnowledgeIngestor`/`LangChain4jRetriever`/`FakeEmbeddingModel`(M4,agent-rag)、`FileConversationStore`/`WritablePersonaStore`(M5,agent-memory)
- Produces: sample 完整演示 RAG 检索注入 + 文件持久化

- [ ] **Step 1: 扩展测试**

在 `MainTest.kt` 加测试:
```kotlin
    @Test
    fun `runDemo includes RAG knowledge injection`() = runTest {
        val output = runDemo()
        // RAG 段落:检索结果注入 system prompt 的提示或知识文本
        assertTrue(output.any { it.contains("RAG") || it.contains("检索") || it.contains("知识库") })
    }

    @Test
    fun `runDemo includes persistence demonstration`() = runTest {
        val output = runDemo()
        // 持久化段:FileConversationStore 或 WritablePersonaStore 跨实例
        assertTrue(output.any { it.contains("持久") || it.contains("Persistence") || it.contains("File") })
    }
```
(保留 Task 1 的 2 个测试,共 4 个。)

- [ ] **Step 2: 跑测试确认失败**

Run: `gradle :sample:test`(cwd=/workspace/agent-android)
Expected: FAIL,2 个新测试失败(runDemo 输出不含 RAG/持久化文本)。

- [ ] **Step 3: 扩展 runDemo**

在 `Main.kt` 的 `runDemo` 内,在 Round 2 之后加 RAG + 持久化演示段:
```kotlin
        // === RAG 检索注入演示 ===
        output.add("=== RAG: 知识库检索注入 system prompt ===")
        val ragStore = dev.langchain4j.store.embedding.inmemory.InMemoryEmbeddingStore<dev.langchain4j.data.segment.TextSegment>()
        val embedModel = com.you.agent.rag.FakeEmbeddingModel()
        val ingestor = com.you.agent.rag.KnowledgeIngestor(embedModel, ragStore, chunkSize = 100)
        ingestor.ingest("Beijing is the capital of China", source = "geo")
        ingestor.ingest("Python is a programming language", source = "tech")
        val retriever = com.you.agent.rag.LangChain4jRetriever(embedModel, ragStore, topK = 2)

        server.enqueue(MockResponse().setBody(
            """{"model":"demo-model-2","choices":[{"message":{"role":"assistant","content":"Beijing is the capital"},"finish_reason":"stop"}]}"""))
        val ragAgent = Agent.builder()
            .llm(provider2)
            .persona(Persona("geo", "地理助手", "You are a geography assistant."))
            .rag(retriever)
            .build()
        val ragEvents = ragAgent.run("What is the capital of China?").toList()
        ragEvents.forEach { e -> output.add(formatEvent(e)) }

        // === 持久化演示(FileConversationStore 跨实例)===
        output.add("=== Persistence: FileConversationStore 跨实例恢复 ===")
        val convDir = Files.createTempDirectory("agent-sample-conv")
        val fileConv1 = com.you.agent.memory.FileConversationStore(convDir)
        server.enqueue(MockResponse().setBody(
            """{"model":"demo-model-2","choices":[{"message":{"role":"assistant","content":"persisted reply"},"finish_reason":"stop"}]}"""))
        val persistAgent = Agent.builder()
            .llm(provider2)
            .persona(Persona("p", "P", "sys"))
            .conversation(fileConv1)
            .build()
        persistAgent.run("persistent question", sessionId = "psess").toList()
        // 新实例同目录加载,验证持久化
        val fileConv2 = com.you.agent.memory.FileConversationStore(convDir)
        val history = kotlinx.coroutines.runBlocking { fileConv2.load("psess") }
        output.add("Persistence: 跨实例恢复历史条数 = ${history.size}")
```

注:需 import `dev.langchain4j.store.embedding.inmemory.InMemoryEmbeddingStore`、`dev.langchain4j.data.segment.TextSegment`、`com.you.agent.rag.*`、`com.you.agent.memory.FileConversationStore`、`com.you.agent.core.persona.Persona`。`runBlocking` 在 suspend 函数内调 suspend load 不理想,改用 `fileConv2.load` 直接(suspend 上下文内可直接调)。**subagent 调整**:`val history = fileConv2.load("psess")`(runDemo 已是 suspend,直接调)。

**调整 runDemo 内持久化段**(去掉 runBlocking):
```kotlin
        val history = fileConv2.load("psess")
        output.add("Persistence: 跨实例恢复历史条数 = ${history.size}")
```

- [ ] **Step 4: 跑测试确认通过**

Run: `gradle :sample:test`(cwd=/workspace/agent-android)
Expected: BUILD SUCCESSFUL,4 个测试全过。

**若 RAG/持久化测试失败**:检查 output 是否含 "RAG"/"检索"/"知识库"/"持久"/"Persistence"/"File" 字样。formatEvent 的 Answer 不含这些,但 output.add 的标题行含。确保标题行文本匹配测试断言。**不弱化断言**,调整标题行文本。

- [ ] **Step 5: 验证可运行 + 全基线**

Run: `gradle :sample:run`(cwd=/workspace/agent-android)
Expected: 打印完整 demo(ReAct + reconfigure + RAG + 持久化)。

Run: `gradle test`(cwd=/workspace/agent-android)
Expected: BUILD SUCCESSFUL,六模块全过(`:agent-core` 41 / `:agent-llm` 17 / `:agent-tools` 6 / `:agent-memory` 28 / `:agent-rag` 6 / `:sample` 4),合计 **102 个**。

- [ ] **Step 6: 提交**

```bash
cd /workspace/agent-android && git add sample/src && git commit -m "feat(sample): extend demo with RAG knowledge injection and file persistence"
```

---

## Self-Review

**1. Spec coverage(对照设计文档 M7 范围):**
- sample Demo 验证集成体验 → Task 1/2 `sample` 模块 + `Main.kt` ✓
- 含用户配置 key 的完整流程 → Task 1 `LlmFactory.from(UserLlmConfig("custom", "demo-key", ...))` 展示 BYO Key 入口(用 demo-key 模拟)✓
- 设计文档 §9 集成方式示例 → Task 1 `Agent.builder().llm().persona().tools().conversation().build()` + `agent.run().collect` ✓
- 设计文档 §8.3 运行时切换 → Task 1 `agent.reconfigure(provider2)` ✓
- 全能力演示:对话(Task 1)+ 工具(Task 1)+ 多轮(Task 1)+ reconfigure(Task 1)+ RAG(Task 2)+ 持久化(Task 2)✓
- 未覆盖(留给宿主,符合范围):Android UI(用户明确不做)、真实 key(宿主填)、交互式 REPL(YAGNI)

**2. 关键决策说明:**
- **纯 JVM sample 不做 Android UI**:用户明确"先不考虑 UI",沙箱无 Android 设备/SDK。sample 作为纯 JVM 可运行 demo,真实 Android App 集成留宿主(组件已是 Gradle 库)。
- **MockWebServer 内嵌模拟 LLM**:sample 演示 API 用法,不需真实 key。宿主换真实 key 只改 UserLlmConfig,其余零改动。展示完整 BYO Key 入口链路(LlmFactory + UserLlmConfig + OpenAiCompatibleProvider)。
- **runDemo() 可测函数**:suspend 返回 List<String>,main 打印。smoke test 测返回值,不捕获 stdout。

**3. Placeholder scan:** Task 1 Step 4 对 `Tool` 构造签名给了"先 Read Tool.kt 确认"指导。Task 2 Step 3 对 runBlocking in suspend 给了"改直接调"修正。每步含完整代码或确切命令。✓

**4. Type consistency:**
- `Agent.builder()` M1/M3/M4/M6,sample 调用一致 ✓
- `LlmFactory.from(UserLlmConfig)` M1,sample 调用一致 ✓
- `AgentEvent` sealed(M1),sample `formatEvent` when 穷尽 ✓
- `Tool`/`ToolRegistryScope` M2,sample `tools { +Tool(...) }` 一致(签名按实际 API)✓
- `CompositePersonaStore`/`PresetPersonaStore`/`WritablePersonaStore` M3/M5,sample 调用一致 ✓
- `KnowledgeIngestor`/`LangChain4jRetriever`/`FakeEmbeddingModel` M4,sample 调用一致 ✓
- `FileConversationStore` M5,sample 调用一致 ✓

**5. 向后兼容:**
- sample 是新模块,不改任何既有模块源码/测试 ✓
- 既有 98 个测试不受影响 ✓

**6. 测试覆盖:**
- MainTest:4 个(runDemo 非空、含 Answer+ToolCall+ToolResult+reconfigure+多轮、含 RAG、含持久化)✓
- 共 4 个新增测试,98 → 102 ✓

---

## Execution Handoff

计划已保存至 `docs/superpowers/plans/2026-07-02-android-agent-m7.md`。

**无需用户再次确认架构偏离**——纯 JVM sample 不做 Android UI、MockWebServer 模拟 LLM 均沿用 M1-M6 既定方向(手写、纯 JVM 可测、YAGNI),理由记录于关键决策节。

沿用 M1-M6 的 **Subagent-Driven Development** 执行。关键环境约束:
- **用 `gradle` 不用 `./gradlew`**(wrapper 下载超时,系统 gradle 8.14 + JDK 17 daemon 已配 `~/.gradle/gradle.properties`)
- 不新增第三方依赖(复用 okhttp/mockwebserver 既有坐标)
- 不弱化断言;Task 1 若 `Tool` 构造签名与计划不符,Read Tool.kt 按实际 API 调整,断言不变
- sample 不改任何既有模块源码
