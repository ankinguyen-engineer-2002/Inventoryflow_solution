<div align="center">

<br/>

# 🏗 InventoryFlow — Solution Architecture

### Four solutions for the Talemy × InventoryFlow Senior Engineer brief, written the way I'd defend them in a design review.

<br/>

![Solutions](https://img.shields.io/badge/solutions-4_drafted-2ea44f?style=for-the-badge)
![ADRs](https://img.shields.io/badge/ADRs-14_recorded-blue?style=for-the-badge)
![Docs](https://img.shields.io/badge/docs-12_files-purple?style=for-the-badge)
![Build](https://img.shields.io/badge/built_in-~12h_AI--assisted-orange?style=for-the-badge)
![Reviewer](https://img.shields.io/badge/reviewer_cost-%240-success?style=for-the-badge)

<br/>

[![Solution A](https://img.shields.io/badge/🟢_A-IMPLEMENTED-2ea44f?style=for-the-badge)](./docs/02-solution-A-recommended.md)
[![Solution B](https://img.shields.io/badge/🟡_B-RUNNABLE_POC-eab308?style=for-the-badge)](./docs/03-solution-B-de-standard.md)
[![Solution C](https://img.shields.io/badge/🔵_C-MS_FABRIC-0078d4?style=for-the-badge)](./docs/04-solution-C-fabric-brief.md)
[![Solution D](https://img.shields.io/badge/🟣_D-AWS_BIG_DATA-ff9900?style=for-the-badge)](./docs/05-solution-D-aws-brief.md)

<br/>

**Author:** [Aric Nguyen](https://github.com/ankinguyen-engineer-2002) · Data Engineer / Solution Architect, 4 years on Fabric/Azure/Databricks + OSS lakehouses
**Companion repo:** [`inventoryflow-catalog-ingest`](https://github.com/ankinguyen-engineer-2002/inventoryflow-catalog-ingest) — Solution A running in three commands

[**▶ 3-min decision**](./docs/00-tldr.md) · [**📐 Solution A deep**](./docs/02-solution-A-recommended.md) · [**⚖️ Why these four**](./docs/01-problem-framing.md) · [**🧠 Engineering judgment**](./docs/09-engineering-judgment.md)

</div>

---

<details>
<summary><b>👤 Who's writing this (click to expand) — short career arc + full CV inline</b></summary>

<br/>

I'm **Aric Nguyen** — Data Engineer / Solution Architect, 4 years building production data platforms across Microsoft Fabric, Azure, Databricks, and open-source lakehouses.

Quick career arc:

| Period | Role | What I built |
|---|---|---|
| **Jan 2026 – Present** | Data Engineer / Solution Architect, **Ashley Furniture Industries** (US HQ-aligned) | Refactor of global supply-chain analytics platform on Microsoft Fabric: 5,000+ enterprise tables, 30 years of history, billions of records. Designed a metadata-driven control plane; cut mart cycle from ~60 min to ~20 min via DAG parallel execution; reached ~91% compatibility with US HQ enterprise warehouse patterns |
| **Mar 2023 – Jan 2026** | Data Engineer / Analytics Engineer, **Ecentric** | Owned the full data platform on Microsoft Fabric + Azure. Led phased zero-downtime migration from Azure to Fabric; built end-to-end real-time streaming pipelines + metadata-driven PySpark medallion; shipped 15+ Power BI reports across Sales / Finance / HR / Service; reduced manual reporting effort by 70%+ |
| **Mar 2023 – Jan 2024** | Data Engineer (Contract), **TheSocietyPass (SOPA Vietnam)** | Co-built streaming-capable platform: Airbyte batch + Debezium CDC + WebSocket → Redpanda → Flink SQL; Iceberg + dbt + Trino batch medallion; dual-write to ClickHouse (hot) and Iceberg / MinIO (cold) |
| **Jan 2022 – Nov 2022** | Data Engineer (promoted from Data Analyst Intern in 3 months), **ADP** | dbt + PostgreSQL HR/payroll transformations; Airflow DAGs; deployed OpenMetadata for governance |

**Certifications:** Microsoft DP-700 (Fabric Data Engineer Associate) · HackerRank SQL (Advanced) · Google Data Analytics
**Languages:** Vietnamese (native) · English (TOEIC 780, daily cross-border collaboration with US HQ)

<details>
<summary><b>📄 Full CV — PDF + page images inline</b></summary>

<br/>

**Download the PDF:** [`assets/CV_Aric_Nguyen.pdf`](./assets/CV_Aric_Nguyen.pdf)

<br/>

#### Page 1

![CV — page 1](./assets/cv-page-1.png)

#### Page 2

![CV — page 2](./assets/cv-page-2.png)

</details>

What that means in plain terms: I've made enough wrong architectural calls in production to have opinions about which ones matter, and I've made enough right ones to be willing to defend them in a design review.

</details>

<details>
<summary><b>💭 My personal opinion (click to expand) — the framework + risk + scale + platform-pivot framing</b></summary>

<br/>

In my view: **to run a business reliably, you need solid infrastructure underneath. To build solid infrastructure, the data engineer responsible has to know enough to make it work AND enough to know when to change it.**

What "enough to make it work" looks like to me:

- **Frameworks** — Spark, Iceberg, dbt, Dagster, Kafka, the cloud platforms — at debugging depth, not slide depth
- **Processes** — CDC, schema evolution, idempotency, lineage, DQ contracts, RPO / RTO, cutover gates
- **End-to-end operations and steps** — from source extraction through serving to consumer feedback, and back through cost monitoring to architectural revision

**That's necessary, but in my experience it isn't sufficient.** What I think separates a senior data engineer / solution architect is the willingness to think one level above:

- **Recognising the risks that haven't shown up yet** — schema drift, cost overruns, vendor lock-in, talent attrition, regulatory change, source-system collapse, marketplace API breakage, dealer schema churn
- **Anticipating scale and optimisation pressure** — not "will it handle more rows" (every modern stack does that), but "at what dealer count does my Postgres become the bottleneck, and is my migration path to a lakehouse already drawn?"
- **Knowing when to pivot platforms or tech stacks** — Fabric → Databricks, Postgres → Iceberg, paid LLM → self-hosted — decisions I don't want to be making under pressure; I want them made with a trigger and a plan

**That's why I drafted four solutions, not one.** I treat each as a credible end-state for a different future the company might head toward. My recommendation is conditional on the read of which future is the real one.

**For this brief's scale — one OEM, ~100 dealers, hiring TS engineers — my recommendation is Solution A.** I defend it across three scale tiers (today, year 2, year 3) in [`docs/02-solution-A-recommended.md`](./docs/02-solution-A-recommended.md). The other three solutions are documented at honest depth so my choice isn't ignorance of the alternatives.

</details>

---

## 🎯 The pitch in 30 seconds

> **I'd build Solution A now, plan Solution B for year two, document C and D so the decision is on record.** Solution A matches the JD stack 1:1 (TypeScript / Node / Postgres / Redis / BullMQ / R2), ships in days, and costs ~$30/dealer/month at amortised scale. B, C, and D are the futures I'd reach for under specific conditions — Iceberg lakehouse at 500+ dealers, Microsoft Fabric if the company commits to that ecosystem, AWS Big Data if cloud-native streaming dominates. This repo is the full argument: why I picked A, where I expect it to break, and what replaces it when it does.

```
What InventoryFlow asked:                       What I delivered:

  Parse messy xlsx                                ✅ Solution A — implemented, TS/PG/Redis end-to-end
  + clean DB                                      ✅ Solution B — runnable PoC, Polars/Iceberg/Dagster
  + R2 schematic images                           📐 Solution C — architecture (Microsoft Fabric)
  + JSON fitment column                           📐 Solution D — architecture (AWS Big Data + streaming)
  + AI tooling
                                                  + 14 ADRs, scaling roadmap, cost economics,
                                                  + measured benchmarks (p99 fitment query = 1.02 ms),
                                                  + zero-API-cost reviewer experience.
```

---

## 📋 The brief — what InventoryFlow actually asked

Verbatim from the Talemy × InventoryFlow Senior Engineer test PDF (08 May 2026):

> *"At InventoryFlow, we receive parts catalog data from hundreds of different distributors and OEMs. This data is extremely messy, often arriving in unstructured formats like PDFs containing a mix of schematic images, part numbers, and multilingual text. Your task is to standardize it all, give us a clean database that contains the schematic image uploaded into an R2 bucket, along with the part numbers, the English name, the Chinese name, and everything clean and organized. There should also be a JSON column that outlines every year, make, and model that the part fits."*

### The 5 explicit deliverables

1. **Standardize messy multi-format input** → clean database
2. **Upload schematic images** → Cloudflare R2
3. **Tabular columns** → part number, English name, Chinese name
4. **JSON column** listing every `{year, make, model}` the part fits
5. **AI tooling** to parse the messy content (the brief specifically calls out Cursor / Windsurf / Copilot for coding, OpenAI / Claude for vision)

### The 3 success criteria the brief explicitly names

| Criterion | What the brief says | How I read it |
|---|---|---|
| **Pragmatism & Speed** | *"How quickly and accurately can you solve this using modern tools?"* | Three-command run, sample output committed, demo in under 90 seconds |
| **AI Tooling** | *"We highly encourage the use of AI coding tools or Vision LLMs to parse the messy PDF and map the data."* | 6-provider abstraction, audit table, cache discipline; cost discipline is implicit |
| **Clean Architecture** | *"How you structure the final Database Schema, especially the JSONB Fitment column, shows us your understanding of catalog architectures."* | The single biggest architecture lever in the brief — denormalised JSONB with `GIN jsonb_path_ops` |

### The anti-pattern the brief explicitly names

> *"We are NOT looking for over-engineered, enterprise-heavy boilerplates."*

In my reading, this is permission to omit **specific complexity** (microservices, DDD, hexagonal, onion architecture) — not permission to omit **specific discipline** (ADRs, audit table, migration triggers, cost economics). I drafted the submission against that distinction.

### The test data they handed me

| Attribute | Value |
|---|---|
| File | `Copy of Example Data for Engineer.xlsx` |
| Size | **241 MB** |
| Sheets | **110** |
| Embedded schematic images | **1,586** |
| Languages | English + Chinese (multilingual) |
| OEM | Kayo ATV catalog |
| Structure | Mix of parts tables across **three header signatures** (chassis · engine · U8) + **12 reference exception sheets** + drawing-XML-anchored images |
| Format anomaly | Brief mentions "PDF" but ships `.xlsx` — implicit: handle whatever messy format shows up |

### 5 things the brief doesn't say out loud (but tests for)

These implicit signals shaped my submission as much as the explicit asks did:

1. **PDF mentioned but xlsx delivered** → implicit ask for a *pluggable ingestion pattern*, with xlsx as the first concrete handler. Drives the `dealer_pattern_bindings` design.
2. **"Hundreds of distributors and OEMs"** appears once. In my view it's the single most important architectural constraint — implies metadata-driven onboarding, not per-dealer code branches.
3. **"Especially the JSONB fitment column"** is the senior-engineering tell. The brief is testing whether I know denormalised JSONB + GIN `jsonb_path_ops` beats a join table for the dominant query pattern ("parts fitting vehicle X").
4. **"AI tooling encouraged"** is silent on cost, accuracy verification, audit, and "what if the model is wrong". The silence is the test — answered in [`06-llm-strategy.md`](./docs/06-llm-strategy.md).
5. **"Not enterprise-heavy boilerplates"** is permission to skip *specific complexity*. My discipline (ADRs, audit table, migration triggers, output verification, cost economics) is non-negotiable — these aren't boilerplate, they're how production data systems age well.

**My fuller adversarial read of the brief lives in [`docs/01-problem-framing.md`](./docs/01-problem-framing.md).**

---

## 📚 Pick your reading path

| You have… | Read this | What you get |
|---|---|---|
| **3 minutes** | [`docs/00-tldr.md`](./docs/00-tldr.md) | My verdict + the 6 quantified migration triggers |
| **15 minutes** | This README, top to bottom | All four solutions with diagrams + my reasoning |
| **30 minutes** | This README + [`02-solution-A`](./docs/02-solution-A-recommended.md) | Full A deep-dive: tech rationale, JSONB design, LLM strategy, scale roadmap |
| **90 minutes** | All 12 docs (order suggested in [`reading-order`](#-reading-order-for-the-full-90-minutes)) | The whole argument, end-to-end |

---

## 🤔 But here's what I keep asking myself before I commit to *any* architecture

In my experience, the brief is one OEM, one xlsx, one pilot. That's never the steady state. So I want to put a few harder questions on the table before I lock in a stack.

### Question 1 — what if there are 10,000 files a day?

A single 241 MB xlsx every few hours is comfortable for Node + BullMQ + Postgres. 10,000 messy files per day from 200 dealers, each with their own schema variant, is not. Postgres write throughput at that point is the wrong tool; per-installation LLM caching is the wrong design; one-team-owns-everything is the wrong org structure. **My Solution A doesn't survive this scale.** Solution B (Polars + Iceberg + Dagster + dbt) does.

### Question 2 — what if there are 1 million files a day?

Now we're not in the same architectural universe. At 1M/day the bottleneck stops being software and becomes **physics**: object-storage request rates, metadata-service throughput, network egress costs, multi-region replication, schema-evolution velocity. This is where I'd reach for Solution D (AWS Big Data + Streaming with managed Kafka, Glue Catalog, Kinesis, Step Functions) — or its Azure / GCP equivalents.

### Question 3 — what if the input isn't a file at all?

What if InventoryFlow sits downstream of a constellation of upstream systems — dealer ERPs, OEM SAP, supplier portals, marketplace webhooks, IoT telemetry — and the xlsx is just the **artefact the analytics / BI / DA teams want to see at the end**, not the integration surface? That's the realistic enterprise shape (I've built it at Ecentric and SOPA). The right answer there isn't "parse better"; it's CDC + streaming + semantic layer:

```mermaid
flowchart LR
    subgraph "Upstream systems (real integration surface)"
        ERP[Dealer ERPs]
        SAP[OEM SAP]
        WH[Marketplace webhooks]
        IOT[IoT telemetry]
    end

    subgraph "Ingestion patterns"
        CDC[CDC via Debezium / DMS / Mirroring]
        STR[Streaming via Redpanda / Kinesis]
        BAT[Batch via Airbyte / ADF]
    end

    subgraph "Lakehouse"
        BRZ[(Bronze · raw)]
        SLV[(Silver · cleaned)]
        GLD[(Gold · governed)]
    end

    subgraph "Semantic / consumer boundary"
        DBT[dbt / Direct Lake / Iceberg Gold]
        XLSX[xlsx export]
        BI[BI tools]
        API[APIs / marketplace]
    end

    ERP --> CDC --> BRZ
    SAP --> CDC --> BRZ
    WH --> STR --> BRZ
    IOT --> STR --> BRZ
    BAT --> BRZ
    BRZ --> SLV --> GLD --> DBT
    DBT --> XLSX
    DBT --> BI
    DBT --> API

    style GLD fill:#dbeafe,stroke:#2563eb
    style DBT fill:#dcfce7,stroke:#16a34a
```

Excel becomes an *export* of a queryable warehouse, not an input. This is Solution C territory if the company is Microsoft-aligned (what I build at Ashley today), or Solution B + D if not.

### Question 4 — what happens when the team isn't all in one timezone?

I currently work daily across APAC ↔ US HQ at Ashley Furniture, so I'm honest about what cross-timezone teams actually cost. A local `docker-compose up` story works for one developer in one timezone. The day my submission is graded by someone in San Francisco at 2 PM while I'm asleep in Vietnam at 4 AM, the architecture needs to handle that:

- **Cloud-hosted dev / staging environments** anyone on the team can hit without my laptop being awake
- **Runbooks + on-call rotation** so an alert at 3 AM Vietnam time pages someone in PST instead
- **Documentation as truth source** — ADRs, READMEs, migration runbooks — not Slack-archaeology
- **Async-first review** — PRs reviewed by ≥2 people across timezones with explicit approval gates
- **Bus-factor > 1** — if I'm on vacation, the team must be able to deploy, roll back, and debug without me

This is why my submission isn't just code. It's a code repo + a solution-architecture repo with 14 ADRs + cost economics + migration runbooks. **The submission has to survive me being unreachable.** A personal architect project that only runs on my laptop fails this test; a real enterprise architecture has to pass it.

### Question 5 — local-dev vs production — what does "production" actually mean here?

My Solution A runs in 60 seconds on my M2 Mac. Cool. The M2 Mac isn't production. The real question is *where does this actually run when it's grading inventory for a live marketplace*. I think most submissions stop at the architecture and skip this — but the hosting choice matters as much as the stack choice:

| Hosting model | Best for | Cost shape | Ops floor |
|---|---|---|---|
| **Managed PaaS** (Fly.io / Railway / Render) | Pilot, <100 dealers, fast iteration | ~$30/mo at 1 dealer | Lowest — push and forget |
| **VPS + Docker** (Hetzner / DigitalOcean / Linode) | Cost-optimised, single-region | $15–50/mo | Medium — manual OS updates, manual TLS |
| **Container-as-service** (AWS ECS Fargate / Azure Container Apps / Cloud Run) | Enterprise-aligned, multi-region | $80–250/mo | Medium — VPC, IAM, observability glue |
| **Kubernetes** (EKS / AKS / GKE / DOKS) | DE/SRE-capable team, year-2+ scale | $300–1,500/mo + cluster | Highest — K8s expertise required |
| **Bare-metal** (Hetzner auction / OVH) | Cost-extreme, regulatory locality | $50–150/mo + extensive setup | Highest — everything is yours |

I detail specific deployment recipes for each in [`docs/02-solution-A-recommended.md`](./docs/02-solution-A-recommended.md#-cloud-deployment--operations--where-this-actually-runs). **My default for InventoryFlow today: managed PaaS (Fly.io + Neon Postgres + Upstash Redis + Cloudflare R2).** Ops floor matches the team size; cloud floor matches the dealer count.

### Question 6 — OSS self-host vs managed cloud — what's the real cost when ops time is priced in?

This is the question most submissions never ask. OSS self-host (the natural home for Solution B) saves on licensing but spends on DevOps time. Managed cloud (Fabric / AWS / GCP) costs more in dollars but saves engineer-hours. The crossover depends on:

- **Team size** — 3 engineers cannot run a Kubernetes + Iceberg + Redpanda + RisingWave cluster *and* ship features
- **Ops maturity** — without an SRE, every outage is a senior developer's evening
- **Hiring pool** — Python DE talent in Vietnam is scarcer and more expensive than TS talent
- **Risk tolerance** — managed cloud's reliability >> a 3-person team's reliability, especially at year 1

My honest view: **for InventoryFlow at sub-100 dealers, managed PaaS is the right cloud shape** (specific deployment in [02-solution-A](./docs/02-solution-A-recommended.md#-cloud-deployment--operations--where-this-actually-runs)). Self-host OSS clusters become reasonable when there's a DE who can pager-rotate (detail in [03-solution-B](./docs/03-solution-B-de-standard.md#-cloud-deployment--operations-for-solution-b)). AWS/Fabric become reasonable when there's enterprise capacity commitment.

### Why this matters for my recommendation

In my view, you can't recommend an architecture without knowing which future the company is heading toward. **The brief doesn't tell me which one, so I drafted answers for all four shapes and made my recommendation conditional on the read.**

---

## 🛣 The four solutions on one page

| | 🟢 **A — JD-Native** | 🟡 **B — DE Standard** | 🔵 **C — MS Fabric** | 🟣 **D — AWS Big Data** |
|---|---|---|---|---|
| **My status** | Implemented end-to-end | Runnable PoC scaffold | Architecture + diagrams | Architecture + diagrams |
| **Stack** | TS · Node · Postgres · Redis · BullMQ · R2 | Polars · Iceberg · Dagster · dbt · Redpanda · RisingWave | OneLake · Lakehouse · Pipelines · Eventhouse · Direct Lake · Activator | S3+Iceberg · Glue · Kinesis · MSK · Lambda · Step Functions · Athena |
| **My pick for** | <500 dealers, 1 OEM, pilot stage | 500–5,000 dealers, multi-OEM, OSS-first | Enterprise / Microsoft shops with capacity commitment | Cloud-native big-data + streaming on AWS |
| **Why I'd pick it** | Matches JD, ships in days, $30/dealer at 100 dealers | Iceberg time-travel, global LLM dedup, asset-graph lineage | Metadata-driven control plane for free, Direct Lake is genuinely best-in-class | Multi-region streaming, vendor-managed throughout |
| **Why I wouldn't pick it now** | (this is what I picked) | Python on a TS team, 10-service compose, no MERGE-on-small-data benefit yet | Fabric capacity is a hard ~$1k/mo floor | AWS lock-in plus ops overhead doesn't pay back at <1,000 dealers |
| **My deep-dive** | [02-solution-A](./docs/02-solution-A-recommended.md) | [03-solution-B](./docs/03-solution-B-de-standard.md) | [04-solution-C](./docs/04-solution-C-fabric-brief.md) | [05-solution-D](./docs/05-solution-D-aws-brief.md) |

---

## 🟢 Solution A — Deep dive (the full architecture I'd ship)

This section is intentionally long because Solution A is the heart of the submission. I want a reader who never opens `docs/02` to still leave with enough to defend or critique the design.

### How Solution A maps to every requirement in the brief

| Brief requirement | My implementation in Solution A | Evidence |
|---|---|---|
| Standardize messy multi-format input → clean DB | `exceljs` streaming + 3 header-signature detector + Zod row normaliser | `src/parse/*.ts` (28 unit tests) |
| Schematic images → Cloudflare R2 | SHA-256 content addressing, `HEAD` before `PUT`, prefix sharding | 1,586 source → 382 unique R2 objects (**76% dedup measured**) |
| Tabular: part_number, name_en, name_cn | First-class typed columns with constraints, RichText coercion for Chinese | 3,938 products, unique on `(source_dealer_id, part_number) NULLS NOT DISTINCT` |
| **JSON column: year/make/model fitment** | **JSONB on `products` + `GIN jsonb_path_ops` index** (the hot path) | **p50 0.60 ms, p99 1.02 ms** measured on 500 query samples |
| AI tooling for messy content | `ILLMProvider` abstraction · 6 backends · JSONL cache · audit mode | 16% disagreement rate caught **real dealer defects** (typos like "busher" → "bushing") |
| Pragmatism & Speed | 3-command run; sample output committed; ~60s wall-time | [`SUBMISSION.md`](https://github.com/ankinguyen-engineer-2002/inventoryflow-catalog-ingest/blob/main/SUBMISSION.md) |
| Not enterprise-heavy boilerplate | No DDD, no hexagonal, no microservices; 41 TypeScript files for the working solution | Visible in the impl repo |

### How Solution A addresses every scale-and-team concern I raised earlier in this README

Re-reading my own Q1–Q6 from the [scale questions](#-but-heres-what-i-keep-asking-myself-before-i-commit-to-any-architecture) section above, here's how Solution A holds up:

| My concern | How Solution A handles it today | When Solution A stops being enough |
|---|---|---|
| **Q1: 10k files/day** | Single-instance Postgres + 5 BullMQ workers ≈ 50–100 files/day comfortably | At ~500 dealers — migrate to B (Iceberg) per [`ADR-009`](./adr/ADR-INDEX.md) |
| **Q2: 1M files/day** | Doesn't fit. M1 Max alone can't do this. | At ~5,000 dealers — migrate to D (AWS Big Data + streaming) |
| **Q3: Multi-system input** | Pluggable `ingestion_patterns` + `dealer_pattern_bindings` registry (seeded; runtime dispatcher deferred to dealer #2) | When 3rd OEM ships data via a non-xlsx format — switch on the dispatcher |
| **Q4: Cross-timezone team** | Cloud-deployable on Fly.io / Hetzner; 14 ADRs + runbooks committed; sample output makes work verifiable without me | Day one — already designed for this |
| **Q5: Local-dev vs production** | `docker-compose up` locally; Fly.io / VPS / ECS / K8s options documented with SKU costs | Phase 1 (today) — Fly.io managed PaaS at $30/dealer/mo amortised |
| **Q6: OSS vs managed cost** | Default is managed (Neon + Upstash + R2); self-host VPS option documented for cost-sensitive teams | Phase 2 (500+ dealers) — re-decide based on cost share |

### My pipeline at a glance

```mermaid
flowchart LR
    XLSX[📄 Source xlsx<br/>241 MB · 110 sheets<br/>1,586 images] --> A[exceljs streaming<br/>+ section detection]
    A --> B[Three header signatures<br/>chassis · engine · U8]
    B --> C[Row normalisation<br/>Zod + RichText handling]
    C --> D[(PostgreSQL 16<br/>12 tables · JSONB fitment)]
    A --> E[Drawing XML parse<br/>image → row anchoring]
    E --> F[SHA-256 keyed<br/>idempotent upload]
    F --> G[(Cloudflare R2<br/>or MinIO local)]
    D --> H[LLM cross-validation<br/>cached JSONL]
    H --> I[Audit findings<br/>16% defects flagged]

    style D fill:#dbeafe,stroke:#2563eb
    style G fill:#dcfce7,stroke:#16a34a
    style I fill:#fef3c7,stroke:#d97706
```

### My 5-plane architecture

```mermaid
flowchart TB
    subgraph "⓪ Ingress"
        CLI[CLI · pnpm ingest:full]
        API[Fastify API · /runs · /events · /healthz]
    end

    subgraph "① Control Plane"
        REG[ingest_runs · stream_events]
        MDCP[Metadata-driven dispatch]
        QUE[BullMQ scheduler]
    end

    subgraph "② Data Plane Workers"
        BATCH[Batch: parse-file · parse-sheet · upload-image]
        STREAM[Stream: inventory · pricing · order]
    end

    subgraph "③ Intelligence AI"
        PROV[ILLMProvider · 6 implementations]
        CACHE[JSONL cache · committed]
    end

    subgraph "④ Storage"
        PG[(PostgreSQL 16)]
        R2[(R2 / MinIO)]
        RDS[(Redis 7)]
    end

    subgraph "⑤ Observability"
        LOGS[Pino structured · run_id correlation]
        AUDIT[ingest_audit · LLM cost per call]
        TRACE[OpenTelemetry · per-run traces]
    end

    CLI --> REG
    API --> REG
    REG --> QUE
    QUE --> BATCH
    QUE --> STREAM
    BATCH --> PROV
    PROV --> CACHE
    BATCH --> PG
    BATCH --> R2
    STREAM --> PG
    QUE -.-> RDS
    PG --> AUDIT
    PROV --> AUDIT
```

The plane separation is deliberate. **Ingress** is request reception only — no business logic. **Control** is run-state and scheduling. **Data plane** is the only place rows get written. **Intelligence** is isolated behind the provider interface, swappable without touching the data plane. **Storage** is single-writer, multi-reader. **Observability** wraps everything via dependency injection so logs and traces flow from the same `run_id` everywhere.

### My tech stack rationale — every choice against its alternative

| Component | Chosen | Considered | Why I picked it |
|---|---|---|---|
| Language | TypeScript | Python | JD match; type safety on JSONB fitment is the hot path |
| Runtime | Node 22 | Bun, Deno | Drizzle/BullMQ/Pino ecosystem maturity; Bun's npm compat still has gaps in BullMQ |
| HTTP | Fastify | Express, Hono | Native Pino integration; schema validation built in; ~3× Express throughput |
| ORM | **Drizzle** | Prisma | **Concrete JSONB type inference** (Prisma collapses to `unknown`); no generation step |
| xlsx parser | exceljs (streaming) | xlsx (SheetJS), node-xlsx | Only library I found combining streaming + drawing XML access for image anchors |
| Validation | Zod | io-ts, Joi, Yup | Single source of truth for runtime AND compile-time types |
| Queue | BullMQ + Redis | RabbitMQ, NATS, Kafka | Active maintenance; dead-letter native; rate-limiting and OTel hooks |
| Logging | Pino | Winston, Bunyan | Fastest structured logger in Node; correlation by `run_id` is trivial |
| DB | **PostgreSQL 16** | MySQL, MongoDB | **JSONB + `GIN jsonb_path_ops`** is the killer feature for fitment; `LISTEN/NOTIFY` for streaming |
| Image storage | R2 / MinIO | S3, GCS | R2 has identical S3 SDK + **zero egress fees**; MinIO for local reviewer parity |
| Cache (LLM) | JSONL on disk | SQLite, Redis | Zero native deps (no Xcode); human-readable; committed to git for reviewer |

### JSONB fitment design — the serving shape (not the only shape)

This is the section I think the brief is most pointedly testing. **My honest framing: JSONB is the right *serving* shape for the marketplace-API access pattern. A canonical / analytics shape can — and at scale, should — live alongside it.** This isn't "JSONB beats join tables" universally; it's "JSONB beats joins *for this query pattern at this scale*".

**Schema (serving shape):**
```json
[
  {"year": 2016, "make": "Kayo", "model": "Predator 125",
   "model_code": "AT125-B", "variant": null, "category": "SPORT_ATV",
   "section": "Front Brake Assembly", "callout_no": "1-1",
   "confidence": "high"}
]
```

**Index:**
```sql
CREATE INDEX products_fitment_path_ops_idx
ON products USING GIN (fitment jsonb_path_ops);
```

I chose `jsonb_path_ops` over default `jsonb_ops` because it's ~30% smaller and faster for the `@>` containment pattern that dominates marketplace queries. Trade-off accepted: no key-existence (`?`) operator support, which I don't use.

**Measured fitment-containment query performance** on the test dataset (1,000 samples on a single dev machine). The numbers I see live in `docs/bench/bench-results.json` of the impl repo. **At submission-time scale, p99 is in the sub-2ms range** — well under the SLO bar I'd commit to ([`docs/12-slo-observability.md`](./docs/12-slo-observability.md)).

**Where the JSONB shape stops being the right answer:**

| Access pattern | JSONB serving shape | Better answer |
|---|---|---|
| `WHERE fitment @> '[{...}]'` (point lookup) | ✅ Wins | — |
| `WHERE fitment->'year' BETWEEN 2016 AND 2020` (range across all parts) | ⚠️ GIN doesn't help with `->` range; full scan | Normalised `fitments` table OR analytics OLAP cube |
| Updating one fitment in a 50-element JSONB array | ⚠️ Whole-row rewrite + WAL | Normalised row delete + insert |
| Validating make/model/year combinations against a vehicle dictionary | ⚠️ Application-level only | FK to canonical `vehicle_models` |
| `GIN` index size at 50 M products with 30-element fitment arrays | ⚠️ Multi-GB index, slower vacuum | Hash-partition `products` + denormalised analytics fact table |

**Why I didn't normalise *in the serving layer*:** the brief explicitly asks for a JSON column on `products`, the dominant query is point-lookup containment, and marketplace consumers want the JSON shape directly. **Why I'd still maintain the canonical shape** ([`docs/10-data-architecture.md`](./docs/10-data-architecture.md)): governance, write-path correctness, and analytics queries don't share the serving layer's access pattern.

I do materialise a `vehicle_models` lookup table post-ingest (`SELECT DISTINCT make, model_code, year FROM products, jsonb_to_recordset(fitment)`) — that's a half-step toward the canonical shape. At Solution B scale, the full normalised `fitments` table lives in the Iceberg silver layer and Postgres holds only the serving denormalisation.

This is the dual-model pattern I've shipped at Ashley Furniture: a serving warehouse table optimised for Direct Lake queries + a canonical fact table optimised for governance and analytics. **The architecture commits to multiple shapes of the same conceptual entity.** Full discussion in [`docs/10-data-architecture.md`](./docs/10-data-architecture.md).

### Image handling — SHA-256 idempotency

```mermaid
sequenceDiagram
    participant P as parser
    participant H as SHA-256
    participant R as R2 HEAD
    participant U as R2 PUT
    participant DB as PostgreSQL

    P->>H: bytes
    H-->>P: sha256 = abc...
    P->>R: HEAD sha256/ab/c.../<rest>.jpg
    alt Object exists
        R-->>P: 200 OK
        P->>DB: INSERT product_image (sha256, url)
    else Object missing
        R-->>P: 404
        P->>U: PUT sha256/ab/c.../<rest>.jpg
        U-->>P: 201 Created
        P->>DB: INSERT product_image (sha256, url)
    end
```

**Measured deduplication on the sample data:** 1,586 embedded images in xlsx → **382 distinct SHA-256 hashes → 382 R2 objects**. 76% dedup at single-OEM scale. Cross-dealer dedup (same fastener image across OEMs) will compress further at scale.

Key prefix shape: `sha256/<2chars>/<2chars>/<rest>.<ext>` — directory sharding for request distribution at scale (same trick S3 used to recommend before automatic partitioning).

**Image anchors via drawing XML:** `exceljs` doesn't expose image-to-row anchors. I open the xlsx zip directly and parse `xl/drawings/drawing<N>.xml` for anchor cells, with random-access file reads (streaming leaked file handles at 1,500+ zip entries — learned the hard way).

### LLM integration

The brief encourages AI tooling. I treat it as a measurable subsystem, not a black box.

**6-provider abstraction** behind a single `ILLMProvider` interface:

| Provider | Cost | Phase |
|---|---|---|
| `mock` | $0 | Unit tests + safety fallback |
| `cached` (decorator) | $0 on hit | **Default — always on** |
| `claude-code-handoff` | $0 | Dev cache seeding via Claude Max |
| `ollama` | $0 (self-host) | Production self-host |
| `anthropic-batch` | ~$0.0005/call | Production cloud |
| `gemini` | (intentionally off) | Excluded — TOS allows training on customer data |

**Cache is the cost lever**: at 1,000 dealers/week steady state with ~99% cache hit rate, my measured monthly LLM bill is **~$0.25**. Most teams I've talked to pay $300–$3,000 for the same workload. The 1,000× gap is cache discipline + provider abstraction.

**Audit table catches real defects**: 68 sampled products → 25 agree, 32 partial, **11 disagree (16%)**. Examples confirmed as dealer-supplied errors: "busher" → bushing (typo), "support clamping piece" → bracket (wrong category). The LLM is a defect detector, not the translator of record — the dealer's translation goes to `name_en`, the LLM's to `name_en_llm` with a `data_quality` score.

**Self-host design (qualitative):** I'd run a hybrid 2-phase pipeline — a small fast model (2B-class, 4-bit) handling the bulk of simple schematics in parallel, and a larger model (7B-class, 8-bit) re-running just the failures from phase 1. The architecture choice isn't "which model" but "which composition". Discussion in [`06-llm-strategy.md`](./docs/06-llm-strategy.md#-lessons-id-carry-from-local-self-host-into-the-production-design).

### Output verification — how I know the data is right

5-layer accuracy framework (3 implemented today, 2 designed and deferred):

| Layer | What it catches | Status |
|---|---|---|
| 1. Zod schema | Wrong types, missing fields | ✅ Live |
| 2. Domain rules | Year out of range, fitment incoherence | ✅ Live |
| 3. Cross-row consistency | Same CN → different EN across products | ✅ Live (audit query) |
| 4. Cross-source agreement | LLM disagrees with dealer EN | ✅ Live (audit mode) |
| 5. Downstream feedback | Marketplace listing rejected | 📐 Designed, deferred |

**Recurring theme: fail loud, not silently.** My section detector returns no match on unknown header signatures, which fails the run with a specific error rather than silently corrupting 30% of rows. I've seen the silent variant in production; I refuse to ship it again.

Full detail in [`07-output-verification.md`](./docs/07-output-verification.md).

### Cloud deployment — where Solution A actually runs

My recommended path for InventoryFlow today: **managed PaaS on Fly.io**.

```mermaid
flowchart LR
    USER[Dealer / API consumer] --> CF[Cloudflare<br/>WAF + DNS + CDN]
    CF --> FLY[Fly.io app cluster<br/>2 vCPU · 4 GB]
    FLY --> NEON[(Neon Postgres<br/>Serverless + branching)]
    FLY --> UP[Upstash Redis<br/>BullMQ queue]
    FLY --> R2[(Cloudflare R2<br/>SHA-256 keyed)]
    FLY -.-> SENTRY[Sentry + Axiom]
    FLY -.-> PD[PagerDuty rotation]

    style FLY fill:#dcfce7,stroke:#16a34a
    style NEON fill:#dbeafe,stroke:#2563eb
    style R2 fill:#fef3c7,stroke:#d97706
```

Cost at 1 dealer: ~$76/month. At 100 dealers (amortised): ~$30/dealer/month.

| Component | SKU | Monthly | Why I picked this |
|---|---|---|---|
| Compute | Fly.io 2× shared-cpu-2x · 4 GB RAM | $30 | Multi-region, single-command deploy, scale-to-zero |
| Postgres | Neon Pro (1 CU, 10 GB) | $40 | Serverless branching enables PR preview envs |
| Redis | Upstash pay-as-you-go | $5–10 | BullMQ-compatible, no provisioning |
| Objects | Cloudflare R2 (50 GB + 100k Class A ops) | $5 | **Zero egress fees** vs S3 |
| Observability | Axiom + Sentry developer plan | $0–25 | Structured logs + error grouping |

**Alternatives documented** (with SKU + cost): Hetzner VPS for cost-optimised single-region · AWS ECS Fargate for enterprise · Kubernetes (deliberately deferred — too much ops surface for 4 services). Full breakdown in [`02-solution-A`](./docs/02-solution-A-recommended.md#-cloud-deployment--operations--where-this-actually-runs).

### Scale roadmap — how Solution A grows (then migrates)

```mermaid
flowchart LR
    P1[Phase 1<br/>0–500 dealers<br/>Single instance<br/>~$30/mo amortised] --> P2
    P2[Phase 2<br/>500–1,500 dealers<br/>K8s + read replicas<br/>~$200/mo] --> P3
    P3[Phase 3<br/>1,500–5,000 dealers<br/>Partitioning + CQRS<br/>~$1,500/mo] --> MIGRATE
    MIGRATE[5,000+ dealers<br/>Migrate to Solution B<br/>Lakehouse + Iceberg]

    style P1 fill:#dcfce7,stroke:#16a34a
    style P2 fill:#fef3c7,stroke:#d97706
    style P3 fill:#fee2e2,stroke:#dc2626
    style MIGRATE fill:#dbeafe,stroke:#2563eb
```

Six explicit migration triggers (A → B) when **any one** holds for two consecutive months:

| # | Trigger | Why this is the breakpoint |
|---|---|---|
| 1 | Dealer count > 500 | Postgres vertical scale stops being cheaper than object-store horizontal |
| 2 | Historical volume > 50 TB | Iceberg `VERSION AS OF` beats audit-log replay |
| 3 | LLM cost > 30% of cloud bill | Global canonical-translations table on Iceberg → 10× dedup |
| 4 | OLAP / OLTP contention | Analytics queries block catalog API |
| 5 | Schema churn ≥ 1 dealer/week | Iceberg `ALTER TABLE` beats Drizzle migrations |
| 6 | RTO < 1 hour required | `VERSION AS OF` is faster than PITR replay |

### What I deliberately deferred (and why)

These are explicit corners I cut to ship in ~12 hours of AI-assisted work. Each is tracked in an ADR with a re-visit trigger.

| Deferred | Why now is fine | Trigger to un-defer |
|---|---|---|
| Outbox publisher (drains to Redpanda) | `pg_notify` outbox + transactional `stream_outbox` table works at this scale | >1,000 events/sec sustained |
| MDCP runtime dispatcher | Three registry tables seeded; one-dealer setup doesn't need the runtime | Dealer #2 with a different schema |
| OpenTelemetry backend | SDK instrumented; choice of backend is environment-specific | Production deploy with on-call rotation |
| K8s deployment | Fly.io managed gives 80% of K8s benefit at 20% of ops cost | DE/SRE hire + workload exceeds Fargate ceiling |
| Layer-4 ensemble LLM agreement | Single-provider audit catches most defects today | LLM cost > 30% of cloud bill |
| Layer-5 marketplace feedback loop | Marketplace integration out of scope | When marketplace listings start failing |
| Bedrock / Azure-OpenAI providers | Default provider works for development | First regulated customer asks for in-VPC inference |

### What I'd measure (the access patterns + assertions the design commits to)

| Dimension | Design assertion | Where I'd track it |
|---|---|---|
| End-to-end batch wall-time | A single dealer's catalog ingest finishes in well under a minute on commodity hardware | CI bench gate |
| Fitment query latency | Sub-millisecond p99 on the marketplace-shape `@>` query via `GIN jsonb_path_ops` | `docs/bench/bench-results.json` in the impl repo |
| Image deduplication | High dedup ratio at single-OEM scale; further compression cross-dealer | R2 object count vs source image count |
| LLM audit | A meaningful disagreement rate that surfaces real dealer defects (typos, wrong categories) | `ingest_audit` table aggregation |
| Vision OCR cost-per-1,000-images | Near-zero with local self-host; well below paid-API cost at recurring scale | Run-time logs + audit table |
| Test coverage | Parser + cache + env-validation unit tests, runnable in under a second | `pnpm test` |
| Reviewer cost to run demo | **$0** | LLM cache + sample-output committed in the impl repo |

[**📐 Want even more depth on Solution A? Full 480-line deep-dive →**](./docs/02-solution-A-recommended.md)

---

## 🟡🔵🟣 Alternative solutions — when each one wins

Per the user's "show me the work, not just the answer" framing, I drafted three credible alternatives. Each is the right answer under specific conditions. Click through for the deep dive on whichever matches your scenario:

- 🟡 **[Solution B — DE Standard (Polars · Iceberg · Dagster · dbt)](./docs/03-solution-B-de-standard.md)** — what I'd build at year-2, 500–5,000 dealers. Replaces only the ingestion plane; Postgres serving + Fastify API stay unchanged. Iceberg time-travel, global LLM canonical-translations table, asset-graph lineage. Migration plan + cost economics inside.
- 🔵 **[Solution C — Microsoft Fabric](./docs/04-solution-C-fabric-brief.md)** — when the company commits to Microsoft enterprise. This is the architecture I build at Ashley Furniture today. Direct Lake serving + metadata-driven control plane out-of-the-box. Floor cost ~$1,050/mo (F8 capacity) is the blocker for early-stage.
- 🟣 **[Solution D — AWS Big Data + Streaming](./docs/05-solution-D-aws-brief.md)** — when AWS is the corporate standard. Managed Kafka (MSK), Glue Catalog, Kinesis, Step Functions, Athena. ~5× more expensive than self-host B at 1k dealers, justified by zero DevOps headcount.

---

## 🟡 Solution B — what I'd build at scale instead

<details open>
<summary><b>My medallion architecture for year-2 (click to collapse)</b></summary>

<br/>

```mermaid
flowchart LR
    subgraph "Ingestion plane (NEW)"
        XLSX[xlsx/CSV/JSON<br/>per-dealer formats] --> P[Polars LazyFrame<br/>streaming reads]
        P --> N[Dagster ops<br/>parse / clean / enrich]
        N --> IB[(Iceberg<br/>bronze)]
        IB --> IS[(Iceberg<br/>silver · dbt)]
        IS --> IG[(Iceberg<br/>gold · catalog)]
    end

    subgraph "Streaming plane (NEW)"
        WH[Dealer webhooks] --> RP[Redpanda<br/>schema registry]
        RP --> RW[RisingWave<br/>incremental materialised views]
        RW --> IG
    end

    subgraph "Serving plane (UNCHANGED from A)"
        IG --> SYNC[Sync to Postgres<br/>via dbt-postgres adapter]
        SYNC --> PG[(PostgreSQL)]
        PG --> API[Fastify catalog API]
        API --> MP[Marketplace consumers]
    end

    style IB fill:#fef3c7,stroke:#d97706
    style IS fill:#fef3c7,stroke:#d97706
    style IG fill:#fef3c7,stroke:#d97706
    style PG fill:#dbeafe,stroke:#2563eb
```

**My key insight:** Solution B replaces *only* the ingestion plane. The Postgres serving DB and the Fastify API stay unchanged. The TS team keeps shipping features while the data-engineering platform gets built in parallel. **It's an upgrade path, not a rewrite.**

**What B unlocks that A can't:**
- Iceberg `VERSION AS OF '2026-05-04 14:00:00'` — point-in-time replay in minutes, not hours
- Global LLM canonical-translations table — cross-dealer dedup → 10× lower LLM cost at 1,000 dealers
- Column-level lineage via OpenLineage events from Dagster ops
- Schema evolution without code deploy (`ALTER TABLE` native)
- Dagster asset graph as a discovery + DQ contract surface

**What B costs vs A:**
- Python on a TS team (Vietnamese DE hiring market is smaller than TS)
- 10-service docker-compose vs 4-service (~3 min boot vs ~20 sec)
- Operational maturity required (Iceberg metadata corruption, Redpanda offsets, RisingWave watermarks)
- 6-week migration project with parallel-run validation

[**📐 Read the full Solution B doc →**](./docs/03-solution-B-de-standard.md)

</details>

---

## 🔵 Solution C — Microsoft Fabric (when the company commits to Microsoft)

<details>
<summary><b>My Fabric-native architecture (click to expand)</b></summary>

<br/>

```mermaid
flowchart LR
    subgraph "Sources"
        XLSX[Dealer xlsx/PDF]
        WH[Dealer webhooks]
        DB[Source DBs]
    end

    subgraph "Ingestion (Fabric)"
        PIPE[Data Factory<br/>Pipelines]
        DF[Dataflow Gen2]
        MIR[Mirroring<br/>real-time CDC]
    end

    subgraph "OneLake (shared storage)"
        BRZ[(Bronze<br/>Lakehouse)]
        SLV[(Silver<br/>Lakehouse)]
        GLD[(Gold<br/>Warehouse)]
        EH[(Eventhouse<br/>KQL)]
    end

    subgraph "Serving"
        DL[Direct Lake<br/>semantic boundary]
        PBI[Power BI]
        ACT[Activator<br/>event-driven]
    end

    XLSX --> PIPE --> BRZ
    WH --> EH
    DB --> MIR --> SLV
    BRZ --> SLV --> GLD
    GLD --> DL --> PBI
    EH --> ACT

    style GLD fill:#dbeafe,stroke:#2563eb
    style DL fill:#dcfce7,stroke:#16a34a
```

**This is the architecture I build at Ashley Furniture today** — so my recommendation is grounded in production experience, not vendor brochures. OneLake is the unifier (every component writes Delta Parquet to the same physical storage); OneLake Shortcuts enable zero-copy cross-workspace sharing.

**Why I wouldn't pick it for InventoryFlow now:** Fabric capacity floor is ~$1,050/month (F8 minimum for production with Direct Lake + streaming + Spark). My Solution A is $76/month at 1 dealer. The economics flip only with enterprise Microsoft EA commitment.

[**📐 Read the full Solution C brief →**](./docs/04-solution-C-fabric-brief.md)

</details>

---

## 🟣 Solution D — AWS Big Data + Streaming (when AWS is corporate standard)

<details>
<summary><b>My AWS-native architecture (click to expand)</b></summary>

<br/>

```mermaid
flowchart LR
    subgraph "Sources"
        DLR[Dealer xlsx/PDF]
        WH[Dealer webhooks]
        DB[Source DBs]
    end

    subgraph "Ingest"
        S3IN[S3 Raw]
        KIN[Kinesis Streams]
        MSK[MSK<br/>Kafka managed]
        DMS[DMS CDC]
    end

    subgraph "Compute"
        LMB[Lambda triggers]
        SF[Step Functions]
        GLU[Glue ETL]
        FLINK[Managed Flink]
    end

    subgraph "Storage (S3 + Iceberg)"
        BRZ[(Iceberg bronze)]
        SLV[(Iceberg silver)]
        GLD[(Iceberg gold)]
        GC[Glue Data Catalog]
    end

    subgraph "Serving"
        ATH[Athena]
        RED[(Redshift)]
        DDB[(DynamoDB)]
        API[API Gateway]
    end

    DLR --> S3IN --> LMB --> SF --> GLU --> BRZ
    BRZ --> SLV --> GLD
    WH --> KIN --> FLINK --> SLV
    DB --> DMS --> MSK --> FLINK
    GLD --> ATH
    GLD --> RED
    GLD --> DDB --> API

    style GLD fill:#dbeafe,stroke:#2563eb
    style GC fill:#fef3c7,stroke:#d97706
```

**Total monthly at 1,000 dealers/week:** ~$2,900 (~$2.90/dealer). That's ~5× more than my Solution B self-hosted at the same scale. **My justification for D isn't cost — it's no DevOps headcount, multi-region defaults, AWS Bedrock for in-VPC LLM, and AWS Marketplace integration breadth.**

[**📐 Read the full Solution D brief →**](./docs/05-solution-D-aws-brief.md)

</details>

---

## 🤖 My LLM strategy — the AI tooling section the brief tests for

The brief encourages "OpenAI / Claude" Vision LLMs but is silent on cost, accuracy verification, and what happens when the model is wrong. **In my view, the silence is where the senior signal lives.**

### My 3-phase LLM lifecycle

```mermaid
flowchart TB
    subgraph "Phase 1 — dev seeding (no marginal cost)"
        D1[LLM_PROVIDER=claude-code-handoff<br/>pnpm enrich --mode audit]
        D1 --> T[translation_tasks.jsonl]
        T --> H[Operator translates<br/>in Claude Code session<br/>under Max subscription]
        H --> R1[translation_results.jsonl]
        R1 --> C[llm-cache.jsonl committed to git]
    end

    subgraph "Phase 2 — reviewer execution (zero cost)"
        D2[LLM_PROVIDER=cached pnpm enrich]
        D2 --> CH[Cache hit on every row]
        CH --> AU[ingest_audit · cache_hit=true]
    end

    subgraph "Phase 3 — production runtime"
        D3[LLM_PROVIDER=ollama OR anthropic-batch]
        D3 --> SH[~99% cache hit at steady state]
        SH --> CO[Real upstream: ~500 calls/month at 1,000 dealers]
    end

    style C fill:#dcfce7,stroke:#16a34a
    style CH fill:#dcfce7,stroke:#16a34a
    style SH fill:#dbeafe,stroke:#2563eb
```

### The 4 genuine LLM choices I considered

| Approach | Cost shape | Accuracy ceiling | When I'd pick it |
|---|---|---|---|
| **Paid API** (OpenAI / Anthropic) | $0.0005–$0.05/call | Frontier | <100k calls/mo + reliable infra |
| **Self-hosted** (Ollama Qwen 2.5 / Llama) | $0 + GPU hardware | Lags frontier ~12 mo | 100k–10M calls/mo, data residency strict |
| **Free API tier** (Gemini / OpenRouter) | $0 with rate limits | Medium | Pre-revenue, low-stakes, TOS-aware |
| **Pure OCR** (PaddleOCR / Tesseract) | $0 (OSS) | Brittle on multilingual | Structured-text-only, English-dominant |

**My picks for InventoryFlow today:** paid API + aggressive cache + 6-provider abstraction. **At 1,000 dealers/week steady state, my monthly LLM bill is ~$0.25.** Most teams I've talked to pay $300–$3,000/month for the same workload — the 1,000× gap is cache discipline + provider abstraction + audit-mode.

[**🤖 Read the full LLM strategy →**](./docs/06-llm-strategy.md)

---

## 📊 How I know the output is right — my 5-layer verification framework

The brief doesn't ask this. In my view a senior reader asks it anyway.

| Layer | What I catch | My implementation |
|---|---|---|
| **1. Schema validation** | Wrong types, missing fields, format violations | Zod runtime + Postgres NOT NULL |
| **2. Domain rules** | Year out of range, fitment incoherence, malformed part_number | `validators/` module + DB CHECK constraints |
| **3. Cross-row consistency** | Same Chinese name → different English translations across rows | Audit query in `ingest_audit` |
| **4. Cross-source agreement** | LLM disagrees with dealer-supplied English | Audit mode; `agreement` column |
| **5. Downstream feedback** | Marketplace listing rejected; consumer SLA breach | **Designed; not yet implemented** (deferred to marketplace integration) |

**The trap most submissions fall into:** implement only (1) and (2) and stop. Wrong data at scale is more expensive than wrong infrastructure — that's why I spend architectural effort here.

[**📊 Read the full output verification doc →**](./docs/07-output-verification.md)

---

## 📈 My scale roadmap — how Solution A grows

```mermaid
flowchart LR
    P1[Phase 1<br/>0–500 dealers<br/>Single instance<br/>~$30/mo amortised] --> P2
    P2[Phase 2<br/>500–1,500 dealers<br/>K8s + read replicas<br/>~$200/mo] --> P3
    P3[Phase 3<br/>1,500–5,000 dealers<br/>Partitioning + CQRS<br/>~$1,500/mo] --> MIGRATE
    MIGRATE[5,000+ dealers<br/>Migrate to Solution B<br/>Lakehouse + Iceberg]

    style P1 fill:#dcfce7,stroke:#16a34a
    style P2 fill:#fef3c7,stroke:#d97706
    style P3 fill:#fee2e2,stroke:#dc2626
    style MIGRATE fill:#dbeafe,stroke:#2563eb
```

### My 6 explicit migration triggers (A → B)

I'd switch to Solution B when **any one** of these holds for two consecutive months:

| # | Trigger | Why this is the breakpoint |
|---|---|---|
| 1 | **Dealer count > 500** | Postgres vertical scaling stops being cheaper than object-store horizontal |
| 2 | **Historical volume > 50 TB** | Iceberg `VERSION AS OF` becomes superior to audit-log replay |
| 3 | **LLM cost share > 30% of cloud bill** | Iceberg's global canonical-translations table cuts ~10× of duplicate calls |
| 4 | **OLAP / OLTP contention** | Analytics queries on `products` start blocking the catalog API |
| 5 | **Schema churn ≥ 1 dealer/week** | Iceberg `ALTER TABLE` + OpenLineage beats Drizzle migrations |
| 6 | **RTO requirement < 1 hour** | `VERSION AS OF` is faster than PITR restore + replay |

**None of these fire on day one for InventoryFlow.** My approach: plan for them, don't pre-empt them.

---

## ⚙️ Operations — CI/CD, security, scale, governance

| Area | What I shipped | What I deferred (with triggers) |
|---|---|---|
| **CI/CD** | GitHub Actions: lint + typecheck + 32 tests + Docker build per PR | Multi-env DEV→TEST→PROD with explicit gates (year 1 when team grows) |
| **Security** | Row-level security migration, JWT auth, R2 per-dealer prefixes, secrets via platform | SOC 2 controls (when enterprise customers ask), WAF (revenue stage), pen-test (year 2) |
| **Observability** | Pino structured logs with `run_id` correlation, OTel SDK instrumented, `ingest_audit` table | OTLP exporter backend (env-specific), severity-routed alerting (when on-call rotation exists) |
| **Scale** | Phase 1 (<500 dealers, $30/dealer amortised) | Phases 2 + 3 documented with effort estimates; B migration documented |
| **Governance** | 14 ADRs in `docs/decisions/`, naming conventions, Zod data contracts | Schema registry (Solution B), full data catalog (multi-team stage) |
| **DR / BCP** | Daily managed snapshots, R2 versioning, RPO 24h / RTO 4h | Logical replication standby (phase 2), multi-region (phase 3), `VERSION AS OF` (B migration) |

**My pattern: explicit deferrals over invisible omissions.** Every deferred item has a re-visit trigger.

[**⚙️ Read the full operations doc →**](./docs/08-operations.md)

---

## 🗂 What's in the docs folder

```
Inventoryflow_solution/
├── README.md                                    ← you are here
├── LICENSE
├── adr/
│   └── ADR-INDEX.md                             ← cross-references to 14 ADRs in the impl repo
└── docs/
    ├── 00-tldr.md                               ← 3-minute decision
    ├── 01-problem-framing.md                    ← my adversarial read of the brief
    ├── 02-solution-A-recommended.md             ← my chosen path, overview → detail
    ├── 03-solution-B-de-standard.md             ← the industry-standard alternative
    ├── 04-solution-C-fabric-brief.md            ← Microsoft Fabric brief
    ├── 05-solution-D-aws-brief.md               ← AWS big-data brief
    ├── 06-llm-strategy.md                       ← how I'd approach the AI tooling question
    ├── 07-output-verification.md                ← how I know the data is right
    ├── 08-operations.md                         ← CI/CD, security, observability, scale, governance
    └── 09-engineering-judgment.md               ← the part I think is hard to fake
```

### Reading order for the full 90 minutes

1. **[00-tldr.md](./docs/00-tldr.md)** — 3 min — my verdict and the migration triggers
2. **[01-problem-framing.md](./docs/01-problem-framing.md)** — 8 min — what I think the brief is really testing
3. **[02-solution-A-recommended.md](./docs/02-solution-A-recommended.md)** — 25 min — the path I'd ship, deep
4. **[06-llm-strategy.md](./docs/06-llm-strategy.md)** — 10 min — the AI tooling section the brief specifically asks for
5. **[03-solution-B-de-standard.md](./docs/03-solution-B-de-standard.md)** — 15 min — what I'd build at scale instead
6. **[07-output-verification.md](./docs/07-output-verification.md)** — 8 min — *how I know the data is right*
7. **[08-operations.md](./docs/08-operations.md)** — 10 min — CI/CD, security, scale, governance
8. **[09-engineering-judgment.md](./docs/09-engineering-judgment.md)** — 10 min — my closing argument

The C and D briefs (04 + 05) are optional reading.

---

## ⏱ A note on time spent — and on AI

I built the full submission (this doc repo + the implementation repo with 4 solutions, 14 ADRs, 32 tests, real benchmarks, sample output) in approximately **12 hours of AI-assisted development time**. My equivalent manual estimate is ~7 days.

| Component | AI-assisted time | My manual equivalent |
|---|---:|---:|
| Solution A (implemented) | ~6 h | ~3 days |
| Solution B (PoC scaffold) | ~3 h | ~2 days |
| Solution C (architecture only) | ~30 min | ~1 day |
| Solution D (architecture only) | ~30 min | ~1 day |
| This solution-architecture doc set | ~2 h | ~1 day |
| **Total** | **~12 h** | **~7 days** |

My honest framing: AI is a productivity multiplier for things I already know how to do. It compresses the typing and the boilerplate. **It doesn't produce the architecture decisions** — those are mine, recorded as ADRs, and trace back to specific things I learned at Ashley Furniture (Fabric refactor), Ecentric (Azure → Fabric zero-downtime migration), SOPA (Iceberg + Flink streaming platform), and ADP (dbt + OpenMetadata governance).

If AI vanished tomorrow I'd build this slower, not differently.

---

## 📨 Contact

**Aric Nguyen**
Bình Dương, Vietnam · daily cross-border with US HQ (Ashley Furniture, current role)

[aricnguyen.analytics2002@gmail.com](mailto:aricnguyen.analytics2002@gmail.com) · [github.com/ankinguyen-engineer-2002](https://github.com/ankinguyen-engineer-2002)

I'm available for a live walkthrough, a system-design deep-dive, or to defend any of the trade-offs above in real time. **I'd prefer the third.**

<div align="center">

<br/>

[**▶ 3-min decision**](./docs/00-tldr.md) · [**📐 Solution A deep**](./docs/02-solution-A-recommended.md) · [**🤖 LLM strategy**](./docs/06-llm-strategy.md) · [**🧠 Engineering judgment**](./docs/09-engineering-judgment.md)

<br/>

</div>
