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

## 📚 Pick your reading path

| You have… | Read this | What you get |
|---|---|---|
| **3 minutes** | [`docs/00-tldr.md`](./docs/00-tldr.md) | My verdict + the 6 quantified migration triggers |
| **15 minutes** | This README, top to bottom | All four solutions with diagrams + my reasoning |
| **30 minutes** | This README + [`02-solution-A`](./docs/02-solution-A-recommended.md) | Full A deep-dive: tech rationale, JSONB design, LLM strategy, scale roadmap |
| **90 minutes** | All 12 docs (order suggested in [`reading-order`](#-reading-order-for-the-full-90-minutes)) | The whole argument, end-to-end |

---

## 💭 My personal opinion — the part I want to be explicit about

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

---

## 👤 Who's writing this

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
<summary><b>📄 Click to view my full CV (PDF + page images inline)</b></summary>

<br/>

**Download the PDF:** [`assets/CV_Aric_Nguyen.pdf`](./assets/CV_Aric_Nguyen.pdf)

<br/>

#### Page 1

![CV — page 1](./assets/cv-page-1.png)

#### Page 2

![CV — page 2](./assets/cv-page-2.png)

</details>

What that means in plain terms: I've made enough wrong architectural calls in production to have opinions about which ones matter, and I've made enough right ones to be willing to defend them in a design review. **This submission is how I'd answer the InventoryFlow brief — recommendation first, reasoning second, alternatives third.**

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

## 🟢 Solution A — what I'd ship today

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

### What's measured (not estimated)

| Metric | Value | Note |
|---|---|---|
| End-to-end wall-time | **~60 sec** | M2 Mac, full xlsx → DB + R2 + LLM audit |
| Fitment query latency | **p50 0.60 ms · p95 0.87 ms · p99 1.02 ms · max 1.32 ms** | 500 samples on 3,938 products |
| Image deduplication | **76%** | 1,586 source images → 382 unique R2 objects |
| LLM audit findings | **16%** disagreement | 11/68 sampled products surfaced real dealer defects |
| Test suite | **32 tests, <400 ms** | Parser + cache + env validation |
| Reviewer cost to run | **$0** | LLM cache committed; reviewer needs no API key |

[**📐 Read the full Solution A doc →**](./docs/02-solution-A-recommended.md)

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
