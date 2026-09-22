# Task 4 — Automation Design (No Code)

**Scenario:** Automate regression coverage for the Bank Operations Console — the internal screen where bank staff configure onboarding rules, KYC thresholds, transfer limits, fees, notifications, channels, and audit settings.

**Objective:** Assess automation design thinking, not coding ability. This document contains no full implementation — only structure, strategy, small illustrative snippets, and explicit trade-off statements.

**Constraint:** The production console has a Save button and persists configuration across a page reload. The attached HTML reference page is a visual reference only — it has no Save button and does not persist anything. This design is written against the production behaviour described in the brief, not against the reference page.

**Suggested time:** 35 minutes | **Actual time:** 45 minutes

---

## Overview and constraint

The console's technical challenge: multiple tabs, dynamic field identifiers (e.g. `slot="field-145"`) that change per session, and three control types (text input, checkbox, dropdown) behind a single repeatable wrapper structure. The wrapper is locatable by visible label text or a unique descriptive attribute, but the child control's identifier is unstable.

The core design question: **how do you write automation that remains stable when the thing you would naturally want to latch onto — the field's identifier — is guaranteed to change every session?**

The answer is: never latch onto the dynamic identifier. Latch onto the stable wrapper instead, and treat the wrapper as a contract between the application and the automation.

---

## 1. Project Structure and Page Object Model Design

### Folder tree

```
bank-operations-console-automation/
├── pom/                      # Page Object Models — one file per tab, one shared field module
│   ├── console-base.ts       # Shared helpers: wrapper locator, field interaction, wait strategies
│   ├── console-header.ts     # Top-level nav: tab switching, save, reload
│   ├── tab-onboarding.ts     # Onboarding rules tab — fields specific to this tab
│   ├── tab-kyc.ts            # KYC thresholds tab
│   ├── tab-limits.ts         # Transfer limits tab
│   ├── tab-fees.ts           # Fees tab
│   ├── tab-notifications.ts  # Notifications tab
│   └── tab-audit.ts          # Audit settings tab
├── config/
│   ├── test-data.ts          # Data definitions: which fields to set, to what, for which scenario
│   ├── environments.ts       # Environment config: base URL, credentials, RUN_ID
│   └── selectors.ts          # Stable selector definitions — label text and descriptive attributes, never slot values
├── framework/
│   ├── field-wrapper.ts       # FieldWrapper interface + implementations for text/checkbox/dropdown
│   ├── tab-manager.ts        # Tab switching with explicit wait for the target tab content to stabilise
│   ├── persistence-verifier.ts # Reads back values from the config panel/display after reload
│   └── test-runner.ts        # Thin orchestration: load config, apply, save, reload, verify, teardown
├── data/
│   ├── scenarios/            # Per-scenario data files (JSON): which fields, what values, expected post-reload values
│   └── baseline-snapshots/   # Optional: captured original values for restore-after-test
├── tests/
│   └── configuration-persistence.test.ts  # The actual test: applies a scenario, saves, reloads, verifies
├── utils/
│   ├── unique-id.ts          # RUN_ID generation — unique per run, used to avoid value collisions
│   └── cleanup.ts            # Restore original values if the environment is shared and we cannot reset the DB
├── package.json
├── tsconfig.json
└── playwright.config.ts
```

### What a page object is here

A page object in this design is **not** "one class per screen." The console is a single screen with multiple tabs, and each tab has the same repeated structure. A page object here is a **tab object** — a module that knows how to:

- locate each configurable field on that tab by its stable wrapper (label text or descriptive attribute),
- interact with it through the shared `FieldWrapper` interface regardless of whether the child is a text input, checkbox, or dropdown,
- report its current value for verification.

The tab object does **not** know how to save, reload, or verify persistence — that is in the orchestration layer (`test-runner.ts`) and the persistence verifier (`persistence-verifier.ts`). This keeps each tab object focused on one tab's fields and nothing else.

### How tabs and repeated field types fit

Each tab file exports a class (or a set of pure functions, depending on taste) that exposes one method per configurable field — or one parametrised method if the tab has many fields of the same shape. For example, a KYC tab with 8 thresholds might expose:

```
setKycThreshold(fieldName: string, value: number)
getKycThreshold(fieldName: string): Promise<number>
```

The important thing is that the caller does not need to know whether `fieldName` maps to a text input, a checkbox, or a dropdown. That is hidden behind `FieldWrapper`.

### How to avoid a 900-line class

A 900-line page object usually happens when a single class absorbs tab switching, field interaction, data setup, assertions, and teardown. This design avoids that by:

- **Separating tab contents from tab navigation.** `console-header.ts` handles switching; each `tab-*.ts` handles only its own fields.
- **Separating interaction from verification.** `pom/` interacts; `persistence-verifier.ts` reads back after reload. A tab object does not assert.
- **Sharing the field interface.** `field-wrapper.ts` is the single place that knows how to detect a control type and act on it. Tab objects delegate to it.
- **Pulling data out.** `config/test-data.ts` and `data/scenarios/*.json` define what to set. The page objects receive values; they do not invent them.

A tab with 20 fields is still 20 field definitions, but they are 20 lines of data mapping, not 20 blocks of locator-plus-interaction-plus-assertion code.

---

## 2. Locator Strategy

### The core idea

Every configurable item follows the same stable parent-wrapper structure. The wrapper can be located by its visible label text or by a unique descriptive attribute. The child control inside the wrapper has a dynamically generated identifier (`slot="field-145"`) that must not be used as the primary locator.

So the locator strategy is:

1. **Find the wrapper by stable text or attribute.** This is the durable anchor.
2. **Within the wrapper, find the child control by its ARIA role or element type, not by its dynamic identifier.**
3. **Interact with the child through the FieldWrapper interface, which abstracts over text/checkbox/dropdown.**

### What we use, and what we never use

| Use | Never use |
|-----|-----------|
| Wrapper located by `data-field-description` descriptive attribute (e.g. `[data-field-description="kyc-threshold-min-age"]`) when present | `slot="field-145"` or any dynamically generated identifier as the primary locator |
| Wrapper located by visible label text (e.g. the element whose text content is "Minimum age") when no stable attribute exists | Absolute XPaths that count into the DOM structure |
| Child control located by ARIA role within the wrapper: `role="textbox"`, `role="checkbox"`, `[role="combobox"]` or `<select>` | Child dynamic id, name, or class that changes per session |
| Dropdown options located by visible text or value within the combobox/select context | Option index (e.g. "third option") — order can change |

### Why this works for three control types behind one interface

The wrapper is the same regardless of child type. The child type is detectable:

- Text input → `role="textbox"` or `<input type="text">` or `<input type="number">`
- Checkbox → `role="checkbox"` or `<input type="checkbox">`
- Dropdown → `role="combobox"` with `aria-expanded`, or a `<select>` element

Once we know the type, we act accordingly: `fill` for text, `check`/`uncheck` for checkbox, `selectOption` for dropdown. The caller does not need to know which — they call `fieldWrapper.set(value)` and the wrapper detects the type and acts.

### Illustrative snippet (TypeScript, Playwright)

This is a short illustrative snippet showing the wrapper-anchor and type-detection approach. It is not a full implementation.

```typescript
// pom/console-base.ts — shared field interaction

import { Page, Locator } from '@playwright/test';

// Find the wrapper by its stable descriptive attribute or its label text.
// NEVER locate by the child's slot="field-..." value.
function wrapperLocator(page: Page, description: string): Locator {
  // Prefer a stable data-testid if the production console provides one.
  // Fall back to locating the element whose visible label matches.
  const byAttr = page.locator(`[data-field-description="${description}"]`);
  if (byAttr.count() > 0) return byAttr;

  // Fallback: find the label element with this text, then the wrapper that contains it.
  return page.locator('label').filter({ hasText: description }).locator('..');
}

// Detect the child control type within the wrapper and return a type-specific locator.
type FieldType = 'text' | 'checkbox' | 'dropdown';

function detectFieldType(wrapper: Locator): FieldType {
  // Check in order of specificity. The production console uses one of these per wrapper.
  if (wrapper.locator('[role="checkbox"], input[type="checkbox"]').count() > 0) return 'checkbox';
  if (wrapper.locator('[role="combobox"], select').count() > 0) return 'dropdown';
  if (wrapper.locator('[role="textbox"], input[type="text"], input[type="number"]').count() > 0) return 'text';
  throw new Error(`Cannot detect field type inside wrapper for "${wrapper}"`);
}

// Set a value through the detected control type.
export async function setFieldValue(
  page: Page,
  description: string,
  value: string | boolean | number
): Promise<void> {
  const wrapper = wrapperLocator(page, description);
  const type = detectFieldType(wrapper);
  const field = wrapper.locator(
    '[role="textbox"], [role="checkbox"], [role="combobox"], ' +
    'input[type="text"], input[type="number"], input[type="checkbox"], select'
  ).first(); // scoped inside the wrapper

  switch (type) {
    case 'text':
      await field.fill(String(value));
      break;
    case 'checkbox':
      const desired = Boolean(value);
      const isChecked = await field.isChecked();
      if (isChecked !== desired) await field.click();
      break;
    case 'dropdown':
      await field.selectOption(String(value));
      break;
  }
}
```

A new engineer adding a field writes one line in a data file mapping the field description to its value — they do not write a new locator or a new branch. If a field needs special handling (e.g. a date picker), they add a specialised case in `setFieldValue` or a separate method on the tab object, not a one-off locator in the test.

---

## 3. Handling Dynamic DOM

### What we do

- **Re-resolve locators at use time, not at page-load time.** We do not store a Locator from one tab and reuse it after a tab switch. Each interaction re-finds the wrapper from the stable description. If the underlying element has been replaced, the locator is fresh.
- **Explicit wait for tab content to stabilise after a tab switch.** When we switch to a tab, we wait for a known stable element on that tab (e.g. the tab's heading or the first configurable field's wrapper) to be visible and attached before we interact. We do not rely on network idle alone — the fields may be rendered client-side after the tab header appears.
- **Auto-wait on actions.** Playwright's built-in actionability checks (visible, enabled, attached) handle most race conditions for free. We do not add manual `sleep()` calls as a first resort — we add them only when we have evidence that an action is timing out and auto-wait is not enough (rare, and a sign to investigate the app's rendering, not to paper over it).
- **Wait for the save confirmation and for the reload to complete.** After Save, we wait for the save indicator (toast, spinner clearance, or a known post-save element) before reloading. After reload, we wait for the console to be back in its initial state before we read back values.

### Stale reference avoidance

The dynamic identifier problem (e.g. `slot="field-145"` changing) is avoided entirely by not using those identifiers. Our locators are relative to stable wrappers, so even if every child control gets a new identifier on reload, our locator still finds the right wrapper and the right child by role.

The one case where stale references can still bite us is if we hold a reference to a locator across a tab switch and the tab's content is replaced. We avoid this by resolving locators inside the method that uses them, not in a constructor or a `beforeEach`. Each test step re-resolves.

### What we do NOT do

- **Do not use sleep-based waits as the primary strategy.** `await page.waitForTimeout(2000)` is a smell. We use it only as a last resort after we have observed a real timing issue and determined that auto-wait and explicit waits are not sufficient.
- **Do not locate by dynamic slot values even if they are convenient.** `page.locator('[slot="field-145"]')` works today and breaks tomorrow when the session regenerates identifiers. That is exactly the fragility the brief warns about.
- **Do not assume elements exist before they are rendered.** If a field only appears after a tab switch or a scroll, we wait for it to be attached and visible before interacting. We do not assume the tab's fields are all present the moment the tab header is clicked.
- **Do not scroll blindly to "make elements visible."** We scroll only when an element is known to be off-screen and the app requires scroll to bring it into the viewport. We do not scroll the whole page in the hope that something appears — that introduces flakiness and makes the test order-dependent.
- **Do not pollute tests with retry loops for DOM elements.** If a test needs to retry finding an element, that is a signal that either the app is rendering slowly or the locator is wrong. We fix the root cause; we do not mask it with retries.
- **Do not use XPath position-based selectors (`//div[3]/div[2]/input`)**. These break on any DOM structure change and are impossible to maintain across 7+ tabs with 20+ fields each.

---

## 4. Test Data Strategy

### No hard-coded values

All field values come from data files (`data/scenarios/*.json`) and config (`config/test-data.ts`). The test reads a scenario, applies the values, saves, reloads, and verifies. To change what we test, you change the data file, not the test code.

A scenario file looks roughly like:

```json
{
  "name": "kyc-and-limits baseline",
  "runIdSuffix": "baseline-01",
  "tabs": {
    "onboarding": {
      "max accounts per customer": 5,
      "default onboarding channel": "mobile"
    },
    "kyc": {
      "minimum age": 18,
      "document verification required": true
    },
    "limits": {
      "daily transfer limit": 20000,
      "per transaction limit": 10000
    }
  }
}
```

The test iterates over the tabs and fields in the scenario and applies them through the tab objects. This is how we cover 12+ fields across 4+ tabs without writing a bespoke test per field.

### Unique values per run

Where a value must be unique per run (e.g. a configuration name, a notification endpoint, a test-only fee description), we append a `RUN_ID` generated at the start of the run. The `RUN_ID` is stable for the whole run and is used consistently across all scenarios. This avoids collisions with other teams running tests in the same shared environment.

### Shared environments, no DB reset

The brief says test environments are shared with other teams, data is restricted, and we cannot reset the database on demand. This shapes the design in three ways:

1. **Isolation by value, not by environment.** We cannot rely on a clean environment for each test. We isolate our test's effects by using unique values (RUN_ID suffixes) and by targeting fields that other teams are unlikely to be configuring at the same time. Where possible, we configure fields that are specific to our scenario and unlikely to collide with another team's work.

2. **Capture baseline, restore after.** Before we change a field, we read its current value (if the console exposes it) and store it as the baseline. After the test, we restore the original value. If the console does not expose the current value for a field, we document that field as "cannot auto-restore" and flag it for manual cleanup or a follow-up with the environment owners.

3. **Cleanup is part of the test, not an afterthought.** The `utils/cleanup.ts` module runs after each scenario and attempts to restore every field we changed. If restoration fails for a field, the test logs it as a cleanup warning and the run records which fields were not restored, so that a human can follow up. We do not leave the environment dirty and hope someone else cleans it up.

### What the data strategy is not

It is not a full data-generation framework. It does not create customers, accounts, or beneficiaries — that is out of scope for a console-configuration test. It configures the console's own settings and verifies they persist. Test data for the console is the settings themselves, not the underlying customer data.

---

## 5. Reusability and Scalability

### Adding a 5th tab

To add a 5th tab, a new engineer creates `pom/tab-<name>.ts` that exports the tab's field set using the same `setFieldValue`/`getFieldValue` interface from `console-base.ts`. They add the tab's selector definitions to `config/selectors.ts`. They add a scenario entry to `data/scenarios/`. They do **not** touch the orchestration layer, the persistence verifier, or the existing tab objects. The new tab plugs in through the same interface.

### Adding a 20th field

Adding a field to an existing tab means adding one entry to the tab's field mapping (description → selector hint if needed) and one line in the scenario data file if the field is part of a scenario. If the field follows the same wrapper structure, `setFieldValue` already handles it — no new code path. If the field is a new control type (e.g. a date picker or a multi-select), the engineer adds a case to `detectFieldType` and `setFieldValue`, and that case then works for every tab that uses that control type — not just one tab.

### Execution time, parallelism, and CI as the suite grows

- **Execution time.** With 7 tabs and ~20 fields each, a full configuration-through-reload verification is one save, one reload, and one read-back per scenario. That is fast — seconds per scenario, not minutes. The expensive part is the reload, and we cannot avoid that, but we do not add extra round-trips.
- **Parallelism.** Scenario-based tests are independent — each scenario applies its own values, saves, reloads, and verifies. They can run in parallel against the same environment as long as each scenario uses unique values (RUN_ID) and restores cleanly. We use Playwright's built-in parallel test execution with a worker count tuned for the environment's capacity. We do not run all scenarios in parallel against a single shared console session — each worker gets its own browser session.
- **CI integration.** The suite runs as a Playwright test job in CI. On every merge to the configuration branch, the suite runs against the staging console. On a schedule (e.g. nightly), it runs against the integration environment to catch configuration drift. The job fails if any scenario fails to persist correctly after reload, or if cleanup fails to restore a field.
- **Reporting.** Playwright's HTML report gives per-test results, trace, and screenshot on failure. For a configuration suite, the most useful report is: which fields did not persist, and what did they revert to. The persistence verifier records the before-save value, the after-reload value, and the expected value, and reports any mismatch with all three — that is what a developer or a bank ops person needs to diagnose a failure.
- **Flaky-test quarantine.** If a test fails intermittently due to environment conditions (slow render, shared environment noise), we quarantine it — mark it as flaky in the runner, keep it in the suite, but exclude it from the blocking CI gate until the cause is found. We do not let flaky tests silently pass or silently block. A flaky configuration test is especially dangerous because it can hide a real persistence failure.

### What scales poorly and how we avoid it

What does not scale is a test that hard-codes 20 field interactions per tab and 7 tabs, with 7 different save-and-reload flows duplicated. That is the 900-line page object anti-pattern, and we avoid it by making the orchestration data-driven: one test function, many scenarios. Adding a field or a tab is a data change, not a code change. That is the scalability lever.

---

## 6. Tooling

### Stack

- **Playwright (TypeScript).** Primary automation library. Chosen for: strong auto-wait and actionability checks (reduces flakiness with dynamic DOM), good tab and frame handling, built-in parallel execution, HTML report and trace viewer for debugging failures, and reliable locator strategies that work well with the wrapper-based approach described above. TypeScript gives us type safety on the scenario data and the field mappings, which matters when a suite grows to 20 tabs and 100+ fields — a mistyped field description is caught at compile time rather than as a vague runtime failure.

- **Playwright test runner.** Built-in test runner, parallel by default, with project configuration for different environments (staging vs. integration) and sharding for large suites.

- **Config-driven scenarios in JSON.** Not a framework feature — a design choice. JSON scenarios are readable by a non-engineer (a bank ops person could review them), easy to extend, and decouple what we test from how we test it.

### The trade-off I am accepting

The trade-off is that Playwright is a **web** automation tool. The Bank Operations Console is described as an internal screen — a web UI — and Playwright is an excellent fit for that. But if the console were a desktop application, or if it had a heavy native-client component, Playwright would not cover it, and we would need a different or additional tool (e.g. a desktop automation tool or a driver for the native layer).

I am accepting that trade-off because the brief describes a console with tabs, text fields, checkboxes, and dropdowns — a web UI — and for that, Playwright's ergonomics, auto-wait, and parallel execution give more value than the marginal coverage a broader tool would add. If the console later gains a native component, we revisit. I am not pretending Playwright is universal; I am choosing it because it fits the stated problem and accepting the cost that it would not fit an unstated one.

A secondary trade-off: config-driven data is flexible, but it depends on the console exposing enough structure (stable wrappers, ARIA roles, descriptive attributes) to be locatable the way we need. If the production console does not provide stable descriptive attributes and we must fall back to visible-label matching, the suite becomes more brittle to copy changes. I would flag that as a risk to the development team and push for `data-field-description` attributes on the wrappers as a small, high-value contract between the app and the automation. That is the honest cost of the approach: it works well if the app cooperates, and it needs the app to cooperate.
