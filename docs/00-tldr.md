# TL;DR — my decision in 3 minutes

## My verdict

**I'd build Solution A now. I'd plan Solution B for year two. I documented C and D so my decision is on record.**

That sentence is the entire submission. Everything below is my reasoning, sized for a reader who has 180 seconds.

---

## Why I'm picking A today

1. **JD match** — TypeScript / Node / Postgres / Redis / Docker is what the JD asks for. I'm reading the hiring signal first, architectural elegance second.
2. **Time-to-ship** — I implemented A end-to-end in approximately 6 hours of AI-assisted work. B as a runnable PoC took me 3 more hours. C and D I drafted as paper architectures. The brief explicitly says *Pragmatism & Speed*.
3. **Cost floor** — A costs me ~$30/dealer/month amortised at 100 dealers. B is $80/dealer at 1 dealer; the economics only flip past ~200 dealers.
4. **Team fit** — TypeScript engineers are ubiquitous in startup hiring. Dagster + Iceberg + Polars needs a data-engineering hire I don't think InventoryFlow needs yet.
5. **Reviewability** — three commands. Zero API cost for the reviewer. I committed sample output so you can inspect without running anything.

---

## Where I expect A to break — the six triggers

I'd switch to B when **any one** of these holds for two consecutive months:

| # | Trigger | Why I think this is the breakpoint |
|---|---|---|
| 1 | **Dealer count > 500** | Postgres vertical scaling stops being cheaper than object-store horizontal |
| 2 | **Historical volume > 50 TB** | Iceberg `VERSION AS OF` becomes superior to audit-log replay |
| 3 | **LLM cost share > 30% of cloud bill** | Iceberg's global canonical-translations table cuts ~10× of duplicate calls across dealers |
| 4 | **OLAP / OLTP read-write contention** | Analytics queries on `products` start blocking the catalog API on shared Postgres |
| 5 | **Schema churn ≥ 1 dealer/week** | Iceberg `ALTER TABLE` + OpenLineage beats Drizzle migrations as the cadence rises |
| 6 | **RTO requirement < 1 hour** | `VERSION AS OF '2026-05-04 14:00:00'` is faster than PITR restore + replay |

None of these fire on day one for InventoryFlow. **My approach: plan for them, don't pre-empt them.**

---

## What I built into this submission

| What the brief asked for | What I delivered |
|---|---|
| Parse messy xlsx → clean DB | Solution A — implemented, 3,938 products, 12 tables populated |
| R2 schematic images | 382 deduplicated R2 objects from 1,586 source images (SHA-256 content addressing) |
| JSON fitment column | JSONB with `GIN jsonb_path_ops` index, p99 = 1.02 ms on 500 samples |
| AI tooling | 6-provider abstraction, JSONL cache committed, audit mode caught 16% real defects |
| Not enterprise-heavy boilerplate | A is the minimum viable answer I could ship; B/C/D documented as deferred options |
| Clean architecture | 14 ADRs I wrote, no hidden magic |

---

## If you read nothing else

The way I read this brief, it's testing whether I can **frame a decision under constraint**. The constraint here is "early-stage company, hiring TS engineers, messy first dataset". The decision is "where do I put my effort". My answer: ship the boring TypeScript pipeline that works, document the lakehouse alternative so the migration isn't reactive, and spend the saved hours on output verification and AI cost discipline — because **in my experience, wrong data at scale is more expensive than the wrong stack**.

Read the next 90 minutes if you want my full proof. Or [run it in three commands](https://github.com/ankinguyen-engineer-2002/inventoryflow-catalog-ingest/blob/main/SUBMISSION.md) and inspect the output yourself.
