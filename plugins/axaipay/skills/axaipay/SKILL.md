---
name: axaipay
description: Integrate the Axaipay eComPay payment gateway (Malaysia) into a website or backend - hosted payment page form POST, HMAC-SHA512 Base64 signatures, redirect and backend callbacks, and the server-to-server inquiry API. Use when writing, reviewing, or debugging Axaipay code or config, when an Axaipay call returns PY016, UAP, MAP, IAP or an "unexpected error" page, when choosing between Axaipay staging and production, or when payments or callbacks do not arrive. Unofficial; based on one integration against eComPay API v1.19 r1.
---

# Axaipay eComPay integration

This skill is one developer's integration experience, written down so an agent
gets an Axaipay integration right the first time. It is **not** official
Axaipay material. Facts apply to **eComPay API v1.19 r1**, and each one in the
references says how it was established. When the vendor document or Axaipay
support says otherwise, they win; tell the user about the difference.

Read these as needed:

- [references/api.md](references/api.md): hosts, field lists, signature orders,
  test vectors, inquiry, code tables, test cards.
- [references/pitfalls.md](references/pitfalls.md): symptom, cause, how to verify.
- [references/dotnet-example.md](references/dotnet-example.md): a minimal ASP.NET
  Core example. It is one way to do it, not a required shape. Follow the user's
  own stack and the conventions of their codebase.

## Hard rules

1. **Signatures are HMAC-SHA512 over the concatenated field values, encoded as
   standard Base64.** Never hex. Each endpoint has its own field order; take it
   from api.md, never from memory, and never reuse one endpoint's order for
   another.
2. **Sign exactly the strings you send.** Optional `redirectUrl` and `backendUrl`
   are signed only when sent. `paymentMethod` and `acquirerCode` are never
   signed. Format the amount once, with two decimals and a dot, and use that
   one string for both the form field and the signature.
3. **Only the URL host selects the environment.** There is no environment
   parameter. Do not infer the environment from the Merchant ID: it can be the
   same on staging and production. Keep each environment's URL, signing key and
   API key together as one set.
4. **At startup, log the environment derived from the configured host**
   (Production, Staging, or Unknown at warning level). Never throw at startup
   because of payment configuration; fail the payment action instead.
5. **The payment host and the inquiry host differ.** Payment:
   `staging.axaipay.my` / `secured.axaipay.my` under `/gateway/v1/payment`.
   Inquiry: `api-staging.axaipay.my` / `api.axaipay.my` under `/trxn/inquiry`.
6. **The backend callback is the source of truth.** Verify the signature before
   reading anything else, check `mchtId`, `mchtTrxnId` and the amount against
   your own order, and record the payment idempotently keyed on `trxnId`
   (callbacks can repeat). The redirect callback only shows a page.
7. **Inquiry authenticates with headers**: `x-api-key` and `x-signature`. The
   body is form-encoded and uses `merchantId`, not `mchtId`.
8. **Gate inquiry results on root `success`.** `data` is present even on
   failure. Read numeric fields (`trxnStatus`, `trxnAcquirer`,
   `trxnPaymentMethod`) as numbers, accepting numeric strings too.
9. **Only `trxnStatus` 11 means paid.**
10. **Never log or commit the signing key or API key**, and keep them out of the
    browser. The payment form carries only the signature.
11. **Do not build on AutoDebit (eMandate)** unless Axaipay confirms it is
    available again; see api.md.

## Workflow

Do the steps in order. Each step ends with a check that would fail if the step
were wrong.

### 1. Gather configuration

Ask the user for, per environment: Merchant ID, signing key, API key, and the
default redirect and backend URLs they registered with Axaipay. Store keys in
the platform's secret store or environment variables, never in source. Put the
two hosts per environment in config, not in code branches.

**Check:** the config for staging contains only `staging.` / `api-staging.`
hosts, and the one for production only `secured.` / `api.` hosts.

### 2. Write the signing function first

One function: `sign(stringToSign, signingKey) -> Base64`. Then one small
function per endpoint that builds the string in that endpoint's order.

**Check:** a unit test reproduces every test vector in
[api.md](references/api.md#test-vectors). Do not continue until it passes.

### 3. Start a payment (hosted payment page)

Create a fresh `mchtTrxnId` for each attempt and link it to the order. Render an
HTML page with a hidden form that POSTs the fields to the payment host,
submitted by script, with a visible submit button as a fallback. HTML-encode
every value.

**Check:** on staging, the browser reaches Axaipay's payment method page. A
generic error page here usually means a signature problem (pitfalls 1).

### 4. Handle the backend callback

An endpoint that accepts an anonymous, form-encoded POST and skips CSRF checks.
Read the raw form values, verify the signature (constant-time compare), check
the order, then record the result idempotently on `trxnId` inside a database
transaction. Respond HTTP 200.

**Check:** post the "Callback, FPX" test vector's fields with the example key
configured: accepted. Change one character of `trxnAmount`: rejected. Post the
same valid callback twice: one payment recorded.

### 5. Handle the redirect callback

Same verification as step 4, but it only decides what to show. Because the
browser arrives by cross-site POST, auth cookies may be missing: verify, keep
the outcome (or just the order id) briefly, and redirect to a GET result page. If the backend callback has not arrived yet, show "processing" or
call inquiry.

**Check:** after a staging payment, the user lands on the result page still
logged in.

### 6. Implement inquiry, and run it

Build the inquiry call (by `mchtTrxnId` unless the user needs `trxnId`).

**Check:** probe with a nonexistent order id. The expected answer is `success:
false` with `errorCode` `UAP`; any other code points at the mistake (table in
api.md). Then query a real staging order and compare your parsed output with the
raw JSON field by field.

### 7. Log the target at startup

Derive Production / Staging / Unknown from the configured payment host and log
it once.

**Check:** start the app with each config and read the line.

### 8. End-to-end on staging

Pay with each staging test card (success without challenge, success with
challenge, failed authentication) and with the FPX test bank.

**Check:** for each, the backend callback recorded the right status, the
redirect page matched it, and inquiry agrees.

### 9. Before going live

- Production config uses production hosts and production keys, as one set.
- The startup log reads "Production".
- Redirect and backend URLs registered with Axaipay (or sent per request) point
  at the production site over HTTPS.
- One small real payment, then inquiry on it.

## When something fails

Match the symptom in [pitfalls.md](references/pitfalls.md) first. If nothing
matches, use the inquiry probe to separate host, header, body and signature
problems before changing code. If the API behaves differently from api.md, tell
the user, and suggest they confirm with Axaipay support (support@axaipay.com).
