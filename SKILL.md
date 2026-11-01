---
name: system-design
description: Use before writing code for any new app, service, or major feature (or when someone says "design", "architecture", "how should I build", "plan this system") to interview the person stage by stage, size the design to their real scale, record trade-offs, and produce an approved DESIGN.md that the coding session builds from.
---

# System design before code

You are running a design session, not writing code. Your job is to help the person
think clearly, make the decisions that are expensive to change later, write them down,
and stop. Code starts only after they approve `DESIGN.md`.

Why this exists: AI coding sessions usually go wrong in two ways. The agent builds
something different from what the person had in mind because it never had the missing
information, or it says "done" with nothing to check against. A short interview that
ends in a written spec and a test list fixes both.

## Hard rules

1. **No code until approval.** No scaffolding, no `npm init`, no "quick prototype".
   Schemas and API shapes inside DESIGN.md are fine. If asked to skip ahead, say what
   is still undecided and offer to fill it with defaults in one step.
2. **Ask 2-3 questions at a time, never more.** Each question comes with your proposed
   default and a one-line reason, so the person can answer "yes" and move on.
3. **Confirm before moving to the next stage.** End each stage with a 2-4 line summary
   and "Correct? Anything to change?".
4. **Numbers drive choices.** Every component you add must point to a requirement or a
   number from Stage 2. If it can't, leave it out.
5. **Every real choice gets a decision record** (format below).
6. **Right-size.** Design for the stated scale plus about one order of magnitude of
   headroom, not for "millions of users" nobody asked for.
7. **Never invent facts about their world.** If you don't know their traffic, budget, or
   stack, ask or label your guess `ASSUMPTION:` and list it in DESIGN.md.

## When to use, and when to keep it light

Use it for: a new app or service, a feature that adds a new data store, queue, external
integration or public API, or a rewrite.

Go light (Stages 1, 3, 4 only, about 10 minutes) for: scripts, CLIs, single-user tools,
hackathon demos, internal tools for fewer than about 20 people.

Skip it for: bug fixes, refactors with no new components, copy or UI tweaks.

If unsure, ask: "Is this a quick build or something people will rely on? I can do a
10-minute light design or the full pass."

## How to run the session

**Start (one message):**
> Before we write code, I'll ask a few questions in 8 short stages, 2-3 at a time, each
> with a suggested answer. Say "default" to accept my suggestion. At the end you get a
> DESIGN.md to approve, and the build starts from that.
> First: what are you building, in one or two sentences, and who is it for?

**Then, right after Stage 1, set the project tier** (see Right-sizing) and tell the person
which stages are Needed, Optional, or Skip. Let them override.

**Each turn:**
- Ask up to 3 questions from the current stage, each as:
  `Q: ...?  Suggested: ... (because ...)`
- Prefer multiple choice when options are known: `(a) ... (b) ... (c) other`.
- If they answer "default" or "idk", take the default and tag it `ASSUMPTION:`.
- If an answer contradicts an earlier one, point it out and ask which wins.
- If they want heavier tech than the numbers justify ("shouldn't I use Postgres/Kafka?"),
  don't just refuse: show the number it would take and write that as the decision's
  "Revisit when" trigger. If they still want it, record it as their choice with the cost.
- When a stage is done, summarize it, record any decisions, confirm, move on.

**Teach while asking.** Beginners won't know terms like "idempotency" or "p99". Explain any
term in one plain clause the first time you use it, e.g. "idempotent (safe to run twice
without doing the thing twice)". Don't lecture beyond that.

**Keep a running draft.** After each confirmed stage, update a draft of DESIGN.md in your
head (or in a scratch file, if file writes are allowed) so nothing is lost.

## Right-sizing (set the tier after Stage 1)

| Tier | Signals | Architecture default | Stages |
|---|---|---|---|
| **T0 Toy / personal** | 1 user, demo, script | One process, SQLite or files, no auth or simple auth | 1, 3, 4 needed; rest skip |
| **T1 Side project / MVP** | under ~10k users, 1-2 builders, small budget | One deployable app (monolith) + one SQL DB (SQLite or Postgres), managed hosting, backups on | 1-4 needed; 5-7 light (1-2 questions each); 8 needed (auth + basic logs) |
| **T2 Growing product** | 10k-1M users, small team, paying users or an uptime promise | Monolith + Postgres + cache where reads are hot + background job queue for slow work; 2+ app instances behind a load balancer | All needed |
| **T3 Large / critical** | 1M+ users, money or safety, multiple teams | Everything above, plus replication, failover plans, SLOs, maybe service splits along proven boundaries | All needed, deep |

Facts behind the defaults:
- Start with a monolith. Fowler: almost all successful microservice systems started as a
  monolith that got too big; systems built as microservices from scratch often ended up in
  serious trouble.
- SQLite's own guidance: sites with fewer than 100K hits/day should work fine (that figure
  is conservative). Move to a client/server DB (e.g. Postgres) when several servers must
  write, the workload is write-heavy, or data heads toward terabytes.
- Scale iteratively: load test, find the bottleneck, fix it, repeat. Don't pre-build for
  bottlenecks you haven't measured.

Tier mistakes to catch: T1 with Kafka, Kubernetes, sharding, microservices, multi-region.
T2+ with no backups, no auth design, or "we'll add monitoring later".

## The 8 stages: question bank

Pick the 2-3 most important open questions per turn. You don't need to ask every one.
Defaults below assume T1; adjust for the tier.

### Stage 1: Define the problem (always)
Goal: one paragraph anyone could build from.
- Who are the users, and what are the 3 most important things they do? *Suggested: list
  3 core use cases as "A user can..."*
- What is explicitly out of scope for v1? *Suggested: anything not in the 3 core use cases.*
- Constraints: deadline, budget per month, team size, must-use stack or host? *Suggested:
  whatever stack the builder already knows best.*
- Non-functional needs: how fast must it feel, how bad is an outage, how bad is losing
  data? *Suggested: pages respond in under ~300 ms typical; a few minutes of downtime is OK;
  losing user data is not OK.*
- How will we know v1 worked? *Suggested: one measurable success criterion.*

Output: users, use cases, functional requirements, non-functional requirements,
constraints, assumptions, out-of-scope, success criteria.

### Stage 2: Estimate the scale (T1+; for T0 one line is enough)
Goal: a small table of numbers that later stages point to. Show your working.
- How many users in the first year, and how many active per day? *Suggested: DAU about
  10-20% of total users if unknown. ASSUMPTION.*
- How many of each core action does an active user do per day? Which are reads vs writes?
- What is stored per write, roughly (bytes, or files and their size)? How long is it kept?
- Is traffic spiky (launches, schedules, a viral link)? *Suggested: peak = 2x average.
  ASSUMPTION.* If one event can cause a spike far above that (a viral link, a launch, a
  daily batch), write it as its own row ("spike: ~N req/s for minutes"). It is often the
  only number that changes a decision.

Run the capacity math below and show a table. Then say what the numbers mean in plain words,
e.g. "That's about 1 request per second. A single small server handles this; no cache or
queue needed yet."

### Stage 3: Map the system (always)
Goal: the boxes and arrows, plus the API surface.
- Clients: web, mobile, CLI, other services? *Suggested: web only for v1.*
- What are the main API calls? List each as `METHOD /path -> what it does`.
- Anything slow or external in a request (emails, payments, LLM calls, image processing)?
  *Suggested: if it takes more than about a second, or calls a third party that can fail,
  move it to a background job.*

Default shape: client -> one app (API + business logic) -> one database. Add a cache,
queue, worker, object storage or CDN only when a requirement or a Stage 2 number calls for it.
Standard building blocks to choose from: database, cache, search index, queue + worker,
batch job, object storage, CDN.

### Stage 4: Store and retrieve (always)
Goal: the schema and how every core use case reads and writes it.
- What are the main entities and how do they relate? Write the tables with columns, types,
  primary keys, and unique constraints.
- What are the most common queries? Each one needs an index or a primary-key lookup.
- Relational or key-value/document? *Suggested: relational (SQLite/Postgres) unless the
  data is truly key -> blob with no joins and huge scale. Relational keeps options open.*
- Retention and deletion: what is deleted, when, and can users delete their own data?
- Backups: *Suggested: automated daily backups and one tested restore before launch.*

Skip partitioning/sharding unless Stage 2 storage is approaching what one database server
can hold or write rate exceeds what one primary can take, and say which number triggered it.
Replication: T1 none (backups instead); T2 a read replica only if reads are the bottleneck;
T3 primary + standby with failover.

### Stage 5: Handle traffic (T1 light; T2+ needed)
Goal: say how load is absorbed and how abuse is stopped.
- Peak requests/sec from Stage 2: can one instance handle it? *Suggested for T1: one
  instance, scale vertically first (bigger machine).* Say "verify with a load test" rather
  than quoting a per-server number you don't know.
- Static files and media: *Suggested: serve via the host's CDN or object storage.*
- Hot reads: is the same data read far more than it changes? If so, cache it, and state
  what invalidates the cache.
- Rate limiting: *Suggested: per-IP and per-account limits on writes and logins.* Always
  needed on any public write endpoint, even at T1.
- Backpressure: if a queue or worker falls behind, what happens? *Suggested: bound the
  queue; when full, return 503 and have clients retry with backoff.*

Autoscaling and multiple load balancers: T2+ only.

### Stage 6: Coordinate work (skip if there is no background or external work)
Goal: say what is synchronous, what is async, and how failures are retried safely.
- Which steps must the user wait for? *Suggested: only the ones whose result they see
  immediately. Cheap work and realtime flows stay synchronous; queues add delay and
  complexity.*
- For async jobs: what queue? *Suggested T1: a jobs table in the same database or the
  framework's built-in job runner. Dedicated brokers at T2+.*
- Retries: *Suggested: timeout on every remote call; at most 3 retries with exponential
  backoff and jitter; retry at one layer only.* Stacked retries multiply load (5 layers x 3
  retries = 243x on the database).
- Idempotency: anything that charges money, sends a message, or creates a record and can be
  retried must take an idempotency key (a unique ID per operation, so a retry doesn't do it
  twice).
- Ordering: does order matter? If yes, say per what key (per user, per order).
- Poison messages: *Suggested: after N failed attempts move the job to a dead-letter list
  and alert.*
- LLM calls: a frontier model's time to first token is about 1 s and a reasoning call about
  30 s. Treat them like slow external APIs: timeout, retry limit, often async with progress UI.

### Stage 7: Make it reliable (T1 light; T2+ needed)
Goal: an honest promise and a plan for the likely failures.
- What availability do you actually need? Use the downtime table below. *Suggested T1:
  99% or "best effort"; T2: 99.9%.* Don't promise 100%.
- Write 1-2 SLOs in this form: "99% of `GET /x` complete in < 200 ms over 30 days".
  Use percentiles, not averages. Start loose; tighten later.
- What fails first and what happens then? Walk through: app crash, DB down, third-party API
  down, deploy goes bad. For each: user impact, and the recovery (restart, rollback, restore).
- Data: RPO (how much data you can lose) and RTO (how long to recover). *Suggested T1: up to
  24 h of data via daily backups, a few hours to restore.*
- Consistency: where must a read see the latest write (balances, inventory, permissions)?
  Where is a few seconds stale OK (counters, feeds)?

Redundancy math: parts in sequence multiply (two 99.9% parts in a chain = 99.8%); redundant
copies in parallel raise it (two 99.9% in parallel = 99.9999%). Every new dependency in the
request path lowers availability.

### Stage 8: Operate and protect (always; depth by tier)
Goal: who can do what, what's protected, and how you'll know it's broken.
- Authentication: *Suggested: the framework's or a managed provider's auth; never hand-roll
  password storage.*
- Authorization: list each role and what it can read and change. Check ownership on every
  request (a valid ID doesn't mean the caller may access it).
- Secrets: *Suggested: environment variables or the host's secret store; never in the repo.*
- Encryption: HTTPS everywhere; encryption at rest via the managed DB/host.
- Input: validate on the server with allowlists, length and size limits before parsing,
  parameterized queries. Validate data from queues and internal services too.
- Observability: the four golden signals per user-facing endpoint: latency (as
  percentiles, errors tracked separately), traffic, errors, saturation. Structured logs
  without secrets or full request bodies. One alert per thing a human must act on.
  Tracing at T2+.
- Abuse: what can a malicious user do with each endpoint (spam, scraping, enumeration,
  cost blow-up)? Add one control per realistic abuse.

## Capacity-math cheat sheet

Recipe (write units on every line, round freely):
```
writes/day   = DAU x writes per user per day
reads/day    = DAU x reads per user per day
avg RPS      = (reads/day + writes/day) / 86,400        (seconds per day)
peak RPS     = avg RPS x peak factor                    (default 2x, ASSUMPTION)
storage/year = writes/day x bytes per write x 365       (+ indexes; roughly x2 to be safe, ASSUMPTION)
bandwidth    = peak RPS x average response size         (bytes/s; x8 for bits/s)
cache size   = hot items x item size                    (only if caching)
```
Monthly shortcut: about 2.5 million seconds per month, so 1 req/s ~ 2.5 M req/month,
40 req/s ~ 100 M/month, 400 req/s ~ 1 B/month.

Sizes: 2^10 ~ 1 thousand (KB), 2^20 ~ 1 million (MB), 2^30 ~ 1 billion (GB), 2^40 ~ 1 trillion (TB).

Latency reference (Jeff Dean / Norvig numbers, roughly 2012 hardware; use for ratios, not SLAs):
| Operation | Time |
|---|---|
| Main memory reference | 100 ns |
| Read 4 KB randomly from SSD | 150 us |
| Read 1 MB sequentially from memory | 250 us |
| Round trip within same datacenter | 500 us |
| Read 1 MB sequentially from SSD | 1 ms |
| Disk seek | 10 ms |
| Read 1 MB sequentially from disk | 20 ms |
| Packet California -> Netherlands -> California | 150 ms |
| Frontier LLM time to first token (2026) | ~1 s |
| Frontier LLM reasoning response (2026) | ~30 s |

Takeaways: memory beats SSD beats disk beats network beats other continents. One cross-world
round trip costs about 300x a same-datacenter round trip, so chatty APIs across regions hurt.

Downtime budgets:
| Availability | Per month | Per year |
|---|---|---|
| 99% | ~7.3 h | ~3.65 days |
| 99.9% | ~43.8 min | ~8.8 h |
| 99.99% | ~4.4 min | ~52.6 min |
| 99.999% | ~26 s | ~5.3 min |

After the table, always translate: "~X req/s at peak means ...". If the numbers are tiny,
say so plainly; that is the most useful finding for most projects.
Don't quote how many requests a server can handle; you don't know their code. Say what
load test would confirm it.

## Decision record format

For each real choice (one where a reasonable person could pick differently), write:
```
### D<n>: <short noun phrase>            Status: proposed | accepted | superseded by D<m>
Context: the requirement or number that forces a choice (cite Stage 1/2 items).
Options: (a) ... (b) ... (c) ...
Decision: We will ...
Why: 1-3 reasons tied to the context.
Given up: what we lose or risk by not picking the others.
Revisit when: the measurable trigger that would change this (e.g. "writes > 50/s", "second team joins").
```
One-line summary form (for the decisions table): `"For <context>, we chose <option> over <alternatives> to get <benefit>, accepting <cost>."`
Keep superseded decisions; mark them, don't delete them.

Typical decisions to expect: database engine, monolith vs services, hosting, sync vs async
for each slow step, auth approach, ID/key scheme, caching or not, backup/restore approach.

## Diagram step

After Stage 8, generate one Mermaid `flowchart LR` from the decisions. Rules:
- Only components that exist in the decisions. No "future" boxes; list those in text instead.
- Node IDs are plain words (`App`, `DB`); put labels in brackets with double quotes when they
  contain spaces or symbols: `App["Web app (Node)"]`.
- Use shapes consistently: `[( )]` for data stores, `[ ]` for services, `(( ))` for users.
- Label arrows with what flows: `App -->|"SELECT by code"| DB`. Use one edge per
  request/response pair (put the response in the label) to avoid tangled layouts.
- Use `subgraph` for a hosting boundary if it helps; keep under ~12 nodes.
- If a renderer is available, render it to check syntax (for example `npx -y @mermaid-js/mermaid-cli -i d.mmd -o d.svg`).
  If not, re-read it line by line for unbalanced brackets and unquoted special characters.

For T2+, add one `sequenceDiagram` for the most important request path.

## Pre-mortem (run before showing DESIGN.md)

Imagine it's 6 months later and this failed. Answer each in one or two lines; fix the design
or record the risk:
1. **Single point of failure:** which one component, if it dies, takes everything down? Is
   that acceptable at this tier? What is the recovery step and how long does it take?
2. **10x traffic:** with Stage 2 numbers x10, what saturates first (DB writes, connections,
   disk, a third-party rate limit, cost)? What's the first change you'd make?
3. **Bad input:** what happens with an oversized body, a malformed or malicious URL, a script
   in a text field, someone else's ID, or 1,000 requests a second from one client? What leaks
   (data, money, other users' info, your API bill)?
4. **Retry storms and duplicates:** can any retry double-charge, double-send or double-create?
5. **Data loss:** if the database disk is gone right now, what is lost and how do you restore?
6. **Cost:** what grows linearly with usage and could surprise the budget?
7. **Over-engineering check:** remove each component in turn. If no requirement or number
   breaks, delete it.

## DESIGN.md template

Write this file at the repo root (or where the person says). Keep it short: one or two screens
per section at most.
```
# <Project> design            Status: DRAFT | APPROVED (<date>)
Tier: T<n> (<why>)

## 1. Problem
<one paragraph>  Users: ...  Core use cases: 1. A user can ... 2. ... 3. ...
Out of scope (v1): ...
Success criteria: ...

## 2. Requirements
Functional: FR1 ..., FR2 ...
Non-functional: NFR1 latency ..., NFR2 availability ..., NFR3 data loss ...
Constraints: budget, deadline, team, stack
Assumptions: every ASSUMPTION tag from the interview

## 3. Numbers
| Quantity | Value | Working |
...
What this means: <plain words>

## 4. Architecture
<Mermaid diagram>
Components: one line each on its job and why it exists (link FR/NFR/number).
API: METHOD /path -> purpose, request, response, errors.

## 5. Data model
Tables/collections with columns, keys, indexes; retention; backups.

## 6. Decisions
Summary table (D#, decision, one-line Y-statement), then each full record.

## 7. Reliability and operations
SLOs, failure modes table (failure | impact | detection | recovery), RPO/RTO,
auth/authz, secrets, input validation, observability (golden signals), alerts.

## 8. Risks and open questions
From the pre-mortem: risk | likelihood | impact | mitigation or accepted.

## 9. Build order
Numbered steps, each small enough for one coding session and testable on its own.
Step 1 is a walking skeleton: the thinnest end-to-end path deployed.

## 10. Test list (acceptance checks)
Each FR and NFR maps to at least one check, e.g. "FR2: GET /abc123 returns 302 to the stored URL".
Include the bad-input and duplicate cases from the pre-mortem, at least one failure-path
check (dependency down, write fails), the load test for any latency SLO, and a backup restore.
```

## Hand-off

1. Show DESIGN.md and ask: "Approve as is, or what should change?" Apply changes and ask again.
2. On approval, set `Status: APPROVED (<date>)`, and only then offer to start coding.
3. Tell the coding session (or yourself, in a fresh session if possible): build from DESIGN.md,
   follow the build order one step at a time, run the test list as the definition of done,
   and don't add components that aren't in the design.
4. If coding reveals a design problem, stop, add or supersede a decision record, get a
   quick OK, then continue. Don't silently drift from the design.

## Common mistakes (yours and the person's)

- Jumping to a solution before the use cases are clear.
- Asking 10 questions at once; people stop answering. Ask 2-3.
- Big-company architecture for a small project (Kafka, Kubernetes, microservices, sharding
  for under ~10k users).
- Numbers that never get used. If a number doesn't change a decision, say so and move on.
- Averages instead of percentiles for latency; ignoring peaks.
- No idempotency on anything that charges, sends, or creates.
- Retries at every layer; no timeouts.
- Caching without saying what invalidates it.
- "Add monitoring later"; no backups or never testing a restore.
- Authorization forgotten (logged in != allowed to see this record).
- Validating only in the browser.
- A design doc so long nobody reads or updates it.
- Treating defaults as facts. Defaults are starting points; tag them ASSUMPTION and let the
  person overrule them.
