# QA Technical Assessment — Local & International Fund Transfer

Senior QA Engineer submission for a retail-banking Quality Engineering role.
Five tasks covering test design, exploratory testing, API testing, automation
design, and quality strategy. Domain: money movement in a retail banking app,
currencies including SAR, daily limit 20,000, per-transaction limit 10,000.

---

## Time Spent

| Task | Description | Time |
|------|-------------|------|
| Task 1 | Test Design & Risk Coverage | 55 min |
| Task 2 | Exploratory Testing & Bug Reporting | 40 min |
| Task 3 | API Testing (Postman + written analysis) | 50 min |
| Task 4 | Automation Design (no code) | 45 min |
| Task 5 | Quality Strategy & Release Judgement (bonus) | 35 min |
| **Total** | | **~3 hours 45 min** |

These are honest estimates including writing and review. I spent extra time on
Task 3 Part B and Task 5 because they require sustained reasoning rather than
artifact production.

---

## Assumptions

Each assumption is stated so that a tester could design a test around it. If
any prove wrong, the affected test cases need revisiting.

1. **Currency scope.** The Local Fund Transfer feature operates in a single
   currency (the account's native currency). Cross-currency logic does not
   apply to this feature. If multi-currency accounts exist, AC1/AC4 need
   revisiting.

2. **Daily limit scope.** The 20,000 daily limit is per customer, aggregated
   across all source accounts. It is not per-account and not per-beneficiary.
   Reset occurs at midnight in the customer's local timezone (not UTC).

3. **Per-transaction limit scope.** The 10,000 limit applies to the transfer
   amount before fees. If fees are added on top, the total debit may exceed
   10,000 — that is acceptable unless the requirement says otherwise.

4. **Fee calculation.** The fee is a fixed amount or a percentage as defined
   in the bank's fee configuration. It is charged at the point of transfer and
   is not refundable on failure after the debit has occurred (unless the
   requirement specifies a reversal).

5. **OTP binding.** The OTP is bound to the specific transfer request — not
   just to the session. An OTP captured from one transfer attempt cannot be
   replayed on a different transfer.

6. **OTP expiry.** OTPs expire after a fixed window (assumed 5 minutes). After
   expiry the customer must request a new one. There is no unlimited retry.

7. **Idempotency.** The transfer initiation endpoint (or UI action) is
   idempotent for the same logical request within a time window. A double-tap
   on "Confirm" should not produce two debits.

8. **Atomicity.** The debit, fee deduction, beneficiary credit, reference
   generation, and SMS dispatch are treated as one atomic unit. If any piece
   fails after the debit, the system either rolls back or compensatory action
   is taken.

9. **Beneficiary status.** A beneficiary must be "active" or "verified" at the
   point of transfer. A beneficiary deleted or blocked after selection but
   before confirmation should cause a clean failure with no debit.

10. **SMS is a side-effect.** The SMS is sent after the transfer is committed.
    SMS gateway failure does not roll back the transfer. The transaction
    reference is the source of truth, not the SMS.

11. **Reference uniqueness.** Transaction reference numbers are unique across
    the bank's systems and include a date component and a sequence. Collisions
    are impossible under normal operation.

12. **Rounding.** Amounts are stored and calculated to two decimal places.
    Rounding follows standard half-up rules. Fee calculations may produce
    sub-cent values that are rounded at the point of application.

13. **Cut-off times.** There is no cut-off time for local transfers — they
    process immediately. (This assumption would be challenged if the bank
    operates a batch-processing window for local transfers.)

14. **Error taxonomy.** The system returns distinct, user-facing error messages
    for: invalid beneficiary, insufficient funds, limit exceeded, OTP expired,
    OTP invalid, session expired, system error. These are not generic "transfer
    failed" messages in production.

15. **Test environment parity.** The test/staging environment behaves the same
    as production for the purposes of this feature, including fee rules, limits,
    and OTP mechanics. Rate-limits and anti-fraud rules may differ.

---

## Questions I Would Ask the BA / PO

| # | Question | Why it matters | Assumption if unanswered |
|---|----------|----------------|--------------------------|
| 1 | Is the 20,000 daily limit per customer aggregated across all source accounts, or per account? | Aggregation scope determines how limit-exceeded tests are designed and which account(s) the limit service is queried against. A per-account limit would let a customer move 20,000 from each of three accounts — a very different risk profile. | Per-customer, aggregated across all source accounts. |
| 2 | Is the 10,000 per-transaction limit applied to the transfer amount alone, or to amount + fee? | If the fee is added on top and pushes the total debit above 10,000, a transfer of 9,990 with a 50 fee would breach the limit. The boundary test at 10,000 would be wrong. | Limit applies to the transfer amount before fee. |
| 3 | How is the transfer fee calculated — fixed amount, percentage of the transfer, tiered by amount band, or tiered by customer segment? | Fee logic is a high-risk area. A flat 5 SAR fee, a 1% percentage fee, and a tiered structure each need different test data and different boundary cases. The fee is also a source of rounding discrepancies. | Fee is a configurable value from the bank's fee configuration; tested as data-driven from a config table. |
| 4 | What is the OTP expiry window, and is it counted from generation or from first display to the customer? | The expiry window defines the boundary for expired-OTP tests. Counting from generation vs. display changes the effective window the customer sees. | 5 minutes from generation. |
| 5 | Is there an OTP retry limit before the transfer is locked, and if so what is it and what is the lockout duration? | Lockout behaviour is a security and usability concern. A customer locked out of transferring needs a documented recovery path. Without a known limit, the test approach is exploratory rather than targeted. | 5 attempts, 15-minute lockout. |
| 6 | Is the OTP bound to the specific transfer payload (amount, source account, beneficiary), or only to the user session? | If bound only to the session, a captured OTP could be replayed on a different transfer. Binding to the payload is the safer design and changes how we test OTP reuse. | OTP is bound to the specific transfer request. |
| 7 | How is transfer idempotency enforced — by a client-generated request ID, by deduplicating the full payload within a time window, or by locking the UI during OTP entry? | Idempotency design determines whether a double-tap on "Confirm" produces one debit or two. This is a top financial-risk area and the test approach depends entirely on the mechanism. | Idempotent by request identity within a time window; a duplicate request with the same identity returns the original result. |
| 8 | What is the atomicity guarantee if the SMS gateway fails after the debit is committed? Is there a compensation/reversal, or is the transfer left as committed with an incident logged? | This determines whether SMS failure is a rollback trigger or a non-rollback side-effect. The test for SMS failure then checks either for rollback or for the transfer being committed without the SMS. | SMS is a post-commit side-effect; failure does not roll back the transfer. |
| 9 | What happens to a transfer in progress if the beneficiary is deactivated or deleted by another channel during the OTP window? | This is a concurrency edge case. A debit with no valid destination is a financial and compliance risk. The behaviour could be a clean failure, a pending state that resolves later, or a silent success with no credit. | Beneficiary status is re-validated at confirmation time; if invalid, the transfer fails with no debit. |
| 10 | What currency does the Local Fund Transfer feature operate in, and is multi-currency supported at this stage? | Currency affects amount validation, fee calculation, and whether there are FX integration points. A single-currency feature is simpler; multi-currency introduces conversion and precision concerns. | Single native currency per account; cross-currency is out of scope for this feature. |
| 11 | What is the daily limit reset time — midnight in the customer's local timezone, midnight UTC, or a fixed hour? | Reset timing affects when a customer can transfer again and how limit tests are sequenced across midnight. A UTC reset would let a customer in a +6 timezone transfer earlier than expected. | Midnight in the customer's local timezone. |
| 12 | Are transaction reference numbers generated by the transfer service or by a shared sequencing service, and are they guaranteed globally unique? | Reference uniqueness is a compliance and reconciliation concern. A collision can cause audit failures and customer confusion. The test approach depends on whether uniqueness is within the transfer service or across the bank. | References are globally unique, date-prefixed with a sequence number. |
| 13 | What is the rounding rule for fee calculation when the result is not a whole cent (e.g., 1.5% of 1,234.56)? | Rounding differences can cause reconciliation failures between the app, the ledger, and the fee system. The test must assert against the correct rounded value, not the raw calculation. | Half-up rounding to two decimal places. |
| 14 | Is there a cut-off time after which local transfers are queued for next-day processing rather than executed immediately? | A cut-off introduces a pending state and changes the happy-path timeline. It also creates a boundary test at the cut-off hour. | No cut-off; transfers execute immediately. |
| 15 | What error messages does the customer see for each failure mode — specific (e.g., "Insufficient funds") or generic ("Transfer failed")? | Generic errors hide root causes and make customer support and bug triage harder. Specific, distinct messages are also a testable assertion: each failure mode should produce its own message. | Specific, distinct messages per failure mode. |

---

## What I Deliberately Left Out and Why

- **UI layout, typography, colour, and responsive behaviour.** Cosmetic and
  UX-polish issues do not create financial or regulatory risk. They are
  important in production but are not the focus of this assessment.
- **Performance and load testing.** The brief focuses on functional correctness.
  Load testing requires a separate environment, tooling, and agreement on
  thresholds.
- **Security and penetration testing.** Explicitly excluded by the assessment
  brief. I flag security-relevant risks in the risk assessment (OTP replay,
  idempotency) but do not perform active security testing.
- **Cross-currency and FX.** The Local Fund Transfer requirement describes a
  local transfer. FX is addressed separately in Task 3.
- **SMS gateway internal behaviour.** I test the integration point (was a
  request sent? did the transfer commit regardless?) but not the gateway's
  delivery mechanics. That is the provider's responsibility.
- **Accessibility (a11y).** Important in production but not in scope for this
  assessment and would require tooling I have not set up here.
- **Compatibility matrix.** Assumes the team has a defined supported-matrix
  document. I would test the critical path on the top 2–3 combinations if
  given a specific matrix.
- **Full beneficiary lifecycle.** I test that a blocked/deleted beneficiary
  blocks the transfer (TC-12). I do not test the full add/edit/verify/
  deactivate lifecycle of beneficiaries — that is a separate feature.

---

## How to Navigate This Repository

```
QA_Technical_Assessment/
├── .github/
│   └── workflows/
│       └── ci.yml           # GitHub Actions CI workflow pipeline
├── README.md                # This file
├── ai-usage.md              # AI tool disclosure
├── task-1/
│   └── test-design.md       # Task 1 — Test Design & Risk Coverage
├── task-2/
│   ├── exploratory-bugs.md  # Task 2 — Two functional bug reports
│   └── evidence/
│       └── README.md        # Evidence folder note
├── task-3/
│   ├── postman-collection.json        # Postman v2.1 collection
│   ├── environment.json               # Postman environment
│   ├── fx-rates-data.csv              # Base currency data file
│   ├── part-a-notes.md                # Part A — collection notes
│   └── part-b-payments-analysis.md    # Part B — payments endpoint analysis
├── task-4/
│   └── automation-design.md   # Task 4 — Automation design (no code)
└── task-5/
    └── quality-strategy.md    # Task 5 — Quality strategy (bonus)
```

---

## Continuous Integration (GitHub Actions)

The repository includes an automated CI workflow at [.github/workflows/ci.yml](file:///Users/t/Desktop/Task%20/QA_Technical_Assessment/.github/workflows/ci.yml). 

### Commands run in CI:
```bash
# 1. Execute Task 3 Postman API Collection via Newman with CSV data driver & environment
newman run task-3/postman-collection.json \
  -e task-3/environment.json \
  -d task-3/fx-rates-data.csv \
  -r cli,htmlextra \
  --reporter-htmlextra-export task-3/newman-report.html

# 2. Audit submission files & validate deliverables structure
test -f task-1/test-design.md
test -f task-2/exploratory-bugs.md
test -f task-4/automation-design.md
test -f task-5/quality-strategy.md
```

---

## AI Usage

See [ai-usage.md](./ai-usage.md) for a full disclosure of tools used, prompts
summarised, what was kept, what was rejected, and what I wrote or rewrote myself.
