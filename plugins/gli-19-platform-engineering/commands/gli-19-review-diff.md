---
description: Review a change for GLI-19 violations it introduces
argument-hint: "[PR number | ref range — defaults to the working branch diff]"
allowed-tools: Read, Grep, Glob, Bash(git diff:*), Bash(git log:*), Bash(git merge-base:*), Bash(git status:*), Bash(gh pr diff:*), Bash(gh pr view:*)
---

Review a change for GLI-19 requirements it **violates or undermines**. This is a diff-scoped
review: it judges what the change does, not what the system as a whole is missing. For a
whole-service gap assessment use `/gli-19-review-surface` instead.

The requirement reference is at
`${CLAUDE_PLUGIN_ROOT}/skills/gli-19-platform-engineering/references/requirements.md`. Read the
entries for the ids actually in play — do not read the whole file.

## 1. Establish the diff

`$ARGUMENTS` may be a PR number (`gh pr diff <n>`) or a ref range (`git diff <range>`). With no
argument, diff the working branch against its merge-base with the default branch, and include
uncommitted changes if any exist.

Read the full diff before judging anything. Also read enough surrounding code to know what the
changed lines actually do — a diff review that misreads intent produces confident nonsense.

## 2. Gate by domain — do this before checking any rule

Decide which of these the change touches. **Check rules only for the domains that come back yes.**

| Domain | Touched when the change involves |
|---|---|
| Server authority | game outcome, RNG, paytable evaluation, anything the client sends that decides a result |
| Time | timestamps on transactions/rounds/events, clock sources, scheduling |
| Records | schema or writes for rounds, plays, themes, player accounts, incentives, jackpots, event logs, operator users |
| Record integrity | mutation of financial or audit rows, admin adjustment/void/cancel, hashing, actor fields |
| Wallet semantics | wager debit, award settlement, balance composition, bonus/restricted credits, holds, transfers |
| Account lifecycle | registration, KYC, activation, credential change, login, lockout, session timeout, limits, exclusions |
| Control surfaces | disable/kill-switch, jackpot parameters, game recall |
| Reporting | any report or aggregate surface |
| Location | geolocation, IP checks, VPN/proxy detection |
| Program integrity | component verification, version pinning, release/deploy manifests |
| Communications | transport security, component enrollment, third-party integration paths, crypto, key handling |
| Availability | backup, recovery, failover, restart/resume of transactional paths |

If no domain is touched, say exactly that in one line and stop. Do not manufacture findings from
an unrelated diff.

## 3. Judge the touched domains

Apply the `gli-19-platform-engineering` skill's rules for those domains. For each candidate
finding, confirm against the requirement entry before reporting it — the criteria there are what
a laboratory tests, and they frequently narrow what looks like a violation.

Report a finding only when the change itself creates the problem:

- it introduces a violation (client-side outcome logic, real-money-before-restricted-credits
  consumption order, a financial mutation with no actor, an `UPDATE` or `DELETE` on an append-only
  financial row, a player-facing credential accepted on an operator surface);
- it removes or weakens a control that was there;
- it adds a record, endpoint or column that a requirement will need and that is being built
  incomplete now — a new financial table with no integrity column, a new admin RPC with no
  authorization, a new theme/paytable model with no aggregate.

Do **not** report a requirement as violated merely because the diff does not implement it. That
belongs to `/gli-19-review-surface`.

## 4. Severity

- **High** — a regulatory or security violation is introduced: outcome authority moved to the
  client, financial data mutable or unattributable, an operator surface without operator identity,
  funds consumed in the wrong order, an exclusion or limit bypassable.
- **Medium** — the change makes a required record, report or reconstruction unsatisfiable, or
  leaves a new obligation-bearing structure incomplete.
- **Low** — the change will complicate evidence later: a timestamp from a local clock, a missing
  reason field on a state change, an event that should be significant and is not logged.

## 5. Output

A findings table, most severe first:

| Requirement | Severity | Location | What the change does | Fix |
|---|---|---|---|---|
| `GLI19-x.y.z` | High | `file.py:120` | one sentence, concrete | one sentence, concrete |

Then two required sections:

**Domains skipped** — which domains the diff did not touch, one line. This tells the reader what
the review deliberately did not look at.

**What this review cannot see** — state plainly that a diff review cannot find absence. It cannot
tell whether the per-theme aggregate, the regulator report, the retention window, the backup
scheme or the operational procedure exists elsewhere; it does not observe runtime, infrastructure
or configuration outside the diff; and it says nothing about the operator's procedural
obligations.

If there are no findings, say so — and keep both sections. A clean diff review is not readiness.

## Constraints

- Never state or imply a certification verdict, a readiness level, or that the change "passes
  GLI-19". You are reviewing a diff against engineering rules derived from the standard.
- Cite requirement ids exactly as they appear in the reference. Do not invent ids or clause numbers.
- Thresholds in the rules (thirty minutes, three attempts, daily) are reference values that the
  jurisdiction overrides. Flag a hard-coded threshold as a configurability issue, not as a wrong
  number.
- For money representation, idempotency and ledger structure, defer to the
  `money-and-payments-best-practices` skill rather than re-deriving them here.
