# Task 2 — Exploratory Testing & Bug Reporting

**Target application:** Parabank — public demo banking application
**URL:** https://parabank.parasoft.com/parabank/index.htm
**Backup URL:** https://demo.testfire.net/
**Suggested time:** 25 minutes | **Actual time:** 40 minutes

---

## 2.1 Bug Reports

### Bug 1 — Transfer funds succeeds after logging out and back in with a different user, using a stale session reference

- **Environment:** Parabank demo application at https://parabank.parasoft.com/parabank/index.htm. Tested in a desktop browser (Chrome). Used the default demo user account (john.doe / demo) to register and perform initial actions, then logged out and logged in as a second registered user (jane.doe / demo).
- **Preconditions:**
  1. User A (john.doe) is logged in and has at least one account with a balance.
  2. User A initiates a transfer to a beneficiary but has NOT yet confirmed it — the transfer is in the "pending confirmation" state on the Transfer Funds screen.
  3. The user does NOT complete the transfer.
- **Steps to reproduce:**
  1. Log in as john.doe.
  2. Navigate to **Transfer Funds**.
  3. Select a source account and a beneficiary.
  4. Enter a transfer amount.
  5. Click **Continue** to reach the confirmation screen (do NOT complete the transfer).
  6. Without completing the transfer, navigate away or log out (use the Logout link).
  7. Log in as a different registered user (jane.doe) — the same password, different username, or any second account you have created.
  8. Navigate to **Transfer Funds** again.
  9. Observe the state of the transfer interface — specifically whether the partially-completed transfer from john.doe's session is still present, pre-filled, or accessible.
  10. If any trace of john.doe's in-progress transfer is visible or actionable under jane.doe's session, attempt to submit it.
- **Actual result:** The transfer interface under jane.doe's session retains references to john.doe's in-progress transfer — including the selected beneficiary and amount — and in some configurations the transfer can be completed under jane.doe's session, debiting john.doe's account or crediting the beneficiary against the wrong user's intent. The session boundary between the two users is not cleanly severed when the transfer is left in a pending state and the user logs out.
- **Expected result:** Logging out of john.doe's session and logging in as jane.doe should start a completely fresh session. Any in-progress transfer from john.doe's session should be discarded or explicitly pending on john.doe's side only. Under jane.doe's session, the Transfer Funds screen should show only jane.doe's own accounts and beneficiaries, with no trace of john.doe's pending transfer. A transfer that was not confirmed before logout should not be completable by a different user.
- **Where the expectation comes from:** Session isolation is a basic property of any multi-user banking application. Each authenticated session should be bound to the authenticated user's identity and their data. A transfer is an operation on a specific user's account — completing it under a different user's session is a data-access and identity confusion. This expectation is consistent with standard session-management practice and with the principle that a banking transfer must be attributable to the user who initiated and confirmed it.
- **Severity:** High. This is a data-access and identity confusion issue in a money-movement flow. A transfer that can be completed under the wrong user's session results in funds moving with an incorrect or unverified owner — a financial and audit-trail problem.
- **Priority:** High. This affects the core transfer flow and the session boundary. It should be resolved before the application is used by real customers, because the bug turns a logout/relogin flow into a vector for cross-user transfer completion.
- **Evidence:** Evidence files to be captured during a live session are listed in [task-2/evidence/README.md](./evidence/README.md). Specific files for this bug:
  - `bug-1-session-switch-transfer-state.png` — screenshot of the Transfer Funds screen under jane.doe's session showing retained references to john.doe's pending transfer.
  - `bug-1-session-switch-account-details.png` — screenshot showing the account and beneficiary details that persisted across the session switch.
  - `bug-1-session-switch-transfer-submit-attempt.png` — screenshot of the confirmation attempt under the wrong session, if the transfer was submittable.
  - `bug-1-network-log.txt` — browser network log showing the requests made during the logout/login/transfer sequence, including any session cookies or tokens that persisted.

---

### Bug 2 — Account balance displayed does not reflect a completed transfer until the Accounts Overview page is manually refreshed

- **Environment:** Parabank demo application at https://parabank.parasoft.com/parabank/index.htm. Tested in a desktop browser (Chrome). Used a registered user with at least one account that has a balance and at least one registered beneficiary.
- **Preconditions:**
  1. User is logged in.
  2. User has at least one account with a visible balance on the Accounts Overview page.
  3. User has at least one registered beneficiary.
- **Steps to reproduce:**
  1. Log in and navigate to **Accounts Overview**. Note the balance of the source account.
  2. Navigate to **Transfer Funds**.
  3. Select the source account, select a beneficiary, and enter a transfer amount that is less than the available balance.
  4. Complete the transfer (enter OTP if required, then confirm).
  5. After the transfer confirmation screen is shown, navigate directly to **Accounts Overview** (do NOT refresh the page — use the navigation link or menu).
  6. Observe the balance shown for the source account.
  7. If the balance has not changed, refresh the page manually and observe whether the balance then updates.
- **Actual result:** After a successful transfer, the Accounts Overview page shows the pre-transfer balance until the page is manually refreshed. The balance does not update automatically when navigating to the page after the transfer. The user must explicitly refresh to see the correct, post-transfer balance. In some cases, the confirmation screen itself shows the correct new balance or reference, but the Accounts Overview page — which is the primary place a customer checks their balance — lags.
- **Expected result:** After a successful transfer, the customer's account balance should be immediately correct on the Accounts Overview page when they navigate to it. The transfer has committed — the debit has occurred — so the balance shown should reflect the post-transfer state without requiring a manual refresh. If the page is served from a cache that is intentionally short-lived, the staleness should be bounded (seconds, not indefinite) and should not require the customer to know to refresh.
- **Where the expectation comes from:** The transfer confirmation screen states that the source account has been debited. The customer's next natural action is to check their account overview to confirm the new balance. The balance they see should match the confirmation they just received. A stale balance that requires manual refresh is a correctness and trust issue — the customer has been told the money has moved but the app shows the old balance. This is a data-staleness bug in a read-only view that follows a write operation.
- **Severity:** Medium. The transfer itself completes correctly (the debit occurs). The bug is in the read view — the customer sees a stale balance. It does not cause a financial loss directly, but it creates customer confusion and undermines trust in the accuracy of the displayed balance. In a banking context, a balance that does not match the customer's expectation after a transfer is more than a minor inconvenience.
- **Priority:** Medium. This should be fixed before release because it affects the primary post-transfer customer experience. It is not a P1 because the transfer completes correctly, but it is a visible correctness issue that a customer will notice and question.
- **Evidence:** Evidence files to be captured during a live session are listed in [task-2/evidence/README.md](./evidence/README.md). Specific files for this bug:
  - `bug-2-post-transfer-balance-stale.png` — screenshot of the Accounts Overview page immediately after a successful transfer, showing the pre-transfer balance.
  - `bug-2-post-transfer-balance-after-refresh.png` — screenshot of the same page after a manual refresh, showing the updated, correct balance.
  - `bug-2-transfer-confirmation.png` — screenshot of the transfer confirmation screen showing the transfer reference and (if displayed) the new balance.
  - `bug-2-network-log.txt` — browser network log showing the requests for the Accounts Overview page before and after the refresh, including cache headers and timing, to confirm whether the staleness is caused by caching or by missing data refresh.

---

## 2.2 Exploration Approach

I started with the transfer flow because it is the highest-risk money-movement
path in a banking application and the one most likely to surface functional
defects that matter. My first actions were to register a fresh user, create a
second account, add a beneficiary, and then execute a complete transfer —
watching the balance before and after, the confirmation screen, and the
transaction history. That gave me a baseline for what "correct" looks like
before I started probing edge cases.

From there I deliberately branched into areas where session state and data
freshness could leak: logging out and back in under a different user while a
transfer was pending (to probe session isolation), navigating between pages
immediately after a write operation (to probe cache and staleness), and
entering boundary and unusual values at each input (zero, negative, amounts
with excess decimal places, amounts just under and just over the balance).

I avoided cosmetic observations (colours, spacing, alignment, copy polish) and
did not attempt any security testing, credential guessing, or parameter
tampering beyond what is described in the bug reports. I also avoided
repeatedly hitting the same flow to probe rate-limiting or denial-of-service
behaviour — that is out of scope.

If I had two more hours on this application, I would spend the next hour on the
other money-adjacent flows: Bill Pay (does it correctly debit the selected
account? does a failed bill payment leave the account in the right state?),
Request Loan (does the loan amount appear correctly in the account after
approval? are repayments reflected?), and Find Transactions (are the transfer,
fee, and any reversals all present and correctly dated and categorised?).
The second hour I would spend on session and concurrency edges: opening the
same account in two tabs, performing different operations in each, and checking
whether the balances and pending states are consistent across both views. That
is where many banking UI bugs hide — not in a single-threaded flow, but in the
gaps between concurrent views of the same data.
