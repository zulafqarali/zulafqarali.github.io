# Python · FastAPI · LLM Interview Learning Guide

Study material for interviewing on this stack: **Python, FastAPI, GCP, LLM/LangChain, Vue, Kibana, Hadoop, Grafana**.
Main focus (≈70% of interview time in most backend roles): **Python + FastAPI**.

## Contents

| File | What it covers |
|---|---|
| [python.md](python.md) | Python deep-dive tutorial + interview Q&A (language, OOP, async, GIL, typing, testing, performance) |
| [fastapi.md](fastapi.md) | FastAPI hands-on tutorial + interview Q&A (params, Pydantic, DI, auth, DB, testing, deployment) |
| [stack.md](stack.md) | Compact guides + Q&A for GCP, LLM/LangChain, Vue, Kibana, Hadoop, Grafana |

## 4-week study plan

**Week 1 — Python core (python.md §1–§8)**
- 1–2h/day: read a section, type the examples yourself (no copy-paste).
- Daily: 5–10 interview questions from the Q&A at the bottom of each section.
- Build: a small CLI tool using argparse + requests + a SQLite database.

**Week 2 — Python advanced + FastAPI basics (python.md §9–§12, fastapi.md §1–§6)**
- Focus: async/await, GIL, generators, decorators — the questions that separate candidates.
- Build: a CRUD FastAPI service with Pydantic models and in-memory storage.

**Week 3 — FastAPI production skills (fastapi.md §7–§12)**
- JWT auth, SQLAlchemy async, dependency overrides, tests with `TestClient`, Docker, Cloud Run deploy.
- Build: same CRUD service, now with Postgres/Cloud SQL, auth, and tests.

**Week 4 — Breadth + mock interviews**
- One day each: GCP, LLM/LangChain (build a RAG chat endpoint), Vue basics, observability (Kibana/Grafana), Hadoop awareness.
- 3+ full mock interviews (explain out loud, whiteboard a design, code a small endpoint under time pressure).

## How to use the Q&A sections

1. Answer out loud **before** reading the answer.
2. Follow every answer with "…and where have I used this / trade-offs?" — interviewers probe with "why".
3. For coding questions, run the code. Knowing >7% of theory, running > knowing.

## Interview day playbook

- **Clarify before coding**: inputs, scale, error cases, expected latency.
- **Talk through trade-offs** (e.g., "sync endpoint is fine here because the DB driver is sync — or I'd use the async driver to not block the loop").
- **Know your resume**: every stack item on the resume-python.html is fair game; be honest about depth ("PHP is my 9-year production language; Python/FastAPI is what I build and study daily now").
- **System design basics to rehearse**: rate limiting, caching layers, queue-based fan-out, idempotency, observability triage with logs (Kibana) + metrics (Grafana).

## Suggested practice projects (in interview order of usefulness)

1. **URL shortener** — FastAPI + Postgres, auth, rate limit, tests. Covers 80% of backend interview topics.
2. **LLM chat/RAG API** — FastAPI + LangChain + vector store; streaming responses. Covers the AI-angle roles.
3. **Observability demo** — add structured logs + Prometheus metrics + a Grafana dashboard to project 1.
