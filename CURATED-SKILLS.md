# Curated Skills

A personal, curated list of Claude Code skills I actually use and why. Each entry has exact source and install instructions so the setup is reproducible on a fresh machine.

Entries are ordered by how often I reach for them.

---

## 1. `claude-md-management` — keeps `CLAUDE.md` honest

Ships **two** skills, both excellent — they cover the two halves of the same problem (drift detection + closing-the-loop):

- **`claude-md-improver`** *(audit)* — scans every `CLAUDE.md` in the repo, evaluates quality against canonical templates, outputs a quality report, then makes targeted improvements. Triggers on phrases like *"audit CLAUDE.md"*, *"check CLAUDE.md files"*, *"project memory optimization"*.
- **`revise-claude-md`** *(update)* — appends learnings from the current session into `CLAUDE.md`. Triggers when wrapping up a session that surfaced a non-obvious convention or workflow worth persisting.

**Why I use it.** `CLAUDE.md` is the only context that loads on *every* conversation, so it has outsized leverage — and outsized blast radius when it rots. Outdated paths, stale conventions, and forgotten decisions silently degrade every future session. The audit skill catches drift before it compounds; the update skill closes the feedback loop after each useful session. Run audit periodically (monthly or after major refactors); run update at the end of any session where Claude learned something the next instance shouldn't have to re-discover.

**Source.** [`anthropics/claude-plugins-official`](https://github.com/anthropics/claude-plugins-official) — Anthropic's official plugin marketplace.

**Install.**

```text
/plugin marketplace add anthropics/claude-plugins-official
/plugin install claude-md-management@claude-plugins-official
/reload-plugins
```

**Invoke.**

```text
/claude-md-management:claude-md-improver   # audit + targeted improvements
/claude-md-management:revise-claude-md     # append session learnings
```

---

## 2. `engineering:code-review` — structured 4-dimension review

A skill from the `engineering` plugin in Anthropic's `knowledge-work-plugins` marketplace. I invoke it **manually** when I want a focused, structured review of a diff or PR — not a multi-agent vote, just a disciplined single-pass review with a defined rubric.

**What it does.** Reviews the provided diff / PR / file along **four explicit dimensions**:

- **Security** — SQL injection / XSS / CSRF, auth & authz flaws, secrets in code, insecure deserialization, path traversal, SSRF.
- **Performance** — N+1 queries, unbounded loops, missing indexes, algorithmic complexity in hot paths, resource leaks.
- **Correctness** — edge cases (empty / null / overflow), race conditions, error handling & propagation, off-by-one, type safety.
- **Maintainability** — naming, single responsibility, duplication, test coverage, documentation for non-obvious logic.

Output is a structured markdown report — Summary, Critical Issues table, Suggestions table, "What Looks Good" section, and a Verdict (Approve / Request Changes / Needs Discussion). Predictable, scannable, easy to act on.

**Why I use it.** CI gates (ruff / mypy / coverage / type checks) verify *syntactic* correctness, not *design* correctness. This skill catches the layer between them — the kind of issues that pass tests but burn you in production. My personal rule: **never claim "ready to merge" on anything that changes behavior or touches >1 file without running this first.** Triggering it manually means I get review on demand at the exact granularity I want — full PR, single function, or a copy-pasted snippet — without spinning up agents I don't need.

Trivial changes (docs typo, single-line lint fix) skip it — overhead-heavy for those. For security-sensitive surfaces (auth, money, audit log, tenant isolation, crypto, RBAC) I additionally chain `/security-review` after.

**Source.** [`anthropics/knowledge-work-plugins`](https://github.com/anthropics/knowledge-work-plugins) — Anthropic's knowledge-work plugin marketplace (marketplace alias is `knowledge-work-plugins`). Note this is a **separate** repo from `anthropics/claude-plugins-official` and `anthropics/claude-code`.

**Install.**

```text
/plugin marketplace add anthropics/knowledge-work-plugins
/plugin install engineering@knowledge-work-plugins
/reload-plugins
```

The `engineering` plugin ships **ten** skills in one bundle: `code-review`, `architecture`, `debug`, `deploy-checklist`, `documentation`, `incident-response`, `standup`, `system-design`, `tech-debt`, `testing-strategy`. Installing it gets you all of them; this entry is specifically about `code-review`, which is the one I reach for most often.

**Invoke.**

```text
/engineering:code-review                  # asks what to review
/engineering:code-review <PR URL>         # reviews a GitHub PR
/engineering:code-review <file path>      # reviews a single file
```

---

## 3. `product-management:write-spec` — turn a vague idea into a structured PRD

The skill I reach for whenever a feature exists as a concept doc, a Slack thread, or a half-formed idea but **doesn't yet have engineering-grade acceptance criteria**. It's the bridge between "*what & why*" and "*when & how*".

**What it does.** Conversationally gathers context (user problem, target users, success metrics, constraints, prior art — asking the most important questions first, not dumping them all at once), then produces a structured PRD with these sections:

- **Problem Statement** — user pain, who's affected, cost of not solving it (grounded in evidence).
- **Goals** — 3–5 measurable outcomes, distinguishing user goals from business goals.
- **Non-Goals** — 3–5 explicit out-of-scope items with rationale (the anti-scope-creep section).
- **User Stories** — standard format ("As a *X*, I want *Y* so that *Z*"), grouped by persona, including edge cases.
- **Requirements** — MoSCoW categorization: **P0** Must-Have, **P1** Nice-to-Have, **P2** Future Considerations, each with acceptance criteria in Given/When/Then or checklist format.
- **Success Metrics** — leading indicators (adoption, activation, time-to-complete) + lagging indicators (retention, revenue, NPS) with specific targets and measurement methods.
- **Open Questions** — tagged by who needs to answer (eng / design / legal / data) and whether blocking or non-blocking.
- **Timeline Considerations** — hard deadlines, dependencies, suggested phasing.

**Why I use it.** PRDs prevent the failure mode where implementation starts from a "we should do something about X" message and silently picks goals, scope, and success criteria along the way — at which point the team is building a target nobody agreed to. The skill enforces the discipline of writing *non-goals* (explicit anti-scope-creep), of separating *outcomes* from *outputs* ("reduce time-to-first-value by 50%" not "build onboarding wizard"), and of pinning *measurable success* before code is written. My workflow: run `write-spec`, save output to `docs/prds/NN-feature-name.md`, then immediately chain `manual-qa-toolkit:analyse-story` on the acceptance-criteria subset to catch ambiguities before milestone breakdown.

Skip for: features with crystal-clear AC already (well-bounded refactors, small endpoints), or features whose PRD layer is already covered by an existing phase scope doc.

**Source.** [`anthropics/knowledge-work-plugins`](https://github.com/anthropics/knowledge-work-plugins) — same marketplace as `engineering` (see entry #2). If you already added it for `engineering`, you just install the second plugin on top.

**Install.**

```text
/plugin marketplace add anthropics/knowledge-work-plugins   # skip if already added
/plugin install product-management@knowledge-work-plugins
/reload-plugins
```

The `product-management` plugin ships **eight** skills: `write-spec`, `product-brainstorming`, `sprint-planning`, `stakeholder-update`, `metrics-review`, `roadmap-update`, `competitive-brief`, `synthesize-research`. Installing it gets all of them; this entry is specifically about `write-spec`, which is the one I reach for most often.

**Invoke.**

```text
/product-management:write-spec                         # asks what to spec
/product-management:write-spec <feature or problem>    # starts from a one-liner
```

---

*More entries to be added.*
