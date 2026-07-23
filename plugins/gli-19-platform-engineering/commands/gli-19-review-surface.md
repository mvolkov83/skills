---
description: Assess a service against the GLI-19 code-shaped requirements
argument-hint: "<path to the service or module to assess>"
allowed-tools: Read, Grep, Glob, Bash(git log:*), Bash(git ls-files:*), Bash(find:*)
---

Assess the code at `$1` against the GLI-19 requirements whose evidence is code, infrastructure or
configuration, and produce an engineering gap list. This is a **surface** assessment — it looks for
what is missing, which a diff review structurally cannot do. For reviewing a specific change use
`/gli-19-review-diff`.

The requirement set is at
`${CLAUDE_PLUGIN_ROOT}/skills/gli-19-platform-engineering/references/requirements.md` — 96
requirements grouped by chapter and section. Work through it; the rules in the
`gli-19-platform-engineering` skill are the engineering restatement of the same set.

If `$1` is empty, ask which service or module to assess. Do not assess a whole monorepo in one
pass — the ownership boundary is what makes the result meaningful.

## 1. Scope the ownership boundary first — this step is not optional

Before assessing anything, establish what this codebase **owns** versus what it **delegates**.
Read the service's entry points, its interface definitions (proto/OpenAPI/schema), its
dependencies and its configuration, and produce:

| Concern | Owned here / delegated / not offered | Evidence |
|---|---|---|

Delegation is a structural fact, not an opinion — if the service's interface carries no
game-outcome field, game design and RNG are delegated to whoever produces the outcome; if identity
verification happens behind an external provider call, the verification mechanics are theirs and
the gating is yours. Record which side of each boundary this code sits on.

Then present the boundary to the user and **ask them to confirm or correct it before continuing**.
A wrong boundary makes every downstream verdict wrong, and scope rulings are the operator's call,
not yours.

## 2. Assess by domain cluster

Work cluster by cluster rather than requirement by requirement — related requirements share
evidence, and a cluster pass reads the relevant code once. The clusters mirror the skill's
sections: server authority · time · records · record integrity · wallet semantics · account
lifecycle · control surfaces · reporting · location · program integrity · communications ·
availability.

For a large surface, run the clusters as parallel subagents, one cluster each, and merge the
results. Tell the user when you do this and what each agent covered.

**Verdicts come from primary artifacts only**: source code, database schema and migrations,
interface definitions, configuration, infrastructure manifests, and tests. Do **not** accept a
README, design doc, changelog, code comment, or a prior audit as evidence that a mechanism exists
— find the mechanism. A claim in prose with no implementation behind it is a `non-compliant`, not a
`compliant`, and saying so is the entire value of this pass.

## 3. Record a verdict per requirement

| Status | Means | Required fields |
|---|---|---|
| `compliant` | the mechanism exists and satisfies the testable criteria | evidence as `file:line` |
| `partial` | built, with a specific identified gap | evidence + the gap in one sentence |
| `non-compliant` | not built | what is absent |
| `not-applicable` | outside the confirmed ownership boundary | which boundary and why |
| `needs-info` | the owner is another service or the evidence is not in this codebase | where to look |

Never mark `compliant` without a file reference. Never mark `not-applicable` on your own judgment —
only against a boundary the user confirmed in step 1. Aggregate a delegated family into one
`not-applicable` record rather than enumerating dozens of them.

## 4. Output

**Counts by status**, as a table. Do **not** compute or print a readiness percentage — the
denominator depends on scoping rulings and jurisdiction, a percentage from this command would be
read as a certification metric, and it is not one.

**Gap list**, ordered as a build order rather than by requirement number: put the items that
others depend on first (an actor field and an integrity column before the reports that read them;
a records model before the aggregates over it). For each: requirement id, what is missing, the
concrete fix in this codebase, and an effort size (S / M / L).

**Excluded**, with reasons: everything marked `not-applicable` or `needs-info`, grouped, one line
each.

**Out of scope of this command** — state plainly that this covers only the requirements whose
evidence is code, infrastructure or configuration; that the operator's procedural and governance
obligations, the submission package and the laboratory's own testing are not assessed; and that
roughly a fifth of the technical-security appendix lives in infrastructure and runbooks rather
than in application code.

## Constraints

- This is not a certification assessment and must never be presented as one. A testing laboratory
  assesses against the standard using evidence the operator supplies; this command reads code.
- Cite requirement ids exactly as they appear in the reference. Do not invent ids, and do not
  assess requirements that are not in the reference file.
- Report honestly. A service with two compliant records out of sixteen applicable is a useful
  result; a padded one is worse than none, because it will be believed.
- Thresholds in the requirements are reference values the jurisdiction overrides. A configurable
  threshold set to a different value is not a gap; a hard-coded one is.
- For money representation, idempotency and ledger structure, defer to the
  `money-and-payments-best-practices` skill.
- If the project keeps its own compliance records (a requirement catalog, scope rulings, prior
  assessment runs), conform to their schema and status vocabulary instead of this command's, and
  say that you did.
