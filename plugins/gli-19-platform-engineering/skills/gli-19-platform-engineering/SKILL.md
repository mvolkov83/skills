---
name: gli-19-platform-engineering
description: Engineering rules for building an interactive (online) gaming platform that can pass a GLI-19 technical certification — server-authoritative game outcomes, authoritative system clock and time-sync, the records a gaming system must be able to produce (per-game play record, per-theme aggregates, player-account record, significant-event log, operator-access record), append-only financial history with integrity hashing and actor attribution, supervised alteration of accounting data, wagering order for restricted incentive credits, wager debit and award settlement semantics, game-cycle exclusivity, interrupted-game hold and completion, game recall depth, player-account lifecycle (age gate, identity verification before first play, one active account, MFA on credential and payout-account changes, enumeration-resistant login, lockout), session inactivity re-authentication, responsible-gaming limit precedence, payout-destination binding, geolocation checks and spoofing detection, on-demand disable of gaming/themes/logins, regulator reporting surfaces (game performance with theoretical vs actual RTP, operator liability, large jackpot payout, alteration report), control-program self-verification and approved-version signature checks, and communications/crypto/key-management obligations. Use this skill whenever the user is designing, writing, reviewing or migrating any part of an online casino / sportsbook / interactive gaming platform — wallet or ledger services, game-session orchestrators, RGS or game-aggregation layers, player-account and identity services, payment services, back-office and reporting surfaces, admin RPCs that mutate financial or gaming data, bonus and incentive engines, jackpot engines, or the schema behind any of them. Trigger on phrases like "gaming platform", "online casino", "player account", "player wallet", "bet/win/refund", "game round", "game session", "bonus credits", "free spins", "wagering requirement", "jackpot", "RTP", "game history", "game recall", "self-exclusion", "deposit limit", "responsible gaming", "KYC before play", "geolocation check", "significant event log", "audit trail for financial adjustment", "back-office adjustment", "void a transaction", "GLI", "GLI-19", "certification", "regulator report", "gaming licence", and on any table or proto named like `rounds`, `transactions`, `wagers`, `player_accounts`, `bonuses`, `jackpots`. Trigger even for a small change — an added admin endpoint, one new column on a financial table, or a tweak to balance-consumption order — because these are exactly the places where a certification finding is created and where retrofitting is expensive. Do NOT use for land-based gaming devices and cabinets (GLI-11), standalone progressive-jackpot controllers assessed on their own (GLI-12), lottery or iLottery systems (GLI-20/GLI-27), sports-betting odds/settlement engines assessed separately, or generic payment-card compliance work (PCI-DSS).
---

# GLI-19 platform engineering

Design and implementation rules for an interactive gaming system that has to survive a technical
certification. Every rule maps to one or more GLI-19 v3.0 requirement ids; the full paraphrased
requirement text, with testable criteria, is in `references/requirements.md` — read the entry when a
rule's scope is unclear or when you need the criteria a laboratory will test against.

Two commands ship alongside these rules. `/gli-19-review-diff` reviews a change for requirements it
violates or undermines. `/gli-19-review-surface` assesses a service for requirements it does not
implement — the absence findings a diff review structurally cannot produce.

**These rules are engineering guidance, not the standard.** They cover the ~96 GLI-19 requirements
whose evidence is code, infrastructure or configuration. They do not cover the operator's procedural
and governance obligations (internal control procedures, service-provider agreements, staffing,
incident-response process), they carry no certification weight, and they are never a substitute for
the standard document or the testing laboratory's judgment. Never quote a rule from this skill to a
regulator or a lab as if it were the requirement.

Applicability is jurisdiction- and scope-dependent. An operator running only third-party games
delegates most game-design obligations to the game provider's own certificate; an in-house studio does
not. Do not mark something out of scope on your own — ask, and record the ruling where the project
keeps its scoping decisions.

## 1. Server authority and the client trust boundary

1. Generate every game outcome server-side. The player client must contain no logic that determines a
   result — no RNG, no paytable evaluation, no win calculation, not even "for animation timing".
   → GLI19-2.6.5.server-authority
   **Why:** this is the single most load-bearing requirement in the standard. A client that can
   compute an outcome can forge one, and no amount of server-side validation afterwards recovers the
   certification argument. It also determines your protocol shape: if outcome fields travel
   client→server anywhere, the design is already wrong.
2. Treat every client-supplied value as an assertion to be re-derived server-side — bet amount, chosen
   line count, gamble decision. The server owns the authoritative copy of session and round state.
   → GLI19-2.6.5.server-authority, GLI19-4.6.1.b
3. Ship an identifiable software version in the client and expose it to the server on session start;
   log it against the session. → GLI19-2.6.2
4. Halt gameplay on loss of connectivity to the platform and surface an explicit error rather than
   degrading to a local mode. Detection on the next communication attempt is acceptable; silent local
   continuation is not. → GLI19-2.6.4
5. Authenticate critical client components at load time for installed clients, block gameplay on
   mismatch, and check device compatibility before the first game rather than mid-round.
   → GLI19-2.6.3, GLI19-2.6.6
6. Keep the client free of behaviours that touch device security: no opening of unnecessary ports, no
   disabling of device protections, no storing of credentials or sensitive data, no volume override.
   → GLI19-2.6.5.client-safety

## 2. Time

7. Derive every timestamp that lands in a transaction, game record, or event log from one
   authoritative platform time source. Never stamp a record from an arbitrary service's local clock
   or from a client-supplied time. → GLI19-2.2.1
8. Synchronize all components to a single mechanism, configured declaratively, and expose clock offset
   as a monitored signal. → GLI19-2.2.2
   **Why:** the audit trail's evidentiary value collapses if two services disagree about ordering.
   Reconstructing a disputed round across a wallet and an orchestrator whose clocks drifted is the
   scenario this requirement exists for.
9. Store timestamps as UTC with an explicit timezone-aware type; keep the display timezone a
   presentation concern.
10. Log a change to the master time source as a significant event. → GLI19-2.8.8

## 3. Records the system must be able to produce

Design these as first-class persisted records, not as something reconstructible by a future report
query. The obligation is that the data exists, is backed up, and is exportable.
→ GLI19-2.8.1

11. Persist a per-game play record sufficient to reconstruct the game: identifiers (game cycle,
    session, player, theme/paytable), timestamps, funds before and after, total wagered and total won
    including incentive credits, player choices, intermediate phases, jackpot indication, and current
    status (in progress / complete / interrupted / cancelled). → GLI19-2.8.2
12. Persist a per-theme/paytable record: configuration, availability date, theoretical RTP, and the
    play/financial aggregates (games played, total wagers, total paid, jackpot payouts, incentive
    credits wagered and won, voids, rake). → GLI19-2.8.3
    **Why:** the raw substrate almost always exists in the rounds table while the aggregate does not,
    and teams assume "we can compute it later". The requirement is a maintained record, and its
    absence also blocks the game-performance report (rule 44). Materialize it when the theme model is
    designed, not after a lab flags it.
13. Persist a complete player-account record: identity and PII (encrypted where required), verification
    method and date, balances with restricted incentive credits tracked separately from unrestricted
    funds, exclusions and limitations with their windows, every access with IP, the full financial
    transaction history with balance before and after per transaction, and account status.
    → GLI19-2.8.5
14. Persist per-incentive and per-jackpot records — offer identity, availability, issued/redeemed/
    expired/adjusted totals; jackpot identity, participating themes, current and reset values,
    contribution pools, and every configurable parameter needed to reconcile the jackpot.
    → GLI19-2.8.6, GLI19-2.8.7
15. Persist a per-contest/tournament record where tournaments are offered. → GLI19-2.8.4
16. Maintain a significant-event log covering at minimum: failed access attempts with IP, program
    errors and authentication mismatches, unavailability of a critical component, large wins and large
    wagers above the regulator threshold, voids/overrides/corrections, changes to game-theme, jackpot
    and incentive parameters, changes to system policies and parameters, changes to the master clock,
    player-account management events (balance adjustments, PII changes, deactivation, negative
    balances), and loss of PII. → GLI19-2.8.8
17. Maintain a per-operator-user access record: identity, role and the functions that role may execute,
    creation, last access with IP, last credential change, deactivation, group membership, status.
    → GLI19-2.8.9
18. Provide a player-facing transaction log / account statement covering at least the past year:
    financial transactions with unique ids, and game history by theme with totals wagered and won.
    → GLI19-2.5.7
19. Provide a data export path for all of the above in an analyzable format. A report UI is not an
    export. → GLI19-2.8.1

## 4. Integrity and mutation of financial and audit data

20. Make financial and audit records append-only. Corrections are new compensating rows, never updates
    to a historical row and never deletes. For ledger structure defer to the
    `money-and-payments-best-practices` skill; the certification adds the requirements below on top.
    → GLI19-B.3.2
21. Attribute every mutation. Every row that alters accounting, reporting or significant-event data
    carries: a unique alteration id, the data element, the value before, the value after, the
    timestamp, and the identity of the person who performed it. → GLI19-B.3.2, GLI19-2.9.5
    **Why:** the "who" field is the one routinely missing. A financial void carrying only a request id
    and a player-facing session token is unattributable — the record cannot answer who authorized it,
    which is both a certification finding and, in a real dispute, an unanswerable question.
22. Put supervised access control on every surface that can alter accounting, reporting or
    significant-event data — admin adjustment, void, cancel, manual settlement, parameter change.
    Operator identity, distinct from any player credential, with an authorization decision that is
    itself logged. → GLI19-B.3.2, GLI19-B.2.3
    **Why:** an admin RPC that authenticates with the same token type as a player session is a
    privilege boundary that does not exist. Reuse of a player session token on a financial-mutation
    endpoint is a high-severity finding and a live security hole.
23. Protect financial records against silent tampering: a content hash per row, chained to the previous
    row, verified during reconciliation. Cover the fields that matter — amounts, currency, account,
    actor, timestamps. → GLI19-B.3.1, GLI19-B.3.2
    **Why:** retrofitting an integrity column into a populated append-only table means a backfill whose
    own trustworthiness is exactly what is in question. It is cheap at schema-design time and
    permanently awkward afterwards.
24. Validate and reject corrupt input at every boundary that writes gaming or accounting data; never
    let a partially-valid message produce a partially-written record. → GLI19-B.3.1
25. Never build a mechanism that clears data on error, and never hold gaming or accounting state only
    in memory where a restart loses it. → GLI19-B.3.1
26. Encrypt PII and sensitive data at rest, and keep production data stores off the network segment
    that serves player interfaces. → GLI19-B.3.1
27. Make a master reset — of any component affecting gaming operations — detectable and recorded, not
    a silent event. → GLI19-B.3.6

## 5. Wallet and wagering semantics

For money representation, idempotency, and double-entry structure, defer to the
`money-and-payments-best-practices` skill. The rules here are the gaming-specific overlay.

28. Consume restricted incentive credits before unrestricted player funds whenever both are combined
    on one balance, unless a game rule explicitly says otherwise. → GLI19-4.3.5.d
    **Why:** the natural implementation instinct — and the one legacy systems usually encode — is
    real-money-first, because it looks better for the player and simpler for the ledger. It is the
    inverse of the requirement. Make consumption order an explicit, tested, per-game-overridable
    policy rather than an emergent property of how the balance list happens to be concatenated.
29. Debit the wager at wager time and reject anything that would drive the balance negative. There is
    no "settle later" path for a debit. → GLI19-4.3.3.b
30. Credit awards to the balance on game-cycle completion, with only the regulator-sanctioned
    exceptions (merchandise, large payouts held for manual handling). → GLI19-4.3.3.d
31. Enforce game-cycle exclusivity: no new game in a session before the current cycle settles and
    balance and history are updated. → GLI19-4.3.3.e
32. Hold wagers on an interrupted game until it completes, and reflect held funds in the player's
    account view — held funds are neither spent nor available. → GLI19-4.16.2
33. Provide an explicit completion mechanism for an interrupted game and resolve it before the player
    can start another instance of the same game. Crash-recovery of a partially-applied transaction is
    a different thing and does not satisfy this on its own. → GLI19-4.16.3.mechanism
    **Why:** idempotent resume of an in-flight write is an infrastructure property; returning the
    player to the pre-interruption game state and letting them finish is a product feature. Teams
    routinely report the first as if it were the second.
34. Implement the resolution paths: where no further input is needed, return the game to a completed
    state with history and balance consistent; where input is needed, restore the pre-interruption
    state — unless superseding recovery rules are disclosed to the player.
    → GLI19-4.16.3.single-player
35. For multi-player games, complete on the player's behalf per the rules when they time out, update
    history and balances, disclose which decisions the platform made, and never let one player's delay
    affect the others. → GLI19-4.16.3.multi-player
36. Block direct player-to-player fund transfers at the API and at the ledger level.
    → GLI19-2.5.6.f
37. Return an explicit confirmation or denial for every financial transaction, carrying type, value,
    and a descriptive reason on denial. → GLI19-2.5.6.a
38. Exclude unauthorized or unsettled deposits from the wagerable balance, and store the issuer
    authorization reference against the transaction. → GLI19-2.5.6.c
39. Bind payout destinations to the player's verified identity; block a withdrawal to a destination
    that does not match registration details. → GLI19-2.5.6.d
40. Handle an over-limit transaction by capping with an explicit notification of the reduced amount, or
    by blocking — never by silently processing the full amount. → GLI19-2.5.6.e
41. Make loyalty-point redemption an atomic transaction that debits the points balance, record every
    accrual, redemption and adjustment, and keep award eligibility uniform for every player who reaches
    the qualification level. → GLI19-2.5.8

## 6. Player account lifecycle, identity and session

42. Gate registration on legal age before the account row is created, and store T&C and privacy
    consent with a timestamp. → GLI19-2.5.2.a
43. Verify identity — legal name, residential address, date of birth — and screen against exclusion
    lists before the first game is playable. Activation happens only when every precondition is
    simultaneously satisfied. → GLI19-2.5.2.b, GLI19-2.5.2.c
44. Enforce one active account per verified player, with deduplication on identity attributes; any
    exception is a configured regulator authorization, never a default. → GLI19-2.5.2.d
45. Require MFA on credential changes, registration-data changes, and changes to the account used for
    financial transactions — and log each with actor and timestamp. → GLI19-2.5.2.e
    **Why:** the payout-destination change is the highest-value target in the whole platform. An
    account takeover that can silently repoint withdrawals turns every other control into decoration.
46. Return an identical failure response regardless of which credential was wrong, and keep timing and
    status codes uniform too. → GLI19-2.5.3.a
47. Require MFA on credential recovery and reset. → GLI19-2.5.3.b
48. Lock an account on suspicious activity — three consecutive failures within thirty minutes is the
    reference threshold — and require MFA to unlock. → GLI19-2.5.3.d
49. Force re-authentication after thirty minutes of inactivity (or the jurisdiction's period), and
    block games and financial transactions on that device until it succeeds. Where a simpler re-auth
    means is offered, still require full authentication at least every thirty days.
    → GLI19-2.5.4.a, GLI19-2.5.4.b
50. Resolve conflicting limits by applying the most restrictive: a player's self-imposed limit never
    relaxes an operator- or regulator-imposed one, and no internal status event may bypass an exclusion.
    → GLI19-2.5.5
    **Why:** limit precedence is usually implemented as "last writer wins" or "the more specific scope
    wins", both of which can silently loosen a self-exclusion. Encode precedence as an explicit
    `min()` over active constraints with a test per pairing.
51. Issue operator session authorization that is random, held in memory, and destroyed at session end;
    fetch authorization data at request time rather than caching it on the component.
    → GLI19-B.2.4

## 7. Operational control surfaces

52. Build the disable controls as product features, each with an actor, a reason and an audit entry:
    global gaming kill-switch, per-theme/paytable/client-version disable, and per-player-login disable.
    → GLI19-2.4.1
53. Let an in-progress game conclude when its theme is disabled — including bonus rounds and gamble
    features — and make it inaccessible once concluded. A disable that voids in-flight rounds is not
    compliant; neither is one that lets new rounds start. → GLI19-4.15.1
54. Constrain live jackpot parameter changes: an RTP-affecting increment change does not take effect
    until the current jackpot is won, a ceiling only moves above the current payoff, the trigger
    probability of the running jackpot does not change, and a mystery trigger is reselected within the
    remaining range. → GLI19-2.4.2
55. Provide an auditable operation to transfer, combine or correct jackpot contribution pools —
    including overflow and diversion pools — with actor and reason. → GLI19-2.4.3
56. Provide player-facing game recall that is clearly marked as a replay, reconstructs the outcome and
    the player's actions with all applicable fields, and covers at least the last 50 bonus/feature
    events. → GLI19-4.14.1, GLI19-4.14.2, GLI19-4.14.3

## 8. Reporting surfaces

Reports are a deliverable of the system, not an afterthought for the BI team. Each needs a data model
that can produce it at the required interval.

57. Generate reports on demand and at the required intervals, each carrying operator identity, report
    title, interval, generation timestamp, labeled fields, and an explicit no-activity indication.
    → GLI19-2.9.1
58. Provide a per-theme game-performance report including theoretical RTP and actual RTP for
    house-banked games, games played, wagers and winnings with incentive-credit amounts broken out,
    voids, rake, and funds still held in interrupted games. → GLI19-2.9.2
59. Provide an operator-liability report: total funds held for player accounts. → GLI19-2.9.3
60. Provide a large-jackpot-payout report identifying the jackpot, the winner, the winning game context
    and the operator users who processed and confirmed the win. → GLI19-2.9.4
61. Provide a significant-event / alteration report with before and after values and the users who
    performed and authorized each change — this is the read surface over rule 21's data.
    → GLI19-2.9.5

## 9. Location

62. Run a location check before the first game after login on a device, and again after an IP change,
    after thirty minutes, or as the regulator specifies. Block play and notify the player when the
    check fails or cannot locate them. → GLI19-2.7.4.checks
63. Log every location violation with a timestamp, the player id and the detected location.
    → GLI19-2.7.4.checks
64. Require the confidence radius to fall entirely inside the permitted boundary, use audited boundary
    polygons, and flag geographically impossible travel on one account for investigation.
    → GLI19-2.7.4.geolocation
65. Detect and block circumvention: VPN and proxy exit points, virtual machines, remote-desktop tooling,
    rooted or jailbroken devices, and man-in-the-middle interposition. → GLI19-2.7.2
66. Where play occurs on a private network, implement either a per-game location check or a real-time
    boundary component. → GLI19-2.7.3

## 10. Control-program integrity and release

67. Make critical components self-verifiable: a scheduled (at least daily) and on-demand check that
    each executable, library, configuration and reporting-critical component matches its approved
    digest, with a message digest of at least 128 bits and an explicit failure signal.
    → GLI19-2.3.2
68. Provide an out-of-band verification path that does not depend on the system's own processes — a
    signature comparison against an approved manifest, not self-attestation. → GLI19-2.3.3
69. Gather production component signatures at install/update, on recovery, at least every 24 hours and
    on demand; compare against the approved-version set; store the result unalterably and notify on
    failure. → GLI19-B.2.6
    **Why:** this is what makes "what is actually running in production" answerable. Immutable image
    digests pinned per environment and recorded per deploy give it to you almost for free at design
    time; it is archaeology if the deploy pipeline never captured them.
70. Keep report and document retention versioned: original plus every subsequent version, a unique
    signature per version, a complete change log with who and when, and restricted modify access.
    → GLI19-B.2.7
71. Prevent user-initiated programming against production data stores, and protect servers from
    unauthorized mobile-code execution. → GLI19-B.2.5

## 11. Communications, cryptography and access

72. Encrypt everything that crosses a public network, and treat PII, wagers, results and financial data
    as always-encrypted; protect against replay, duplication, misrouting and truncation.
    → GLI19-B.4.4
73. Authenticate and encrypt component-to-component traffic that is critical to gaming or player-account
    management; harden the protocol against malformed messages; do not resume communication after an
    interruption until the resumption routine completes. → GLI19-B.4.3
74. Default-deny component connectivity: components are un-enrolled and disabled until explicitly
    enrolled and enabled, with methods for both directions. → GLI19-B.4.2
75. Isolate third-party integrations — payment, identity, location, live-game, loyalty — on a segmented
    path that cannot reach production directly and never routes packets between a third party and
    production; log third-party login events. → GLI19-B.5.1
76. Apply a cryptographic-controls policy in code and configuration: appropriate encryption grade for
    the data's sensitivity, message authentication where data needs integrity but not concealment,
    certificates from an approved authority, and distinct keys per purpose so an algorithm can be
    replaced. → GLI19-B.6.2
77. Implement the key lifecycle explicitly: generation, restricted storage, expiry, revocation, keyset
    rotation, and recovery of data encrypted under a revoked key for a defined period.
    → GLI19-B.6.3
78. Give every operator user an individual credential with role-based access, hashed credential storage
    to current standards, forced rotation, prompt deactivation on termination, separation of duties on
    critical parameters, lockout after repeated failures, and a secure log of every access attempt.
    → GLI19-B.2.3
79. Centralize logs from every critical component, protect them against tampering and unauthorized
    access, retain them for the investigation window, and review them on a documented cadence. For
    instrumentation shape defer to the `observability-best-practices` skill.
    → GLI19-B.6.5

## 12. Availability and recovery

80. Design for no critical data loss on any single component failure, and verify with a pre-production
    test that restart and recovery of linked components neither loses nor duplicates transactions and
    that components resynchronize on recovery. → GLI19-B.3.5
    **Why:** "no lost or duplicated transactions across a restart" is a testable claim, and the test is
    the evidence. Write it as an automated crash-recovery test the day the transactional path is
    built.
81. Back up at least daily, keep redundant copies on separate media in a physically separate location
    with the same access controls as production, and include everything §2.8 requires plus
    configuration, security accounts and current encryption keys in the backup set.
    → GLI19-B.3.3, GLI19-B.3.4, GLI19-B.3.7

## When applying these rules

**Be opinionated about the footguns.** Rules 1, 21, 22, 23, 28, 33, 45 and 50 are where certification
findings and real incidents are manufactured, and all of them are far cheaper to satisfy at design
time than to retrofit. Server authority (1) constrains the protocol; actor attribution and integrity
hashing (21, 23) constrain the schema; consumption order (28) and limit precedence (50) are inverted by
the natural implementation instinct. Raise these before the design is committed, not at review.

**Be flexible about everything else.** Thresholds — thirty-minute inactivity, three failed attempts,
daily backup, the large-win reporting level — are the standard's reference values and are frequently
overridden by the jurisdiction. Make them configuration, not constants, and do not hard-code the
number this skill quotes.

**Read the requirement before arguing about scope.** `references/requirements.md` carries the testable
criteria a laboratory assesses. When a rule seems not to apply, check the entry: the usual outcome is
that it applies to a different component than assumed, not that it is out of scope.

**Read the existing code first.** A platform of any age already has a session model, a balance model
and an audit-log shape. Conform to it and extend it; a second, parallel audit log or a competing
consumption-order policy is worse than an imperfect single one.

**Know what this skill does not cover.** Operator procedures, service-provider due diligence, staffing
and segregation of duties, the submission package, and the laboratory's own testing are outside it.
Roughly a fifth of the technical-security appendix is infrastructure and network policy that lives in
IaC and runbooks rather than application code — DNS hardening, firewall boundaries and rules review,
remote-access control, patching policy, asset registers — and is deliberately not turned into
application rules here.

**Cross-skill awareness.** Money representation, idempotency and double-entry structure belong to
`money-and-payments-best-practices`. Logging, tracing and metric shape belong to
`observability-best-practices`. Schema and session mechanics belong to the relevant framework skills.
This skill only adds what certification requires on top of them.
