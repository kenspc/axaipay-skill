# Axaipay pitfalls

Each entry gives the symptom you see, the cause, and how to confirm it. Facts
apply to eComPay API v1.19 r1 and carry the labels defined in
[api.md](api.md#how-each-fact-was-established).

## 1. Payment page shows a generic "unexpected error" right after the redirect

- **Symptom:** the browser reaches Axaipay, which answers with a generic message
  such as "Unexpected error when processing init payment request" instead of the
  payment method picker.
- **Cause we found:** the signature was hex-encoded. Axaipay expects Base64.
  Observed during integration on 2026-04-05. Note that this did **not** come back
  as PY016 (invalid signature), so the message does not point at the signature.
- **How to verify:** run your signing function on the test vectors in
  [api.md](api.md#test-vectors). A 128-character hex string instead of an
  88-character Base64 string is this bug. Other signature mistakes may produce the
  same generic message; we have not tested which ones do.

## 2. Inquiry returns PY016 (invalid signature)

- **Cause, any of:** fields concatenated in the wrong order; `mchtId` sent in
  the body instead of `merchantId`; a malformed signature; the wrong signing key.
  The first three were verified by live call on 2026-07-31.
- **How to verify:** probe with a nonexistent order id (see
  [api.md](api.md#probing-inquiry-safely)). A correct request answers `UAP`.
  Change one thing at a time until PY016 turns into UAP.

## 3. Inquiry returns IAP, and fixing the signature changes nothing

- **Cause:** the API key is wrong. Axaipay checks the key before the signature,
  so while the key is wrong you never see signature errors. Verified by live call
  on 2026-07-31.
- **How to verify:** fix the key first, then re-run the probe. Expect `UAP` or
  `PY016`, never `IAP`.

## 4. Inquiry returns MAP or MSI

- **Cause:** a missing header. `MAP` means no `x-api-key` header (verified by
  live call on 2026-07-31). `MSI` means no `x-signature` header (from the vendor
  document, Appendix B). Both values go in headers, not in the body.

## 5. Inquiry returns HTTP 404

- **Cause:** the request went to the payment host (`secured.` or `staging.`
  with `/gateway/v1/...`). Inquiry lives on `api.` / `api-staging.` under
  `/trxn/inquiry`. Verified by live call on 2026-07-31.
- **How to verify:** compare your configured inquiry URL with the hosts table in
  [api.md](api.md#hosts).

## 6. Inquiry code reports success for an order that does not exist, or crashes

- **Cause:** the code checks `data != null` instead of root `success`. A failed
  inquiry still returns a `data` object that echoes your order id. Verified by
  live call on 2026-07-31.
- **Second cause:** the parser expects strings, but `trxnStatus`,
  `trxnAcquirer` and `trxnPaymentMethod` are typed Integer (from the vendor
  document, §3.4). A strict string read throws, and a broad `catch` turns every
  inquiry into a failure.
- **How to verify:** probe with a nonexistent id. Your code must report "not
  found" (`UAP`), not success. Feed it a sample response with numeric fields.

## 7. Inquiry code has never run

- **Symptom:** there is an inquiry method, but nothing calls it, so none of the
  bugs above can surface.
- **Cause:** the hosted page and callbacks are enough to take payments, so the
  inquiry path is easy to leave untested. Integration practice: treat code with
  no callers as unverified.
- **How to verify:** call it once against staging with a nonexistent id, and once
  with a real staging order id, and compare its output with the raw JSON.

## 8. Payments go through, but no callback ever arrives (suspected)

- **Suspected cause:** the configured URL points at one environment while the
  signing key or API key belongs to the other. Because the Merchant ID can be the
  same on both (confirmed by the merchant account holder on 2026-07-31), checking
  `mchtId` does not catch this. This cause was not confirmed by a controlled test.
- **How to verify:** at startup, log which environment the configured host
  points to (see SKILL.md, hard rule 4), and confirm the keys came from the same
  environment's merchant portal.

## 9. Customer looks logged out on the payment result page

- **Cause:** Axaipay returns the customer to your redirect URL with a cross-site
  form POST. Browsers do not send `SameSite=Lax` or `Strict` cookies on a
  cross-site POST, so your auth cookie is missing on that one request. Observed
  during integration on 2026-04-12.
- **Fix:** let the redirect endpoint accept anonymous requests and skip CSRF
  checks, verify the signature, keep the order id, then redirect the browser to
  a GET page. The GET is a same-site navigation, so the cookie
  is sent again. Integration practice.

## 10. Valid callbacks fail signature verification

- **Causes to check:** the amount was parsed and re-formatted (`49.9` instead of
  `49.90`); `trxnBuyerName` was included (it is not signed, from the vendor
  document §1.6); fields were taken in the wrong order; a missing field became
  the text `null` instead of an empty string. The empty-string rule is
  integration practice; the vendor document does not specify it.
- **How to verify:** log the field names that were posted (not their values in
  production), rebuild the string from the raw values, and compare with the
  callback test vectors in [api.md](api.md#test-vectors).

## 11. Staging test card is declined

- **Cause:** an expiry date copied from an older version of the document. v1.19
  r1 changed it to `08/30`. From the vendor document (revision history and
  Appendix A).

## 12. A payment configuration error takes the whole site down

- **Cause:** startup validation that throws when a payment setting is missing or
  looks wrong. The site then fails to start, and nothing else works either.
- **Fix:** log the problem loudly (a warning naming the derived target, such as
  "Unknown") and let the site start. Fail the payment action itself, not the
  process. Integration practice.

## 13. Callback URL sits behind a redirect

- **Risk:** your site redirects one host to another, for example `www.` to the
  bare domain with HTTP 301, and the redirect or backend URL uses the host that
  redirects. A 301 or 302 answer to a POST is usually followed with a GET and no
  body, so the callback fields never reach your handler. Whether Axaipay's
  server follows redirects at all is not documented.
- **Prevention:** register and send callback URLs on the final host, the one
  that answers 200 without redirecting. Integration practice; we set it up this
  way from the start rather than after a lost callback.
- **How to verify:** `curl -s -o /dev/null -w '%{http_code}\n' -X POST <backend URL>`
  must not print 301, 302, 307 or 308.

## 14. PY022 on a retry

- **Cause:** the same `mchtTrxnId` was sent twice. Generate a new one for every
  payment attempt and link attempts to your order. From the vendor document
  (Appendix B).
