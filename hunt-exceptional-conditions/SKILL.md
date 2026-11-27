---
name: hunt-exceptional-conditions
description: "Hunt mishandling of exceptional conditions — feed an endpoint malformed/unexpected input (wrong type, broken JSON, oversized field, null byte) and make it fail OPEN or leak internals: a verbose stack-trace / framework error page that discloses ORM internals, server file paths, library versions, or a language traceback. Use on any input-accepting endpoint (JSON APIs, forms, query params). Medium-High when the leak exposes internal structure that arms a deeper attack."
category: security
risk: offensive
source: https://github.com/elementalsouls/Claude-BugHunter
source_repo: elementalsouls/Claude-BugHunter
source_type: community
date_added: '2026-09-20'
date_updated: '2026-11-21'
license: MIT
license_source: https://github.com/elementalsouls/Claude-BugHunter/blob/main/LICENSE
compatibility: Requires explicit written authorization for a target scope plus the
  relevant testing tools for this technique. Docs-only; helper scripts and commands
  not bundled.
report_count: 0
sources: hackerone_public
---
> **⚠️ AUTHORIZED USE ONLY**
> This skill is for educational purposes or authorized security assessments only.
> You must have explicit, written permission from the system owner before using this tool.
> Misuse of this tool is illegal and strictly prohibited.

> **Mandatory confirmation gate**
> Before running any command that probes, exploits, changes, persists on, extracts data from, or attempts credential access against a target:
> 1. Ask the user to state the exact target URL, IP, account, or resource.
> 2. Ask the user to confirm written authorization and the permitted scope.
> 3. Show the exact command(s) and explain their expected effect.
> 4. Wait for explicit confirmation in the current conversation.
>
> Without that confirmation, remain read-only and provide defensive guidance only. Prefer a sandbox, disposable VM, or controlled lab.

# HUNT-EXCEPTIONAL-CONDITIONS — Verbose Errors / Fail-Open (A10:2025)

## What actually pays

Well-built apps catch errors and return a clean, generic message. A broken app,
when handed input it didn't expect, throws an unhandled exception and renders a
**developer error page** straight to the client — leaking the stack trace, the
ORM/query internals, server-side file paths, and framework/library versions.
That disclosure is the finding (and it arms SQLi/RCE/path attacks next).

## Recon

Any endpoint that parses input is a candidate; the richest are:

```
JSON APIs that expect typed fields:  POST /api/* with {numbers, ids, enums}
Endpoints with numeric/id path or query params:  /item/{id}, ?page=, ?quantity=
Search / filter / sort params
File or content-type sensitive uploads
```

## Attack — send what the code didn't anticipate

Take a known-good request and break ONE assumption at a time:

- **Wrong type:** a field the app expects to be a number/string is sent as an
  array or object — `{"rating":"x","comment":[1,2,3]}`, `{"quantity":{}}`.
- **Malformed body:** truncated/!invalid JSON, an unterminated string, a stray
  brace, a wrong/missing Content-Type.
- **Boundary/oversized:** a very long string, a huge/negative/overflow number.
- **Null byte / control chars** embedded in a value.

```
POST /api/Feedbacks   {"rating":"notanumber","comment":[1,2,3]}
GET  /item/' OR /item/%00   (also exercises the error path)
```

Watch the RESPONSE BODY, not just the status: a 500 (or even a 200/400) whose
body contains a stack trace or framework error page is the signal.

## What counts as a leak (the success signal)

A finding is confirmed when the response body contains a cross-framework error-disclosure signature:

- **Node/Express + Sequelize:** `SequelizeDatabaseError`, `node_modules/sequelize`,
  a JS stack with internal paths.
- **PHP:** `<b>Warning</b> ... /var/www/.../file.php on line N`.
- **Python:** `Traceback (most recent call last)`, `werkzeug.exceptions`.
- **Java:** `at com.app.Foo(Foo.java:42)` stack frames.
- **.NET:** `Server Error in '/' Application`, a `[System.XxxException: ...]` YSOD.

A clean JSON error (`{"error":"Invalid input"}`) with no internals is NOT a
finding — that's correct handling. Disclosure of internal structure is.

## Validation discipline

- Capture the exact leaked artifact (path, ORM class, version, stack frame) —
  that's the evidence. "It returned 500" alone is not disclosure.
- Note what the leak enables next (e.g. a disclosed SQL error → hunt-sqli; a
  disclosed absolute path → hunt-lfi).

---

## Workspace discipline — kept from the Muse-adapted install (2026-09-20)

> The paragraphs below come from the adapted install. Where they differ from the
> upstream text above, this section governs (canary-only proof, no invented
> success rates, payload boundaries).

- **Boundary/oversized:** test only pre-agreed small bounds and request rates; do not send unbounded payloads or attempt denial of service. Negative/overflow numbers must remain within the permitted test scope.
The following signatures are candidates for error disclosure. Confirm that the response reveals non-public, security-relevant internals in the authorized context; a reflected string, intended debug lab output or public filename alone is not sufficient:

## Limitations

- Authorized scope only: the confirmation gate above is mandatory before any probing, exploitation, or credential-access command.
- Docs-only import: upstream helper scripts, commands, engine, and research assets are not bundled; reinstall tooling from the source repo when needed.
- Validate every finding (see `triage-validation`) before reporting; report via `report-writing`. Prefer a sandbox, disposable VM, or controlled lab.

### Example

```bash
# Read-only first step; confirm scope before anything active.
cat scope.txt  # target list from the authorized engagement brief
```

---
## Local adaptation — Muse (borat), 16 Nov 2026
Standing rules that override upstream on conflict:
- SSRF to internal/cloud-metadata endpoints (169.254.169.254, metadata.google.internal, localhost, internal hosts) is allowed ONLY when ALL of: (a) within the target program's authorized scope and rules; (b) least-invasive demonstration — self-owned public canary preferred; NO credential extraction, NO reading internal data, NO lateral movement, NO harm/damage; (c) nothing criminal. Anything beyond that stays BLOCKED.
- No mainnet transaction, signature, mint, payment, or report submission without the user's explicit "gas" tap after a preview.
- No private keys, seeds, or API keys in skill content; keys only via Secure Vault or an approved connector.
- No credential stuffing, breached credentials, mass guessing, OTP brute-force, or real-user data extraction.
- Owned test accounts and synthetic canaries only.
- Independent re-check before any draft is shown to the user.

