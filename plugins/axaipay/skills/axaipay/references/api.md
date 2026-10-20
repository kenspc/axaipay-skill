# Axaipay eComPay API reference (v1.19 r1, paraphrased)

This file restates, in our own words, the parts of Axaipay's eComPay API that an
integration needs. It applies to **eComPay API v1.19 r1**. It is not a copy of
the vendor document. When this file and the vendor document disagree, ask
Axaipay which one is current.

The public copy of the vendor document is on Axaipay's knowledge base:
<https://support.axaipay.com/support/solutions/articles/73000523456-axaipay-ecompay-api-technical-document-v1-18>.
That page carried **v1.18 r2** when this file was written. v1.19 r1 was obtained
from Axaipay support (support@axaipay.com).

## How each fact was established

Every fact below carries one of these labels:

| Label | Meaning |
|---|---|
| verified by live call on DATE | A direct request to Axaipay on that date produced the stated result. |
| observed during integration on DATE | Seen while building a real integration and recorded when it was fixed; not a controlled test. |
| confirmed with Axaipay, as of DATE | Axaipay told us by call or email. |
| confirmed by the merchant account holder on DATE | Stated by the person holding the merchant account, for that one account. |
| from the vendor document (§n) | Stated in eComPay API v1.19 r1, section n. Not independently tested. |
| integration practice | Our own design choice. Axaipay does not state it. |

## Scope

- **Hosted payment page** (browser form POST to Axaipay): covered.
- **Callbacks** (Axaipay posts the result back to you): covered.
- **Inquiry** (server-to-server status lookup): covered.
- **QR code transaction**: exists in v1.19 r1 (§2). We did not use it; see the
  short note at the end and read the vendor document.
- **Void**: v1.19 r1 removed the Void section. From the vendor document (revision history).
- **AutoDebit (eMandate)**: documented separately (AutoDebit API v1.6); eComPay
  v1.19 r1 only lists `eMandate` as a possible `trxnFpxMethod` value. Axaipay
  told us it has been discontinued, and it was not available to us. Confirmed
  with Axaipay, as of 2026-04-05. Confirm with Axaipay before building on it.

## Credentials and environments

You receive three values per environment:

| Value | Used for |
|---|---|
| Merchant ID | `mchtId` on the payment form; `merchantId` in the inquiry body. |
| Signing key | HMAC key for every signature you send and verify. |
| API key | `x-api-key` header on inquiry (and QR) calls. Not used on the payment form. |

- **The API has no environment parameter.** The only thing that selects staging
  or production is the URL host you send to. From the vendor document (no such
  field exists in §1–§3).
- **The Merchant ID can be identical on staging and production.** It therefore
  cannot tell you, or your callback handler, which environment you are talking
  to. Confirmed by the merchant account holder on 2026-07-31.
- Keep the URL, signing key and API key of one environment together as one set.
  Integration practice.

### Hosts

| API | Staging | Production | Source |
|---|---|---|---|
| Hosted payment page | `https://staging.axaipay.my/gateway/v1/payment` | `https://secured.axaipay.my/gateway/v1/payment` | from the vendor document (§1.2) |
| Inquiry | `https://api-staging.axaipay.my/trxn/inquiry` | `https://api.axaipay.my/trxn/inquiry` | from the vendor document (§3.1); both verified by live call on 2026-07-31 |
| QR code generate | `https://api-staging.axaipay.my/trxn/qrcode/generate` | `https://api.axaipay.my/trxn/qrcode/generate` | from the vendor document (§2.1) |

The inquiry host and path differ from the payment host and path. The guessed
form `https://secured.axaipay.my/gateway/v1/inquiry` returns HTTP 404. Verified
by live call on 2026-07-31.

## Signatures

All signatures use the same recipe:

1. Take the listed field **values** in the listed order. The order is the
   alphabetical order of the field **names**.
2. Concatenate the values with no separator.
3. Compute HMAC-SHA512 with the signing key.
4. Encode the 64-byte result as **standard Base64** (with `+`, `/` and `=` padding).

Base64 output: from the vendor document (§1.4 code samples; the prose does not
name the encoding). Hex output fails, see pitfalls. Observed during integration
on 2026-04-05.

UTF-8 for both key and data: integration practice. The vendor document does not
state a character encoding. The vendor's own worked examples are plain ASCII.

### Payment request signature (hosted payment page)

Order: `backendUrl`, `customerEmail`, `customerName`, `customerPhone`, `mchtId`,
`mchtTrxnId`, `orderDescription`, `redirectUrl`, `txnAmount`.

- `backendUrl` and `redirectUrl` take part **only when you send them**. If you
  omit a field from the form, omit it from the string.
- `paymentMethod` and `acquirerCode` (the bypass fields) **never** take part.

From the vendor document (§1.3, §1.4).

- Send the amount with exactly two decimals and a dot, for example `49.90`, and
  sign the exact same string you post. The two-decimal shape is from the vendor
  document (§1.3 type `Decimal(12,2)` and its examples). Formatting with an
  invariant culture is integration practice.

### Callback signature (redirect URL and backend URL)

Order: `mchtId`, `mchtTrxnId`, `trxnAcquirer`, `trxnAmount`, `trxnBankName`,
`trxnFpxMethod`, `trxnFpxType`, `trxnId`, `trxnPaymentMethod`, `trxnStatus`,
`trxnTime`.

- `trxnBuyerName` is posted for FPX payments but is **not** part of the signature.
  From the vendor document (§1.5, §1.6).
- Use the raw posted strings. Do not parse the amount and format it again.
  Integration practice.
- A field that was not posted (for example the FPX-only fields on a card
  payment) contributes an empty string. Integration practice; the vendor
  document does not say how absent fields are treated.

### Inquiry request signature

| Inquiry by | Order | Source |
|---|---|---|
| `mchtTrxnId` (your order id) | `mchtTrxnId`, `merchantId` | from the vendor document (§3.3); verified by live call on 2026-07-31 |
| `trxnId` (Axaipay's id) | `merchantId`, `trxnId` | from the vendor document (§3.3) |

The signature goes in the `x-signature` header, not in the body.

### Inquiry response signature

Same 11 fields and order as the callback signature. In the inquiry response the
buyer name field is called `trxnFpxBuyerName`, and it is not signed either. From
the vendor document (§3.4, §3.5).

### Test vectors

These vectors are ours. They use the made-up key `example-signing-key`. They
were computed with Python's `hmac` module, and the same code reproduced the
vendor document's own worked examples. Check your implementation against these
before you send anything to Axaipay.

| Case | String to sign | Expected signature |
|---|---|---|
| Payment, no optional URLs | `buyer@example.comJane Tan60123456789DEMO_MERCHANTORDER-1001Pro plan49.90` | `wOiP8ZtCkJUYRdFXRo9+cRpizDyx4xghuLnIL4JnFkw+XZUolH29wVwplFhn4QhQ0O38KChrQHpq8+SshCq9rg==` |
| Payment, both URLs | `https://shop.example.com/pay/callbackbuyer@example.comJane Tan60123456789DEMO_MERCHANTORDER-1001Pro planhttps://shop.example.com/pay/return49.90` | `pxrrleA9ujrdy442mesKHQ8kcZGRZxlvB57CzAUJvtbiwlI43IVMv9ZlJr2ncgVG2Lu++AWKIqgGJ133fOZjVw==` |
| Callback, FPX | `DEMO_MERCHANTORDER-10012149.90OCBC BankB2C (Retail Banking)CASA20260101120000123201120260101120000` | `6FvdQyQN+9S5IJX45JOBP3t8cw6UZZZ8+iuhiijxqw7GsRS0C5mH/dRJRG/i/ksExJrdD3FlAOulsUovaWK6cw==` |
| Callback, card (FPX fields absent, as empty strings) | `DEMO_MERCHANTORDER-1002149.902026010112050045611120260101120500` | `Pz7JJkJwWYs6cD9/jamnypSi0+RjKTdgxs+J7PMPkkQHI9D7TT2miTf9glRd2ebgQ/cBWv5GhkwwoWKh90ONfg==` |
| Inquiry by `mchtTrxnId` | `ORDER-1001DEMO_MERCHANT` | `vVtdLnMiWLC426vnzp9JLBfE/r1NHlGAjHK8xdlKFr9zINRJU6Y+coqTNRQNvH/LFfgro/lOJSUgzjdqrScZ+Q==` |
| Inquiry by `trxnId` | `DEMO_MERCHANT20260101120000123` | `W8NBzq5ULUUrFLaOakrLiIm7oz3kXSwxKesuacjJInkATjEW8BNYhon+UpMAAsATybyp70rYX1Q1/pUvmv5ekg==` |

The card row only tests your concatenation code. It does not prove how Axaipay
treats absent fields.

Axaipay also runs a signature comparison page on staging at
`https://staging.axaipay.my:8888/generate-signature`. From the vendor document (§1.4).

## Hosted payment page

The browser submits an HTML form with `method="POST"` to the payment host. It is
not a JSON API. Render a page with a hidden form and submit it with a script,
keeping a visible submit button for browsers without JavaScript.

### Form fields

| Field | Required | Notes |
|---|---|---|
| `mchtId` | yes | Merchant ID. |
| `mchtTrxnId` | yes | Your transaction id. Must be unique per attempt; a reused id is rejected with PY022. |
| `txnAmount` | yes | Two decimals, for example `49.90`. |
| `customerEmail` | yes | |
| `customerName` | yes | |
| `customerPhone` | yes | Digits. Without a country code it is treated as a Malaysian number. |
| `orderDescription` | yes | |
| `redirectUrl` | no | Overrides the default redirect URL registered with Axaipay. `http` or `https`. |
| `backendUrl` | no | Overrides the default backend URL registered with Axaipay. `http` or `https`. |
| `signature` | yes | See "Payment request signature". |
| `paymentMethod` | no | Skips Axaipay's method picker: `OB` online banking, `EW` e-wallet. Not signed. |
| `acquirerCode` | no | Bank or wallet code to skip straight to. On staging use `B2C_TEST` or `B2B_TEST`. Not signed. |

From the vendor document (§1.3). The full bank code list is in the vendor
document's Appendix C.

## Callbacks

Axaipay calls two URLs when a payment finishes. You register default values with
Axaipay, and a payment request can override them. From the vendor document (§1.3).

| URL | Who calls it | Use it for |
|---|---|---|
| Redirect URL | The customer's browser, as a form POST | Showing the result page. |
| Backend URL | Axaipay's server, as a form POST | Recording the payment. This is the source of truth. |

The vendor document gives one list of posted fields for both (§1.5):

| Field | Meaning |
|---|---|
| `mchtId` | Merchant ID. |
| `mchtTrxnId` | Your transaction id. |
| `trxnId` | Axaipay's transaction id. |
| `trxnAmount` | Amount in MYR. |
| `trxnTime` | `yyyyMMddHHmmss`. |
| `trxnStatus` | Status code (table below). |
| `trxnAcquirer` | Acquirer code (table below). |
| `trxnPaymentMethod` | Payment method code (table below). |
| `trxnFpxType` | FPX only: `CASA` bank transfer, `LCA` card. |
| `trxnBankName` | FPX only: bank name. |
| `trxnFpxMethod` | FPX only: `B2C (Retail Banking)` or `B2B (Corporate Banking)`. |
| `trxnBuyerName` | FPX only: payer name. Not signed. |
| `signature` | See "Callback signature". |

What the backend URL must return, and whether Axaipay retries, is not stated in
the vendor document. We log every backend call and return HTTP 200, including
calls that fail verification. Integration practice.

The redirect is a cross-site POST, so cookies set with `SameSite=Lax` or
`Strict` are not sent with it. Observed during integration on 2026-04-12. See
pitfalls.

## Inquiry

`POST` to the inquiry host, body `application/x-www-form-urlencoded`.

| Part | Content | Source |
|---|---|---|
| Header `x-api-key` | API key | vendor document (§3.2); verified by live call on 2026-07-31 |
| Header `x-signature` | Inquiry request signature | vendor document (§3.2); verified by live call on 2026-07-31 |
| Body, by order id | `mchtTrxnId`, `merchantId` | vendor document (§3.2); verified by live call on 2026-07-31 |
| Body, by Axaipay id | `merchantId`, `trxnId` | vendor document (§3.2) |

The body field is `merchantId`, not `mchtId` as on the payment form. Sending
`mchtId` returns PY016. Verified by live call on 2026-07-31.

### Response

```json
{
  "success": true,
  "errorCode": null,
  "message": "…",
  "data": { "mchtTrxnId": "…", "trxnId": "…", "trxnStatus": 11, "trxnAmount": 49.90, "trxnPaymentMethod": 20, "…": "…" }
}
```

- `success`, `errorCode` and `message` are at the root. Transaction fields are
  inside `data`. From the vendor document (§3.4); verified by live call on 2026-07-31.
- **Gate on root `success`.** `data` is not null on failure: an unknown order id
  returns `success: false`, `errorCode: "UAP"` and `data` holding only the
  `mchtTrxnId` you sent. Verified by live call on 2026-07-31.
- `trxnStatus`, `trxnAcquirer` and `trxnPaymentMethod` are typed Integer, so
  expect JSON numbers, not strings. From the vendor document (§3.4). Parse them
  so that both a number and a numeric string are accepted. Integration practice.
- `trxnFpxType` is described as `CASA` or `Card` here, but as `CASA` or `LCA` on
  the callback. Accept both. From the vendor document (§1.5, §3.4).

### Probing inquiry safely

Inquiry is read-only. A request for an order id that does not exist moves no
money, and the error code tells you which part of your request is wrong.
Verified by live call on 2026-07-31.

| What you send | What comes back | What it means |
|---|---|---|
| A correct request for a nonexistent id | `UAP` | Host, headers, body and signature are all accepted. |
| Wrong signature order, a garbage signature, or `mchtId` instead of `merchantId` | `PY016` Invalid signature | The signature really is checked. |
| No `x-api-key` header | `MAP` Missing API key | |
| A wrong API key | `IAP` Invalid API key | Returned **before** the signature is checked, so it hides signature errors. Fix the key first. |
| The payment host instead of the inquiry host | HTTP 404 | Wrong host or path. |

## Code tables

Paraphrased from the vendor document (§1.5, §3.4, Appendix B).

### `trxnStatus`

| Code | Meaning | Final? (integration practice) |
|---|---|---|
| 1 | Created | no |
| 2 | Being processed | no |
| 3 | Waiting for authorisation | no |
| 11 | Paid | yes, success |
| 22 | Failed | yes |
| 23 | Timed out | yes |
| 33 | Voided | yes |
| 44 | Refunded | yes |
| 55 | Cancelled | yes |

Only 11 means the money was taken.

### `trxnPaymentMethod`

| Code | Method |
|---|---|
| 1 | Visa |
| 2 | Mastercard |
| 3 | Debit card |
| 10 | E-wallet |
| 20 | FPX |
| 30 | Instalment |
| 50 | UnionPay |
| 60 | DuitNow QR |

### `trxnAcquirer`

| Code | Acquirer |
|---|---|
| 1 | Credit card |
| 10 | MCash |
| 11 | Boost |
| 12 | GrabPay |
| 13 | Touch 'n Go |
| 14 | WannaPay |
| 15 | Alipay |
| 16 | Alipay+ |
| 17 | ShopeePay |
| 18 | WavPay |
| 21 | CIMB |
| 40 | UnionPay |
| 50 | DuitNow QR |

### Error codes

"Seen" marks codes we produced ourselves by live call on 2026-07-31. The rest
are from the vendor document only.

| Code | Meaning | What to check | Seen |
|---|---|---|---|
| PY016 | Signature rejected | Field order, signing key, encoding, field names | yes (inquiry) |
| PY022 | Duplicate transaction | Use a new `mchtTrxnId` for every attempt | |
| PY034 | Amount rejected | Amount missing or outside the allowed range | |
| PY035 | Merchant transaction id rejected | `mchtTrxnId` missing | |
| PY036 | Customer name rejected | `customerName` missing | |
| PY037 | Customer email rejected | `customerEmail` missing | |
| PY054 | Backend URL rejected | Not a valid `http`/`https` URL | |
| PY055 | Redirect URL rejected | Not a valid `http`/`https` URL | |
| I02 | Order description rejected | `orderDescription` missing | |
| IAP | API key not recognised | `x-api-key` value; hides signature errors | yes |
| MAC | API account rejected | Merchant ID | |
| MAP | API key header missing | Add `x-api-key` | yes |
| MSI | Signature header missing | Add `x-signature` | |
| QRG | QR code could not be generated | Retry or contact Axaipay | |
| UAP | Unexpected error | On inquiry for an unknown id it means "not found". Otherwise retry or contact Axaipay. | yes (unknown id) |
| UAQ | Unknown acquirer | `acquirer` on QR requests | |

## Staging test cards

From the vendor document (Appendix A, v1.19 r1). Card holder name `Test Card`,
expiry `08/30`, CVV `123`. v1.19 changed the expiry date, so older
copies of the document show a different one.

| Scenario | Visa | Mastercard | 3-D Secure password |
|---|---|---|---|
| Succeeds without a challenge | 4100000000000100 | 5100000000000107 | none |
| Succeeds after a challenge | 4100000000005000 | 5100000000005007 | 123456 |
| Fails authentication | 4100000000300005 | 5100000000300002 | 111111 |

For FPX on staging, pick the test bank (`B2C_TEST` or `B2B_TEST`).

## QR code transaction (not exercised by us)

A server-to-server alternative to the hosted page: you POST to
`/trxn/qrcode/generate` on the inquiry host with `x-api-key` and `x-signature`,
and display the returned `qrCodePayload`. The request signature order is
`acquirer`, `backendUrl` (only when sent), `customerEmail`, `customerName`,
`customerPhone`, `mchtId`, `mchtTrxnId`, `orderDescription`, `txnAmount`. The
response signature order is `mchtId`, `mchtTrxnId`, `qrCodePayload`,
`trxnAcquirer`, `trxnAmount`, `trxnId`. From the vendor document (§2). Read §2
before building on it.
