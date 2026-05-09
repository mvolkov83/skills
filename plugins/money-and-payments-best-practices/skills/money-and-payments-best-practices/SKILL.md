---
name: money-and-payments-best-practices
description: Engineering best practices for systems that handle money, payments, and ledgers — money representation (`Decimal` or integer minor units, never `float`; currency code as separate field; explicit rounding modes), the two-layer idempotency model (client-driven `idempotency_key` for request dedup AND chain-CAS for state transitions — they are distinct concerns and must never be mixed), double-entry ledger architecture (Accounts + Transfers as canonical entities, debit=credit invariant enforced at the database level, append-only immutable history, balance computed from entries — never stored), two-phase transfers (single-phase for funds inbound to user, two-phase via `HOLD` balance for funds outbound — `Reserve → Commit | Release` with mutual exclusion), atomic chains for composite operations (transfer-with-fee, multi-leg settlements as one DB transaction), transaction state machines with optimistic concurrency control via append-only state log + `UNIQUE(aggregate_id, parent_id)` chain CAS (no `SELECT FOR UPDATE`, no `SERIALIZABLE`, no `version int` column), external payment provider integration via stateless proxy/facade pattern with sync request + async webhook callback architecture (signature verification, three-layer dedup via webhook event ids, `QueryStatus` fallback for timeout reconciliation), reversibility via separate transaction with `parent_transaction_id` link and direction inversion, and DB-enforced invariants (UNIQUE constraints for both idempotency keys and chain CAS, partial UNIQUE for webhook event dedup, CHECK constraints for non-negative balances, foreign key integrity). Use this skill whenever the user is writing, reviewing, debugging, or designing money-handling code in a Python backend — including any work involving `Decimal` arithmetic on monetary amounts, currency conversion, idempotency key handling, payment intent / charge / refund / chargeback flows, ledger or journal table design, account balance computation, payment provider (PSP) integration with webhooks, transaction state machines with terminal states, refund / void / recall / chargeback patterns, optimistic concurrency control with append-only logs, parent-child transaction relationships, payment hash-chain audit trails, or compliance-grade financial audit. Trigger on imports of `decimal.Decimal`, money libraries (`py-money`, `dinero`, `stockholm`, `moneyed`), code defining `Transaction` / `Transfer` / `Ledger` / `Journal` / `Entry` / `Account` / `Balance` ORM models, payment-related proto / gRPC services, PSP webhook handlers, refund or chargeback handlers, or any database migration touching financial tables. Do NOT use for inventory management without payment flow, generic CRUD apps without ledger semantics, frontend payment form UI rendering, cryptocurrency-only systems with UTXO models (different patterns), tax computation engines (jurisdiction-specific encyclopedia), or KYC / onboarding flows (separate concern).
---

# Money & Payments Best Practices (Engineering, Python-First)

This skill captures production-tested patterns for systems that move money — payment processors, wallet/ledger services, billing engines, settlement systems, marketplaces with payouts. It assumes Python backends with PostgreSQL, async stack, and gRPC for internal RPC.

The patterns are anchored in two canonical public references and validated against production codebases:

- **Stripe's idempotency engineering** ([stripe.com/blog/idempotency](https://stripe.com/blog/idempotency)) — the de-facto standard for client-driven request deduplication in payment APIs.
- **TigerBeetle's debit/credit model** ([docs.tigerbeetle.com/concepts/debit-credit](https://docs.tigerbeetle.com/concepts/debit-credit/)) — the modern canonical reference for double-entry accounting at the database level.
- **HTTP Idempotency-Key header** (IETF in progress) — formalising the de-facto standard.
- **Modern Treasury — Enforcing Immutability in Your Double-Entry Ledger** — append-only ledger patterns.

User-emphasised depth: **money representation, idempotency (the two-layer model), double-entry, two-phase transfers, state machines, PSP integration, reversibility, DB-enforced invariants**. Reconciliation is out of scope for v1 (defer to project-specific tooling). FX / multi-currency conversion is out of scope (deserves its own skill if needed).

## Stack

- `Decimal` from `decimal` module OR integer minor units (Stripe pattern) — never `float`.
- A money library if monetary arithmetic is non-trivial: `py-money` (vimeo, enforces decimal places per currency), `dinero` (Decimal-backed, Decimal-style API), `stockholm` (modern, GraphQL/protobuf integrations), `moneyed`.
- PostgreSQL with `UNIQUE` and partial `UNIQUE` constraints — they are the database-level guards, not just performance hints.
- Async SQLAlchemy 2.0 — see the `sqlalchemy-best-practices` skill.
- A workflow engine (Temporal, Airflow, Celery) for long-running multi-step operations is helpful but optional.

## Money representation

The first decision sets the rest. Get it wrong here and every downstream calculation lies.

1. **Never `float` for money.** `0.1 + 0.2 == 0.30000000000000004`, not `0.3`. Two banks running this code arrive at different balances. The fix is to never enter floating-point territory in the first place.
   - **Why:** Python's `float` is IEEE 754 binary64 — most decimal fractions are not exactly representable. The error is small per operation, but compounds across millions of transactions and cents drift between systems that should reconcile to zero.

2. **Pick exactly one representation, project-wide:**

   **Option A — integer minor units** (Stripe's approach):
   ```python
   amount: int  # cents. $5.00 stored as 500.
   currency: str  # ISO 4217 code: "USD", "EUR", "JPY" (note: JPY has 0 decimal places).
   ```
   Pros: simple, fast, no precision concerns within `int` range; equality / ordering trivial.
   Cons: per-currency decimal place awareness needed at display boundaries; arithmetic on raw ints loses currency context (a typed wrapper class is recommended).

   **Option B — `Decimal` with currency**:
   ```python
   from decimal import Decimal
   amount: Decimal  # Decimal("5.00")
   currency: str
   ```
   Pros: matches accounting-style notation; arbitrary precision.
   Cons: must construct from strings (`Decimal("0.1")` not `Decimal(0.1)` — the latter inherits float error); needs explicit rounding.

   The choice is mostly aesthetic — both work. **What's not optional**: pick one and enforce it via a typed wrapper class throughout the codebase. Don't mix.

3. **Use a money library** unless the project is genuinely trivial. The libraries handle currency awareness, formatting, rounding modes, and arithmetic safety:

    | Library | Backing | Notable feature |
    |---|---|---|
    | `py-money` ([vimeo/py-money](https://github.com/vimeo/py-money)) | `Decimal` | Enforces correct decimal places per ISO 4217 currency |
    | `dinero` ([pypi](https://pypi.org/project/dinero/)) | `Decimal` | Modern Decimal-backed API |
    | `stockholm` ([kalaspuff/stockholm](https://github.com/kalaspuff/stockholm)) | `Decimal` | Built-in GraphQL + Protocol Buffers transports |
    | `moneyed` | `Decimal` | Mature, broad currency catalogue |

4. **Always pair `amount` with `currency`.** A bare `amount` field is undefined data. Schemas, API responses, log lines, span attributes all carry both.

5. **Currency conversion is an explicit operation with a rate snapshot.** Never auto-convert on read. The exchange rate at the time of conversion must be persisted alongside the resulting amount — converting back later at a different rate produces a different result, and audit trails must show the rate that was actually applied.
   - **Why:** FX rates fluctuate; an account in EUR shown as USD with a "current rate" displays a different value every refresh. The rate at `commit` time is the truth. Persist `rate_used` and `converted_at` with every conversion.

6. **Explicit rounding modes**, not implicit. `Decimal` supports `ROUND_HALF_UP`, `ROUND_HALF_EVEN` (banker's rounding), `ROUND_DOWN` etc. Banking domains typically use `ROUND_HALF_EVEN` (statistically unbiased, the default for financial reporting); receipt-style rounding to the customer's favour uses `ROUND_HALF_UP`. Document the choice; don't let it default silently.

7. **Single-currency invariant is a valid MVP scope.** If your system is single-currency in v1, **enforce that as an invariant** at the schema and validation layers — `transaction.currency == site.currency == account.currency`. Multi-currency / FX is a major scope explosion; deferring it deliberately is a sensible engineering decision.

## Idempotency — the two-layer model

The single most-misunderstood pattern in payment systems. The two layers below solve **different problems** and must never be conflated.

8. **Layer 1: Client-driven idempotency** — a `idempotency_key` (UUID v7) generated **client-side** on first attempt, sent verbatim on every retry. Server stores `(scope_id, idempotency_key) → original_response` in a dedicated table with a `UNIQUE` constraint.

   ```sql
   CREATE TABLE transaction_intents (
       transaction_id UUID PRIMARY KEY,           -- the resulting tx id
       scope_id UUID NOT NULL,                    -- per-tenant / per-user scope
       idempotency_key UUID NOT NULL,
       request_payload_hash BYTEA NOT NULL,       -- detect mismatches
       created_at TIMESTAMPTZ NOT NULL,
       UNIQUE (scope_id, idempotency_key)
   );
   ```

   The replay/collision dispatch:
    - **Same key, same payload hash** → return original `transaction_id` (replay; idempotent success).
    - **Same key, different payload hash** → return `422 Unprocessable Entity` with `idempotency_key_mismatch` reason (collision; client bug).
    - **New key** → create new transaction.

9. **Idempotency key is generated CLIENT-SIDE, never server-side.**
   - **Why:** If the server generates the key, a network timeout that drops the response means the client retries — but with no shared identifier, the server treats the retry as a brand-new request and creates a duplicate charge. Client-generated keys survive timeouts correctly. This is the canonical pattern (see [Stripe's idempotency blog](https://stripe.com/blog/idempotency) and the [HTTP Idempotency-Key RFC](https://httptoolkit.com/blog/idempotency-keys/)).

10. **Layer 2: Concurrency control via chain CAS** — solves the *different* problem of two concurrent state transitions on the same aggregate (operator cancel races with PSP webhook settle, scheduled expire fires alongside webhook arrival). This is **not** request dedup — it's serialisation of state transitions.

    Implemented via append-only log with `UNIQUE(aggregate_id, parent_id)`:

    ```sql
    CREATE TABLE transaction_state_log (
        id UUID PRIMARY KEY,                       -- UUIDv7
        aggregate_id UUID NOT NULL,                -- the transaction being mutated
        parent_id UUID NOT NULL,                   -- previous tip; self-id for bootstrap
        from_state VARCHAR(32),
        to_state VARCHAR(32) NOT NULL,
        payload JSONB NOT NULL,
        prev_hash BYTEA,
        hash BYTEA NOT NULL,                       -- hash-chain (audit)
        occurred_at TIMESTAMPTZ NOT NULL,
        UNIQUE (aggregate_id, parent_id)           -- the chain-CAS guard
    );
    ```

    Two concurrent writers reading the same tip both try to insert with the same `parent_id`. The `UNIQUE` constraint lets exactly one win. The other gets `IntegrityError`, which the application maps to a typed `StateChangedConcurrentlyError`.

11. **Idempotency vs chain CAS — different layers, different semantics:**

    | Aspect | Layer 1: Idempotency | Layer 2: Chain CAS |
    |---|---|---|
    | Protects against | Duplicate client requests (network retry) | Concurrent state transitions on one aggregate |
    | Conflict resolution | Silent return of original entity | Loud `StateChangedConcurrentlyError` (no retry) |
    | DB constraint | `UNIQUE(scope, idempotency_key)` on intent table | `UNIQUE(aggregate_id, parent_id)` on state log |
    | Example trigger | Same webhook delivered twice (PSP retry) | Operator cancel races with webhook settle |

    Don't try to solve both with one mechanism. Don't auto-retry on chain CAS conflict — the precondition that drove the original decision is now invalid.

12. **Chain CAS conflict is an assertion failure, NOT a retry trigger.**
    - **Why:** when the tip moves, the preconditions that justified the original decision (risk evaluation result, balance check, business rule outcome) may no longer hold. Re-reading the tip and inserting with a fresh `parent_id` silently corrupts the audit trail. Surface the conflict to the caller (HTTP `409 Conflict`, gRPC `ABORTED`); let them refresh state and decide again.

13. **Webhook deduplication is a third orthogonal layer.** External providers retry webhooks. Use a partial `UNIQUE` on the state log (or a separate inbox table):

    ```sql
    UNIQUE (inbox_source, inbox_event_id)
        WHERE inbox_event_id IS NOT NULL
    ```

    The PSP-supplied `inbox_event_id` (verbatim from the provider) deduplicates retries at the database level. Don't try to make webhook dedup share the chain-CAS mechanism — the unique key (`event_id`) is provider-supplied and orthogonal to the CAS chain.

## Double-entry ledger architecture

The canonical model, formalised by TigerBeetle: **two entities — Accounts and Transfers — and one invariant — every debit has an equal and opposite credit**. This minimal vocabulary expresses any exchange of value.

14. **Two entities, period.**
    - **Accounts** carry balances (sum of entries, computed not stored). Account types are *normal-side* qualified: ASSET-normal (player wallet, cash, receivables) or LIABILITY-normal (PSP counterparty, payable, deferred revenue). The "normal side" determines whether a debit increases or decreases the account.
    - **Transfers** (or "entries", "journal entries") are immutable records of value movement: `(from_account, to_account, amount, currency, timestamp)`. A transfer always touches exactly two accounts (the canonical double-entry model — multi-leg compositions are atomic chains of two-leg transfers, see #20).

15. **Append-only.** Transfers are NEVER updated, NEVER deleted. Corrections are *new compensating transfers*, not edits. This is the foundation of audit truthfulness.
    - **Why:** regulators and auditors require an unbroken evidence chain. A row updated last week looks identical in the database to a row that was always that way — the difference matters for compliance, dispute resolution, and forensic investigation. UPDATE on a ledger entry is a bug.

16. **Balance is computed from entries — never stored as the source of truth.** A `balance` column on the account is acceptable as a *denormalised cache* (updated atomically with the inserting transfer for fast reads), but the truth is `sum(credit_entries) - sum(debit_entries)` over the journal. Reconciliation routines must be able to recompute balance and verify the cache.

17. **DB-enforced invariants over application-enforced ones.** Where possible, encode the constraints in schema:

    ```sql
    CREATE TABLE account (
        id UUID PRIMARY KEY,
        account_type VARCHAR(32) NOT NULL,
        currency CHAR(3) NOT NULL,
        balance BIGINT NOT NULL,                    -- cached; minor units
        CHECK (account_type IN ('ASSET', 'LIABILITY', 'EQUITY', 'INCOME', 'EXPENSE')),
        CHECK (balance >= 0 OR account_type IN ('LIABILITY', 'EQUITY'))
    );

    CREATE TABLE transfer (
        id UUID PRIMARY KEY,
        from_account_id UUID NOT NULL REFERENCES account(id),
        to_account_id UUID NOT NULL REFERENCES account(id),
        amount BIGINT NOT NULL,
        currency CHAR(3) NOT NULL,
        occurred_at TIMESTAMPTZ NOT NULL,
        CHECK (amount > 0),
        CHECK (from_account_id <> to_account_id)
    );
    ```

    - **Why:** application-layer enforcement loses to race conditions, code paths bypassed by manual SQL, and the inevitable "we'll just patch this directly" production incident. The database is the only layer that all writers go through. TigerBeetle's tagline applies: *"enforce that accounts never go negative — at the database level, not in your application code."*

18. **Account-type taxonomy used in payment systems** (typical subset):

    | Type | Normal side | Examples |
    |---|---|---|
    | `REAL` (cashable) | ASSET | User's spendable balance |
    | `HOLD` (reserved / pending) | ASSET | Reserved funds awaiting settlement (see two-phase below) |
    | `LIABILITY` | LIABILITY | PSP counterparty account (we owe the PSP after a deposit settles) |
    | `BONUS` (restricted) | ASSET | Promotional credit with restrictions on use |
    | `FROZEN` (compliance) | ASSET | Funds frozen by compliance / risk hold |

    Specific systems extend or contract this; the principle is *typed accounts with explicit balance-flow semantics*, not a generic `Decimal` field.

## Two-phase transfers

Some money flows are atomic; others need two phases. The canonical examples are mirror images:

19. **Funds inbound to user — single-phase.** Deposit completion, refund credit, passive return. One atomic transfer:

    ```python
    # Single ApplyTransfer call
    Transfer(
        from_account=psp_counterparty,    # LIABILITY-normal: balance increases (we owe PSP for the deposit)
        to_account=user_real_balance,      # ASSET-normal: balance increases (user gains funds)
        amount=amount,
        currency=currency,
    )
    ```

    No reservation needed — the funds arrive from outside, there's no risk of the user spending them before settlement.

20. **Funds outbound from user — two-phase via `HOLD` balance.** Withdrawal, refund debit, chargeback compensating debit. Three-step protocol with mutual exclusion on the terminal step:

    | Step | Transfer | Triggered by |
    |---|---|---|
    | **Reserve** | `user_REAL → user_HOLD` | At intent commit (before any external call) |
    | **Commit** (terminal: success) | `user_HOLD → psp_counterparty` | PSP confirms success |
    | **Release** (terminal: failure) | `user_HOLD → user_REAL` | PSP declines / times out |

    - **Why two-phase:** between intent commit and PSP confirmation, the user might try to spend the same funds elsewhere (place a bet, initiate a second withdrawal). The HOLD makes the funds unavailable for new operations the moment the intent is committed — closing the fraud window without requiring distributed transactions across the wallet and PSP.

21. **Reserve happens at intent commit, NOT at PSP-call time.**
    - **Why:** if reserve waits until just before the PSP call, the user has the entire async risk-evaluation window to drain their balance into another flow (e.g., place a wager). HOLD active from the moment the withdrawal exists ensures the balance is unavailable throughout the lifecycle.

22. **Commit and Release are mutually exclusive — exactly one terminal transfer per HOLD.** The state machine guarantees this, not application code. Crash-recovery reads which step was already executed from the durable state log; on replay, the same `transaction_id` is used so the wallet de-duplicates idempotently.

23. **HOLD is a singleton balance per `(account, currency)`.** Don't create a new HOLD per transaction — that fragments accounting. Multiple in-flight outbound transactions share the same HOLD; the sum of in-flight reserves equals the HOLD balance. The wallet's idempotency on `transaction_id` distinguishes which transaction's HOLD-portion to release vs commit.

## Atomic chains for composite operations

24. **Composite operations (transfer-with-fee, multi-leg settlements, fund splits) are an atomic chain of two-leg transfers committed in one DB transaction.** TigerBeetle calls these "linked events"; in a relational DB they're multiple `INSERT` statements inside a single transaction with `BEGIN ... COMMIT`.

    ```python
    # Withdrawal with fee — three transfers, atomically:
    async with session.begin():
        # 1. Reserve from user
        await create_transfer(from=user_REAL, to=user_HOLD, amount=gross_amount)
        # 2. Capture the fee to operator revenue
        await create_transfer(from=user_HOLD, to=operator_revenue, amount=fee)
        # 3. Send the net to the PSP counterparty
        await create_transfer(from=user_HOLD, to=psp_counterparty, amount=net_amount)
    ```

    Either all three commit, or none do. The DB transaction is the unit of atomicity; no two-phase commit (2PC) across systems is needed.

25. **Why not 2PC across services?** Because 2PC is brittle (coordinator failure leaves participants in unresolved state) and not necessary if you keep ledger writes in one DB. Cross-service consistency goes via *sagas + idempotency*, not 2PC: each service's local DB transaction is atomic; cross-service compensation handles failures.

## Transaction state machines + chain-CAS OCC

The pattern formalised in #10. Worth its own section because it's the orchestrating mechanism for everything above.

26. **State as VARCHAR + Python `StrEnum`, not PG enum.** Avoid `CREATE TYPE state AS ENUM(...)` — renaming or removing values requires expensive migrations on multi-billion-row tables, and the lock-in is real.

    ```python
    import enum

    class TransactionState(enum.StrEnum):
        CREATED = "CREATED"
        VALIDATED = "VALIDATED"
        APPROVED = "APPROVED"
        REJECTED = "REJECTED"
        PROCESSING = "PROCESSING"
        AWAITING_CALLBACK = "AWAITING_CALLBACK"
        SETTLED = "SETTLED"        # terminal (success)
        FAILED = "FAILED"          # terminal (failure)
        CANCELLED = "CANCELLED"    # terminal (user/operator cancel)
        REVERSING = "REVERSING"    # transitioning to REVERSED
        REVERSED = "REVERSED"      # terminal (refunded/voided)
    ```

    Schema column: `current_state VARCHAR(32) NOT NULL`. App-layer enforces validity via the StrEnum.

27. **Explicit transition allowlist.** Don't let arbitrary `from_state → to_state` happen. A direction-aware allowlist (separate for inbound vs outbound flows where applicable) catches programming errors at the boundary.

28. **Atomic tip-and-CAS via anti-JOIN INSERT.** The "read tip → validate → INSERT with parent_id" sequence has a race window. Compress to a single SQL statement:

    ```sql
    INSERT INTO transaction_state_log (id, aggregate_id, parent_id, from_state, to_state, payload, hash)
    SELECT :new_id, :tx_id, tip.id, tip.to_state, :new_state, :payload, :hash
    FROM (
        SELECT id, to_state FROM transaction_state_log
        WHERE aggregate_id = :tx_id
        ORDER BY occurred_at DESC LIMIT 1
    ) AS tip
    WHERE NOT EXISTS (
        SELECT 1 FROM transaction_state_log child
        WHERE child.parent_id = tip.id
    );
    ```

    Zero rows inserted = conflict (without an `IntegrityError`); wrap in repository code that translates to `StateChangedConcurrentlyError`.

29. **READ COMMITTED, no SELECT FOR UPDATE, no SERIALIZABLE.** The chain-CAS pattern works at the lowest standard isolation level. SERIALIZABLE adds retry overhead under contention; pessimistic locks serialise access unnecessarily. Optimistic via UNIQUE is faster and clearer.

30. **Hash-chain over the state log for audit-grade evidence.** Each entry stores `prev_hash` (hash of the previous entry in the chain) and `hash` (hash of this entry's content + prev_hash). Tampering with any historical entry breaks the chain at that point, detectable by re-walking. This is *primary audit*; structured logs are defense-in-depth (see the `observability-best-practices` skill on log correlation).

## External payment provider (PSP) integration

31. **Stateless proxy / facade per PSP family.** One proxy service per PSP "flavor" (Stripe-style, M-Pesa-style), not per integration instance. The application service (payment system) holds credentials and config; the proxy holds *no state* — credentials flow per-call.

    ```
    PaymentService ─── ChargeRequest ───▶ Proxy ─── HTTP/SDK ───▶ PSP
                                          │
                                          ▼
                                       (translates PSP-specific quirks
                                        to canonical proto)
    ```

    The proxy:
    - Owns PSP-specific HTTP/SDK communication, signing, OAuth.
    - Normalises PSP webhook payloads to a canonical message before forwarding to the application service.
    - Has no DB, no per-customer state, no scheduled jobs.

    The application service:
    - Owns transaction state machine, idempotency, ledger.
    - Treats every PSP through the same `Charge / Payout / Refund / QueryStatus` API surface.

32. **Sync request + async webhook callback architecture.** The PSP returns "accepted, awaiting outcome" synchronously; the actual outcome arrives later via webhook. The application service must handle:
    - **Synchronous response**: outcome unknown — transition to `AWAITING_CALLBACK` state, store the PSP-side identifier (`psp_tx_id`), set a timeout.
    - **Webhook arrival**: signature verification, idempotency check (same `inbox_event_id` from PSP = no-op), state transition based on outcome.
    - **Timeout fallback**: if the webhook never arrives, call `QueryStatus(psp_tx_id)` to ask the PSP authoritatively. Reconciliation backstop catches anything the active flow missed.

33. **Webhook signature verification is non-negotiable.** Every webhook handler verifies the provider's HMAC signature (or equivalent — JWT, X.509) **before** any state mutation. Unverified webhooks are an attack vector for crediting attacker-controlled accounts.
   - **Why:** webhook URLs are sometimes leaked (in logs, in code, by URL probing). Without signature verification, anyone who knows the URL can post crafted payloads.

34. **Three-layer webhook deduplication:**
    1. Signature verification (rejects forgeries).
    2. `(inbox_source, inbox_event_id)` partial UNIQUE on the state log (rejects PSP retries).
    3. Application-level signal-handler idempotency (rejects in-process retries).

    All three together cost almost nothing at runtime and catch failures the others miss.

35. **`QueryStatus` as the authoritative timeout fallback.** When the webhook is late, don't guess — call the PSP. The PSP's view of "did this transaction succeed?" is the truth. The application reconciles its state against that answer.

## Reversibility — refunds, recalls, chargebacks

36. **A reversal is a separate transaction with `parent_transaction_id` linking to the original.** Don't mutate the original transaction's state to "refunded" — the original happened, the refund is a new economic event, and they have independent lifecycles.

    ```python
    class Transaction:
        id: UUID
        parent_transaction_id: UUID | None     # set on reversals
        direction: Direction                    # INBOUND or OUTBOUND
        # ... rest
    ```

37. **Direction inversion.** A reversal's `direction` is the *opposite* of its parent's:

    | Original direction | Reversal type | Reversal direction |
    |---|---|---|
    | INBOUND deposit | Refund | OUTBOUND |
    | OUTBOUND withdrawal | Recall (operator-initiated) | INBOUND |
    | OUTBOUND withdrawal | Passive return (PSP couldn't deliver) | INBOUND |
    | INBOUND deposit | Chargeback (lost dispute) | OUTBOUND |

    Money-flow direction (player credit vs player debit) follows from this — and so do the wallet primitives (single-phase vs two-phase from #19-23).

38. **Atomic chain CAS on the parent's state log when creating a reversal.** Two operators concurrently initiating a refund on the same transaction must result in *one* refund, not two. One DB transaction:
    - Inserts a chain-CAS entry on the parent's state log: `SETTLED → REVERSING` with `expected_from_state=SETTLED`.
    - Inserts the reversal transaction row with `parent_transaction_id`.

    Concurrent attempt → only one wins the chain CAS → other gets `409 Conflict`.

39. **Same backbone state machine, separate identity.** The reversal goes through `CREATED → PROCESSING → SETTLED` (its own lifecycle). The original transitions through `SETTLED → REVERSING → REVERSED` (the reversal's success drives the original's terminal-after-reversal state). Two coupled but distinct flows.

40. **Partial reversals are reversals with `amount < parent.amount`.** Multiple partial reversals can chain off the same parent; track cumulative-reversed via a derived view, never mutate the parent's `amount`.

## DB-enforced invariants — a checklist

The patterns above lean heavily on the database. Centralise the constraints:

41. **Idempotency uniqueness:** `UNIQUE(scope_id, idempotency_key)` on the intent table.
42. **Chain CAS uniqueness:** `UNIQUE(aggregate_id, parent_id)` on the state log.
43. **Webhook event uniqueness:** partial `UNIQUE(inbox_source, inbox_event_id) WHERE inbox_event_id IS NOT NULL` on the state log (or a dedicated inbox table).
44. **Account constraints:** `CHECK (balance >= 0 OR account_type IN ('LIABILITY', 'EQUITY'))` — application code that "would have caught it" in code review will not catch it under load.
45. **Transfer constraints:** `CHECK (amount > 0)`, `CHECK (from_account_id <> to_account_id)`, FK integrity to account rows.
46. **State VARCHAR length:** `VARCHAR(32)` is generous for state names, supports app-side `StrEnum`, avoids PG enum migration pain.
47. **UUIDv7 for time-ordered primary keys.** B-tree friendly (better cache locality than UUIDv4), millisecond-precise creation order, no PRNG dependency for replay safety. Use `uuid6.uuid7()` (real OS randomness) — never derive UUIDs deterministically inside workflow code.

## Things explicitly out of scope (with rationale)

- **Specific compliance regimes** (PCI DSS, SOC 2, GLI-19, PSD2, AML, KYC tiers) — encyclopedia-level, jurisdiction-dependent, regulatory, not engineering pattern.
- **PSP-specific quirks** (Stripe SCA, M-Pesa STK push polling, 3DS challenge flows) — defer to the PSP's documentation. The proxy/facade pattern (#31) isolates these from the application.
- **FX / multi-currency conversion** — significant scope. Deserves its own skill if needed.
- **Reconciliation** — daily match against external systems, mismatch taxonomy, evidence retention. Production patterns are typically project-specific (cadence, tooling, ticketing integration). Out of v1.
- **Tax computation** — jurisdiction-specific encyclopedia.
- **Cryptocurrency UTXO models** — different fundamental data model (no accounts, just spent/unspent outputs). The patterns here apply to account-based money systems.

## When applying these rules

- **Be opinionated about footguns** (#1 no floats, #9 client-generated idempotency keys, #11 don't conflate idempotency with chain CAS, #12 chain-CAS conflict is fail-loud not retry, #15 ledger entries are immutable, #17 DB-enforced invariants over app-enforced, #20 two-phase for outbound, #21 reserve at intent commit not PSP-call, #33 webhook signature verification, #38 atomic chain CAS on reversal creation) — these are the patterns where shortcuts cause silent data corruption, double-charges, lost funds, audit-trail tampering, or compliance failures. Not preferences.
- **Be flexible about taxonomy** (specific account types, state names, status enums, `transaction_type` values) — match the project's existing domain vocabulary. The *shapes* in this skill are the grammar; the *names* are the project's vocabulary.
- **Read the existing schema first.** If the project has an established ledger / journal / state-log table layout, conform to it rather than introducing this skill's column names. New services adopt; don't reinvent.
- **Cross-skill awareness:**
  - `sqlalchemy-best-practices` — ledger / journal table design, async session lifecycle, `expire_on_commit=False` for post-commit attribute access on transaction objects.
  - `grpc-python-best-practices` — sync RPC for application↔proxy, async webhook delivery via gRPC, the `<service>:<RpcError>` translation hierarchy for cross-service error handling.
  - `observability-best-practices` — log correlation via `trace_id`, business IDs as `<service>.<key>` span attributes for Tempo search across the lifecycle, hash-chain audit log as *primary* (not redundant with structured logs which are defense-in-depth).
  - `pytest-best-practices` — property-based testing with `hypothesis` for money invariants (`add(a,b) == add(b,a)`, `total = sum(parts)`, currency conversion round-trip identities), real-DB integration tests for ledger semantics (mocked SQLAlchemy hides invalid SQL).
- **The hardest part is restraint.** Money systems accumulate "small" features that turn into compliance liabilities (mid-flow currency changes, stored card details, balance corrections without audit). The patterns above lean on the database and the schema deliberately — not because schemas are fashionable, but because every other layer eventually has a bypass path. If a money rule isn't in the schema or in an immutable log, it's not actually a rule.
