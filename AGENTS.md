# Rules for maintaining this repo

This repo packages one developer's Axaipay eComPay integration experience as an
agent skill. These rules apply to every change, by a person or an agent.

## Language

Everything in this repo is in English: docs, code, comments, commit messages.

## Never add

- **Anything that identifies the merchant this was learned on:** its name,
  domain, business, product names, or merchant ID. Examples stay generic
  (`DEMO_MERCHANT`, `shop.example.com`, `ORDER-1001`).
- **Incident history:** outages, release or version numbers, dates of a
  merchant's own releases, who did what. Keep the lesson, drop the story.
- **Text, tables or code copied from Axaipay's documents.** Paraphrase the
  fact in your own words and link to the source. Do not copy the vendor's worked
  examples or sample keys; write your own test vectors with an obviously fake key.
- **Secrets:** real signing keys, API keys, merchant IDs, tokens, passwords.
- **AutoDebit (eMandate) as a supported feature.** It is mentioned once, in
  `references/api.md`, as unavailable. Change that only when Axaipay confirms it
  is available again.
- **A LICENSE file.** The future owner of this repo decides the license.

## Every fact carries its version and its source

State the API version a fact applies to (currently **eComPay API v1.19 r1**)
and one label from this list:

| Label | Use when |
|---|---|
| `verified by live call on <date>` | A direct request to Axaipay produced the result. The date is when the call was made. |
| `observed during integration on <date>` | Seen in a real integration and recorded when fixed, without a controlled test. |
| `confirmed with Axaipay, as of <date>` | Axaipay said so by call or email. |
| `confirmed by the merchant account holder on <date>` | The account holder stated it, for their account. |
| `from the vendor document (§n)` | Stated in the vendor document. Not tested by us. |
| `integration practice` | Our own design choice; Axaipay does not state it. |

- Never upgrade a label without new evidence. A fact that is only in the vendor
  document stays "from the vendor document", however plausible it is.
- A suspicion is written as one ("suspected cause"), with what would confirm it.
- When a newer API version arrives, re-check facts against it and update the
  version and labels. Do not mix versions silently.

## Where things go

| File | Holds |
|---|---|
| `plugins/axaipay/skills/axaipay/SKILL.md` | Hard rules and the workflow. Language-neutral. Short. |
| `references/api.md` | The paraphrased reference and the label definitions. |
| `references/pitfalls.md` | Symptom, cause, how to verify. |
| `references/dotnet-example.md` | One minimal example. Not the only valid shape. |
| `feedback/doc-gaps.md` | Suggestions for Axaipay's document, framed as "where integrators get stuck". |

## Checks before committing

- The test vectors in `api.md` and the expected values in
  `dotnet-example.md` agree, and the example code compiles and passes its tests.
- `claude plugin validate . --strict` and
  `claude plugin validate ./plugins/axaipay --strict` both pass. These check
  the manifests only, not SKILL.md frontmatter.
- The Agent Skills reference validator passes on the skill folder:
  `agentskills validate plugins/axaipay/skills/axaipay` (from the `skills-ref`
  package).
- A case-insensitive search for merchant-identifying terms and secret-looking
  strings finds nothing. Include one term that must match, so a broken search
  cannot pass as clean.
- `version` in `plugins/axaipay/.claude-plugin/plugin.json` is bumped when the
  skill content changes, so installed copies can update.
