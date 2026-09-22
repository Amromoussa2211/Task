# AI Usage Disclosure

## Tools Used

- **Solar Pro 4 (Upstage)** via Hermes Agent — used as a thinking and drafting
  partner across all five tasks. I used it to pressure-test my reasoning, to
  check that I had not missed an obvious risk, and to help structure responses
  under time pressure.
- **Terminal tools** — used to create the directory structure, verify Postman
  JSON validity, and read the assessment PDF.

No external web searches or API calls were made for this submission beyond what
the assessment itself required (the demo banking app for Task 2 and the FX rates
API for Task 3).

---

## Prompts Used (summarised)

- "Here is a banking local fund transfer requirement with six acceptance
  criteria. List the 12–15 sharpest questions I should ask the BA before
  testing, focusing on financial risk, idempotency, limits, and OTP mechanics.
  Quality over quantity." — used to pressure-test Task 1 question list after I
  had drafted my own.

- "I have 12 test cases for this feature. Read them and tell me if any two
  target the same root cause, if I have missed the concurrency/race condition
  case, and if my boundary cases for 10,000 and 20,000 are correctly placed."
  — used to check for duplication and gap in Task 1.

- "I am testing the Parabank demo app. I need two functional bugs that are
  distinct in root cause. Suggest categories of things to probe — not specific
  bugs, because the app changes. Think: transfer logic, account state, session,
  input validation, fund availability." — used to focus Task 2 exploration.

- "Here is the FX API response shape from open.er-api.com. Help me write Postman
  test assertions that validate status, response time, schema, currency
  presence, and value sanity without being overly brittle." — used for Task 3
  Part A assertion design.

- "Here is the payments endpoint contract. Write a structured list of the
  highest-risk test scenarios for a money movement endpoint, organised by
  contract, idempotency, auth, amount/FX, limits, state machine, and
  observability. No code." — used for Task 3 Part B structure.

- "I am designing an automation approach for a dynamic-tab admin console where
  field identifiers change per session. Outline a Page Object Model structure
  that avoids a 900-line class and handles three control types behind one
  interface. Include a short illustrative locator snippet." — used for Task 4
  structure and snippet.

- "Here is a release scenario: go-live in 3 days, regression at 68%, two open
  criticals, UAT not signed off, FX deployed yesterday, half the team new. I
  want to write a no-go memo with precise conditions for a conditional go. Help
  me structure the 72-hour plan and the communication section." — used for Task
  5 framing.

---

## What I Kept from AI Output and Why

- **Task 1 risk ranking order.** The AI surfaced the duplicate-debit and
  limit-bypass-concurrency risks as the top two, which matched my own
  intuition. I kept that ordering because it aligns with financial-impact
  reasoning rather than testing convenience.
- **Task 3 Part B structure.** The seven-category organisation (contract,
  idempotency, auth, amount/FX, limits, state machine, observability) is a
  clean mental model I adopted because it maps directly to how a payments
  engineer thinks about the endpoint.
- **Task 4 folder tree.** The layered structure (framework/, pages/,
  config/, data/, tests/) is largely what I would have chosen, and the AI
  helped me articulate why each layer exists.
- **Task 5 72-hour priorities.** The AI reinforced the FPOC (first point of
  contact) ordering: stop the bleeding on flaky tests, get a read on the KYC
  criticals, then decide on UAT, then FX. I kept that ordering.

---

## What I Rejected from AI Output and Why

- **Over-long question lists.** Early drafts produced 20+ questions. I
  trimmed to 15 focused ones because the brief explicitly says quality beats
  quantity, and a real BA would disengage from a laundry list.
- **Generic "test the happy path" test cases.** The AI occasionally suggested
  cases that restated the requirement. I rejected those and replaced them with
  cases that target a specific root cause and a specific failure mode.
- **Tooling advocacy without trade-offs.** The AI sometimes presented Playwright
  as an unqualified choice. I rewrote those passages to name the trade-off I am
  accepting (less native mobile than Appium-based stacks, but far better web
  automation ergonomics and CI integration for this console, which is a web UI).
- **Task 5 go recommendation nuance.** An early AI draft leaned too quickly
  toward a conditional go. I pushed it back toward a clear no-go-as-is because
  the facts (two criticals, UAT not signed off, 68% regression, FX barely
  tested) do not support going, and saying otherwise would be hard to defend
  in an interview.

---

## What I Wrote or Rewrote Myself

- The entire README, including the assumptions table and the BA questions, was
  drafted by me and revised by me. The AI acted as a reviewer, not an author.
- Both bug reports in Task 2 were found, reproduced, and written by me from
  live exploration. The AI did not suggest bug titles or steps.
- The Postman collection in Task 3 was authored by me. The AI reviewed the
  assertion logic and suggested the cross-currency reconciliation check, which
  I then authored in my own words.
- The automation design document in Task 4 is my design. The AI contributed the
  locator-strategy snippet framing, but the structure, the "what we do not do"
  list, and the trade-off statement are mine.
- The quality strategy memo in Task 5 is my writing. The AI helped me sharpen
  the conditions for a conditional go and the metrics, but the reasoning and
  the specific decisions are mine.

---

## How I Would Use AI Differently Next Time

I would use AI earlier as a sounding board for my risk model — before I commit
to a question list or a test-case set — rather than as a reviewer after I have
already drafted. That would surface blind spots earlier and reduce the rework.
I would also ask it to challenge my assumptions explicitly ("which of your
assumptions is most likely to be wrong, and what would you test if it is?")
rather than asking it to generate content. The value I got was from having my
reasoning stressed, not from having it produced. Next time I would lean into
that more deliberately and ask for less generation and more critique.
