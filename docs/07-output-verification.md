# Output Verification — *how do you know the data is right?*

> The brief doesn't ask this question. Senior architects ask it anyway. A data pipeline that produces wrong data confidently is worse than one that produces no data — and at scale, "wrong" is the default state unless you build the discipline to prevent it.

---

## The question, framed correctly

A naive framing: "are the outputs correct?" produces yes/no answers and a false sense of safety.

The senior framing decomposes into:

1. **Structural correctness** — does the row conform to the declared schema?
2. **Referential consistency** — do foreign keys, joins, and cross-row invariants hold?
3. **Domain correctness** — does the value make sense for the field's meaning?
4. **Temporal consistency** — does the data make sense relative to its history?
5. **Cross-source agreement** — do independent measurements agree?
6. **Downstream usability** — does the consumer of this data get the right answer?

A pipeline that handles only (1) and (2) is what most submissions ship. The brief implicitly tests for (3)–(6), via the JSONB fitment design ask and the AI tooling encouragement (both of which are domain-correctness conversations).

---

## The five-layer accuracy framework

This is the production discipline I apply at Ashley Furniture (preflight DQ gates with severity tiers) and at Ecentric (multi-checkpoint DQ framework with automated halt on critical failures). Translated to the InventoryFlow context:

| Layer | Catches | Implementation (Solution A) |
|---|---|---|
| **1. Schema validation** | Wrong types, missing required fields, format violations | Zod runtime + Postgres NOT NULL constraints |
| **2. Domain rules** | Year out of range, fitment that's incoherent, part_number malformed | `validators/` module + DB CHECK constraints |
| **3. Cross-row consistency** | Same Chinese name translated differently across rows; same part with conflicting fitment | Audit query in `ingest_audit` |
| **4. Cross-source agreement** | LLM disagrees with dealer-supplied English | Audit mode; `agreement` column |
| **5. Downstream feedback** | Marketplace listing rejected; consumer SLA breach | Designed; not yet implemented (deferred to marketplace integration) |

### Layer 1 — Schema validation

```typescript
// src/parse/normalize.ts (abbreviated)
const FitmentEntry = z.object({
  year: z.number().int().min(1990).max(2030).nullable(),
  make: z.string().min(1).max(50),
  model: z.string().min(1).max(100),
  model_code: z.string().min(1).max(50),
  variant: z.string().nullable(),
  category: z.enum(['SPORT_ATV', 'KIDS_ATV', 'UTILITY_ATV', /*...*/]),
  section: z.string().nullable(),
  callout_no: z.string().nullable(),
  confidence: z.enum(['high', 'medium', 'low']),
})

const ProductRow = z.object({
  part_number: z.string().regex(/^[A-Z0-9-]{3,40}$/),
  name_en: z.string().min(1).max(200),
  name_cn: z.string().min(1).max(200),
  fitment: z.array(FitmentEntry).min(1),
  // ...
})
```

The Zod schema is the single source of truth for both TypeScript types and runtime validation. If a dealer ships a year `1850`, this layer rejects it before it touches the database. **Failures here log the offending row with `run_id` correlation and continue the run** — the policy decision (fail-fast vs continue-with-skip) is a config flag, default to continue-with-skip with a hard fail on >5% rejection rate.

### Layer 2 — Domain rules

Beyond shape, the values must make sense for the field. A fitment of `{ year: 2030, make: "Kayo", model_code: "AT125-B" }` passes the schema but is wrong if Kayo didn't make the AT125-B model in 2030.

Domain rules in code:

```typescript
// src/validators/fitment.ts
const KNOWN_MAKE_PREFIXES: Record<string, string[]> = {
  Kayo: ['AT', 'AY', 'AZ', 'KY'],
  // ...
}

function validateModelCodeMatchesMake(make: string, modelCode: string): Result<void> {
  const prefixes = KNOWN_MAKE_PREFIXES[make]
  if (!prefixes) return ok() // unknown make, skip
  const ok = prefixes.some(p => modelCode.startsWith(p))
  return ok ? ok() : err(`Model code ${modelCode} doesn't match ${make} prefix set`)
}
```

These rules are **OEM-specific** and live in a configurable rule set, not hard-coded. At Ashley I built this as a registry-driven rule engine — each rule has an owner, a severity (CRITICAL halts the pipeline, WARNING logs), and a coverage report.

### Layer 3 — Cross-row consistency

Same Chinese string should produce the same English. If it doesn't, one of the translations is wrong (the LLM call was non-deterministic, or the dealer's English varies by part).

```sql
-- runs after ingestion, before audit-mode finishes
SELECT
  name_cn,
  array_agg(DISTINCT name_en) as distinct_translations,
  count(*) as row_count
FROM products
GROUP BY name_cn
HAVING count(DISTINCT name_en) > 1
ORDER BY row_count DESC;
```

Each disagreement is a candidate defect. The audit table records the disagreement; the catalog API can either:
- Surface the dealer-supplied translation (current default)
- Surface the most common translation across rows
- Surface a flagged disagreement to marketplace listing review

This is the same multi-checkpoint DQ pattern from Ecentric — domain-parallel execution with critical-halt on multi-row inconsistency. Adapted for InventoryFlow's smaller scale.

### Layer 4 — Cross-source agreement (the LLM audit)

Covered in [`06-llm-strategy.md`](./06-llm-strategy.md); summarised:

The LLM is asked to translate `name_cn` independently. The output is compared with `name_en` (dealer-supplied). The result is `agree | partial | disagree`. The 11/68 disagreement findings on the reference data revealed real dealer defects (typos like "busher", terminology mismatches, category errors).

**The LLM is a defect detector, not the translator of record.** This is the policy decision that matters. The dealer's translation goes to `name_en`; the LLM's goes to `name_en_llm` with a `data_quality` score; disagreements get an `audit_status` flag.

### Layer 5 — Downstream feedback loop (designed, deferred)

When a marketplace listing fails ("Amazon rejected the listing because the part category is wrong"), the failure should:

1. Locate the source row in the catalog via the marketplace SKU mapping
2. Mark the `data_quality` score as degraded
3. Invalidate the LLM cache entry for that translation
4. Schedule a re-translation with a more capable model
5. Notify a human reviewer if the second attempt also disagrees

This loop is designed; not yet implemented because the marketplace integration is out of scope for the take-home. Documented here so the design intent is visible.

---

## Drift detection — the silent failure mode

The most insidious failure mode in OEM data is **schema drift without notification**. The dealer changes their xlsx column header from "Description" to "Part Description" tomorrow. The naive parser either:
- (a) Silently skips the column (returning empty descriptions for the new run)
- (b) Crashes with an unhelpful error
- (c) Mis-maps the column to something else

Solution A's section detector uses **header-signature regex matching** specifically to make this failure explicit:

```typescript
// src/parse/section-detect.ts (abbreviated)
const SIGNATURES = [
  { name: 'chassis-v1', headers: ['part_number', 'name_en', 'name_cn', 'year', ...] },
  { name: 'engine-v1',  headers: ['part_no', 'name_en', 'name_cn', 'spec', ...] },
  { name: 'u8-v1',      headers: ['part_id', 'description_en', 'description_cn', ...] },
]

function detectSection(headerRow: string[]): Signature | null {
  for (const sig of SIGNATURES) {
    if (matchesSignature(headerRow, sig.headers)) return sig
  }
  return null // explicit no-match
}
```

If the headers don't match any known signature, the parser fails the run with the specific signature mismatch reported. **No silent partial ingestion.** Adding a new signature is a code change with a code review.

This is the most important single design decision in the parser. I have seen the silent-mis-mapping failure mode hit production at Ecentric (a source DB column was renamed; mirroring continued silently with NULL values for two weeks) — the cost was a downstream report showing $0 revenue for a product line. Source-target reconciliation, source-vs-target drift detection: now non-negotiable in everything I build.

---

## Golden sample testing

Beyond unit tests, the pipeline has a **golden output set** committed in [`sample-output/`](https://github.com/ankinguyen-engineer-2002/inventoryflow-catalog-ingest/tree/main/sample-output) of the impl repo:

- `products.csv` — 3,938 rows expected
- `product_images.csv` — 10,524 associations expected
- `vehicle_models.csv` — 35 distinct models expected
- `queries/01-fitment-query.md` — expected latency
- `queries/02-llm-audit-disagreements.md` — expected 11 disagreements out of 68 sampled

After any code change, the run is compared against these expectations. Significant deviations fail CI before review.

This pattern is borrowed from the data-quality discipline I established at ADP using OpenMetadata + dbt tests — naming conventions, source-of-truth documentation, expected-value contracts. The InventoryFlow version is the same idea at smaller scale.

---

## Drift over time — the year-2 conversation

The catalog will change. Parts get superseded. OEMs adjust naming conventions. Categories evolve. Without explicit tracking:

- Last quarter's `name_en` becomes inconsistent with this quarter's
- A part with `model_code: AT125-B` quietly stops fitting a `2024 Kayo Predator` because the OEM removed that fitment

Solution A's design accommodates drift via:

1. **`source_file_sha256` on every row** — every product traces back to the specific source file
2. **`ingest_runs` registry** — every run has a UUID, started/completed timestamps, file hash
3. **`updated_at` columns** — last-modified timestamp per row
4. **Soft-delete via `effective_until`** — superseded rows are marked, not deleted

What's *not* in Solution A but is in Solution B's design:
- **Iceberg `VERSION AS OF`** — point-in-time query without replay
- **Column-level lineage** — `name_en` for product X derives from cell row 200 of file Y at timestamp Z

When the year-2 conversation about supersession, recall handling, and historical reporting becomes load-bearing, Solution B's time-travel becomes the answer. Until then, Solution A's audit + soft-delete is sufficient.

---

## Human-in-loop escalation paths

Defect detection is only half the story. The other half is what happens with the detected defects.

### Path 1 — Auto-correct on high confidence

If the LLM is `high` confidence and disagrees with the dealer's English, and the cross-row consistency check agrees with the LLM, surface the LLM's translation by default and mark the audit row as `auto_corrected`.

Conservative policy: this is *off* in Solution A as shipped. The dealer's translation goes to `name_en`; the LLM's goes to `name_en_llm`. Enabling auto-correct is a config flag.

### Path 2 — Marketplace-grade review

For rows that will be surfaced to a marketplace listing, the data quality bar is higher. Any disagreement on a marketplace-bound row is escalated to a human reviewer queue.

The queue interface is a simple Fastify route showing the original dealer English, the LLM English, the Chinese, the schematic image, and a confidence score. Reviewer picks one. The choice updates the LLM cache so future runs get the corrected answer.

This is not built in Solution A as shipped. The shape is designed — three tables (`review_queue`, `review_decisions`, `cache_overrides`) with predictable migration paths.

### Path 3 — Cohort-level review

When a class of disagreements appears (e.g., all "carburetor jet" parts have category mismatches), the right answer is not 100 individual reviews — it's a single cohort fix that updates the LLM prompt, the validator rules, or both.

The pattern at Ashley for this kind of issue: a registry health check that surfaces cohort-level anomalies, a Streamlit-based Lineage Explorer to investigate, and a centralised fix that propagates.

For InventoryFlow, the equivalent would be a `pnpm audit --cohort name_cn` command that shows aggregate disagreement patterns. Designed; not implemented.

---

## CI/CD verification gates

Output verification continues at the CI level (more in [`08-operations.md`](./08-operations.md)):

| Gate | When | What it does | Failure action |
|---|---|---|---|
| **Unit tests** | Every PR | 32 tests across parser + cache + env | Block PR |
| **Idempotency** | Every PR | Re-run ingest; row counts must match | Block PR |
| **Benchmark regression** | Every PR | Fitment query > 50ms p95 = regression | Block PR |
| **Golden sample diff** | Every PR | Output matches committed `sample-output/` | Warn + manual review |
| **LLM audit findings** | Nightly | 16% disagreement rate baseline; > 25% = alert | PagerDuty |
| **Cache health** | Hourly | Cache hit rate > 95%; staleness > 7 days = warn | Slack alert |

Most submissions implement the first two and stop. The point of the rest is to **catch the slow-drift failure modes that don't fail any single test but degrade the system over weeks**.

---

## The single most important sentence in this doc

**Wrong data at scale is more expensive than wrong infrastructure.** Solution A is calibrated to that belief — the audit table, the section-detector explicit failure mode, the LLM provider abstraction with cache discipline, the golden sample testing — these are not boilerplate. They are the things that, in my four years of building data platforms, made the difference between systems that aged well and systems that became technical debt.

The submission spends architectural effort on verification because that's where production data systems either thrive or quietly die.

---

**Next:** [08-operations.md](./08-operations.md) — CI/CD, security, observability, scale, governance.
