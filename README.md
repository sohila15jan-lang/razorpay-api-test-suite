# Razorpay API Test Suite — QA Portfolio Project

A hands-on API testing investigation of Razorpay's live test-mode Orders, Customers, and Payments APIs, built by a QA engineer with 13 years' experience (8 of them testing interest-rate-derivative trade lifecycle systems at a UK investment bank) as a demonstration piece for freelance QA/test-strategy work — applying the same discipline (heavy negative/boundary testing, checking business state rather than status codes, treating documentation as a hypothesis to verify rather than ground truth) to a real, production-grade fintech API instead of a mock.

**Jump to:** [Key findings](#key-findings) · [What's covered](#whats-covered) · [Known limitations](#known-limitations) · [How to run this](#how-to-run-this)

## Key findings

Three confirmed discrepancies between Razorpay's documented behavior and its live test-mode behavior, each backed by a saved request/response pair. Full detail, evidence, and recommendations are in [`findings.md`](./findings.md).

1. A malformed request body (an unquoted variable, producing invalid JSON) was silently accepted with `200 OK`, creating a customer record with `name`, `email`, and `contact` all `null` — instead of being rejected. Reproduced twice, independently.
2. Customer names up to 100 characters were accepted, despite documentation stating a 50-character maximum — double the stated limit, with no rejection at either length tested.
3. A contact number below the documented 8-digit minimum was accepted without error.

These are the kind of gaps a validation-only test pass would miss, and the kind of finding a client is actually paying a consultant to surface — not "does the happy path work," but "does the system enforce what it claims to enforce."

## What's covered

| Area | Folder | What's tested |
|---|---|---|
| **Orders** | `orders/` | Trade-adjacent creation flow: required-field validation, amount/currency rules, notional-style boundaries, duplicate receipt handling, fetch/list/pagination/date filtering |
| **Customers** | `customers/` | Field validation (name, email, contact, notes) mapped directly to Razorpay's documented error messages, boundary pairs either side of stated limits, duplicate-customer detection, `customer_id` validation on both fetch (`GET`) and edit (`PUT`) |
| **Payments — Capture** | `payments/` | The full authorize → capture lifecycle: amount/currency mismatch, missing/blank/non-integer amount, double-capture prevention, invalid payment id, wrong-method routing. Requires a real payment generated via `generate-test-payment.html`, since payments can't be created by a direct API call in a standard test account |

## Known limitations

- **Capture collection cannot be re-run end-to-end.** Capturing a payment is a one-way action — once a payment is captured, every subsequent capture attempt against it returns "already captured," regardless of which case is being tested. Each full run needs a freshly generated payment; see `payments/README.md` for the steps.
- **Four documented Payment-capture error states were not tested**: "pending authorization from approver," "another operation in progress," "amount greater than amount due," and "order already paid." These require account states or race conditions that couldn't be reliably triggered in a standard test account, so they're listed here rather than faked with a test that doesn't actually prove anything.
- **The "customer belongs to a different merchant" variant** of the invalid-id error wasn't tested, since it requires a second Razorpay account.
- **The true upper bound on customer name length is still unknown.** 100 characters is the longest value confirmed accepted so far (see Finding 2 in `findings.md`) — not a confirmed ceiling.
- Several other assertions (the real 512-character notes-value boundary, the true `customer_id` malformed-vs-not-found split) were written from documentation and still need to be reconciled against a live run before being trusted — see the inline comments marked "record what you see" in each collection for exactly which ones.

## How to run this

1. Sign up for a free Razorpay account and switch to Test Mode (no KYC required) — see `customers/README.md` for the exact steps.
2. Generate a test Key Id and Key Secret from Settings → API Keys.
3. Import each collection's `.json` file into Postman, along with the shared environment in `environments/`.
4. Fill in `razorpayKeyId` and `razorpayKeySecret` in the environment — never commit real keys to the repo.
5. Run `orders/` and `customers/` collections via the Collection Runner, in order (later requests in several collections depend on ids saved by earlier ones).
6. For `payments/`, follow `payments/README.md` first to generate one real authorized payment before running the collection.

---

Most cases here are negative or boundary conditions, not happy-path checks — and every collection includes at least one case designed to test whether documentation matches reality, not just confirm it.
