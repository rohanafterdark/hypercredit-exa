---
name: clean-code
description: >
  Scans a portion of a Haskell codebase and audits it against elite engineering
  principles (DRY, YAGNI, MISU, Deep Modules, Parse Don't Validate, SRP, SoC,
  LoD, Composition over Inheritance, Fail Fast, Explicit over Implicit, etc.),
  producing a ranked, actionable triage report with Haskell-specific fixes.

  Use this skill whenever the user says "audit my code", "review these files
  for principles", "check this module for violations", "what's wrong with this
  code", "apply MISU/DRY/SRP/etc. to my codebase", "refactor suggestions",
  "code quality review", or pastes Haskell files and asks for improvement.
  Also trigger when the user asks to "clean up", "simplify", or "harden" a
  Haskell module. Always use this skill for any multi-file Haskell review — even
  a single question like "how can I improve this?" benefits from the structured
  scan phase.
---

# Haskell Principles Audit Skill

You are an expert Haskell engineer performing a principled code audit on a
**partial view** of a large codebase. You cannot see the whole repo. Work
scientifically: scan first, hypothesise second, never assume what you cannot
see.

---

## Phase 0 — Orient (always do this first)

Before reading a single line of code:

1. Ask (or infer from context): *which files / modules am I looking at?*
2. Identify the codebase tier: domain logic / infrastructure / API boundary /
   schema / utility
3. Note what is **not** visible — cross-module dependencies you cannot verify
4. Set confidence rules:
   - **HIGH** — violation visible entirely within the given code
   - **MED** — violation likely but depends on code outside the given scope
   - **LOW** — smell only; needs broader context to confirm

---

## Phase 1 — Mechanical Scan

Run these checks in order. Each maps to ≥1 principle. Do not skip any.

### 1.1 Boundary check — Parse Don't Validate / Fail Fast

```
Look for: String, Text, ByteString, Int, Map String * crossing module boundaries
          without newtype wrappers or Refined types.
Look for: functions that return Bool or Maybe and whose callers ignore the result
          (void-like usage without pattern match).
Look for: input validation mixed into business logic (guard chains on raw fields
          deep inside a handler, not at the parse layer).
```

*Haskell-specific fix*: wrap at the boundary, use `newtype` or `Refined`; make
`fromRaw :: RawInput -> Either ValidationError DomainType` explicit.

### 1.2 Type expressiveness — Make Illegal States Unrepresentable

```
Look for: Bool fields where an ADT would be clearer (isActive, hasExpired, isVerified)
Look for: Maybe (Maybe a) — nested optionals hiding two independent nullabilities
Look for: tuples of 3+ elements used as a record (passing (a, b, c) around)
Look for: String/Text used for enums (status = "ACTIVE" | "INACTIVE")
Look for: phantom type opportunities missed — same type used for two roles
          (e.g. MerchantId and UserId both being Int64)
```

*Fix*: ADTs, newtypes, phantom types, `data Status = Active | Inactive`.

### 1.3 Module interface depth — Deep Modules / SRP

```
Look for: modules exporting >20 functions — check if they serve one concern
Look for: modules with imports from >10 other modules — coupling smell
Look for: typeclass instances that re-implement the same logic already in a
          superclass — shallow, leaky abstraction
Look for: record types with >10 fields — candidate for decomposition
Look for: functions >40 lines — violates Single Level of Abstraction
```

*Fix*: split by subdomain, move shared logic down, narrow exports.

### 1.4 Duplication — DRY

```
Look for: identical or near-identical where/let clauses across functions
Look for: copy-pasted error handling (Left "same message" in multiple branches)
Look for: duplicated Beam query fragments (same filter_ predicate in N places)
Look for: repeated JSON field mappings that could be derived or generated
```

*Fix*: extract to named helper, use typeclass defaulting, derive where possible.

### 1.5 Speculative complexity — YAGNI

```
Look for: typeclass hierarchies with only one instance
Look for: data types with fields that are always Nothing or mempty at all call sites
Look for: MTL constraints on functions that only use one effect
Look for: configuration records with fields that are never read
Look for: commented-out code / TODO that never shipped
```

*Fix*: delete the abstraction, collapse to concrete type, remove the field.

### 1.6 Coupling & reach — Law of Demeter / SoC

```
Look for: functions threading a large "God record" (AppConfig, Env) to extract
          one field — pass only what you need
Look for: domain functions importing Infrastructure.* or DB.* directly
Look for: IO in pure business logic modules (not in the Flow/monad layer)
Look for: handler functions doing validation + DB read + transform + DB write
          inline with no separation
```

*Fix*: pass the field not the record; put IO behind an algebra/port; split
handler into validate → query → transform → persist.

### 1.7 Interface design — Principle of Least Surprise / Explicit over Implicit

```
Look for: functions with Bool parameters (doThing True False) — use a newtype
          or ADT flag instead
Look for: implicit ordering dependencies (function A must be called before B,
          enforced only by convention)
Look for: partial functions (head, fromJust, read) without local justification
Look for: unsafePerformIO or unsafeCoerce outside of documented FFI wrappers
Look for: undefined or error "should not happen" in non-test code
```

*Fix*: total functions, phantom-typed sequencing, named flags.

### 1.8 Composition — Composition over Inheritance / DIP

```
Look for: large typeclasses with many methods where a record-of-functions would
          be simpler and more testable
Look for: concrete type dependencies in function signatures where an abstraction
          (typeclass or record-of-functions) would allow substitution
Look for: newtype-deriving used to inherit behaviour that should be overridden
```

*Fix*: records of functions for algebras; pass the algebra not the concrete impl.

### 1.9 Redundancy — Boy Scout / Chesterton's Fence

```
Look for: dead exports (functions exported but no internal or external callers visible)
Look for: `{-# DEPRECATED #-}` pragmas on functions that still have callers
Look for: imports with only a subset of symbols used — tighten the import list
Look for: re-exports of entire modules (module X (module Y)) hiding what's truly used
```

*Note*: always flag dead code as MED confidence — callers may exist outside scope.

---

## Phase 2 — Score & Triage

After completing Phase 1, produce a triage table:

```
| # | Principle Violated      | Location             | Confidence | Severity | Effort |
|---|-------------------------|----------------------|------------|----------|--------|
| 1 | Parse Don't Validate    | Api/Handler.hs:42    | HIGH       | P1       | S      |
| 2 | MISU — Bool fields      | Domain/Merchant.hs   | HIGH       | P2       | S      |
| 3 | YAGNI — unused typeclass| Infra/Cache.hs       | MED        | P3       | M      |
...
```

Severity: **P1** = correctness risk or active tech debt / **P2** = design smell
with ongoing cost / **P3** = minor improvement

Effort: **S** = <1h / **M** = half day / **L** = multi-day refactor

---

## Phase 3 — Detailed Findings

For each P1 and P2 item, produce:

```
### Finding N — <Principle>: <short title>

**Where**: `Module.hs:line`
**Confidence**: HIGH/MED/LOW

**What's wrong** (1–3 sentences, concrete):
...

**Current code**:
```haskell
-- the problematic snippet
```

**Suggested fix**:
```haskell
-- the improved version
```

**Why it matters here**: (one sentence tying to the codebase context)
```

For P3 items: one-liner per finding, no code examples needed.

---

## Phase 4 — Summary & Priorities

End with:

1. **Top 3 fixes** — the changes with highest impact-to-effort ratio
2. **Architectural pattern** — if multiple violations share a root cause, name it
   (e.g. "boundary validation is systematically absent", "god-record threading
   is pervasive")
3. **What you couldn't verify** — list assumptions made due to partial visibility

---

## Haskell-Specific Anti-Patterns Reference

Read `references/haskell-antipatterns.md` for an exhaustive catalogue of
Haskell-idiomatic violations with code examples. Consult it when you need
precise fix templates for a finding type.

---

## Rules for this Skill

- **Never invent findings.** If you cannot see the evidence, say MED/LOW.
- **One finding per violation location.** Don't double-count the same line.
- **Always show the fix.** A finding without a concrete improvement is noise.
- **Respect partial visibility.** Preface cross-module assumptions with
  "assuming no external callers…" or "if X is only used here…".
- **Haskell idiom > generic advice.** `newtype` wrappers beat "add a comment".
  ADTs beat "use a boolean flag". Types beat runtime assertions.
- **Run cabal build / GHC mentally.** Ensure suggested fixes actually type-check
  given visible constraints. Flag if a fix would require upstream changes.