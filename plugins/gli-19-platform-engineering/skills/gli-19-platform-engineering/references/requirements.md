# GLI-19 v3.0 — Interactive gaming platform engineering: requirement reference

GENERATED FILE — do not edit by hand.
Source of truth: `standards/gli-19/catalog.yaml` (regenerate with `scripts/build_skill.py --standard gli-19 --profile platform`).

Scope: the **96** requirements of GLI-19 v3.0 whose evidence is code, infrastructure or configuration — the subset a developer can satisfy while writing the system. Procedure, policy, certificate and screenshot evidence is out of scope here and belongs to the operator's governance programme.

> These entries are **paraphrases** written for engineering use. They are not the normative text and carry no certification weight — the standard document itself governs, and a testing laboratory assesses against it, not against this file.

## 2. Platform/System Requirements

### 2.2 System Clock Requirements

#### GLI19-2.2.1 — Authoritative system clock for timestamping and reporting

*shall* · maps to `2.2.1 [a–c]`

The Interactive Gaming System maintains an internal clock reflecting the current date/time that serves as the authoritative time source for stamping transactions and games, stamping significant events, and as the reference clock for reporting.

Testable criteria:
- Every transaction and game record is timestamped from the system clock (not an ad-hoc per-service clock).
- Every significant event is timestamped from the system clock.
- Reports use the system clock as their reference time source.

Typically owned by: platform backend.

#### GLI19-2.2.2 — Time synchronization across all system components

*shall* · maps to `2.2.2`

All Interactive Gaming System components maintain synchronized and correctly set date/time via a single synchronization mechanism, so that timestamps for transactions, significant events, and reports are mutually consistent across services.

Testable criteria:
- Every service/pod derives time from a single trusted source (NTP/chrony or a platform time-sync layer), not from a drifting local container clock.
- Clock offset between any two components stays within an acceptable threshold (e.g. < 1 s), confirmable via a clock-offset metric or probe.
- The time source is configured declaratively (not hard-coded) and identical across all cluster nodes.

Typically owned by: infrastructure, platform backend.

### 2.3 Control Program Requirements

#### GLI19-2.3.2 — Control-program self-verification

*shall* · maps to `2.3.2 [a–c]`

The Interactive Gaming System can verify that all critical control-program components are authentic copies of the approved components — at least every 24 hours and on demand — using a regulator-approved authentication mechanism, and signals any authentication failure.

Testable criteria:
- Self-verification runs at least once every 24 hours and on demand.
- Authentication uses a cryptographic hash producing a message digest of at least 128 bits (alternative methodologies only if regulator-approved).
- Coverage includes all critical control-program components: executables, libraries, gaming/ system configurations, OS files, components controlling required reporting, and DB elements affecting system operations.
- An authentication failure is indicated when any critical component is determined invalid.

Typically owned by: platform backend, infrastructure.

#### GLI19-2.3.3 — Independent third-party verification of critical components

*shall* · maps to `2.3.3`

Each critical control-program component can be verified via an independent third-party verification procedure that operates independently of any process or security software within the system.

Testable criteria:
- An out-of-band verification method exists for each critical control-program component (e.g. external signature/hash comparison against an approved manifest).
- The verification process runs independently of the system's own processes and security software (not self-attesting).

Typically owned by: platform backend, infrastructure.

### 2.4 Gaming Management

#### GLI19-2.4.1 — On-demand disable of gaming, themes, and logins

*shall* · maps to `2.4.1 [a–c]`

The Interactive Gaming System can, on demand, disable all gaming activity, an individual game theme/paytable or client version, and an individual player login.

Testable criteria:
- An operator can disable all gaming activity on demand (global kill-switch).
- An operator can disable an individual game theme/paytable or version (e.g. desktop, mobile, tablet) on demand.
- An operator can disable an individual player login on demand.

Typically owned by: platform backend.

#### GLI19-2.4.2 — Integrity constraints on live jackpot parameter changes

*shall* · maps to `2.4.2 [a–d]`

When jackpot parameters are modified after player contributions (without decommissioning the jackpot), the system enforces integrity constraints so the change cannot unfairly alter RTP, ceiling, trigger probability, or mystery-trigger fairness.

Testable criteria:
- RTP-affecting increment-rate changes do not take effect until the current jackpot is won.
- Ceiling changes are only to a value greater than the current payoff, or do not take effect until the current jackpot is won.
- Parameter changes do not alter the probability of triggering the current jackpot.
- For mystery-triggered jackpots, the hidden trigger amount is reselected within the range [current payoff, ceiling] and never yields an immediate or uncontributed trigger.

Typically owned by: platform backend.

#### GLI19-2.4.3 — Secure jackpot contribution transfer and correction

*shall* · maps to `2.4.3`

The system provides a secure, auditable means to transfer or combine contributions from a decommissioned jackpot (including its overflow/diversion pools), correct jackpot errors, or make other regulator-required jackpot adjustments.

Testable criteria:
- Decommissioned-jackpot contributions, including overflow and diversion pools, can be transferred/combined via a controlled, auditable operation (no silent loss).
- Jackpot error corrections and other regulator-required adjustments are supported and recorded in the audit trail with actor and reason.

Typically owned by: platform backend.

### 2.5 Player Account Management

#### GLI19-2.5.2.a — Legal-age gate + registration disclosures and consents

*shall* · maps to `2.5.2(a) [a.i–a.vi]`

Registration is permitted only for players of the jurisdiction's legal gaming age. The system collects PII and, during registration, enforces the age check, informs the player which fields are required and the consequences of not completing them, and captures the player's mandatory consents and affirmations.

Testable criteria:
- Registration rejects a birth date indicating an age below the jurisdiction's legal gaming age before the account is created (`player-account service`).
- The registration form marks which fields are required / optional and the consequences of not completing them.
- The player agrees to the terms & conditions and privacy policy as a condition of registration; consent is stored with a timestamp.
- The following affirmations are captured: prohibition on granting access to unauthorized persons, consent to monitoring/recording of account use, and confirmation that the supplied PII is accurate.

Typically owned by: player-account service, identity provider.

#### GLI19-2.5.2.b — Identity verification before first game

*shall* · maps to `2.5.2(b) [b.i–b.iii]`

Before a player is allowed to play, the system (directly or via a third-party provider) verifies the player's identity against a minimum attribute set, checks against exclusion lists and other prohibitions, and stores verification details securely.

Testable criteria:
- Play is unavailable until identity verification completes successfully (`identity provider` / `player-account service` gate).
- Verification authenticates at least legal name, residential address, and date of birth.
- A check is performed against operator/regulator exclusion lists and other prohibitions; on a match the account is not activated.
- Verification data is stored securely (encryption and/or restricted access).

Typically owned by: player-account service, identity provider.

#### GLI19-2.5.2.c — Account activation conditions

*shall* · maps to `2.5.2(c)`

An account becomes active only when all preconditions are simultaneously met: age and identity verified, the player is not on any exclusion list and not otherwise prohibited, the T&C/privacy policy are acknowledged, and registration is complete.

Testable criteria:
- The account state does not transition to `active` until all preconditions (age+identity verified, not excluded, T&C acknowledged, registration complete) are satisfied.
- An activation attempt with an unmet precondition is rejected or leaves the account pending.

Typically owned by: player-account service, identity provider.

#### GLI19-2.5.2.d — One active account per player

*shall* · maps to `2.5.2(d)`

A player may hold only one active account at a time, except where specifically authorized by the regulatory body.

Testable criteria:
- Creating a second active account for the same verified player is blocked (deduplication on identity attributes).
- An exception is possible only via a configured regulator authorization (not by default).

Typically owned by: player-account service, identity provider.

#### GLI19-2.5.2.e — Credential/account update with MFA

*shall* · maps to `2.5.2(e)`

The system lets a player update authentication credentials, registration information, and the account used for financial transactions; a multi-factor authentication process is applied to these operations.

Testable criteria:
- Updating authentication credentials / registration info / the financial-transaction account requires passing MFA (`identity provider`).
- Each such change is recorded in an audit log with a timestamp and the acting subject.

Typically owned by: player-account service, identity provider.

#### GLI19-2.5.3.a — Uniform authentication-failure message

*shall* · maps to `2.5.3(a)`

On unrecognized credentials the system shows a retry prompt with a message that is identical regardless of which credential was incorrect (no username/password enumeration leak).

Testable criteria:
- Failed login returns a generic "try again" message that does not reveal which credential was wrong.
- The message/behaviour is identical for unknown-user vs wrong-password cases.

Typically owned by: player-account service, identity provider.

#### GLI19-2.5.3.b — MFA for credential recovery/reset

*shall* · maps to `2.5.3(b)`

Retrieval or reset of forgotten authentication credentials requires a multi-factor authentication process.

Testable criteria:
- The forgotten-credentials flow enforces MFA before a reset/retrieval completes.

Typically owned by: player-account service, identity provider.

#### GLI19-2.5.3.d — Account lockout on suspicious activity + MFA unlock

*shall* · maps to `2.5.3(d)`

The system can lock an account on detected suspicious activity (e.g. three consecutive failed access attempts within thirty minutes) and requires MFA to unlock it.

Testable criteria:
- A lockout mechanism triggers on suspicious activity such as 3 consecutive failed attempts within a 30-minute window.
- Unlocking a locked account requires passing MFA.

Typically owned by: player-account service, identity provider.

#### GLI19-2.5.4.a — Inactivity timeout requires re-authentication

*shall* · maps to `2.5.4 + 2.5.4(a)`

After thirty minutes of inactivity on a Remote Player Device (or a regulator-defined period), the player must re-authenticate; no further games or financial transactions are permitted on that device until re-authentication succeeds.

Testable criteria:
- An inactivity timer (default 30 min, configurable per regulator) forces re-authentication.
- Games and financial transactions are blocked on the device until the player re-authenticates.

Typically owned by: identity provider, player-account service, player client.

#### GLI19-2.5.4.b — Simpler re-authentication with periodic full auth

*may* · maps to `2.5.4(b) [b.i, b.ii]`

A simpler re-authentication means (OS-level biometrics, PIN, etc.) may be offered on the device; it can be disabled by player/regulator preference, and full authentication is required at least once every thirty days (or a regulator-specified period).

Testable criteria:
- If a simpler re-auth means is offered, it can be disabled by player/regulator preference.
- Full authentication is enforced at least once every 30 days (or regulator period) on that device regardless of the simpler means.

Typically owned by: identity provider, player-account service, player client.

#### GLI19-2.5.5 — Correct enforcement of player/operator limitations and exclusions

*shall* · maps to `2.5.5 [a–c]`

The system correctly implements any player- and operator-imposed limitations and exclusions: the more restrictive limit always wins, and limits/exclusions cannot be bypassed by internal status events.

Testable criteria:
- Player- and operator-imposed limitations and exclusions are enforced as configured (deposit, wager, session, exclusion, etc.).
- A self-imposed limitation never overrides a more restrictive operator-imposed limitation; the more restrictive value takes priority.
- Limitations are not compromised by internal status events (e.g. self-imposed exclusion orders and revocations).

Typically owned by: risk / responsible-gaming service, player-account service.

#### GLI19-2.5.6.a — Confirmation/denial of every financial transaction

*shall* · maps to `2.5.6(a) [a.i–a.iii]`

Every initiated financial transaction returns an explicit confirmation or denial carrying the transaction type, value, and — on denial — a descriptive reason.

Testable criteria:
- Each deposit/withdrawal returns confirmation or denial with the transaction type and value.
- A denied transaction includes a descriptive reason why it did not complete as initiated.

Typically owned by: payments service, wallet / ledger service, player-account service.

#### GLI19-2.5.6.b — Deposits via auditable instruments

*may* · maps to `2.5.6(b)`

Deposits may be made via debit instrument, credit card, or other methods that produce a sufficient audit trail.

Testable criteria:
- Supported deposit methods each produce a sufficient, retrievable audit trail per transaction.

Typically owned by: payments service, wallet / ledger service, player-account service.

#### GLI19-2.5.6.c — Funds not wagerable until authorized

*shall* · maps to `2.5.6(c)`

Deposited funds are not available for wagering until received from the issuer or an authorization number is provided; the authorization number is retained in an audit log.

Testable criteria:
- Balance available for wagering excludes funds not yet received/authorized from the issuer.
- The issuer authorization number is stored in an audit log against the transaction.

Typically owned by: payments service, wallet / ledger service, player-account service.

#### GLI19-2.5.6.d — Payouts only to the player's own verified account/address

*shall* · maps to `2.5.6(d)`

Payments from an account are made only to a financial account in the player's name (or to the player at their registered address via a secure method); the name/address must match the player's registration details.

Testable criteria:
- Withdrawal destinations are validated against the player's registered identity (name/address match); mismatches are blocked.
- Non-account payouts use a secure delivery method to the registered address only.

Typically owned by: payments service, wallet / ledger service, player-account service.

#### GLI19-2.5.6.e — Over-limit transaction requires clear reduced-amount notification

*shall* · maps to `2.5.6(e)`

If a transaction would exceed an operator/regulator limit, it may be processed only after the player is clearly notified that they have withdrawn or deposited less than requested.

Testable criteria:
- An over-limit transaction is either blocked or processed at the capped amount with a clear notification of the reduced amount to the player.

Typically owned by: payments service, wallet / ledger service, player-account service.

#### GLI19-2.5.6.f — No fund transfers between player accounts

*shall* · maps to `2.5.6(f)`

It is not possible to transfer funds directly between two player accounts.

Testable criteria:
- No API/flow permits a player-to-player balance transfer.
- Ledger design prevents account-to-account transfers outside sanctioned deposit/withdrawal paths.

Typically owned by: payments service, wallet / ledger service, player-account service.

#### GLI19-2.5.7 — Player transaction log / account statement on request

*shall* · maps to `2.5.7 [a.i–a.v, b.i–b.iii]`

On request the system provides a player a transaction log / account statement (at least the past year or a requested/regulator period) sufficient to reconcile against the player's own records, covering both financial transactions and game history.

Testable criteria:
- Financial transactions are listed, time-stamped with a unique transaction ID: deposits, withdrawals, incentive credits added/removed (outside game wins), manual adjustments/refunds, and any non-wager purchases.
- Game history is listed by game theme: theme name and game type, total amount wagered (incl. incentive credits), and total amount won (incl. incentive credits/prizes and progressive/incrementing jackpots).
- The statement is retrievable for at least the past year or a player-requested / regulator period.

Typically owned by: player-account service, wallet / ledger service, platform backend.

#### GLI19-2.5.8 — Player loyalty program integrity

*shall* · maps to `2.5.8 [a–c]`

Where player loyalty programs are supported, awards are equally available to all players who reach the qualification level, redemptions are secure transactions that automatically debit the points balance, and all loyalty-points transactions are recorded.

Testable criteria:
- All loyalty awards are equally available to every player meeting the defined qualification level (no discriminatory eligibility).
- Redemption is a secure, atomic transaction that debits the points balance by the value of the redeemed prize.
- Every loyalty-points transaction (accrual, redemption, adjustment) is recorded by the system.

Typically owned by: content service, player-account service, wallet / ledger service.

### 2.6 Player Software

#### GLI19-2.6.2 — Player Software carries identity and version

*shall* · maps to `2.6.2`

The Player Software contains sufficient information to identify the software and its version.

Testable criteria:
- The Player Software exposes information sufficient to identify the software and its version.

Typically owned by: player client.

#### GLI19-2.6.3 — Authenticate critical software components on load

*shall* · maps to `2.6.3`

For locally installed Player Software, all critical software components can be authenticated each time the software is loaded (and on demand where supported); a failed authentication prevents gaming operations and shows an error.

Testable criteria:
- All critical software components (gaming rules, pay tables, comms-control elements, and other components needed for proper operation) are authenticated each time the software is loaded, and on demand where supported by the system.
- On a program mismatch or authentication failure, the software prevents gaming operations and displays an appropriate error message.
- The verification mechanism follows industry-standard security practices (evaluated case-by-case by the regulator/independent test laboratory).

Typically owned by: player client, platform backend.

#### GLI19-2.6.4 — Secure communications and halt on loss

*shall* · maps to `2.6.4`

Player Software communicates only with authorized components over secure communications; if communication with the Interactive Gaming System is lost, it prevents further gaming operations and displays an error.

Testable criteria:
- The software communicates only with authorized components, and only through secure communications.
- On loss of communication between the system and the Remote Player Device, the software prevents further gaming operations and displays an appropriate error message (detection on next communication attempt is permissible).

Typically owned by: player client, platform backend.

#### GLI19-2.6.5.client-safety — Client software behaves safely on the device

*shall* · maps to `2.6.5 [a–f]`

The Player Software respects device security and privacy: no peer data transfer, no tampering with device security, no unnecessary ports, no integrity-altering extras, no volume override, and no sensitive-data storage.

Testable criteria:
- Players cannot use the software to transfer data to one another, other than chat (text/voice/video) and approved files (e.g. profile pictures, photos). [a]
- The software does not automatically disable virus scanners/detection programs or alter device firewall rules to open blocked ports. [b]
- The software does not access any TCP/UDP ports (automatically or by prompting) that are unnecessary for device-to-server communication. [c]
- Any additional non-gaming functionality does not alter the software's integrity in any way. [d]
- The software cannot override the Remote Player Device's volume settings. [e]
- The software is not used to store sensitive information; autocomplete/password caching that fills the password field is recommended to be disabled by default. [f]

Typically owned by: player client, platform backend, game-session orchestrator.

#### GLI19-2.6.5.server-authority — No game logic on the client; server generates all outcomes

*shall* · maps to `2.6.5(g)`

The Player Software contains no logic to generate any game result; all critical functions, including generation of game outcome, are produced by the Gaming Platform independently of the Remote Player Device.

Testable criteria:
- The software contains no logic used to generate the result of any game.
- All critical functions, including the generation of any game outcome, are generated by the Gaming Platform and are independent of the Remote Player Device.

Typically owned by: player client, platform backend, game-session orchestrator.

#### GLI19-2.6.6 — Detect device incompatibility before gaming

*shall* · maps to `2.6.6`

During installation/initialization and before gaming operations, the Player Software detects any incompatibility or resource limitation with the Remote Player Device that would prevent proper operation, and on detection prevents gaming and shows an error.

Testable criteria:
- Before commencing gaming operations, the software detects incompatibilities or resource limitations (software version, minimum specs, browser type/version, plug-in version, etc.).
- If any are detected, the software prevents gaming operations and displays an appropriate error message.

Typically owned by: player client.

### 2.7 Location Requirements

#### GLI19-2.7.2 — Detect and block location-spoofing techniques

*shall* · maps to `2.7.2 [a–e]`

The system incorporates a mechanism to detect and block tools that circumvent location detection (remote desktop, rootkits, virtualization, VPN/proxy, jailbroken devices, MITM).

Testable criteria:
- Detects and blocks location-data fraud (fake-location apps, VMs, remote-desktop programs, etc.) prior to initiating each game. [a]
- Examines the IP Address on each device connection to ensure a known VPN or proxy service is not in use. [b]
- Detects and blocks devices indicating system-level tampering (rooting, jailbreaking, etc.). [c]
- Stops "Man-In-The-Middle" attacks or similar techniques and prevents code manipulation. [d]
- Uses detection/blocking mechanisms verifiable to an application level. [e]

Typically owned by: risk / responsible-gaming service, platform backend.

#### GLI19-2.7.3 — Track player location on a private network

*shall* · maps to `2.7.3 [a–b]`

Where gaming occurs over a private network, the system tracks all connected players' locations using one of two permitted methods.

Testable criteria:
- One of the following is implemented:
- a location-detection service/application where each player passes a location check prior to initiating each game, meeting the §2.7.4 public-network requirements; [a] OR
- a real-time location component that detects when any player is no longer within the permitted boundary and prevents further games (e.g. directional antennas, Bluetooth sensors; evaluated case-by-case). [b]

Typically owned by: risk / responsible-gaming service, platform backend.

#### GLI19-2.7.4.checks — Location-check cadence, blocking, and logging

*shall* · maps to `2.7.4(a) [a.i–a.ii]`

The system detects and dynamically monitors player location on a public network, checks location at login and on defined triggers, blocks out-of-boundary play, and logs violations.

Testable criteria:
- A location detection service reasonably detects and dynamically monitors the player's location and enables blocking of unauthorized attempts to play.
- Each player passes a location check prior to initiating the first game after login on a specific device; subsequent checks occur prior to initiating games after an IP Address change, after 30 minutes since the previous check, or as otherwise specified by the regulator. [a]
- If the check indicates the player is outside the permitted boundary, or cannot locate the player, the game does not initiate and the player is notified. [a.i]
- A time-stamped log entry is recorded on any location violation, including the unique player ID and the detected location. [a.ii]

Typically owned by: risk / responsible-gaming service, platform backend.

#### GLI19-2.7.4.geolocation — Geolocation-method accuracy and integrity

*shall* · maps to `2.7.4 [b–f]`

The geolocation method produces a physical location with a confidence radius inside the permitted boundary, uses accurate data sources, controls buffer-zone overlap, uses audited boundary polygons, and flags impossible-travel patterns.

Testable criteria:
- The method provides the player's physical location and an associated confidence radius; the radius is entirely within the permitted boundary. [b]
- Accurate location data sources (Wi-Fi, GSM, GPS, etc.) are used; where only an IP Address is available, a registered mobile device's location may support it if the two devices are near one another [c.i], and (if the regulator allows) carrier-based mobile location may be used when no other sources exist. [c.ii]
- The method controls whether the accuracy radius may overlap or exceed defined buffer zones or the permitted boundary. [d]
- Boundary polygons based on regulator-audited maps, with location data overlaid onto them, are used to account for mapping/geospatial variances. [e]
- The method monitors and flags for investigation any games played by a single account from geographically inconsistent (impossible-travel) locations. [f]

Typically owned by: risk / responsible-gaming service, platform backend.

### 2.8 Information to be Maintained

#### GLI19-2.8.1 — Maintain, back up, time-stamp, and export recorded data

*shall* · maps to `2.8.1 [a–b]`

The system maintains and backs up all recorded data covered by §2.8, time-stamps it from the system clock, and can export it for analysis/auditing.

Testable criteria:
- All recorded data discussed in §2.8 can be maintained and backed up.
- The system clock is used for all time stamping. [a]
- A mechanism exists to export the data for analysis and auditing/verification (e.g. CSV, XLS). [b]

Typically owned by: platform backend, infrastructure.

#### GLI19-2.8.2 — Maintain per-game play record

*shall* · maps to `2.8.2 [a–r]`

For each game played, the system maintains and backs up a complete game-play record sufficient to reconstruct the game.

Testable criteria:
- Per game (per player for multi-player games), the record includes as applicable: date/time [a]; denomination (multi-denom) [b]; final-outcome display (graphic or text) [c]; funds available at start/end [d]; total wagered incl. incentive credits [e]; total won incl. incentive credits/prizes and progressive/incrementing jackpots [f]; non-wager purchases during play [g]; rake/commission/fees [h]; results of player choices [i]; results of intermediate phases (double-up/gamble, bonus/feature) [j]; indication a jackpot was awarded [k]; player advice for skill games [l]; jackpot contributions [m]; relevant location information [n]; current game status (in progress/complete/interrupted/ cancelled) [o]; unique game cycle ID and/or gaming session ID [p]; unique game theme/paytable ID [q]; unique player ID [r].

Typically owned by: platform backend, game-session orchestrator.

#### GLI19-2.8.3 — Maintain per-theme/paytable record and aggregates

*shall* · maps to `2.8.3 [a–q]`

For each game theme/paytable available for play, the system maintains and backs up its configuration, theoretical RTP, and play/financial aggregates.

Testable criteria:
- Per theme/paytable, the record includes as applicable: unique theme/paytable ID [a]; configuration data (denominations, wager categories) [b]; date/time made available [c]; theoretical RTP % [d]; number of games played [e]; total wagers (excl. incentive credits and re-wagered intermediate winnings) [f]; total paid on winning wagers (excl. incentive credits/prizes and jackpots) [g]; total paid as jackpots [h]; total incentive credits wagered [i]; total incentive credits/prizes won [j]; total wagers voided/cancelled incl. incentive credits [k]; total non-wager purchases for the game [l]; times each jackpot awarded [m]; double-up/gamble aggregates (amount wagered/won, games played/won) [n]; total rake/commission/fees [o]; current status (active/disabled/decommissioned) [p]; scheduled/actual decommission date/time (blank until known) [q].

Typically owned by: platform backend, game-session orchestrator.

#### GLI19-2.8.4 — Maintain per-contest/tournament record

*shall* · maps to `2.8.4 [a–h]`

For each contest/tournament, the system maintains and backs up identification, timing, participants, entry fees, rankings, winnings, fees, and status.

Testable criteria:
- Per contest/tournament, the record includes as applicable: name/identification [a]; date/time occurred/will occur [b]; participating theme/paytable ID(s) [c]; per registered player — unique player ID, entry fee collected (incl. incentive credits) and date, scorings/rankings, winnings paid (incl. incentive credits) and date [d]; total entry fees collected [e]; total winnings paid [f]; total rake/commission/fees [g]; current status (in progress/complete/interrupted/cancelled) [h].

Typically owned by: platform backend, game-session orchestrator.

#### GLI19-2.8.5 — Maintain complete player-account record

*shall* · maps to `2.8.5 [a–m]`

For each player account, the system maintains and backs up identity, PII (encrypted where required), verification, balances, exclusions, full financial-transaction history, and status.

Testable criteria:
- Unique player ID and username [a].
- PII: registration data (legal name, residential address, DOB) [b.i]; encrypted PII — government ID, authentication credential, personal financial info [b.ii].
- Date/method of identity verification incl. credential description and expiry [c].
- Date of agreement to T&C and privacy policy [d].
- Account details and current balance incl. incentive credits, with restricted/expiring incentive credits maintained separately [e].
- Previous accounts and reason for de-activation [f].
- Date/method of registration (remote vs on-site) [g].
- Date/time of every access by any person incl. IP Address [h].
- Exclusions/limitations: request date/time, description/reason, type, start date, end date [i].
- Financial transaction info per transaction: type; date/time; unique transaction ID; amount; total account balance before/after transaction [j.v]; fees; user who handled it; status; deposit/ withdrawal method; deposit authorization number; relevant location [j].
- Persistence-game info (theme/paytable ID, achievements, last save point) if supported [k].
- Identifier info (theme/paytable ID, date/time, tx ID, use criteria, action/alteration made) if supported [l].
- Current account status (active/inactive/closed/excluded) [m].

Typically owned by: player-account service, wallet / ledger service, payments service, identity provider.

#### GLI19-2.8.6 — Maintain per-incentive record

*shall* · maps to `2.8.6 [a–i]`

For each incentive offered, the system maintains and backs up its ID, availability, balances, issuance/redemption/expiry/adjustment totals, and status.

Testable criteria:
- Per incentive, the record includes as applicable: unique incentive offer ID [a]; date/time made available [b]; current balance for incentive awards [c]; total issued [d]; total redeemed [e]; total expired [f]; total adjustments [g]; current status (active/disabled/decommissioned) [h]; scheduled/ actual decommission date/time (blank until known) [i].

Typically owned by: platform backend.

#### GLI19-2.8.7 — Maintain per-jackpot record and parameters

*shall* · maps to `2.8.7 [a–j]`

For each jackpot, the system maintains and backs up its identity, participating themes, current/reset values, contribution pools, configurable parameters, and status.

Testable criteria:
- Per jackpot, the record includes as applicable: unique jackpot ID [a]; date/time made available [b]; participating theme/paytable ID(s) [c]; player ID(s) if player-tied [d]; current value (payoff) [e]; other contribution pools — overflow value, diversion-pool value [f]; reset value if different from startup [g]; configurable parameters where applicable — startup value, increment %, ceiling, secondary increment, hidden increment, diversion limit, odds, time limit, and any info needed to reconcile the jackpot [h]; current status (active/disabled/decommissioned) [i]; scheduled/actual decommission date/time (blank until known) [j].

Typically owned by: game-session orchestrator, platform backend.

#### GLI19-2.8.8 — Maintain significant-event audit log

*shall* · maps to `2.8.8 [a–q]`

The system maintains and backs up significant-event information covering security, availability, large financial activity, configuration/data changes, and player-account events.

Testable criteria:
- The log captures as applicable: failed account access attempts incl. IP [a]; program error / authentication mismatch [b]; significant unavailability of any critical component [c]; large wins (single/aggregate) above regulator threshold incl. game-play info [d]; large wagers likewise [e]; system voids/overrides/corrections [f]; changes to live data files outside normal execution [g]; changes to the download data library (add/change/delete software) [h]; changes to OS/DB/network/app policies and parameters [i]; changes to date/time on master time server [j]; changes to game-theme parameters (rules, payout schedules, rake %, paytables) [k]; changes to jackpot parameters [l]; changes to incentive parameters [m]; player-account management events — balance adjustments, PII/ sensitive-info changes, account deactivation, large financial transactions above threshold, negative balances [n]; irrecoverable loss of PII/sensitive info [o]; any activity requiring user intervention outside normal operation [p]; other significant/unusual events per the regulator [q].

Typically owned by: platform backend, risk / responsible-gaming service, identity provider, infrastructure.

#### GLI19-2.8.9 — Maintain per-user (operator) access record

*shall* · maps to `2.8.9 [a–i]`

For each user (operator/employee) account, the system maintains and backs up identity, role/functions, lifecycle timestamps, group membership, and status.

Testable criteria:
- Per user account, the record includes: employee name and title/position [a]; user identification [b]; full list/description of functions each group or user may execute [c]; account creation date/time [d]; last access date/time incl. IP [e]; last password-change date/time [f]; disable/deactivate date/time [g]; group membership if applicable [h]; current status (active/inactive/closed/suspended) [i].

Typically owned by: identity provider, platform backend.

### 2.9 Reporting Requirements

#### GLI19-2.9.1 — Generate reports on demand with required intervals and headers

*shall* · maps to `2.9.1 [a–b]`

The system generates the information needed to compile regulator-required reports, on demand and at required intervals, each report carrying identifying headers, no-activity indication, and clearly labeled fields.

Testable criteria:
- Reporting information is available on demand, daily, and for other required intervals (MTD, YTD, LTD, etc.). [a]
- Each report contains: operator name/identifier, report title, selected interval, and generation date/time [b.i]; a "No Activity" (or similar) indication when no data appears for the period [b.ii]; labeled fields understandable per their function [b.iii].

Typically owned by: platform backend, back office.

#### GLI19-2.9.2 — Game-performance report per theme/paytable

*shall* · maps to `2.9.2 [a–k]`

The system provides game-performance data per theme/paytable including theoretical vs actual RTP for house-banked games, play counts, and financial aggregates.

Testable criteria:
- Per theme/paytable, the report provides as applicable: game theme name and type [a]; date/time made available [b]; for house-banked games — theoretical RTP % [c.i] and actual RTP % [c.ii]; number of games played [d]; total wagers collected with separate incentive-credit amounts [e]; total winnings paid with separate incentive-credit/prize amounts [f]; total wagers voided/cancelled with separate incentive-credit amounts [g]; total rake/commission/fees [h]; total funds remaining in interrupted games with separate incentive-credit amounts [i]; theme/paytable ID [j]; current status [k].

Typically owned by: platform backend, game-session orchestrator, back office.

#### GLI19-2.9.3 — Operator-liability report

*shall* · maps to `2.9.3 [a–b]`

The system provides operator-liability data: total funds held for player accounts and any operational funds covering other operator liability.

Testable criteria:
- Total amount held by the operator for player accounts. [a]
- Any operational funds used to cover all other operator liability, if defined by the regulator. [b]

Typically owned by: wallet / ledger service, platform backend, back office.

#### GLI19-2.9.4 — Large-jackpot-payout report

*shall* · maps to `2.9.4 [a–g]`

For jackpot payouts exceeding a regulator-defined value, the system provides jackpot ID, winner, winning game context, trigger time, payoff, and the user(s) who processed/confirmed the win.

Testable criteria:
- Per qualifying jackpot payout, the report provides as applicable: unique jackpot ID [a]; winning player ID [b]; winning game theme/paytable ID [c]; winning game cycle ID and/or gaming session ID [d]; date/time of jackpot trigger [e]; jackpot hit and payoff amount [f]; identification of user(s) who processed and/or confirmed the win [g].

Typically owned by: game-session orchestrator, platform backend, back office.

#### GLI19-2.9.5 — Significant-event / alteration report

*shall* · maps to `2.9.5 [a–f]`

For each significant event or alteration, the system provides time, event/component, responsible user(s), reason/description, and before/after values.

Testable criteria:
- Per significant event/alteration, the report provides as applicable: date/time [a]; event/component identification [b]; identification of user(s) who performed and/or authorized it [c]; reason/ description including data or parameter altered [d]; data/parameter value before alteration [e]; data/parameter value after alteration [f].

Typically owned by: platform backend, risk / responsible-gaming service, back office.

## 4. Game Requirements

### 4.3 Gaming Session Requirements

#### GLI19-4.3.3.b — Wager debit with no negative balance

*shall* · maps to `4.3.3(b)`

Amounts wagered are debited from the player's balance at wager time; a wager that would cause a negative balance is rejected. Applies to all games (own and third-party).

Testable criteria:
- Each wager debits the player balance at the start of / during the game cycle.
- A wager that would result in a negative balance is not accepted (insufficient-funds rejection).

Typically owned by: game-session orchestrator, game aggregation layer, wallet / ledger service.

#### GLI19-4.3.3.d — Award settlement to balance on game-cycle completion

*shall* · maps to `4.3.3(d)`

On game-cycle completion (all wagered funds lost, or the final transfer occurs), every award value is credited to the player's balance, except merchandise and large payouts handled per the regulatory body. Applies to all games.

Testable criteria:
- The award value at game-cycle end is credited to the player balance.
- Merchandise / large payouts are handled per regulator policy (may be excluded from automatic credit and settled manually).

Typically owned by: game-session orchestrator, game aggregation layer, wallet / ledger service.

#### GLI19-4.3.3.e — No new game before the current cycle completes

*shall* · maps to `4.3.3(e)`

A new game cannot start within the same session before the current game cycle completes and the available funds and game history are updated — unless the start action terminates the current game in an orderly manner (limited operator-defined exceptions, e.g. pending large-payout consideration).

Testable criteria:
- Starting a new game is blocked until the prior cycle settles and balance/history are updated, save for the standard's orderly-termination and large-payout exceptions.

Typically owned by: game-session orchestrator, game aggregation layer, wallet / ledger service.

#### GLI19-4.3.5.d — Restricted incentive credits wagered first

*shall* · maps to `4.3.5(d)`

Where restricted incentive credits and unrestricted player funds are combined on one meter/balance, restricted credits are wagered first (as game rules allow) before any unrestricted funds. Applies to all games.

Testable criteria:
- Wagering consumes restricted incentive credits before unrestricted player funds.

Typically owned by: game-session orchestrator, player client, wallet / ledger service.

### 4.6 Game Fairness

#### GLI19-4.6.1.b — No exploitable hidden source code

*shall* · maps to `4.6.1(b)`

Games contain no hidden source code a player could leverage to circumvent the rules or intended design behaviors; intentional undocumented "discovery features" are permitted.

Testable criteria:
- No hidden source code exists that can be leveraged by a player to circumvent the rules of play and/or intended game-design behaviors.
- Intentional "discovery features" (by design, though possibly undocumented/unknown to the player) are not precluded.

Typically owned by: game-session orchestrator, game aggregation layer.

### 4.14 Game Recall

#### GLI19-4.14.1 — Provide player-facing game recall

*shall* · maps to `4.14.1`

A game-recall facility is provided to the player (re-enactment or description) and clearly indicates it is a replay of the previous game.

Testable criteria:
- A game-recall facility is provided to the player, either as a re-enactment or by description.
- The facility clearly indicates that it is a replay of the previous game.

Typically owned by: game-session orchestrator, game aggregation layer.

#### GLI19-4.14.2 — Game recall reconstructs full game outcome and actions

*shall* · maps to `4.14.2 [a–l]`

Game recall (graphical, textual, video, or other means) enables full and accurate reconstruction of game outcome and player actions, displaying all applicable last-play fields.

Testable criteria:
- Recall content enables full and accurate reconstruction of game outcome and/or player actions (currency may be shown in place of credits), and displays as applicable:
- date and time the game was played; [a]
- denomination played (for multi-denomination games); [b]
- the display associated with the final outcome (graphically or via clear text); [c]
- funds available for wagering at start and/or end of play; [d]
- total amount wagered, including any incentive credits; [e]
- total amount won, including incentive credits/prizes [f.i] and progressive/incrementing jackpots; [f.ii]
- any non-wager purchase between start and end of play; [g]
- rake, commission, or fees collected; [h]
- the results of any player choices involved in the outcome; [i]
- the results of any intermediate phases (double-up/gamble, bonus/feature); [j]
- an indication that a progressive/incrementing jackpot was awarded, if won; [k]
- any player advice offered for games with skill. [l]

Typically owned by: game-session orchestrator, game aggregation layer.

#### GLI19-4.14.3 — Recall at least last 50 bonus/feature events

*shall* · maps to `4.14.3`

Game recall reflects at least the last 50 events of completed bonus/feature games, each with its corresponding outcome.

Testable criteria:
- Game recall reflects at least the last 50 events of completed bonus/feature games.
- Where a bonus/feature consists of x events each with separate outcomes, each of the x events (up to 50) is displayed with its corresponding outcome, regardless of win or loss.

Typically owned by: game-session orchestrator, game aggregation layer.

### 4.15 Disable Requirements

#### GLI19-4.15.1 — Allow in-progress games to conclude on disable

*shall* · maps to `4.15.1`

When a game is disabled while in progress, players may conclude the current game (including bonus rounds, double-up/gamble, and wager-related features); once fully concluded the game is no longer accessible.

Testable criteria:
- On disable, all players playing that game are permitted to conclude their current game in play — bonus rounds, double-up/gamble, and other wager-related features conclude immediately or the next time the game becomes available.
- Once the game has fully concluded, it is no longer accessible to a player.

Typically owned by: game-session orchestrator, game aggregation layer.

### 4.16 Interrupted Games

#### GLI19-4.16.2 — Hold wagers and reflect held funds

*shall* · maps to `4.16.2`

Wagers on a continuable interrupted game are held by the platform until the game completes, and player accounts reflect any funds held in interrupted games.

Testable criteria:
- Wagers associated with a continuable interrupted game are held by the Gaming Platform until the game completes.
- Player accounts reflect any funds held in interrupted games.

Typically owned by: game-session orchestrator, game aggregation layer, wallet / ledger service.

#### GLI19-4.16.3.mechanism — Provide completion mechanism; resolve before replay

*shall* · maps to `4.16.3 (opening)`

The platform provides a mechanism to complete an interrupted game, and the interrupted game is resolved before the player participates in another instance of the same game.

Testable criteria:
- A mechanism is provided for a player to complete an interrupted game.
- An interrupted game is resolved before the player is permitted to participate in another instance of the same game.

Typically owned by: game-session orchestrator, game aggregation layer, wallet / ledger service.

#### GLI19-4.16.3.multi-player — Multi-player auto-completion

*shall* · maps to `4.16.3(c) [c.i–c.iv]`

For multi-player games where a player cannot complete a required action in time, the platform completes the game on their behalf per the rules, updates histories/balances, discloses which decisions it made, and isolates the delay from other players.

Testable criteria:
- The platform completes the game on behalf of the player per the game rules/terms. [c.i]
- Game history and credit meters / player account balances are updated accordingly. [c.ii]
- Results are available to the player and indicate which decisions (if any) were made by the platform on their behalf. [c.iii]
- One player not completing an action in the required time does not impact any other players in the same session with respect to completing the game and being credited/debited. [c.iv]

Typically owned by: game-session orchestrator, game aggregation layer, wallet / ledger service.

#### GLI19-4.16.3.single-player — Single-player resolution paths

*shall* · maps to `4.16.3 [a–b]`

For games needing no further input the game returns to a completed state; for single-player games needing input the player is returned to the pre-interruption state to complete the game (unless superseding recovery rules are disclosed).

Testable criteria:
- Where no player input is required, the game may return to a completion state, provided the game history and credit meter / player account balance reflect a completed game. [a]
- For single-player games requiring input, the game returns the player to the game state immediately prior to the interruption and allows completion, unless superseding game rules/terms for recovery are disclosed to the player. [b]

Typically owned by: game-session orchestrator, game aggregation layer, wallet / ledger service.

## B. Operational Audit for Technical Security Controls

### B.2 System Operation and Security

#### GLI19-B.2.1 — Documented system security and operations procedures

*shall* · maps to `B.2.1 [a–e]`

The operator documents and follows procedures to monitor critical components/data, maintain security, handle security incidents, monitor performance, and investigate/resolve malfunctions.

Testable criteria:
- Monitor critical components and data transmission across the whole system (incl. third-party services) for integrity, reliability, and accessibility. [a]
- Maintain system security for secure, reliable communications, protected from hacking/tampering. [b]
- Define, monitor, document, report, investigate, respond to, and resolve security incidents incl. breaches and suspected/actual hacking/tampering. [c]
- Monitor/adjust resource consumption and log system performance with reporting. [d]
- Investigate, document, and resolve malfunctions: determine cause [e.i]; review records/logs [e.ii]; repair/replace the component [e.iii]; verify integrity before restoring [e.iv]; file a regulator incident report with dates/times/reason and restore time [e.v]; void plays/pays if full recovery is impossible [e.vi].

Typically owned by: operator governance, infrastructure, platform backend.

#### GLI19-B.2.3 — Logical access secured by controlled credentials and RBAC

*shall* · maps to `B.2.3 [a–j]`

The system is logically secured with individual credentials, encrypted/hashed credential storage, strong reset methods, multi-level access with separation of duties, suspect-account lockout, and access logging.

Testable criteria:
- Each user has an individual credential provisioned by a formal process with periodic review of access rights; generic-account use is limited and documented. [a]
- Credential records are maintained (manual or automated) with forced credential changes. [b]
- Stored credentials are encrypted or hashed to current standards (ISO/IEC 19790, FIPS 140-2, or equivalent). [c]
- A credential-reset fallback (e.g. forgotten password) is at least as strong as the primary and uses MFA. [d]
- Lost/compromised/terminated-user credentials are deactivated/secured/destroyed ASAP. [e]
- Multiple security access levels control view/change/delete of critical files; procedures assign/ review/modify/remove rights with separation of duties [f.i], limited critical-parameter access [f.ii], and enforced credential parameters (min length, expiry) [f.iii].
- Suspect accounts are flagged/locked: admin notification + lockout/audit entry after max 3 failed attempts [g.i]; flagging of possibly-stolen credentials [g.ii]; invalidating accounts and transferring stored info to a new account [g.iii].
- All logical access attempts are recorded in a secure log. [h]
- Utility programs that can override controls are restricted and tightly controlled. [i]
- Connection-time restrictions (e.g. session timeouts) secure high-risk applications such as remote access. [j]

Typically owned by: identity provider, infrastructure, operator governance.

#### GLI19-B.2.4 — Verified, ephemeral user-session authorization

*shall* · maps to `B.2.4 [a–d]`

A secure mechanism verifies authorized access to critical components; equipment identification is documented; authorization info is fetched at request time (not stored on the component); and session authorization is random, in-memory, and removed at session end.

Testable criteria:
- A secure, controlled mechanism verifies critical-component access by authorized personnel on demand and regularly. [a]
- Automated equipment-identification methods (when used) are documented and included in access-rights review. [b]
- Authorization info communicated for identification is obtained at request time from the system and not stored on the system component. [c]
- Session authorization info is always created randomly, in memory, and removed after the session ends. [d]

Typically owned by: identity provider, platform backend, infrastructure.

#### GLI19-B.2.5 — Prevent user-initiated server programming and mobile code

*shall* · maps to `B.2.5`

The system is secure against user-initiated programming that could modify the database (authorized admin maintenance excepted) and is protected from unauthorized mobile-code execution.

Testable criteria:
- The system prevents any user-initiated programming capabilities on the server that could modify the database (authorized network/system admin maintenance and troubleshooting with sufficient rights is acceptable).
- The server is protected from the unauthorized execution of mobile code.

Typically owned by: infrastructure, platform backend.

#### GLI19-B.2.6 — Verify production components match approved versions

*shall* · maps to `B.2.6 [a–e]`

Procedures verify that production critical control-program components are identical to regulator-approved versions via signature comparison at defined triggers, with unalterable results retained, failure notification, and a response process.

Testable criteria:
- Signatures of critical control-program components are gathered from production via an approved process upon install/update [a.i], on power-up/recovery [a.ii], at least every 24 hours [a.iii], and on demand [a.iv].
- The process compares current production signatures against current approved-version signatures. [b]
- Output includes current and expected signatures, stored unalterably; recorded in a log/report retained 90 days (or as specified) [c.i]; accessible to the regulator for analysis [c.ii]; part of records recoverable in a disaster/failure [c.iii].
- Any verification failure triggers authentication-failure notification to the operator and regulator as required. [d]
- A process exists to respond to authentication failures — determine cause and perform corrections/ reinstallations timely. [e]

Typically owned by: infrastructure, platform backend, operator governance.

#### GLI19-B.2.7 — Report retention with versioning and audit trail

*shall* · maps to `B.2.7 [a–i]`

An electronic document retention system for reports maintains version history with unique signatures, a complete change log, indexing, access restriction, admin audit trail, logical/physical security, and redundancy.

Testable criteria:
- Maintains the original version plus all subsequent versions for alterable-format reports. [a]
- Maintains a unique signature for each report version, including the original. [b]
- Retains and reports a complete change log (who and when). [c]
- Provides complete indexing to locate reports (generation date/time, generating app/system, title/ description, generating user, other useful info). [d]
- Limits modify/add access via logical security of specific accounts. [e]
- Provides a complete audit trail of all admin account activity. [f]
- Is secured with logical security measures (accounts, event logging, version control). [g]
- Is physically secured with the other critical components. [h]
- Prevents report-availability disruption and data loss via redundancy and backup. [i]

Typically owned by: platform backend, infrastructure, back office.

#### GLI19-B.2.8 — Account for and securely manage assets

*shall* · maps to `B.2.8 [a–g]`

All physical/logical assets housing/processing/communicating sensitive information are accounted for, with procedures for onboarding/removal, secure disposal, acceptable use, ownership/ classification, periodic inventory reconciliation, and media sanitization before disposal/reuse.

Testable criteria:
- Procedures exist for adding new assets and removing assets from service. [a]
- Assets are disposed of securely and safely via documented procedures. [b]
- An acceptable-use policy covers system and operating-environment assets. [c]
- Each asset's designated owner ensures classification (confidentiality/integrity/accountability/ availability) [d.i] and defines/periodically reviews access restrictions and classifications [d.ii].
- Recorded asset accountability is reconciled with actual assets at least annually, with action on discrepancies. [e]
- Copy protection, if used, is fully documented to the ITL [f.i] or individually verifiable by an approved methodology [f.ii].
- Before disposal/reuse, storage media are checked to ensure licensed software and PII/sensitive info is removed or securely overwritten (not just deleted). [g]

Typically owned by: operator governance, infrastructure.

#### GLI19-B.2.9 — Maintain a Critical Asset Register

*shall* · maps to `B.2.9 [a–g]`

A Critical Asset Register documents every asset affecting system functionality or PII handling, including component inter-relationships/dependencies and per-asset classification relevance codes (1–3) for confidentiality, integrity, availability, and accountability.

Testable criteria:
- The CAR includes hardware and software components and their inter-relationships/dependencies.
- Per asset it documents: name/definition [a]; unique ID [b]; version number [c]; identifying characteristics (component/database/VM/hardware) [d]; owner [e]; geographical location of hardware [f]; and relevance codes (1=none, 2=some, 3=substantial) on the asset's role in confidentiality [g.i], integrity [g.ii], availability [g.iii], and accountability [g.iv].

Typically owned by: operator governance, infrastructure.

### B.3 Data Integrity

#### GLI19-B.3.1 — Layered data security for PII and sensitive data

*shall* · maps to `B.3.1 [a–j]`

A layered security approach secures PII and sensitive data (accounting, reporting, significant-event, player/gaming) against alteration, tampering, and unauthorized access, with input validation, encryption, restricted access, network segregation, and power-independent persistence.

Testable criteria:
- Appropriate data-handling methods validate input and reject corrupt data. [a]
- The number of workstations that can access critical apps/databases is limited. [b]
- Encryption/password protection/equivalent secures data files/directories; if not encrypted, viewing is restricted with duty segregation and access monitoring/recording. [c]
- Normal equipment operation has no options/mechanisms that may compromise the data. [d]
- No equipment has a mechanism whereby an error auto-clears the data. [e]
- Equipment holding data in memory does not allow removal unless the data was first transferred to the database/secured component. [f]
- PII/sensitive info is stored in encrypted server areas secured from external and internal unauthorized access. [g]
- Production databases reside on networks separated from servers hosting player interfaces. [h]
- Data is maintained regardless of server power state. [i]
- Data is stored to prevent loss when replacing parts/modules during normal maintenance. [j]

Typically owned by: operator governance, infrastructure, identity provider, wallet / ledger service.

#### GLI19-B.3.2 — Supervised, logged data alteration

*shall* · maps to `B.3.2 [a–f]`

Alteration of accounting, reporting, or significant-event data requires supervised access controls, and every change is logged with full before/after detail.

Testable criteria:
- Alteration of any accounting/reporting/significant-event data is not permitted without supervised access controls.
- Each change is documented/logged with: unique alteration ID [a]; data element altered [b]; value prior to alteration [c]; value after alteration [d]; time and date of alteration [e]; personnel who performed it (user identification) [f].

Typically owned by: platform backend, infrastructure, wallet / ledger service.

#### GLI19-B.3.3 — Daily backups

*shall* · maps to `B.3.3`

The backup scheme runs at least once every day (or as otherwise specified by the regulator); methods are reviewed case-by-case.

Testable criteria:
- Backup scheme implementation occurs at least once every day, or as otherwise specified by the regulator.

Typically owned by: infrastructure.

#### GLI19-B.3.4 — Redundant, geographically-separated backups

*shall* · maps to `B.3.4 [a–c]`

Audit logs, databases, and gaming data are stored with reasonable protection and redundant copies, on non-volatile media, transferred to a physically separate secure location, so no single failure loses or corrupts data.

Testable criteria:
- Redundant copies are kept with open support for backup/restoration so no single failure causes data loss/corruption; data integrity is protected in a failure.
- Backup is on non-volatile physical media (or equivalent architecture) so system and audit functions continue with no critical data loss if primary storage fails; disk-based backups assure integrity on disk failure. [a]
- On completion, backup media is immediately transferred to a physically separate location [b]; the storage location is secured against unauthorized access and permanent loss [b.i]; backup files/recovery components have at least the same security/access controls as the system [b.ii].
- Where the regulator allows cloud platforms, a backup stored in one cloud may have another copy in a different cloud platform or region. [c]

Typically owned by: infrastructure.

#### GLI19-B.3.5 — Redundancy and clean cross-component recovery

*shall* · maps to `B.3.5 [a–b]`

The system has sufficient redundancy/modularity that any single component failure loses no critical data, and linked components are tested pre-production to confirm restart/recovery does not lose/duplicate transactions and that components resynchronize on recovery.

Testable criteria:
- Sufficient redundancy and modularity so a single component (or part) failure lets system and audit functions continue with no critical data loss.
- For linked components, a pre-production test verifies: gaming operations between components are not adversely affected by restart/recovery (no lost or duplicated transactions) [a]; on restart/recovery, components immediately synchronize the status of all transactions, data, and configurations [b].

Typically owned by: infrastructure, platform backend.

#### GLI19-B.3.6 — Identify and handle master resets

*shall* · maps to `B.3.6`

The operator can identify and properly handle a master reset occurring on any component affecting gaming operations.

Testable criteria:
- The operator can identify and properly handle the situation where a master reset has occurred on any component affecting gaming operations.

Typically owned by: infrastructure, platform backend.

#### GLI19-B.3.7 — Full recovery from last backup

*shall* · maps to `B.3.7 [a–d]`

On catastrophic failure where the system cannot otherwise restart, it can be restored from the last backup point and fully recover, with the backup containing all critical information.

Testable criteria:
- Restoration from the last backup point achieves full recovery on catastrophic failure.
- The backup contains, at minimum: the "Information to be Maintained" (§2.8) [a]; site/venue info such as configuration and security accounts [b]; current system encryption keys [c]; any other system parameters, modifications, reconfiguration, additions/merges/deletions/adjustments/parameter changes [d].

Typically owned by: infrastructure, operator governance.

#### GLI19-B.3.9 — Business continuity and disaster recovery plan

*shall* · maps to `B.3.9 [a–e]`

A BC/DR plan can recover gaming operations if production is rendered inoperable, addressing data-loss minimization, invocation circumstances, a physically separate recovery site, technical recovery guides, and resumption of administrative operations.

Testable criteria:
- The plan considers disasters (weather, water/flood, fire, spills, malicious destruction, terrorism/ war, strikes, epidemics/pandemics, etc.) and:
- addresses storage of PII/sensitive/gaming data to minimize loss; if asynchronous replication is used, the recovery method or potential loss is documented [a];
- delineates the circumstances under which it is invoked [b];
- establishes a recovery site physically separated from production (cloud evaluated case-by-case) [c];
- contains recovery guides with technical steps to re-establish gaming functionality at the recovery site [d];
- addresses resuming administrative operations after activation for a range of scenarios [e].

Typically owned by: operator governance, infrastructure.

### B.4 Communications

#### GLI19-B.4.2 — Only enrolled, enabled critical components communicate

*shall* · maps to `B.4.2 [a–d]`

Only authorized devices establish communications between critical components, with methods to enroll/un-enroll and enable/disable components, and a default-deny (un-enrolled, disabled) state.

Testable criteria:
- A method exists to enroll and un-enroll critical components. [a]
- A method exists to enable and disable specific critical components. [b]
- Only enrolled and enabled critical components can participate in gaming operations. [c]
- The default condition for critical components is un-enrolled and disabled. [d]

Typically owned by: infrastructure.

#### GLI19-B.4.3 — Secure, authenticated, hardened communication protocol

*shall* · maps to `B.4.3 [a–e]`

Components communicate via a documented secure protocol with error detection/recovery, encryption and authentication for gaming/account-critical data, restriction to enrolled/authenticated components, hardening against malformed messages, and post-interruption resumption gating.

Testable criteria:
- Protocols use techniques with proper error detection/recovery designed to prevent intrusion, interference, eavesdropping, and tampering (alternatives reviewed case-by-case). [a]
- All data communications critical to gaming or player-account management employ encryption and authentication. [b]
- Secure-network communication is only possible between approved, enrolled, authenticated components; no unauthorized communications to components/access points. [c]
- Communications are hardened to be immune to malformed-message attacks. [d]
- After interruption/shutdown, communication with components is not established/authenticated until the resumption routine (incl. self-tests) completes successfully. [e]

Typically owned by: infrastructure, platform backend.

#### GLI19-B.4.4 — Encrypt sensitive data over public networks

*shall* · maps to `B.4.4`

Communications over internet/public networks (including Remote Player Devices) are secured by encryption or a secure protocol; PII, wagers, results, and financial/transaction data are always encrypted and protected against incomplete transmission, misrouting, modification, disclosure, duplication, or replay.

Testable criteria:
- Communications over internet/public networks encrypt data packets or use a secure protocol ensuring integrity and confidentiality.
- PII, sensitive info, wagers, results, financial info, and player transaction info are always encrypted over the public network and protected from incomplete transmissions, misrouting, unauthorized modification, disclosure, duplication, or replay.

Typically owned by: infrastructure, platform backend, player client.

#### GLI19-B.4.5 — WLAN adheres to wireless security standards

*shall* · maps to `B.4.5`

WLAN communications adhere to applicable jurisdictional wireless device/network-security requirements, or (absent jurisdictional standards) the GLI-26 wireless requirements.

Testable criteria:
- WLAN communications adhere to applicable jurisdictional wireless device and network-security requirements, or the GLI-26 "Wireless Device Requirements" and "Wireless Network Security Requirements" where no jurisdictional standards exist.
- Periodic WLAN integrity inspection/verification is performed (recommended).

Typically owned by: infrastructure.

#### GLI19-B.4.6 — Segmented, monitored, attack-resistant network

*shall* · maps to `B.4.6 [a–k]`

Networks are logically separated and hardened with authenticated/encrypted management, no single point of DoS, IDS/IPS, secured network equipment, 24/7 monitored entry/exit points, hypervisor separation for redundancy, stateful transport for sensitive data, change logging, anti-virus, and cyberattack monitoring.

Testable criteria:
- Networks are logically separated (no traffic on a link that cannot be serviced by its hosts).
- Network management authenticates all users and encrypts all management communications. [a]
- No single-item failure results in denial of service. [b]
- An IDS/IPS listens to internal and external comms and detects/prevents DDoS [c.i], shellcode [c.ii], ARP spoofing [c.iii], and other MITM indicators — severing comms immediately if detected [c.iv].
- On WLAN, the IDS/IPS scans for rogue APs/devices ≥quarterly [d.i], auto-disables rogue devices [d.ii], and keeps a 90-day wireless-access log reconcilable with other devices [d.iii].
- Network Communication Equipment is damage/corruption-resistant [e.i], physically [e.ii] and logically [e.iii] secured, and offloads/disables on full audit log [e.iv].
- All network entry/exit points are identified, managed, controlled, monitored 24/7 [f]; hubs/services/ ports secured [f.i]; unused services/ports blocked/disabled [f.ii].
- In cloud/virtualized environments, redundant instances do not run under the same hypervisor [g], each instance performs one function [g.i], with equivalent mechanisms considered as tech advances [g.ii].
- Stateless protocols (e.g. UDP) are not used for sensitive info without stateful transport (HTTP over TCP is acceptable). [h]
- All network-infrastructure changes are logged. [i]
- Virus scanners/detection programs are installed and updated regularly. [j]
- The operator monitors system/network to prevent, detect, mitigate, and respond to cyberattacks. [k]

Typically owned by: infrastructure.

#### GLI19-B.4.7 — Detect/respond to attacks and use threat intelligence

*shall* · maps to `B.4.7`

Appropriate measures detect, prevent, mitigate, and respond to common active and passive technical attacks, and a procedure gathers and acts on cyber threat intelligence.

Testable criteria:
- Measures are in place to detect, prevent, mitigate, and respond to common active and passive technical attacks.
- An established procedure gathers cyber threat intelligence and acts on it appropriately.

Typically owned by: infrastructure, operator governance.

### B.5 Third-Party Service Providers

#### GLI19-B.5.1 — Secure, segmented third-party communications

*shall* · maps to `B.5.1 [a–c]`

Third-party communications (loyalty, payment, location, security, cloud, live-game, identity verification) use encryption and strong authentication, log login events, and are segmented so they do not interfere with or route into the production environment.

Testable criteria:
- The system securely communicates with third-party providers using encryption and strong authentication. [a]
- All third-party login events are recorded to an audit file. [b]
- Third-party communication does not interfere with or degrade normal system functions: third-party data does not affect player communications [c.i]; providers are on a segmented network separate from player-connection segments [c.ii]; gaming is disabled on all network connections except production [c.iii]; the system does not route packets directly between third parties and production [c.iv]; the system does not act as an IP router between production and third parties [c.v].

Typically owned by: infrastructure, platform backend.

### B.6 Technical Controls

#### GLI19-B.6.1 — Secure DNS configuration

*shall* · maps to `B.6.1 [a–g]`

Public/external DNS uses secure, separated primary/secondary servers, restricted access, no arbitrary zone transfers, cache-poisoning prevention (DNSSEC), MFA, and registry lock.

Testable criteria:
- Secure primary and secondary DNS servers are logically and physically separate. [a]
- The primary DNS server is in a secure data center or an appropriately secured virtualized host. [b]
- Logical and physical DNS access is restricted to authorized personnel. [c]
- Zone transfers to arbitrary hosts are disallowed. [d]
- A cache-poisoning prevention method such as DNSSEC is used. [e]
- MFA is in place. [f]
- Registry lock is in place so DNS-change requests require manual verification. [g]

Typically owned by: infrastructure.

#### GLI19-B.6.2 — Cryptographic controls policy

*shall* · maps to `B.6.2 [a–g]`

A cryptographic-controls policy governs encryption of PII/sensitive data across lower-trust networks and portable media, message authentication, approved certificates, encryption grade, periodic algorithm review, key agility, and encrypted key storage.

Testable criteria:
- PII/sensitive info is encrypted when traversing a lower-trust network and when stored on portable systems (laptops, USB, etc.). [a]
- Data needing authentication but not concealment uses a message-authentication technique. [b]
- Authentication uses a security certificate from an approved organization (owner, issuer, valid dates, unique serial/identifier). [c]
- The encryption grade is appropriate to data sensitivity. [d]
- Encryption algorithms are periodically reviewed for continued security. [e]
- Different encryption keys are used so algorithms can be changed/replaced to correct weaknesses ASAP. [f]
- Encryption keys are stored on secure, redundant media after being themselves encrypted via a different method/key. [g]

Typically owned by: operator governance, infrastructure.

#### GLI19-B.6.3 — Defined encryption key-management lifecycle

*shall* · maps to `B.6.3 [a–e]`

Key management follows defined processes for generation/secure storage, expiry, revocation, keyset rotation, and recovery of data under revoked/expired keys for a defined period.

Testable criteria:
- Obtaining/generating keys and securely storing them with limited access. [a]
- Managing key expiry where applicable. [b]
- Revoking keys. [c]
- Securely changing the current encryption keyset. [d]
- Recovering data encrypted with a revoked/expired key for a defined period after it becomes invalid. [e]

Typically owned by: infrastructure, operator governance.

#### GLI19-B.6.4 — Harden critical components to best practice

*shall* · maps to `B.6.4 [a–e]`

Configuration procedures harden critical components against known vulnerabilities per industry best practice, regularly reassessed, removing defaults, one function per server, securing insecure services, configuring security parameters, and removing unnecessary functionality.

Testable criteria:
- Hardening procedures address all known security vulnerabilities per industry best practice and are regularly assessed and improved.
- Default/standard configuration parameters presenting a security risk are removed from all components. [a]
- Only one primary function per server (functions requiring different security levels do not co-exist). [b]
- Additional security features are implemented for required insecure services/protocols/daemons. [c]
- System security parameters are configured to prevent misuse. [d]
- All unnecessary functionality (scripts, drivers, features, subsystems, file systems, unnecessary web servers) is removed. [e]

Typically owned by: infrastructure.

#### GLI19-B.6.5 — Central, tamper-protected, reviewed security logs

*shall* · maps to `B.6.5 [a–d]`

Procedures centrally monitor/manage user activity, exceptions, and security events; logs are generated on each critical component, stored appropriately, protected against tampering, and periodically reviewed with records kept.

Testable criteria:
- Logs are generated on each critical component to monitor/rectify anomalies, flaws, and alerts. [a]
- Logs are stored for an appropriate period to assist future investigations and access-control monitoring. [b]
- Logs are protected against tampering and unauthorized access. [c]
- Logs are reviewed periodically using a documented process, with a record kept of each review. [d]

Typically owned by: infrastructure, platform backend.

### B.7 Remote Access and Firewalls

#### GLI19-B.7.1 — Secured, least-privilege remote access

*shall* · maps to `B.7.1 [a–d]`

Remote access (only if regulator-authorized) uses a secured method (MFA), can be disabled, accepts only firewall-permitted connections, and is limited to job-necessary functions with no unauthorized user administration or OS/database access.

Testable criteria:
- Performed via a secured method such as MFA. [a]
- Has the option to be disabled. [b]
- Accepts only remote connections permissible by the firewall application and system settings. [c]
- Limited to only the application functions necessary for job duties [d]: no unauthorized remote user administration (adding users, changing permissions) [d.i]; no unauthorized OS access or database access beyond information retrieval via existing functions [d.ii].

Typically owned by: infrastructure, operator governance.

#### GLI19-B.7.2 — Controlled supplier remote access

*shall* · maps to `B.7.2 [a–c]`

Supplier remote access for support/updates uses purpose-reserved accounts that are continuously monitored, disabled when not in use, and restricted to the necessary applications/ databases.

Testable criteria:
- Supplier remote-access accounts are continuously monitored by the operator. [a]
- They are disabled when not in use. [b]
- They are restricted via logical security controls to access only the applications/databases necessary for product/user support or updates/upgrades. [c]

Typically owned by: infrastructure, operator governance.

#### GLI19-B.7.3 — Automatic remote-access activity log

*shall* · maps to `B.7.3 [a–d]`

The remote-access application maintains an auto-updating activity log capturing user identity, network details, connection timing, and in-session activity, reviewed regularly.

Testable criteria:
- The log captures: user(s) who performed/authorized the remote access [a]; remote IP addresses, port numbers, protocols, and where possible MAC addresses [b]; connection time/date and duration [c]; activity while logged in, including areas accessed and changes made [d].
- The log is regularly reviewed as required.

Typically owned by: infrastructure.

#### GLI19-B.7.4 — Application-level firewall at security boundaries

*shall* · maps to `B.7.4 [a–h]`

All communications pass through at least one approved application-level firewall at security-domain boundaries, with no bypass paths, minimal firewall-resident applications/accounts, default-deny rules, anti-spoofing, and encrypted remote access.

Testable criteria:
- All communications (incl. remote access and connections to/from non-system hosts) pass through at least one approved application-level firewall.
- The firewall is located at the boundary of any two dissimilar security domains. [a]
- No device in the same broadcast domain as the host can establish an alternate path bypassing the firewall. [b]
- Any redundancy path also passes through at least one application-level firewall. [c]
- Only firewall-related applications reside on the firewall. [d]
- Only a limited number of user accounts exist on the firewall (e.g. admins only). [e]
- The firewall rejects all connections except specifically approved ones (default-deny). [f]
- The firewall rejects connections from destinations that cannot reside on the originating network (e.g. RFC1918 addresses on the public side). [g]
- Remote access uses encryption meeting current standards (ISO/IEC 19790, FIPS 140-2, or equivalent). [h]

Typically owned by: infrastructure.

#### GLI19-B.7.5 — Preserve firewall audit logs

*shall* · maps to `B.7.5 [a–c]`

Firewalls protecting production log audit information preserved against loss/alteration, capturing configuration changes, connection attempts, and network details; a threshold on failed attempts may deny further requests and notify the admin.

Testable criteria:
- The firewall logs, preserved and secured from loss/alteration: all firewall configuration changes [a]; all successful and unsuccessful connection attempts through the firewall [b]; source/destination IP addresses, port numbers, protocols, and where possible MAC addresses [c].
- A configurable 'unsuccessful connection attempts' threshold may deny further requests and notify the system administrator.

Typically owned by: infrastructure.
