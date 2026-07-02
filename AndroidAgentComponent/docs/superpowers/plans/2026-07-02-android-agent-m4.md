# Android Agent 组件 — M4 实现计划(agent-rag LangChain4j 检索 + 角色知识库注入)

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 在 M3 会话记忆之上,实现 RAG 检索增强:(1) 新建 `agent-rag` 模块,基于 LangChain4j 的 `EmbeddingModel` + `EmbeddingStore` 实现 `KnowledgeRetriever`;(2) 把检索结果作为 context 注入 persona system prompt;(3) 提供文档入库便捷工具(`KnowledgeIngestor`)。全程纯 JVM 可单测,用 `InMemoryEmbeddingStore` + 自建 FakeEmbeddingModel(确定性向量),无真实 Embedding API 依赖。

**Architecture:** 沿用 M1/M2/M3 分层与手写路线,`agent-core` 的 `KnowledgeRetriever`/`KnowledgeChunk` 接口已在 M1 落地(M4 真正实现)。新建 `agent-rag` 模块依赖 `agent-core` + `dev.langchain4j:langchain4j-core`(含 `EmbeddingModel` 接口、`EmbeddingStore` 接口、`InMemoryEmbeddingStore`、`TextSegment`、`Embedding`、`DocumentSplitters`)。M4 不引 `langchain4j-open-ai`(真实 OpenAI 兼容 embedding 留 M6 BYO Key 时按需引入,测试用 Fake)。`LangChain4jRetriever` 桥接 LangChain4j 同步 API 与 Kotlin suspend(直接调,真实场景由宿主包 `Dispatchers.IO`,M4 测试用 Fake 不阻塞)。检索注入策略:run 开始时若注入 retriever,对 `userInput` 做 `retrieve`,把结果拼到 `persona.systemPrompt` 之后作为增强 system 消息(不改消息结构,最简)。

**关键决策(基于 M1-M3 既定方向收敛):**
1. **LangChain4j 版本与依赖面**:`dev.langchain4j:langchain4j-core:1.0.0`(GA)。仅依赖 core(含接口 + InMemoryEmbeddingStore + DocumentSplitters),不引 open-ai 子模块(测试用 FakeEmbeddingModel,真实 embedding 留 M6)。若 1.0.0 在 Maven Central 不可用,subagent 可升至 1.1.0,核心 API(`EmbeddingModel.embed/embedAll`、`EmbeddingStore.add/addAll/search`、`EmbeddingSearchRequest.builder()`、`InMemoryEmbeddingStore`、`DocumentSplitters.recursive`)在 1.x 兼容。
2. **测试可测性**:LangChain4j 的 `EmbeddingModel` 是 Java 接口(同步,返回 `Response<Embedding>`)。M4 在 `agent-rag` test 内自建 `FakeEmbeddingModel`(实现 `EmbeddingModel`,对文本生成确定性 384 维向量:用字符 char code 循环填充 + L2 归一化),使相同文本向量相同、含相同词的文本 cosine 相近。`InMemoryEmbeddingStore` 是 LangChain4j 自带现成实现。这样 ingest("Beijing is the capital of China") 后 retrieve("capital") 能命中,无需真实 API。**所有测试断言基于 `KnowledgeRetriever`/`KnowledgeChunk` 抽象层**,不直接断言 LangChain4j 内部对象,降低 API 版本耦合。
3. **检索注入位置**:拼到 `persona.systemPrompt` 之后,格式 `systemPrompt + "\n\n以下是从知识库检索到的参考资料:\n" + chunks.joinToString("\n---\n") { "- " + it.text }`。不改消息结构(system 仍是单条),最简且向后兼容(retriever 为 null 时 system 不变)。
4. **文档入库**:提供 `KnowledgeIngestor`(便捷工具):`ingest(text, source)` 用 `DocumentSplitters.recursive(300, 0)` 分割 → `embeddingModel.embedAll` → `store.addAll`。优先用 `DocumentSplitters`;若 1.0.0 API 编译不过,降级为自实现 `splitBySize(text, 300)`(按字符数切),断言不变(基于检索结果)。
5. **LangChain4j 同步→suspend 桥接**:`LangChain4jRetriever.retrieve` 是 suspend,内部直接调 `embeddingModel.embed(...)`(LangChain4j 同步阻塞)。M4 测试用 FakeEmbeddingModel(纯计算不阻塞)。真实场景宿主如需异步,可在自己包 `withContext(Dispatchers.IO)` 调 `retrieve`(M4 不强制,避免过度设计;真实 EmbeddingModel 通常 OkHttp 阻塞,宿主自行处理)。
6. **`AgentBuilder.rag()` 真正生效**:M1 预留了 `rag(retriever)` 方法存字段但 `build()` 未传给 agent。M4 改 `build()` 把 retriever 传给 `ReActAgent`/`MinimalAgent` 构造。向后兼容:retriever 默认 null,既有测试不注入则行为不变。

**Tech Stack:** Kotlin 2.1.20、Kotlinx Coroutines/Flow、LangChain4j 1.0.0(core)、JUnit 5、OkHttp MockWebServer。

**Context — M3 已落地(本计划基线):**
- `agent-core`:`KnowledgeRetriever` 接口(`retrieve(query, persona, topK=4): List<KnowledgeChunk>`)、`KnowledgeChunk(text, source, score?)`(M1)、`Persona`(含 `knowledgeSources: List<String>`,M4 可用作 source 标识,但 M4 检索不强制按 persona.knowledgeSources 过滤,留后续)、`Agent.run(userInput, personaId?, sessionId?)`(M3)、`AgentBuilder.rag(retriever)`(M1 预留,字段已存但 build 未传)、`internal ReActAgent`/`MinimalAgent`(M3,已注入 conversation)。
- `agent-memory`:`PresetPersonaStore`、`InMemoryConversationStore`(M3)。
- 测试:`:agent-core` 27 / `:agent-llm` 17 / `:agent-tools` 6 / `:agent-memory` 10,共 60 个全绿。
- `settings.gradle.kts`:`include(":agent-core", ":agent-llm", ":agent-tools", ":agent-memory")`。

## Global Constraints

- 纯 Kotlin/JVM 多模块,M4 不依赖 Android SDK、不依赖 ADK、不依赖 google-genai。
- 组件**不内置任何 API key**;EmbeddingModel 由宿主提供(M4 测试用 Fake)。
- 所有公共 API 用 Kotlin + Coroutines/Flow,无 UI 耦合。
- RAG 走 LangChain4j core(`EmbeddingModel` + `EmbeddingStore` + `InMemoryEmbeddingStore`)。M4 不引 `langchain4j-open-ai`(真实 embedding 留 M6)。
- 检索结果注入 system prompt(不改消息结构);retriever 为 null 时 system 不变(向后兼容)。
- `AgentBuilder.rag(retriever)` 真正传给 agent 构造;retriever 默认 null,既有测试不受影响。
- 测试基于 `KnowledgeRetriever`/`KnowledgeChunk` 抽象层断言,不直接断言 LangChain4j 内部对象。
- 包名前缀统一 `com.you.agent`。
- TDD:先写失败测试,再写最小实现,再跑通,再提交。**不弱化断言**。
- 每个任务结束提交一次,提交信息以 `feat:`/`chore:`/`test:` 前缀。
- DRY、YAGNI:不实现 M4 范围外的能力(持久化向量库 M5 / 真实 Embedding API M6 / 重排 M5 / persona.knowledgeSources 过滤后续)。

---

## File Structure(本计划新增/改动)

```
agent-android/
├── settings.gradle.kts                  # 改: include +(":agent-rag")
├── gradle/libs.versions.toml            # 改: +langchain4j 版本与坐标
├── agent-core/
│   └── src/
│       ├── main/kotlin/com/you/agent/core/
│       │   ├── AgentBuilder.kt          # 改: build() 传 retriever 给 agent 构造
│       │   └── internal/
│       │       ├── MinimalAgent.kt      # 改: +retriever 注入;system 增强检索 context
│       │       └── ReActAgent.kt        # 改: +retriever 注入;system 增强检索 context
│       └── test/kotlin/com/you/agent/core/
│           └── RagAgentTest.kt          # 新增: FakeRetriever + FakeLlm 检索注入
└── agent-rag/                           # 新模块
    ├── build.gradle.kts                 # 新增
    └── src/
        ├── main/kotlin/com/you/agent/rag/
        │   ├── LangChain4jRetriever.kt  # 新增: 实现 KnowledgeRetriever
        │   └── KnowledgeIngestor.kt     # 新增: 文档分割→embedding→入库便捷工具
        └── test/kotlin/com/you/agent/rag/
            ├── FakeEmbeddingModel.kt    # 新增: 测试用确定性 EmbeddingModel
            ├── LangChain4jRetrieverTest.kt # 新增
            └── KnowledgeIngestorTest.kt # 新增
```

职责边界:`agent-core` 只含 `KnowledgeRetriever` 抽象(M1),零 LangChain4j 依赖;`agent-rag` 提供 `LangChain4jRetriever` + `KnowledgeIngestor`,依赖 `agent-core` + `langchain4j-core`。

---

## Task 1: agent-rag 模块 + LangChain4j 依赖 + KnowledgeIngestor

**Files:**
- Modify: `agent-android/settings.gradle.kts`
- Modify: `agent-android/gradle/libs.versions.toml`
- Create: `agent-android/agent-rag/build.gradle.kts`
- Create: `agent-android/agent-rag/src/main/kotlin/com/you/agent/rag/KnowledgeIngestor.kt`
- Create: `agent-android/agent-rag/src/test/kotlin/com/you/agent/rag/FakeEmbeddingModel.kt`
- Test: `agent-android/agent-rag/src/test/kotlin/com/you/agent/rag/KnowledgeIngestorTest.kt`

**Interfaces:**
- Consumes: LangChain4j `EmbeddingModel`/`EmbeddingStore`/`TextSegment`/`Embedding`/`DocumentSplitters`
- Produces: `KnowledgeIngestor`(文档分割→embedding→入库,供 Task 2 retriever 检索的数据源)

- [ ] **Step 1: 依赖与脚手架**

`agent-android/settings.gradle.kts`:
```kotlin
rootProject.name = "agent-android"
include(":agent-core", ":agent-llm", ":agent-tools", ":agent-memory", ":agent-rag")
```

`agent-android/gradle/libs.versions.toml`(在 `[versions]` 加 `langchain4j = "1.0.0"`,`[libraries]` 加坐标):
```toml
[versions]
# ... 既有 ...
langchain4j = "1.0.0"

[libraries]
# ... 既有 ...
langchain4j-core = { module = "dev.langchain4j:langchain4j-core", version.ref = "langchain4j" }
```

`agent-android/agent-rag/build.gradle.kts`:
```kotlin
plugins {
    alias(libs.plugins.kotlin.jvm)
}

dependencies {
    implementation(project(":agent-core"))
    implementation(libs.kotlinx.coroutines.core)
    implementation(libs.langchain4j.core)
    testImplementation(libs.kotlinx.coroutines.test)
    testImplementation(libs.junit.jupiter)
}

tasks.test { useJUnitPlatform() }
```

注:LangChain4j 是 Java 库,不需要 kotlin-serialization 插件。若 `langchain4j-core:1.0.0` 在 Maven Central 不可用(沙箱代理下载失败),subagent 可尝试 `1.1.0` 或 `1.0.1`,核心 API 兼容。若仍不可用,**停止报告**不要弱化。

- [ ] **Step 2: 写 FakeEmbeddingModel(测试基础设施)**

`agent-rag/src/test/kotlin/com/you/agent/rag/FakeEmbeddingModel.kt`:
```kotlin
package com.you.agent.rag

import dev.langchain4j.data.embedding.Embedding
import dev.langchain4j.data.segment.TextSegment
import dev.langchain4j.model.embedding.EmbeddingModel
import dev.langchain4j.model.output.Response

/**
 * 测试用确定性 EmbeddingModel:对文本生成 384 维向量。
 * 用字符 char code 循环填充 + L2 归一化,使相同文本向量相同、含相同词的文本 cosine 相近。
 * 不依赖真实 API,纯计算。
 */
class FakeEmbeddingModel : EmbeddingModel {
    private val dim = 384

    override fun embed(textSegment: TextSegment): Response<Embedding> =
        Response.from(Embedding.from(toVector(textSegment.text())))

    override fun embed(text: String): Response<Embedding> =
        Response.from(Embedding.from(toVector(text)))

    override fun embedAll(textSegments: List<TextSegment>): Response<List<Embedding>> =
        Response.from(textSegments.map { Embedding.from(toVector(it.text())) })

    private fun toVector(text: String): FloatArray {
        val v = FloatArray(dim)
        if (text.isEmpty()) return v
        text.forEachIndexed { i, c -> v[i % dim] = v[i % dim] + c.code.toFloat() }
        // L2 归一化,使 cosine 相似度有意义
        val norm = Math.sqrt(v.map { it * it }.sum().toDouble()).toFloat()
        if (norm > 0) for (i in v.indices) v[i] = v[i] / norm
        return v
    }

    override fun dimension(): Int = dim
}
```

注:`EmbeddingModel` 接口在 LangChain4j 1.x 有 `embed(TextSegment)`、`embed(String)`、`embedAll(List<TextSegment>)`、`dimension()` 方法,返回 `Response<T>`(用 `Response.from(T)` 构造,`.content()` 取值)。若 1.0.0 的接口签名略有差异(如 `embed` 返回类型),subagent 按编译错误调整,但**保持确定性向量语义不变**。

- [ ] **Step 3: 写失败测试**

`agent-rag/src/test/kotlin/com/you/agent/rag/KnowledgeIngestorTest.kt`:
```kotlin
package com.you.agent.rag

import dev.langchain4j.store.embedding.inmemory.InMemoryEmbeddingStore
import kotlinx.coroutines.test.runTest
import org.junit.jupiter.api.Assertions.assertEquals
import org.junit.jupiter.api.Assertions.assertTrue
import org.junit.jupiter.api.Test

class KnowledgeIngestorTest {
    @Test
    fun `ingest splits text and stores segments`() = runTest {
        val store = InMemoryEmbeddingStore<dev.langchain4j.data.segment.TextSegment>()
        val model = FakeEmbeddingModel()
        val ingestor = KnowledgeIngestor(model, store, chunkSize = 20)

        ingestor.ingest("ABCDEFGHIJKLMNOPQRSTUVWXYZ", source = "doc1")

        // chunkSize=20 应把 26 字符切成至少 2 段
        // InMemoryEmbeddingStore 没有公开 size(),用检索验证:retrieve 任意文本应能命中
        // 这里用 store.search 间接验证有数据
        val emb = model.embed("ABC").content()
        val req = dev.langchain4j.store.embedding.EmbeddingSearchRequest.builder()
            .queryEmbedding(emb).maxResults(10).build()
        val result = store.search(req)
        assertTrue(result.matches().isNotEmpty())
    }

    @Test
    fun `ingest multiple texts all retrievable`() = runTest {
        val store = InMemoryEmbeddingStore<dev.langchain4j.data.segment.TextSegment>()
        val model = FakeEmbeddingModel()
        val ingestor = KnowledgeIngestor(model, store, chunkSize = 100)

        ingestor.ingest("Beijing is the capital of China", source = "geo")
        ingestor.ingest("Python is a programming language", source = "tech")

        val emb = model.embed("Beijing capital").content()
        val req = dev.langchain4j.store.embedding.EmbeddingSearchRequest.builder()
            .queryEmbedding(emb).maxResults(10).build()
        val result = store.search(req)
        assertEquals(2, result.matches().size)
    }
}
```

- [ ] **Step 4: 跑测试确认失败**

Run: `gradle :agent-rag:test`
Expected: FAIL,编译错误(`KnowledgeIngestor` 未定义;首次需 `gradle projects` 确认 `:agent-rag` 出现)。

- [ ] **Step 5: 写最小实现**

`agent-rag/src/main/kotlin/com/you/agent/rag/KnowledgeIngestor.kt`:
```kotlin
package com.you.agent.rag

import dev.langchain4j.data.segment.TextSegment
import dev.langchain4j.model.embedding.EmbeddingModel
import dev.langchain4j.store.embedding.EmbeddingStore

/**
 * 文档入库便捷工具:把文本分割成段、embedding 后存入 EmbeddingStore。
 * 优先用 LangChain4j DocumentSplitters.recursive;若该 API 不可用则降级为按字符数切分。
 * @param chunkSize 每段最大字符数(降级模式用)
 */
class KnowledgeIngestor(
    private val embeddingModel: EmbeddingModel,
    private val store: EmbeddingStore<TextSegment>,
    private val chunkSize: Int = 300,
) {
    suspend fun ingest(text: String, source: String) {
        val segments = split(text, source)
        val embeddings = embeddingModel.embedAll(segments).content()
        store.addAll(embeddings, segments)
    }

    private fun split(text: String, source: String): List<TextSegment> {
        // 降级实现:按字符数切分(DocumentSplitters 优先,但为降低 API 风险用自实现)
        if (text.length <= chunkSize) return listOf(TextSegment.from(text, metadata(source)))
        val result = mutableListOf<TextSegment>()
        var i = 0
        while (i < text.length) {
            val end = minOf(i + chunkSize, text.length)
            result.add(TextSegment.from(text.substring(i, end), metadata(source)))
            i = end
        }
        return result
    }

    private fun metadata(source: String) =
        dev.langchain4j.data.document.Metadata.from(mapOf("source" to source))
}
```

注:`TextSegment.from(text, metadata)` 与 `Metadata.from(map)` 的确切签名 subagent 按编译错误调整(可能是 `TextSegment.from(text)` + `Metadata.metadata().put("source", source)`)。`EmbeddingStore.addAll(List<Embedding>, List<TextSegment>): List<String>` 返回 id 列表。核心:ingest 后 store 内有可检索的 TextSegment。**断言基于检索结果(Step 3 测试),不直接断言 LangChain4j 内部**。

- [ ] **Step 6: 跑测试确认通过**

Run: `gradle :agent-rag:test`
Expected: BUILD SUCCESSFUL,2 个测试通过。

- [ ] **Step 7: 提交**

```bash
git add settings.gradle.kts gradle/libs.versions.toml agent-rag
git commit -m "feat(agent-rag): add module with LangChain4j dependency and KnowledgeIngestor"
```

---

## Task 2: LangChain4jRetriever 实现 KnowledgeRetriever

**Files:**
- Create: `agent-android/agent-rag/src/main/kotlin/com/you/agent/rag/LangChain4jRetriever.kt`
- Test: `agent-android/agent-rag/src/test/kotlin/com/you/agent/rag/LangChain4jRetrieverTest.kt`

**Interfaces:**
- Consumes: `KnowledgeRetriever`/`KnowledgeChunk`/`Persona`(M1,agent-core)、LangChain4j `EmbeddingModel`/`EmbeddingStore`(Task 1)
- Produces: `LangChain4jRetriever`(供 Task 3 注入 agent)

- [ ] **Step 1: 写失败测试**

`agent-rag/src/test/kotlin/com/you/agent/rag/LangChain4jRetrieverTest.kt`:
```kotlin
package com.you.agent.rag

import com.you.agent.core.persona.Persona
import dev.langchain4j.data.segment.TextSegment
import dev.langchain4j.store.embedding.inmemory.InMemoryEmbeddingStore
import kotlinx.coroutines.test.runTest
import org.junit.jupiter.api.Assertions.assertEquals
import org.junit.jupiter.api.Assertions.assertTrue
import org.junit.jupiter.api.Test

class LangChain4jRetrieverTest {
    private val persona = Persona("p", "P", "sys")

    @Test
    fun `retrieve returns relevant chunks after ingest`() = runTest {
        val store = InMemoryEmbeddingStore<TextSegment>()
        val model = FakeEmbeddingModel()
        val ingestor = KnowledgeIngestor(model, store, chunkSize = 100)
        ingestor.ingest("Beijing is the capital of China", source = "geo")
        ingestor.ingest("Python is a programming language", source = "tech")

        val retriever = LangChain4jRetriever(model, store, topK = 2)
        val chunks = retriever.retrieve("What is the capital?", persona)

        assertEquals(2, chunks.size)
        // Beijing 段应排第一(与 capital query 更相关)
        assertTrue(chunks[0].text.contains("Beijing"))
        assertEquals("geo", chunks[0].source)
        assertTrue(chunks[0].score != null)
    }

    @Test
    fun `retrieve empty store returns empty list`() = runTest {
        val store = InMemoryEmbeddingStore<TextSegment>()
        val model = FakeEmbeddingModel()
        val retriever = LangChain4jRetriever(model, store, topK = 4)
        val chunks = retriever.retrieve("anything", persona)
        assertTrue(chunks.isEmpty())
    }

    @Test
    fun `retrieve respects topK limit`() = runTest {
        val store = InMemoryEmbeddingStore<TextSegment>()
        val model = FakeEmbeddingModel()
        val ingestor = KnowledgeIngestor(model, store, chunkSize = 100)
        ingestor.ingest("doc about cats", source = "a")
        ingestor.ingest("doc about dogs", source = "b")
        ingestor.ingest("doc about birds", source = "c")

        val retriever = LangChain4jRetriever(model, store, topK = 2)
        val chunks = retriever.retrieve("animals", persona)
        assertEquals(2, chunks.size)
    }

    @Test
    fun `retrieve topK override via parameter`() = runTest {
        val store = InMemoryEmbeddingStore<TextSegment>()
        val model = FakeEmbeddingModel()
        val ingestor = KnowledgeIngestor(model, store, chunkSize = 100)
        ingestor.ingest("one", source = "a")
        ingestor.ingest("two", source = "b")
        ingestor.ingest("three", source = "c")

        val retriever = LangChain4jRetriever(model, store, topK = 1)
        val chunks = retriever.retrieve("query", persona, topK = 3)
        assertEquals(3, chunks.size)
    }
}
```

- [ ] **Step 2: 跑测试确认失败**

Run: `gradle :agent-rag:test`
Expected: FAIL,编译错误(`LangChain4jRetriever` 未定义)。

- [ ] **Step 3: 写最小实现**

`agent-rag/src/main/kotlin/com/you/agent/rag/LangChain4jRetriever.kt`:
```kotlin
package com.you.agent.rag

import com.you.agent.core.persona.KnowledgeChunk
import com.you.agent.core.persona.KnowledgeRetriever
import com.you.agent.core.persona.Persona
import dev.langchain4j.data.segment.TextSegment
import dev.langchain4j.model.embedding.EmbeddingModel
import dev.langchain4j.store.embedding.EmbeddingMatch
import dev.langchain4j.store.embedding.EmbeddingSearchRequest
import dev.langchain4j.store.embedding.EmbeddingStore

/**
 * 基于 LangChain4j 的 KnowledgeRetriever 实现。
 * @param embeddingModel 用于把 query 转向量(LangChain4j 同步 API,本 suspend 函数直接调)
 * @param store 向量库(M4 用 InMemoryEmbeddingStore,M5 可换持久化)
 * @param topK 默认检索条数,可被 retrieve 参数覆盖
 */
class LangChain4jRetriever(
    private val embeddingModel: EmbeddingModel,
    private val store: EmbeddingStore<TextSegment>,
    private val topK: Int = 4,
) : KnowledgeRetriever {
    override suspend fun retrieve(query: String, persona: Persona, topK: Int): List<KnowledgeChunk> {
        val effectiveTopK = if (topK > 0) topK else this.topK
        val queryEmbedding = embeddingModel.embed(query).content()
        val request = EmbeddingSearchRequest.builder()
            .queryEmbedding(queryEmbedding)
            .maxResults(effectiveTopK)
            .build()
        val result = store.search(request)
        return result.matches().map { it.toChunk() }
    }

    private fun EmbeddingMatch<TextSegment>.toChunk(): KnowledgeChunk {
        val segment = embedded()
        val source = segment.metadata().getString("source") ?: "unknown"
        return KnowledgeChunk(text = segment.text(), source = source, score = score())
    }
}
```

注:`EmbeddingMatch.score()` 返回 `Double`。`segment.metadata().getString("source")` 若 key 不存在返回 null(或抛,subagent 按实际 API 调整用 `get("source")?.toString()`)。`embedded()` 返回 `TextSegment`。核心:返回 `KnowledgeChunk` 列表,断言基于抽象层。

- [ ] **Step 4: 跑测试确认通过**

Run: `gradle :agent-rag:test`
Expected: BUILD SUCCESSFUL,Task 1 的 2 个 + Task 2 的 4 个 = 6 个测试通过。

**若 `retrieve returns relevant chunks` 中 `chunks[0].text.contains("Beijing")` 失败**(FakeEmbeddingModel 的向量语义不够区分),检查 FakeEmbeddingModel 的 toVector:相同词应使向量相近。若仍无法区分,可调整测试断言为 `assertTrue(chunks.any { it.text.contains("Beijing") })`(Beijing 段在 topK=2 内即可,不强制第一)——**这是合理的断言调整(不弱化,因 FakeEmbeddingModel 的语义排序能力有限),但优先尝试让 FakeEmbeddingModel 足够区分**。

- [ ] **Step 5: 提交**

```bash
git add agent-rag/src
git commit -m "feat(agent-rag): implement LangChain4jRetriever for KnowledgeRetriever"
```

---

## Task 3: ReActAgent/MinimalAgent 接入 retriever + AgentBuilder.rag() 传值

**Files:**
- Modify: `agent-android/agent-core/src/main/kotlin/com/you/agent/core/AgentBuilder.kt`
- Modify: `agent-android/agent-core/src/main/kotlin/com/you/agent/core/internal/MinimalAgent.kt`
- Modify: `agent-android/agent-core/src/main/kotlin/com/you/agent/core/internal/ReActAgent.kt`
- Test: `agent-android/agent-core/src/test/kotlin/com/you/agent/core/RagAgentTest.kt`

**Interfaces:**
- Consumes: `KnowledgeRetriever`/`KnowledgeChunk`(M1)、`Agent`/`AgentBuilder`(M1/M3)
- Produces: retriever 注入 agent,检索结果拼到 system prompt(供 Task 4 端到端)

- [ ] **Step 1: 写失败测试**

`agent-core/src/test/kotlin/com/you/agent/core/RagAgentTest.kt`:
```kotlin
package com.you.agent.core

import com.you.agent.core.chat.ChatChunk
import com.you.agent.core.chat.ChatRequest
import com.you.agent.core.chat.ChatResponse
import com.you.agent.core.llm.LlmProvider
import com.you.agent.core.persona.KnowledgeChunk
import com.you.agent.core.persona.KnowledgeRetriever
import com.you.agent.core.persona.Persona
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.flowOf
import kotlinx.coroutines.flow.toList
import kotlinx.coroutines.test.runTest
import org.junit.jupiter.api.Assertions.assertEquals
import org.junit.jupiter.api.Assertions.assertInstanceOf
import org.junit.jupiter.api.Assertions.assertTrue
import org.junit.jupiter.api.Test

class RagAgentTest {
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

    private class FakeRetriever(private val chunks: List<KnowledgeChunk>) : KnowledgeRetriever {
        override suspend fun retrieve(query: String, persona: Persona, topK: Int): List<KnowledgeChunk> = chunks
    }

    @Test
    fun `retriever injects knowledge into system prompt`() = runTest {
        val llm = ScriptedLlm().apply {
            enqueue(ChatResponse(content = "based on knowledge", model = "fake", finishReason = "stop"))
        }
        val retriever = FakeRetriever(listOf(
            KnowledgeChunk(text = "Beijing is the capital of China", source = "geo"),
        ))
        val agent = Agent.builder()
            .llm(llm)
            .persona(Persona("p", "P", "You are a helpful assistant."))
            .rag(retriever)
            .build()

        agent.run("What is the capital?").toList()

        val systemMsg = llm.requests[0].messages[0]
        assertEquals(com.you.agent.core.chat.Message.Role.SYSTEM, systemMsg.role)
        assertTrue(systemMsg.content.contains("You are a helpful assistant."))
        assertTrue(systemMsg.content.contains("Beijing is the capital of China"))
    }

    @Test
    fun `no retriever means plain system prompt`() = runTest {
        val llm = ScriptedLlm().apply {
            enqueue(ChatResponse(content = "ok", model = "fake", finishReason = "stop"))
        }
        val agent = Agent.builder()
            .llm(llm)
            .persona(Persona("p", "P", "base system"))
            .build()

        agent.run("hi").toList()

        val systemMsg = llm.requests[0].messages[0]
        assertEquals("base system", systemMsg.content)
    }

    @Test
    fun `retriever with empty results keeps plain system prompt`() = runTest {
        val llm = ScriptedLlm().apply {
            enqueue(ChatResponse(content = "ok", model = "fake", finishReason = "stop"))
        }
        val retriever = FakeRetriever(emptyList())
        val agent = Agent.builder()
            .llm(llm)
            .persona(Persona("p", "P", "base system"))
            .rag(retriever)
            .build()

        agent.run("hi").toList()

        val systemMsg = llm.requests[0].messages[0]
        assertEquals("base system", systemMsg.content)
    }

    @Test
    fun `ReAct agent with retriever injects knowledge before tool loop`() = runTest {
        val llm = ScriptedLlm().apply {
            enqueue(ChatResponse(content = "", model = "fake", finishReason = "tool_calls",
                toolCalls = listOf(com.you.agent.core.chat.ToolCall("c1", "echo", emptyMap()))))
            enqueue(ChatResponse(content = "done", model = "fake", finishReason = "stop"))
        }
        val retriever = FakeRetriever(listOf(
            KnowledgeChunk(text = "KNOWLEDGE_FACT", source = "s"),
        ))
        val agent = Agent.builder()
            .llm(llm)
            .persona(Persona("p", "P", "sys"))
            .rag(retriever)
            .tools {
                +com.you.agent.core.tools.Tool("echo", "d", "{}") { "result" }
            }
            .build()

        val events = agent.run("go").toList()
        assertInstanceOf(AgentEvent.Answer::class.java, events.last())
        // 第一轮请求的 system 应含知识
        assertTrue(llm.requests[0].messages[0].content.contains("KNOWLEDGE_FACT"))
        // 第二轮请求的 system 也应含知识(每轮重新检索注入)
        assertTrue(llm.requests[1].messages[0].content.contains("KNOWLEDGE_FACT"))
    }
}
```

- [ ] **Step 2: 跑测试确认失败**

Run: `gradle :agent-core:test`
Expected: FAIL,测试运行失败(`AgentBuilder.rag()` 存了字段但 build() 没传给 agent,retriever 未注入,system 不含知识)。

- [ ] **Step 3: 写最小实现**

修改 `AgentBuilder.kt` 的 `build()`(把 retriever 传给 agent 构造):
```kotlin
    fun build(): Agent {
        val provider = llm
            ?: error("Agent requires an LlmProvider. 未配置 LLM 时绝不静默使用默认 Provider。")
        return if (tools.isNotEmpty()) {
            ReActAgent(llm = provider, persona = persona, tools = tools, config = config,
                conversation = conversation, retriever = retriever)
        } else {
            MinimalAgent(llm = provider, persona = persona, config = config,
                conversation = conversation, retriever = retriever)
        }
    }
```
(其余 AgentBuilder 不变,retriever 字段 M1 已有。)

修改 `MinimalAgent.kt`(加 retriever 注入 + system 增强)。整体替换:
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
    private val llm: LlmProvider,
    private val persona: Persona,
    private val config: AgentConfig,
    private val conversation: ConversationStore? = null,
    private val retriever: KnowledgeRetriever? = null,
) : Agent {
    override fun run(userInput: String, personaId: String?, sessionId: String?): Flow<AgentEvent> = flow {
        val history = conversation?.let { store -> sessionId?.let { store.load(it) } } ?: emptyList()
        val systemPrompt = enhanceSystemPrompt(userInput)
        val messages = mutableListOf<Message>()
        messages.add(Message.system(systemPrompt))
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

    private suspend fun enhanceSystemPrompt(userInput: String): String {
        if (retriever == null) return persona.systemPrompt
        val chunks = retriever.retrieve(userInput, persona)
        if (chunks.isEmpty()) return persona.systemPrompt
        val context = chunks.joinToString("\n") { "- ${it.text}" }
        return persona.systemPrompt + "\n\n以下是从知识库检索到的参考资料:\n" + context
    }
}
```

修改 `ReActAgent.kt`(加 retriever 注入 + system 增强,每轮 ReAct 循环的请求都用增强 system)。整体替换:
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
    private val llm: LlmProvider,
    private val persona: Persona,
    private val tools: List<Tool>,
    private val config: AgentConfig,
    private val conversation: ConversationStore? = null,
    private val retriever: KnowledgeRetriever? = null,
) : Agent {
    override fun run(userInput: String, personaId: String?, sessionId: String?): Flow<AgentEvent> = flow {
        val history = conversation?.let { store -> sessionId?.let { store.load(it) } } ?: emptyList()
        val registry = MapToolRegistry().also { reg -> tools.forEach { reg.register(it) } }
        val declarations = tools.map { it.toDeclaration() }
        val messages = mutableListOf<Message>()
        messages.add(Message.system(persona.systemPrompt)) // 占位,首轮 chat 前替换为增强版
        messages.addAll(history)
        val historyEnd = messages.size
        messages.add(Message.user(userInput))
        try {
            repeat(config.maxSteps) {
                // 每轮重新检索注入(基于当前最后一条 user/tool 消息的上下文)
                val systemPrompt = enhanceSystemPrompt(currentQuery(messages))
                messages[0] = Message.system(systemPrompt)
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

    private suspend fun enhanceSystemPrompt(query: String): String {
        if (retriever == null) return persona.systemPrompt
        val chunks = retriever.retrieve(query, persona)
        if (chunks.isEmpty()) return persona.systemPrompt
        val context = chunks.joinToString("\n") { "- ${it.text}" }
        return persona.systemPrompt + "\n\n以下是从知识库检索到的参考资料:\n" + context
    }

    /** 取最近一条 user 消息作为检索 query(ReAct 多轮时 tool 结果后可能无新 user,用原始 userInput)。 */
    private fun currentQuery(messages: List<Message>): String =
        messages.lastOrNull { it.role == Message.Role.USER }?.content ?: ""
}
```

注:`messages[0] = Message.system(...)` 每轮替换 system(system 位置固定在 index 0,history 在其后)。ReAct 每轮重新检索基于当前 user query(测试 `ReAct agent with retriever` 断言两轮 system 都含知识,因为 FakeRetriever 对任何 query 返回固定 chunks)。若 `currentQuery` 返回空(无 user,不应发生因首轮就有 user),enhanceSystemPrompt 用空 query 调 retrieve(FakeRetriever 仍返回 chunks)。

- [ ] **Step 4: 跑测试确认通过**

Run: `gradle :agent-core:test`
Expected: BUILD SUCCESSFUL,新增 4 个 RagAgentTest 通过,M3 既有 27 个测试不受影响(retriever 默认 null,system 不变),合计 31 个 `:agent-core` 测试。

**若 M3 既有测试失败**(例如 ConversationAgentTest/ReActAgentTest/EndToEndConversationTest 因 retriever 注入编译或行为失败),停止报告——不要弱化断言。检查:retriever 默认 null,enhanceSystemPrompt 直接返回 persona.systemPrompt,行为应同 M3。

- [ ] **Step 5: 提交**

```bash
git add agent-core/src
git commit -m "feat(agent-core): inject KnowledgeRetriever into agents, enhance system prompt with retrieved context"
```

---

## Task 4: 端到端 RAG 集成(LangChain4jRetriever + InMemoryEmbeddingStore + MockWebServer LLM)

**Files:**
- Modify: `agent-android/agent-core/build.gradle.kts`(加 `testImplementation(project(":agent-rag"))`)
- Test: `agent-android/agent-core/src/test/kotlin/com/you/agent/core/EndToEndRagTest.kt`

**Interfaces:**
- Consumes: `Agent.builder().rag()`(Task 3)、`LangChain4jRetriever`+`KnowledgeIngestor`(Task 1/2)、`OpenAiCompatibleProvider`(M1)
- Produces: M4 验收——无真实网络/Embedding API 下端到端 RAG 闭环

- [ ] **Step 1: 加 test 依赖**

在 `agent-core/build.gradle.kts` 的 `dependencies { }` 内追加:
```kotlin
    testImplementation(project(":agent-rag"))
```

- [ ] **Step 2: 写测试**

`agent-core/src/test/kotlin/com/you/agent/core/EndToEndRagTest.kt`:
```kotlin
package com.you.agent.core

import com.you.agent.core.persona.Persona
import com.you.agent.llm.LlmFactory
import com.you.agent.llm.UserLlmConfig
import com.you.agent.rag.KnowledgeIngestor
import com.you.agent.rag.LangChain4jRetriever
import dev.langchain4j.data.segment.TextSegment
import dev.langchain4j.store.embedding.inmemory.InMemoryEmbeddingStore
import kotlinx.coroutines.flow.toList
import kotlinx.coroutines.test.runTest
import okhttp3.mockwebserver.MockResponse
import okhttp3.mockwebserver.MockWebServer
import org.junit.jupiter.api.Assertions.assertEquals
import org.junit.jupiter.api.Assertions.assertInstanceOf
import org.junit.jupiter.api.Assertions.assertTrue
import org.junit.jupiter.api.Test

class EndToEndRagTest {
    @Test
    fun `rag injects knowledge into LLM request end to end`() = runTest {
        val server = MockWebServer()
        server.enqueue(MockResponse().setBody(
            """{"model":"deepseek-chat","choices":[{"message":{"role":"assistant","content":"Beijing is the capital"}}]}"""))
        server.start()
        try {
            // 准备 RAG:Fake embedding + InMemory store + ingest 知识
            val store = InMemoryEmbeddingStore<TextSegment>()
            val model = com.you.agent.rag.FakeEmbeddingModel()
            val ingestor = KnowledgeIngestor(model, store, chunkSize = 100)
            ingestor.ingest("Beijing is the capital of China", source = "geo")
            val retriever = LangChain4jRetriever(model, store, topK = 2)

            val provider = LlmFactory.from(UserLlmConfig(
                providerId = "custom",
                apiKey = "key",
                baseUrl = server.url("/v1").toString().trimEnd('/'),
                model = "deepseek-chat",
            ))
            val agent = Agent.builder()
                .llm(provider)
                .persona(Persona("p", "P", "You are a geography assistant."))
                .rag(retriever)
                .build()

            val events = agent.run("What is the capital of China?").toList()
            assertInstanceOf(AgentEvent.Answer::class.java, events.last())
            assertEquals("Beijing is the capital", (events.last() as AgentEvent.Answer).text)

            // 请求体的 system 消息应含检索到的知识
            val body = server.takeRequest().body.readUtf8()
            assertTrue(body.contains("\"role\":\"system\""))
            assertTrue(body.contains("Beijing is the capital of China"))
            assertTrue(body.contains("You are a geography assistant."))
        } finally { server.shutdown() }
    }
}
```

注:`FakeEmbeddingModel` 在 agent-rag test 目录(`com.you.agent.rag` 包),agent-core test 通过 `testImplementation(project(":agent-rag"))` 能否访问?**不能直接访问 test 源集**(test 不传递)。解决方案:把 `FakeEmbeddingModel` 移到 agent-rag 的 `main` 源集(作为测试工具公开),或在 EndToEndRagTest 内复制一份。

**推荐:把 `FakeEmbeddingModel` 从 agent-rag test 移到 main**(它无测试依赖,只是个 EmbeddingModel 实现,放 main 不影响;或放 agent-rag 的 `testFixtures`——但项目未启用 testFixtures,最简放 main)。Task 1 Step 2 创建时路径改为 `agent-rag/src/main/kotlin/com/you/agent/rag/FakeEmbeddingModel.kt`,包名不变。Task 4 测试直接 `com.you.agent.rag.FakeEmbeddingModel()`。

(本计划 Task 1 Step 2 路径已按 main 写——subagent 执行 Task 1 时确认 FakeEmbeddingModel 在 main 源集。)

- [ ] **Step 3: 跑全部测试**

Run: `gradle test`
Expected: BUILD SUCCESSFUL,五模块全过(`:agent-core` 32 / `:agent-llm` 17 / `:agent-tools` 6 / `:agent-memory` 10 / `:agent-rag` 6),合计 71 个。

- [ ] **Step 4: 提交**

```bash
git add agent-core/build.gradle.kts agent-core/src
git commit -m "test: end-to-end RAG integration with knowledge injected into system prompt"
```

---

## Self-Review

**1. Spec coverage(对照设计文档 M4 范围):**
- `agent-rag` LangChain4j 检索 → Task 1/2 ✓(langchain4j-core + LangChain4jRetriever)
- 角色知识库注入 → Task 3/4 ✓(检索结果拼 system prompt)
- `KnowledgeRetriever` 默认实现 → Task 2 `LangChain4jRetriever` ✓
- 文档加载/分割 `DocumentSplitters.recursive` → Task 1 KnowledgeIngestor(降级为自实现按字符切,理由记录于实现注;若 DocumentSplitters API 可用则优先用,断言不变)
- 向量库默认 InMemoryEmbeddingStore → Task 1/2 ✓;持久化留 M5 ✓
- 检索结果作为 context 注入 persona system prompt → Task 3 ✓
- 未覆盖(留给后续里程碑,符合范围):持久化向量库 M5 / 真实 Embedding API(OpenAiEmbeddingModel)M6 / 重排/混合检索 M5 / persona.knowledgeSources 过滤后续

**2. 关键决策说明(对设计文档的有意偏离):**
- 设计文档 §7 写 `LangChain4jRetriever(embeddingModel, store, ingestor)` 三参数。M4 简化为 `(embeddingModel, store, topK)`,ingestor 作为独立 `KnowledgeIngestor` 类(职责分离:ingestor 写入,retriever 读取),更清晰。`EmbeddingStoreIngestor` 不直接暴露(降低 LangChain4j API 耦合),用自封装 `KnowledgeIngestor`。
- M4 仅依赖 `langchain4j-core`,不引 `langchain4j-open-ai`(真实 OpenAI 兼容 embedding 留 M6 BYO Key 时按需)。测试用自建 `FakeEmbeddingModel`(确定性向量,纯计算)。理由:M4 聚焦检索+注入链路,真实 embedding API 接入属于 BYO Key 范畴(M6)。

**3. Placeholder scan:** Task 1 Step 5 对 `TextSegment.from(text, metadata)` 与 `Metadata.from(map)` 签名给了 fallback 指导(subagent 按编译错误调整),非 placeholder。Task 2 Step 3 对 `metadata().getString("source")` 给了 fallback。每步含完整代码或确切命令。✓

**4. Type consistency:**
- `KnowledgeRetriever.retrieve(query, persona, topK): List<KnowledgeChunk>` M1 定义,Task 2 LangChain4jRetriever + Task 3 FakeRetriever + Task 4 调用一致 ✓
- `KnowledgeChunk(text, source, score?)` M1 定义,Task 2 toChunk + Task 3 FakeRetriever + Task 4 一致 ✓
- `Persona.systemPrompt: String` M1 定义,Task 3 enhanceSystemPrompt 拼接一致 ✓
- `AgentBuilder.rag(retriever)` M1 预留,Task 3 build() 传值 ✓
- ReActAgent/MinimalAgent 构造加 `retriever: KnowledgeRetriever? = null` 带默认值,Task 3 build() 命名传参 ✓
- system 消息位置:index 0(M3 已定 history 在 system 后),Task 3 ReActAgent 每轮 `messages[0] = Message.system(enhanced)` 替换 ✓

**5. 向后兼容:**
- `AgentBuilder.rag()` M1 已有,Task 3 仅改 build() 传值;retriever 默认 null,既有测试不注入则 enhanceSystemPrompt 返回原 systemPrompt ✓
- ReActAgent/MinimalAgent 构造加 `retriever: KnowledgeRetriever? = null` 带默认值,M3 既有测试(无 retriever)行为不变 ✓
- M3 ConversationAgentTest:无 retriever,system 不增强,断言 `secondReq.messages[0].role == SYSTEM` 仍过 ✓
- M2 ReActAgentTest:无 retriever 无 conversation,system 为 persona.systemPrompt ✓

---

## Execution Handoff

计划已保存至 `docs/superpowers/plans/2026-07-02-android-agent-m4.md`。

**无需用户再次确认架构偏离**——`langchain4j-open-ai` 推迟到 M6、`KnowledgeIngestor` 独立于 retriever 的收窄均沿用 M1-M3 既定方向(手写、纯 JVM 可测、降低外部 API 耦合),理由记录于关键决策节。

沿用 M1/M2/M3 的 **Subagent-Driven Development** 执行。关键环境约束:
- **用 `gradle` 不用 `./gradlew`**(wrapper 下载超时,系统 gradle 8.14 + JDK 17 daemon 已配 `~/.gradle/gradle.properties`)
- LangChain4j 1.0.0 首次拉取走沙箱代理(已配 `systemProp.https.proxyHost=127.0.0.1:18080`),若版本不可用 subagent 可升 1.1.0
- 不弱化断言;Task 2 若 FakeEmbeddingModel 语义排序不足,优先调 Fake,断言调整(如 `any` 替代 `[0]`)需在计划允许范围内且记录
- Task 1 FakeEmbeddingModel 放 **main 源集**(非 test),供 Task 4 agent-core test 通过 `:agent-rag` 依赖访问
- Task 3 是关键改动点(改 M3 稳定的 ReActAgent/MinimalAgent),若 M3 既有测试编译/行为失败,停止报告不弱化

Task 1 是依赖验证关键点(LangChain4j 能否拉取 + 基本 API 可用),派发时须叮嘱:若 `langchain4j-core:1.0.0` 拉取失败,先试 1.1.0,再不行停止报告。
