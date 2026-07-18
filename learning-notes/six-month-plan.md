# The 6-Month Plan — From Solid Foundations to Hunted Engineer

> Noor Tantawi — starts after finishing the DDD/CQRS repo (Phase 0)
> Schedule: 3:30 PM – 9:00 PM, 5 days/week. One thread at a time.

---

## The Operating System (Read This Daily Until It's Automatic)

Discipline is not willpower. It is the removal of decisions. Every session runs the same way:

```
3:30 – 3:50   Read yesterday's log line. Open the queued task. No deciding.
3:50 – 6:00   Deep Work Block 1 — BUILD (no videos, no articles, hands on keyboard)
6:00 – 6:30   Break. Away from the screen. Eat something.
6:30 – 8:30   Deep Work Block 2 — build continues, or structured study (course/docs)
8:30 – 9:00   Close-down ritual:
                1. Commit today's work (even broken WIP goes on a branch)
                2. One-sentence log: what I built, what broke, what I now understand
                3. Queue tomorrow's FIRST task in one line
```

**Rules:**
- One thread at a time. The month's theme is the only theme. Everything else waits.
- Friday is different: no new material. Review the week, write, run the weekly checkpoint.
- The daily log lives in a single `LOG.md`. By month 6 it becomes your interview story bank.
- Claude Pro is your senior colleague: paste your code for review, ask it to drill you, have it explain what you're stuck on. Never let it write what you haven't understood.

---

## Phase 0 — Finish What You Started (Now → ~3-4 weeks)

Complete this repo. Not negotiable, no skipping ahead:

- [ ] ADR-004 Outbox (Drill 4 pending right now)
- [ ] ADR-005 Error Management
- [ ] ADR-006 Unobtrusive Integration Events
- [ ] CQRS read side (Dapper, read repositories)
- [ ] Custom mediator + all 4 pipelines
- [ ] Architecture tests deep-dive
- [ ] Integration tests (Testcontainers)
- [ ] **Graduation deliverable:** one merged (or at least submitted) PR to this repo —
      candidate: the `AssetErrors.ConcurrencyConflict` hardcoding bug found in ADR-002

---

## Month 1 — Apply Everything to the SA System

**Theme:** Your ERP becomes a DDD/CQRS system. This is real product work + retention + portfolio, all at once.

**Weeks 1–2: The Write Side**
- Pick your 3 core aggregates: `Invoice`, `Payment`, `JournalEntry`
- Refactor each: private setters, factory methods, invariants inside the aggregate, domain exceptions
- Introduce domain events: `InvoicePostedEvent`, `PaymentReceivedEvent`
- Command handlers touch ONE aggregate; event handlers coordinate the rest (ADR-001 applied)

**Weeks 3–4: Concurrency + Errors + Enforcement**
- RowVersion on every aggregate root (ADR-002 applied)
- Result pattern + typed errors + exception middleware (ADR-005 applied)
- Architecture tests project: enforce "command handlers use only their own context's repositories"
- Outbox table + background processor for email/notification events (ADR-003/004 applied)

**Checkpoint 1 (last Friday):**
- [ ] Two users editing the same invoice → second gets 409. Demonstrated, not assumed.
- [ ] Posting an invoice with a failing journal-entry handler → full rollback. Demonstrated.
- [ ] Architecture test fails when you deliberately inject a cross-context repository.
- [ ] Written: case study #1 — "How I applied DDD to a production ERP" (1-2 pages, your log is the raw material)

---

## Month 2 — Testing + Observability (The Production-Minded Engineer Signal)

**Theme:** You can prove your system works and see inside it while it runs.

**Weeks 1–2: The Test Pyramid, For Real**
- Unit tests on aggregates: every invariant has a test that tries to break it
- Integration tests with `WebApplicationFactory` + Testcontainers (real SQL Server in Docker)
- Test the pipeline end-to-end: HTTP → command → event handlers → DB → response
- Coursera support: search "automated testing C#" / use xUnit docs as primary source

**Weeks 3–4: The Three Pillars**
- Serilog structured logging + Correlation-ID middleware (React request → API → SQL, one ID)
- OpenTelemetry: traces exported to Jaeger (or Seq), see the full request waterfall
- Metrics + health checks endpoint
- Coursera support: any OpenTelemetry/observability fundamentals course

**Checkpoint 2:**
- [ ] A slow endpoint diagnosed using ONLY the trace (which span ate the time?)
- [ ] CI-runnable test suite: `dotnet test` green from a clean clone
- [ ] Case study #2: "Tracing a request through my ERP"

---

## Month 3 — Containers + CI/CD (Ship It Like a Team Would)

**Theme:** From "works on my machine" to reproducible, automated delivery.

**Weeks 1–2: Docker Properly**
- Multi-stage Dockerfile for the API (build → runtime, small final image)
- docker-compose: API + SQL Server + Seq/Jaeger, one command brings up everything
- .dockerignore, layer caching, image scanning with `docker scout` or `trivy`
- Coursera support: "Docker for Developers" style course, first half only — then build

**Weeks 3–4: The Pipeline**
- GitHub Actions: build → test (with Testcontainers) → publish image on every push
- DB migrations run as a pipeline step (DbUp or EF migrations bundle)
- Branch protection: nothing merges red
- Versioned releases: tag → image with that tag

**Checkpoint 3:**
- [ ] Fresh machine test: clone → `docker compose up` → working system in under 5 minutes
- [ ] A PR with a failing test cannot merge. Demonstrated.
- [ ] Case study #3: "My ERP's path from commit to running container"

---

## Month 4 — Kubernetes Fundamentals + Real Messaging

**Theme:** Orchestration without the buzzwords. Your system on k8s, your outbox on a real broker.

**Weeks 1–2: K8s Core (local kind/minikube cluster)**
- Deployment, Service, Ingress for the SA API
- ConfigMaps + Secrets (and why secrets in Git end careers)
- Liveness/readiness probes — simulate a hung startup, watch k8s restart it
- Resource requests/limits; HPA with a k6 load test to find the scale point
- Graceful shutdown: SIGTERM → finish in-flight unit of work → clean exit

**Weeks 3–4: RabbitMQ + Outbox Evolution**
- RabbitMQ in the cluster; outbox processor publishes to the broker instead of in-process
- Consumer service (can be the same app, different mode) with idempotent handlers
- Kill RabbitMQ for 5 minutes during activity → zero lost events after recovery. This is ADR-004's promise, proven by you.

**Checkpoint 4:**
- [ ] Zero-downtime deploy: new version rolls out during a k6 load test, no 5xx errors
- [ ] The RabbitMQ kill test passes
- [ ] Case study #4: "Guaranteed delivery: outbox + broker in production shape"

---

## Month 5 — AI Engineering (Your Excitement, Channeled Properly)

**Theme:** Build with LLMs like an engineer, not a prompt hobbyist. Claude Pro is your lab.

**Weeks 1–2: Foundations + Tool Use**
- Claude API fundamentals: messages, system prompts, streaming, token economics
- Tool use / function calling: give Claude tools and reason about the loop
- Structured outputs, error handling, retries — treat the LLM as an unreliable dependency (apply your resilience thinking!)
- Coursera support: DeepLearning.AI generative-AI / LLM application courses
- Build: a CLI assistant that answers questions about SA data via tool calls

**Weeks 3–4: MCP + Agent Harnesses + Evals**
- Build an MCP server exposing your SA system: `get_invoice`, `search_customers`, `revenue_summary` — Claude becomes a natural-language interface to your ERP
- Understand the harness layer: context management, tool permissioning, agent loops (you use Claude Code daily — now study how it's built)
- Evals: a small harness that runs 20 test questions against your MCP server and scores answers. AI without evals is a demo; with evals it's engineering.
- Security thinking: prompt injection against your own MCP server — try to make it leak or mutate data it shouldn't

**Checkpoint 5:**
- [ ] Working MCP server against real SA data, demoable in 2 minutes
- [ ] Eval suite with pass-rate tracked across 3 iterations of your prompts/tools
- [ ] Case study #5: "I gave my ERP a natural-language interface" — this one gets attention

---

## Month 6 — Security, System Design, and the Hunt

**Theme:** Harden what you built, learn to talk about it, make yourself findable.

**Weeks 1–2: Security Essentials**
- OAuth2/OIDC properly: authorization code + PKCE flow, JWT validation, refresh rotation
- Secrets out of appsettings: environment/key-vault pattern
- OWASP Top 10 pass over the SA system: injection, broken access control, SSRF
- Rate limiting + input validation at the API boundary

**Weeks 3–4: System Design + Portfolio Offensive**
- System design practice: 2 designs/week on paper (a payment system, a notification system) — defend aggregate boundaries, consistency choices, failure modes. Use Claude as the interviewer.
- Consolidate: 5 case studies polished and published (blog/GitHub/portfolio site)
- CV + LinkedIn rewritten around evidence: "built, hardened, containerized, orchestrated, and AI-enabled a production ERP"
- Second open-source PR (this repo or another .NET project you now understand)
- Mock interviews: 2 technical + 1 system design, recorded, reviewed

**Final Checkpoint:**
- [ ] 30-minute walkthrough of your SA system's architecture, delivered cold, no notes
- [ ] Portfolio: 5 case studies + 2 OSS PRs + demoable AI feature
- [ ] You can answer "why domain events over orchestration?", "what does your outbox guarantee and what doesn't it?", "how would you scale reads?" — from experience, not memory

---

## Resource Map

| Resource | Use for |
|---|---|
| **Claude Pro** | Daily code review, drills, mock interviews, MCP/agent building in Month 5 |
| **Coursera** | Structured fill-ins: testing (M2), Docker (M3), LLM courses (M5) — always second half of the evening, never instead of building |
| **This repo** | The reference implementation you return to when unsure |
| **Your SA system** | The lab. Every month's output lands here. |
| **Official docs** | EF Core, xUnit, OpenTelemetry, k8s, RabbitMQ, Anthropic docs — primary sources over tutorials |
| **LOG.md** | One line per day. Your future interview answers. |

---

## The Contract With Yourself

1. The month's theme is the only theme.
2. Build first, study second. Every evening starts hands-on-keyboard.
3. Friday: review + write, never new material.
4. Nothing counts until it's demonstrated (checkpoints are demos, not feelings).
5. When motivation dips — and it will, around week 3 of every month — the system carries you: open the log, do the queued task, nothing more.

Six months from now the goal isn't "I studied Kubernetes." It's:
**"Here is a production ERP I built, hardened, tested, containerized, orchestrated, made observable, wired to a message broker, and gave an AI interface — and here are five write-ups proving I understood every decision."**

That is what gets you hunted.
