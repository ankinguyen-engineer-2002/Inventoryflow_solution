# InventoryFlow — Solution Architecture

> A four-tier solution architecture for the Talemy × InventoryFlow Senior Engineer brief, written as a working consultant would write it — recommendation first, reasoning second, alternatives kept honest.

**Author:** [Aric Nguyen](https://github.com/ankinguyen-engineer-2002) — Data Engineer / Solution Architect, 4 years on Fabric, Azure, Databricks, and open-source lakehouse stacks.
**Companion implementation repo:** [`inventoryflow-catalog-ingest`](https://github.com/ankinguyen-engineer-2002/inventoryflow-catalog-ingest) — running Solution A, end-to-end, in three commands.

---

## TL;DR — the recommendation in one paragraph

For the current stage of InventoryFlow (single OEM ingestion, sub-100 dealers, a hiring team that signaled TypeScript / Node / Postgres / Redis / Docker in the JD), **Solution A** is correct. A medallion lakehouse on Iceberg or a Fabric-native control plane *would be* correct two years from now at 500+ dealers, but shipping that today would be the kind of over-engineering the brief explicitly warns against. This document explains exactly why A wins today, exactly where it breaks, and exactly what replaces it when it does — with the alternative architectures (B, C, D) drafted at the level of detail a senior engineer would actually defend in a design review.

If you only have **3 minutes**, read [`docs/00-tldr.md`](./docs/00-tldr.md).
If you only have **15 minutes**, read this README + [`docs/02-solution-A-recommended.md`](./docs/02-solution-A-recommended.md).
If you want the whole argument, the docs are ordered for a 90-minute read.

---

## The four solutions on one page

| | 🟢 **A — JD-Native** | 🟡 **B — DE Standard** | 🔵 **C — MS Fabric** | 🟣 **D — AWS Big Data** |
|---|---|---|---|---|
| **Status** | Implemented, end-to-end | Runnable PoC scaffold | Architecture + diagrams | Architecture + diagrams |
| **Stack** | TS · Node · Postgres · Redis · BullMQ · R2 | Polars · Iceberg · Dagster · dbt · Redpanda · RisingWave | OneLake · Lakehouse · Pipelines · Eventhouse · Direct Lake · Activator | S3+Iceberg · Glue · Kinesis · MSK · Lambda · Step Functions · Athena |
| **Sweet spot** | < 500 dealers | 500–5,000 dealers | Enterprise / Microsoft shops | Cloud-native big-data + streaming |
| **Why pick it** | Matches JD, ships in days, $30/dealer infra at scale-out | Iceberg time-travel, global LLM dedup, asset-graph lineage | Metadata-driven control plane, Direct Lake serving, governed | Multi-region streaming, AWS-native, vendor-managed at scale |
| **Why not now** | (this is what we picked) | Python on a TS team, 10-service compose, no MERGE on small data needed yet | Fabric capacity is a hard commitment, Microsoft-centric | AWS lock-in, ops complexity, higher floor cost |
| **Deep read** | [02-solution-A](./docs/02-solution-A-recommended.md) | [03-solution-B](./docs/03-solution-B-de-standard.md) | [04-solution-C](./docs/04-solution-C-fabric-brief.md) | [05-solution-D](./docs/05-solution-D-aws-brief.md) |

> **Why present four when you only asked for one?** Because the brief's actual test is *can you reason like an architect*, not *can you write TypeScript*. Showing only the answer hides the work. Showing the work shows whether the answer was lucky or thought-through.

---

## What's in this repo

```
Inventoryflow_solution/
├── README.md                                    ← you are here
├── LICENSE
├── adr/
│   └── ADR-INDEX.md                             ← cross-references to 14 ADRs in the impl repo
├── diagrams/                                    ← Mermaid C4 + sequence + data flow
└── docs/
    ├── 00-tldr.md                               ← 3-minute decision
    ├── 01-problem-framing.md                    ← what the brief actually asks, restated honestly
    ├── 02-solution-A-recommended.md             ← the chosen path, overview → detail
    ├── 03-solution-B-de-standard.md             ← the industry-standard alternative
    ├── 04-solution-C-fabric-brief.md            ← Microsoft Fabric brief
    ├── 05-solution-D-aws-brief.md               ← AWS big-data brief
    ├── 06-llm-strategy.md                       ← free API vs paid vs self-host vs pure OCR
    ├── 07-output-verification.md                ← how do you know the output is right?
    ├── 08-operations.md                         ← CI/CD, security, observability, scale, governance
    └── 09-engineering-judgment.md               ← the part that's actually hard to fake
```

The implementation lives in a separate repo so this one stays readable. The trade is: **this repo is for reading; that repo is for running.**

| Question | Answer this repo gives | Answer the impl repo gives |
|---|---|---|
| Why these four solutions? | docs/01 + docs/02–05 | — |
| Which one ships today? | docs/02 + 00-tldr | code under `track-a-jd-native/` |
| Why those tech choices? | docs/02 (rationale tables) | code + 14 ADRs under `docs/decisions/` |
| What does it look like running? | sequence diagrams in `diagrams/` | `SUBMISSION.md` + sample-output |
| What about scale? | docs/08-operations | benchmark numbers in `docs/bench/` |
| What about cost? | docs/02 + docs/06 (LLM section) | LLM audit + cache hit rate evidence |
| What about governance? | docs/08 | RLS migrations + ADRs |

---

## How to read this in 90 minutes

Recommended order:

1. **[00-tldr.md](./docs/00-tldr.md)** — 3 min — the verdict and the migration triggers
2. **[01-problem-framing.md](./docs/01-problem-framing.md)** — 8 min — what the brief is really testing
3. **[02-solution-A-recommended.md](./docs/02-solution-A-recommended.md)** — 25 min — chosen path, deep
4. **[06-llm-strategy.md](./docs/06-llm-strategy.md)** — 10 min — the AI-tooling section that the brief specifically asks for
5. **[03-solution-B-de-standard.md](./docs/03-solution-B-de-standard.md)** — 15 min — the alternative I would have built at scale
6. **[07-output-verification.md](./docs/07-output-verification.md)** — 8 min — *how do you know the data is right?*
7. **[08-operations.md](./docs/08-operations.md)** — 10 min — CI/CD, security, scale, governance
8. **[09-engineering-judgment.md](./docs/09-engineering-judgment.md)** — 10 min — the closing argument

C and D briefs (04 + 05) are optional reading; they exist to demonstrate that the "build now vs migrate later" choice was made after considering the enterprise and cloud-native alternatives, not in ignorance of them.

---

## What this submission is, and what it isn't

**It is**:
- A solution architecture document grounded in a working implementation
- An honest accounting of trade-offs, including the parts I'd skip if I were rushed
- Cross-referenced with 14 Architecture Decision Records in the companion repo
- Written assuming the reader has built systems before and will push back

**It isn't**:
- A vendor pitch — every alternative includes "why not"
- A theory exercise — Solution A is running, every number quoted is measured
- A code walkthrough — the impl repo has that; this one stays at architecture level
- 100% AI-generated prose. AI accelerated drafting; the judgments are mine and trace back to four years of production data platforms ([CV](https://github.com/ankinguyen-engineer-2002))

If you spot a claim that doesn't survive scrutiny, that's a useful conversation. I would rather defend it live than have it pass on the merits of formatting.

---

## A note on time spent — and on AI

The full submission (this doc repo + the implementation repo with 4 solutions, 14 ADRs, 32 tests, real benchmarks, sample output) was built in approximately **12 hours of AI-assisted development time**. The equivalent manual estimate is ~7 days.

| Component | AI-assisted time | Manual equivalent |
|---|---:|---:|
| Solution A (implemented) | ~6 h | ~3 days |
| Solution B (PoC scaffold) | ~3 h | ~2 days |
| Solution C (architecture only) | ~30 min | ~1 day |
| Solution D (architecture only) | ~30 min | ~1 day |
| This solution-architecture doc set | ~2 h | ~1 day |
| **Total** | **~12 h** | **~7 days** |

The honest framing: AI is a productivity multiplier for things I already know how to do. It compresses the typing and the boilerplate. It does not produce the architecture decisions — those are mine, recorded as ADRs, and traceable to specific lessons from prior systems (e.g., Ashley Furniture's Fabric refactor, Ecentric's Azure→Fabric zero-downtime migration, SOPA's Iceberg + Flink streaming platform).

If AI vanished tomorrow I would build this slower, not differently.

---

## Contact

**Aric Nguyen**
Bình Dương, Vietnam · daily cross-border with US HQ (Ashley Furniture Industries, current role)
[aricnguyen.analytics2002@gmail.com](mailto:aricnguyen.analytics2002@gmail.com) · [github.com/ankinguyen-engineer-2002](https://github.com/ankinguyen-engineer-2002)

Available for live walkthrough, system-design deep-dive, or to defend any of the trade-offs above in real time.
