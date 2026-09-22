# Evidence Folder

This folder is populated when the tester runs a live exploratory session
against the target application. It is not pre-populated because the evidence is
tied to a specific session — screenshots, network logs, and console output need
to be captured at the time of the finding to be reproducible and credible.

When live testing is performed, the following files would be captured and named
to match the bug report references in `../exploratory-bugs.md`:

**For Bug 1 — Session switch / stale transfer state:**
- `bug-1-session-switch-transfer-state.png` — screenshot of the Transfer Funds
  screen under the second user's session showing retained references to the
  first user's pending transfer.
- `bug-1-session-switch-account-details.png` — screenshot showing the account
  and beneficiary details that persisted across the session switch.
- `bug-1-session-switch-transfer-submit-attempt.png` — screenshot of the
  confirmation attempt under the wrong session, if the transfer was submittable.
- `bug-1-network-log.txt` — browser network log (or exported HAR subset) showing
  the requests made during the logout / login / transfer sequence, including any
  session cookies or tokens that persisted across the switch.

**For Bug 2 — Post-transfer balance staleness:**
- `bug-2-post-transfer-balance-stale.png` — screenshot of the Accounts Overview
  page immediately after a successful transfer, showing the pre-transfer balance.
- `bug-2-post-transfer-balance-after-refresh.png` — screenshot of the same page
  after a manual refresh, showing the updated, correct balance.
- `bug-2-transfer-confirmation.png` — screenshot of the transfer confirmation
  screen showing the transfer reference and (if displayed) the new balance.
- `bug-2-network-log.txt` — browser network log showing the requests for the
  Accounts Overview page before and after the refresh, including cache headers
  and timing, to confirm whether the staleness is caused by caching or by a
  missing data refresh after the write.

If a screenshot is not sufficient (e.g., the bug is in a network response or
console output), a text or JSON capture is used instead. Evidence files are
saved to this folder with the naming convention above so that a reader of the
bug report can find the exact supporting artifact without guessing.
