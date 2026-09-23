---
description: Full browser QA through Playwright MCP before committing — Boy Scout mode with fixes, every flow variant, a11y, UX, layout — seeded by the change, ending in a commit verdict
argument-hint: "[app URL] [PR number | ref range — defaults to the working branch diff incl. uncommitted] [--report-only] [free-text focus]"
---

Run the **complete** `playwright-web-qa` session in the running app and leave the product shining.
This is the pre-commit gate for user-facing work, the browser counterpart of `/code-review` and
`/security-review` — but where those read a diff, this one exercises the product and repairs it.

**Defaults are maximal — no scaling down.** Boy Scout mode, fixing, every flow variant, the full
breakpoint set, accessibility, UX, layout, exploratory tours, and product opportunities are all on.
The skill's advice to "scale the session to the change" does **not** apply to this command. The
only opt-out is `--report-only` (test and report everything, change no files).

**The diff seeds the session; it does not fence it.** The change decides *where testing starts and
what gets regression priority*. It never limits what is looked at, reported, or fixed: every defect
found anywhere on the way is in scope.

Load the `playwright-web-qa` skill first and apply all of its rules throughout. This command sets
scope, mode, and verdict; the skill decides how to test and how to fix.

## 0. Preconditions — stop early rather than fake it

- **Playwright MCP must be available.** Make one cheap call (`browser_tabs` with `action: "list"`)
  before anything else. If the `browser_*` tools are missing, disconnected, or erroring, stop and ask
  the user to restore the Playwright MCP server (`/mcp`, its config, or a restart). Never substitute
  scripts, `curl`, or reading the source.
- Parse `$ARGUMENTS`: an `http(s)://` URL is the app URL; a number is a PR (`gh pr diff <n>`); an
  `a..b` / `a...b` string is a ref range; `--report-only` disables file changes; any other text is a
  focus the user wants covered in addition to the diff.

## 1. Establish the diff

With no PR or range, diff the working branch against its merge-base with the default branch **and
include staged and unstaged changes** (`git diff <merge-base>` covers both) — the point is to test
before the commit exists. Read the full diff and enough surrounding code to know what the changed
lines do at runtime.

If the diff has no user-facing effect (tests, docs, CI, infra with no runtime change), say so in one
line — and still run the session on the product's primary flows unless the user asked only about
the diff. A shining product does not depend on the last commit touching the UI.

## 2. Charter

Build the charter (skill §1) in three rings, all of them in scope:

1. **Seed — the change.** For each user-facing change, find where it surfaces: router config,
   component and endpoint usages (search the code), templates, emails, exports. List screen → URL
   path and the flows through them. These are tested first and deepest, and every finding here is
   classified against the change (§4).
2. **Reach — everything connected.** Flows reachable from the seed screens, flows that reuse a
   changed shared component, and each defect's family as it is found (skill rules on hunting the
   family and widening the net).
3. **Product — the rest.** The product's primary flows (skill: flow inventory from navigation,
   primary CTAs, routes, object life histories), walked after rings 1–2.

Top risks come from the diff first (what would hurt most if this change broke), then from the
product (money, auth, data loss). Mode: **Boy Scout**. Budget: until the skill's Boy Scout exit
criterion is met — or until the user stops it; if the session must end early, the report says
exactly which rings and flows remain.

Show the charter to the user in a few lines, then proceed without waiting for approval unless
something is genuinely missing (credentials, a safe environment, how to start the app).

## 3. Find the running app

In order: the URL from `$ARGUMENTS`; a URL named in the project's `CLAUDE.md` / README / env files
for local development; a dev server already listening on a port the project's docs name. If none,
start the app the way the project's docs say (and stop it at the end), or ask for the URL if
starting it is not documented. Confirm the environment from evidence before any state-changing
action (skill §1): this run is for local or dev — if the only target is shared or live, ask before
submitting anything.

The app must serve **the working tree**. Rebuild or restart after every fix whose effect the running
app would not pick up by hot reload; otherwise confirmation of a fix proves nothing.

## 4. Run the full session

Execute every section of the skill on rings 1 → 2 → 3: all flow variants, the full breakpoint set,
functional heuristics, axe and the keyboard walk per screen and state, UX walkthrough and severity,
the measured layout pass, exploratory tours, product-opportunity hunting, console and network after
every step. Keep the session log.

Classify each finding by origin:

- **Introduced** — caused by this change (the behaviour differs from the base, or the changed code
  is the root cause). To check the base, read the base version with `git show <merge-base>:<path>`;
  never `git stash`, checkout, or reset — they touch the user's work. If it cannot be determined,
  mark it **Unclear**.
- **Pre-existing** — present regardless of this change. Fixed just the same.
- **Opportunity** — a missing feature (skill §8): proposed, never implemented without a decision.

## 5. Fix — by default

Unless `--report-only`, fix every reproduced "fix now" finding — Introduced and Pre-existing alike —
following the skill's Boy Scout rules exactly: reproduce first, fix the root cause, one change per
defect, tidyings kept apart from behaviour fixes, a failing-then-passing automated test for each
behaviour fix, never mask a symptom, confirmation with the original steps and then regression over
the flows the change touches plus the project's tests, lint, and type check. Findings that need a
product decision are **proposals**, not fixes. Loop until the Boy Scout exit criterion holds.

Never commit, stage, or stash. Group the resulting working-tree changes so the user can commit them
cleanly:

- **Change fixes** — repairs to Introduced findings; they belong with the user's change.
- **Boy Scout fixes** — repairs to Pre-existing defects; separate commit(s), one concern each.
- **Tidyings** — presentation-only cleanups; their own commit.

List the files of each group and a suggested Conventional Commits message per group.

## 6. Verdict and report

Produce the skill's report skeleton (§10) with findings split into **Introduced**, **Unclear**,
**Pre-existing**, **Proposals**, and **Opportunities**, each marked *fixed* / *open*. Then end with
exactly one verdict line:

- **Ready to commit** — the Boy Scout exit criterion holds across all walked flows: no open finding
  at Blocker / Critical / Major, no open UX severity 3–4, no open WCAG A / AA violation, no
  unexplained console error or unexpected 4xx / 5xx, every seed and reach flow passed in every
  variant, and all fixes passed confirmation and regression.
- **Fix before commit** — an open blocking finding remains (e.g. it needs a decision, or
  `--report-only` was used). List them by id, with the proposal for each.
- **Cannot verify** — the session could not cover the seed ring (app would not start, credentials or
  data missing, environment not safe, Playwright MCP lost). Say exactly what was not covered.

Proposals and opportunities that do not hide a blocking defect do not block the commit; they are
listed for the product owner.

State plainly what the run did **not** cover — rings or flows not reached, roles without
credentials, `browser_resize` vs real devices, third-party sandboxes unavailable. A green verdict is
evidence about what was walked, not a guarantee about the whole product.

Finish with the skill's cleanup: `browser_close`, stop anything started, reset throttling and
routes, remove injected styles, delete the `.playwright-mcp/` artefacts directory from the workspace
(or confirm it is git-ignored), and list the test data created.
