# Task 3 — API Testing

**Suggested time:** 30 minutes | **Actual time:** 50 minutes
**Provider used:** open.er-api.com (https://open.er-api.com/v6/latest/USD)

---

## Part A — Collection Notes

### Which provider was used and why

I used open.er-api.com because it was reachable from my environment on the day
of testing, requires no API key, and returns a clean and predictable JSON shape
with a `result` field that distinguishes success from failure. The response
includes `time_last_update_unix` and `time_last_update_human` which are useful
for freshness checks. The currency set is broad enough to cover the currencies
in the data file (USD, EUR, GBP, SAR, AED) without special handling.

The backup provider (api.frankfurter.app) returns a different shape — it uses
`base` instead of `base_code`, nests rates differently, and does not include a
top-level `result` field. Because the two providers differ in shape, error
format, and currency set, I designed the collection against the provider I
actually used. The assertions in the collection reflect open.er-api.com's
behaviour. If the backup were used, the schema assertions would need to change
to match frankfurter's response shape.

### Assertion table

| Assertion group | What it verifies | Why chosen |
|-----------------|------------------|------------|
| Status code | Response is 200 for valid requests | A non-200 response from a rates service is a hard failure — the app cannot proceed with a rate lookup. Asserting 200 upfront fails fast. |
| Response time | Response time < 2000ms | A rates service that is slow becomes a bottleneck for the transfer flow. 2 seconds is a conservative threshold for a customer-facing lookup; a production app would tune this against its own SLA. |
| Top-level schema | Response has `result`, `base_code`, and `rates` | Validates that the provider returned the shape we depend on. If any of these fields is missing, the app's parsing logic will fail. |
| Result value | `result` == "success" | open.er-api.com uses `result` to signal success or failure. A `result` of "fail" means the lookup did not succeed even though status is 200 — this is the key negative-path signal for this provider. |
| Base code echo | `base_code` matches the requested `baseCurrency` | Confirms the response is for the currency we asked for, not a default or a misrouted request. A mismatch here would be a subtle but serious bug. |
| Target currency presence and sanity | EUR, GBP, JPY, CAD, AUD, CHF, CNY, INR, SAR, AED are present and > 0 | These are the currencies the app is likely to need. A missing currency or a zero/negative rate would break a conversion. The assertion is broad enough to catch a partial outage. |
| Plausibility of captured EUR rate | Captured EUR rate is between 0.1 and 10.0 | A sanity bound that catches wildly implausible values (e.g., a service returning 0 or 1,000,000). Not a precise check — just a guard against obvious corruption. |
| Cross-currency reconciliation | EUR-base response's USD rate is within 0.5% of 1 / captured USD-to-EUR | Proves the captured value is used meaningfully. We do not just store `rates.EUR` — we use it to independently verify a later response. The 0.5% tolerance accounts for the fact that rates can move between the two calls. |
| Negative case — actual behaviour | For "INVALID", the API returns 200 with `result` = "fail" and an `error` field | We assert on what the provider actually does, not on what we wish it did. This is documented so the assertion is honest about the provider's behaviour. |

### Why I captured rates.EUR

I captured `rates.EUR` because it is the most likely cross-currency anchor in a
SAR-based banking app. The app's primary customer currency is SAR, and the most
common secondary currency for international transfers is likely EUR or USD. By
capturing EUR from a USD-base call, I create a value that can be used in a
subsequent request to verify consistency — in this collection, that verification
is the EUR-base cross-check (request 2), where `rates.USD` from the EUR-base
response should be approximately `1 / usdToEur`.

This is not an arbitrary capture. The point of capturing a response value and
using it in a later request is to demonstrate that the collection is doing
something useful with state, not just storing it. The cross-check proves the
captured value is correct enough to be relied on, and it simulates a pattern
that would be useful in a real collection: capture a rate, then use it to
validate a related rate from a different base, or to seed a conversion
calculation in a later step.

### Negative-case finding and banking-app judgement

When I requested `GET /v6/latest/INVALID`, open.er-api.com returned HTTP 200
with a JSON body where `result` = "fail" and an `error` field described the
problem (unsupported currency code). The API did not return a 4xx status code.

**Is this acceptable for a banking app?** Not ideally. For a banking
application that consumes this API, a 200-with-error is a riskier contract than
a 400-class response, for two reasons:

1. **Client error handling.** A client that checks only the HTTP status code
   will treat the failed lookup as a success and may proceed with no rate, a
   stale rate, or a default rate. In a money-movement context, that could mean
   an international transfer proceeds without a valid FX rate — which is exactly
   the kind of thing that causes financial discrepancies. A 400-class response
   forces the client to handle the failure explicitly.

2. **Operational visibility.** A 200-with-error is harder to monitor and alert
   on than a 4xx. In a banking app, you want to know when a rates service is
   returning failures, and a 4xx series makes that visible at the HTTP layer
   without parsing every body.

That said, the behaviour is not unsafe if the client is written correctly — it
must check `result` regardless of status code. The provider's choice is a
contract design decision, and the collection asserts on the actual behaviour
honestly rather than pretending the API returns a 4xx. If I were designing the
client integration, I would add a defence-in-depth check: status 200 AND
`result` == "success", and treat anything else as a rate-failure that blocks
the transfer.

---

## Part B — Payments Endpoint Analysis

**Endpoint:** `POST /api/v1/payments`
**Assessed as:** Senior QA — written analysis, no code.

This is a money-movement endpoint. The testing approach must treat it as a
financial transaction, not a CRUD operation. The highest-risk areas are the
ones where money can move incorrectly, twice, to the wrong place, or without a
proper audit trail.

### 1. Contract and schema validation

Before testing any money movement, verify that the endpoint enforces its own
contract. The body must contain `sourceAccountId`, `beneficiaryId`, and an
`amount` object with `value` and `currency`. `fxQuoteId` is optional but
context-dependent (see section 4). `purposeCode` and `note` are strings.

What to validate:

- Required fields are enforced. A request missing `sourceAccountId`,
  `beneficiaryId`, or `amount` returns 400 with a clear field-level error, not
  a generic "bad request".
- `amount.value` is a number, not a string, and `amount.currency` is a valid
  three-letter code. A string value or a non-numeric value is rejected at the
  boundary, not silently coerced.
- `amount.currency` matches the expected currency for the source account if the
  system enforces single-currency accounts. If cross-currency is supported,
  the currency must be one the bank deals in.
- `purposeCode`, if required by regulation (e.g., Saudi Arabia's SAMA purpose
  codes for international transfers), is validated against an allowed set, not
  accepted as free text.
- Extra or unknown fields are either rejected or ignored consistently — the API
  should not silently accept a field that has no defined meaning, because that
  hides client bugs.

The contract is the first line of defence. A request that fails contract
validation should never reach the money-movement logic.

### 2. Idempotency (highest priority)

The `Idempotency-Key` header is the primary guard against duplicate money
movement. This is the single most important header on this endpoint. Testing
it is not optional and not a nice-to-have — it is a core financial-safety
check.

What to test:

- **Same key, same body, first request succeeds.** The first request with a
  given idempotency key and a valid body returns 201 with a new payment. The
  payment is created, the money moves, and a reference is generated.

- **Same key, same body, second request.** The second request with the same key
  and the same body returns the same result as the first — ideally 201 with the
  same `paymentId`, `reference`, and status. The money must not move a second
  time. The response should make it clear that this is the original result, not
  a new payment. The cheapest correct behaviour is to return the original
  response unchanged. A 409 or a 200 with a "already processed" indicator is
  also acceptable if the money did not move again — but 201 with a new paymentId
  is wrong.

- **Same key, different body.** This is the subtle case. If the client sends the
  same idempotency key but with a different body (different amount, different
  beneficiary, different source account), what should happen? The safest design
  is to reject the second request — because the key no longer identifies the
  same operation. Returning the original result would be wrong (the client asked
  for a different transfer), and processing it as a new transfer would defeat the
  purpose of the key (the key is now associated with two different operations).
  The expected behaviour is a 409 Conflict or a 422 with a clear "idempotency key
  reused with different body" message. Test that this actually happens — a common
  bug is that the server ignores the body mismatch and returns the original result,
  which silently suppresses the client's intended change.

- **TTL and expiry.** Idempotency keys should expire after a reasonable window
  (minutes to hours, not days). After expiry, the same key should be treatable as
  a new request — otherwise a client that retries a key after a long pause gets an
  unexpected conflict. Test that a key reused after the TTL window is accepted as a
  fresh request.

- **Retry after network timeout.** The client sends a request, the server processes
  it and creates the payment, but the response is lost (network timeout, connection
  drop). The client retries with the same key. The second request must return the
  original result — the payment must not be created again. This is the canonical use
  case for idempotency and the one that matters most in production.

- **Retry after 503.** The client sends a request, the server returns 503 Service
  Unavailable before processing. The client retries. If the server never processed
  the first request, the retry should succeed normally. If the server started
  processing but failed mid-way, the retry should be safe — either the partial
  work is rolled back and the retry creates a new payment, or the server detects the
  in-flight key and returns a safe response. Test both paths. The 503 case is
  especially important because a 503 does not tell the client whether the operation
  happened or not — that is exactly why idempotency matters.

- **Key uniqueness and collision.** The client must generate sufficiently unique
  keys (e.g., UUIDs). Test that two different clients (or two different transfers
  from the same client) do not accidentally share a key and cause one to block the
  other. A coarse test: send two concurrent requests with different bodies and
  different keys; both should succeed independently.

### 3. Authorization and tenancy

Money movement must be authorized at multiple levels. The endpoint sees an
`Authorization` header and an `X-Channel` header — both are signals that the
caller's identity and context matter.

What to test:

- **Source account ownership.** The authenticated caller must own or have
  authority over the `sourceAccountId`. A caller who references a source account
  they do not own must receive 403 (or 404 to avoid leaking account existence,
  depending on the bank's information-leakage policy). Test with a valid token but
  a source account belonging to a different customer.

- **Beneficiary ownership and status.** The `beneficiaryId` must belong to the
  same customer (or be a valid external beneficiary the customer is authorised to
  send to). A beneficiary that belongs to a different customer, or that has been
  deactivated, blocked, or frozen, must cause a clean rejection. Test with a
  beneficiary that exists but is not owned by the source account's customer.

- **Beneficiary–source account relationship.** If the bank requires beneficiaries
  to be pre-registered before they can receive a transfer, test that an unregistered
  beneficiary is rejected. If the system supports ad-hoc external beneficiaries,
  test that the required validation (KYC, block-list check, sanctions screen) is
  applied.

- **X-Channel context.** The `X-Channel` header likely identifies the channel
  (mobile app, web, branch terminal, partner integration). Test that the channel is
  validated or at least logged, and that a request with a missing or unexpected
  channel is handled consistently. If certain channels have different limits,
  fees, or approval rules, test that the channel is correctly applied to the
  transfer — a mobile-app request should not inherit branch-terminal limits, and
  vice versa.

- **Token scope and expiry.** The Authorization token must be valid and not
  expired. Test with an expired token (401), an invalid token (401), and a token
  that is valid but lacks the scope to create payments (403). These are basic
  authz tests, but on a money endpoint they are non-negotiable.

### 4. Amount and FX validation

This is where the money movement is defined. The `amount` object carries `value`
and `currency`, and the optional `fxQuoteId` ties the transfer to an FX rate.

What to test:

- **Zero and negative amounts.** `amount.value` of 0 should be rejected (no money
  movement is not a transfer). Negative values should be rejected outright. A
  negative transfer is not a valid operation — it would be a credit, not a debit,
  and the endpoint is for outgoing transfers.

- **Decimal precision.** Money values must be validated for precision. If the bank
  operates in a currency with two decimal places, a value with more than two
  decimals should be rejected or rounded explicitly and documented. A value like
  1500.755 should not be accepted silently and stored as 1500.76 without the client
  knowing — that is a rounding discrepancy that causes reconciliation failures. Test
  the boundary: exactly two decimals is fine, three decimals is rejected or
  explicitly rounded.

- **Currency mismatch.** If the source account is a SAR account and the amount is
  specified in EUR, the system needs an FX conversion. If `fxQuoteId` is provided,
  it must match the amount's currency and be a valid, unexpired quote. If
  `fxQuoteId` is missing on a cross-currency transfer, the request must be rejected
  — you cannot move money across currencies without a rate. Test both: a cross-currency
  transfer with a valid `fxQuoteId` (should proceed or enter a PENDING state awaiting
  confirmation), and one without `fxQuoteId` (should be rejected).

- **FX quote binding and expiry.** If `fxQuoteId` is provided, the transfer must use
  that specific quote — not a fresh rate from the rates service. Test that the quote
  is validated (exists, belongs to this customer, is for the correct currency pair,
  has not expired). A quote that has expired should cause a clear rejection, not a
  silent fallback to a current rate (which would change the amount the customer
  agreed to).

- **Amount vs. available funds (pre-check).** The amount must be checked against
  available balance, including any holds, pending debits, or reserved amounts. A
  transfer of 1500.75 against an account with exactly 1500.75 available and no
  buffer should be rejected if the bank requires a buffer. Test the boundary: amount
  equal to available balance, amount one unit above available balance.

- **Rounding in the amount object.** If the FX conversion produces a rounded amount
  (e.g., the quoted rate converts 1500 SAR to 399.999 EUR, rounded to 400.00 EUR),
  the amount the customer sees and the amount actually debited must be consistent.
  Test that the rounded amount is what is debited and what is shown, and that the
  rounding rule is applied consistently.

### 5. Limits and funds

The endpoint must enforce limits server-side, not trust the client to send an
amount within limits. The brief mentions a daily transfer limit and a
per-transaction limit; these must be enforced at the server.

What to test:

- **Per-transaction limit.** A transfer above the per-transaction limit (10,000 in
  the local-transfer context, or the equivalent for this endpoint) must be rejected
  with a limit-exceeded response. Test at the boundary: exactly at the limit (should
  succeed), one unit above (should fail). If fees apply on top, test whether the
  limit is applied to the amount alone or to amount + fee (see Task 1 assumptions).

- **Daily cumulative limit.** The endpoint must check the customer's cumulative
  transfers today against the daily limit. Test reaching the limit exactly (should
  succeed), then exceeding it (should fail). Test that the limit is aggregated
  correctly across multiple transfers, multiple source accounts if applicable, and
  across channels if the limit is channel-agnostic.

- **Concurrency race on limits.** Two transfers from the same customer, sent nearly
  simultaneously, each see available limit and both succeed, collectively exceeding
  the daily limit. This is the concurrent limit-bypass case. Test with two requests
  in quick succession and verify that the second does not bypass the limit. The
  correct behaviour is that the limit check and the debit are atomic, or that the
  second request sees the updated balance after the first commits.

- **Insufficient funds.** A transfer for more than the available balance must be
  rejected. Test the boundary and the atomicity: if the debit and the limit check
  are not atomic, a transfer could succeed against insufficient funds. This is a
  money-loss risk and must be tested explicitly.

- **Limits and fees combined.** If the transfer has a fee, the total debit (amount +
  fee) must be checked against both the per-transaction limit and the available
  balance. A transfer of 9,990 with a 50 fee totals 10,050, which may exceed a
  10,000 limit. Test the combined check.

### 6. State machine and error taxonomy

A payment is not a single operation — it moves through states. The response
returns `status: PENDING | COMPLETED`, and the endpoint must handle the full
state lifecycle, not just the creation.

What to test:

- **PENDING → COMPLETED.** A payment created in PENDING state must transition to
  COMPLETED when the money movement is confirmed. Test that the transition happens,
  that the status is updated, and that the transition is recorded in the audit log.

- **PENDING → FAILED.** If the payment cannot be completed (insufficient funds
  discovered after creation, FX quote expired, beneficiary blocked), the PENDING
  payment must transition to FAILED with a clear reason. Test each failure path from
  PENDING, not just from the initial request.

- **REVERSED / cancelled.** If a PENDING payment is cancelled before completion, the
  system must reverse any provisional holds and return the funds. Test cancellation
  from PENDING and verify the balance is restored.

- **Stable error codes.** The listed error responses (400, 402, 409, 422, 503) must
  be used consistently and for the right reasons. 400 for contract/validation
  failures, 402 for insufficient funds, 409 for conflicts (idempotency key reuse with
  different body, concurrent modification), 422 for limit exceeded or business-rule
  violations, 503 for service unavailability. Test that each code is returned in the
  right situation and that the same situation does not sometimes return a different
  code.

- **No PII in errors.** Error responses must not leak sensitive data — full account
  numbers, beneficiary details beyond what is necessary, internal identifiers that
  reveal system structure. Test that error bodies contain only what a client needs to
  handle the error, not internal details.

- **Idempotent state transitions.** Once a payment is COMPLETED, further attempts to
  create or modify it must be handled safely — either rejected with a conflict or
  returned as the existing state. The payment must not be debited again.

### 7. Observability and reconciliation

For a money endpoint, the API response is not the source of truth — the audit trail
is. Testing must verify that the right things are recorded, not just that the right
response is returned.

What to test:

- **Unique reference.** The `reference` returned in the 201 response must be unique
  and traceable. Test that two consecutive payments get different references, and that
  the reference format is consistent and includes enough information to identify the
  payment in downstream systems (date, sequence, customer identifier, or equivalent).

- **Audit log entry.** Every payment creation, state transition, and failure must be
  recorded in an audit log with timestamp, actor (customer or system), source account,
  beneficiary, amount, currency, fee, FX quote used, and the event type. Test that the
  audit log entry matches the API response and the actual money movement.

- **Reconciliation alignment.** The amount debited from the source account must match
  the amount in the payment record, the amount in the audit log, and the amount the
  customer sees. A discrepancy between any of these is a reconciliation bug. Test that
  these three figures agree for a successful transfer, including the fee.

- **Payment lookup.** The `paymentId` returned in the response must be usable to look
  up the payment later — its status, amount, and state transitions must be queryable.
  Test that the lookup returns the correct current state, not a stale one.

- **Idempotency record.** The idempotency key must be recorded alongside the payment
  so that the system can return the original result on retry. Test that the key is
  stored and that a retry with the same key returns the original payment.

### What I would NOT do here

- **Assert on a hard-coded expected status without probing the actual contract.**
  I would not write a test that assumes a given situation returns 400 without first
  verifying that the endpoint actually returns 400 for that situation. Error codes
  can be wrong, and asserting on an assumption without checking is how you miss a
  mismatch between the documented contract and the implemented one.

- **Perform load or stress testing on this endpoint as part of functional API
  testing.** Load testing requires a different environment, tooling, and stakeholder
  agreement on thresholds. Functional API testing verifies that the endpoint behaves
  correctly under normal and edge-case conditions; it does not verify that it survives
  1,000 requests per second. Those are separate activities.

- **Perform security testing beyond functional authorization checks.** Verifying that
  an unauthorised caller cannot move money is part of functional API testing. Actively
  probing for injection, token forgery, or endpoint enumeration is security testing and
  is out of scope for this analysis. I would flag any obvious authz gap I encounter,
  but I would not go looking for it.

- **Test the FX rates provider as part of the payments endpoint testing.** The FX
  provider is a separate service with its own contract (tested in Part A). For the
  payments endpoint, I would test that the endpoint binds to a quote correctly and
  rejects an invalid or expired quote — I would not retest the rates provider's
  behaviour here.

- **Assume the happy path is sufficient.** A money endpoint's happy path is necessary
  but not sufficient. The highest-risk cases are the ones where the happy path is
  violated — wrong key, wrong body, wrong currency, wrong time, wrong caller. Those
  are the cases I would prioritise.
