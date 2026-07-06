# DeepResearch 多 Agent 行业深度研究助手 — 需求分析报告（Spring AI 2.0 落地版）

> 报告生成时间：2026 年 7 月
> 目标技术栈：Spring AI 2.0.0 GA（2026-06-12 发布）+ Spring Boot 4.x + Spring Framework 7.x + Java 21 LTS
> 报告用途：作为后续 AI 辅助编码的"需求规格说明书"（SRS），需保证可落地、可验证、可追踪

---

## 第 0 部分：阅读指引与口径说明

本报告基于对原项目（Python + LangGraph + LangChain Tongyi + FastAPI 实现）的逆向拆解，重新组织为面向 Java/Spring 技术栈的实现需求。每个模块均包含三层信息：
- **L1 业务需求**：用户/产品视角必须达成的目标，与实现语言无关；
- **L2 技术规格**：从原项目提炼的工程细节（含数据结构、算法、阈值、Prompt）；
- **L3 Spring AI 2.0 落地方案**：基于 Spring AI 2.0 GA 能力的具体实现路径，含原项目未明确处我做的补充扩展（标记为【补充】）。

凡标记【补充】的内容，均为原拆解文档未明确、但工程落地必须补齐的部分，依据 2026 年 7 月时点业界成熟做法给出建议。

---

## 第 1 部分：项目背景与产品定位

### 1.1 业务背景

在大模型时代，用户信息获取需求从"单轮问答"升级为"深度研究"。目标场景具备四大共性：
1. **信息分散**：需多源检索交叉验证；
2. **需要验证**：需识别冲突和谣言；
3. **逻辑复杂**：需多步推理形成结论；
4. **可追溯性**：需精确到来源 ID 的引用。

典型场景：投资研究、技术调研、竞品分析、政策解读。

### 1.2 产品定位

面向企业级深度研究场景的 AI 多智能体系统，定位为"AI 研究员助手"。

**能力矩阵对照**（提炼自原文档）：

| 能力维度 | 传统 Chatbot | DeepResearch |
|---|---|---|
| 信息来源 | 训练数据（静态） | 实时 Web 检索 + 本地知识库 |
| 回答长度 | 简短回复 | 3000+ 字深度报告 |
| 引用溯源 | 无 | 精确到 source_id |
| 冲突检测 | 无 | 自动识别并标记 audit_flags |
| 迭代优化 | 单轮 | 多轮补搜直到完备 |
| 记忆继承 | 单会话 | 跨会话长期记忆 |

**目标用户**：投资分析师、行业研究员、技术架构师、产品经理、咨询顾问、政策分析师、知识管理专员。

### 1.3 技术演进路线（当前 V2.0）

V1.0 单 Agent RAG → V2.0 多 Agent 协作（当前版本），包含 8 个 Agent 角色：Intent Router、Planner、Web Scout、Local Scout、Evidence Judge、Analyst、Reflect、Writer。

---

## 第 2 部分：系统总体架构

### 2.1 分层架构（四层）

```
┌────────────────────────────────────────────────────────────┐
│  L4 接口层  CLI / FastAPI REST / WebSocket 实时推送          │
├────────────────────────────────────────────────────────────┤
│  L3 编排层  LangGraph 状态机引擎（含 8 节点 + 条件路由）       │
├────────────────────────────────────────────────────────────┤
│  L2 智能体层  8 个 Agent（独立 LLM 实例 + 独立 Prompt + 温度）│
├────────────────────────────────────────────────────────────┤
│  L1 服务层  Bocha 搜索 / Milvus 向量库 / PostgreSQL / Redis  │
│            + Memory Service（短期/长期/语义三层）             │
└────────────────────────────────────────────────────────────┘
```

### 2.2 核心数据流（7 步主流程）

1. **意图识别**：判定 direct / multiagent 路由
2. **任务规划**：拆解为 objective + sub_questions + outline + budget
3. **并行检索**：Web Scout（Bocha）+ Local RAG（Milvus）并行执行
4. **证据审计**：评分、去重、冲突检测，构建 evidence_pool + source_index
5. **分析归纳**：形成 findings + claim_map，评估 needs_more_research
6. **反思补搜**（条件分支）：生成 supplementary_queries 回到第 3 步
7. **报告撰写**：生成 Markdown + 自动校验引用 + 拼接参考资料

### 2.3 【补充】四层架构在 Spring AI 2.0 下的映射

| 原架构层 | Spring AI 2.0 对应方案 |
|---|---|
| L1 服务层 | Spring AI VectorStore 抽象（MilvusVectorStore）+ Spring Data JPA（PostgreSQL）+ Spring Data Redis + 自定义 BochaClient（HTTP Interface） |
| L2 智能体层 | Spring AI `ChatClient` + `Advisor` + `StructuredOutputConverter`；每个 Agent 一个独立 ChatClient bean，温度参数通过 `OpenAiChatOptions` / `DashScopeChatOptions` 配置 |
| L3 编排层 | 【补充】Spring AI 2.0 GA 未内置完整 DAG 工作流引擎，三选一：(a) Spring StateMachine；(b) 自研基于 `TaskExecutor` + `CompletableFuture` 的图执行器；(c) 集成 LangGraph4j（社区已有 Java 移植）。**推荐方案**：自研轻量级 `WorkflowGraph` + 节点抽象，理由是 8 节点规模无需重型框架，且便于嵌入 Spring 的 Observation/Retry/事务。 |
| L4 接口层 | Spring Boot 4 `@RestController` + `WebMvc`（同步）+ `SseEmitter`/`Flux<ServerSentEvent>`（流式）+ Spring Shell（CLI） |

---

## 第 3 部分：8 个 Agent 详细需求规格

每个 Agent 拆解为：职责、输入、输出、Prompt 要点、温度参数、回退方案、Spring AI 落地点。

### 3.1 Intent Router（意图路由器）

- **职责**：判断 direct（问候/闲聊/简单事实）vs multiagent（调研/对比/分析/趋势）。
- **温度**：0.0（确定性最高）。
- **判断标准（双重判定）**：
    - 规则引擎初判（`detect_intent`）：扫描强制多 Agent 关键词（调查、调研、来源、证据、盘点、趋势、最新、热门项目等）+ "年份+趋势"组合词 + 扩展关键词列表（方案、架构、对比、报告、知识库等）。
    - LLM 复判：输出 `{"route":"direct|multiagent","reason":"..."}`
- **输出契约**：JSON 严格 schema，非法值回退到规则引擎结果。
- **路由条件边**：`direct → direct_answer`；`multiagent → plan`。
- **【补充】Spring AI 落地**：
    - 使用 `chatClient.prompt().user(prompt).call().entity(IntentResult.class)` 走 `BeanOutputConverter`，自动绑定 JSON 到 POJO。
    - 规则引擎用 `IntentRuleEngine` Spring `@Component`，便于单元测试。
    - 配置 `ChatClient` bean 名 `intentRouterChatClient`，注入 `DashScopeChatOptions.builder().withTemperature(0.0).build()`。

### 3.2 Planner（规划师）

- **职责**：将一句话 Query 拆解为可执行研究计划。
- **温度**：0.3。
- **输出 JSON 结构**（严格 schema）：
  ```
  {
    "objective": "...",
    "sub_questions": ["原问题", "扩展1", "扩展2"],
    "outline": [
      {
        "id":"sec_1","title":"...","description":"...",
        "section_type":"mixed","requires_data":true,"requires_chart":false,
        "priority":1,"search_queries":["..."],"status":"pending"
      }
    ],
    "research_questions": ["..."],
    "budget": {"max_rounds":2,"max_sources":12,"max_tokens":12000,"max_seconds":45}
  }
  ```
- **关键算法**：`_derive_search_plan` 从 outline + sub_questions + research_questions 派生搜索计划：
    1. 添加围绕原问题的直接查询；
    2. 遍历 outline 章节，提取 search_queries；
    3. `_is_query_grounded` 校验查询是否接地（避免凭空生成）；
    4. `_dedupe_sources` 去重；
    5. 最多保留 6 个查询。
- **回退方案**：`_default_plan` 生成单章节默认大纲。
- **【补充】**：
    - `section_type` 取值原文档只写 "mixed"，应明确枚举为 `text|data|mixed|chart`，以驱动 Writer 后续差异化渲染。
    - `requires_chart=true` 时应触发【补充】图表生成子流程（可用 Spring AI Tool Calling 调用图表渲染服务）。
    - **预算控制**：`max_rounds`、`max_sources`、`max_tokens`、`max_seconds` 必须在编排层强约束，超出立即终止迭代进入 write 节点（原项目仅作配置，未明确执行机制）。

### 3.3 Web Scout（网络侦察员）

- **职责**：执行 Bocha 搜索、过滤、结构化、分配 source_id。
- **温度**：0.4。
- **source_id 命名规则**：`WEB{iteration}_{queryIndex}-{resultIndex}`，例 `WEB1_1-1`。
- **过滤策略**：
    - 黑名单域名（`_is_bad_web_domain`）；
    - 空记录（title + snippet 都为空）；
    - 相关性评分（`_estimate_relevance`，低于 0.2 且非官方域名丢弃）。
- **Bocha API 调用**：count=4，减少无用 token。
- **可信度标记枚举**：`official|media|community|unknown`。
- **输出 JSON**：summary + evidence[] + gaps[] + rejected_source_ids[] + reject_reason。
- **重要约束**：evidence 中的 source_id 必须来自输入，禁止编造；`_prune_evidence_to_allowed_sources` 做白名单校验；`_enrich_evidence_from_raw` 补充 URL。
- **【补充】**：
    - Bocha 客户端用 Spring 4 `@HttpExchange` 声明式 HTTP Interface + `WebClient` 异步实现，支持响应式回压。
    - 搜索结果缓存：相同 query + count 在 N 分钟内复用 Redis 缓存，key = `bocha:{sha256(query+count)}`。
    - **限流**：Bocha QPS 限制需用 `Resilience4j RateLimiter` 包装，避免 429。
    - **重试**：网络抖动用 `RetryTemplate` 指数退避 3 次。
    - **国际化**：原项目默认中文，【补充】应支持 `locale` 参数，影响 Bocha 搜索区域和语言。

### 3.4 Local RAG Scout（本地知识侦察员）

- **职责**：检索 Milvus 向量库，过滤、结构化、分配 source_id。
- **温度**：0.4。
- **source_id 命名规则**：`LOC{iteration}_{queryIndex}-{resultIndex}`。
- **与 Web Scout 对比**：
    - 数据源：Milvus vs Bocha；
    - 默认可信度：0.92 vs 0.45-0.88；
    - 定位器：doc_id vs URL。
- **检索参数**：`limit=4` 每查询。
- **去重键**：`["doc_id","snippet"]`（Web 用 `["url","title"]`）。
- **【补充】**：
    - Spring AI 2.0 已原生支持 Milvus：`MilvusVectorStore` 实现 `VectorStore` 接口。
    - 检索使用 `vectorStore.similaritySearch(SearchRequest.query(q).withTopK(4).withSimilarityThreshold(0.6))`。
    - **租户隔离**：`collection_name` 应按 tenant_id 隔离，或在 metadata filter 中带 `tenant_id` 字段。
    - **embedding 模型**：原项目用 DashScope embedding，Spring AI 2.0 已支持 `DashScopeEmbeddingModel`；【补充】建议支持 `EmbeddingModel` 切换，便于切换 BGE-M3 等开源模型。
    - **文档导入**：`rag/ingest.py` 在 Spring 中对应 `@Service KnowledgeIngestService`，支持 PDF/Word/Markdown/HTML，使用 Apache Tika 解析，按 token 数分块（建议 500 token + 50 重叠）。

### 3.5 Evidence Judge（证据裁判）

- **职责**：评分、去重、冲突检测、构建 source_index。
- **温度**：0.2。
- **评分标准**（关键工程参数）：
  | 来源类型 | 分数 |
  |---|---|
  | 本地知识库 | 0.92 |
  | 官方/政府域名 | 0.88 |
  | 主流媒体 | 0.72 |
  | 普通网站 | 0.58 |
  | 来源不明 | 0.45 |
- **官方域名识别**：`.gov.cn`、`.gov`、`.edu`、`.edu.cn`、含 "gov" 或 "official"。
- **主流媒体识别**：域名含 `news|finance|reuters|bloomberg|people|xinhuanet`。
- **冲突检测 audit_flags 三类**：`low_confidence`（<0.6）、`conflict`（与其他证据矛盾）、`missing_evidence`（子问题缺少直接证据）。
- **输出**：evidence_pool[] + audit_flags[] + source_index[]。
- **白名单校验**：source_id 必须来自 raw_evidence。
- **补充缺失证据**：LLM 漏掉时由 `_score_evidence` 算法补齐。
- **【补充】**：
    - 域名分类规则过于简化，【补充】引入域名信誉库（domain_reputation 表），支持运营人员维护，避免硬编码。
    - 冲突检测原项目依赖 LLM 判断，【补充】可前置一个基于 embedding 余弦相似度的预筛：相似度 >0.85 但关键实体值不同的两证据，标记为候选冲突交 LLM 复判。
    - evidence_pool 应持久化到 PostgreSQL，便于审计与回放。

### 3.6 Analyst（分析师）

- **职责**：从证据池形成 findings + claim_map，评估完备性，决定是否补搜。
- **温度**：0.3。
- **输出 JSON**：
  ```
  {
    "analysis_summary":"...",
    "needs_more_research": bool,
    "missing_gaps": ["..."],
    "findings": [
      {"claim_id":"c_1","claim":"...","confidence":"high|medium|low","source_ids":["..."]}
    ],
    "claim_map": [{"claim_id":"c_1","source_ids":["..."]}],
    "next_actions": ["..."]
  }
  ```
- **决策逻辑**：
    - 证据充足 → needs_more_research=false → write
    - 证据不足 → needs_more_research=true → reflect
    - 达到 max_iterations → 强制 write（编排层兜底）
- **重要约束**：每个结论必须绑定 source_id；不确定明确写 uncertain；needs_more_research 必须诚实评估。
- **【补充】**：
    - "诚实评估"缺乏量化标准，【补充】引入显式规则：
        - 任一 sub_question 在 findings 中无 claim 覆盖 → needs_more_research=true
        - findings 中 confidence=low 占比 >50% → needs_more_research=true
        - audit_flags 含 missing_evidence → needs_more_research=true
    - claim_map 与 findings 冗余存储，建议合并为 findings 内嵌 source_ids 字段，减少一致性风险。

### 3.7 Reflect（反思补搜）

- **职责**：基于 missing_gaps 生成新查询，避免重复。
- **温度**：原项目 `build_agents` 中未显式列出 reflect，但根据功能定位建议 0.4。
- **触发条件**：`needs_more_research=true && iteration < max_iterations`。
- **输出 JSON**：
  ```
  {
    "reflection_summary":"...",
    "supplementary_queries": [
      {"section_id":"gap_1","query":"...","source_preference":"hybrid","reason":"..."}
    ]
  }
  ```
- **重要约束**：新查询必须与已执行查询不同（避免重复）；每个 gap 生成 1-2 个查询；source_preference 取值 `web|local|hybrid`。
- **【补充】**：
    - 原项目"避免重复"依赖 LLM 自觉，【补充】在编排层用 `Set<String> executedQueries` 强校验，重复查询直接丢弃并触发再生成（最多 3 次，否则回退）。
    - source_preference 在 Reflect 中输出，但 Web/Local Scout 都用 `_build_queries(state, source_preference)` 筛选，需保证 hybrid 同时被两端取到（原代码 `pref in (source_preference, "hybrid")` 已正确处理）。
    - 迭代计数器 `iteration` 在 reflect_node 末尾 +1，编排层据此判断 max_iterations。

### 3.8 Writer（撰稿人）

- **职责**：整合 findings，生成 2000-3000+ 字 Markdown 报告。
- **温度**：0.4。
- **输出**：纯 Markdown 正文，不输出 JSON、不输出代码块标记。
- **引用规则**：
    - 引用 ID 必须来自 source_index 的合法集合；
    - 格式：`[WEB1_1-1]`、`[LOC1_1-3]`；
    - `_validate_and_fix_citations` 正则 `\[([A-Z]+\d+_\d+-\d+)\]` 校验，非法引用直接移除；
    - 系统自动在末尾拼接 `## 参考资料`，Writer 不需要列举。
- **参考资料渲染规则**（`_render_reference_list`）：
    - 从正文提取实际引用的 source_id；
    - 分离 WEB 和 LOCAL；
    - LOCAL 按 locator 去重；
    - 格式：`- [source_id] [type]: label | locator`。
- **强制清理**：移除开头的 ```json / ```markdown / ``` 和结尾的 ```。
- **【补充】**：
    - Writer Prompt 中明确"绝不能写成简短大纲"是软约束，【补充】在编排层加硬校验：字数 < 1500 字时自动追加扩展指令重试一次（最多 1 次）。
    - 报告模板系统：【补充】支持按行业（金融/科技/政策）切换模板，模板定义章节结构和写作风格。
    - 导出能力：【补充】支持 PDF（Flying Saucer/iText）、Word（Apache POI）、Markdown 三种导出，对应接口 `/api/reports/{id}/export?format=pdf|docx|md`。
    - 流式输出：Writer 是最终用户最关心的环节，【补充】必须支持 SSE 流式输出，使用 Spring AI `stream()` 返回 `Flux<String>`。

---

## 第 4 部分：编排层（Workflow）需求

### 4.1 节点拓扑

```
START
  ↓
intent ──(direct)──→ direct_answer ──→ END
  │
  └─(multiagent)──→ plan
                      ↓
                ┌─────┴─────┐
                ↓           ↓
           web_search   local_rag        ← 并行
                └─────┬─────┘
                      ↓
                  deep_dive                  ← 等待两个上游完成
                      ↓
                   analyze
                      ↓
              ┌─(needs_more_research && iter<max)─→ reflect
              │                                       ↓
              │                                  (回到 web_search + local_rag)
              │
              └─(否则)─→ write ──→ END
```

### 4.2 条件路由规则

| 路由点 | 函数 | 条件 | 目标 |
|---|---|---|---|
| intent 后 | `route_after_intent` | intent=="direct" / 否则 | direct_answer / plan |
| analyze 后 | `should_continue_research` | iteration>=max / needs_more_research / 否则 | write / reflect / write |

### 4.3 并行节点约束

- `plan → web_search` 与 `plan → local_rag` 同时添加边；
- `web_search → deep_dive` 与 `local_rag → deep_dive` 同时添加边；
- **deep_dive 必须等待两个上游都完成**（barrier 同步）。

### 4.4 【补充】Spring AI 2.0 落地方案

由于 Spring AI 2.0 GA 未内置完整 DAG 引擎，推荐自研 `WorkflowGraph`：

```java
public interface WorkflowNode<S> {
    String name();
    Mono<S> execute(S state);   // 响应式
    default boolean isBarrier() { return false; }
}
public class WorkflowGraph<S> {
    void addNode(String name, WorkflowNode<S> node);
    void addEdge(String from, String to);
    void addConditionalEdge(String from, Function<S,String> router, Map<String,String> mapping);
    void addParallelBarrier(List<String> upstreams, String barrier);
    Mono<S> run(S initialState);
}
```

**关键技术点**：
- 使用 Project Reactor `Mono.zip` 实现 barrier 同步；
- 使用 `Schedulers.boundedElastic()` 避免阻塞虚拟线程；
- 每个节点用 `@Observed` 注解接入 Micrometer Observation；
- 节点失败用 `Retry.backoff(3, Duration.ofMillis(200))` 重试，超出后走 fallback；
- 状态持久化：每个节点执行后 `state` 序列化为 JSON 入 PostgreSQL `research_run_state` 表，支持断点续跑。

### 4.5 【补充】异步任务管理

深度研究流程耗时 30-120 秒，必须异步化：
- 接口层接收 query → 返回 `runId` + 202 Accepted；
- 后台 `@Async` 或 `TaskExecutor` 执行 workflow；
- 客户端通过 `GET /api/runs/{runId}/status` 轮询或 `WebSocket /ws/runs/{runId}` 订阅；
- 状态机：`PENDING → RUNNING → ANALYZING → WRITING → COMPLETED / FAILED`；
- 【补充】使用 Spring Boot 4 的 `ScheduledExecutorService` + Virtual Threads 实现轻量并发。

---

## 第 5 部分：状态管理（ResearchState）

### 5.1 状态字段完整清单（提炼自 state.py）

```
query, user_id, tenant_id, memory_context,
messages: Annotated[List<BaseMessage>, operator.add],   // 自动累积
intent, phase, plan, outline, sub_questions, research_questions,
search_plan, budget,
web_search, local_rag, web_evidence[], local_evidence[],
evidence_pool[], deep_dive, audit, audit_flags[],
analysis, needs_more_research, missing_gaps[], supplementary_queries[],
findings[], claim_map[], source_index[],
web_retrieval_stats, local_retrieval_stats,
web_search_trace[], local_rag_trace[],
draft, final, iteration, max_iterations
```

### 5.2 【补充】Spring AI 2.0 落地

- 用 Java `record` 或 `@Data` POJO 替代 TypedDict；
- `messages` 累积语义用自定义合并函数 + Spring AI `MessageWindowChatMemory`（限制窗口大小，避免 token 爆炸）；
- 状态持久化：每个节点完成后 `StateRepository.save(runId, state)` 写 PostgreSQL JSONB；
- 【补充】状态字段过多（30+），建议拆分为 4 个聚合根：`ResearchRun`（核心）、`EvidencePool`、`AnalysisResult`、`ReportDraft`，避免单表过大。

---

## 第 6 部分：记忆系统（三层架构）

### 6.1 三层结构

```
MemoryManager (统一接口)
   ├── ShortTermMemory   会话级，存当前 thread 摘要
   ├── LongTermMemory    用户级/线程级，存历史结论
   └── SemanticMemory    向量检索，存语义相关记忆
       └── MilvusVectorStore
```

### 6.2 核心接口

```python
build_personalized_prompt_context(user_id, thread_id, query)
  → 拼接 "用户画像 + 对话摘要 + 相关记忆"
```

### 6.3 后端配置项

- `short_term_backend`: postgres / memory
- `long_term_backend`: postgres / sqlite / disabled
- `enable_milvus`: bool
- `long_term_scope`: user / thread

### 6.4 【补充】Spring AI 2.0 落地

Spring AI 2.0 提供 `ChatMemory` 抽象：
- `InMemoryChatMemory` → 对应 short_term_backend=memory
- `JdbcChatMemoryRepository` + `PostgresChatMemoryRepository` → 对应 postgres
- 自定义 `SemanticChatMemory` 实现，内部委托 `MilvusVectorStore` 做语义检索

**`MemoryManager` 实现建议**：

```java
@Service
public class MemoryManager {
    private final ChatMemory shortTerm;          // Spring AI 内置
    private final LongTermMemoryRepository longTerm;  // JPA
    private final VectorStore semanticStore;     // Milvus
    public String buildPersonalizedPromptContext(String userId, String threadId, String query) {
        var profile = longTerm.findUserProfile(userId);
        var summary = shortTerm.get(threadId).stream()
            .map(Message::getText).collect(joining("\n"));
        var memories = semanticStore.similaritySearch(
            SearchRequest.query(query).withFilterExpression("userId=='"+userId+"'").withTopK(5));
        return template.render(Map.of("profile",profile,"summary",summary,"memories",memories));
    }
}
```

**【补充】记忆生命周期管理**：
- 短期记忆 TTL：默认 24h，超时自动归档到长期；
- 长期记忆压缩：每 N 条触发 LLM 摘要压缩；
- 隐私：PII 字段（手机号/身份证）入库前脱敏；
- 记忆遗忘：支持 `DELETE /api/memories/{userId}` 满足 GDPR/个保法。

---

## 第 7 部分：双源检索与证据融合

### 7.1 并行检索设计

- `plan → web_search` + `plan → local_rag` 同时触发；
- 两节点独立执行，互不依赖；
- `deep_dive` 作为 barrier 等待两者完成。

### 7.2 证据评分算法（核心业务规则）

```python
local           → 0.92  企业内部
.gov/.edu/官方   → 0.88
news/finance    → 0.72
普通网站         → 0.58
来源不明         → 0.45
```

### 7.3 去重规则

| 数据类型 | 去重键 |
|---|---|
| Web 原始记录 | `["url","title"]` |
| Local 原始记录 | `["doc_id","snippet"]` |
| source_index | `["source_id"]` |
| 搜索计划 | `["query"]` |

### 7.4 【补充】融合策略增强

原项目仅做评分+去重，【补充】增加：
- **时间衰减**：发布时间 > 365 天的证据 score *= 0.85；
- **多源加分**：同一 claim 被 3+ 独立来源支撑，confidence 自动升一档；
- **来源多样性**：单次报告同一域名引用上限 3 次，避免单源垄断；
- **冲突仲裁**：冲突证据中官方/本地优先级 > 媒体 > 普通网站。

---

## 第 8 部分：引用验证系统（核心创新点）

### 8.1 验证算法

```
正则：\[([A-Z]+\d+_\d+-\d+)\]
   ↓ 提取所有引用 ID
对每个 ID：
   在 valid_source_ids_set 中？ → 保留 [ID]
   否则                          → 移除（空字符串）
```

### 8.2 参考资料渲染

```
1. 从正文提取 cited_ids（去重保序）
2. 分离 web_ids 和 local_ids
3. local_ids 按 locator 去重（同一文档多次引用只列一次）
4. 渲染为 "- [source_id] [type]: label | locator"
```

### 8.3 【补充】Spring AI 落地

- 用 `Pattern.compile("\\[([A-Z]+\\d+_\\d+-\\d+)\\]")` 编译为静态 Pattern；
- `CitationValidator` 作为 `@Component`，可单元测试；
- 【补充】增强：识别 `[WEB1_1-1][WEB1_1-2]` 连续引用合并显示为 `[WEB1_1-1, WEB1_1-2]`；
- 【补充】增强：报告渲染时为每个引用生成超链接（web 类型 URL 可点击）。

---

## 第 9 部分：接口层需求

### 9.1 原项目接口

- CLI（main.py，支持 `--user-id`、`--thread-id`、`--once-query`、`--short-term-backend` 等参数）
- FastAPI REST API
- WebSocket 实时推送

### 9.2 【补充】Spring Boot 4 REST API 设计

```
POST   /api/research/runs                    创建研究任务（异步，返回 runId）
GET    /api/research/runs/{runId}            查询任务详情
GET    /api/research/runs/{runId}/status     查询任务状态
GET    /api/research/runs/{runId}/stream     SSE 流式订阅（推荐）
WS     /ws/research/runs/{runId}             WebSocket 实时推送
POST   /api/research/runs/{runId}/cancel     取消任务
GET    /api/research/runs/{runId}/report     获取最终报告
GET    /api/research/runs/{runId}/export     导出 (?format=pdf|docx|md)
POST   /api/knowledge/ingest                 文档导入（multipart）
GET    /api/knowledge/documents              文档列表
DELETE /api/knowledge/documents/{docId}      删除文档
GET    /api/memories/{userId}                用户记忆查询
DELETE /api/memories/{userId}                删除用户记忆
GET    /api/health                           健康检查
GET    /api/metrics                          指标（Prometheus）
```

### 9.3 【补充】安全与多租户

- 认证：Spring Security + JWT（OAuth2 Resource Server）；
- 租户隔离：`@TenantId` 注解 + `TenantInterceptor` 自动注入到查询条件；
- API 限流：`RequestRateLimiter`（Spring Cloud Gateway 或本地 Bucket4j）；
- 输入校验：`@Valid` + Bean Validation，防止 Prompt 注入（基础过滤）；
- 内容审核：【补充】Writer 输出前调用审核服务（自建或阿里云内容安全）。

---

## 第 10 部分：配置与部署

### 10.1 环境变量（提炼自 .env）

```
DASHSCOPE_API_KEY=sk-xxx
BOCHA_API_KEY=sk-xxx
POSTGRES_DSN=postgresql://user:pass@host:5432/db
MILVUS_HOST=xxx.xxx.xxx.xxx
MILVUS_PORT=19530
MILVUS_COLLECTION=mult_agent_memory
```

### 10.2 配置文件（提炼自 config.json）

```json
{
  "api_key": "",
  "model": "qwen-turbo",
  "max_iterations": 3,
  "enable_memory": true,
  "short_term_backend": "postgres",
  "long_term_backend": "postgres",
  "enable_milvus": true,
  "checkpointer_backend": "postgres"
}
```

### 10.3 【补充】Spring Boot 4 配置

`application.yml`：

```yaml
deep-research:
  model:
    provider: dashscope
    name: qwen-turbo
    embeddings: text-embedding-v3
  agents:
    intent-router:   { temperature: 0.0 }
    planner:         { temperature: 0.3 }
    scout-web:       { temperature: 0.4 }
    scout-local:     { temperature: 0.4 }
    evidence-judge:  { temperature: 0.2 }
    analyst:         { temperature: 0.3 }
    reflect:         { temperature: 0.4 }
    writer:          { temperature: 0.4 }
  workflow:
    max-iterations: 3
    max-sources: 12
    max-tokens: 12000
    max-seconds: 45
  memory:
    short-term-backend: postgres
    long-term-backend: postgres
    enable-milvus: true
    long-term-scope: user
  storage:
    milvus:
      host: ${MILVUS_HOST}
      port: ${MILVUS_PORT}
      collection: mult_agent_memory
    postgres:
      url: ${POSTGRES_DSN}
    redis:
      host: ${REDIS_HOST}
      port: 6379
  bocha:
    base-url: https://api.bochaai.com
    api-key: ${BOCHA_API_KEY}
    count: 4
    cache-ttl: 600s
  observation:
    enabled: true
    tracing: otel
```

### 10.4 【补充】部署架构

- 容器化：Docker + Docker Compose（开发）/ Kubernetes（生产）；
- Java 21 LTS + Spring Boot 4 原生镜像支持 AOT；
- 数据库：PostgreSQL 16 + pgvector（备选 Milvus 独立部署）；
- 缓存：Redis 7（Cluster 模式生产）；
- 反向代理：Nginx / Spring Cloud Gateway；
- 可观测：Micrometer + OpenTelemetry → Prometheus + Grafana + Tempo；
- 日志：ELK / Loki。

---

## 第 11 部分：关键设计模式（必须复刻）

### 11.1 节点绑定模式

原项目用 `functools.partial` 将 Agent 绑定到节点函数。Spring 中用 `BiFunction<State, Agent, State>` 或显式构造 `new PlanNode(agents.planner())` 实现。

### 11.2 JSON 安全解析模式

每个节点都有 fallback：LLM 输出非法 JSON 时回退到默认方案，保证系统不崩溃。

**【补充】Spring AI 落地**：使用 `BeanOutputConverter<T>` + try-catch，失败时调用 `FallbackProvider<T>` 接口提供默认值。每个 Agent 一个 FallbackProvider 实现。

### 11.3 来源去重模式

`_dedupe_sources(items, key_fields)` 通用工具函数，基于指定字段的 tuple key 去重。Spring 中实现为静态工具方法或 `Deduplicator` 组件。

### 11.4 【补充】其他必须模式

- **温度参数隔离**：每个 Agent 独立 ChatClient bean，禁止共用（防止温度参数污染）；
- **白名单校验**：source_id、doc_id 等 ID 字段必须经过白名单校验；
- **追踪日志**：每个节点的输入输出全量记录到 `node_trace` 表（JSONB），便于回放与调试；
- **幂等性**：节点重试时检查 state 中是否已有结果，避免重复调用 LLM 浪费 token。

---

## 第 12 部分：技术栈对照与选型清单

| 模块 | 原项目 | Spring AI 2.0 落地 |
|---|---|---|
| 语言 | Python 3.11+ | Java 21 LTS |
| Web 框架 | FastAPI | Spring Boot 4 + Spring MVC/WebFlux |
| Agent 框架 | LangGraph + LangChain | Spring AI 2.0 `ChatClient` + 自研 WorkflowGraph |
| LLM SDK | langchain-tongyi (DashScope) | Spring AI DashScope starter（2.0 已官方支持） |
| 工作流引擎 | LangGraph | 自研 `WorkflowGraph`（基于 Reactor） |
| 向量库 | pymilvus | Spring AI `MilvusVectorStore` |
| 关系数据库 | psycopg2 / SQLAlchemy | Spring Data JPA + Hibernate 7 + JDBC |
| 缓存 | （未明确） | Spring Data Redis |
| 搜索 API | Bocha（自封装） | Spring 4 HTTP Interface + WebClient |
| 记忆系统 | 自实现 MemoryManager | Spring AI `ChatMemory` + 自定义扩展 |
| 结构化输出 | 自实现 JSON 解析 | `StructuredOutputConverter` / `BeanOutputConverter` |
| 工具调用 | LangChain tools | Spring AI `@Tool` + `ToolCallback` |
| 可观测 | logging | Micrometer + OpenTelemetry |
| 配置 | .env + config.json | Spring Boot `@ConfigurationProperties` |
| 打包 | pip + requirements.txt | Maven / Gradle + Spring Boot AOT |

---

## 第 13 部分：业务需求验收清单（Acceptance Checklist）

为便于 AI 实现后验证，给出可测试的验收点：

### 13.1 功能性需求

- [ ] 输入"你好" → 走 direct_answer，不走多 Agent 流程；
- [ ] 输入"调查 2026 年 AI 产品趋势" → 走 multiagent 流程，最终输出 2000+ 字报告；
- [ ] 报告中所有 `[XXX]` 引用 ID 均出现在末尾参考资料列表中；
- [ ] 参考资料 list 中无重复项（LOCAL 按 locator 去重）；
- [ ] evidence_pool 中本地证据 reliability_score=0.92；官方域名=0.88；主流媒体=0.72；
- [ ] analyze 节点输出 needs_more_research=true 时，会触发 reflect → 二次检索；
- [ ] iteration 达到 max_iterations=3 时，强制进入 write；
- [ ] web_search 与 local_rag 并行执行，deep_dive 等待两者完成；
- [ ] 同一用户跨会话查询时，能从记忆系统中读取历史上下文；
- [ ] 文档导入后能被 local_rag 检索到。

### 13.2 非功能性需求

- [ ] 单次研究任务平均耗时 < 90 秒；
- [ ] 接口支持 SSE 流式输出 Writer 阶段文本；
- [ ] LLM 输出非法 JSON 时节点不崩溃，使用 fallback；
- [ ] 节点失败自动重试 3 次，仍失败则标记任务 FAILED；
- [ ] 所有节点输入输出可在 `node_trace` 表回放；
- [ ] 支持 tenant_id 多租户隔离；
- [ ] 接口支持 JWT 认证；
- [ ] 关键指标（节点耗时、token 消耗、迭代次数）接入 Prometheus；
- [ ] 一键 Docker Compose 启动全部依赖；
- [ ] 提供 OpenAPI 3 文档（Spring Boot 4 自动生成）。

---

## 第 14 部分：实施路线图建议（给落地工程师）

**阶段 1（MVP，1-2 周）**：基础设施 + 单 Agent 链路
- Spring Boot 4 工程骨架、配置系统、PostgreSQL/Milvus/Redis 接入；
- 实现 IntentRouter + Planner + WebScout（不含并行）；
- 单链路 CLI 跑通：query → intent → plan → web_search → 简单输出。

**阶段 2（核心，2-3 周）**：完整工作流
- 自研 `WorkflowGraph` 含并行 + 条件路由；
- 补齐 LocalRAG、EvidenceJudge、Analyst、Reflect、Writer；
- 实现引用验证与参考资料渲染；
- 多轮迭代机制跑通。

**阶段 3（生产化，2 周）**：记忆系统 + 接口 + 可观测
- 三层记忆系统；
- REST + SSE + WebSocket 接口；
- Micrometer + OTel 可观测；
- 多租户与安全。

**阶段 4（增强，2 周）**：报告模板、PDF/Word 导出、域名信誉库、冲突仲裁、报告质量评估。

---

## 结语

本报告将原 Python 项目的业务逻辑、数据结构、Prompt、算法阈值、回退策略完整提炼为 14 个章节的规格说明，并针对 Spring AI 2.0 GA + Spring Boot 4 的技术栈差异给出了对应实现路径，对原项目未明确的工程细节（多租户、流式、异步任务、可观测、记忆生命周期、冲突仲裁、安全等）做了显式补充。
将此报告整体提交给 AI 编码助手（如 Claude Code、Cursor、GitHub Copilot Workspace）时，建议按章节顺序逐模块生成，每生成一个 Agent 即配套单元测试，最后做端到端集成测试。**优先实现第 11 部分"关键设计模式"中的 fallback 和白名单校验**，这是保证系统在 LLM 不稳定情况下仍可运行的关键工程护栏。



