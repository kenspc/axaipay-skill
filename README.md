# Axaipay integration skill

A skill that helps AI coding agents integrate the
[Axaipay](https://axaipay.com) eComPay payment gateway correctly the first time:
hosted payment page, signatures, callbacks, and the inquiry API.

> **Unofficial.** This is a personal summary of what I learned integrating
> Axaipay eComPay into one merchant website. It is not an official Axaipay
> product, and Axaipay has not reviewed it. When it disagrees with Axaipay's
> documentation or support, they are right.
>
> **Official API documentation:**
> [AXAIPAY eComPay API Technical Document](https://support.axaipay.com/support/solutions/articles/73000523456-axaipay-ecompay-api-technical-document-v1-18)
> on Axaipay's knowledge base. At the time of writing that page carries
> **v1.18 r2**. This skill was checked against **v1.19 r1**, which is not on the
> knowledge base yet; ask Axaipay support (support@axaipay.com) for the current
> version.

## What is inside

| Path | Content |
|---|---|
| [`plugins/axaipay/skills/axaipay/SKILL.md`](plugins/axaipay/skills/axaipay/SKILL.md) | Hard rules and a step-by-step integration workflow, each step with a check. |
| [`references/api.md`](plugins/axaipay/skills/axaipay/references/api.md) | Hosts, field lists, signature orders, test vectors, inquiry, code tables, test cards. |
| [`references/pitfalls.md`](plugins/axaipay/skills/axaipay/references/pitfalls.md) | Symptom, cause, and how to verify, for each problem we hit. |
| [`references/dotnet-example.md`](plugins/axaipay/skills/axaipay/references/dotnet-example.md) | A minimal ASP.NET Core example. The skill itself is language-neutral. |
| [`feedback/doc-gaps.md`](feedback/doc-gaps.md) | Suggestions for the official documentation. |

Every fact states the API version it applies to and how it was established:
verified by a live call (with the date), observed during integration, confirmed
with Axaipay, taken from the vendor document, or our own practice. The labels
are explained at the top of
[`api.md`](plugins/axaipay/skills/axaipay/references/api.md#how-each-fact-was-established).

## Install

### Claude Code (plugin)

```text
/plugin marketplace add kenspc/axaipay-skill
/plugin install axaipay@axaipay-skill
```

Or from a shell:

```bash
claude plugin marketplace add kenspc/axaipay-skill
claude plugin install axaipay@axaipay-skill
```

Third-party marketplaces do not auto-update by default. To update later, run
`/plugin marketplace update axaipay-skill`.

### Claude Code (copy the skill)

Copy `plugins/axaipay/skills/axaipay/` to `~/.claude/skills/axaipay/` (all your
projects) or to `.claude/skills/axaipay/` inside one project.

### Other agents

`SKILL.md` follows the [Agent Skills](https://agentskills.io) format and uses
only its standard frontmatter fields. Point your agent at the
`plugins/axaipay/skills/axaipay/` folder the way it loads skills, or reference
`SKILL.md` and the `references/` files from your agent's instruction file.

## Using it

Ask your agent things like "integrate Axaipay into this checkout", "why does
Axaipay inquiry return PY016", or "review our Axaipay callback handler". The
skill tells the agent to write and test the signing code against fixed test
vectors before anything else, and to probe the inquiry API safely with a
nonexistent transaction id.

## Contributing

Corrections are welcome, especially facts verified against a newer API version.
Read [AGENTS.md](AGENTS.md) first: it explains how facts are sourced and dated,
and what must never be added.
