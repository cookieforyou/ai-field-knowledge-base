# DeepResearch 多 Agent 行业深度研究助手 - 需求分析与技术实现报告

**（基于 Spring AI 2.0 + Spring Boot 4.x 技术栈重构版）**

---

## 一、 项目概述与核心价值

### 1.1 产品定位

本项目是一款面向企业级深度研究场景的 **AI 多智能体（Multi-Agent）系统**，定位为“AI 研究员助手”。旨在解决传统单轮对话 Chatbot 无法处理的**信息分散、需交叉验证、逻辑复杂、需引用溯源**等深度研究痛点。

### 1.2 核心能力矩阵

| 能力维度 | 传统 Chatbot | DeepResearch Multi-Agent |
| :--- | :--- | :--- |
| **信息来源** | 静态训练数据 | 实时网络检索 + 企业内部知识库 (RAG) |
| **输出深度** | 简短回复 | 3000+ 字结构化深度研报 |
| **引用溯源** | 无/易幻觉 | 精确到 Source ID，自动校验合法性 |
| **冲突检测** | 无 | 自动识别多源信息矛盾并标记 |
| **迭代优化** | 单轮 | 多轮反思补搜，直到证据完备 |
| **记忆继承** | 单会话 | 跨会话长期记忆 + 用户画像 |

---

## 二、 业务功能需求详述

### 2.1 核心工作流 (7步研究法)

1. **意图识别 (Intent Recognition)**：判断是简单问答（Direct）还是深度研究（Multi-Agent）。
2. **任务规划 (Task Planning)**：将Query拆解为子问题、报告大纲、搜索计划及资源预算。
3. **双源并行检索 (Dual-Source Retrieval)**：
    - **Web Scout**：调用外部搜索API，过滤、打分、结构化网页证据。
    - **Local RAG Scout**：检索本地向量数据库，提取企业内部知识库证据。
4. **证据裁判 (Evidence Audit)**：对双源证据进行可信度评分、去重、冲突检测，构建统一来源索引。
5. **分析归纳 (Analysis)**：基于证据池形成结论（Findings），评估证据完备性，识别信息缺口。
6. **反思补搜 (Reflect)**：*(条件触发)* 若证据不足且未达最大迭代次数，生成新搜索词重新检索。
7. **报告撰写 (Write)**：整合结论，生成带合法引用标记的长篇 Markdown 深度研报。

### 2.2 Agent 角色与温度设定

| Agent 角色 | 核心职责 | Temperature | 设计理由 |
| :--- | :--- | :--- | :--- |
| **Intent Router** | 路由分发 (Direct/Multi-Agent) | 0.0 | 意图判断需绝对确定 |
| **Planner** | 任务拆解、大纲与搜索计划生成 | 0.3 | 需一定创意，但不能太发散 |
| **Web Scout** | 网络取证、相关性过滤、Source ID分配 | 0.4 | 平衡覆盖率与相关性 |
| **Local Scout** | 本地知识库取证、相关性过滤 | 0.4 | 同上 |
| **Evidence Judge** | 证据评分、去重、冲突检测 | 0.2 | 评分需严格标准 |
| **Analyst** | 形成结论、评估完备性、识别缺口 | 0.3 | 逻辑严谨 |
| **Reflect** | 生成补搜查询、避免重复 | 0.3 | 针对性强 |
| **Writer** | 撰写3000+字深度研报、合法引用 | 0.4 | 需文采和流畅度 |

---

## 三、 技术架构与实现方案 (基于 Spring AI 2.0 重构)

> **核心技术栈**：Spring Boot 4.x + Spring Framework 7.x + **Spring AI 2.0 (GA)** + Spring WebFlux + Spring Data JPA/Redis + Milvus/PgVector。

### 3.1 整体架构演进

原 Python + FastAPI + LangGraph 架构将全面迁移至 Java 生态，利用 Spring AI 2.0 提供的原生 Agent 编排能力和 Spring Framework 7.x 的虚拟线程（Virtual Threads）特性，实现高并发、低延迟的企业级服务。

```text
┌─────────────────────────────────────────────────────────────────────────────┐
 │                         接口层 (Spring WebFlux / SSE)                        │
 │  ┌──────────────┐  ┌──────────────────────┐  ┌──────────────────────┐       │
 │  │  REST API    │  │  SSE 实时流式推送     │  │  WebSocket 双向通信   │       │
 │  └──────────────┘  └──────────────────────┘  └──────────────────────┘       │
 └─────────────────────────────────────────────────────────────────────────────┘
                                       │ (Virtual Threads 协程调度)
                                       ▼
 ┌─────────────────────────────────────────────────────────────────────────────┐
 │                  编排层 (Spring AI 2.0 Workflow / StateMachine)              │
 │                                                                              │
 │   ┌─────────┐  ┌─────────┐  ┌───────┐  ┌─────────┐  ┌─────────┐            │
 │   │  START  │─▶│ intent  │─▶│ plan  │─▶│web/local│─▶│deep_dive│            │
 │   └─────────┘  └────┬────┘  └───┬───┘  └────┬────┘  └────┬────┘            │
 │                      │          │           │            ▼                 │
 │                 ┌────┘          │           │       ┌─────────┐             │
 │                 ▼               │           │       │ analyze │             │
 │           ┌──────────┐          │           │       └────┬────┘             │
 │           │direct_ans│          │           │       ┌────┴────┐             │
 │           └────┬─────┘          │           │       │reflect? │             │
 │                │                │           │       └────┬────┘             │
 │                └───────────────▶│◀──────────┘            │                 │
 │                                 ▼                        │                 │
 │                           ┌─────────┐◀───────────────────┘                 │
 │                           │  write  │──▶ END                               │
 │                           └─────────┘                                      │
 └─────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       ▼
 ┌─────────────────────────────────────────────────────────────────────────────┐
 │               智能体层 (Spring AI 2.0 ChatClient + Advisors)                 │
 │   [IntentRouter] [Planner] [WebScout] [LocalScout] [Judge] [Analyst] [Writer]│
 └─────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       ▼
 ┌─────────────────────────────────────────────────────────────────────────────┐
 │                           服务与基础设施层                                    │
 │  [Bocha Search API] [Milvus/PgVector] [PostgreSQL] [Redis] [Micrometer]     │
 │  [Spring AI Memory (Short/Long/Semantic)] [Spring AI RAG (ETL/Retrieval)]   │
 └─────────────────────────────────────────────────────────────────────────────┘
```

### 3.2 核心模块 Spring AI 2.0 映射方案

#### 3.2.1 Agent 构建 (ChatClient + System Prompt)

在 Spring AI 2.0 中，使用 `ChatClient` 替代原有的 `create_agent`，通过 `defaultSystem` 注入 Prompt，通过 `defaultOptions` 控制 Temperature。

```java
@Configuration
public class AgentConfig {
    
    @Bean
    public ChatClient intentRouterClient(ChatModel chatModel) {
        return ChatClient.builder(chatModel)
                .defaultSystem(Prompts.INTENT_ROUTER)
                .defaultOptions(OpenAiChatOptions.builder().temperature(0.0).build())
                .build();
    }

    @Bean
    public ChatClient plannerClient(ChatModel chatModel) {
        return ChatClient.builder(chatModel)
                .defaultSystem(Prompts.PLANNER)
                .defaultOptions(OpenAiChatOptions.builder().temperature(0.3).build())
                .build();
    }
    // ... 其他 Agent Client 同理
}
```

#### 3.2.2 状态机与工作流编排 (替代 LangGraph)

**方案选择**：Spring AI 2.0 提供了基础的 Advisor 链，但对于复杂的条件分支和循环（如 Reflect 补搜），建议使用 **Spring StateMachine** 或 **Temporal/Camunda** 工作流引擎，或者利用 Spring AI 2.0 新增的 **`Workflow` API**（若已GA）结合 Java 21+ 的 `StructuredTaskScope` (虚拟线程) 实现并行编排。

**状态定义 (ResearchState)**：使用 Java `Record` 或不可变对象结合 `Builder` 模式实现类型安全的状态传递。

```java
public record ResearchState(
    String query,
    String userId,
    String tenantId,
    String memoryContext,
    List<Message> messages,
    String intent,
    List<String> subQuestions,
    List<SearchPlan> searchPlan,
    List<Evidence> webEvidence,
    List<Evidence> localEvidence,
    List<Evidence> evidencePool,
    List<AuditFlag> auditFlags,
    List<Finding> findings,
    boolean needsMoreResearch,
    List<String> missingGaps,
    int iteration,
    int maxIterations,
    String finalReport
) {
    // 提供 withXxx 方法实现状态的不可变更新
}
```

#### 3.2.3 双源并行检索 (Virtual Threads + CompletableFuture)

利用 Spring Framework 7.x 默认开启的虚拟线程，实现 Web 和 Local 检索的真正并行，大幅提升响应速度。

```java
@Service
public class DualSourceRetrievalService {
    
    // 使用虚拟线程执行器
    private final ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor();

    public ResearchState retrieve(ResearchState state) {
        CompletableFuture<List<Evidence>> webFuture = CompletableFuture.supplyAsync(
            () -> webScoutAgent.search(state), executor);
            
        CompletableFuture<List<Evidence>> localFuture = CompletableFuture.supplyAsync(
            () -> localScoutAgent.search(state), executor);
            
        // 等待两者完成
        CompletableFuture.allOf(webFuture, localFuture).join();
        
        return state.withWebEvidence(webFuture.join())
                    .withLocalEvidence(localFuture.join());
    }
}
```

#### 3.2.4 记忆系统 (Spring AI Memory)

Spring AI 2.0 提供了完善的 `ChatMemory` 抽象。
- **短期记忆 (Short-Term)**：使用 `MessageWindowChatMemory` 或 `VectorStoreChatMemory`，基于 Redis 存储会话上下文。
- **长期记忆 (Long-Term)**：结合 PostgreSQL 存储用户画像、历史偏好，通过 Spring AI 的 `Advisor` 机制在每次请求前自动注入上下文。
- **语义记忆 (Semantic)**：利用 Milvus/PgVector 存储历史研究成果，通过向量相似度检索相关记忆。

#### 3.2.5 RAG 与向量检索 (Spring AI RAG)

使用 Spring AI 2.0 的 `DocumentReader`, `DocumentTransformer`, `DocumentWriter` 构建 ETL 管道，使用 `VectorStore` (Milvus/PgVector) 进行检索。

```java
@Service
public class LocalRagScout {
    private final VectorStore vectorStore;
    private final ChatClient localScoutClient;

    public List<Evidence> search(ResearchState state) {
        // 1. 向量检索
        List<Document> docs = vectorStore.similaritySearch(
            SearchRequest.builder()
                .query(state.query())
                .topK(4)
                .similarityThreshold(0.7)
                .build()
        );
        // 2. 转换为 Evidence 并分配 Source ID
        // 3. 调用 LLM 进行相关性过滤和结构化
    }
}
```

---

## 四、 关键设计模式与工程实践

### 4.1 JSON 安全解析与 Fallback 机制

大模型输出 JSON 经常带有 Markdown 代码块标记或格式错误。必须在 Java 层做 robust 的解析和兜底。
- **实践**：使用 Jackson 或 Gson，结合正则表达式提取 `{...}` 块。每个 Agent 节点必须提供 `Fallback` 默认对象，确保 LLM 解析失败时工作流不崩溃。
- **Spring AI 2.0 特性**：使用 `BeanOutputConverter<T>` 结合 Schema 生成，强制 LLM 输出符合 Java Record/Class 结构的 JSON。

### 4.2 智能引用验证系统 (Citation Validation)

**痛点**：LLM 经常幻觉出不存在的引用 ID（如 `[WEB99_1-1]`）。
**实现**：在 Writer 节点输出后，使用正则 `$$(WEB|LOC)\d+_\d+-\d+$$` 提取所有引用，与 `source_index` 中的合法 ID 集合比对，**自动剔除非法引用**，并在文末动态渲染真实的参考资料列表。

### 4.3 证据评分算法 (Rule + LLM Hybrid)

不完全依赖 LLM 评分，采用**规则引擎前置打分 + LLM 微调**：
- 本地知识库：0.92
- `.gov` / `.edu` / 官方域名：0.88
- 主流媒体 (news, reuters, bloomberg)：0.72
- 普通网站：0.58
- 来源不明：0.45

---

## 五、 细节补充与扩展 (结合 2026 年 AI Agent 落地最佳实践)

在 2026 年的企业级 AI Agent 落地中，仅有业务逻辑是不够的，以下是原报告未明确但对**生产环境至关重要**的补充：

### 5.1 可观测性与监控 (Observability)

*2026年标准：没有 Trace 的 Agent 系统无法上线。*
- **OpenTelemetry 集成**：利用 Spring Boot 4.x 原生支持的 Micrometer Tracing，为每个 Agent 节点、每次 LLM 调用、每次 Vector Search 生成 Span。
- **Token 与成本监控**：在 `ChatClient` 外层包裹 Advisor，拦截 `ChatResponse` 中的 `usage` 信息，实时写入 Prometheus，监控单次 Research 的 Token 消耗和 API 成本。
- **LLM 评估 (LLM-as-a-Judge)**：在 Writer 节点后增加一个异步的 `EvalAgent`，对生成的报告进行相关性、连贯性、引用准确性打分，用于持续优化 Prompt。

### 5.2 容错、降级与限流 (Resilience)

- **LLM API 限流与重试**：使用 **Resilience4j** 或 Spring Retry，针对 LLM API 的 429 (Rate Limit) 和 5xx 错误配置指数退避重试。
- **模型降级策略 (Model Fallback)**：当主模型（如 qwen-max）超时或不可用时，自动降级到备用模型（如 qwen-plus），保证研究流程不中断。
- **搜索 API 熔断**：若 Bocha Search API 连续失败，触发熔断，自动切换至备用搜索引擎（如 Tavily 或 Bing Search），或仅依赖 Local RAG 并明确告知用户。

### 5.3 安全、合规与数据隔离 (Security & Multi-tenancy)

- **多租户数据隔离**：在 `ResearchState` 中强制携带 `tenantId`。在 Milvus/VectorStore 检索时，必须通过 `FilterExpression` (如 `tenant_id == 'xxx'`) 进行硬隔离，防止企业 A 查到企业 B 的知识库。
- **PII 脱敏**：在 Query 进入 LLM 前，通过 Spring AI 的 `PII Masking Advisor` 自动掩码手机号、身份证等敏感信息。
- **Prompt 注入防护**：在 Intent Router 前增加安全过滤层，识别并拦截恶意的 Prompt 注入攻击。

### 5.4 性能优化与用户体验 (UX)

- **SSE 细粒度流式推送**：不要等 3000 字报告全写完再推。利用 WebFlux 和 SSE，**按章节、按 Agent 阶段**实时推送进度（如：“正在规划大纲...”、“正在检索网络证据 (3/6)...”、“正在撰写第二章...”），极大缓解用户等待焦虑。
- **语义缓存 (Semantic Cache)**：使用 Redis + 向量相似度，对高度相似的 Query（如“2026年AI产品趋势”和“2026年人工智能产品发展趋势”）直接命中缓存，跳过昂贵的多 Agent 研究流程。

---

## 六、 项目目录结构 (Java / Spring Boot 规范)

```text
deep-research-agent/
 ├── src/main/java/com/example/deepresearch/
 │   ├── DeepResearchApplication.java          # Spring Boot 主入口
 │   │
 │   ├── api/                                  # 接口层 (WebFlux/SSE)
 │   │   ├── controller/                       # REST & SSE 控制器
 │   │   ├── dto/                              # 请求/响应 DTO
 │   │   └── websocket/                        # WebSocket 处理器 (可选)
 │   │
 │   ├── agent/                                # 智能体层 (核心)
 │   │   ├── bundle/AgentBundle.java           # Agent 集合管理
 │   │   ├── intent/IntentRouterAgent.java     # 意图路由
 │   │   ├── planner/PlannerAgent.java         # 规划师
 │   │   ├── scout/WebScoutAgent.java          # 网络侦察
 │   │   ├── scout/LocalScoutAgent.java        # 本地 RAG 侦察
 │   │   ├── judge/EvidenceJudgeAgent.java     # 证据裁判
 │   │   ├── analyst/AnalystAgent.java         # 分析师
 │   │   ├── reflect/ReflectAgent.java         # 反思补搜
 │   │   └── writer/WriterAgent.java           # 撰稿人
 │   │
 │   ├── workflow/                             # 编排层 (状态机/工作流)
 │   │   ├── ResearchWorkflow.java             # 工作流定义与编排
 │   │   ├── state/ResearchState.java          # 状态定义 (Record)
 │   │   ├── node/                             # 各节点执行逻辑
 │   │   └── router/                           # 条件路由逻辑
 │   │
 │   ├── memory/                               # 记忆系统
 │   │   ├── MemoryManager.java                # 记忆管理器
 │   │   ├── ShortTermMemoryService.java       # 短期记忆 (Redis)
 │   │   └── LongTermMemoryService.java        # 长期记忆 (PG)
 │   │
 │   ├── rag/                                  # RAG 检索系统
 │   │   ├── VectorStoreConfig.java            # Milvus/PgVector 配置
 │   │   ├── DocumentIngestionService.java     # 文档导入 ETL
 │   │   └── CitationValidator.java            # 引用合法性校验
 │   │
 │   ├── tool/                                 # 外部工具集成
 │   │   ├── search/BochaSearchTool.java       # Bocha 搜索 API 封装
 │   │   └── search/FallbackSearchTool.java    # 备用搜索
 │   │
 │   └── common/                               # 公共组件
 │       ├── config/                           # Spring 配置类
 │       ├── exception/                        # 全局异常处理
 │       ├── util/JsonUtils.java               # JSON 安全解析
 │       └── observability/                    # 监控与 Tracing
 │
 ├── src/main/resources/
 │   ├── application.yml                       # 主配置文件
 │   ├── prompts/                              # Prompt 模板 (独立管理)
 │   │   ├── intent-router.st                  # StringTemplate 格式
 │   │   ├── planner.st
 │   │   └── ...
 │   └── schema/                               # JSON Schema (用于 OutputConverter)
 │
 └── pom.xml                                   # Maven 依赖 (Spring AI 2.0 BOM)
```

---

## 七、 给 AI 编码助手的精准 Prompt 建议

当您拿着这份报告让 AI 帮您写代码时，建议采用以下 **分阶段、模块化** 的 Prompt 策略，以确保精准无误地高质量落地：

### 阶段 1：基础设施与配置

> "请基于 Spring Boot 4.x 和 Spring AI 2.0 (2.0.0 GA) 初始化项目。配置 `pom.xml` 引入 `spring-ai-starter-model-openai` (或通义千问对应的 starter)、`spring-ai-starter-vector-store-milvus`、`spring-boot-starter-webflux`。创建 `application.yml`，配置多模型支持和虚拟线程。"

### 阶段 2：状态定义与工具类

> "请根据报告中的 `ResearchState` 定义，使用 Java `Record` 实现不可变状态类，并提供 `withXxx` 更新方法。实现 `JsonUtils` 工具类，包含从 LLM 输出中安全提取 JSON 块并反序列化为 Java 对象的逻辑，必须包含 Fallback 机制。"

### 阶段 3：单 Agent 实现

> "请使用 Spring AI 2.0 的 `ChatClient` 和 `BeanOutputConverter` 实现 `PlannerAgent`。System Prompt 从 `resources/prompts/planner.st` 加载。Temperature 设为 0.3。输入为 Query，输出为 `PlanResult` (包含 objective, subQuestions, outline 等字段)。请处理 LLM 输出格式错误的异常。"

### 阶段 4：工作流编排与并行检索

> "请使用 Java 21+ 的 `StructuredTaskScope` 和虚拟线程，实现 `WebScout` 和 `LocalScout` 的并行检索逻辑。实现 `CitationValidator`，使用正则校验 Writer 输出的引用 ID 是否在合法的 `source_index` 中，并自动剔除非法引用。"

### 阶段 5：SSE 流式接口

> "请使用 Spring WebFlux 实现 `/api/research/stream` 接口，返回 `Flux<ServerSentEvent<String>>`。在工作流执行的每个阶段（Intent, Plan, Search, Write），向客户端推送进度状态和最终的 Markdown 报告流。"

---

**总结**：本报告将原有的 Python 原型方案全面升级为了符合 2026 年企业级标准的 Java/Spring AI 架构。通过引入虚拟线程、完善的可观测性、多租户隔离和严格的容错机制，确保了 DeepResearch 系统不仅“能跑通”，更能“在高并发生产环境中稳定、安全、低成本地运行”。



