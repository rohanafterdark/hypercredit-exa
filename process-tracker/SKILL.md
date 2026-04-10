---
name: process-tracker
description: >
  Write a Process Tracker (PT) in a Haskell-Nix-Cabal codebase. Use this skill
  whenever the user asks to create, write, implement, or scaffold a PT, process
  tracker, backfill job, or data pipeline job in a Haskell codebase. Also trigger
  when the user mentions loanAnalyticsPT, merchantDataAggregationPT, ClickHouse,
  Kafka push, backfill PT, update field PT, or mail+CSV PT. Always use this skill
  before writing any PT code — even for "simple" PTs — as there are many
  structural and naming conventions that must be followed.
---

# Process Tracker (PT) Skill

A step-by-step workflow for writing a Process Tracker in a Haskell-Nix-Cabal codebase. Follow every step in order. Ask one question at a time. Do not proceed to the next step until the current one is resolved.

---

## Step 0 — Determine PT Type

Ask the user:

> "What type of PT is this and name of the PT?"
> 1. **Query & Update PT** — pulls data from Postgres and backfills/updates via Kafka → ClickHouse
> 2. **Mail + CSV PT** — fetches data from Postgres/ClickHouse/S3 and sends it via email as a CSV attachment"

Proceed to the relevant workflow below.

---

## Workflow A: Query & Update PT

### A1 — Find Base Template

Search the codebase for `loanAnalyticsPT` automatically using available tools (grep, file search, LSP, etc.). Do not ask the user for the path first.

- If found: load it and use it as the structural template.
- If not found: ask the user to provide a base template file or path.

### A2 — Backfill vs Update Field

Ask:

> "Is this PT a **backfill** (populating missing data) or an **update** (modifying an existing field)?"

If **update field**:
- Ask: "What is the exact field name to update?" (must be the exact Haskell/DB column name)
- Ask: "Which table does this field belong to?"

### A3 — Source Query

Ask:

> "Do you already have the Postgres query (or queries) for this PT, or should we figure it out from the codebase?"

- If the user **provides a query**: use it directly.
- If the user **wants us to figure it out**:
  1. Search for similar existing PTs in the codebase to infer the query pattern.
  2. If pattern is clear, propose a query and confirm with the user.
  3. If still unclear, proceed to A3a to get schema info and build from scratch.
  4. Once a query approach is decided, if writing Haskell Beam queries: check if the `haskell-beam-postgres` skill is available and invoke it. If not available, write queries following standard beam-postgres patterns.

#### A3a — Table Schema

Ask:

> "Can you provide the schema for the tables we need to query? If not, I'll look for it."

Resolution order:
1. User provides schema directly → use it.
2. Search the repo for a project named `euler-credit-db` and extract schemas from it. READ ACCESS ONLY — do not modify any files.
3. If not found, infer schema from other PT implementations in the codebase.

### A4 — Destination Table

Ask:

> "Which ClickHouse table should this PT push data to?"

Record the exact table name — do not infer.

### A5 — Implement the PT

Using the base template from A1 as structural reference:

1. Scaffold the PT module following the same file/module naming convention.
2. Implement the Postgres fetch logic using the query from A3.
3. Implement the Kafka producer step to push to ClickHouse (table from A4).
4. Wire up the PT entry point (main runner, scheduling config, etc.) mirroring the template.
5. Add any field-update logic if this is an update PT (from A2).

### A6 — Build

Run:

```bash
cabal build all
```

- If the build **succeeds**: report success and summarise what was built.
- If the build **fails**:
  1. Surface the full error output clearly to the user.
  2. Attempt to fix each error (type errors, missing imports, module resolution, etc.).
  3. Re-run `cabal build all` after fixes.
  4. Repeat until green or until a fix requires user input (e.g. missing dependency in `.cabal` file).

---

## Workflow B: Mail + CSV PT

### B1 — Find Base Template

The default template for Mail + CSV PTs is `merchantDataAggregationPT`. Search for it automatically in the codebase using available tools.

- If found: load and use as structural template.
- If not found: ask the user for an alternative template.

### B2 — Data Source

Ask:

> "Where should this PT fetch data from?
> 1. Postgres
> 2. ClickHouse
> 3. S3
> 4. Multiple sources"

Record the answer. If multiple sources, clarify each one.

### B3 — Source Query / Fetch Logic

Ask:

> "Do you already have the query or fetch logic, or should we derive it from the codebase?"

- If the user **provides it**: use directly.
- If not:
  1. Search similar PTs in the codebase for patterns.
  2. For Postgres queries using Beam: invoke `haskell-beam-postgres` skill if available.
  3. For ClickHouse or S3: follow patterns from `merchantDataAggregationPT`.

Apply schema resolution order from A3a if table schemas are needed.

### B4 — Email Configuration

Ask the following (can be batched as a short list since they're all related):

> "A few quick questions about the email:
> - Who are the recipients? (email addresses or distribution list)
> - What should the subject line be?
> - What should the email body say? (or should we copy it from `merchantDataAggregationPT`?)"

If the user is unsure about any of these, fall back to the equivalent fields in `merchantDataAggregationPT`.

### B5 — CSV Shape

Ask:

> "What columns should the CSV have, and in what order? Or should we derive this from the query/fetch results?"

### B6 — Implement the PT

Using `merchantDataAggregationPT` as structural reference:

1. Scaffold the module with correct naming conventions.
2. Implement data fetch logic (B2/B3).
3. Serialise results to CSV with the column shape from B5.
4. Wire up the mailer with recipients, subject, and body from B4.
5. Add scheduling/runner entry point mirroring the template.

### B7 — Build

Same as A6 — run `cabal build all`, surface errors, attempt fixes, repeat.

---

## General Guidelines

- **One question at a time.** Do not batch unrelated questions.
- **Always verify exact names.** Field names, table names, module names, and ClickHouse table names must be confirmed with the user or taken verbatim from the codebase — never guessed.
- **Mirror the template.** Module structure, import style, runner wiring, and config patterns must match the base template found in the codebase.
- **Beam queries.** When writing Haskell Postgres queries, always use the `haskell-beam-postgres` skill if it is available in the current skill set. If it is not available, follow standard beam-postgres patterns (avoid raw SQL unless the codebase already does so).
- **Build is mandatory.** Never consider a PT complete until `cabal build all` passes.
- **Nix.** If the project uses a Nix shell, remind the user to run inside `nix-shell` or `nix develop` before building if the build environment is not already active.
- **Do not blindly import** — only import modules that are actually used in the PT implementation. DO NOT COPY PASTE IMPORTS FROM TEMPLATE MODULES. Remove any unused imports to keep the code clean.
- **READ ACCESS ONLY to `euler-credit-db`** — you may search and read files to extract schema information, but do not modify any files in that project under any circumstances.