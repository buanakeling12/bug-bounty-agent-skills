---
name: ffuf-mastery
description: Master ffuf (Fuzz Faster you Fool) for bug bounty recon and pen testing — directory brute force, recursion, multi-keyword fuzzing, calibration, auth, match/filter, Burp integration, clusterbomb vs pitchfork. Use when fuzzing endpoints, directories, vhosts, or parameters on an authorized target. Grounded in ffuf 2.1.0 on hermes-wb with SecLists at /root/wordlists/.
category: security
risk: offensive
source: CODINGO — "How to Master FFUF for Bug Bounties and Pen Testing" (PDF, supplied by the researcher 2026-11-27)
source_type: user_supplied
date_added: '2026-11-27'
license: guide is open source (codingo.github.io); ffuf itself is MIT
compatibility: Requires an authorized target scope (bounty_gate.py / SCOPE.md) before any fuzzing. ffuf 2.1.0 confirmed on hermes-wb; SecLists at /root/wordlists/.
---

> **AUTHORIZED USE ONLY**
> Fuzz only targets the researcher names and authorizes, inside the frozen scope.
> Run the bounty gate + scope-check before any fuzzing batch.
> Keep request rate modest on live targets (see Throttling) — a scan that
> knocks a target over or gets the account rate-limited is a failed run.

## Where ffuf lives here

- **Binary:** `ffuf` 2.1.0 on hermes-wb (bounty box). Verify with `ffuf -V`.
- **Wordlists:** SecLists at `/root/wordlists/` on hermes-wb.
  Start with the discovery lists:
  - `Discovery/Web-Content/directory-list-2.3-small.txt`
  - `Discovery/Web-Content/directory-list-2.3-medium.txt`
  - `Discovery/Web-Content/directory-list-2.3-big.txt`
  - `Discovery/Web-Content/burp-parameter-names.txt` (parameter discovery)
- **Sandbox:** ffuf is a Go binary; the sandbox has Go 1.25.0, so
  `go install github.com/ffuf/ffuf@latest` works if a local copy is ever needed.

## Basic usage

```bash
# First directory brute force — FUZZ is replaced by each wordlist line
ffuf -u https://target.com/FUZZ -w /root/wordlists/Discovery/Web-Content/directory-list-2.3-small.txt

# Only show hits (strip the banner + footer) — pipe-friendly
ffuf -u https://target.com/FUZZ -w wordlist.txt -s

# Save hits to a file AND see them live
ffuf -u https://target.com/FUZZ -w wordlist.txt -s | tee ./output.txt

# Structured output for later parsing
ffuf -u https://target.com/FUZZ -w wordlist.txt -o results.json -of json
```

Default matcher shows status 200, 204, 301, 302, 307, 401, 403. A 301
usually means "something lives here" — investigate it, don't just log it.

## Recursion and extensions

```bash
# Recurse into every directory found (depth 2)
ffuf -u https://target.com/FUZZ -w wordlist.txt -recursion -recursion-depth 2

# Also try file extensions on every word (finds .bak / .zip backups)
ffuf -u https://target.com/FUZZ -w wordlist.txt -e .bak,.zip,.old,.txt
```

Note: recursion only works with the literal `FUZZ` keyword, not custom
keywords (see Multi-keyword below).

## Fuzzing multiple locations (named keywords)

Replace `FUZZ` with named keywords (`W1`, `W2`) and bind each to a wordlist:

```bash
# One keyword, one wordlist
ffuf -u https://target.com/W1 -w wordlist.txt:W1

# Two keywords: fuzz a path across many domains
ffuf -u https://W2/W1 -w wordlist.txt:W1 -w domains.txt:W2
# (older ffuf needs -w repeated per wordlist; 2.1.0 accepts the comma form too)
```

**Wordlist order matters for rate-limiting.** In clusterbomb mode ffuf
exhausts the FIRST wordlist before moving to the next item of the second.
- `wordlist:W1, domains:W2` → all 1000 dirs on domain 1, then domain 2…
  (hammers one server, likely rate-limit / ban)
- `domains:W2, wordlist:W1` → dir 1 on ALL domains, then dir 2 on all…
  (spreads load, same coverage, far less likely to trip limits)

Put the **largest / slowest-changing** list second when you want spread.

## Authentication

```bash
# Cookie-based auth (paste a session cookie; test APIs in-browser pattern applies)
ffuf -u https://target.com/FUZZ -w wordlist.txt -b "session=abc123; other=xyz"

# Header-based auth / custom headers (also for "call your shot" bounty headers)
ffuf -u https://target.com/FUZZ -w wordlist.txt -H "Authorization: Bearer TOKEN"
ffuf -u https://target.com/FUZZ -w wordlist.txt -H "X-Bug-Bounty: the researcher"
```

Headers and cookies can themselves be fuzz points (`-H "X-Custom: FUZZ"`).

## Threads and throttling (live targets)

```bash
ffuf -u https://target.com/FUZZ -w wordlist.txt -t 40        # default 40 threads
ffuf -u https://target.com/FUZZ -w wordlist.txt -t 10 -p 0.5  # 10 threads, 0.5s delay
ffuf -u https://target.com/FUZZ -w wordlist.txt -p 0.1-2.0   # random delay 0.1–2s
ffuf -u https://target.com/FUZZ -w wordlist.txt -rate 2      # max 2 req/sec
```

Rule of thumb: on a live production target, prefer `-rate` / `-p` over raw
threads. 40 threads against a small target is how you get banned mid-scan.
If the program's rules cap request rate, obey the cap, not the default.

## Calibration (killing false positives)

```bash
# Auto-calibrate: ffuf sends random preflight requests and filters the
# common response (e.g. every random path returns 200 with the same length)
ffuf -u https://target.com/FUZZ -w wordlist.txt -ac

# Custom calibration seed — best for vhost scanning, where the baseline
# must be a KNOWN host, not a random string
ffuf -w vhostnames.txt -u https://target.com -H "Host: FUZZ.target.com" -acc "www"
```

`-acc` is more accurate than `-ac` for vhosts: it captures the real
baseline (content-length + status of `www`) and only shows deviations.

## Match and filter

```bash
# Match only 200s
ffuf -u https://target.com/FUZZ -w wordlist.txt -mc 200

# Match a regex in the body (e.g. path traversal proof)
ffuf -u https://target.com/FUZZ -w wordlist.txt -mr "root:"

# Filters (inverse of match) — drop the noise
ffuf -u https://target.com/FUZZ -w wordlist.txt -fc 404        # filter status 404
ffuf -u https://target.com/FUZZ -w wordlist.txt -fs 1234       # filter size 1234 bytes
ffuf -u https://target.com/FUZZ -w wordlist.txt -fw 42         # filter word count 42
ffuf -u https://target.com/FUZZ -w wordlist.txt -fl 5          # filter line count 5
```

When a sink reflects your input, filter by **words** (`-fw`) or **lines**
(`-fl`) — content-length alone is often identical across hits.

## Stop conditions (don't burn the run)

```bash
ffuf -u https://target.com/FUZZ -w wordlist.txt -sa   # stop immediately on first error
ffuf -u https://target.com/FUZZ -w wordlist.txt -se   # stop if recent requests error heavily
                                                       # (95% of last 50 = 403, or 20% = 429)
ffuf -u https://target.com/FUZZ -w wordlist.txt -sf   # stop if a share of the whole pool errors
```

`-se` is safer than `-sf`: it reacts to a *recent* error burst (you just
got rate-limited) instead of waiting for the whole pool to fail.

## Clusterbomb vs pitchfork

```bash
# clusterbomb (default): every combination of both wordlists
ffuf -u "https://target.com/login?user=W1&pass=W2" \
  -w users.txt:W1 -w passwords.txt:W2 -mode clusterbomb

# pitchfork: wordlists in series — user[0]+pass[0], user[1]+pass[1], …
# use when the pairs belong together (known user→password mapping)
ffuf -u "https://target.com/login?user=W1&pass=W2" \
  -w users.txt:W1 -w passwords.txt:W2 -mode pitchfork
```

Default to clusterbomb unless the pairs are known to belong together.

## Importing a captured request

Save a request from Burp (right-click → **Copy to file**, NOT "Save item"),
add `FUZZ` where you want to fuzz, then:

```bash
ffuf -request /tmp/request.txt -w wordlist.txt
ffuf -request /tmp/request.txt -w wordlist.txt -s | tee ./output.txt
```

This is the easiest way to fuzz an authenticated/complex request without
rebuilding headers and cookies by hand.

## Sending scans via Burp Suite

```bash
# Replay only MATCHED hits to Burp (for manual follow-up)
ffuf -u https://target.com/FUZZ -w wordlist.txt -replay-proxy http://127.0.0.1:8080

# Send ALL traffic via Burp (engagement logging / complex auth via Burp macros)
ffuf -u https://target.com/FUZZ -w wordlist.txt -x http://127.0.0.1:8080
```

Interface trick (when Burp must see everything but ffuf talks to a fixed
port): add a second Burp interface on 8181 with "Redirect to host/port"
set to the target, then point ffuf at `https://127.0.0.1:8181/FUZZ`.
Caution: routing big wordlists through Burp bloats the project file fast.

Remote VPS → local Burp: `ssh -R 8888:localhost:8080 user@remotevps`,
then `-replay-proxy http://127.0.0.1:8888` on the VPS.

## Recon recipes (authorized targets only)

```bash
# Directories (small → medium → big as the target allows)
ffuf -u https://target.com/FUZZ -w /root/wordlists/Discovery/Web-Content/directory-list-2.3-small.txt -s -ac

# Parameter discovery on an endpoint (OOB callback as the value)
ffuf -w /root/wordlists/Discovery/Web-Content/burp-parameter-names.txt \
  -u "https://target.com/api/endpoint?FUZZ=https://YOUR.callback.com" -fs 0 -mc all

# Vhost discovery (custom calibration kills the default-vhost false positives)
ffuf -w vhostnames.txt -u https://target.com -H "Host: FUZZ.target.com" -acc "www" -mc all -fs 0

# File-extension sweep on a known directory
ffuf -u https://target.com/admin/FUZZ -w wordlist.txt -e .php,.bak,.zip,.old -s
```

## What to do with hits

A hit is not a finding. For each hit: fetch it manually (curl / browser),
confirm what it actually is (admin panel? backup? debug endpoint?), check
whether it is in scope, and only then decide if it is a candidate. Feed
real candidates through the normal validation gate (Claude + GPT +
execution test + recheck) before any report.
