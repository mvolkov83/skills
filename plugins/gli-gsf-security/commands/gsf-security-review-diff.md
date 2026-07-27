---
description: Review a change for GLI-GSF security-control violations it introduces
argument-hint: "[PR number | ref range — defaults to the working branch diff]"
allowed-tools: Read, Grep, Glob, Bash(git diff:*), Bash(git log:*), Bash(git merge-base:*), Bash(git status:*), Bash(gh pr diff:*), Bash(gh pr view:*)
---

Review a change for GLI-GSF-1 security controls it **violates or weakens**. This is a diff-scoped
security review: it judges what the change does, not the whole system's security posture. For a
posture audit of a service or its infrastructure, use `/gsf-security-review-surface`.

The control reference is at
`${CLAUDE_PLUGIN_ROOT}/skills/gli-gsf-security/references/requirements.md`, and the engineering
rules are in the `gli-gsf-security` skill. Read the entries for the GIS ids actually in play — do
not read the whole file.

## 1. Establish the diff

`$ARGUMENTS` may be a PR number (`gh pr diff <n>`) or a ref range (`git diff <range>`). With no
argument, diff the working branch against its merge-base with the default branch, and include
uncommitted changes if any exist. Include infrastructure diffs — Terraform, Helm, Kubernetes
manifests, security-group / IAM / network policy — not only application code; most GSF controls
are code *or infra or config*.

Read the full diff and enough surrounding code and configuration to know what the changed lines
actually do.

## 2. Gate by security domain — before checking any control

Decide which of these the change touches. **Check controls only for the domains that come back yes.**

| Domain | Touched when the change involves |
|---|---|
| Access control & RBAC | roles, permissions, service accounts, credential storage, separation of duties, lockout |
| Authn & session | authentication, session tokens, authorization caching, timeout, MFA |
| Secrets & crypto | keys, algorithms, certificates, encryption grade, key rotation/lifecycle |
| Data at rest | encryption of stored data, error handling that could clear data, durable-write, masking, DLP |
| Network segmentation | subnets, VPCs, security groups, network policy, DB-tier placement, IDS/IPS, ports |
| Communications | TLS, protocol hardening, component enrollment, public-network traffic, replay protection |
| Hardening & config | server config, default parameters, one-function-per-server, removed functionality |
| DNS | DNS config, DNSSEC, zone transfers, registrar changes |
| Remote access & firewall | remote-access config, firewall rules, default-deny, bypass paths, anti-spoofing |
| Logging & monitoring | security logs, log tamper-protection, aggregation, incident detection/escalation |
| Backup & recovery | backup cadence, immutability, off-site copies, restart/recovery of transactional paths |
| Change mgmt & SDLC | version control, release gating, env separation, patching, deploy pipeline |
| Provider integration | third-party accounts, provider network paths, routing between provider and production |

If no domain is touched, say exactly that in one line and stop.

## 3. Judge the touched domains

Apply the `gli-gsf-security` rules for those domains. Confirm each candidate finding against the
control entry — note its assurance tier (GIG1 baseline / GIG2 / GIG3 enhanced), because a control
above baseline may not apply to this deployment.

Report a finding only when the change itself creates or worsens the problem:

- it introduces a violation (a service account with broad privilege, a firewall rule that is not
  default-deny, a session token cached on the component, a provider path that can reach production,
  an error handler that clears sensitive data, a secret in code/config, a mutable-backup policy);
- it removes or weakens a control that was there (tightened rule loosened, an audit log made
  editable, a network segment flattened, SoD collapsed into one role);
- it adds a security-relevant surface built incomplete — a new endpoint with no authorization, a new
  data store with no encryption-at-rest, a new integration on a shared segment.

Do **not** report a control as violated merely because the change does not implement it — that is
`/gsf-security-review-surface` territory.

## 4. Severity

- **High** — introduces an exploitable exposure or defeats a containment control: privilege
  escalation path, a provider or public path into production, secrets in the diff, a firewall no
  longer default-deny, an audit log an actor can rewrite, SoD collapsed.
- **Medium** — weakens defence in depth or leaves a security surface incomplete: missing
  encryption-at-rest on a new store, a broad-but-not-yet-exploited grant, a backup without
  immutability.
- **Low** — will complicate evidence or response later: a hard-coded threshold, a log without the
  acting actor, an unlogged network change.

## 5. Output

A findings table, most severe first:

| Control | Tier | Severity | Location | What the change does | Fix |
|---|---|---|---|---|---|
| `GIS-x.y.z` | GIG1 | High | `main.tf:42` | one sentence, concrete | one sentence, concrete |

Then two required sections:

**Domains skipped** — which domains the diff did not touch, one line.

**What this review cannot see** — state plainly that a diff review cannot find absence: it cannot
tell whether encryption-at-rest, network segmentation, immutable backups, DNSSEC, tamper-evident
logging or the disaster-recovery plan exist elsewhere in the system; it does not observe runtime,
cloud console state, or IAM outside the diff; and it says nothing about the operator's GISMS
governance programme.

If there are no findings, say so — and keep both sections. A clean diff review is not a secure
system.

## Constraints

- Never state or imply a certification verdict, an assurance-tier attainment, or that the change
  "passes GLI-GSF". You are reviewing a diff against engineering rules derived from the framework.
- Cite GIS control ids exactly as they appear in the reference. Do not invent ids.
- Thresholds in the rules (5-/15-minute timeouts, three-attempt lockout, daily backup, quarterly
  scan) are reference values the jurisdiction overrides. Flag a hard-coded threshold as a
  configurability issue, not as a wrong number.
- For application-level gaming correctness and the financial-record integrity model, defer to
  `gli-19-platform-engineering`; for money/idempotency/ledger, to `money-and-payments-best-practices`.
