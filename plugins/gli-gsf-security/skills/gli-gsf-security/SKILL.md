---
name: gli-gsf-security
description: Security-engineering rules for an interactive gaming platform under the GLI Gaming Security Framework (GLI-GSF-1) — the deep security-controls layer that GLI-19's technical-security appendix only summarizes. Covers identity and logical access control (individual credentials, RBAC with least privilege, separation of duties, hashed credential storage, three-attempt lockout, timely deprovisioning, access-attempt and override audit logs with actor), user authentication and session authorization (ephemeral per-request authorization never stored on the component, random in-memory session tokens cleared at session end, inactivity timeout, failed-authorization session termination), secrets and cryptography (approved algorithms, encryption grade by sensitivity, key agility with distinct keys per purpose, key generation / storage / expiry / rotation lifecycle, trusted-authority certificates), protecting sensitive data at rest (encryption or segregation-of-duties + access logging, no error-driven auto-clear, durable-write-before-eviction, masking, DLP), network security and segmentation (production databases isolated from patron-facing servers, logical network separation, no single point of DoS, IDS/IPS, 24/7 monitored entry-exit points, disabled unused ports), communications security (documented secure protocol, encryption + authentication of critical traffic, hardening against malformed messages, always-encrypted public-network traffic with anti-replay), component hardening (remove defaults, one primary function per server, remove unnecessary functionality, prevent unauthorized server-side programming), DNS security (DNSSEC, MFA-gated changes, restricted zone transfers), remote access and firewalls (default-disabled least-function remote access with audit log, application-level firewall at security-domain boundaries, default-deny with no bypass path, anti-spoofing), logging and monitoring (per-component security logs protected against tampering, centralized aggregation, incident detection and escalation, versioned tamper-evident audit-log retention), backup and recovery (at-least-daily immutable backups, off-site redundancy, no lost or duplicated transactions on restart, disaster recovery), change management and secure SDLC (version control, authorized-versions-only, change log, rollback, environments separated from production, secure coding, tested patching), and service-provider integration security (dedicated monitored accounts, encrypted strongly-authenticated segmented communication that cannot route into production). Use this skill whenever the user is designing, writing, reviewing, hardening or auditing the SECURITY of an interactive gaming / online casino platform or its infrastructure — access control and RBAC, authentication and session management, secrets management and key rotation, encryption at rest and in transit, network segmentation and firewall rules, DNS configuration, remote access, security logging and SIEM, backup immutability and disaster recovery, secure SDLC and change management, or integrating a third-party provider securely. Trigger on phrases like "gaming security", "GLI-GSF", "GISMS", "security controls", "RBAC", "least privilege", "separation of duties", "session token", "credential storage", "password hashing", "MFA", "key rotation", "KMS", "encryption at rest", "TLS", "network segmentation", "firewall rules", "default-deny", "DNSSEC", "remote access", "SIEM", "audit log tampering", "immutable backup", "disaster recovery", "secure SDLC", "patch management", "third-party provider integration", and on infrastructure work (Terraform / Helm / Kubernetes / security groups / IAM) for a gaming platform. Also trigger on the phrasings a developer uses to START a security review — "security review", "review the security", "security audit", "harden this", "threat model" / "threat-model this", "penetration test" / "pentest", "OWASP", "vulnerability assessment", "find security issues" — whenever the target is an interactive gaming / online casino platform or its infrastructure. Trigger even when the request is bare and does not repeat the word "gaming" — e.g. a plain "do a security review of this branch", a `/security-review`-style gate, or "any vulnerabilities here?" — as long as the surrounding codebase is a gaming platform (player accounts, wallet or ledger, game sessions or rounds, RGS or aggregation, bonus or jackpot engines): in that context this is the applicable security authority and should load alongside any generic review. Trigger even for a small change — one firewall rule, a new service account, a session-token store, a backup policy, a provider integration path — because these are where a security finding or a breach originates. Do NOT use for the application-level gaming correctness that the `gli-19-platform-engineering` skill owns (server-authoritative outcomes, wagering order, records, game recall), for payment-card-specific controls (PCI-DSS), for land-based gaming devices (GLI-11), or for generic non-gaming application security with no interactive-gaming platform in scope.
---

# GLI-GSF security engineering

Security-engineering rules for an interactive gaming platform assessed under the GLI Gaming Security
Framework (GLI-GSF-1), the framework that is progressively superseding the security portions of
GLI-11 / GLI-19 / GLI-27 / GLI-33. Every rule maps to one or more GIS control ids; the full
paraphrased control text, with testable criteria and assurance tiers, is in
`references/requirements.md` — read the entry when a rule's scope is unclear.

Two commands ship alongside these rules. `/gsf-security-review-diff` reviews a change for security
controls it violates or weakens. `/gsf-security-review-surface` audits a service or its
infrastructure for controls it does not implement — the posture gaps a diff review cannot find.

**How this relates to `gli-19-platform-engineering`.** That skill covers application-level gaming
correctness and the code-shaped security touchpoints a *feature* developer hits, and it deliberately
declined the infrastructure and network security territory. **This skill owns that territory** — DNS,
firewalls, remote access, hardening, network segmentation, IDS/IPS, patching, secure SDLC — and it is
the *deep* version of the access-control, data-at-rest, communications and crypto controls that the
platform skill only summarized. The overlap is intentional and layered: reach for the platform skill
when building a gaming feature, this one when hardening the system it runs on. Where a control here is
the security depth behind a gaming-application rule there, the cross-reference is noted.

**These rules are engineering guidance, not the standard.** They cover the GLI-GSF-1 controls whose
evidence is code, infrastructure or configuration. They carry no certification weight, they do not
cover the operator's governance programme (policies, procedures, staffing, risk assessments, the
GISMS itself), and they are never a substitute for the framework document or the testing laboratory's
judgment.

**Assurance tiers.** GLI-GSF assigns each control a tier: **GIG1** is the baseline every deployment
meets; **GIG2** and **GIG3** are enhanced-assurance controls for higher-risk deployments. Rules below
flag *(GIG2)* / *(GIG3)* where the control is above baseline, so you can tell table-stakes from
enhancement. If you must sequence work, land every GIG1 control first.

## 1. Identity and logical access control

1. Give every human and every service its own individual credential, provisioned through a formal
   process. No shared or generic accounts — including for third-party providers.
   → GIS-3.5.2, GIS-2.3.3, GIS-3.5.11
2. Enforce role-based access at least privilege: a subject gets only the functions its role needs,
   checked at authorization time, not merely hidden in a UI. → GIS-3.5.3, GIS-2.3.4 *(GIG2)*, GIS-2.3.5
3. Separate duties across multiple access levels controlling view / change / delete of critical data,
   and split account administration from the accounts it administers. → GIS-3.5.10
   **Why:** the account that can grant privileges must not be the account that uses them. A single
   super-role that both administers users and operates on data means one compromised credential is a
   total compromise, and it defeats every audit-log-based control because the same identity can grant
   itself access and act.
4. Store credentials only hashed (passwords via a strong KDF) or encrypted; never in plaintext or
   reversible encoding. → GIS-3.5.7
5. Gate credential reset with a fallback at least as strong as the primary method, using MFA.
   → GIS-3.5.8 *(GIG2)*
6. Lock an account after at most three failed authentication attempts and notify an administrator;
   flag accounts with suspected-stolen credentials. → GIS-3.5.12
7. Deprovision on time: deactivate lost, compromised or terminated credentials as soon as reasonably
   possible, and have a process to change / block / deactivate / remove an account on role change or
   termination. → GIS-3.5.9, GIS-3.5.17
8. Restrict privileged utility programs that can override application or OS controls, and restrict
   access to inactive or closed accounts. → GIS-3.5.14, GIS-3.5.18
9. Gate every change to a critical system parameter (audit settings, password policy, security
   levels, manual DB updates) behind an authorized process, and record it with date/time, the
   parameter, the reason plus initial and final values, and the acting account. → GIS-3.5.4
10. Log every logical access attempt with date/time, account id, source IP, success/failure and
    session duration; log every out-of-scope intervention (void, override, correction) with the
    affected component, reason, before/after values and the acting account. → GIS-3.5.13, GIS-3.5.15
    **Why:** the acting-account field is the one that goes missing, and it is the same footgun the
    `gli-19-platform-engineering` skill flags for financial mutations — an override with no attributed
    actor is unauditable. Bind the actor at the point of the write, not from an ambient request
    context that may be a shared service identity.
11. Model the operator-user account with every field the framework enumerates — account id, name and
    title, executable functions, created, last access with IP, last credential change, deactivation,
    group membership, current and previous statuses — and back it up. → GIS-3.5.16

## 2. Authentication and session authorization

12. Validate authorization on every request and re-verify periodically; do not trust a
    once-authenticated session indefinitely. → GIS-3.6.1
13. Derive authorization information per request from the system; never store it on the component that
    is being authorized. → GIS-3.6.3 *(GIG2)*
    **Why:** authorization data cached on the component is authorization that outlives its grant and
    travels with a compromised node. Fetch-at-request-time is what lets a revocation take effect
    immediately instead of at the next cache expiry.
14. Make session-authorization tokens random, hold them in memory (or a short-TTL store), and remove
    them at session end. → GIS-3.6.4 *(GIG2)*
15. Terminate the active session when authorization exceeds a configurable failure threshold.
    → GIS-3.6.2
16. Time out idle sessions — no more than 5 minutes on portable/mobile devices and 15 minutes on
    everything else, unless the regulator specifies otherwise — and tighten connection-time limits
    further on high-risk paths such as remote access. → GIS-3.6.6, GIS-3.6.5
    **Why:** this is materially stricter than the 30-minute inactivity window GLI-19 states for player
    sessions — GSF is about operator/administrative access to the system, not player gameplay. Keep
    the timeout configurable, but default it to the tighter GSF value for staff and admin surfaces.

## 3. Secrets, cryptography and key management

17. Choose the encryption grade for the data's sensitivity and use only approved algorithms; review
    them periodically for continued strength. → GIS-7.2.2, GIS-5.7.8
18. Keep keys agile — use distinct keys per purpose so any one algorithm or key can be changed or
    replaced without touching the others. → GIS-7.2.4
    **Why:** one key (or one algorithm) used everywhere means a rotation or a break is an all-or-
    nothing, downtime-forcing event, so in practice it never happens and the weak key stays. Distinct
    keys per purpose make rotation routine, which is the only way it actually gets done.
19. Restrict key generation to authorized processes, store keys encrypted (under a different key or
    method) on redundant media, monitor expiry, and implement secure keyset rotation. → GIS-7.2.6,
    GIS-7.2.7, GIS-7.2.8, GIS-7.2.10
20. Authenticate with certificates from a trusted authority carrying owner, issuer, validity dates and
    a verifiable unique serial; validate them rather than accepting any presented certificate.
    → GIS-4.1.13
21. Apply message authentication (a MAC or signature) to integrity-critical data that need not be
    concealed but must not be silently altered. → GIS-4.1.12 *(GIG2)*
    For ledger hash-chaining and financial-record integrity specifically, defer to the
    `money-and-payments-best-practices` and `gli-19-platform-engineering` skills.

## 4. Protecting sensitive data at rest

22. Encrypt files, directories and databases holding sensitive data; where encryption is genuinely not
    used, restrict viewing with segregation of duties and monitor and record every access. → GIS-4.1.4
    *(GIG2)*, GIS-4.1.6, GIS-4.1.10 *(GIG2)*
23. Never let an error path auto-clear sensitive data, and never allow in-memory sensitive state to be
    evicted before it has been durably transferred to the database or another secured component.
    → GIS-4.1.8, GIS-4.1.9
    **Why:** durable-write-before-acknowledge is the same discipline the ledger requires; here it is a
    data-integrity control, and the failure mode is a crash or an error handler that discards data the
    system has already told a caller it accepted.
24. Store sensitive data durably so it survives power loss and hardware/module replacement during
    maintenance. → GIS-4.1.15, GIS-4.1.17
25. Mask or redact sensitive data in UIs, logs and exports per policy, and apply data-leakage-
    prevention controls on the paths that process, store or transmit it. → GIS-4.1.20, GIS-4.1.16
    *(GIG2)*. For redaction at the log/serialization boundary, defer to
    `observability-best-practices`.
26. Validate and reject corrupt data at the boundaries that write it. → GIS-4.1.3 *(GIG2)*

## 5. Network security and segmentation

27. Put production databases holding sensitive data on a network segment isolated from any patron-
    facing servers. → GIS-4.1.14
    **Why:** this is the control that turns an application-layer compromise of a public-facing service
    into a contained incident instead of direct database access. Flat networks where the web tier can
    reach the DB tier directly are how an SQL-injection or an RCE becomes a full data breach.
28. Separate networks logically so traffic only rides links its hosts can service, authenticate and
    encrypt all network management, and ensure no single component failure causes denial of service.
    → GIS-5.7.2, GIS-5.7.3, GIS-5.7.4
29. Identify, secure and monitor all network entry and exit points 24/7; secure hubs, services and
    ports and disable unused services and non-essential ports. → GIS-5.7.5 *(GIG2)*, GIS-5.7.6,
    GIS-5.7.7
30. Run an IDS/IPS covering the attack set; scan for rogue devices at least quarterly and auto-disable
    them; keep a reconcilable device-access log. → GIS-5.6.1, GIS-5.6.2 *(GIG2)*, GIS-5.6.3 *(GIG2)*,
    GIS-5.6.4
31. Log every network-infrastructure change. → GIS-5.7.9
32. In cloud/virtualized environments, apply the same controls, and place redundant instances on
    separate hypervisors (spread replicas across availability zones / anti-affinity) so there is no
    shared single point of failure. → GIS-3.8.1 *(GIG2)*, GIS-3.8.2 *(GIG2)*

## 6. Communications security

33. Give each component pair a documented secure communication protocol with error detection and
    recovery and anti-tamper techniques; harden it to survive malformed-message attacks. → GIS-5.1.1,
    GIS-5.1.2, GIS-5.1.5
34. Encrypt and authenticate all traffic critical to gaming or sensitive-data handling, and restrict
    communication to enrolled, authenticated components. → GIS-5.1.3, GIS-5.1.4
35. Ensure a communications failure cannot damage data integrity, and do not re-authenticate a
    component after an interruption until its resumption self-tests have passed. → GIS-5.1.6, GIS-5.1.7
36. Always encrypt sensitive data over the internet or any public network, with protection against
    replay and misrouting. → GIS-5.2.1, GIS-5.2.2
37. Encrypt and strongly authenticate any channel carrying financial-transaction detail. → GIS-2.8.4.
    For the financial semantics behind these channels, defer to `money-and-payments-best-practices`.

## 7. Component hardening and configuration

38. Establish, document, monitor and review component configurations; harden against known
    vulnerabilities to best practice and reassess regularly. → GIS-7.3.1, GIS-7.3.2
39. Remove default parameters that present a risk, configure security parameters to prevent misuse, and
    remove all unnecessary functionality — scripts, drivers, subsystems, unused web servers. → GIS-7.3.4,
    GIS-7.3.7, GIS-7.3.8
40. Run one primary function per server; do not co-locate functions that require different security
    levels. → GIS-7.3.5
    **Why:** co-locating a high-trust and a low-trust function on one host makes the low-trust one the
    attack surface for the high-trust one. Separation caps the blast radius of any single compromised
    service.
41. Add compensating controls for any insecure service, protocol or daemon that must run. → GIS-7.3.6
42. Prevent unauthorized user-initiated server-side programming that could modify the database, while
    still allowing gated, logged administrative maintenance. → GIS-3.7.1

## 8. DNS security

43. Run separate, secured primary and secondary DNS; restrict zone transfers to known hosts; enable a
    cache-poisoning defence such as DNSSEC; gate DNS changes behind MFA. → GIS-7.1.1 *(GIG2)*,
    GIS-7.1.4 *(GIG2)*, GIS-7.1.5 *(GIG2)*, GIS-7.1.3 *(GIG2)*

## 9. Remote access and firewalls

44. Default remote access to disabled; when enabled, secure it (MFA), accept only firewall-permitted
    connections, limit it to the functions the job needs, and forbid unauthorized remote user
    administration or OS/DB access beyond retrieval. → GIS-8.1.3, GIS-8.1.2, GIS-8.1.4, GIS-8.1.5,
    GIS-8.1.6, GIS-8.1.7
45. Keep a remote-access audit log. → GIS-8.1.8
46. Route all communications through at least one approved application-level firewall at the boundary
    of any two dissimilar security domains, and firewall any redundant path too. → GIS-8.2.1, GIS-8.2.2,
    GIS-8.2.4
47. Leave no path that bypasses the firewall, run only firewall-related applications on it, and keep
    its accounts to a minimum. → GIS-8.2.3 *(GIG2)*, GIS-8.2.5, GIS-8.2.6
    **Why:** a default-deny firewall is worth nothing if a peer on the same broadcast domain can reach
    the protected host by another route. The bypass path — a second NIC, a management interface, a
    forgotten peering — is the control's most common silent failure.
48. Default-deny: reject every connection except those specifically approved, reject impossible source
    addresses (anti-spoofing), require encryption and strong authentication for remote access through
    the firewall, keep a tamper-resistant firewall audit log, and lock out on a failed-connection
    threshold. → GIS-8.2.7, GIS-8.2.8, GIS-8.2.9, GIS-8.2.10, GIS-8.2.11

## 10. Logging, monitoring and detection

49. Generate predefined security logs on each critical component and protect them against tampering
    and unauthorized access. → GIS-7.4.1, GIS-7.4.2 *(GIG2)*
    **Why:** a log an attacker can edit is not evidence. Ship logs off the generating host to
    append-only / tamper-evident storage; a security log that lives only where the incident happens is
    the first thing an intruder rewrites.
50. Monitor the whole platform — every critical component and data transmission, including provider
    traffic — for confidentiality, integrity, availability and anomalies; centralize aggregation and
    wire it to a response workflow. → GIS-3.1.1 *(GIG2)*, GIS-3.1.6 *(GIG2)*. For instrumentation and
    correlation, defer to `observability-best-practices`.
51. Detect, prevent, mitigate and respond to active and passive attacks; continuously monitor for
    incidents, log all suspected or actual ones, and escalate automatically on threshold breach.
    → GIS-3.1.4, GIS-3.3.3
52. Monitor and adjust resource capacity for availability, and keep a performance audit log with
    reporting. → GIS-3.1.2, GIS-3.1.3 *(GIG2)*
53. Where an electronic document retention system holds alterable reports or audit logs, keep the
    original plus every version, a unique signature per version, a change log with actor / time /
    delta, complete indexing, restricted write access with an admin-activity log, and redundant
    backed-up storage. → GIS-3.9.1, GIS-3.9.2, GIS-3.9.3, GIS-3.9.4, GIS-3.9.5, GIS-3.9.7

## 11. Backup, redundancy and recovery

54. Back up at least daily, with immutability safeguards that prevent alteration or deletion of the
    backup. → GIS-4.2.1, GIS-4.2.2
    **Why:** a backup that the same credentials which run production can delete or rewrite is not a
    backup against ransomware or a malicious insider — it is a second copy with the same blast radius.
    Object-lock / WORM immutability is what makes it a recovery point rather than another target.
55. Keep mirrored/redundant copies on a non-volatile medium with restore support and disk-failure
    integrity; grant the backup the same security and access controls as production. → GIS-4.2.3,
    GIS-4.2.4, GIS-4.2.5, GIS-4.2.9
56. Transfer backups immediately to a physically separate, secured location; an optional cross-region
    or cross-cloud copy satisfies the enhanced tier. → GIS-4.2.6, GIS-4.2.7, GIS-4.2.8 *(GIG2)*
57. Build redundancy and modularity so no single component failure loses critical data, and log any
    significant component unavailability. → GIS-4.3.1, GIS-4.3.2
58. Verify with a pre-production test that restart and recovery of linked components neither lose nor
    duplicate transactions, and that components immediately resynchronize on recovery. → GIS-4.3.4,
    GIS-4.3.5
    **Why:** "no lost or duplicated transactions across a restart" is a claim only a test can back, and
    it is the same crash-recovery discipline the gaming-platform and payments skills demand. Write it
    as an automated test the day the transactional path exists.
59. Maintain a disaster-recovery plan with a physically separated recovery site. → GIS-4.4.5 *(GIG3)*

## 12. Change management, SDLC and patching

60. Put all code and binaries under version control, implement only authorized versions, and keep a
    change-management log linking each change to its record. → GIS-9.3.3, GIS-9.3.2, GIS-9.3.4
61. Gate program changes before operational use, require a tested migration with authorized sign-off,
    and enforce segregation of duties in the release. → GIS-9.6.3, GIS-9.3.8, GIS-9.3.9
    **Why:** the developer who wrote a change must not be the sole party that ships it to production.
    Release SoD is what stops both an honest untested push and a malicious one, and it is a control an
    auditor checks in the pipeline configuration, not in a policy document.
62. Keep a rollback / field-issue strategy for every change. → GIS-9.3.6
63. Separate development and test from production with no direct connection between them, and follow a
    documented secure-coding method. → GIS-9.4.2, GIS-9.4.4
64. Monitor for and apply patches to sensitive-data components, testing each on an identically
    configured environment first. → GIS-9.5.2, GIS-9.5.3
65. For in-house software touching patron outcomes, preserve logs / access history / change tracking,
    restrict access, and keep the outcome logic auditable. → GIS-9.6.7, GIS-9.6.8, GIS-9.6.12

## 13. Service-provider integration security

66. Give each provider a dedicated, named, scoped account — restricted to the applications and
    databases it needs, continuously monitored, disabled when idle, uniquely attributable; never a
    shared provider login. → GIS-3.5.11
67. Encrypt and strongly authenticate provider communication and log every provider login event.
    → GIS-6.2.1, GIS-6.2.2
68. Segment provider connectivity so it cannot degrade platform functions or affect patron
    communications: providers on a segment separate from patron segments, gaming disabled outside the
    platform, and no direct packet routing between a provider and production. → GIS-6.2.3, GIS-6.2.4,
    GIS-6.2.5, GIS-6.2.6, GIS-6.2.7, GIS-6.2.8, GIS-6.2.9
    **Why:** the provider integration is the classic third-party breach path — the vendor is trusted,
    lightly monitored, and connected. A platform that acts as an IP router between a provider and
    production, or that lets provider traffic share a patron segment, has made the vendor's security
    posture its own. Terminate provider traffic in a DMZ; never bridge it into production.

## 14. Control-program integrity and clock

69. Verify critical control programs against the regulator-approved baseline by signature, at defined
    triggers (deployment, reboot, periodic, on demand), using a hash of at least 128 bits. → GIS-1.2.2,
    GIS-1.2.3. This is the security depth behind the platform skill's program-integrity rules; for the
    application view defer to `gli-19-platform-engineering`.
70. Timestamp all transactions, configuration changes and significant events from one NTP-synced
    internal clock. → GIS-1.1.1

## 15. PII and financial-transaction data

71. Maintain a PII inventory — what is collected, from where, for what purpose. → GIS-2.7.4
72. Protect payment methods from fraudulent use, collect only the sensitive data the transaction
    strictly needs, and verify the protection of transaction-related PII and payment data. → GIS-2.8.1,
    GIS-2.8.2, GIS-2.8.3. For payment-flow and ledger specifics defer to
    `money-and-payments-best-practices`; card-scheme controls (PCI-DSS) are a separate domain.

## When applying these rules

**Be opinionated about the footguns.** Rules 3, 10, 13, 18, 23, 27, 40, 47, 49, 54, 61 and 68 are
where breaches and findings originate, and every one is far cheaper at design time than as a retrofit.
Separation of duties (3, 61), network and provider segmentation (27, 68), firewall bypass paths (47),
tamper-evident logs (49) and immutable backups (54) are the controls that decide whether a single
compromise stays contained or becomes a breach. Raise them before the topology and the pipeline are
committed.

**Respect the tiers.** GIG1 is baseline — land it everywhere first. GIG2/GIG3 controls (flagged
inline) are enhanced-assurance; apply them where the deployment's risk warrants and the regulator
requires, and do not let a GIG2 aspiration delay a GIG1 gap.

**Thresholds are configuration.** The 5-/15-minute idle timeouts, the three-attempt lockout, the
daily backup cadence, the quarterly rogue-device scan are the framework's reference values and are
frequently overridden by the jurisdiction. Make each a config value; flag a hard-coded one, not a
different number.

**Read the existing security posture first.** A platform of any age already has an IAM model, a
network topology, a secrets store and a logging pipeline. Conform to and extend them; a second,
parallel access-control scheme or a competing secrets store is worse than an imperfect single one.

**Know the boundary with the other skills.** Application gaming correctness and the financial-record
integrity model belong to `gli-19-platform-engineering`; money representation, idempotency and
double-entry to `money-and-payments-best-practices`; instrumentation, log correlation and metric
shape to `observability-best-practices`; ORM-level data access to `sqlalchemy-best-practices`. This
skill is the security-controls layer beneath all of them — it says what must be true of the system's
security, not how each feature is built.
