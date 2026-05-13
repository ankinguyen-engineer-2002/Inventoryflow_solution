# TL;DR — the decision in 3 minutes

## The verdict

**Build Solution A now. Plan Solution B for year two. Document C and D so the decision is on record.**

That sentence is the entire submission. Everything below is the reasoning, sized for a reader who has 180 seconds.

---

## Why A wins today

1. **JD match** — TypeScript / Node / Postgres / Redis / Docker is what the JD asks for. Hiring signal first, architectural elegance second.
2. **Time-to-ship** — A was implemented end-to-end in approximately 6 hours of AI-assisted work. B as a runnable PoC took 3 more hours. C and D are paper architectures. The brief explicitly says *Pragmatism & Speed*.
3. **Cost floor** — $30/dealer/month amortized at 100 dealers. B is $80/dealer at 1 dealer; the economics only flip past ~200 dealers.
4. **Team fit** — TypeScript engineers are ubiquitous in startup hiring. Dagster + Iceberg + Polars requires a data-engineering hire InventoryFlow doesn't need yet.
5. **Reviewability** — Three commands. Zero API cost to the reviewer. Sample output committed for inspection without running anything.

---

## Where A breaks — the six triggers

Switch to B when **any one** of these holds for two consecutive months:

| # | Trigger | Why this is the breakpoint |
|---|---|---|
| 1 | **Dealer count > 500** | Postgres vertical scaling stops being cheaper than object-store horizontal |
| 2 | **Historical volume > 50 TB** | Iceberg `VERSION AS OF` becomes superior to audit-log replay |
| 3 | **LLM cost share > 30% of cloud bill** | Iceberg's global canonical-translations table breaks ~10× of duplicate calls across dealers |
| 4 | **OLAP / OLTP read-write contention** | Analytics queries on `products` start blocking the catalog API on shared Postgres |
| 5 | **Schema churn ≥ 1 dealer/week** | Iceberg `ALTER TABLE` + OpenLineage beats Drizzle migrations as the cadence rises |
| 6 | **RTO requirement < 1 hour** | `VERSION AS OF '2026-05-04 14:00:00'` is faster than PITR restore + replay |

None of these fire on day one. **Plan for them, don't pre-empt them.**

---

## What this submission shows

| The brief asked for… | This submission delivers… |
|---|---|
| Parse messy xlsx → clean DB | Solution A — implemented, 3,938 products, 12 tables populated |
| R2 schematic images | 382 deduplicated R2 objects from 1,586 source images (SHA-256 content addressing) |
| JSON fitment column | JSONB with `GIN jsonb_path_ops` index, p99 = 1.02 ms on 500 samples |
| AI tooling | 6-provider abstraction, JSONL cache committed, audit mode caught 16% real defects |
| Not enterprise-heavy boilerplate | A is the minimum viable answer; B/C/D documented as deferred options |
| Clean architecture | 14 ADRs, no hidden magic |

---

## If you read nothing else

The brief is testing whether you can **frame a decision under constraint**. The constraint is "early-stage company, hiring TS engineers, messy first dataset". The decision is "where do you put your effort". My answer: ship the boring TypeScript pipeline that works, document the lakehouse alternative so the migration isn't reactive, and put the saved hours into output verification and AI cost discipline — because **wrong data at scale is more expensive than the wrong stack**.

Read the next 90 minutes if you want the proof. Or [run it in three commands](https://github.com/ankinguyen-engineer-2002/inventoryflow-catalog-ingest/blob/main/SUBMISSION.md) and inspect the output yourself.
