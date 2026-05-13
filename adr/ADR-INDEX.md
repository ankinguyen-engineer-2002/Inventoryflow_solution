# My ADR Index

> My Architecture Decision Records live in the implementation repo at [`docs/decisions/`](https://github.com/ankinguyen-engineer-2002/inventoryflow-catalog-ingest/tree/main/docs/decisions). I cross-reference them here with the architecture documents in this repo.
>
> My ADR format is the standard Michael Nygard template: **Context · Decision · Consequences · Alternatives**.

---

## Why I write ADRs at all

In my experience, a decision that isn't recorded gets re-litigated every 6 months. I use ADRs to convert "we chose X because Y" into a stable historical record. At Ashley Furniture I maintain ~40 ADRs across the platform — my InventoryFlow submission has 14 because the platform is smaller and younger.

My discipline: **every non-obvious choice gets an ADR before the code that depends on it lands**. Reverse-engineering an ADR from existing code is, in my view, a tell that the discipline broke.

---

## My 14 ADRs

| # | Topic | My decision summary | Related doc |
|---|---|---|---|
| **001** | Two-track monorepo | One repo, two `track-*` subdirectories. Track A is JD-native; Track B is OSS DE alternative. Shared modules in `shared/`. | [README](../README.md) |
| **002** | JSONB fitment design | Denormalised JSONB on `products` with `GIN jsonb_path_ops`. I measured sub-50ms containment queries. | [02-solution-A](../docs/02-solution-A-recommended.md#jsonb-fitment-design--my-approach-to-the-test-specifications-hot-path) |
| **003** | SHA-256 idempotent images | Content addressing for R2 keys; HEAD before PUT. My measured 76% dedup on reference data. | [02-solution-A](../docs/02-solution-A-recommended.md#image-handling--my-sha-256-idempotency-and-r2-upload-approach) |
| **004** | Drizzle vs Prisma | Drizzle wins on concrete JSONB type inference. Prisma's `Json` collapses to `unknown`. | [02-solution-A](../docs/02-solution-A-recommended.md#my-tech-stack-rationale) |
| **005** | Section detection strategy | Header-signature regex match, not row-index heuristics. Fails loud on unknown signatures. | [02-solution-A](../docs/02-solution-A-recommended.md#ten-engineering-decisions-id-defend) + [07-output-verification](../docs/07-output-verification.md#drift-detection--the-silent-failure-mode) |
| **006** | Part number aliases | Engine sheets ship OLD/NEW part number pairs; I preserved them as `part_number_aliases` table with cross-references. | impl repo |
| **007** | LLM provider cost strategy | 6-provider abstraction with cache decorator default. Reviewer runs at $0; production swaps via env var. | [06-llm-strategy](../docs/06-llm-strategy.md) |
| **008** | Medallion + Iceberg + Dagster (Solution B) | My Solution B uses Iceberg + Dagster + dbt; bronze/silver/gold separation; OpenLineage events. | [03-solution-B](../docs/03-solution-B-de-standard.md) |
| **009** | When to switch tracks | Six explicit migration triggers I defined (dealer count, volume, LLM cost share, OLAP contention, schema churn, RTO requirement). | [00-tldr](../docs/00-tldr.md#where-i-expect-a-to-break--the-six-triggers) |
| **010** | Batch + streaming hybrid | `pg_notify` outbox for streaming; transactional consistency via outbox pattern; swap to Redpanda when volume justifies. | [02-solution-A](../docs/02-solution-A-recommended.md#ten-engineering-decisions-id-defend) |
| **012** | Data contracts + schema registry | Zod schemas are my contract at Solution A. Schema-registry-based contracts deferred to Solution B. | [07-output-verification](../docs/07-output-verification.md) + [08-operations](../docs/08-operations.md#my-data-contracts) |
| **013** | DR / BCP / RPO / RTO | Phase-specific targets. Phase 1 = 24h RPO / 4h RTO; migration to B unlocks `VERSION AS OF` for sub-15min RTO. | [08-operations](../docs/08-operations.md#disaster-recovery--business-continuity) |
| **014** | Metadata-driven control plane | Three registry tables seeded (`dealers`, `ingestion_patterns`, `dealer_pattern_bindings`); runtime dispatcher deferred to dealer #2. | [02-solution-A](../docs/02-solution-A-recommended.md#ten-engineering-decisions-id-defend) |
(I skipped ADR-011 during numbering — a small discipline failure of mine; I'm calling it out so the gap is visible rather than retroactively renumbered. ADR-015+ are tracked in the implementation repo and not yet referenced here until the corresponding measurements are verified end-to-end.)

---

## My cross-reference: docs in this repo → ADRs

| Doc | Most relevant ADRs |
|---|---|
| [00-tldr](../docs/00-tldr.md) | 009 (migration triggers) |
| [01-problem-framing](../docs/01-problem-framing.md) | 001, 014 |
| [02-solution-A-recommended](../docs/02-solution-A-recommended.md) | 002, 003, 004, 005, 006, 007, 010, 014 |
| [03-solution-B-de-standard](../docs/03-solution-B-de-standard.md) | 008, 009 |
| [04-solution-C-fabric-brief](../docs/04-solution-C-fabric-brief.md) | (cross-references ADRs in Ashley Furniture's internal docs, not this repo) |
| [05-solution-D-aws-brief](../docs/05-solution-D-aws-brief.md) | — |
| [06-llm-strategy](../docs/06-llm-strategy.md) | 007 |
| [07-output-verification](../docs/07-output-verification.md) | 005, 012 |
| [08-operations](../docs/08-operations.md) | 012, 013, 014 |
| [09-engineering-judgment](../docs/09-engineering-judgment.md) | All (this is my meta-doc) |

---

## How I'd read an ADR

Each ADR file in the impl repo follows:

```markdown
# ADR-NNN: Title

## Context
What was the situation? What constraints? What was driving the decision?

## Decision
What I decided.

## Consequences
What I expect to follow — good and bad.

## Alternatives considered
What else I had on the table; why I rejected each.
```

In my view the most important section is often **Alternatives considered**. An ADR with no alternatives is suspect to me.

---

## What I deliberately *don't* turn into ADRs

To prevent ADR sprawl, I deliberately don't ADR the following:

- **Library version pins** — go in `package.json` with comments
- **Coding style choices** — go in `.eslintrc.json` or `tsconfig.json`
- **One-line code changes** — git commit messages are sufficient
- **Decisions reversed in <1 month** — not stable enough to record

My threshold: "if a future engineer would ask 'why did we do it this way?' and the answer takes more than 2 sentences", it's an ADR for me.

---

**Back to:** [README](../README.md)
