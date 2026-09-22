# Task 5 — Quality Strategy & Release Judgement

**Status:** BONUS — Senior level optional. Included here as a Lead-level demonstration.
**Suggested time:** 30 minutes | **Actual time:** 35 minutes
**Format:** Memo

---

**To:** Delivery Manager
**From:** QA Lead
**Re:** International Transfer Journey — Go / No-Go Recommendation
**Date:** 3 working days before announced go-live

---

## 5.1 Go / No-Go Recommendation

**Recommendation: NO-GO as-is. Conditional GO only if the conditions below are met within the remaining 72 hours.**

I am not confirming "QA is green." I cannot, honestly. Here is why.

The release has four structural problems, any one of which would give me pause,
and which together make a go decision indefensible:

1. **Regression suite at 68%.** A regression suite that passes 68% of its tests
   is not measuring quality — it is measuring noise. Of the failing 32%, around
   30 are believed to be flaky, but nobody has investigated them this sprint.
   That means we do not know which failures are real. A suite that fails for the
   wrong reasons is worse than no suite at all, because it creates a false sense
   of control. I cannot sign off on a release whose primary quality signal is
   untrustworthy.

2. **Two open critical defects in onboarding/KYC.** These are not part of this
   release, but they share the customer profile service. That is the exact kind
   of shared-component dependency that turns a scoped release into a systemic
   problem. If the KYC criticals involve the profile service in a way that
   affects read or write paths used by the international transfer journey, those
   criticals are in our blast radius whether we like it or not. We do not know
   that they do not, because the cross-impact analysis has not been done. A
   release that goes live while a critical defect is open in a shared service it
   depends on is a release I would not put my name on.

3. **UAT is not signed off.** The client's business team has run about half their
   scripts. Half-run UAT is not UAT — it is a mid-point status. We do not know
   whether the remaining half passes, fails, or has not been started because the
   scripts are blocked by an environment issue. The client's business stakeholders
   and marketing team have already been told go-live is happening. If we go and
   the client's own tests were going to fail, we have not just shipped a bug — we
   have shipped a surprise to the people who own the business outcome.

4. **FX rate integration deployed to staging yesterday and only happy-path tested.**
   The FX integration is brand new in staging. Happy-path testing is one scenario
   — one currency pair, one amount, one successful rate lookup. It does not cover
   rate expiry, rate mismatch, fallback rates, rounding, failed rate lookups, or
   the behaviour when the rates service is slow or returns an error. International
   transfers depend on this service for the actual money movement. Shipping it with
   one round of happy-path testing is shipping untested integration code in a
   money-movement path.

Add to this that two of four QA engineers are new to the project (5 weeks), and I
do not have the experienced bench I would want for a high-stakes go decision with
72 hours left.

### Conditions for a conditional GO

I would reconsider a go only if, by the end of the next 72 hours, all of the
following are true:

- **Regression suite:** The 30 flaky failures have been triaged. At minimum, we
  know which are genuinely flaky (and have a plan to fix or quarantine them) and
  which are real defects. The suite's pass rate on *known-good* tests is at least
  95%, and no new real failures have been introduced by this release. I do not
  need 100%, but I need to know that the remaining failures are understood.

- **KYC criticals:** We have a written cross-impact analysis from the engineer
  responsible for the customer profile service, stating explicitly whether the two
  criticals affect any code path the international transfer journey uses. If they
  do, those criticals must be fixed or there must be a documented risk acceptance
  from the bank's business owner before we go. If they do not, I want that analysis
  in writing, not in a conversation.

- **UAT:** The client's business team has completed their scripts and signed off,
  or has explicitly accepted the outstanding items in writing with a agreed
  post-go-live remediation plan and dates. "Half done" is not acceptance.

- **FX integration:** We have done more than happy-path. At minimum: a failed rate
  lookup (what does the transfer do?), an expired rate quote, a rate that returns a
  value outside the expected range, and a concurrent transfer using the same quote.
  If the FX service is down or slow in staging, the transfer must behave correctly —
  not hang, not debit without a rate, not show a wrong amount.

- **Smoke test on the full journey:** A complete international transfer — from
  customer initiation through FX rate display, confirmation, debit, and credit — has
  been executed end to end in staging and the money has been verified on both sides.
  Not just "the API returned 201," but the ledger shows the correct debit and the
  beneficiary shows the correct credit in the correct currency and amount.

If any of these conditions is not met, my recommendation remains no-go.

---

## 5.2 Next 72 Hours — Priority Order

### What I would do, in order

1. **Stop and assess the regression suite.** Within the first 4 hours, I would get
   the 30 flaky failures triaged by whoever owns them — even if that means pulling
   the two newer QA engineers off other work for a few hours and pairing them with
   someone more experienced. The goal is not to fix all 30; it is to know which are
   noise and which are signal. A list of "30 failures, 27 flaky, 3 real, here they
   are" is more useful than "68% pass, probably okay."

2. **Get the KYC cross-impact analysis immediately.** This is the highest-stakes
   unknown. I would ask the person responsible for the customer profile service for
   a written assessment: do the two criticals touch the paths the international
   transfer journey uses? If the answer is "we don't know yet," that is itself an
   answer — it means we cannot rule out impact, and that pushes us toward no-go.

3. **Get UAT status and a clear path to sign-off.** Talk to the client's business
   lead. How many scripts are left? Are they blocked? What would it take to finish
   in 72 hours? If the remaining scripts are blocked by something we control, remove
   the block. If they are blocked by something we do not control, that is a go-live
   risk we need to surface now, not on day 3.

4. **Expand FX integration testing above happy-path.** The two newer QA engineers,
   with support from a more experienced engineer, run the FX failure and edge cases
   in staging. This is targeted and fast — a few hours of focused testing, not a
   full regression. The objective is to find the obvious failure modes before the
   client does.

5. **Run the full end-to-end smoke test.** One complete international transfer,
   verified on both sides of the money movement. If this fails, nothing else matters
   — we are not going.

6. **Risk-accept what we cannot fix.** For any remaining open items that are not
   critical and not in the money-movement path, document them, assign owners and
   dates, and get a written risk acceptance from the delivery manager and the client
   business lead. Not verbal — written, because verbal risk acceptance is forgotten
   the moment go-live pressure arrives.

### What I would drop

- **General regression of non-affected areas.** I would not run the full regression
  suite again hoping it passes more this time. I would focus on the items above. The
  suite is not trustworthy at the moment, and re-running it without triage just gives
  us a different unreliable number.

- **Polishing and non-critical defect fixing.** Any defect that is not in the
  money-movement path and not customer-facing-critical gets documented and deferred.
  We do not have time to fix everything, and pretending we do is how we miss the
  things that matter.

- **New feature work.** Nothing new is being started in the next 72 hours. The team
  is focused on the release decision and the conditions for it.

- **A full re-test of the entire FX provider integration.** The FX provider (the
  rates service we call) is not our code to re-test. We test our integration with it,
  not the provider itself. I would not spend time re-verifying the provider's rates —
  I would verify that our integration handles the provider's responses correctly.

---

## 5.3 Communication

### (a) To the Delivery Manager

I would write this in the same channel the go-live confirmation request came in —
the manager asked me, in writing, to confirm QA is green, so I respond in writing.

Message: **"I cannot confirm QA is green as the request stands. Here is the current
state: regression at 68% with 30 un-triaged flaky failures, two open criticals in a
shared service, UAT at approximately 50% and not signed off, and the FX integration
tested only on the happy path. None of these is necessarily a release-blocker on its
own, but together they mean I am not confident enough to recommend go. I have a
72-hour plan to resolve the unknowns, and a conditional-go checklist. If we can meet
the conditions, I will sign off. If we cannot, I will recommend a go-live delay and
will name the specific conditions that were not met. I am not asking for permission to
run this plan — I am asking for alignment that a conditional go is the target and that
the team will be supported in triaging the regression and getting the KYC
cross-impact analysis."**

Where this differs from the client message: With the delivery manager, I am explicit
about the internal numbers, the flaky-test problem, and the fact that QA is not green.
I am asking for support and alignment, and I am naming the conditions. The delivery
manager needs the unvarnished internal state to make the right call.

### (b) To the Client

The client message is different in tone and in what it reveals. The client does not
need to know that our regression suite is flaky — they will not understand the
nuance, and the right message to them is not "our tests are unreliable" but "we are
not ready to release on the announced date."

Message: **"We have completed our internal testing and identified a small number of
open items that need resolution before we can confidently go live with the
international transfer journey. These are not blocking bugs in the core transfer
flow, but they include integration points — specifically the FX rate service and the
customer profile service — that we need to validate end to end before go-live. We are
working through them now and expect to have a clear go-live recommendation within 72
hours. If we are not confident by then, we will recommend a short, specific delay with
a revised date rather than release something we cannot stand behind. We will keep you
updated daily."**

Where the messages must not differ: Both messages must agree that **we are not
confirming go-live today**, that **there are specific open items**, and that **the
decision will be made on evidence within 72 hours**. The delivery manager must not
hear "we are probably fine" while the client hears "we are not ready." That gap is
how trust is lost. The client must not hear a confident go-live promise from anyone
else that contradicts what QA has said.

Where the messages must not differ, part 2: If we do go, both messages must agree on
what was accepted, what is being monitored, and what the client should do if they see
an issue. A go decision communicated differently to the two audiences is a
reputational risk.

---

## 5.4 What I Would Change and What I Would Measure

### What I would change in how the team works

1. **Stop treating flaky tests as background noise.** A flaky test that is not
   investigated for a sprint is a test that has silently stopped protecting us.
   From now on, any test that fails intermittently is triaged within the sprint it
   fails — not left for "later." If it is genuinely flaky, it is either fixed,
   quarantined with a ticket, or removed. A flaky test in the regression suite is
   a liability, not a safeguard. This is a practice change, not a tooling change.

2. **Cross-impact analysis as a release gate, not a conversation.** Any release that
   touches a shared service must have a written cross-impact assessment from the owner
   of that service before go-live. Not a verbal "I think it's fine." A written
   assessment that names the specific code paths and says whether they are affected.
   This becomes a release checklist item, not something we remember to ask about.

3. **UAT as a tracked, visible state, not a status we infer.** UAT progress is
   visible to QA and to the delivery manager in the same tool, with the same
   definition of "done" as development. When the client says "about half," that
   translates to a concrete number of scripts completed and a list of what remains.
   We do not go live with "about half" as the UAT status. We either finish it or we
   do not go.

4. **New-service integration requires more than happy-path before it is "tested."**
   A new integration (like the FX service deployed yesterday) is not "tested" after
   one happy path. The minimum bar for "tested" on a money-movement integration is:
   success path, failure path, timeout/slow path, and a concurrency or reuse path.
   Whatever we call "tested" in the team, it includes those four. This is a practice
   change that prevents the exact situation we are in now.

5. **QA has a real say in the go-live decision, and the go-live criteria are written
   down before the release starts, not invented on the day.** The delivery manager's
   request for a "QA is green" confirmation should not be the first time we articulate
   what "green" means for this release. We should have agreed on the go-live criteria
   at the start of the release cycle. If we had, the gap between "68% regression" and
   "green" would have been visible weeks ago, not three days before go-live.

### Metrics I would put in place to make quality visible to the client

These are concrete and observable, not principles.

1. **UAT script completion and pass rate, reported weekly and at release readiness.**
   Not "about half" — a specific count: N of M scripts completed, P% pass, with a
   list of open failures and their severity. This is the client's own testing, made
   visible. The client sees their own progress and their own open items, in their
   terms.

2. **Regression suite trustworthiness:** the percentage of tests that pass
   consistently (not just once) over the last 5 runs, and the count of known-flaky
   tests that have not yet been resolved. This tells the client and the delivery
   manager whether the suite is a signal or noise. It also creates pressure to fix
   flaky tests, because they show up as a visible number.

3. **Release readiness checklist completion:** a written checklist with the go-live
   conditions (UAT sign-off, cross-impact analysis, new-integration test coverage,
   regression trustworthiness, open criticals) and a red/amber/green state for each,
   reviewed at the release-readiness meeting. Not a verbal status — a written
   checklist that both QA and the delivery manager sign off on. The client sees the
   checklist state for the items that affect them (UAT, open criticals, integration
   status).

### What I would NOT measure

- **Test case count.** The number of test cases we have is not a quality metric. A
  team can write 1,000 trivial test cases and still ship a defective release. I would
  not report test count to the client as if it meant anything.

- **Test execution speed or number of tests run per day.** These measure throughput,
  not quality. A fast test suite that misses the important cases is worse than a slow
  one that catches them.

- **Defect count without context.** "We found 40 defects" means nothing without
  severity, age, and whether they are open or fixed. A raw defect count encourages
  the wrong behaviour — closing defects to improve the count rather than fixing the
  important ones.

- **Code coverage.** Code coverage is a developer metric that is easily gamed and
  does not tell the client whether the money movement is correct. I would not show it
  to the client as a quality signal. It may be useful internally, but it is not a
  client-facing quality metric for a release decision.
