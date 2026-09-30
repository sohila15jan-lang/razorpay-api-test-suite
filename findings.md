# Findings — Documentation vs. Live Behavior

Three discrepancies between Razorpay's documented API behavior and its actual behavior in test mode, found while building the collections in this repo. Written in the same format a client engagement would use: what was expected, what actually happened, evidence, and a recommendation.

---

## Finding 1: Malformed request body silently accepted, creating an empty customer record

**Severity:** High — this is a data-integrity issue, not a cosmetic one.

**Endpoint:** `POST /v1/customers`

**Steps to reproduce:**
1. Send a request body where a Postman variable is used without quotes around it, e.g. `"name": {{name100}},` instead of `"name": "{{name100}}",` — producing syntactically invalid JSON once the variable is substituted in.
2. Send the request.

**Expected:** A `400 Bad Request`, since the body isn't valid JSON and none of the required fields (`name`, `contact`, `email` — one of the latter two is mandatory) can be read from it.

**Actual:** `200 OK`. A new customer record was created, with `name`, `email`, and `contact` all `null`:
```json
{
  "id": "cust_ThNhPjPqpKI608",
  "entity": "customer",
  "name": null,
  "email": null,
  "contact": null,
  "gstin": null,
  "notes": [],
  "shipping_address": [],
  "created_at": 1790581997
}
```

**Reproduced a second time, independently:** while attempting an unrelated boundary test (a 100-character name, see Finding 2), the same unquoted-variable mistake was made again, and produced the identical result — `200 OK`, customer `cust_TiE6oSnDEdY5Ng` created with `name`, `email`, and `contact` all `null`. This wasn't a deliberate re-test; it was an accidental repeat of the same slip while testing something else. That's worth noting on its own: a mistake this easy to make by accident, twice, independently, is exactly the kind of input a real client integration is likely to produce sooner or later, not just a hypothetical edge case.

**Why it matters:** If a client's front-end has a bug that produces malformed JSON (a missing quote, a bad template substitution — exactly the kind of mistake this test accidentally reproduced, twice), the API does not protect them from it. Instead of a clear rejection they can catch and log, they silently accumulate empty, useless customer records with no way to distinguish them from a legitimate API error further upstream. This is the kind of defect that surfaces weeks later as "why do we have thousands of blank customers," not at integration time.

**Recommendation:** Do not rely on the API to reject malformed bodies with clear errors. Client-side/application-layer validation should confirm the request body is well-formed *before* sending it, and a post-creation check (does the returned customer actually have the fields you sent?) should be part of any integration test suite that calls this endpoint.

---

## Finding 2: Customer name accepted well beyond the documented 50-character limit

**Severity:** Medium — a data-validation gap, not a data-integrity one.

**Endpoint:** `POST /v1/customers`

**Documented rule:** "The name may not be greater than 50 characters" (400 error).

**Steps to reproduce:**
1. Send a valid, well-formed request with `"name"` set to a 76-character string: "This Is A Deliberately Very Long Customer Name That Exceeds Fifty Characters".

**Expected:** `400 Bad Request`, per the documented rule.

**Actual:** `200 OK`. The customer was created with the full 76-character name stored exactly as sent. Confirmed twice, independently, with a clean and well-formed request body both times:
```json
{
  "id": "cust_TiAHK4AQ2sUmao",
  "entity": "customer",
  "name": "This Is A Deliberately Very Long Customer Name That Exceeds Fifty Characters",
  "email": "longname.1790753074@example.com",
  "contact": "+919198123679"
}
```

**Confirmed at a second, longer length.** After two invalidated attempts (see Finding 1 — both hit the malformed-body bug instead of actually testing this), a clean, well-formed request with a **100-character** name (a repeated-character string, to isolate length from format) was also accepted:
```json
{
  "id": "cust_TiEAmeGDycoDvy",
  "entity": "customer",
  "name": "AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA",
  "email": "longname.1790766789@example.com",
  "contact": "+9198123235"
}
```
So the documented 50-character limit is not just exceeded slightly (76 characters) — it isn't enforced at double that length either (100 characters). **Still open:** the true maximum, if one exists at all, is unknown — 100 characters is simply the longest value tested so far, not a confirmed ceiling. Testing progressively longer values (e.g. 500, 2000 characters) would be a reasonable next step to see if a limit exists anywhere.

**Why it matters:** If a client's UI or downstream system (a printed invoice, a fixed-width export column, a database field defined at 50 characters) assumes the API enforces this limit, they may hit a truncation or overflow bug in their own system that the API's documentation implied couldn't happen.

**Recommendation:** Do not treat the documented character limit as enforced at all — evidence so far shows it isn't, at least up to 100 characters. Any client system with its own length constraint on customer name should validate independently rather than relying on this endpoint to do it.

---

## Finding 3: Contact number accepted below the documented 8-digit minimum

**Severity:** Medium

**Endpoint:** `POST /v1/customers`

**Documented rule:** "Contact number should be at least 8 digits, including country code" (400 error).

**Steps to reproduce:**
1. Send a valid, well-formed request with `"contact": "+9112645"` — 7 digits total, including the `91` country code.

**Expected:** `400 Bad Request`, per the documented rule.

**Actual:** `200 OK`. The customer was created with the 7-digit contact number stored exactly as sent:
```json
{
  "id": "cust_TiDxZY1REC5U07",
  "entity": "customer",
  "name": "Rahul Mehta",
  "email": "rahul.1790766038@example.com",
  "contact": "+9112645",
  "gstin": null,
  "notes": [],
  "shipping_address": [],
  "created_at": 1790766039
}
```

**Why it matters:** A client relying on this endpoint to validate phone number length before using it downstream (SMS delivery, WhatsApp notifications, OTP verification) will not get the protection the documentation implies. A 7-digit number accepted here could fail silently later, at the point of actually trying to contact the customer — further from where the bad data was introduced, and harder to trace back to its source.

**Recommendation:** Do not rely on this endpoint to validate contact number length. Any client system that depends on a minimum-length phone number (particularly one sending SMS/OTP) should validate independently before or after calling this endpoint.

---

## Notes on methodology

All three findings are now backed by saved response bodies from live test-mode requests. Finding 2 is confirmed at two lengths (76 and 100 characters); whether any upper limit exists at all remains an open question, and is reported as such rather than assumed. A client-facing report should never overstate confidence in an unverified result — reporting "this looks like a gap, pending confirmation" is more credible, and more useful, than quietly upgrading it to a confirmed defect before the evidence supports it.
