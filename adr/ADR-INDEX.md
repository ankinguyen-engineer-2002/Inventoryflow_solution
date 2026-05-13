# ADR Index

> Architecture Decision Records live in the implementation repo at [`docs/decisions/`](https://github.com/ankinguyen-engineer-2002/inventoryflow-catalog-ingest/tree/main/docs/decisions). This index cross-references them with the architecture documents in this repo.
>
> The ADR format is the standard Michael Nygard template: **Context · Decision · Consequences · Alternatives**.

---

## Why ADRs at all

A decision that isn't recorded gets re-litigated every 6 months. ADRs convert "we chose X because Y" into a stable historical record. At Ashley Furniture I maintain ~40 ADRs across the platform — the InventoryFlow submission has 14 because the platform is smaller and younger.

The discipline that matters: **every non-obvious choice gets an ADR before the code that depends on it lands**. Reverse-engineering an ADR from existing code is a tell that the discipline broke.

---

## The 14 ADRs

| # | Topic | Decision summary | Related doc |
|---|---|---|---|
| **001** | Two-track monorepo | One repo, two `track-*` subdirectories. Track A is JD-native; Track B is OSS DE alternative. Shared modules in `shared/`. | [README](../README.md) |
| **002** | JSONB fitment design | Denormalised JSONB on `products` with `GIN jsonb_path_ops`. Sub-50ms containment queries. | [02-solution-A](../docs/02-solution-A-recommended.md#jsonb-fitment-design--the-test-specifications-hot-path) |
| **003** | SHA-256 idempotent images | Content addressing for R2 keys; HEAD before PUT. 76% dedup on reference data. | [02-solution-A](../docs/02-solution-A-recommended.md#image-handling--sha-256-idempotency-and-r2-upload) |
| **004** | Drizzle vs Prisma | Drizzle wins on concrete JSONB type inference. Prisma's `Json` collapses to `unknown`. | [02-solution-A](../docs/02-solution-A-recommended.md#tech-stack-rationale) |
| **005** | Section detection strategy | Header-signature regex match, not row-index heuristics. Fails loud on unknown signatures. | [02-solution-A](../docs/02-solution-A-recommended.md#ten-engineering-decisions-worth-defending) + [07-output-verification](../docs/07-output-verification.md#drift-detection--the-silent-failure-mode) |
| **006** | Part number aliases | Engine sheets ship OLD/NEW part number pairs; preserved as `part_number_aliases` table with cross-references. | impl repo |
| **007** | LLM provider cost strategy | 6-provider abstraction with cache decorator default. Reviewer runs at $0; production swaps via env var. | [06-llm-strategy](../docs/06-llm-strategy.md) |
| **008** | Medallion + Iceberg + Dagster (Solution B) | Solution B uses Iceberg + Dagster + dbt; bronze/silver/gold separation; OpenLineage events. | [03-solution-B](../docs/03-solution-B-de-standard.md) |
| **009** | When to switch tracks | Six explicit migration triggers (dealer count, volume, LLM cost share, OLAP contention, schema churn, RTO requirement). | [00-tldr](../docs/00-tldr.md#where-a-breaks--the-six-triggers) |
| **010** | Batch + streaming hybrid | `pg_notify` outbox for streaming; transactional consistency via outbox pattern; swap to Redpanda when volume justifies. | [02-solution-A](../docs/02-solution-A-recommended.md#ten-engineering-decisions-worth-defending) |
| **012** | Data contracts + schema registry | Zod schemas are the contract at Solution A. Schema-registry-based contracts deferred to Solution B. | [07-output-verification](../docs/07-output-verification.md) + [08-operations](../docs/08-operations.md#data-contracts) |
| **013** | DR / BCP / RPO / RTO | Phase-specific targets. Phase 1 = 24h RPO / 4h RTO; migration to B unlocks `VERSION AS OF` for sub-15min RTO. | [08-operations](../docs/08-operations.md#disaster-recovery--business-continuity) |
| **014** | Metadata-driven control plane | Three registry tables seeded (`dealers`, `ingestion_patterns`, `dealer_pattern_bindings`); runtime dispatcher deferred to dealer #2. | [02-solution-A](../docs/02-solution-A-recommended.md#ten-engineering-decisions-worth-defending) |

(ADR-011 was skipped during numbering — a small discipline failure; called out so the gap is visible rather than retroactively renumbered.)

---

## Cross-reference: docs in this repo → ADRs

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
| [09-engineering-judgment](../docs/09-engineering-judgment.md) | All (this is the meta-doc) |

---

## How to read an ADR

Each ADR file in the impl repo follows:

```markdown
# ADR-NNN: Title

## Context
What was the situation? What constraints? What was driving the decision?

## Decision
What we decided.

## Consequences
What we expect to follow — good and bad.

## Alternatives considered
What else was on the table; why each was rejected.
```

The most important section is often **Alternatives considered**. An ADR with no alternatives is suspect.

---

## What's NOT an ADR

To prevent ADR sprawl, the following are deliberately not ADRs:

- **Library version pins** — go in `package.json` with comments
- **Coding style choices** — go in `.eslintrc.json` or `tsconfig.json`
- **One-line code changes** — git commit messages are sufficient
- **Decisions reversed in <1 month** — not stable enough to record

The threshold: "if a future engineer would ask 'why did we do it this way?' and the answer takes more than 2 sentences", it's an ADR.

---

**Back to:** [README](../README.md)
