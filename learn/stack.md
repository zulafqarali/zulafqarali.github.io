# The Rest of the Stack — Compact Guides + Interview Q&A

GCP · LLM/LangChain · Vue · Kibana · Hadoop · Grafana.
Aim: conversational depth (15–25 min each), not expert depth. Python/FastAPI get the depth — see the other docs.

---

## 1. GCP (Google Cloud Platform)

**Core services to know by name + one-line purpose**

| Service | What it's for |
|---|---|
| Cloud Run | Serverless containers (our default for FastAPI) — scale to zero, revisions |
| Cloud Functions | Event-driven single-purpose functions (Pub/Sub, HTTP, Storage events) |
| GCE / GKE | VMs / managed Kubernetes (when you need clusters) |
| Cloud SQL | Managed Postgres/MySQL (we use it with asyncpg) |
| Cloud Storage (GCS) | Object storage (files, backups, static assets) |
| Pub/Sub | Async messaging — decouple producers/consumers, fan-out |
| Cloud Tasks | Deferred/retryable task execution (vs in-process BackgroundTasks) |
| BigQuery | Serverless data warehouse (analytics on event data) |
| Memorystore | Managed Redis |
| Secret Manager | Secrets, not env files in git |
| IAM + Cloud Logging/Monitoring | Access control, logs, metrics, traces |

**Cloud Run vs GKE (classic question):** Cloud Run = containers only, serverless scale-to-zero, no cluster ops, HTTP-first, concurrency-based scaling — pick it unless you need GPUs, long-lived non-HTTP workloads, DaemonSets, or fine-grained networking.

**Cloud SQL vs BigQuery vs Firestore:** transactional app data (Postgres) vs analytics warehouse (BQ, SQL over object storage) vs document/noSQL mobile-scale (Firestore).

**Hands-on:** `gcloud auth login`, `gcloud run deploy api --image ... --region asia-south1 --allow-unauthenticated`, `gcloud sql instances describe`, IAM roles principle-of-least-privilege.

**Q: How do you connect Cloud Run to Cloud SQL privately?** Unix socket via Cloud SQL Auth connector/library (or sidecar/connector image), private IP/VPC connector — never expose the DB publicly.

**Q: Scale-to-zero — implications?** First request after idle is cold (100ms–seconds). Keep a min instance for user-facing APIs; fine for internal/batch.

**Q: How do you handle retries across services?** Pub/Sub with ack deadlines + dead-letter topics; idempotent consumers (dedupe keys); Cloud Tasks with exponential backoff for user-visible work.

---

## 2. LLMs & LangChain

**Fundamentals (must be fluent):**
- **Tokens** — models price/limit by tokens (~¾ word each). Context window = max tokens in/out.
- **Temperature** — 0 = deterministic (extraction, SQL), 0.3–0.7 = balanced, >0.8 = creative.
- **Chat messages** — system (rules/role), user, assistant; multi-turn = resend history (costly) or summarize.
- **Embeddings** — vectors of text meaning; similarity search = RAG retrieval. Vector DBs: pgvector, Chroma, Pinecone, Vertex AI Vector Search.
- **RAG pipeline**: load → split → embed → store → retrieve (top-k) → build prompt with context → LLM → cite/answer. Add re-ranking for quality.
- **Function/tool calling** — model returns a tool call; your code executes and returns the result. This is how agents work.
- **Evaluation** — golden sets, LLM-as-judge with rubric, regression tests per prompt change; track latency + cost per request.
- **Safety** — prompt injection defenses (untrusted content labeled as data, never as instructions), output filtering, PII redaction, rate limits.

**LangChain shape (current generation):**
- **LCEL** (LangChain Expression Language): `prompt | model | parser` — composable runnable pipelines, `.invoke()`, `.stream()`, `.batch()`.
- Building blocks: chat models (`ChatOpenAI`, `ChatAnthropic`, Vertex models), prompts (templates + few-shot), output parsers (JSON, Pydantic), retrievers, tools, agents (model + tools loop), memory (usually just conversation history in your DB now).
- **LangGraph** — stateful, graph-based agents (nodes = steps, edges = control flow, checkpointing) — the modern answer for "multi-step agent with human-in-the-loop".
- Practical tips: constrain outputs with structured output/`with_structured_output(PydanticModel)`; stream tokens for UX; cache repeated retrievals; keep prompts versioned like code.

**Minimal RAG sketch (FastAPI + LangChain):**
```python
from langchain_openai import ChatOpenAI, OpenAIEmbeddings
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import JsonOutputParser

model = ChatOpenAI(model="gpt-4o-mini", temperature=0)
prompt = ChatPromptTemplate.from_messages([
    ("system", "Answer ONLY from context. If unsure, say you don't know.\nContext: {context}"),
    ("human", "{question}"),
])
chain = prompt | model | JsonOutputParser()
# retrieve context with embeddings + vector store, then: chain.invoke({...})
```

**Q: When is RAG better than fine-tuning?** RAG = fresh/private knowledge, citations, cheap iteration. Fine-tuning = style/format/behavior at scale, stable domain. Start RAG; fine-tune only after prompts+RAG plateau.

**Q: How do you cut LLM costs?** Smaller model per task tier, prompt compression, caching (semantic + exact), shorter history/summaries, batch APIs, cap `max_tokens`.

**Q: How do you prevent hallucination in a product feature?** Ground in retrieved context, force citations, "I don't know" in the system prompt, structured outputs, eval harness + human review for high-stakes paths.

**Q: What is an agent vs a chain?** Chain = fixed steps. Agent = model decides next action (tool calls) until done — flexible, harder to test; use graphs (LangGraph) when the flow is known but branching.

---

## 3. Vue (3)

**Core mental model:** progressive framework; Composition API + `<script setup>` is the modern default.

```vue
<script setup lang="ts">
import { ref, computed, onMounted } from 'vue'
import { useUserStore } from '@/stores/user'

const props = defineProps<{ userId: number }>()
const emit = defineEmits<{ saved: [id: number] }>()
const user = useUserStore()
const count = ref(0)                       // reactive primitive
const double = computed(() => count.value * 2)

onMounted(() => user.load(props.userId))
</script>

<template>
  <button @click="count++">{{ double }}</button>
</template>
```

- **Reactivity:** `ref` (any value, `.value` in script) / `reactive` (object); `computed`, `watch`/`watchEffect`. Vue 3 uses proxies — no more `Vue.set`.
- **Components:** props down, events up (`emit`), slots, provide/inject for deep globals; **Pinia** for store; **Vue Router** for SPA routing; `v-model` = prop + update event sugar.
- **`:key` on lists** — key by stable id, never index, or DOM state (inputs, animations) glitches.
- **Async components + Suspense**, lazy route loading, `defineAsyncComponent` for heavy charts.
- Performance: `v-once`/`v-memo` for static, virtual scrolling for big lists, watch bundle size (route-level code split).

**Q: Options API vs Composition API?** Options = per-option blocks (data/methods), mixins get messy. Composition = functions per concern, better TS inference, reusable logic via composables.

**Q: How do you talk to a FastAPI backend?** `fetch`/axios composable; handle FastAPI's 422 shape (`detail: [{loc, msg}]`); auth via bearer token in interceptor; typed DTOs shared from OpenAPI (openapi-typescript codegen is a nice trick with FastAPI).

---

## 4. Kibana (ELK)

**The stack:** Elasticsearch (search/store) ← Logstash/Beats (ingest) → Kibana (UI). OpenSearch is the AWS-flavored fork; GCP shops may use Cloud Logging instead — concepts transfer.

**Know how to:**
- Create a **data view** (index pattern) over your logs (e.g., `logs-api-*`), pick `@timestamp`.
- **Discover**: filter with KQL (`level: "error" AND service: api`), field sidebar, histogram over time.
- Build **dashboards** (logs per service, error rate, latency percentiles if logged).
- **Alerting** in Kibana (threshold on error count) or ship alerts from elsewhere.
- Mapping pitfalls: dynamic mapping exploding field cardinality (`.keyword` subfields; don't index UUIDs as text).

**Q: How do your FastAPI logs get into Kibana?** JSON logs on stdout → agent (Fluent Bit/Filebeat/DaemonSet) → Elasticsearch (or via Kafka). Include `request_id`, `user_id`, `latency_ms` as fields.

**Q: Kibana vs Grafana?** Kibana = log search/analysis in Elasticsearch, text-first. Grafana = dashboards/alerts over metrics/many datasources. Teams use both: Grafana for RED/SLI dashboards, Kibana to drill into log lines behind a spike.

---

## 5. Hadoop (awareness level)

**Big picture:** commodity-cluster batch processing.
- **HDFS** — distributed filesystem; files split into blocks (e.g., 128 MB) across DataNodes; **NameNode** holds metadata/namespace; replication factor 3 by default for fault tolerance.
- **MapReduce** — batch compute model: map (filter/transform) → shuffle/sort → reduce (aggregate). Written for throughput, not latency.
- **YARN** — resource manager scheduling jobs on the cluster.
- **Around it:** Hive (SQL over HDFS), HBase (NoSQL on HDFS), Spark (in-memory compute, largely replaced classic MapReduce for jobs; can run on YARN).

**When it still comes up:** historical data lakes, legacy pipelines, interviews at data-heavy companies. Modern default on GCP is BigQuery / Dataproc (managed Spark) / Dataflow — mention Hadoop as foundation, propose the managed option.

**Q: Why replication factor 3?** Node/rack failure tolerance; HDFS places copies on different racks. Consistency is write-once-read-many style, tuned for large sequential files.

**Q: MapReduce vs Spark?** MapReduce writes intermediate results to disk per stage; Spark keeps data in memory (RDDs/DataFrames) → 10–100× faster for iterative/interactive jobs.

---

## 6. Grafana

**Role:** visualization + alerting over metrics from many sources (Prometheus, Cloud Monitoring, Elasticsearch, Loki, BigQuery…).

**Know how to:**
- Add a **datasource**, build a **dashboard** with panels (time series, stat, table), use **variables** (`$service`, `$env`) for reusable dashboards.
- **PromQL basics** (if Prometheus is the metrics source): `rate(http_requests_total[5m])`, `histogram_quantile(0.95, sum by (le) (rate(latency_bucket[5m])))`, error ratio `sum(rate(status=~"5..")) / sum(rate(...))`.
- **RED/USE** dashboards: Rate (throughput), Errors, Duration (latency p95) per service — the default backend interview dashboard.
- **Alerting**: threshold rules, duration (avoid flapping), route to Slack/PagerDuty/Opsgenie; alert on symptoms (p95 latency, error rate), not just causes.
- **Instrumentation:** app emits metrics (prometheus-client in Python: counters/histograms) → scrape → Grafana. Or OpenTelemetry for traces feeding the same stack.

**Q: How do you instrument a FastAPI app?** `prometheus-fastapi-instrumentator` (or OTel middleware) for request count/latency/status; custom counters for business events (orders, LLM tokens); exemplars/traces to correlate with logs via `request_id`.

**Q: An alert fired — walk me through triage.** Grafana: which signal (error rate? latency?), when it started, which service/region/version (annotations from deploys). Correlate deploy markers → Kibana logs for the same `request_id` window → downstream dependencies → rollback or mitigation.

---

## Suggested drill for the "breadth" topics

1. Deploy the FastAPI mini-project to Cloud Run with Cloud SQL + Secret Manager (one evening).
2. Add an LLM endpoint with LangChain (`with_structured_output` + one tool) and stream the response.
3. Add Prometheus metrics + a Grafana dashboard (RED) + one alert.
4. Point one service's JSON logs at Elasticsearch and build a Kibana error dashboard.
5. Explain Hadoop's MapReduce to a rubber duck in 5 minutes using a word-count example — done = ready.
