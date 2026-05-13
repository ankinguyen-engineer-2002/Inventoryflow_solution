# Problem Framing — what the brief actually asks

> A senior architect's first job on any brief is to *re-read it adversarially*. The brief is not the problem; the brief is one team's compressed description of the problem. Half the engineering value lives in the gap between the two.

---

## What the brief literally says

From the Talemy × InventoryFlow Senior Engineer test PDF (08 May 2026):

> *"At InventoryFlow, we receive parts catalog data from hundreds of different distributors and OEMs. This data is extremely messy, often arriving in unstructured formats like PDFs containing a mix of schematic images, part numbers, and multilingual text. Your task is to standardize it all, give us a clean database that contains the schematic image uploaded into an R2 bucket, along with the part numbers, the English name, the Chinese name, and everything clean and organized. There should also be a JSON column that outlines every year, make, and model that the part fits."*

The explicit deliverables, lifted out:

1. **Standardize messy multi-format input** → clean database
2. **Upload schematic images** → Cloudflare R2
3. **Part numbers, English name, Chinese name** → tabular columns
4. **JSON column with year/make/model fitment** → JSONB-shaped
5. **Use AI tooling** to parse the messy content

The success criteria, also explicit:

- **Pragmatism & Speed** — not "enterprise-heavy boilerplate"
- **AI tooling** — Cursor / Windsurf / Copilot for coding, OpenAI / Claude for vision
- **Clean Architecture** — especially the JSONB fitment column shape

---

## What the brief is *actually* asking

A senior reader notices the gaps and the silences:

### Gap 1: PDF mentioned, xlsx delivered

The brief talks about PDFs. The actual sample file is `Copy of Example Data for Engineer.xlsx` (241 MB, 110 sheets, 1,586 embedded schematic images, EN + CN multilingual, Kayo ATV catalog).

This is informative. The brief was probably drafted before someone realized the test data is xlsx. The implicit instruction is: *handle whatever messy format shows up*. A literal "PDF parser" implementation would be the wrong answer; a parser keyed to xlsx is also the wrong answer if it can't be extended. The correct shape is a **pluggable ingestion pattern**, with xlsx as the first concrete handler. This is part of why Solution A seeds the `ingestion_patterns` and `dealer_pattern_bindings` tables even though the runtime dispatcher is deferred.

### Gap 2: "Hundreds of distributors and OEMs"

This phrase appears once and is easy to miss. It is the single most important architectural constraint in the brief.

A naive reading: "build one parser for this one xlsx". A senior reading: "build a system that onboards a new dealer in <1 day without code changes to the core". The first reading produces a 200-line script; the second produces metadata-driven dispatch, schema-on-read, per-dealer config, and the dealer / ingestion-pattern / binding registry tables.

At Ashley Furniture (current role) I work on a metadata-driven control plane covering 5,000+ enterprise tables. Onboarding a new dataset is a *registry insert*; zero per-asset code. That pattern is the shape of "hundreds of dealerships" — it is the only shape that doesn't collapse under its own technical debt by dealer 20.

### Gap 3: "Especially the JSONB fitment column"

This line is the senior-engineering tell. The brief is explicitly asking about *catalog architecture*, not *data parsing*. The interesting choice is:

- **Normalised** (joined `product_fitments` table) — relational integrity, harder marketplace API
- **Denormalised JSONB on `products`** — exact shape requested, GIN index for fast containment, easier marketplace API

The brief is testing whether the candidate knows that the dominant query pattern is "find parts fitting vehicle X" and that this query against a denormalised JSONB column with `jsonb_path_ops` runs sub-millisecond. A normalised schema would require a join on every catalog API call.

This is one of those small-on-paper, large-in-practice choices that defines whether the system feels fast at scale.

### Gap 4: "AI tooling encouraged" — but in which role?

The brief lists Cursor, Windsurf, Copilot as IDE assistants, and OpenAI / Claude as Vision LLMs. It is silent on:

- Cost discipline (no mention of API budget)
- Reviewer execution (will the reviewer pay to run the demo?)
- Production swap (what happens when the API key in the demo runs out?)
- Audit and rollback (what happens when the LLM is wrong?)

These silences are the actual AI-tooling test. A naive implementation calls Claude or GPT-4 on every row in the demo, costs the reviewer money, and has no story for "what if the model is wrong". A senior implementation has:

1. A provider abstraction with multiple backends (paid API, self-host, cache-only)
2. A committed cache so reviewers pay zero
3. An audit table that records every call, cost, and disagreement with the dealer's translation
4. An explicit deferral story (offline batch with model X, online cache with the result, escalation path on disagreement)

The whole of [`06-llm-strategy.md`](./06-llm-strategy.md) exists to address what the brief leaves unsaid.

### Gap 5: "Not enterprise-heavy boilerplates"

This is the warning, but it's also a permission. The senior reader notices that the brief specifically calls out *boilerplate* — the implication being that the reviewer has seen submissions overdosed on cargo-culted enterprise patterns and is filtering them out.

What's not boilerplate (per my reading):

- ADRs explaining specific decisions — signal of judgment
- Audit tables explaining LLM cost / latency / disagreement — signal of production discipline
- A migration plan from A to B with quantified triggers — signal of long-horizon thinking
- Cost economics for 1,000 dealers — signal of business awareness

What *is* boilerplate (deliberately omitted from Solution A as shipped):

- Hexagonal architecture for a 3,000-line codebase
- Onion / clean architecture layering for a 12-table schema
- Domain Driven Design ubiquitous-language docs for a problem domain that is, in fact, just a parts catalog
- A microservices decomposition for a system that fits in one Postgres database

These are good ideas at the wrong scale. Including them would be the over-engineering the brief warns against.

---

## The hidden constraints

Beyond the brief, a few constraints are inferable from context:

| Constraint | Inferred from | Architectural implication |
|---|---|---|
| **Early-stage hiring** | Senior Engineer role on Talemy (Vietnam recruiter); $3.5–5k USD NET salary range | The company can't afford the data-engineering hiring needed for Solution B today |
| **Pilot scale** | "Hundreds of dealerships" is aspirational, not current | Solution at year 1 scale, not at year 3 — but with year-3 paths drawn |
| **TS / Node hiring pool** | JD lists TypeScript / Node / Postgres / Redis / Docker explicitly | Solution must be maintainable by the TS engineers they'll hire |
| **Test data is one OEM** | Single Kayo xlsx in sample | Multi-OEM is the *next* concern, not the current one |
| **Reviewer will run the demo** | Implied by "AI tooling" mention | Reviewer should not need an API key |
| **Reviewer time is finite** | Implied by "Pragmatism & Speed" | Three-command run, sample output committed, README that respects the reader's hour |

The submission is calibrated to these constraints. Solution A is the answer; Solutions B, C, D are the *next* answers, drawn at honest detail so the year-2 conversation isn't a panic.

---

## What "good" looks like for this brief

Restating the bar:

| Dimension | Bar |
|---|---|
| **Functional correctness** | xlsx in → clean DB out → R2 has images → fitment is queryable JSON |
| **JSONB design** | Fitment query is sub-50ms with proper index; structure is marketplace-ready |
| **AI tooling** | Provider-pluggable; reviewer runs free; audit recorded; disagreement detected |
| **Pragmatism** | Demo runs in 3 commands; setup is one `docker-compose up`; under 90 seconds wall-time |
| **Architecture clarity** | Plane separation visible; ADRs explain non-obvious choices; migration triggers quantified |
| **Senior signal** | Four solutions presented; trade-offs argued; *what comes next* documented; corners cut explicitly listed |

What is *not* the bar (despite being tempting to over-deliver on):

- A 50-page architecture treatise (this is, deliberately, a 9-doc set, not 50)
- A custom UI for the catalog
- A full marketplace integration
- A migration tool to a future stack
- Operational maturity beyond what's needed at sub-100 dealers

---

## Restating the problem in one sentence

**Build a TypeScript pipeline that ingests one messy xlsx today and is on a credible path to ingesting hundreds of varied catalog formats from hundreds of dealers in two years, with AI tooling used as a measured subsystem (not a black box), JSONB fitment indexed for marketplace queries, and a documented alternative architecture for the day the current one breaks.**

Solution A in this submission implements that sentence. Solutions B, C, D draft the alternatives for the day the second clause becomes load-bearing. The rest of this repo is the proof that the framing above isn't retrofitted onto a code dump — it shaped what got built and what got skipped.

---

**Next:** [02-solution-A-recommended.md](./02-solution-A-recommended.md) — the chosen path, overview to detail.
