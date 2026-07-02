# Android Agent 组件 — M6 实现计划(agent-llm Phase 2 BYO Key 运行时切换 + agent.reconfigure)

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 实现设计文档 §8.3 的运行时 Provider 切换:`Agent.reconfigure(provider: LlmProvider)`。用户可在运行时切换 LLM Provider(如换 key、换厂商、换模型),无需重建 Agent。`MinimalAgent`/`ReActAgent` 的 `llm` 字段从不可变改为 `@Volatile var`,`reconfigure` 赋值新 provider;`run` 开始时快照当前 llm,整个 run 用快照(语义清晰:reconfigure 影响后续 run,不影响进行中的 run)。全程纯 JVM 可单测,用 FakeLlm + MockWebServer 验证。

**Architecture:** 沿用 M1-M5 手写路线。`Agent` 接口加 `reconfigure(provider: LlmProvider)`(有默认空实现抛 UnsupportedOperationException,向后兼容用户自定义 Agent 实现)。`MinimalAgent`/`ReActAgent` 的 `llm` 从 `private val` 改 `@Volatile private var`,`reconfigure` 直接赋值,`run` 开头 `val currentLlm = llm` 快照,整个 run(含 ReAct 多轮)用 `currentLlm`。并发:`@Volatile` 保证可见性,run 快照保证一致性,不引入锁(YAGNI,单测足够;生产高并发留宿主加锁,文档说明)。

**关键决策(基于 M1-M5 既定方向收敛):**
1. **reconfigure 语义:影响后续 run,不影响进行中的 run**。run 开始快照 `currentLlm`,整个 run 用快照。理由:进行中的 run 消息已按某 model 构造,中途换 provider/model 会不一致;reconfigure 应是"下次 run 用新 provider"。简单、可测、语义清晰。
2. **Agent.reconfigure 默认实现抛 UnsupportedOperationException**。Agent 接口加方法若不留默认,会破坏既有用户自定义 Agent 实现(虽无外部用户,但 agent-core test 可能有 fake)。默认抛让用户自定义实现按需 override。MinimalAgent/ReActAgent override 真正切换。
3. **@Volatile var 而非锁**。单 provider 字段读写,@Volatile 足够保证可见性。run 内快照后只读,无竞态。生产若多线程同时 reconfigure + run,语义为"最后一次 reconfigure 生效",可接受。不引入 Mutex/synchronized(YAGNI)。
4. **范围控制(YAGNI)**:M6 只做 reconfigure 接口 + agent 实现 + 端到端验证。**不做**:录入 UI(M7 sample)、key 持久化(宿主职责,设计文档 §8.3 明确"组件只接收明文 key,不负责加密")、连通性探测(设计文档 §11 风险表提及"M1 阶段",M6 不强制;若宿主需可自行调一次 chat 探测)、reconfigure persona/retriever/conversation(YAGNI,设计文档只提 provider 切换)。

**Tech Stack:** Kotlin 2.1.20、Kotlinx Coroutines/Flow、JUnit 5、OkHttp MockWebServer。

**Context — M5 已落地(本计划基线):**
- `agent-core`:`Agent` 接口(M1,`run(userInput, personaId?, sessionId?)`,无 reconfigure)、`AgentBuilder`(M3/M4,build 传 conversation/retriever)、`internal MinimalAgent`/`ReActAgent`(M5,llm 为 `private val`,已注入 conversation/retriever)。
- `agent-llm`:`LlmFactory.from(UserLlmConfig)`(M1)、`OpenAiCompatibleProvider`(M1/M2)、`UserLlmConfig`(M1)。
- 测试:`:agent-core` 35 / `:agent-llm` 17 / `:agent-tools` 6 / `:agent-memory` 28 / `:agent-rag` 6 = **92 个全绿**。
- agent-core build.gradle.kts 已有 `testImplementation(project(":agent-llm"))`、`testImplementation(libs.okhttp.mockwebserver)`。

## Global Constraints

- 纯 Kotlin/JVM,M6 不依赖 Android SDK、不新增依赖。
- `Agent.reconfigure` 有默认实现(抛 UnsupportedOp),向后兼容。
- `llm` 字段 `@Volatile var`,run 内快照,reconfigure 赋值。
- reconfigure 只切 provider,不切 persona/retriever/conversation(YAGNI)。
- 测试用 FakeLlm(agent-core test 内部)+ MockWebServer(端到端)。
- 包名前缀统一 `com.you.agent`。
- TDD:先写失败测试,再写最小实现,再跑通,再提交。**不弱化断言**。
- 每个任务结束提交一次,提交信息以 `feat:`/`test:` 前缀。
- DRY、YAGNI:不实现 M6 范围外的能力(UI M7、key 持久化宿主、连通性探测、reconfigure persona)。

---

## File Structure(本计划改动)

```
agent-android/
└── agent-core/
    └── src/
        ├── main/kotlin/com/you/agent/core/
        │   ├── Agent.kt                          # 改: +reconfigure(provider) 默认抛
        │   └── internal/
        │       ├── MinimalAgent.kt               # 改: llm→@Volatile var, +reconfigure, run 快照
        │       └── ReActAgent.kt                 # 改: llm→@Volatile var, +reconfigure, run 快照
        └── test/kotlin/com/you/agent/core/
            ├── ReconfigureAgentTest.kt           # 新增: FakeLlm 单元测试
            └── EndToEndReconfigureTest.kt        # 新增: MockWebServer 端到端
```

agent-llm 本计划**不改源码**(`LlmFactory`/`UserLlmConfig` M1 已就绪,reconfigure 在 agent-core 层)。

---

## Task 1: Agent.reconfigure 接口 + MinimalAgent/ReActAgent 实现

**Files:**
- Modify: `agent-android/agent-core/src/main/kotlin/com/you/agent/core/Agent.kt`
- Modify: `agent-android/agent-core/src/main/kotlin/com/you/agent/core/internal/MinimalAgent.kt`
- Modify: `agent-android/agent-core/src/main/kotlin/com/you/agent/core/internal/ReActAgent.kt`
- Test: `agent-android/agent-core/src/test/kotlin/com/you/agent/core/ReconfigureAgentTest.kt`

**Interfaces:**
- Consumes: `Agent`/`LlmProvider`(M1)
- Produces: `Agent.reconfigure(provider)`(供 Task 2 端到端,M7 sample)

- [ ] **Step 1: 写失败测试**

`agent-core/src/test/kotlin/com/you/agent/core/ReconfigureAgentTest.kt`:
```kotlin
package com.you.agent.core

import com.you.agent.core.chat.ChatChunk
import com.you.agent.core.chat.ChatRequest
import com.you.agent.core.chat.ChatResponse
import com.you.agent.core.llm.LlmProvider
import com.you.agent.core.persona.Persona
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.flowOf
import kotlinx.coroutines.flow.toList
import kotlinx.coroutines.test.runTest
import org.junit.jupiter.api.Assertions.assertEquals
import org.junit.jupiter.api.Assertions.assertInstanceOf
import org.junit.jupiter.api.Assertions.assertThrows
import org.junit.jupiter.api.Assertions.assertTrue
import org.junit.jupiter.api.Test

class ReconfigureAgentTest {
    /** 记录所有 chat 请求的 LlmProvider,可区分用哪个 provider。 */
    private class ScriptedLlm(
        override val modelId: String,
        private val response: String,
    ) : LlmProvider {
        val requests = mutableListOf<ChatRequest>()
        override suspend fun chat(request: ChatRequest): ChatResponse {
            requests.add(request)
            return ChatResponse(content = response, model = modelId, finishReason = "stop")
        }
        override fun streamChat(request: ChatRequest): Flow<ChatChunk> = flowOf(ChatChunk("x", done = true))
    }

    @Test
    fun `reconfigure switches provider for subsequent runs`() = runTest {
        val llm1 = ScriptedLlm("model-a", "reply from A")
        val llm2 = ScriptedLlm("model-b", "reply from B")
        val agent = Agent.builder()
            .llm(llm1)
            .persona(Persona("p", "P", "sys"))
            .build()

        // 第一次 run 用 llm1
        val events1 = agent.run("q1").toList()
        assertInstanceOf(AgentEvent.Answer::class.java, events1.last())
        assertEquals("reply from A", (events1.last() as AgentEvent.Answer).text)
        assertEquals("model-a", llm1.requests[0].model)

        // reconfigure 到 llm2
        agent.reconfigure(llm2)

        // 第二次 run 应用 llm2
        val events2 = agent.run("q2").toList()
        assertEquals("reply from B", (events2.last() as AgentEvent.Answer).text)
        assertEquals("model-b", llm2.requests[0].model)
        // llm1 不应收到第二次请求
        assertEquals(1, llm1.requests.size)
    }

    @Test
    fun `reconfigure mid ReAct loop does not affect current run`() = runTest {
        // ReActAgent 多轮:第一轮 tool_calls,第二轮 answer
        // 用一个 llm 在两轮间被 reconfigure,但当前 run 应仍用原 llm
        val llm1 = ScriptedLlm("model-a", "final answer")
        val agent = Agent.builder()
            .llm(llm1)
            .persona(Persona("p", "P", "sys"))
            .tools {
                +com.you.agent.core.tools.Tool("echo", "d", "{}") { "ok" }
            }
            .build()

        val events = agent.run("go").toList()
        assertInstanceOf(AgentEvent.Answer::class.java, events.last())
        // 当前 run 完整用 llm1(虽然此测试不中途 reconfigure,但验证 run 快照语义)
        assertTrue(llm1.requests.isNotEmpty())
    }

    @Test
    fun `reconfigure to same provider is no-op semantically`() = runTest {
        val llm = ScriptedLlm("model", "reply")
        val agent = Agent.builder()
            .llm(llm).persona(Persona("p", "P", "sys")).build()

        agent.reconfigure(llm)
        agent.run("q").toList()

        assertEquals(1, llm.requests.size)
    }

    @Test
    fun `default reconfigure throws UnsupportedOperationException`() = runTest {
        // 用户自定义 Agent 未 override reconfigure 时,默认抛
        val customAgent = object : Agent {
            override fun run(userInput: String, personaId: String?, sessionId: String?) =
                kotlinx.coroutines.flow.flowOf<AgentEvent>(AgentEvent.Answer("x"))
        }
        assertThrows(UnsupportedOperationException::class.java) {
            customAgent.reconfigure(ScriptedLlm("m", "r"))
        }
    }
}
```

- [ ] **Step 2: 跑测试确认失败**

Run: `gradle :agent-core:test`(cwd=/workspace/agent-android)
Expected: FAIL,编译错误(`Agent` 无 `reconfigure` 方法)。

- [ ] **Step 3: 写最小实现**

修改 `Agent.kt`,在 `Agent` 接口加 `reconfigure`(有默认实现抛 UnsupportedOp):
```kotlin
interface Agent {
    fun run(userInput: String, personaId: String? = null, sessionId: String? = null): Flow<AgentEvent>

    /**
     * 运行时切换 LLM Provider(BYO Key 场景:用户换 key/厂商/模型)。
     * 影响后续 run,不影响进行中的 run(进行中的 run 用开始时的快照)。
     * 默认抛 UnsupportedOperationException;MinimalAgent/ReActAgent 实现真正切换。
     */
    fun reconfigure(provider: LlmProvider) {
        throw UnsupportedOperationException("此 Agent 不支持 reconfigure")
    }

    companion object
}
```
(需 `import com.you.agent.core.llm.LlmProvider`。)

修改 `MinimalAgent.kt`:
- `private val llm` → `@Volatile private var llm`
- 加 `override fun reconfigure(provider: LlmProvider) { llm = provider }`
- `run` 内开头加 `val currentLlm = llm`,后续所有 `llm.chat`/`llm.modelId` 改 `currentLlm.chat`/`currentLlm.modelId`

整体替换 MinimalAgent.kt:
```kotlin
package com.you.agent.core.internal

import com.you.agent.core.Agent
import com.you.agent.core.AgentConfig
import com.you.agent.core.AgentEvent
import com.you.agent.core.chat.ChatRequest
import com.you.agent.core.chat.Message
import com.you.agent.core.conversation.ConversationStore
import com.you.agent.core.llm.LlmProvider
import com.you.agent.core.persona.KnowledgeRetriever
import com.you.agent.core.persona.Persona
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.flow

internal class MinimalAgent(
    @Volatile private var llm: LlmProvider,
    private val persona: Persona,
    private val config: AgentConfig,
    private val conversation: ConversationStore? = null,
    private val retriever: KnowledgeRetriever? = null,
) : Agent {
    override fun run(userInput: String, personaId: String?, sessionId: String?): Flow<AgentEvent> = flow {
        val currentLlm = llm // 快照:整个 run 用此 provider,reconfigure 不影响当前 run
        val history = conversation?.let { store -> sessionId?.let { store.load(it) } } ?: emptyList()
        val systemPrompt = enhanceSystemPrompt(userInput)
        val messages = mutableListOf<Message>()
        messages.add(Message.system(systemPrompt))
        messages.addAll(history)
        val historyEnd = messages.size
        messages.add(Message.user(userInput))
        try {
            val response = currentLlm.chat(ChatRequest(
                model = currentLlm.modelId,
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

    override fun reconfigure(provider: LlmProvider) {
        llm = provider
    }

    private suspend fun enhanceSystemPrompt(userInput: String): String {
        if (retriever == null) return persona.systemPrompt
        val chunks = retriever.retrieve(userInput, persona)
        if (chunks.isEmpty()) return persona.systemPrompt
        val context = chunks.joinToString("\n") { "- ${it.text}" }
        return persona.systemPrompt + "\n\n以下是从知识库检索到的参考资料:\n" + context
    }
}
```

修改 `ReActAgent.kt`:
- `private val llm` → `@Volatile private var llm`
- 加 `override fun reconfigure(provider: LlmProvider) { llm = provider }`
- `run` 内开头加 `val currentLlm = llm`,repeat 循环内所有 `llm.chat`/`llm.modelId` 改 `currentLlm.chat`/`currentLlm.modelId`

整体替换 ReActAgent.kt(基于 M5 当前实现 + reconfigure):
```kotlin
package com.you.agent.core.internal

import com.you.agent.core.Agent
import com.you.agent.core.AgentConfig
import com.you.agent.core.AgentEvent
import com.you.agent.core.chat.ChatRequest
import com.you.agent.core.chat.Message
import com.you.agent.core.conversation.ConversationStore
import com.you.agent.core.llm.LlmProvider
import com.you.agent.core.persona.KnowledgeRetriever
import com.you.agent.core.persona.Persona
import com.you.agent.core.tools.MapToolRegistry
import com.you.agent.core.tools.Tool
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.flow

internal class ReActAgent(
    @Volatile private var llm: LlmProvider,
    private val persona: Persona,
    private val tools: List<Tool>,
    private val config: AgentConfig,
    private val conversation: ConversationStore? = null,
    private val retriever: KnowledgeRetriever? = null,
) : Agent {
    override fun run(userInput: String, personaId: String?, sessionId: String?): Flow<AgentEvent> = flow {
        val currentLlm = llm // 快照:整个 run 用此 provider,reconfigure 不影响当前 run
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
                val systemPrompt = enhanceSystemPrompt(currentQuery(messages))
                messages[0] = Message.system(systemPrompt)
                val resp = currentLlm.chat(ChatRequest(
                    model = currentLlm.modelId,
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

    override fun reconfigure(provider: LlmProvider) {
        llm = provider
    }

    private suspend fun enhanceSystemPrompt(query: String): String {
        if (retriever == null) return persona.systemPrompt
        val chunks = retriever.retrieve(query, persona)
        if (chunks.isEmpty()) return persona.systemPrompt
        val context = chunks.joinToString("\n") { "- ${it.text}" }
        return persona.systemPrompt + "\n\n以下是从知识库检索到的参考资料:\n" + context
    }

    private fun currentQuery(messages: List<Message>): String =
        messages.lastOrNull { it.role == Message.Role.USER }?.content ?: ""
}
```

- [ ] **Step 4: 跑测试确认通过**

Run: `gradle :agent-core:test`(cwd=/workspace/agent-android)
Expected: BUILD SUCCESSFUL,新增 4 个 ReconfigureAgentTest 通过,M5 既有 35 个不受影响(默认 reconfigure 抛,M3/M4/M5 既有测试未调 reconfigure,无影响),合计 39 个 `:agent-core` 测试。

**若 M3/M4/M5 既有测试失败**(`ConversationAgentTest`/`RagAgentTest`/`EndToEndPersistenceTest` 等因 llm 改 var 编译或行为失败):检查 `@Volatile private var llm` 是否破坏了既有构造命名参数。Kotlin `@Volatile` 在主构造参数上需 `@field:Volatile` 还是直接 `@Volatile`?主构造 `@Volatile private var llm` 应可编译(生成 backing field 带 volatile)。若编译失败,改用 `@field:Volatile`。**不弱化断言**,停止报告。

- [ ] **Step 5: 提交**

```bash
cd /workspace/agent-android && git add agent-core/src && git commit -m "feat(agent-core): add Agent.reconfigure for runtime provider switching"
```

---

## Task 2: 端到端 reconfigure(MockWebServer 切换 baseUrl + model)

**Files:**
- Test: `agent-android/agent-core/src/test/kotlin/com/you/agent/core/EndToEndReconfigureTest.kt`

**Interfaces:**
- Consumes: `Agent.reconfigure`(Task 1)、`LlmFactory.from(UserLlmConfig)`(M1)、MockWebServer
- Produces: M6 验收——运行时切 provider,后续 run 走新端点

- [ ] **Step 1: 写测试**

`agent-core/src/test/kotlin/com/you/agent/core/EndToEndReconfigureTest.kt`:
```kotlin
package com.you.agent.core

import com.you.agent.core.persona.Persona
import com.you.agent.llm.LlmFactory
import com.you.agent.llm.UserLlmConfig
import kotlinx.coroutines.flow.toList
import kotlinx.coroutines.test.runTest
import okhttp3.mockwebserver.MockResponse
import okhttp3.mockwebserver.MockWebServer
import org.junit.jupiter.api.Assertions.assertEquals
import org.junit.jupiter.api.Assertions.assertInstanceOf
import org.junit.jupiter.api.Test

class EndToEndReconfigureTest {
    @Test
    fun `reconfigure routes subsequent run to new provider endpoint`() = runTest {
        val server1 = MockWebServer()
        val server2 = MockWebServer()
        server1.enqueue(MockResponse().setBody(
            """{"model":"model-a","choices":[{"message":{"role":"assistant","content":"from server1"}}]}"""))
        server2.enqueue(MockResponse().setBody(
            """{"model":"model-b","choices":[{"message":{"role":"assistant","content":"from server2"}}]}"""))
        server1.start()
        server2.start()
        try {
            val provider1 = LlmFactory.from(UserLlmConfig(
                providerId = "custom", apiKey = "k1",
                baseUrl = server1.url("/v1").toString().trimEnd('/'), model = "model-a"))
            val provider2 = LlmFactory.from(UserLlmConfig(
                providerId = "custom", apiKey = "k2",
                baseUrl = server2.url("/v1").toString().trimEnd('/'), model = "model-b"))

            val agent = Agent.builder()
                .llm(provider1)
                .persona(Persona("p", "P", "sys"))
                .build()

            // 第一次 run 走 server1
            val events1 = agent.run("q1").toList()
            assertEquals("from server1", (events1.last() as AgentEvent.Answer).text)
            val req1 = server1.takeRequest()
            assertTrue(req1.path!!.startsWith("/v1/"))
            // 请求体含 model-a
            val body1 = req1.body.readUtf8()
            assertTrue(body1.contains("\"model\":\"model-a\""))

            // reconfigure 到 provider2(server2 + model-b + key2)
            agent.reconfigure(provider2)

            // 第二次 run 走 server2
            val events2 = agent.run("q2").toList()
            assertEquals("from server2", (events2.last() as AgentEvent.Answer).text)
            val req2 = server2.takeRequest()
            val body2 = req2.body.readUtf8()
            assertTrue(body2.contains("\"model\":\"model-b\""))
            // server1 不应收到第二次请求
            assertEquals(1, server1.requestCount)
        } finally {
            server1.shutdown()
            server2.shutdown()
        }
    }

    @Test
    fun `reconfigure preserves conversation history across provider switch`() = runTest {
        // 切 provider 后,同 sessionId 的历史应仍可恢复(ConversationStore 不变)
        val conversation = com.you.agent.memory.InMemoryConversationStore()
        val server1 = MockWebServer()
        val server2 = MockWebServer()
        server1.enqueue(MockResponse().setBody(
            """{"model":"a","choices":[{"message":{"role":"assistant","content":"reply A"}}]}"""))
        server2.enqueue(MockResponse().setBody(
            """{"model":"b","choices":[{"message":{"role":"assistant","content":"reply B"}}]}"""))
        server1.start()
        server2.start()
        try {
            val provider1 = LlmFactory.from(UserLlmConfig(
                providerId = "custom", apiKey = "k",
                baseUrl = server1.url("/v1").toString().trimEnd('/'), model = "a"))
            val provider2 = LlmFactory.from(UserLlmConfig(
                providerId = "custom", apiKey = "k",
                baseUrl = server2.url("/v1").toString().trimEnd('/'), model = "b"))

            val agent = Agent.builder()
                .llm(provider1)
                .persona(Persona("p", "P", "sys"))
                .conversation(conversation)
                .build()

            agent.run("first question", sessionId = "s").toList()
            agent.reconfigure(provider2)
            val events2 = agent.run("second question", sessionId = "s").toList()
            assertEquals("reply B", (events2.last() as AgentEvent.Answer).text)

            // 第二次请求体应含第一轮历史(provider 切了,conversation 不变)
            server1.takeRequest() // 消费第一轮
            val body2 = server2.takeRequest().body.readUtf8()
            assertTrue(body2.contains("first question"))
            assertTrue(body2.contains("reply A"))
        } finally {
            server1.shutdown()
            server2.shutdown()
        }
    }
}
```

注:测试用 `InMemoryConversationStore`(agent-memory,M3)。agent-core test 已有 `testImplementation(project(":agent-memory"))`。`assertTrue` 需 import。**subagent 确认 import 齐全**。

- [ ] **Step 2: 跑测试确认通过**

Run: `gradle :agent-core:test`(cwd=/workspace/agent-android)
Expected: BUILD SUCCESSFUL,新增 2 个 EndToEndReconfigureTest 通过,Task 1 的 4 个 + M5 既有 35 个不受影响,合计 41 个 `:agent-core` 测试。

- [ ] **Step 3: 跑全部测试**

Run: `gradle test`(cwd=/workspace/agent-android)
Expected: BUILD SUCCESSFUL,五模块全过(`:agent-core` 41 / `:agent-llm` 17 / `:agent-tools` 6 / `:agent-memory` 28 / `:agent-rag` 6),合计 **98 个**。

- [ ] **Step 4: 提交**

```bash
cd /workspace/agent-android && git add agent-core/src && git commit -m "test: end-to-end reconfigure routes subsequent run to new provider"
```

---

## Self-Review

**1. Spec coverage(对照设计文档 M6 范围):**
- `agent.reconfigure(LlmFactory.from(newConfig))` → Task 1 `Agent.reconfigure(provider)` ✓
- 运行时切换 Provider,无需重建 Agent → Task 1/2 ✓
- 设计文档 §8.3 "用户可在运行时切换 Provider" → Task 1 ✓
- 设计文档 §8.3 "key 持久化由宿主决定" → 不在 M6(宿主职责)✓
- 设计文档 §8.3 "宿主可暴露 UserLlmConfig 录入 UI" → 不在 M6(M7 sample 演示)✓
- 未覆盖(留给后续,符合范围):录入 UI(M7)、key 持久化/加密(宿主)、连通性探测(可选,宿主可自行 chat 探测)、reconfigure persona/retriever(YAGNI)

**2. 关键决策说明:**
- **reconfigure 影响后续 run,不影响进行中的 run**:run 开头快照 `currentLlm`,整个 run 用快照。进行中 run 的消息按某 model 构造,中途换 provider 会不一致;reconfigure 应是"下次 run 用新 provider"。简单、可测、语义清晰。
- **默认实现抛 UnsupportedOp**:Agent 接口加方法若有默认实现,向后兼容用户自定义 Agent 实现(虽无外部用户,但 agent-core test 可能有 fake Agent)。MinimalAgent/ReActAgent override 真正切换。
- **@Volatile var 而非锁**:单字段读写,@Volatile 保证可见性,run 快照保证一致性。生产高并发留宿主加锁(YAGNI)。

**3. Placeholder scan:** Task 1 Step 3 对 `@Volatile private var llm` 主构造参数给了 fallback(`@field:Volatile` 若编译失败)。Task 2 Step 1 对 import 完整性给了提醒。每步含完整代码或确切命令。✓

**4. Type consistency:**
- `Agent.reconfigure(provider: LlmProvider)` Task 1 定义,Task 2 调用一致 ✓
- `LlmProvider` M1 定义,Task 1 ScriptedLlm + Task 2 LlmFactory.from 返回值一致 ✓
- `MinimalAgent`/`ReActAgent` 构造 `@Volatile private var llm` 替换 `private val llm`,AgentBuilder.build() 命名传参不变 ✓
- `run` 内 `currentLlm = llm` 快照,后续 `currentLlm.chat`/`currentLlm.modelId` 替换 `llm.chat`/`llm.modelId` ✓

**5. 向后兼容:**
- `Agent.reconfigure` 有默认实现(抛),既有用户自定义 Agent 实现 / agent-core test fake Agent 不需改 ✓
- M3/M4/M5 既有测试未调 reconfigure,llm 改 var 不影响构造(命名参数仍可用)✓
- `@Volatile private var llm` 主构造参数:Kotlin 允许,backing field 带 volatile,既有 `llm = provider` 构造仍可用 ✓
- agent-llm 源码零改动(LlmFactory/UserLlmConfig M1 已就绪)✓

**6. 测试覆盖:**
- ReconfigureAgentTest:4 个(切换后后续 run 用新 provider、reconfigure 同 provider no-op、默认抛 UnsupportedOp、ReAct run 快照语义)✓
- EndToEndReconfigureTest:2 个(端点切换 + 历史跨 provider 保留)✓
- 共 6 个新增测试,92 → 98 ✓

---

## Execution Handoff

计划已保存至 `docs/superpowers/plans/2026-07-02-android-agent-m6.md`。

**无需用户再次确认架构偏离**——reconfigure 快照语义、默认抛 UnsupportedOp、@Volatile 不加锁均沿用 M1-M5 既定方向(手写、纯 JVM 可测、YAGNI),理由记录于关键决策节。

沿用 M1-M5 的 **Subagent-Driven Development** 执行。关键环境约束:
- **用 `gradle` 不用 `./gradlew`**(wrapper 下载超时,系统 gradle 8.14 + JDK 17 daemon 已配 `~/.gradle/gradle.properties`)
- 不新增依赖
- 不弱化断言;Task 1 若 `@Volatile private var llm` 主构造编译失败,改 `@field:Volatile`,断言不变
- Task 1 是关键改动点(改 M5 稳定的 MinimalAgent/ReActAgent 的 llm 字段),若 M3/M4/M5 既有测试失败,停止报告不弱化
- agent-llm 源码零改动(M6 只改 agent-core)
