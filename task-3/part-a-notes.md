# Task 3 — Part A Notes

## Provider used

**Provider:** open.er-api.com (https://open.er-api.com/v6/latest/{baseCurrency})

**Why this provider:**

- Reachable from my environment on the day of testing without an API key.
- Returns a clean JSON shape with a top-level `result` field that distinguishes
  success from failure, which makes assertion design straightforward.
- Includes `time_last_update_unix` and `time_last_update_human`, which are useful
  for freshness checks and audit reference.
- Has a broad enough currency set to cover USD, EUR, GBP, SAR, and AED without
  special handling.

**What I would do if it were unavailable:**

Switch to the backup provider, api.frankfurter.app. I would then rewrite the
schema assertions to match frankfurter's response shape (which uses `base`
instead of `base_code`, nests rates differently, and does not include a
`result` field). The collection's logic and cross-check design would remain the
same; only the field names in the assertions would change. I would document the
switch in the collection description and in this file.

---

## Assertion table

| Assertion group | What it verifies | Why chosen |
|-----------------|------------------|------------|
| Status code | Response is 200 for valid base currencies | A non-200 from a rates service is a hard failure — the app cannot proceed with a rate lookup. Asserting 200 upfront fails fast and is the simplest gate. |
| Response time | Response time < 2000ms | A slow rates service becomes a bottleneck for the transfer flow. 2 seconds is a conservative threshold for a customer-facing lookup; a production app would tune this against its own SLA and the amount of time the customer is willing to wait on a transfer screen. |
| Top-level schema | Response has `result`, `base_code`, and `rates` | Validates that the provider returned the shape the app depends on. If any of these is missing, the app's parsing logic fails. This is a structural guard, not a business-rule check. |
| Result value | `result` == "success" | open.er-api.com uses `result` to signal success or failure. A `result` of "fail" means the lookup did not succeed even though the HTTP status is 200. This is the key signal for the negative path on this provider. |
| Base code echo | `base_code` matches the requested `baseCurrency` | Confirms the response is for the currency we asked for. A mismatch here — e.g. we request USD and get a response whose `base_code` is EUR — would be a subtle but serious bug. The app could display the wrong base currency to the customer. |
| Target currency presence and sanity | EUR, GBP, JPY, CAD, AUD, CHF, CNY, INR, SAR, AED are present and have values > 0 | These are the currencies the app is likely to need for an international transfer from a SAR-based account. A missing currency or a zero/negative rate would break a conversion. The check is broad enough to catch a partial outage where some currencies are missing. |
| Plausibility of captured EUR rate | Captured EUR rate is between 0.1 and 10.0 | A sanity bound that catches wildly implausible values — e.g. a service returning 0, null, or 1,000,000. Not a precise check; just a guard against obvious data corruption or a parsing error. |
| Cross-currency reconciliation | EUR-base response's `rates.USD` is within 0.5% of 1 / captured USD-to-EUR | Proves the captured value is used meaningfully. We do not just store `rates.EUR` in an environment variable and forget it — we use it to independently verify a later response. The 0.5% tolerance accounts for the fact that rates can move between the two calls, so the check is not brittle to normal rate movement. |
| Negative case — actual behaviour | For "INVALID", the API returns 200 with `result` = "fail" and an `error` field | We assert on what the provider actually does, not on what we wish it did. This is honest about the provider's behaviour and documents it so the assertion is not pretending the API returns a 4xx. |

---

## Why I captured rates.EUR

I captured `rates.EUR` because EUR is the most likely cross-currency anchor in a
SAR-based banking app's international transfer flow. The app's primary customer
currency is SAR, and the most common secondary currency for international transfers
is likely EUR or USD. By capturing EUR from a USD-base call, I create a value that
can be used in a subsequent request to verify consistency — in this collection, that
verification is the EUR-base cross-check (the second request), where `rates.USD` from
the EUR-base response should be approximately `1 / usdToEur`.

This is not an arbitrary capture for the sake of having a captured value. The point
of capturing a response value and using it in a later request is to demonstrate that
the collection is doing something useful with state, not just storing it for the sake
of it. The cross-check proves two things:

1. The captured value is correct enough to be relied on — if it were wildly wrong, the
   cross-check would fail.
2. The collection can use state from one request to validate another, which is the
   pattern that matters in a real collection — e.g. capturing an access token, a rate,
   a quote ID, or a transaction reference and using it in a subsequent step.

If I were building a collection for the actual international transfer flow, I would
capture the FX quote ID from a rate-lookup step and use it in the payment creation
step, then verify that the payment was created with the bound rate. The EUR capture
here is a simplified stand-in for that pattern.

---

## Negative-case finding and banking-app judgement

### What the API actually does

When I request `GET /v6/latest/INVALID`, open.er-api.com returns:

- HTTP status: **200**
- Body: `{"result": "fail", "error": "...", "base_code": "INVALID", "rates": {}}`

The API does **not** return a 4xx status code for an invalid currency. It returns
200 and signals failure inside the body via the `result` field.

### Is this acceptable for a banking app?

**Not ideally. I would prefer a 400-class response.** Here is the reasoning.

For a banking application that consumes this API, a 200-with-error is a riskier
contract than a 4xx response, for two reasons:

1. **Client error handling.** A client that checks only the HTTP status code will
   treat the failed lookup as a success and may proceed with no rate, a stale rate,
   or a default rate. In a money-movement context, that could mean an international
   transfer proceeds without a valid FX rate — which is exactly the kind of thing
   that causes financial discrepancies. A 400-class response forces the client to
   handle the failure explicitly before it proceeds.

2. **Operational visibility and monitoring.** A 200-with-error is harder to monitor
   and alert on than a 4xx. In a banking app, you want to know when a rates service
   is returning failures, and a 4xx series makes that visible at the HTTP layer
   without parsing every response body. With a 200-for-failure contract, every
   client and every monitoring tool must parse the body to discover that the request
   failed.

That said, the behaviour is not unsafe if the client is written correctly. A correct
client checks `result` regardless of status code, and the API is documented. The risk
is in clients that assume 200 means success — and in a banking context, you cannot
assume all clients are correctly written.

### My judgement

For the demo provider, the 200-for-invalid behaviour is acceptable as a demo —
it is discoverable, documented, and not unsafe if handled correctly. For a banking
app's production rates provider, I would push for a 400-class response for invalid
input, because the cost of a client misunderstanding a 200-with-error in a
money-movement flow is too high. If I could not change the provider, I would add a
defence-in-depth check in the client: the transfer flow requires both HTTP 200 **and**
`result` == "success", and treats anything else as a rate failure that blocks the
transfer. The collection asserts on the actual behaviour honestly, and the
part-b-payments-analysis.md discusses what I would require from the payments endpoint
that consumes this rate.
