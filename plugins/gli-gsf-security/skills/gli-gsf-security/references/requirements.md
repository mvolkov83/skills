# GLI-GSF-1 v1.1 — Gaming security framework engineering: requirement reference

GENERATED FILE — do not edit by hand.
Source of truth: `standards/gli-gsf-1/catalog.yaml` (regenerate with `scripts/build_skill.py --standard gli-gsf-1 --profile security`).

Scope: the **163** requirements of GLI-GSF-1 v1.1 whose evidence is code, infrastructure or configuration — the subset a developer can satisfy while writing the system. Procedure, policy, certificate and screenshot evidence is out of scope here and belongs to the operator's governance programme.

> These entries are **paraphrases** written for engineering use. They are not the normative text and carry no certification weight — the standard document itself governs, and a testing laboratory assesses against it, not against this file.

## GIS-1. GPE Critical Control Program Functions

### GIS-1.1 GPE Internal Clock

#### GIS-1.1.1 — NTP-synced internal clock timestamps all events

*must · tier GIG1* · maps to `GIS-1.1.1 [timestamp all transactions/config-changes/significant-events; NTP reference clock]`

The GPE maintains an internal clock reflecting current date/time, used to timestamp all transactions, configuration changes, and significant events, and as a reference clock for reporting, synchronized via NTP or equivalent.

Testable criteria:
- All in-scope GPE systems synchronize to a common time source (NTP/chrony / a cloud time-sync service) (`infrastructure`).
- Transactions, configuration changes, and significant events carry accurate timestamps from that clock.

Typically owned by: infrastructure, logging library.

### GIS-1.2 Critical Control Program Signature Verification

#### GIS-1.2.2 — Verify CCPs identical to regulator-approved via signature procedure

*must · tier GIG1* · maps to `GIS-1.2.2 [regulator-approved procedure; intervals/deployment/reboot/baseline; on-demand; as required]`

The Gaming Enterprise can verify Critical Control Programs as identical to the Regulatory-Body-approved versions via a regulator-approved signature-verification procedure, performed at risk-based intervals (deployment, reboot, periodic baseline scans), on demand, and as required by the Regulatory Body.

Testable criteria:
- A regulator-approved signature-verification procedure verifies CCPs against the approved baseline; cadence defined by risk-based thresholds; on-demand capability exists (`game-session orchestrator`/`game aggregation layer`).
- Third-party RGS (the third-party provider): delegated to provider certification — evidence = provider attestation (`operator governance`).

Typically owned by: game-session orchestrator, game aggregation layer, operator governance.

#### GIS-1.2.3 — Signature verification uses ≥128-bit hash

*must · tier GIG1* · maps to `GIS-1.2.3 [≥128-bit message digest; other methodologies case-by-case]`

The signature-verification procedure employs a cryptographic hash algorithm producing a message digest of at least 128 bits; other test methodologies are reviewed case-by-case.

Testable criteria:
- The hash algorithm used for CCP verification produces ≥128-bit digests (e.g. SHA-256) (`game-session orchestrator`/`infrastructure`).

Typically owned by: game-session orchestrator, game aggregation layer, operator governance.

## GIS-2. Gaming Information Security (GIS)

### GIS-2.3 Access Control Policy

#### GIS-2.3.3 — Formal user registration / de-registration procedure

*must · tier GIG1* · maps to `GIS-2.3.3`

A formal user registration and de-registration procedure grants and revokes GPE access.

Testable criteria:
- Documented joiner/leaver procedure; provisioning/deprovisioning enforced (`identity provider`/`operator governance`).

Typically owned by: operator governance, identity provider.

#### GIS-2.3.4 — Least-privilege allocation of access rights

*must · tier GIG2* · maps to `GIS-2.3.4`

Allocation and use of user access rights/privileges is restricted and controlled by business need and the principle of least privilege.

Testable criteria:
- RBAC enforces least privilege; privileges scoped to business need (`identity provider`).

Typically owned by: operator governance, identity provider.

#### GIS-2.3.5 — Access limited to specifically authorized services/facilities

*must · tier GIG1* · maps to `GIS-2.3.5`

Personnel are only provided access to the services/facilities they are specifically authorized to use.

Testable criteria:
- Access grants map to explicit authorization (`identity provider`/`operator governance`).

Typically owned by: operator governance, identity provider.

### GIS-2.7 Personally Identifiable Information (PII) Privacy Program

#### GIS-2.7.4 — PII inventory (nature/scope/sources/purposes)

*must · tier GIG1* · maps to `GIS-2.7.4`

Procedures determine the nature/scope of all PII collected and processed — types, sources of collection, and purposes of use.

Testable criteria:
- A maintained PII inventory / records-of-processing (`player-account service`/`operator governance`).

Typically owned by: operator governance, player-account service, platform backend.

### GIS-2.8 Securing Financial Transactions within the GPE

#### GIS-2.8.1 — Payment methods protected from fraudulent use

*must · tier GIG1* · maps to `GIS-2.8.1`

Payment methods used for financial transactions in the GPE are protected from fraudulent use.

Testable criteria:
- Fraud controls on payment flows (gateway fraud checks, `risk / responsible-gaming service` scoring) (`payments service`).

Typically owned by: payments service, operator governance.

#### GIS-2.8.2 — Collect only strictly-needed sensitive data

*must · tier GIG1* · maps to `GIS-2.8.2`

Only the sensitive data strictly needed for the financial transaction is collected.

Testable criteria:
- Transaction flows collect the minimum sensitive data; no superfluous capture (`payments service`).

Typically owned by: payments service, operator governance.

#### GIS-2.8.3 — Verify protection of transaction-related sensitive data

*must · tier GIG1* · maps to `GIS-2.8.3 [incl. patron PII + payment-related data]`

Processes verify the protection of sensitive data directly related to each financial transaction, including patron PII and payment-related data.

Testable criteria:
- Verification process confirms encryption/redaction of transaction-related sensitive data (`payments service`/`operator governance`).

Typically owned by: payments service, operator governance.

#### GIS-2.8.4 — Transaction channels encrypted + strongly authenticated

*must · tier GIG1* · maps to `GIS-2.8.4`

Any communication channels within the GPE conveying financial-transaction details are secure, employing encryption and strong authentication to protect against interception.

Testable criteria:
- Transaction channels use TLS + strong (mutual/token) authentication (`payments service`/`infrastructure`).

Typically owned by: payments service, operator governance.

## GIS-3. GPE Operation & Security

### GIS-3.1 Security Procedures

#### GIS-3.1.1 — Monitor Critical System Components + all data transmissions

*must · tier GIG2* · maps to `GIS-3.1.1 [incl. Service Provider components/transmissions; CIA + accountability; anomaly detection]`

The whole GPE — Critical System Components and every data transmission, including any Service Provider services involved — is monitored for confidentiality, integrity, availability, and accountability, and to identify anomalous behavior.

Testable criteria:
- Fleet-wide telemetry (metrics/traces/logs) covers all services incl. gRPC inter-service + provider proxy traffic (`telemetry library`/`infrastructure`).
- Anomaly-detection / alerting on the collected signals (`operator governance`/`infrastructure`).

Typically owned by: infrastructure, operator governance.

#### GIS-3.1.2 — Monitor and adjust resource capacity for availability

*must · tier GIG1* · maps to `GIS-3.1.2`

GPE resource capacity and consumption are monitored and adjusted so availability is maintained.

Testable criteria:
- Capacity metrics + autoscaling/HPA and headroom alerts on the Kubernetes fleet (`infrastructure`).

Typically owned by: infrastructure, operator governance.

#### GIS-3.1.3 — Maintain a GPE performance audit log + reporting

*must · tier GIG2* · maps to `GIS-3.1.3`

An audit log of GPE performance is maintained, including a function to compile performance reports.

Testable criteria:
- Retained performance metrics with report/export capability (`telemetry library`/`infrastructure`).

Typically owned by: infrastructure, operator governance.

#### GIS-3.1.4 — Detect/prevent/mitigate active & passive attacks

*must · tier GIG1* · maps to `GIS-3.1.4`

The GPE is monitored to detect, prevent, mitigate, and respond to common active and passive technical attacks and compromises.

Testable criteria:
- WAF/IDS/IPS + alerting on attack signatures; documented response path (`infrastructure`/`operator governance`).

Typically owned by: infrastructure, operator governance.

#### GIS-3.1.6 — Centrally monitor/manage user activities & adverse events

*must · tier GIG2* · maps to `GIS-3.1.6 [user activities, exceptions, malfunctions, adverse events]`

Procedures are established to centrally monitor, manage, and respond to user activities, exceptions, malfunctions, and adverse events.

Testable criteria:
- Centralized log/event aggregation (a central log store + alerting) with response workflow (`infrastructure`/`operator governance`).

Typically owned by: infrastructure, operator governance.

### GIS-3.3 GIS Incident Management

#### GIS-3.3.3 — Technical & procedural detection/escalation mechanisms

*must · tier GIG1* · maps to `GIS-3.3.3 [a. continuous monitoring; b. detect+log; c. auto/manual escalation on threshold]`

Appropriate technical and procedural mechanisms continuously monitor the GPE for incidents, detect and log all suspected/actual incidents, and automatically or manually escalate those meeting/exceeding defined thresholds.

Testable criteria:
- Monitoring + alerting wired to escalation on threshold breach (`infrastructure`/`operator governance`).

Typically owned by: operator governance.

### GIS-3.5 Logical Access Control

#### GIS-3.5.1 — Logically secured by regulator-allowed credentials

*must · tier GIG1* · maps to `GIS-3.5.1 [passwords, MFA, digital certificates, PINs, biometrics, etc.]`

The GPE is logically secured against unauthorized access using authentication credentials allowed by the Regulatory Body (passwords, MFA, digital certificates, PINs, biometrics, and other methods).

Testable criteria:
- Enforced auth on all GPE entry points; credential types match regulator-allowed set (`identity provider`).

Typically owned by: identity provider, operator governance.

#### GIS-3.5.2 — Individual credential per account, formal provisioning

*must · tier GIG1* · maps to `GIS-3.5.2`

Each user account has its own individual authentication credential, provisioned through a controlled formal process.

Testable criteria:
- No shared credentials; documented provisioning workflow (`identity provider`/`operator governance`).

Typically owned by: identity provider, operator governance.

#### GIS-3.5.3 — Least-privilege access by role

*must · tier GIG1* · maps to `GIS-3.5.3`

Users only have access to the functionality and features appropriate for their role and responsibilities within the GPE.

Testable criteria:
- RBAC roles scoped to job function; enforced authorization checks (`identity provider`/`platform backend`).

Typically owned by: identity provider, operator governance.

#### GIS-3.5.4 — Critical-parameter changes gated + audit-logged

*must · tier GIG1* · maps to `GIS-3.5.4 [audit log: a. date/time; b. params changed; c. reason+initial/final values; d. user account ID]`

Critical system parameters (OS/DB/network/application policies — audit settings, password-complexity, security levels, manual DB updates, etc.) cannot be modified without an authorized secure process, and each change is audit-logged with date/time, parameters changed, reason + initial/final values, and the authorizing/performing user account ID.

Testable criteria:
- Change-controlled config with immutable audit trail carrying all four fields (`operator governance`/`infrastructure`).

Typically owned by: identity provider, operator governance.

#### GIS-3.5.6 — Credential-change records maintained

*must · tier GIG1* · maps to `GIS-3.5.6`

Records for authentication credentials are maintained — manually or by systems that automatically record credential changes and force credential changes.

Testable criteria:
- Credential change history retained; forced-change capability (`identity provider`).

Typically owned by: identity provider, operator governance.

#### GIS-3.5.7 — Stored credentials encrypted or hashed

*must · tier GIG1* · maps to `GIS-3.5.7`

Any authentication credentials stored on the system are encrypted or hashed using authorized cryptographic algorithms.

Testable criteria:
- Passwords hashed with a strong KDF; secrets encrypted at rest (`identity provider`).

Typically owned by: identity provider, operator governance.

#### GIS-3.5.8 — Credential-reset fallback ≥ primary strength (MFA)

*must · tier GIG2* · maps to `GIS-3.5.8 [MFA employed for reset]`

A fallback method for resetting credentials (e.g., forgotten passwords) is at least as strong as the primary method, and an MFA process is employed for these purposes.

Testable criteria:
- Password-reset flow enforces MFA (`identity provider`).

Typically owned by: identity provider, operator governance.

#### GIS-3.5.9 — Timely deactivation of lost/terminated credentials

*must · tier GIG1* · maps to `GIS-3.5.9`

Lost/compromised credentials and credentials of terminated users are deactivated, secured, or destroyed as soon as reasonably possible.

Testable criteria:
- Deprovisioning workflow triggered on termination/compromise (`identity provider`/`operator governance`).

Typically owned by: identity provider, operator governance.

#### GIS-3.5.10 — Multiple access levels + separation of duties

*must · tier GIG1* · maps to `GIS-3.5.10 [a. SoD in account admin; b. limit critical-param permissions; c. credential parameter enforcement — min length, expiry]`

The system has multiple security access levels controlling view/change/delete of critical files/directories, with procedures to assign/review/modify/remove rights — including SoD in account administration, limiting who can adjust critical parameters, and enforcing credential parameters (minimum length, expiration intervals).

Testable criteria:
- Tiered RBAC + SoD; credential-policy enforcement (length/expiry) (`identity provider`/`operator governance`).

Typically owned by: identity provider, operator governance.

#### GIS-3.5.11 — Service-Provider dedicated accounts (restricted, monitored, unique)

*must · tier GIG1* · maps to `GIS-3.5.11 [a. restricted to necessary app/db; b. continuously monitored; c. disabled when not in use; d. uniquely identified — generic/shared prohibited]`

Service Providers may access the GPE via dedicated user accounts (with regulator/enterprise permission) that are restricted to the necessary application(s)/database(s), continuously monitored, disabled when not in use/no longer required, and uniquely identified to the individual — generic and shared Service-Provider accounts are prohibited.

Testable criteria:
- Named, scoped, monitored, time-bound provider accounts; no shared provider logins (`identity provider`/`operator governance`).

Typically owned by: identity provider, operator governance.

#### GIS-3.5.12 — Suspect-account flagging + 3-attempt lockout

*must · tier GIG1* · maps to `GIS-3.5.12 [a. admin notification + lockout after max 3 attempts; b. flag stolen-credential accounts; c. invalidate + transfer to new account]`

Procedures identify and flag suspect user accounts to prevent unauthorized use — administrator notification and user lockout after a maximum of three incorrect authentication attempts, flagging of accounts with potentially stolen credentials, and invalidating accounts while transferring critical stored account information to a new account.

Testable criteria:
- Lockout ≤ 3 failed attempts + admin alert; suspect-account handling (`identity provider`).

Typically owned by: identity provider, operator governance.

#### GIS-3.5.13 — Logical access-attempt audit log

*must · tier GIG1* · maps to `GIS-3.5.13 [a. date/time; b. user account ID; c. IP; d. success/fail; e. duration if successful]`

All logical access attempts to system applications or operating systems are recorded in an audit log with date/time, user account ID, source IP, success/failure, and — if successful — access duration.

Testable criteria:
- Auth events logged with all five fields, retained (`identity provider`/`logging library`).

Typically owned by: identity provider, operator governance.

#### GIS-3.5.14 — Utility programs restricted and controlled

*must · tier GIG1* · maps to `GIS-3.5.14`

Use of utility programs that can override application or operating-system controls is restricted and tightly controlled.

Testable criteria:
- Privileged tooling access restricted + logged (`infrastructure`/`operator governance`).

Typically owned by: identity provider, operator governance.

#### GIS-3.5.15 — Out-of-scope activity audit log (voids/overrides/corrections)

*must · tier GIG1* · maps to `GIS-3.5.15 [audit log: a. date/time; b. components affected; c. reason+initial/final values; d. user account ID]`

System voids, overrides, corrections, or other user-intervention activities outside the normal scope of operation are recorded in an audit log with date/time, affected components, reason + initial/final values, and the authorizing/performing user account ID.

Testable criteria:
- Manual-intervention/override events logged with all four fields (`platform backend`/`operator governance`).

Typically owned by: identity provider, operator governance.

#### GIS-3.5.16 — Per-account information maintained and backed up

*must · tier GIG1* · maps to `GIS-3.5.16 [a. account ID; b. name+title; c. functions; d. created; e. last access+IP; f. last password change; g. disabled/deactivated; h. access rights/group; i. current+previous statuses]`

For each user account, the GPE maintains and backs up: account ID; individual name and title/position; full list/description of executable functions; created date/time; last-access date/time + IP; last password-change date/time; disabled/deactivated date/time; access-rights/group membership (if applicable); and current + previous account statuses.

Testable criteria:
- Account data model captures all nine fields; backed up (`identity provider`).

Typically owned by: identity provider, operator governance.

#### GIS-3.5.17 — Timely account change/block/deactivate/remove

*must · tier GIG1* · maps to `GIS-3.5.17`

A process exists for authorized personnel to change, block, deactivate, or remove a user account in a timely manner upon unauthorized use, or upon suspension/termination/change of role or responsibility.

Testable criteria:
- Documented + implemented timely account-change process (`identity provider`/`operator governance`).

Typically owned by: identity provider, operator governance.

#### GIS-3.5.18 — Restrict access to inactive/closed accounts

*must · tier GIG1* · maps to `GIS-3.5.18`

Only authorized personnel have access to inactive or closed user accounts.

Testable criteria:
- Access to inactive/closed account records is restricted (`identity provider`/`operator governance`).

Typically owned by: identity provider, operator governance.

### GIS-3.6 User Authentication and Authorization

#### GIS-3.6.1 — On-demand + periodic identity verification

*must · tier GIG1* · maps to `GIS-3.6.1`

A secure, controlled mechanism verifies that the Critical System Component is accessed by authorized personnel on demand and on a regular basis as required by the Regulatory Body.

Testable criteria:
- Token/session validation on each request + periodic re-verification (`identity provider`).

Typically owned by: identity provider, web framework.

#### GIS-3.6.2 — Terminate session on excessive failed authorization

*must · tier GIG1* · maps to `GIS-3.6.2`

Active user sessions are terminated if user authorization exceeds a configurable number of failed attempts.

Testable criteria:
- Configurable failure threshold → session termination (`identity provider`/`web framework`).

Typically owned by: identity provider, web framework.

#### GIS-3.6.3 — Ephemeral authorization info (not stored on CSC)

*must · tier GIG2* · maps to `GIS-3.6.3`

Authorization information communicated for identification purposes is obtained at request time from the system and not stored on the Critical System Component.

Testable criteria:
- Authorization derived per-request; not persisted on the component (`identity provider`).

Typically owned by: identity provider, web framework.

#### GIS-3.6.4 — Random, in-memory session auth removed at session end

*must · tier GIG2* · maps to `GIS-3.6.4`

Where sessions are tracked for authorization, the authorization information is always created randomly, held in memory, and removed after the session ends.

Testable criteria:
- Random session tokens; in-memory/short-TTL store; cleared on logout/expiry (`identity provider`/`web framework`).

Typically owned by: identity provider, web framework.

#### GIS-3.6.5 — Connection-time restrictions for high-risk access

*must · tier GIG1* · maps to `GIS-3.6.5`

Connection-time restrictions (such as, but not limited to, session timeouts) provide additional security for high-risk applications such as remote access.

Testable criteria:
- Tighter session/connection limits on high-risk/remote paths (`identity provider`/`operator governance`).

Typically owned by: identity provider, web framework.

#### GIS-3.6.6 — Inactivity timeout (5 min mobile / 15 min other)

*must · tier GIG1* · maps to `GIS-3.6.6 [a. 5 min portable devices; b. 15 min all other]`

User sessions automatically time out after a management-defined inactivity period — unless the Regulatory Body specifies otherwise, not exceeding 5 minutes for portable/mobile devices and 15 minutes for all other systems and devices.

Testable criteria:
- Idle-timeout ≤ 5 min (mobile) / ≤ 15 min (other) enforced (`identity provider`/`web framework`).

Typically owned by: identity provider, web framework.

### GIS-3.7 Server Programming

#### GIS-3.7.1 — Prevent unauthorized server-side programming

*must · tier GIG1* · maps to `GIS-3.7.1 [admin maintenance/troubleshooting with sufficient access rights acceptable]`

The GPE is sufficiently secure to prevent unauthorized user-initiated programming on the server that could modify the database, while allowing network/system administrators to perform authorized infrastructure maintenance or application troubleshooting with sufficient access rights.

Testable criteria:
- No unauthorized code/DB-mutation paths on hosts; admin access gated + logged (`infrastructure`/`operator governance`).

Typically owned by: infrastructure, operator governance.

### GIS-3.8 Cloud and Virtualized Environments

#### GIS-3.8.1 — GIS controls apply to cloud/virtualized environments

*must · tier GIG2* · maps to `GIS-3.8.1 [validate cloud/virtualization infrastructure + server-instance usage]`

If sensitive data is stored, processed, or transmitted in a cloud or virtualized environment, appropriate GIS controls apply — typically validating both the cloud/virtualization infrastructure and the usage of server instances within it.

Testable criteria:
- GIS controls mapped onto the the cloud Kubernetes environment; infrastructure + instance usage validated (`infrastructure`/`operator governance`).

Typically owned by: infrastructure, operator governance.

#### GIS-3.8.2 — Redundant instances on separate hypervisors

*must · tier GIG2* · maps to `GIS-3.8.2`

Redundant server instances in a cloud/virtualized environment are deployed on separate hypervisors to prevent a single point of failure.

Testable criteria:
- Replicas spread across AZs / anti-affinity so no shared hypervisor SPOF (`infrastructure`).

Typically owned by: infrastructure, operator governance.

### GIS-3.9 Optional Use of an Electronic Document Retention System (EDRS)

#### GIS-3.9.1 — Retain original + all subsequent versions

*must · tier GIG1* · maps to `GIS-3.9.1`

The EDRS is configured to maintain the original version plus all subsequent versions reflecting every change to reports/audit logs stored in an alterable format.

Testable criteria:
- Full version history retained for alterable reports/logs (`logging library`/`operator governance`).

Typically owned by: operator governance, logging library.

#### GIS-3.9.2 — Unique signature per audit-log version

*must · tier GIG1* · maps to `GIS-3.9.2`

The EDRS maintains a unique signature for each version of the audit log, including the original.

Testable criteria:
- Per-version cryptographic signature/hash (`logging library`).

Typically owned by: operator governance, logging library.

#### GIS-3.9.3 — Change audit log (who/when/what)

*must · tier GIG1* · maps to `GIS-3.9.3 [user account ID, date/time, what changed]`

The EDRS retains an audit log of changes to all reports — including the user account ID that performed the change, the date/time it occurred, and what was changed.

Testable criteria:
- Change log captures actor + timestamp + delta (`logging library`/`operator governance`).

Typically owned by: operator governance, logging library.

#### GIS-3.9.4 — Complete indexing for audit-log retrieval

*must · tier GIG1* · maps to `GIS-3.9.4 [a. generated date/time; b. generating component; c. title/description; d. generating user account ID; e. other identifying info]`

The EDRS provides complete indexing to easily locate/identify audit logs, including at least generated date/time, generating Critical System Component, title/description, generating user account ID, and any other useful identifying information (user-input allowed).

Testable criteria:
- Searchable index over the five enumerated fields (`logging library`).

Typically owned by: operator governance, logging library.

#### GIS-3.9.5 — Restrict modify/add + admin-activity audit log

*must · tier GIG1* · maps to `GIS-3.9.5 [a. limit modify/add via logical security; b. audit log of admin activity]`

The EDRS limits access to modify/add reports or audit logs to specific user accounts through logical security, and provides an audit log of all administrative user-account activity.

Testable criteria:
- Write access restricted by account; admin-activity logged (`operator governance`/`logging library`).

Typically owned by: operator governance, logging library.

#### GIS-3.9.7 — Redundancy + backup to prevent log loss

*must · tier GIG1* · maps to `GIS-3.9.7`

The EDRS is equipped to prevent disruption of log availability and loss of data through hardware/software redundancy best practices and backup processes.

Testable criteria:
- Redundant storage + backup for the audit-log store (`infrastructure`/`logging library`).

Typically owned by: operator governance, logging library.

## GIS-4. Data Integrity

### GIS-4.1 Sensitive Data Management

#### GIS-4.1.3 — Validate/reject corrupt sensitive data

*must · tier GIG2* · maps to `GIS-4.1.3`

Appropriate handling methods validate corrupt input and reject corrupt sensitive data.

Testable criteria:
- Input validation + integrity checks reject corrupt data at boundaries (`platform backend`/`data-access layer`).

Typically owned by: data-access layer, infrastructure, operator governance.

#### GIS-4.1.4 — Encrypt (or SoD + access-monitor) data files/directories

*must · tier GIG2* · maps to `GIS-4.1.4 [if not encrypted: restrict viewing + SoD + monitor/record access]`

Encryption or equivalent security protects files/directories containing sensitive data; if encryption is not used, users are restricted from viewing contents, with at minimum segregation of duties and monitoring/recording of all access to those files/directories.

Testable criteria:
- Encryption-at-rest on sensitive files/DB (the managed database/KMS, block/object-store encryption); or documented SoD + access logging (`infrastructure`/`data-access layer`).

Typically owned by: data-access layer, infrastructure, operator governance.

#### GIS-4.1.6 — Logical protection against tampering/unauthorized access

*must · tier GIG1* · maps to `GIS-4.1.6 [both external and internal]`

The GPE provides a logical means for securing and protecting sensitive data against alteration, tampering, or unauthorized access — both external and internal.

Testable criteria:
- Access controls + integrity protection covering internal actors (`data-access layer`/`identity provider`).

Typically owned by: data-access layer, infrastructure, operator governance.

#### GIS-4.1.8 — Errors must not auto-clear sensitive data

*must · tier GIG1* · maps to `GIS-4.1.8`

Critical System Components must not have a mechanism whereby an error will cause sensitive data to automatically clear.

Testable criteria:
- Error paths do not wipe sensitive data (durable writes; no destructive error handling) (`wallet / ledger service`/`data-access layer`).

Typically owned by: data-access layer, infrastructure, operator governance.

#### GIS-4.1.9 — Memory-held data transferred to DB before removal

*must · tier GIG1* · maps to `GIS-4.1.9`

Any Critical System Component holding sensitive data in memory must not allow removal of the information unless it has first transferred that information to the associated database or other secured component(s) of the GPE.

Testable criteria:
- In-memory sensitive state persisted before eviction (durable-write-before-ack) (`wallet / ledger service`/`platform backend`).

Typically owned by: data-access layer, infrastructure, operator governance.

#### GIS-4.1.10 — Protect CIA+accountability of data at-rest; limit workstations

*must · tier GIG2* · maps to `GIS-4.1.10 [servers, critical applications, databases; limit access workstations]`

The confidentiality, integrity, availability, and accountability of sensitive data at-rest on servers, critical applications, and databases is protected, including limiting the number of workstations from which it can be accessed.

Testable criteria:
- At-rest protection + restricted access endpoints for sensitive-data stores (`infrastructure`/`data-access layer`).

Typically owned by: data-access layer, infrastructure, operator governance.

#### GIS-4.1.11 — Encrypt data in-use / on portable / at-rest on workstations

*must · tier GIG2* · maps to `GIS-4.1.11 [in use; portable systems — laptops/USB; at-rest on workstations]`

Encryption protects the CIA+accountability of sensitive data when in use, when stored on portable computer systems (laptops, USB devices, etc.), and when held at-rest on workstations.

Testable criteria:
- Endpoint/portable-media encryption policy enforced (`operator governance`/`infrastructure`).

Typically owned by: data-access layer, infrastructure, operator governance.

#### GIS-4.1.12 — Message authentication for non-hidden authenticated data

*must · tier GIG2* · maps to `GIS-4.1.12`

Sensitive data not required to be hidden but requiring authentication uses some form of message-authentication technique.

Testable criteria:
- MAC/signature on integrity-critical non-encrypted data (`wallet / ledger service`/`platform backend`).

Typically owned by: data-access layer, infrastructure, operator governance.

#### GIS-4.1.13 — Trusted-authority certificates for authentication

*must · tier GIG1* · maps to `GIS-4.1.13 [owner, issuer, valid dates, unique serial/ID verifiable]`

Authentication uses a security certificate from a trusted authority containing owner, issuer, valid dates, and a unique serial number/identifier usable to verify the certificate's contents.

Testable criteria:
- Certificates issued by trusted CA with verifiable fields; validation enforced (`infrastructure`/`identity provider`).

Typically owned by: data-access layer, infrastructure, operator governance.

#### GIS-4.1.14 — Segment production DBs from patron-interface servers

*must · tier GIG1* · maps to `GIS-4.1.14`

Production databases containing sensitive data reside on networks separated from the servers hosting any patron interfaces.

Testable criteria:
- DB tier in a segmented network/subnet isolated from public-facing/patron servers (`infrastructure`).

Typically owned by: data-access layer, infrastructure, operator governance.

#### GIS-4.1.15 — Data persists across power loss

*must · tier GIG1* · maps to `GIS-4.1.15`

Sensitive data is maintained at all times regardless of whether the server is being supplied with power.

Testable criteria:
- Durable (non-volatile) storage; data survives power loss (`infrastructure`/`data-access layer`).

Typically owned by: data-access layer, infrastructure, operator governance.

#### GIS-4.1.16 — Data leakage prevention (DLP)

*must · tier GIG2* · maps to `GIS-4.1.16 [systems, networks, devices that process/store/transmit sensitive data]`

Sensitive-data leakage-prevention measures are applied to systems, networks, and any other devices that process, store, or transmit sensitive data.

Testable criteria:
- DLP controls on sensitive-data paths/endpoints (`infrastructure`/`operator governance`).

Typically owned by: data-access layer, infrastructure, operator governance.

#### GIS-4.1.17 — Data survives parts/module replacement

*must · tier GIG1* · maps to `GIS-4.1.17`

Sensitive data is stored so as to prevent loss when replacing parts or modules during normal maintenance.

Testable criteria:
- Redundant/replicated storage tolerating node/disk replacement (`infrastructure`).

Typically owned by: data-access layer, infrastructure, operator governance.

#### GIS-4.1.20 — Sensitive-data masking per policy

*must · tier GIG1* · maps to `GIS-4.1.20`

Sensitive-data masking is applied per the Gaming Enterprise's access-control and related policies, based on business/security requirements and in compliance with applicable regulations/standards.

Testable criteria:
- Masking/redaction in UI/logs/exports per policy (`platform backend`/`player-account service`).

Typically owned by: data-access layer, infrastructure, operator governance.

### GIS-4.2 Backup Process Implementation

#### GIS-4.2.1 — Backups at least daily

*must · tier GIG1* · maps to `GIS-4.2.1 [methods reviewed case-by-case]`

Backup process implementation occurs at least daily (or as the Regulatory Body specifies); all methods are reviewed case-by-case.

Testable criteria:
- Scheduled ≥daily backups across sensitive-data stores (`infrastructure`).

Typically owned by: infrastructure, operator governance.

#### GIS-4.2.2 — Immutable backups of data/apps/databases

*must · tier GIG1* · maps to `GIS-4.2.2 [immutability safeguards preventing alterations/deletions]`

Sensitive data, critical applications, and databases are backed up with immutability safeguards preventing alterations or deletions, ensuring GPE integrity.

Testable criteria:
- Immutable/WORM backups (object storage Object Lock / vault-lock) (`infrastructure`).

Typically owned by: infrastructure, operator governance.

#### GIS-4.2.3 — Mirrored/redundant copies with restore support

*must · tier GIG1* · maps to `GIS-4.2.3`

Mirrored or redundant copies of sensitive data are kept on the GPE with open support for backups and restoration.

Testable criteria:
- Redundant copies + tested restore capability (`infrastructure`).

Typically owned by: infrastructure, operator governance.

#### GIS-4.2.4 — Non-volatile backup medium

*must · tier GIG1* · maps to `GIS-4.2.4 [or equivalent architectural implementation]`

The backup is contained on a non-volatile physical medium or an equivalent architectural implementation.

Testable criteria:
- Backups on durable storage (object storage/Glacier/snapshot) (`infrastructure`).

Typically owned by: infrastructure, operator governance.

#### GIS-4.2.5 — Disk-failure integrity for HDD backup storage

*must · tier GIG1* · maps to `GIS-4.2.5`

If HDDs are used as backup storage, data integrity is assured in the event of a disk failure.

Testable criteria:
- Redundant/checksummed backup storage tolerating disk failure (`infrastructure`).

Typically owned by: infrastructure, operator governance.

#### GIS-4.2.6 — Immediate transfer to physically-separate location

*must · tier GIG1* · maps to `GIS-4.2.6 [temporary and permanent storage]`

Upon backup completion, backup storage is immediately transferred to a location physically separate from the servers/sensitive data being backed up (for temporary and permanent storage).

Testable criteria:
- Backups replicated to a separate region/AZ off the production location (`infrastructure`).

Typically owned by: infrastructure, operator governance.

#### GIS-4.2.7 — Secured backup location preventing data loss

*must · tier GIG1* · maps to `GIS-4.2.7`

The backup storage location is secured against unauthorized access and provides adequate protection against permanent loss of sensitive data.

Testable criteria:
- Access-controlled + durable backup location (`infrastructure`).

Typically owned by: infrastructure, operator governance.

#### GIS-4.2.8 — Optional cross-cloud/region backup copy

*may · tier GIG2* · maps to `GIS-4.2.8`

If the backup is stored in a cloud platform, another copy may be stored in a different cloud platform or region.

Testable criteria:
- (Optional) cross-cloud/region backup copy where adopted (`infrastructure`).

Typically owned by: infrastructure, operator governance.

#### GIS-4.2.9 — Backup security parity with GPE

*must · tier GIG1* · maps to `GIS-4.2.9`

Backup data files and data-recovery components are managed with at least the same level of security and access controls as the GPE.

Testable criteria:
- Backup IAM/encryption ≥ production controls (`infrastructure`/`operator governance`).

Typically owned by: infrastructure, operator governance.

### GIS-4.3 System Failure and Recovery

#### GIS-4.3.1 — Redundancy/modularity with no data loss on single failure

*must · tier GIG1* · maps to `GIS-4.3.1 [GPE functions + auditing continue with no loss/corruption]`

The GPE has sufficient redundancy and modularity so that if any single Critical System Component (or part) fails, GPE functions and the auditing of those functions continue with no loss or corruption of sensitive data.

Testable criteria:
- Multi-AZ/replicated topology; single-node failure survivable without data loss (`infrastructure`).

Typically owned by: infrastructure, wallet / ledger service, operator governance.

#### GIS-4.3.2 — Audit-log significant component unavailability

*must · tier GIG1* · maps to `GIS-4.3.2 [log: a. component; b. became-unavailable time; c. reason; d. became-available time]`

Significant periods of Critical System Component unavailability (operations halted for all users and/or transactions not completable) are audit-logged with component identification, unavailable-since time, reason/description, and available-again time.

Testable criteria:
- Outage events recorded with the four fields (`infrastructure`/`telemetry library`).

Typically owned by: infrastructure, wallet / ledger service, operator governance.

#### GIS-4.3.4 — No lost/duplicated transactions on restart/recovery

*must · tier GIG1* · maps to `GIS-4.3.4 [transactions not lost or duplicated on component recovery]`

Gaming operations between Critical System Components are not adversely affected by restart/recovery of either component — transactions are not lost or duplicated because of recovery of one component or the other.

Testable criteria:
- Idempotent/exactly-once transaction handling across service boundaries; recovery neither loses nor double-applies (`wallet / ledger service`/`platform backend`).

Typically owned by: infrastructure, wallet / ledger service, operator governance.

#### GIS-4.3.5 — Immediate synchronization on restart/recovery

*must · tier GIG1* · maps to `GIS-4.3.5 [transactions, sensitive data, configurations synchronized]`

Upon restart/recovery, Critical System Components immediately synchronize the status of all transactions, sensitive data, and configurations with one another.

Testable criteria:
- Reconciliation/sync on recovery brings components to a consistent state (`wallet / ledger service`/`infrastructure`).

Typically owned by: infrastructure, wallet / ledger service, operator governance.

### GIS-4.4 Business Continuity and Disaster Recovery Plan

#### GIS-4.4.5 — Physically-separated recovery site

*must · tier GIG3* · maps to `GIS-4.4.5 [distance vs environmental threats, power failures, data-replication difficulty, reasonable-time access]`

The BC/DR plan addresses establishing a recovery site physically separated from production; the inter-site distance balances environmental threats/hazards, power failures, and other disruptions against data-replication difficulty and reasonable-time access. *(The framework's only GIG3-tier control.)*

Testable criteria:
- Documented recovery site in a separate region; distance rationale recorded (`infrastructure`/`operator governance`).

Typically owned by: operator governance, infrastructure.

## GIS-5. Communications

### GIS-5.1 Communication Protocol

#### GIS-5.1.1 — Documented secure communication protocol per component

*must · tier GIG1* · maps to `GIS-5.1.1`

Each Critical System Component of the GPE functions as indicated by a documented secure communication protocol.

Testable criteria:
- Documented, enforced secure protocols (TLS/mTLS/gRPC) per component (`infrastructure`/`RPC layer`).

Typically owned by: infrastructure, RPC layer.

#### GIS-5.1.2 — Error-detection/recovery + anti-tamper techniques

*must · tier GIG1* · maps to `GIS-5.1.2 [prevent intrusion, interference, eavesdropping, alteration, tampering; alternatives regulator-approved]`

All protocols use communication techniques with proper error detection/recovery designed to prevent intrusion, interference, eavesdropping, unauthorized alterations, and tampering; alternative implementations are reviewed case-by-case and regulator-approved.

Testable criteria:
- Protocols provide integrity/error-recovery; non-standard choices documented + approved (`infrastructure`/`RPC layer`).

Typically owned by: infrastructure, RPC layer.

#### GIS-5.1.3 — Encrypt + authenticate critical sensitive-data comms

*must · tier GIG1* · maps to `GIS-5.1.3`

All critical communications of sensitive data employ encryption and authentication for integrity.

Testable criteria:
- Sensitive-data channels use encryption + authentication (TLS/mTLS) (`infrastructure`/`RPC layer`).

Typically owned by: infrastructure, RPC layer.

#### GIS-5.1.4 — Enrollment/authentication-gated network communications

*must · tier GIG1* · maps to `GIS-5.1.4 [no unauthorized comms to components/access points]`

Communications on the secure network are only possible between authorized Critical System Components enrolled and authenticated as valid; no unauthorized communications to components/access points are allowed.

Testable criteria:
- mTLS/network-policy allowlisting so only enrolled components communicate (`infrastructure`).

Typically owned by: infrastructure, RPC layer.

#### GIS-5.1.5 — Hardened against malformed-message attacks

*must · tier GIG1* · maps to `GIS-5.1.5`

Communications are hardened to be immune to all possible malformed-message attacks.

Testable criteria:
- Message parsing validates/rejects malformed input; fuzz-resilience (`RPC layer`/`platform backend`).

Typically owned by: infrastructure, RPC layer.

#### GIS-5.1.6 — Comms failure must not affect data integrity

*must · tier GIG1* · maps to `GIS-5.1.6`

Failure of communications must not affect the integrity of sensitive data.

Testable criteria:
- Transactional/atomic handling so dropped comms leave data consistent (`wallet / ledger service`/`RPC layer`).

Typically owned by: infrastructure, RPC layer.

#### GIS-5.1.7 — Post-restart re-auth only after self-tests

*must · tier GIG1* · maps to `GIS-5.1.7`

After a system interruption/shutdown, communication with all Critical System Components necessary for GPE operation is not established/authenticated until the program-resumption routine (including any self-tests) completes successfully.

Testable criteria:
- Readiness/health-check gating before a component accepts/initiates comms (`infrastructure`).

Typically owned by: infrastructure, RPC layer.

### GIS-5.2 Communications Over Internet/Public Networks

#### GIS-5.2.1 — Protect CSC comms over internet/public networks

*must · tier GIG1* · maps to `GIS-5.2.1 [encrypt packets or secure protocol; CIA + accountability]`

Communications between Critical System Components over internet/public networks are protected from fraudulent activity, contract dispute, and unauthorized disclosure/modification by encrypting data packets or using a secure communications protocol ensuring CIA+accountability of the transmission.

Testable criteria:
- TLS/secure protocol on all public-network CSC traffic (`infrastructure`).

Typically owned by: infrastructure.

#### GIS-5.2.2 — Always-encrypt sensitive data; anti-replay/misrouting

*must · tier GIG1* · maps to `GIS-5.2.2 [safeguard from incomplete transmission, misrouting, modification, disclosure, duplication, replay]`

Sensitive data is always encrypted over the internet/public network and safeguarded from incomplete transmissions, misrouting, unauthorized message modification, disclosure, duplication, or replay.

Testable criteria:
- Encryption in transit + integrity/anti-replay controls on sensitive-data flows (`infrastructure`/`RPC layer`).

Typically owned by: infrastructure.

### GIS-5.5 Network Communication Equipment (NCE)

#### GIS-5.5.2 — Defined install plan + NCE inventory

*must · tier GIG1* · maps to `GIS-5.5.2 [records of all installed NCE maintained]`

NCE is installed per a defined plan and records of all installed NCE are maintained.

Testable criteria:
- Network defined as IaC (Terraform) = the install plan + inventory of record (`infrastructure`).

Typically owned by: infrastructure, operator governance.

#### GIS-5.5.5 — GPE comms via NCE logically secured

*must · tier GIG1* · maps to `GIS-5.5.5`

GPE communications via NCE are logically secured from unauthorized access.

Testable criteria:
- Security groups/NACLs + least-privilege routing on the virtual network (`infrastructure`).

Typically owned by: infrastructure, operator governance.

#### GIS-5.5.6 — Audit-log-full handling on limited-storage NCE

*must · tier GIG1* · maps to `GIS-5.5.6 [disable all comms or offload logs to dedicated audit log server]`

NCE with limited onboard storage must, if the audit log becomes full, either disable all communication or offload audit logs to a dedicated audit-log server.

Testable criteria:
- Network/flow logs streamed to central store (the central metrics/log store) so no on-device log-full loss (`infrastructure`).

Typically owned by: infrastructure, operator governance.

### GIS-5.6 Intrusion Detection System/Intrusion Prevention System (IDS/IPS)

#### GIS-5.6.1 — IDS/IPS detect/prevent attack set

*must · tier GIG1* · maps to `GIS-5.6.1 [a. DDoS; b. shellcode; c. ARP spoofing; d. MITM — sever comms on detection]`

An IDS/IPS spanning internal and external communications detects or prevents DDoS attacks, network-traversing shellcode, ARP spoofing, and other man-in-the-middle indicators — severing communications immediately on detection.

Testable criteria:
- IDS/IPS + DDoS protection deployed (GuardDuty/Network Firewall/Shield) with active response (`infrastructure`).

Typically owned by: infrastructure.

#### GIS-5.6.2 — ≥ quarterly rogue-device scan

*must · tier GIG2* · maps to `GIS-5.6.2 [at least quarterly or regulator-specified]`

The IDS/IPS scans the network for unauthorized/rogue access points or devices at least quarterly (or as the Regulatory Body specifies).

Testable criteria:
- Scheduled ≥quarterly rogue-device/asset scan of the network (`infrastructure`).

Typically owned by: infrastructure.

#### GIS-5.6.3 — Auto-disable rogue devices

*must · tier GIG2* · maps to `GIS-5.6.3`

The IDS/IPS automatically disables any unauthorized/rogue devices connected to the GPE.

Testable criteria:
- Automated quarantine/isolation of rogue devices on detection (`infrastructure`).

Typically owned by: infrastructure.

#### GIS-5.6.4 — Reconcilable device access audit log

*must · tier GIG1* · maps to `GIS-5.6.4 [a. time/date, name, hardware identifier of all requesting devices; b. reconcilable with all GPE network devices]`

The IDS/IPS maintains an access audit log containing time/date, name, and hardware identifier of all devices requesting network access, reconcilable with all other GPE networking devices.

Testable criteria:
- Device-access logs with the required fields, reconcilable against the network inventory (`infrastructure`).

Typically owned by: infrastructure.

### GIS-5.7 Network Security Management

#### GIS-5.7.2 — Logical network separation

*must · tier GIG1* · maps to `GIS-5.7.2 [no traffic on a link its hosts cannot service]`

Networks are logically separated so there is no network traffic on a link that cannot be serviced by hosts on that link.

Testable criteria:
- VPC/subnet segmentation + security groups scope traffic to serviceable hosts (`infrastructure`).

Typically owned by: infrastructure, operator governance.

#### GIS-5.7.3 — Authenticated + encrypted network management

*must · tier GIG1* · maps to `GIS-5.7.3`

All network-management functions authenticate all users and encrypt all network-management communications.

Testable criteria:
- Management-plane access authenticated + encrypted (no plaintext mgmt) (`infrastructure`).

Typically owned by: infrastructure, operator governance.

#### GIS-5.7.4 — No single-point denial of service

*must · tier GIG1* · maps to `GIS-5.7.4`

The failure of any single item must not result in a denial of service.

Testable criteria:
- Redundant network paths / multi-AZ so no single-item failure causes outage (`infrastructure`).

Typically owned by: infrastructure, operator governance.

#### GIS-5.7.5 — 24/7 monitored entry/exit points

*must · tier GIG2* · maps to `GIS-5.7.5 [identified, managed, controlled, monitored 24/7]`

All network entry and exit points are identified, managed, controlled, and monitored on a 24/7 basis.

Testable criteria:
- Inventory of ingress/egress points with continuous monitoring/alerting (`infrastructure`).

Typically owned by: infrastructure, operator governance.

#### GIS-5.7.6 — Secure hubs, services, connection ports

*must · tier GIG1* · maps to `GIS-5.7.6`

All network hubs, services, and connection ports are secured to prevent unauthorized network access.

Testable criteria:
- Ports/services access-controlled (security groups, no open management ports) (`infrastructure`).

Typically owned by: infrastructure, operator governance.

#### GIS-5.7.7 — Disable unused services / non-essential ports

*must · tier GIG1* · maps to `GIS-5.7.7`

Unused services and non-essential ports are physically blocked or software-disabled whenever possible.

Testable criteria:
- Minimal open ports/services; default-deny egress/ingress (`infrastructure`).

Typically owned by: infrastructure, operator governance.

#### GIS-5.7.8 — Approved cryptographic protocols

*must · tier GIG1* · maps to `GIS-5.7.8 [TLS/HTTPS; stateless (UDP) not used without stateful transport]`

Approved cryptographic protocols providing CIA+accountability (e.g., TLS/HTTPS) are used for sensitive data; stateless protocols such as UDP are not used without stateful transport. *(Source cell ends mid-sentence at "Instead," — see chapters note.)*

Testable criteria:
- TLS/HTTPS on sensitive-data flows; UDP only over stateful/authenticated transport (`infrastructure`/`RPC layer`).

Typically owned by: infrastructure, operator governance.

#### GIS-5.7.9 — Network-infrastructure change audit log

*must · tier GIG1* · maps to `GIS-5.7.9 [log: a. date/time; b. reason + initial/final values; c. user account ID]`

All changes to network infrastructure are recorded in an audit log with date/time, reason + initial/final values, and the authorizing/performing user account ID.

Testable criteria:
- Network changes via IaC/change-control with an immutable audit trail (`infrastructure`/`operator governance`).

Typically owned by: infrastructure, operator governance.

## GIS-6. Service Providers

### GIS-6.2 Service Provider Communications

#### GIS-6.2.1 — Encrypted + strongly-authenticated SP communication

*must · tier GIG1* · maps to `GIS-6.2.1`

The GPE can securely communicate with Service Providers using encryption and strong authentication.

Testable criteria:
- Provider integrations use TLS/mTLS + strong auth (API keys/signatures/mTLS) (`infrastructure`/`proxy_*`).

Typically owned by: infrastructure, operator governance.

#### GIS-6.2.2 — Audit-log all SP login events

*must · tier GIG1* · maps to `GIS-6.2.2`

All login events involving Service Providers are recorded in an audit log.

Testable criteria:
- SP authentication/login events logged + retained (`proxy_*`/`logging library`).

Typically owned by: infrastructure, operator governance.

#### GIS-6.2.3 — SP communication must not degrade GPE functions

*must · tier GIG1* · maps to `GIS-6.2.3`

Communication with Service Providers must not interfere with or degrade normal GPE functions.

Testable criteria:
- Provider calls isolated (timeouts/circuit-breakers/bulkheads) so they can't degrade the GPE (`proxy_*`/`infrastructure`).

Typically owned by: infrastructure, operator governance.

#### GIS-6.2.4 — SP data must not affect patron communications

*must · tier GIG1* · maps to `GIS-6.2.4`

Service Provider data must not affect patron communications.

Testable criteria:
- SP data paths isolated from patron-facing communication paths (`infrastructure`).

Typically owned by: infrastructure, operator governance.

#### GIS-6.2.5 — SP on segmented network separate from patron segments

*must · tier GIG1* · maps to `GIS-6.2.5`

Service Providers are on a segmented network separate from network segments hosting patron connections.

Testable criteria:
- Dedicated SP subnet/security-group segmented from patron-facing segments (`infrastructure`).

Typically owned by: infrastructure, operator governance.

#### GIS-6.2.6 — Gaming disabled outside the GPE

*must · tier GIG1* · maps to `GIS-6.2.6`

Gaming is disabled on all network connections except those within the GPE.

Testable criteria:
- Gaming functions reachable only within GPE segments; disabled elsewhere (`infrastructure`).

Typically owned by: infrastructure, operator governance.

#### GIS-6.2.7 — No direct SP↔GPE packet routing

*must · tier GIG1* · maps to `GIS-6.2.7`

The GPE must not route data packets from Service Providers directly to the GPE and vice-versa.

Testable criteria:
- SP traffic mediated by proxy/broker layer, not directly routed to GPE internals (`infrastructure`/`proxy_*`).

Typically owned by: infrastructure, operator governance.

#### GIS-6.2.8 — GPE not acting as IP router

*must · tier GIG1* · maps to `GIS-6.2.8`

The GPE must not act as IP routers between the GPE and Service Providers.

Testable criteria:
- GPE hosts do not perform IP forwarding/routing between SP and GPE (`infrastructure`).

Typically owned by: infrastructure, operator governance.

#### GIS-6.2.9 — Prevent unauthorized-SP data view/alter

*must · tier GIG1* · maps to `GIS-6.2.9`

Unauthorized Service Providers must be prevented from viewing or altering sensitive data.

Testable criteria:
- Authorization scoping so only authorized providers touch their permitted data (`proxy_*`/`identity provider`).

Typically owned by: infrastructure, operator governance.

## GIS-7. Technical Controls

### GIS-7.1 Domain Name Service (DNS) Requirements

#### GIS-7.1.1 — Separate secure primary + secondary DNS

*must · tier GIG2* · maps to `GIS-7.1.1 [logically + physically separate; SPOF/attack resilience]`

A secure primary DNS server and a secure secondary DNS server, logically and physically separate, enhance resilience against single points of failure and attacks.

Testable criteria:
- Redundant managed DNS (the managed DNS service) across separated infrastructure (`infrastructure`).

Typically owned by: infrastructure.

#### GIS-7.1.3 — MFA-gated DNS access

*must · tier GIG2* · maps to `GIS-7.1.3 [logical + physical access via MFA; records protected]`

Logical and physical access to DNS servers is restricted to authorized personnel via MFA, keeping DNS records secure from malicious/unauthorized changes.

Testable criteria:
- DNS management console/API access gated by MFA + IAM (`infrastructure`).

Typically owned by: infrastructure.

#### GIS-7.1.4 — Restrict zone transfers

*must · tier GIG2* · maps to `GIS-7.1.4`

Zone transfers to arbitrary hosts are disallowed, preventing unauthorized access/replication of DNS zone data.

Testable criteria:
- No open AXFR; zone data not transferable to arbitrary hosts (managed DNS default) (`infrastructure`).

Typically owned by: infrastructure.

#### GIS-7.1.5 — Cache-poisoning prevention (DNSSEC)

*must · tier GIG2* · maps to `GIS-7.1.5`

A method to prevent cache poisoning, such as DNSSEC, is required.

Testable criteria:
- DNSSEC enabled (or equivalent anti-poisoning) on hosted zones (`infrastructure`).

Typically owned by: infrastructure.

### GIS-7.2 Cryptographic Controls

#### GIS-7.2.2 — Encryption grade appropriate to sensitivity

*must · tier GIG1* · maps to `GIS-7.2.2`

The grade of encryption used is appropriate to the sensitivity of the data.

Testable criteria:
- Algorithm/key-length selection matched to data classification (`infrastructure`).

Typically owned by: infrastructure, operator governance.

#### GIS-7.2.4 — Key agility (change/replace algorithms)

*must · tier GIG1* · maps to `GIS-7.2.4 [different keys; other methodologies reviewed case-by-case]`

The encryption method uses different encryption keys so algorithms can be changed/replaced to correct weaknesses as soon as practical; other methodologies are reviewed case-by-case.

Testable criteria:
- Key/algorithm swap-ability without data-loss (envelope encryption / KMS key aliases) (`infrastructure`).

Typically owned by: infrastructure, operator governance.

#### GIS-7.2.6 — Authorized-only key generation

*must · tier GIG1* · maps to `GIS-7.2.6`

Procedures for obtaining/generating encryption keys ensure only authorized personnel are involved.

Testable criteria:
- Key-generation restricted via KMS IAM to authorized roles (`infrastructure`).

Typically owned by: infrastructure, operator governance.

#### GIS-7.2.7 — Encrypted key-at-rest on redundant media

*must · tier GIG1* · maps to `GIS-7.2.7 [keys encrypted under a different method/key; redundant storage]`

Encryption keys are stored on secure, redundant media after being themselves encrypted via a different encryption method and/or a different key.

Testable criteria:
- Keys wrapped (KMS envelope) + durably stored (`infrastructure`).

Typically owned by: infrastructure, operator governance.

#### GIS-7.2.8 — Monitor key expiry

*must · tier GIG1* · maps to `GIS-7.2.8 [where applicable]`

Procedures monitor the expiration dates of encryption keys, where applicable.

Testable criteria:
- Key-expiry/rotation tracking + alerts (`infrastructure`).

Typically owned by: infrastructure, operator governance.

#### GIS-7.2.10 — Secure keyset rotation

*must · tier GIG1* · maps to `GIS-7.2.10 [new-key generation + old-key retirement]`

Procedures securely change the current encryption keyset, including generating new keys and retiring old keys.

Testable criteria:
- Automated/documented key rotation with retirement (`infrastructure`).

Typically owned by: infrastructure, operator governance.

### GIS-7.3 Critical System Component Hardening

#### GIS-7.3.1 — Established/documented/monitored/reviewed configs

*must · tier GIG1* · maps to `GIS-7.3.1`

Critical System Component configurations are established, documented, implemented, monitored, and reviewed.

Testable criteria:
- Config baselines as IaC/image definitions with drift monitoring (`infrastructure`).

Typically owned by: infrastructure.

#### GIS-7.3.2 — Vulnerability-addressing, best-practice hardening

*must · tier GIG1* · maps to `GIS-7.3.2`

Configuration procedures address all known security vulnerabilities and are consistent with industry-accepted system-hardening best practices.

Testable criteria:
- Hardening to CIS/industry benchmarks; vulns remediated (`infrastructure`).

Typically owned by: infrastructure.

#### GIS-7.3.4 — Remove risk-presenting default parameters

*must · tier GIG1* · maps to `GIS-7.3.4`

All default/standard configuration parameters presenting a security risk are removed from all Critical System Components.

Testable criteria:
- No risky defaults (accounts/passwords/settings) in baseline images (`infrastructure`).

Typically owned by: infrastructure.

#### GIS-7.3.5 — One primary function per server

*must · tier GIG1* · maps to `GIS-7.3.5 [prevent different-security-level functions co-existing]`

Only one primary function is implemented per server, preventing functions requiring different security levels from co-existing on the same server.

Testable criteria:
- Single-responsibility microservices / one-function containers (`infrastructure`).

Typically owned by: infrastructure.

#### GIS-7.3.6 — Extra controls for insecure services/protocols/daemons

*must · tier GIG1* · maps to `GIS-7.3.6`

Additional security features are implemented for any required services, protocols, or daemons considered insecure.

Testable criteria:
- Compensating controls documented for any required insecure service (`infrastructure`).

Typically owned by: infrastructure.

#### GIS-7.3.7 — Misuse-preventing security parameters

*must · tier GIG1* · maps to `GIS-7.3.7`

System security parameters are configured to prevent misuse.

Testable criteria:
- Security parameters set to safe values in baseline (`infrastructure`).

Typically owned by: infrastructure.

#### GIS-7.3.8 — Remove unnecessary functionality

*must · tier GIG1* · maps to `GIS-7.3.8 [scripts, drivers, features, subsystems, file systems, web servers]`

All unnecessary functionality is removed — scripts, drivers, features, subsystems, file systems, and unnecessary web servers.

Testable criteria:
- Minimal/distroless images; no superfluous packages/services (`infrastructure`).

Typically owned by: infrastructure.

### GIS-7.4 Generation and Storage of Security Reports or Logs

#### GIS-7.4.1 — Predefined per-component security logs

*must · tier GIG1* · maps to `GIS-7.4.1 [monitor/rectify anomalies, flaws, alerts]`

Security reports/logs are predefined and generated on each Critical System Component to monitor and rectify anomalies, flaws, and alerts.

Testable criteria:
- Every service emits structured security-relevant logs; alerts wired (`logging library`/`infrastructure`).

Typically owned by: logging library, infrastructure.

#### GIS-7.4.2 — Protect logs against tampering/unauthorized access

*must · tier GIG2* · maps to `GIS-7.4.2`

Security reports/logs are protected against tampering and unauthorized access.

Testable criteria:
- Immutable/append-only log store + restricted access (`infrastructure`).

Typically owned by: logging library, infrastructure.

## GIS-8. Remote Access and Firewalls

### GIS-8.1 Remote Access Security

#### GIS-8.1.2 — Secured and managed remote-access methods

*must · tier GIG1* · maps to `GIS-8.1.2`

Remote-access methods are appropriately secured and managed.

Testable criteria:
- VPN/bastion/SSM with MFA + centralized management (`infrastructure`).

Typically owned by: infrastructure, operator governance.

#### GIS-8.1.3 — Default-disabled remote access

*must · tier GIG1* · maps to `GIS-8.1.3 [enable/disable capability; default = disabled]`

The GPE can enable/disable remote access, with the default state set to disabled.

Testable criteria:
- Remote access off by default; explicitly enabled when needed (`infrastructure`).

Typically owned by: infrastructure, operator governance.

#### GIS-8.1.4 — Only firewall-permissible remote connections

*must · tier GIG1* · maps to `GIS-8.1.4`

Remote access accepts only remote connections permissible by the firewall application and system settings.

Testable criteria:
- Remote connections constrained by security-group/firewall allowlist (`infrastructure`).

Typically owned by: infrastructure, operator governance.

#### GIS-8.1.5 — Least-function remote access

*must · tier GIG1* · maps to `GIS-8.1.5`

Remote access is limited to only the application functions necessary for users to perform their job duties.

Testable criteria:
- Remote sessions scoped to job-necessary functions (RBAC) (`infrastructure`/`identity provider`).

Typically owned by: infrastructure, operator governance.

#### GIS-8.1.6 — No unauthorized remote user-administration

*must · tier GIG1* · maps to `GIS-8.1.6 [adding users, changing permissions, etc.]`

No unauthorized remote user-administration functionality (adding users, changing permissions, etc.) is permitted.

Testable criteria:
- User-admin actions blocked/authorized-only over remote access (`identity provider`/`infrastructure`).

Typically owned by: infrastructure, operator governance.

#### GIS-8.1.7 — No unauthorized OS/DB access beyond retrieval

*must · tier GIG1* · maps to `GIS-8.1.7`

Unauthorized remote access to the operating system, or to any database other than information-retrieval access, is prohibited.

Testable criteria:
- Remote OS/DB access restricted to authorized paths; retrieval-only otherwise (`infrastructure`).

Typically owned by: infrastructure, operator governance.

#### GIS-8.1.8 — Remote-access audit log

*must · tier GIG1* · maps to `GIS-8.1.8 [a. user ID + auth verification; b. IP/port/protocol/MAC; c. connect time + duration; d. reason + work; e. activity/areas/changes]`

The GPE maintains an audit log of all remote-access information/activity, minimally including authorized user account ID (with auth verification), remote IP/port/protocol/MAC, connection time + duration, reason + work description, and in-session activity (areas accessed, changes made).

Testable criteria:
- Remote-access sessions logged with all five field groups (`infrastructure`/`logging library`).

Typically owned by: infrastructure, operator governance.

### GIS-8.2 Firewall Security

#### GIS-8.2.1 — All comms through an approved application-level firewall

*must · tier GIG1* · maps to `GIS-8.2.1 [incl. remote access + non-system hosts]`

All communications (including remote access) pass through at least one approved application-level firewall, including connections to/from any non-system hosts used by the Gaming Enterprise.

Testable criteria:
- WAF + security groups front all ingress/egress; no unfiltered paths (`infrastructure`).

Typically owned by: infrastructure.

#### GIS-8.2.2 — Firewall at dissimilar-security-domain boundaries

*must · tier GIG1* · maps to `GIS-8.2.2`

The firewall is located at the boundary of any two dissimilar security domains.

Testable criteria:
- VPC/subnet boundaries enforced by security groups/NACLs between trust zones (`infrastructure`).

Typically owned by: infrastructure.

#### GIS-8.2.3 — No firewall-bypass alternate path

*must · tier GIG2* · maps to `GIS-8.2.3`

A device in the same broadcast domain as the system host must not have a facility allowing an alternate network path that bypasses the firewall.

Testable criteria:
- No routes/peering bypass the enforced firewall boundary (`infrastructure`).

Typically owned by: infrastructure.

#### GIS-8.2.4 — Redundant paths also firewalled

*must · tier GIG1* · maps to `GIS-8.2.4`

Any alternate network path existing for redundancy also passes through at least one application-level firewall.

Testable criteria:
- Failover/redundant paths traverse the same firewall controls (`infrastructure`).

Typically owned by: infrastructure.

#### GIS-8.2.5 — Only firewall-related apps on the firewall

*must · tier GIG1* · maps to `GIS-8.2.5`

Only firewall-related applications may reside on the firewall.

Testable criteria:
- Managed firewall services run no unrelated workloads (inherent) (`infrastructure`).

Typically owned by: infrastructure.

#### GIS-8.2.6 — Limited firewall accounts

*must · tier GIG1* · maps to `GIS-8.2.6 [e.g., network/system administrators only]`

User accounts on the firewall are limited (e.g., network or system administrators only).

Testable criteria:
- Firewall/security-group management restricted via IAM to admins (`infrastructure`).

Typically owned by: infrastructure.

#### GIS-8.2.7 — Default-deny (reject all but approved)

*must · tier GIG1* · maps to `GIS-8.2.7`

The firewall rejects all connections except those specifically approved.

Testable criteria:
- Default-deny rulebase; only explicit allows (`infrastructure`).

Typically owned by: infrastructure.

#### GIS-8.2.8 — Anti-spoofing reject (impossible source addresses)

*must · tier GIG1* · maps to `GIS-8.2.8 [e.g., RFC1918 on public side]`

The firewall rejects connections from destinations that cannot reside on the originating network (e.g., RFC1918 addresses on the public side of an internet firewall).

Testable criteria:
- Anti-spoofing/bogon filtering at the perimeter (`infrastructure`).

Typically owned by: infrastructure.

#### GIS-8.2.9 — Remote access only via encryption + strong auth

*must · tier GIG1* · maps to `GIS-8.2.9`

The firewall only allows remote access using encryption and strong authentication.

Testable criteria:
- Remote-access ingress requires encrypted + strongly-authenticated channels (`infrastructure`).

Typically owned by: infrastructure.

#### GIS-8.2.10 — Tamper-resistant firewall audit log

*must · tier GIG1* · maps to `GIS-8.2.10 [a. config changes; b. successful+unsuccessful connections; c. src/dst IP/port/protocol/MAC]`

The firewall logs — in a loss/alteration-preserving audit log — all firewall configuration changes, all successful/unsuccessful connection attempts, and the source/destination IP/port/protocol/MAC.

Testable criteria:
- Flow logs + config-change logs (VPC Flow Logs/CloudTrail) to an immutable store (`infrastructure`).

Typically owned by: infrastructure.

#### GIS-8.2.11 — Threshold-based lockout on failed connections

*may · tier GIG1* · maps to `GIS-8.2.11 [configurable deny + admin notification on threshold]`

For unsuccessful connection attempts through the firewall, a configurable parameter may be used to deny further connection requests and notify the system administrator once a predefined threshold is exceeded.

Testable criteria:
- (Optional) rate-limit/deny + alert on repeated failed connections where adopted (`infrastructure`).

Typically owned by: infrastructure.

## GIS-9. Critical Asset and Change Management Review

### GIS-9.1 Asset Management

#### GIS-9.1.1 — Account for all sensitive-data assets

*must · tier GIG1* · maps to `GIS-9.1.1 [physical + logical; incl. GPE]`

All physical or logical assets housing, processing, or communicating sensitive data — including those comprising the GPE — are accounted for.

Testable criteria:
- Complete asset inventory (a cloud config/resource-inventory service + IaC) covering sensitive-data assets (`infrastructure`).

Typically owned by: infrastructure, operator governance.

#### GIS-9.1.6 — ≥ annual maintenance/inspection/service

*must · tier GIG1* · maps to `GIS-9.1.6 [free from defects/mechanisms interfering with operation]`

To ensure continued CIA+accountability, assets are correctly maintained, inspected, and serviced at least annually (or at required intervals) to be free from defects/mechanisms that could interfere with operation.

Testable criteria:
- Managed-service patching/maintenance + periodic review of critical assets (`infrastructure`).

Typically owned by: infrastructure, operator governance.

### GIS-9.3 Change Management Program (CMP)

#### GIS-9.3.2 — Only authorized versions implemented

*must · tier GIG1* · maps to `GIS-9.3.2`

Program change procedures ensure only authorized versions of programs and modifications are implemented in the GPE.

Testable criteria:
- PR-approval + protected-branch + signed/immutable artifacts gate deploys (`infrastructure`/CI-CD).

Typically owned by: operator governance, infrastructure.

#### GIS-9.3.3 — Version control for all code/binaries

*must · tier GIG1* · maps to `GIS-9.3.3 [software components, source code, binary controls]`

An appropriate software version-control mechanism is in place for all software components, source code, and binary controls.

Testable criteria:
- Git for source; versioned/immutable artifact registry for binaries (`infrastructure`).

Typically owned by: operator governance, infrastructure.

#### GIS-9.3.4 — Change Management Log (5 fields, CAR-linked)

*must · tier GIG1* · maps to `GIS-9.3.4 [a. date; b. reason/nature; c. component + CAR unique ID + version (+ hw location); d. performer; e. authorizer]`

A Change Management Log records all new installations/modifications with date, reason/nature, component(s) (incl. CAR unique ID, version info, hardware location), the performing user(s), and the authorizing user(s).

Testable criteria:
- Change log (git history + deploy records) capturing all five fields, referencing CAR IDs (`infrastructure`/`operator governance`).

Typically owned by: operator governance, infrastructure.

#### GIS-9.3.6 — Rollback / field-issue strategy

*must · tier GIG1* · maps to `GIS-9.3.6 [a. app-store/outside-party release management; b. rollback plan + prior-version backups + tested rollback]`

A strategy covers unsuccessful installs/field issues — managing releases through outside parties (e.g., app stores) where applicable, and otherwise reverting to the last implementation (rollback plan with prior-version backups and a tested rollback before GPE deployment).

Testable criteria:
- Documented + tested rollback (blue-green/canary + prior-version artifacts) (`infrastructure`).

Typically owned by: operator governance, infrastructure.

#### GIS-9.3.8 — Tested migration + authorized signoff

*must · tier GIG1* · maps to `GIS-9.3.8`

Procedures for testing and migrating changes include identifying authorized personnel for signoff prior to release.

Testable criteria:
- CI test gates + required reviewer approval before release (`infrastructure`/`operator governance`).

Typically owned by: operator governance, infrastructure.

#### GIS-9.3.9 — Segregation of duties in release

*must · tier GIG1* · maps to `GIS-9.3.9`

There is segregation of duties within the release process.

Testable criteria:
- Author ≠ approver/deployer enforced (branch protection, deploy roles) (`infrastructure`).

Typically owned by: operator governance, infrastructure.

### GIS-9.4 System Development Lifecycle

#### GIS-9.4.2 — GPE separated from dev/test (no direct connection)

*must · tier GIG1* · maps to `GIS-9.4.2 [logically + physically; no direct connection to any other environment]`

The GPE is logically and physically separated from development and testing environments such that no direct connection exists between the GPE and any other environment.

Testable criteria:
- Separate cloud accounts/VPCs/clusters for prod vs dev/test; no direct connectivity (`infrastructure`).

Typically owned by: operator governance, infrastructure.

#### GIS-9.4.4 — Documented secure-coding method

*must · tier GIG1* · maps to `GIS-9.4.4 [industry standards + best practices for coding]`

The Gaming Enterprise establishes and documents a method for developing software securely, following industry standards and best practices for coding.

Testable criteria:
- Secure-coding standard + SAST/lint/dependency-scan in CI (`operator governance`/`infrastructure`).

Typically owned by: operator governance, infrastructure.

### GIS-9.5 Patch Management

#### GIS-9.5.2 — Monitor + apply patches to sensitive-data CSCs

*must · tier GIG1* · maps to `GIS-9.5.2 [collection, processing, storage, transmission of sensitive data]`

Patches are monitored and applied to all Critical System Components involved in collecting, processing, storing, and transmitting sensitive data.

Testable criteria:
- Vulnerability monitoring (Dependabot/ECR/OS scan) + patch cadence on all CSCs (`infrastructure`).

Typically owned by: infrastructure, operator governance.

#### GIS-9.5.3 — Test patches on identically-configured env

*must · tier GIG1* · maps to `GIS-9.5.3 [whenever possible]`

Whenever possible, all patches are tested on a development/testing environment configured identically to the target GPE.

Testable criteria:
- Patches validated in a prod-parity staging environment before rollout (`infrastructure`).

Typically owned by: infrastructure, operator governance.

### GIS-9.6 In-House Developed/Modified Software

#### GIS-9.6.3 — Program-change gate before operational use

*must · tier GIG1* · maps to `GIS-9.6.3`

In-house software is not implemented or used operationally until it has gone through the required program-change procedures.

Testable criteria:
- Deploy blocked until CMP/change-control steps complete (`infrastructure`/`operator governance`).

Typically owned by: operator governance, platform backend.

#### GIS-9.6.7 — Preserve logs / access history / change tracking

*must · tier GIG1* · maps to `GIS-9.6.7 [a. comprehensive log files; b. user access history; c. change tracking]`

In-house software includes/is subject to procedures generating and preserving comprehensive log files, user access history, and change tracking.

Testable criteria:
- Application logs + access history + change tracking (git) preserved (`platform backend`/`logging library`).

Typically owned by: operator governance, platform backend.

#### GIS-9.6.8 — Restrict access (internal or network-enforced)

*must · tier GIG1* · maps to `GIS-9.6.8 [a. internal access-control; or b. network permissions + GIS Controls]`

Access to in-house software is restricted via internal access-control mechanisms or external enforcement through network permissions and GIS controls.

Testable criteria:
- RBAC in-app and/or network-level access restriction (`identity provider`/`infrastructure`).

Typically owned by: operator governance, platform backend.

#### GIS-9.6.12 — Patron-outcome software auditability

*must · tier GIG1* · maps to `GIS-9.6.12 [patron outcomes — promotions, loyalty, drawings; a. review records; b. reconstruct behavior; c. report to Regulatory Body]`

In-house software affecting patron outcomes (e.g., promotions, loyalty tracking, drawings) includes a documented method for reviewing relevant records, reconstructing the software's behavior, and reporting findings to the Regulatory Body.

Testable criteria:
- Promotions/loyalty/bonus logic auditable + reconstructable from records (`segmentation service`/`CRM / messaging service`/`platform backend`).

Typically owned by: operator governance, platform backend.
