# InventoryFlow — Solution Architecture

## Hi, I'm Aric 👋

I'm a Data Engineer / Solution Architect based in Bình Dương, Vietnam, with 4 years of building production data platforms across Microsoft Fabric, Azure, Databricks, and open-source lakehouses. I'm currently at Ashley Furniture Industries (US HQ-aligned), where I lead the global supply-chain analytics platform refactor — 5,000+ enterprise tables, 30 years of history, billions of records. Before that I owned end-to-end data platforms at Ecentric (Mar 2023 – Jan 2026), co-built a streaming-capable Iceberg + Flink + Trino platform at SOPA/TheSocietyPass, and got promoted from Data Analyst Intern to Data Engineer in 3 months at ADP.

What that means in plain terms: I've made enough wrong architectural calls in production to have opinions about which ones matter, and I've made enough right ones to be willing to defend them in a design review. This submission is how I'd answer the InventoryFlow brief — recommendation first, reasoning second, alternatives third.

**Companion repo:** [`inventoryflow-catalog-ingest`](https://github.com/ankinguyen-engineer-2002/inventoryflow-catalog-ingest) — my Solution A running end-to-end in three commands.

---

## My summary of the brief

InventoryFlow receives parts catalog data from hundreds of distributors and OEMs in messy formats — PDFs, xlsx, multilingual text, schematic images. The ask:

- Standardise messy multi-format input into a clean database
- Upload schematic images to Cloudflare R2
- Tabular columns: part number, English name, Chinese name
- A JSON column listing every {year, make, model} the part fits
- Use AI tooling to parse the messy content
- "Pragmatism & Speed", "not enterprise-heavy boilerplates"
- Clean architecture, especially the JSONB fitment column

I read the brief as a one-OEM, sub-100-dealer pilot today. The actual sample file I received is a 241 MB Kayo ATV catalog xlsx with 110 sheets and 1,586 embedded schematic images. The implementation I shipped processes it in ~60 seconds wall-time with measured fitment-query latency p99 = 1.02 ms.

[`docs/01-problem-framing.md`](./docs/01-problem-framing.md) is where I read the brief adversarially and find the gaps the brief doesn't say out loud.

---

## But here's what I keep asking myself

The brief is one OEM, one xlsx, one pilot. In my experience, that's never the steady state. So I want to put a few harder questions on the table before I commit to *any* architecture:

### Question 1 — what if there are 10,000 files a day?

A single 241 MB xlsx every few hours is comfortable for Node + BullMQ + Postgres. 10,000 messy files per day from 200 dealers, each with their own schema variant, is not. Postgres write throughput at that point is the wrong tool; per-installation LLM caching is the wrong design; one-team-owns-everything is the wrong org structure. **My Solution A doesn't survive this scale.** Solution B (Polars + Iceberg + Dagster + dbt) does.

### Question 2 — what if there are 1 million files a day?

Now we're not even in the same architectural universe. At 1M/day the bottleneck stops being software and becomes **physics**: object-storage request rates, metadata-service throughput, network egress costs, multi-region replication, schema-evolution velocity across hundreds of independent dealer feeds. This is where I'd reach for Solution D (AWS Big Data + Streaming with managed Kafka, Glue Catalog, Kinesis, Step Functions) — or its Azure / GCP equivalents. The cost structure flips completely: I'd stop optimising for "low floor" and start optimising for "low marginal cost per million events".

### Question 3 — what if the input isn't a file at all?

What if InventoryFlow is sitting downstream of a constellation of *upstream systems* — dealer ERPs, OEM SAP, supplier portals, marketplace webhooks, IoT telemetry — and the xlsx is just the **artefact that the analytics / BI / DA teams want to see at the end**, not the actual integration surface? That's the realistic enterprise shape, and it's something I built at Ecentric and SOPA. The right answer there isn't "parse better"; it's:

- **CDC ingestion** from source DBs via Debezium / DMS / Fabric Mirroring
- **Streaming ingestion** of webhooks via Redpanda / Kinesis / Event Hubs
- **Batch ingestion** of legacy file drops via Airbyte / ADF / Glue
- **Schema registry** to handle dealer-by-dealer schema evolution
- A **semantic boundary** (Direct Lake / dbt / Iceberg Gold) where the analytics / BI / DA teams consume *governed* data
- Excel becomes an *export* of a queryable warehouse, not an input

This is Solution C territory if the company is Microsoft-aligned (the architecture I build at Ashley today), or Solution B + D if not.

### Why this matters for the recommendation

In my view, you can't recommend an architecture without knowing which of these three futures the company is heading toward. If InventoryFlow is "one xlsx per dealer per quarter, 50 dealers" → Solution A is correct and the others are over-engineering. If it's "10,000 dealer feeds, real-time inventory" → Solution A is the wrong starting point, even today. **The brief doesn't tell me which one it is, so I drafted answers for all four shapes and made my recommendation conditional on the read.**

---

## My personal philosophy on this

To run a business reliably on data, in my opinion, you need solid infrastructure underneath. To build solid infrastructure, the data engineer responsible has to:

1. **Master the frameworks** — Spark, Iceberg, dbt, Dagster, Kafka, the cloud platforms — at a depth where you can debug them under pressure
2. **Master the process** — ingestion patterns, CDC, schema evolution, idempotency, lineage, DQ contracts, RPO / RTO, cutover gates
3. **Master the end-to-end workflow** — from source extraction through serving to consumer feedback, and back through cost monitoring to architectural revision

**But that's not enough.** What separates a data engineer from a solution architect, in my view, is the willingness to think one level above all of that:

- **What risks haven't shown up yet?** Schema drift, cost overruns, vendor lock-in, talent attrition, regulatory change, source-system collapse
- **What does scale do to the architecture?** Not "will it handle more rows" (every modern stack does that), but "at what dealer count does my Postgres become the bottleneck, and is my migration path to a lakehouse ready?"
- **When do I switch platforms or tech stacks?** Fabric → Databricks, Postgres → Iceberg, paid LLM → self-hosted — these are decisions you don't make under pressure, you make them with a trigger and a plan

That's why I drafted **four** solutions, not one. I treat each as a credible end-state. The recommendation is conditional on which end-state the company is heading toward.

**For this brief's scale (one OEM, ~100 dealers, hiring TS engineers), my recommendation is Solution A** — and I'll defend it across three scale tiers (today, year 2, year 3) in [`docs/02-solution-A-recommended.md`](./docs/02-solution-A-recommended.md). The other three solutions are documented at honest depth so the recommendation isn't ignorance of the alternatives.

---

## The four solutions on one page

| | 🟢 **A — JD-Native** | 🟡 **B — DE Standard** | 🔵 **C — MS Fabric** | 🟣 **D — AWS Big Data** |
|---|---|---|---|---|
| **My status** | I implemented this end-to-end | I scaffolded as a runnable PoC | I drafted the architecture + diagrams | I drafted the architecture + diagrams |
| **Stack** | TS · Node · Postgres · Redis · BullMQ · R2 | Polars · Iceberg · Dagster · dbt · Redpanda · RisingWave | OneLake · Lakehouse · Pipelines · Eventhouse · Direct Lake · Activator | S3+Iceberg · Glue · Kinesis · MSK · Lambda · Step Functions · Athena |
| **My pick for** | <500 dealers, 1 OEM, pilot stage | 500–5,000 dealers, multi-OEM, OSS-first | Enterprise / Microsoft shops with capacity commitment | Cloud-native big-data + streaming on AWS |
| **Why I'd pick it** | Matches JD, ships in days, $30/dealer at 100 dealers | Iceberg time-travel, global LLM dedup, asset-graph lineage | Metadata-driven control plane for free, Direct Lake is genuinely best-in-class | Multi-region streaming, vendor-managed throughout |
| **Why I wouldn't pick it now** | (this is what I picked) | Python on a TS team, 10-service compose, no MERGE-on-small-data benefit yet | Fabric capacity is a hard ~$1k/mo floor commitment | AWS lock-in plus ops overhead doesn't pay back at <1,000 dealers |
| **My deep read** | [02-solution-A](./docs/02-solution-A-recommended.md) | [03-solution-B](./docs/03-solution-B-de-standard.md) | [04-solution-C](./docs/04-solution-C-fabric-brief.md) | [05-solution-D](./docs/05-solution-D-aws-brief.md) |

> **Why I'm presenting four when you asked for one.** In my reading the brief's actual test is *can you reason like an architect under constraint* — not *can you write TypeScript*. I'd lose the point if I only showed the answer; showing the work is how I show whether the answer was lucky or thought-through.

---

## My recommendation in one paragraph

**I'd build Solution A now, plan Solution B for year two, document C and D so my decision is on record.** For where InventoryFlow is today — one OEM, sub-100 dealers, a hiring signal that says TypeScript / Node / Postgres / Redis / Docker — A is the right shape. I'd reach for B at 500+ dealers, C if the company commits to Microsoft enterprise, D if AWS. The thing I won't do is build B / C / D today — that would be the over-engineering the brief explicitly warns against.

If you only have **3 minutes**, read [`docs/00-tldr.md`](./docs/00-tldr.md).
If you have **15 minutes**, read this README + [`docs/02-solution-A-recommended.md`](./docs/02-solution-A-recommended.md).
For the whole argument, the docs are ordered for a 90-minute read.

---

## What's in this repo

```
Inventoryflow_solution/
├── README.md                                    ← you are here
├── LICENSE
├── adr/
│   └── ADR-INDEX.md                             ← cross-references to 14 ADRs in the impl repo
├── diagrams/                                    ← Mermaid C4 + sequence + data flow (inline in docs)
└── docs/
    ├── 00-tldr.md                               ← 3-minute decision
    ├── 01-problem-framing.md                    ← my adversarial read of the brief
    ├── 02-solution-A-recommended.md             ← my chosen path, overview → detail
    ├── 03-solution-B-de-standard.md             ← the industry-standard alternative I'd build at scale
    ├── 04-solution-C-fabric-brief.md            ← Microsoft Fabric brief
    ├── 05-solution-D-aws-brief.md               ← AWS big-data brief
    ├── 06-llm-strategy.md                       ← how I'd approach the AI tooling question
    ├── 07-output-verification.md                ← how I know the output is right
    ├── 08-operations.md                         ← CI/CD, security, observability, scale, governance
    └── 09-engineering-judgment.md               ← the part I think is hard to fake
```

I kept the implementation in a separate repo so this one stays readable. **This repo is for reading; the impl repo is for running.**

| If you want to know… | This repo gives you… | The impl repo gives you… |
|---|---|---|
| Why these four solutions? | docs/01 + docs/02–05 | — |
| Which one I'd ship today? | docs/02 + 00-tldr | `track-a-jd-native/` code |
| Why I made those tech choices? | docs/02 (rationale tables) | code + 14 ADRs in `docs/decisions/` |
| What it looks like running? | sequence diagrams in `diagrams/` | `SUBMISSION.md` + sample-output |
| What about scale? | docs/08 + scale roadmap in docs/02 | benchmark numbers in `docs/bench/` |
| What about cost? | docs/02 + docs/06 (LLM section) | LLM audit + cache hit rate evidence |
| What about governance? | docs/08 | RLS migrations + ADRs |

---

## How I'd read this in 90 minutes

My suggested order:

1. **[00-tldr.md](./docs/00-tldr.md)** — 3 min — my verdict and the migration triggers
2. **[01-problem-framing.md](./docs/01-problem-framing.md)** — 8 min — what I think the brief is really testing
3. **[02-solution-A-recommended.md](./docs/02-solution-A-recommended.md)** — 25 min — the path I'd ship, deep
4. **[06-llm-strategy.md](./docs/06-llm-strategy.md)** — 10 min — the AI-tooling section the brief specifically asks for
5. **[03-solution-B-de-standard.md](./docs/03-solution-B-de-standard.md)** — 15 min — what I'd build at scale instead
6. **[07-output-verification.md](./docs/07-output-verification.md)** — 8 min — *how I know the data is right*
7. **[08-operations.md](./docs/08-operations.md)** — 10 min — CI/CD, security, scale, governance
8. **[09-engineering-judgment.md](./docs/09-engineering-judgment.md)** — 10 min — my closing argument

The C and D briefs (04 + 05) are optional reading. I included them because I want it clear that picking A wasn't ignorance of enterprise and cloud-native alternatives — I weighed both and decided they don't fit InventoryFlow's current stage.

---

## What this submission is, and what it isn't

**It is**:
- A solution architecture document grounded in a working implementation
- My honest accounting of trade-offs, including the parts I'd skip if I were rushed
- Cross-referenced with 14 Architecture Decision Records I wrote in the companion repo
- Written assuming the reader has built systems before and will push back

**It isn't**:
- A vendor pitch — for every alternative I included a "why I wouldn't"
- A theory exercise — Solution A is running today, every number I quote is measured
- A code walkthrough — the impl repo has that; this one stays at architecture level
- 100% AI-generated prose. I used AI to accelerate drafting; the judgments and trade-offs are mine and trace back to four years of running production data platforms

If you find a claim I make that doesn't survive scrutiny, that's a useful conversation. I'd rather defend it live than have it pass on the merits of formatting.

---

## A note on time spent — and on AI

I built the full submission (this doc repo + the implementation repo with 4 solutions, 14 ADRs, 32 tests, real benchmarks, sample output) in approximately **12 hours of AI-assisted development time**. My equivalent manual estimate is ~7 days.

| Component | AI-assisted time | My manual equivalent |
|---|---:|---:|
| Solution A (implemented) | ~6 h | ~3 days |
| Solution B (PoC scaffold) | ~3 h | ~2 days |
| Solution C (architecture only) | ~30 min | ~1 day |
| Solution D (architecture only) | ~30 min | ~1 day |
| This solution-architecture doc set | ~2 h | ~1 day |
| **Total** | **~12 h** | **~7 days** |

My honest framing: AI is a productivity multiplier for things I already know how to do. It compresses the typing and the boilerplate. It doesn't produce the architecture decisions — those are mine, recorded as ADRs, and trace back to specific things I learned at Ashley Furniture (Fabric refactor), Ecentric (Azure → Fabric zero-downtime migration), SOPA (Iceberg + Flink streaming platform), and ADP (dbt + OpenMetadata governance).

If AI vanished tomorrow I'd build this slower, not differently.

---

## Contact

**Aric Nguyen**
Bình Dương, Vietnam · daily cross-border with US HQ (Ashley Furniture, current role)
[aricnguyen.analytics2002@gmail.com](mailto:aricnguyen.analytics2002@gmail.com) · [github.com/ankinguyen-engineer-2002](https://github.com/ankinguyen-engineer-2002)

I'm available for a live walkthrough, a system-design deep-dive, or to defend any of the trade-offs above in real time. I'd prefer the third.
