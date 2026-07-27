---
description: Audit a service or its infra against the GLI-GSF security controls
argument-hint: "<path to the service, module, or infra directory to audit>"
allowed-tools: Read, Grep, Glob, Bash(git log:*), Bash(git ls-files:*), Bash(find:*)
---

Audit the code and infrastructure at `$1` against the GLI-GSF-1 security controls whose evidence is
code, infrastructure or configuration, and produce a security-posture gap list. This is a
**surface** audit — it looks for controls that are missing, which a diff review structurally cannot
do. For reviewing a specific change use `/gsf-security-review-diff`.

The control set is at `${CLAUDE_PLUGIN_ROOT}/skills/gli-gsf-security/references/requirements.md` —
163 controls across nine GIS domains, each carrying an assurance tier (GIG1 baseline / GIG2 / GIG3
enhanced). The `gli-gsf-security` skill is the engineering restatement of the same set.

If `$1` is empty, ask which service or infrastructure to audit. Audit one ownership boundary at a
time, not a whole platform in one pass.

## 1. Scope the ownership boundary first — not optional

Security controls are heavily shared with the cloud provider, managed services, and third parties.
Before auditing, establish what this codebase/infra **owns** versus **inherits or delegates**. Read
entry points, IaC (Terraform / Helm / Kubernetes / security groups / IAM), dependency and network
configuration, and produce:

| Control area | Owned here / inherited (cloud/managed) / delegated (provider) / not offered | Evidence |
|---|---|---|

Inheritance is a real, creditable thing here in a way it is not for gaming-application rules: disk
encryption, physical DR-site separation, DNSSEC, hypervisor isolation and daily-backup mechanics
are frequently provided by the managed platform. Record which side of each boundary this target
sits on, and what the inherited control's evidence would be (a cloud config, a managed-service
setting).

Present the boundary to the user and **ask them to confirm or correct it before continuing** — a
wrong boundary makes every downstream verdict wrong, and these rulings are the operator's call.

## 2. Audit by GIS domain

Work domain by domain — access control, authn/session, secrets & crypto, data at rest, network,
communications, hardening, DNS, remote access & firewall, logging & monitoring, backup & recovery,
change mgmt & SDLC, provider integration. Related controls share evidence; a domain pass reads the
relevant code and config once. For a large surface, run domains as parallel subagents and merge;
say so and say what each covered.

**Verdicts come from primary artifacts only**: source code, IaC, Kubernetes manifests, security-
group / IAM / network-policy definitions, configuration, and tests. Do **not** accept a security
policy document, a runbook, a README, or a prior audit as evidence that a control exists — find the
mechanism in code or configuration. A control asserted in prose with no implementation behind it is
`non-compliant`, not `compliant`; surfacing that gap is the point of this audit.

## 3. Record a verdict per control

| Status | Means | Required fields |
|---|---|---|
| `compliant` | the control is implemented and satisfies the criteria | evidence as `file:line` or the IaC resource |
| `partial` | implemented with a specific identified gap | evidence + the gap in one sentence |
| `non-compliant` | not implemented | what is absent |
| `inherited` | provided by the cloud/managed platform | which service + the config that shows it enabled |
| `not-applicable` | outside the confirmed ownership boundary | which boundary and why |
| `needs-info` | owned elsewhere or evidence not in this target | where to look |

Never mark `compliant` or `inherited` without concrete evidence — an inherited control still needs
the setting that proves it is on (an unencrypted bucket is not "inherited encryption"). Never mark
`not-applicable` on your own judgment — only against the boundary the user confirmed. Note each
control's tier; a missing GIG2/GIG3 control is a lower priority than a missing GIG1 one.

## 4. Output

**Counts by status and by tier**, as a table. Do **not** compute a security score or a readiness
percentage — the denominator depends on scoping and inheritance rulings, and a number from a code
reader reads as an assurance rating it is not.

**Gap list**, ordered as a remediation order rather than by control number, GIG1 gaps before
GIG2/GIG3: put containment controls others depend on first (network segmentation and default-deny
firewalls before the services behind them; separation of duties before the pipelines that rely on
it; tamper-evident logging before the monitoring that reads it). For each: control id, tier, what
is missing, the concrete fix in this codebase/infra, and an effort size (S / M / L).

**Excluded**, grouped with reasons: `inherited`, `not-applicable`, `needs-info`.

**Out of scope of this command** — state plainly that this covers only controls whose evidence is
code, infrastructure or configuration; that the operator's GISMS governance programme (policies,
procedures, staffing, risk assessments, incident-response process) and the laboratory's own testing
are not audited.

## Constraints

- This is not a certification assessment or a penetration test, and must never be presented as one.
  It reads code and configuration; it does not exploit, and a laboratory assesses the framework
  using evidence the operator supplies.
- Cite GIS control ids exactly as they appear in the reference; do not invent ids or audit controls
  absent from it.
- Report honestly. A handful of compliant controls out of the applicable set is a useful result; a
  padded one is worse than none, because it will be trusted and it hides real exposure.
- Thresholds in the controls are reference values the jurisdiction overrides — a configurable value
  set differently is not a gap; a hard-coded one is.
- If the project keeps its own security or compliance records (a control catalog, scope rulings,
  prior audit runs), conform to their schema and status vocabulary instead of this command's, and
  say that you did.
- For application-level gaming correctness defer to `gli-19-platform-engineering`; for money/ledger
  to `money-and-payments-best-practices`; for instrumentation and log correlation to
  `observability-best-practices`.
