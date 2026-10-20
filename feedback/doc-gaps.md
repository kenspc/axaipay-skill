# Where integrators get stuck: suggestions for the eComPay API document

These notes come from one integration against **eComPay API Technical
Documentation v1.19 r1**. Each entry describes a place where we lost time, what
the document says today, and wording that would have saved us. Section numbers
refer to v1.19 r1. The suggested wording is only a starting point.

## 1. Signature encoding is shown, not stated

- **Where integrators get stuck:** the prose says the signature is "encrypted
  using HMACSHA512", but not how the bytes become text. A hex-encoded signature
  looks plausible, and the gateway answers it with a generic "unexpected error"
  page rather than PY016, so nothing points back at the encoding.
- **What the document says now (§1.4, §2.3, §3.3):** the encoding is visible only
  in the PHP and Java samples (`base64_encode`, `Base64.getEncoder()`).
- **Suggested wording:** "The signature is the HMAC-SHA512 of the UTF-8 string,
  keyed with the UTF-8 signing key, encoded as standard Base64 (88 characters,
  ending in `==`). Hex encoding is not accepted."

## 2. Payment and inquiry use different hosts

- **Where integrators get stuck:** it is natural to assume inquiry lives next to
  payment, for example `secured.axaipay.my/gateway/v1/inquiry`. That URL returns
  404, which reads like an outage rather than a wrong address.
- **What the document says now (§1.2, §2.1, §3.1):** the correct URLs are given,
  but in three separate sections, so the difference is easy to miss.
- **Suggested wording:** a single table near the start: "Hosted payment page:
  `staging.axaipay.my` / `secured.axaipay.my`, path `/gateway/v1/payment`.
  Server-to-server APIs (inquiry, QR): `api-staging.axaipay.my` /
  `api.axaipay.my`, paths under `/trxn/`. These are different hosts."

## 3. How the environment is chosen

- **Where integrators get stuck:** developers look for an environment flag, or
  use the Merchant ID to tell staging from production. The Merchant ID can be
  the same in both, so a staging URL paired with production keys is not caught
  by any check.
- **What the document says now:** nothing explicit. The examples say they were
  "performed in staging environment".
- **Suggested wording:** "There is no environment parameter. The URL host alone
  decides whether a request goes to staging or production. Your Merchant ID may
  be identical in both, but the signing key and API key are per environment.
  Always change the URLs and keys together."

## 4. `mchtId` versus `merchantId`

- **Where integrators get stuck:** the payment form uses `mchtId`. The inquiry
  body uses `merchantId`. Sending `mchtId` to inquiry returns PY016 (invalid
  signature), which sends people to debug the signature instead of the field
  name.
- **What the document says now (§3.2, §3.3):** the body tables say `merchantId`,
  but the example in §3.3.1.2 lists the value as `mchtId = …`, and the PHP
  comment in §3.3.2.1 also labels it `mchtId`.
- **Suggested wording:** a callout in §3.2: "Note: the inquiry body field is
  `merchantId`, not `mchtId` as on the payment form." Align the §3.3 examples
  and code comments with it.

## 5. Two different sample keys in one example

- **Where integrators get stuck:** checking an implementation against the
  inquiry-by-`mchtTrxnId` example gives a mismatch, because the prose and the
  code use different sample signing keys.
- **What the document says now (§3.3.1.2 versus §3.3.2.2):** the prose example
  uses one sample key; the PHP and Java samples for the same case use another.
- **Suggested wording:** use one sample key throughout §3.3, and state the
  expected signature for each code sample.

## 6. What an inquiry failure looks like

- **Where integrators get stuck:** code written from the success sample checks
  whether `data` is present. On failure `data` is still present (an unknown
  `mchtTrxnId` is echoed back inside it), so a failed lookup can be read as a
  success.
- **What the document says now (§3.4):** only the success response is shown.
  Appendix B lists error codes without saying which endpoint returns which, or
  in what order they are checked.
- **Suggested wording:** add a failure sample and these lines: "Check `success`
  first. On failure, `errorCode` and `message` describe the problem and `data`
  may still be present. An unknown transaction returns `UAP`. The API key is
  checked before the signature, so `IAP` can hide a signature error."

## 7. Callback delivery rules

- **Where integrators get stuck:** developers have to guess what the backend URL
  should return, whether Axaipay retries, how many times, and whether both URLs
  are always called. The redirect also arrives as a cross-site POST, so session
  cookies with `SameSite=Lax` are missing and the customer appears logged out.
- **What the document says now (§1.3, §1.5):** the two URLs and the field list.
  Nothing on expected response, retries, timing, or ordering.
- **Suggested wording:** "Respond to the backend URL with HTTP 200 within N
  seconds. If you do not, Axaipay retries up to N times over N minutes. The
  backend call may arrive before or after the browser redirect. The redirect is
  a cross-site form POST: verify the signature, then redirect the browser to a
  page of your own." (Fill in Axaipay's actual values.)

## 8. Absent fields in the response signature

- **Where integrators get stuck:** `trxnBankName`, `trxnFpxMethod` and
  `trxnFpxType` apply only to FPX. For a card or e-wallet payment it is not
  stated whether they are omitted, sent empty, or sent as `null`, and how they
  enter the signature string.
- **What the document says now (§1.5, §1.6):** the signed field list, with an
  FPX example only.
- **Suggested wording:** "Fields that do not apply to the payment method are
  sent as empty strings and contribute an empty string to the signature." (Or
  whatever the actual rule is.) Add a worked example for a card payment.

## 9. Amount format in the string to sign

- **Where integrators get stuck:** `4.25` versus `4.250`, `49.9` versus `49.90`,
  or a comma as decimal separator in some locales. Any of these breaks the
  signature.
- **What the document says now (§1.3):** the type `Decimal(12,2)` and examples.
- **Suggested wording:** "Send and sign `txnAmount` with exactly two decimal
  places and a dot as the separator, with no thousands separator, for example
  `1250.00`. Verify the callback signature over `trxnAmount` exactly as
  received."

## 10. Small inconsistencies between sections

- **Where integrators get stuck:** parsers written from one section break on
  data described in another.
- **What the document says now:**
  - The FPX buyer name is `trxnBuyerName` in the callback (§1.5) but
    `trxnFpxBuyerName` in inquiry (§3.4).
  - `trxnFpxType` is `CASA` or `LCA` in §1.5 but `CASA` or `Card` in §3.4.
  - `trxnAmount` is typed Integer in §3.4 with the sample `110.00`.
  - The action text for PY055 (invalid redirect URL) mentions the backend URL.
- **Suggested wording:** use one name and one value set per field across all
  sections, type amounts as Decimal, and fix the PY055 text to "redirect URL".

## 11. The public copy is older than the current version

- **Where integrators get stuck:** the knowledge base article links v1.18 r2,
  while v1.19 r1 changes the test card expiry date and the acquirer list.
  Someone working from the public copy can fail staging card tests because the
  older expiry date has passed.
- **What the document says now:** the revision history in v1.19 r1 lists the
  changes, but only people with v1.19 r1 can see it.
- **Suggested wording:** publish the current version in the knowledge base, keep
  the revision history at the top, and add a one-line "last updated" note on the
  article itself.

## 12. AutoDebit availability

- **Where integrators get stuck:** the knowledge base still lists the AutoDebit
  API v1.6 document, and eComPay v1.19 r1 still lists `eMandate` as a possible
  `trxnFpxMethod` value, but Axaipay told us AutoDebit is discontinued. Teams can plan and build on a product they cannot use.
- **What the document says now:** nothing about availability.
- **Suggested wording:** a note at the top of the AutoDebit article: "AutoDebit
  is not available to new integrations. Contact support@axaipay.com for
  recurring payment options."
