---
name: bb-recon-pipeline
description: Adapted recon pipeline + per-area test checklists for authorized bug bounty hunts. Use at the START of recon on a new target (after scope is frozen) and when testing a specific auth/session area (2FA, CSRF, password reset, rate limit, JWT, OAuth, captcha, support portal). Routes the 16-step response-dump pipeline and the per-area checklists, all gated by the researcher's bounty rules.
category: security
risk: offensive
source: https://github.com/tuhin1729/Bug-Bounty-Methodology
source_repo: tuhin1729/Bug-Bounty-Methodology
source_type: community
date_added: '2026-11-23'
license: MIT
license_source: https://github.com/tuhin1729/Bug-Bounty-Methodology/blob/main/LICENSE
compatibility: Requires a target the researcher named + authorized in that conversation, a frozen SCOPE.md, and the bounty gate passed. Docs + commands only; no bundled scripts. Adapted — NOT a verbatim copy.
sources: community, public_research
---

> **AUTHORIZED USE ONLY**
> This skill is for authorized security assessments only. Use it only for a
> target the researcher named and authorized in that conversation, inside the frozen
> scope, after the bounty gate passed. No Immunefi-hosted programs. No
> deposit-first programs. No on-chain tx, no KYC, no filing, no spend
> without his explicit approval for that exact step. Testing is manual-style,
> low-volume, targeted — never high-volume automated scanning where the
> program bans it. No DoS. No accessing anyone else's data. No social
> engineering. Own accounts only.

> **Adaptation note (2026-11-23)**
> Adapted from tuhin1729/Bug-Bounty-Methodology (MIT, 940 stars, active).
> Changes vs the original: (1) added the authorization/scope gate; (2)
> DROPPED the entire DoS checklist — DoS is excluded by every program
> the researcher hunts; (3) support-portal and OAuth pre-account-takeover restricted
> to OWN accounts only — never spoof a victim's email, never read others'
> tickets; (4) rate-limit testing is observe-only — no IP rotation or
> header spoofing to evade (programs explicitly exclude it); (5) added the
> validation gate (Claude + GPT + execution test + recheck) before filing;
> (6) recon pipeline gated by the program's automation rules. The Ars0N
> framework (R-s0n/ars0n-framework) was reviewed from the same OneHack
> research and is NOT adopted — it is on hold since Feb 2025, unmaintained;
> only its Fire-Starter tool chain is referenced below as a checklist.

# Recon Pipeline (adapted)

The core idea, taken from the source: **dump the whole attack surface
first, then grep it offline.** From a list of in-scope root domains to a
de-duplicated, response-dumped attack surface ready for manual testing.

Run only the steps the program's rules allow. If the program bans
high-volume automated tools, do the manual/exec equivalents (curl,
targeted checks) instead of the bulk pipeline. Record every step and its
result in the hunt's FINDINGS.md.

## The 16 steps

1. **Input.** All in-scope root domains in a file `domains`.
2. **Passive subdomain enum.** `subfinder -dL domains -o subdomains`
   (with all available API keys). Complement with Amass, Assetfinder,
   GAU, CRT (Certificate Transparency), ShuffleDNS, GoSpider,
   Subdomainizer (the Ars0N Fire-Starter chain). Use real DNS via Google
   DoH — sandbox `dig` returns fake 198.18.81.x proxy IPs.
3. **Webapp enum.** httpx twice over common ports, two passes:
   `httpx -l subdomains -p 80,443,3000,5000,7001,7002,8000,8008,8009,8080,8081,8082,8088,8181,8443,8880,8888,9000,9090,9443,10000 -o webapps1`
   and a second pass to `webapps2`.
4. **Dedup + merge.** `sort -u webapps1 webapps2 -o webapps`, then drop
   duplicates that differ only by protocol (awk oneliner).
5. **Active crawling.** `katana -list webapps -d 5 -s breadth-first -jc -jsl -kf all -iqp -ct 10m -timeout 10 -retry 1 -p 5 -c 20 -rl 100 -hrl 20 -silent -o katana_output` (grabs JS files, endpoints).
6. **Passive crawling.** `urlfinder -list domains -all -silent -o urlfinder_output`.
7. **Merge endpoints.** `sort -u ffuf_output katana_output urlfinder_output -o all_endpoints`, then strip static assets (jpg, jpeg, png, gif, svg, webp, woff/woff2, ttf, css, mp3/mp4, etc.).
8. **Response dump.** `cat all_endpoints | fff -S -k -o all_endpoints_responses` — dump every header + response locally. This is the step that makes the rest fast: grep offline instead of re-hitting the target.
9. **Secret detection.** (a) kingfisher, trufflehog, gitleaks on the dump; (b) `rg` with custom regex for Basic/Bearer tokens and API keys; (c) identify Google API keys and run agneyastra on them.
10. **Low-hanging fruit.** `nuclei -l webapps -severity critical,high,medium,low -o nuclei_output`. Only where the program allows nuclei/templates; never scanner-output-only reports — every hit needs manual validation + PoC.
11. **Interesting endpoints.** Grep the dump for `/admin`, `/api/`, `/graphql`, `/debug`, `/swagger`, `/internal`, `/openapi`, `/actuator`, and similar.
12. **Fuzzing / content discovery (DIRB method).** Per URL in webapps:
    `ffuf -u "${url%/}/FUZZ" -w files.txt -ac -mc all -fc 404 -recursion
    -recursion-depth 2 -t 40 -rate 100 -c -of json` for hidden
    endpoints/dirs/files. Low-volume; respect the program's rate rules.
    **Baseline first (the DIRB lesson):** before fuzzing, fetch a random
    nonexistent path and record its status + size — that is the site's
    NOT_FOUND signature. Filter it out (`-fc <code>` and/or `-fs
    <size>`), otherwise every word "matches". Unpublished SITE123 sites
    return 200 + the same 4,209-byte placeholder for every path; the
    marketing site returns a 404 + a fixed 185,533-byte page.
    **High-value path classes (from the DIRB video, 2026-11-24):**
    unprotected admin panels (`/admin`, `/cpanel`, `/controlpanel`),
    internal folders (`/config`, `/api`, `/cgi-sys`), revision history
    (`/.git`, `/.svn`, `/.hg`), and backup files (`.bak`, `.sql`, `.zip`,
    `.old`, `.swp`, `.env`, `error_log`) — one exposed DB backup can leak
    the whole customer base. **Installed (2026-11-24, hermes-wb):**
    ffuf 2.1.0 at /usr/local/bin/ffuf; SecLists
    Discovery/Web-Content/common.txt (4,751 words — the direct analogue
    of dirb's common.txt, 4,612 words) and Common-DB-Backups.txt (336
    words) at /root/wordlists/ (mirrored in
    ~/workspace/research_notes/tiktok-dirb/wordlists/). **ffuf 2.1.0
    gotcha: `-silent` does NOT exist — use `-s`.** Rate: `-rate 10
    -t 10` for production hosts unless the program allows more. A hit is
    a LEAD, not a finding: fetch it, read it, and only own-account data
    counts — never download or open another tenant's backup/config.
13. **Oneliners (targeted, low-volume).** XSS: `cat all_endpoints | grep "=" | qsreplace ""><abcd>" | httpx -ms "<abcd>" -o vuln`. SSRF / open redirect: same with `qsreplace "WEBHOOK" | httpx -fr`. CORS: grep `Access-Control-Allow-Credentials` in the fff `*.headers` files. Template injection: `qsreplace "{{4327*8191}}" | httpx -ms "35442457"`. Every hit must be confirmed by a browser execution test (no alert = falsified) — a reflected string is not a finding.
14. **Parameter bruteforce.** x8 (github.com/Sh1Yo/x8) on the endpoints found; add results to all_endpoints.
15. **GitHub recon.** gitdorker (github.com/obheda12/GitDorker) for leaks in public GitHub (read-only; never use leaked credentials).
16. **Report.** Validate secrets post-discovery, identify vulns, escalate, manual/LLM validation, manual verification. Then the validation gate below.

## WAF note (learned on KuCoin + Bullish, 2026-11-23)

Cloudflare WAF blocks standard XSS payloads before the app (returns a
challenge page). Double-URL-encoding the payload (irwanjugabro's FACEIT
technique) does NOT bypass Cloudflare here — it normalizes/decodes
before rule matching. Do not burn time on encoding bypasses when the
server HTML shows zero reflection of a benign canary param: no
reflection = falsified, move on.

## 403 Bypass (adapted from YogaRmdn/403bypass, via @yogaramadani_s.kom)

When a target returns 403 on an in-scope path, test whether the access
control can be bypassed — before concluding the path is closed. Source:
https://github.com/YogaRmdn/403bypass (9 stars, single Python script,
read 2026-11-23 — no exfiltration, no obfuscation, plain requests to
the target only). Adapted: the original fires ~320 requests in
`--exploit` mode with no delay. NEVER run the full scan against a
bounty target — it violates low-volume / no-automated-scanner rules.
Use ONE mode at a time, a small curated subset, with a delay between
requests. Drop the SQLi mode (payloads are nonsense).

Header bypass (try one at a time, observe the status code):
- `X-Original-URL: /<path>` and `X-Rewrite-URL: /<path>` (most reliable
  — the backend may trust these to route past the WAF/proxy).
- `X-Forwarded-For: 127.0.0.1`, `X-Client-IP: 127.0.0.1`,
  `X-Real-IP: 127.0.0.1`, `X-Remote-IP: 127.0.0.1`,
  `X-Originating-IP: 127.0.0.1`, `CF-Connecting-IP: 127.0.0.1`,
  `True-Client-IP: 127.0.0.1`, `X-Custom-IP-Authorization: 127.0.0.1`
  (tests whether the server trusts a spoofed internal IP for access).
- `X-Forwarded-Host: 127.0.0.1`, `X-Host: 127.0.0.1`,
  `X-Forwarded-Proto: http://<domain>`, `Referer: <target>`.
- Special: `X-Original-URL` with a request to `<target>/anything`;
  `X-Custom-IP-Authorization: 127.0.0.1` with `<target>..;/`.

Method bypass: try GET, POST, HEAD, OPTIONS, PUT, PATCH, TRACE on the
same path — a 403 on GET sometimes allows another method.

Protocol / port: try http:// vs https://; `X-Forwarded-Scheme: http`
or `https`; `X-Forwarded-Port: 80/443/8080/8443/4443`.

URL-encoding bypass (use a SHORT list, not all 245): append
`/%2e%2e/`, `/..;/`, `/%2e/`, `/;`, `/.json`, `/%20/`, `//`, `/.` and
similar to the path — tests whether the WAF/proxy normalizes the path
differently from the backend.

A bypass counts ONLY if the response body actually contains the
protected content (not just a 200 with an error page). Confirm by
comparing the body length/content with the 403 response. A 200 that
returns the same "Forbidden" text is not a bypass.

## Additional recon tools (from yogsec/Hacking-Tools)

The 16-step pipeline above is the core. These extra tools fill gaps in it — passive recon, tech fingerprinting, and URL discovery. All of them are read-only or low-volume; none of them changes the pipeline's rules. Source: yogsec/Hacking-Tools (MIT, 2,581 stars), researched 2026-11-23 — a curated tool list, not a methodology; only the tools below are adopted, each under the researcher's existing gates.

Passive recon (no extra requests to the target beyond what the pipeline already does):
- `theHarvester -d <domain> -b all` — emails, subdomains, hosts from public sources. Never use a gathered email to contact or spoof anyone.
- `recon-ng` — modular recon framework; passive modules only.
- `spiderfoot -s <domain>` — OSINT automation across 100+ sources; targeted modules only, never the full automated scan on a bounty target.
- Shodan / Censys / Netlas (web search, no install) — look up the target org's domains/IPs for exposed services. A hit is recon context, not a finding: verify the service is live, in scope, and testable before touching it.
- `waybackurls <domain>` — historical URLs from the Wayback Machine; add the output to all_endpoints, then response-dump as usual.
- `unfurl` — parse already-collected URLs (params, paths) offline.
- `dnsrecon -d <domain>` / `fierce --domain <domain>` — DNS enum alongside subfinder/Amass. Real DNS via Google DoH.
- `sherlock <username>` / `maigret <username>` — username OSINT on the target org's public usernames only; never impersonate or contact.
- FOCA — metadata extraction from the target's public documents (author, software versions); read-only, recon context.
- VirusTotal (web lookup) — quick reputation check on a domain/file.

## Service discovery (nmap, adapted from the DIRB video, 2026-11-24)

The video's nmap footage (open ports, a /backup/ directory listing, a
login page on the target) is adapted as a service-discovery lane — NOT
as a full port sweep. **Installed (2026-11-24, hermes-wb): nmap 7.94SVN
(apt).** Rules: in-scope hosts/IPs only, from the frozen SCOPE.md;
`-sV` service/version detection on a SHORT explicit port list (e.g.
`-p 80,443` or `--top-ports 10` with `--host-timeout 60s`); `-Pn` when
ICMP is blocked; NO `-A` aggressive scan by default, NO `--script`
vuln/exploit scripts, NO UDP sweep, NO full-range (1-65535) scans —
those are high-volume and read as hostile by most programs. A
CDN-fronted host (Cloudflare/BunnyCDN) will only show the CDN's
80/443 — that is expected, not a finding. An open non-web port on an
in-scope host is a LEAD: identify the service, check whether it is in
scope, and test it only through the validation gate. **VM egress note
(2026-11-24): a top-10 scan of a CDN host timed out on the VM's slow
egress — keep scans bounded (explicit ports + --host-timeout) and do
not re-run a timed-out scan unchanged.** The /backup/-style finding in
the video maps to the content-discovery lane (step 12), not to nmap.

Tech fingerprinting (informs which tests are plausible):
- `whatweb <url>` / Wappalyzer (browser extension) — identify the stack (framework, CMS, JS libraries, server). Record it in FINDINGS.md.
- `retire.js` on the collected JS — identify vulnerable JS library versions. Recon context ONLY: a vulnerable library version without a working PoC is a dependency-only report (spam per Aiven) — never file on version alone.

JS endpoint extraction (offline, never executes JS):
- `broadside crawl` — walks pages and extracts API routes, GraphQL operations, fetch/XHR calls, and source-map references from static `<script src>` bodies with a bounded lexer. Use it as an alternative JS endpoint extractor alongside katana step 5; add what it finds to all_endpoints.

Reference resources (not tools — read them, do not run them):
- PayloadsAllTheThings (github.com/swisskyrepo/PayloadsAllTheThings) — the canonical payload cheat-sheet (XSS, SSRF, SSTI, etc.). Pick ONE payload at a time and test it manually.
- ExploitDB / SearchSploit — before reporting a version/software finding, check whether a public exploit already covers it. If it does, it is likely an N-day / duplicate — check before filing.

## Web vulnerability tools (targeted, low-volume)

The pipeline's oneliners (step 13) are the default. These tools are for when a specific class needs deeper testing. One target, one parameter, one payload at a time. Every hit needs a manual PoC + browser execution test; a tool's "vulnerable" verdict alone is never a finding.

- `xsstrike -u "<url with param>"` — XSS detection. Confirm in the browser (no alert = falsified).
- `commix --url="<url>"` — command-injection detection. Confirm with a benign canary (e.g. echo of a marker string), never a destructive command.
- `sqlmap` — SQL injection detection. **Installed (2026-11-24,
  hermes-wb): sqlmap 1.10.11 (pip, /root/.local/bin/sqlmap). Method
  adapted from the DIRB video (2026-11-24):** the video shows sqlmap
  against a GET parameter (`custom-v04.php`, parameter `id`): it
  fingerprints the DBMS first (MySQL), then works the ladder —
  error-based, then boolean-based blind — and reports which techniques
  the parameter is vulnerable to. Reproduce that ladder, not the
  video's target: `sqlmap -u "<url with param>" --batch --level 1
  --risk 1` (add `--dbms=mysql` only after the fingerprint confirms
  it). Read the output the same way: DBMS fingerprint → technique
  (error-based / boolean-blind / time-based) → injectable parameter.
  STRICT: own test account + own test data only, low-volume,
  read-only flags. NEVER `--os-shell`, `--dump` of anyone else's data,
  `--file-read`/`--file-write`, or any destructive option. A detected
  injection still needs a manual PoC + the validation gate — sqlmap
  saying "injectable" is not a finding. Live testing deferred: no
  sqlmap run has been executed yet on any hunt target from this lane.
- `gobuster dir -u <url> -w <wordlist>` / `dirsearch -u <url>` — directory/DNS brute-force, alternatives to ffuf step 12. Same rate rules as ffuf.
- `nikto -h <url>` — web server misconfig / outdated software scan. Low-volume; every hit needs manual validation.
- Burp Suite / OWASP ZAP — the manual HTTP proxy. Use it to inspect YOUR OWN requests (headers, tokens, params) and to replay them by hand. The proxy is for manual interception, not for automated scanning.

## Mobile apps (only when the program's scope includes them)

Most programs the researcher hunts are web/API only — skip this section unless the brief explicitly lists iOS/Android apps (e.g. KuCoin does; Aiven does not). All testing on your OWN test device and OWN test account.

- MobSF — static + dynamic analysis of the app (Android/iOS). Start here; it lists exported components, hardcoded secrets, and insecure configs.
- `jadx` / `apktool` — decompile the APK and read the code: API endpoints, hosts, and secrets. A found secret needs a live PoC to count; never use a key that belongs to someone else.
- `apkleaks -f app.apk` — scan the APK for URIs, endpoints, and hardcoded secrets.
- Frida / Objection — runtime instrumentation: SSL pinning bypass (so you can inspect YOUR OWN app traffic through the proxy), method hooking, filesystem/keychain access.
- Drozer — Android IPC testing: exported activities, content providers, broadcast receivers.

## REA reverse-engineering workflows (adapted from morluto/rea, 2026-11-23)

REA (Reverse Engineer Anything) v6.10.1, MIT — MCP server + CLI that inspects
binaries, JS/Electron apps, APKs, .NET, firmware and recorded network captures,
and returns findings WITH evidence + unknowns. Full note:
`~/workspace/research_notes/rea-adaptation-20261123.md`. Reference copy (read-only):
`~/workspace/vendor/rea-main/`. Red-flag grep clean (no eval, no shell:true,
no telemetry, no credential reads). **Only use the official source** —
morluto/rea, rea.tools, npm `rea-agents`; copycat repos spread trojans.

**Do NOT run `npx rea-agents setup`** — it modifies agent configs and can
install Hopper. Use the pinned CLI directly (Node 22/24+; sandbox has 24.22.1):

```bash
npx -y rea-agents@6.10.1 analyze-javascript-application /path/to/app --json > app-evidence.json
```

**Workflows (bounty use):**
- **JS bundle → endpoint inventory (highest value).** Download the target's JS
  bundle(s) into a dir, run the command above. It returns a module graph +
  endpoint nodes (routes + network endpoints). Verified on the Aiven console
  bundle (586 KB minified): 1,044 modules, 224 endpoints, 8,525 edges in
  ~31 s — recovered routes like `/marketplace/azure/verify-subscription`,
  `ai-gateway/:projectName/keys/:keyId`. Use the inventory to build the API
  surface for IDOR/auth testing. Evidence files can be huge (72 MB for one
  bundle) — query with jq/python, delete after extracting what matters.
- **HAR → derived API surface.** Feed a HAR/SAZ capture (your own session
  only) and REA derives the API surface + compares static JS endpoints vs
  HAR. Use it to map endpoints your browser session actually hit vs ones
  the bundle declares but you never called (untested surface).
- **APK (only when the program's scope includes apps).** Needs headless JADX
  + full JDK (not installed — install only on the researcher's go-ahead). Returns
  manifest, classes, decompiled methods, references.
- **EVM bytecode (offline).** Dispatch selectors, offsets, inferred args,
  mutability — for contract audits where only bytecode is available.
- **.NET assemblies / model artifacts (GGUF/safetensors).** Static only,
  no engine needed.

**Evidence model (adopt this discipline for ALL recon, not just REA):**
distinguish observation (seen in the artifact), derivation (computed from
observations), inference (best guess, labeled as such), and unknown (stays
unknown — never guessed). Keep provenance (file + hash + location) with every
claim. REA evidence deliberately does NOT retain auth header values.

**Rules:** only on in-scope, authorized targets — your own account's bundles,
your own APKs, public repos. Static JS/.NET analysis reads files without
running them (safe). Runtime capture runs/interacts with the target — only on
your own/authorized targets. A REA-derived endpoint is a LEAD, not a finding:
it still goes through the validation gate (Claude + GPT + execution test).
Native deep analysis (Hopper/Ghidra/IDA/Kuna) is NOT installed — only if a
hunt needs it, on the researcher's go-ahead.

## Tools to never use (from the same list — explicitly excluded)

Do not install or run any of these for bounty work, even if a video or a list recommends them:
- **DoS / stress testing** (Slowloris, LOIC, HULK, GoldenEye, Xerxes, Synful, T50, etc.) — DoS is excluded by every program.
- **Social engineering / phishing** (SET, King Phisher, Evilginx2, Gophish, HiddenEye, BlackEye, Zphisher, CredSniper, etc.) — phishing and credential theft are prohibited by every program.
- **Malware development / C2** (Sliver, Havoc, Donut, BYOB, ScareCrow, Veil-Evasion, PEzor, etc.) — no bounty purpose.
- **Post-exploitation frameworks** (Metasploit, Empire, Covenant, Merlin, PowerSploit, CrackMapExec, etc.) — bounty work is find-the-vuln + PoC, not post-exploitation.
- **Login brute-forcers vs live logins** (Hydra, Medusa, Patator) — violates the rate-limit / low-volume rules. Hashcat/John are offline-only, for a hash you legitimately obtained.
- **Automated exploitation frameworks** (Nishang, AutoPentestFramework, SentinelX, etc.) — automated exploitation violates the manual-style rule.

# Per-Area Checklists (adapted)

Test one area at a time on your OWN test account. Every candidate goes
through the validation gate before filing. Do NOT test an area the
program's brief excludes.

## 2FA / verification codes

- Response / status-code manipulation on the verify step.
- Brute-force OTP (only within the program's rate rules; stop on lockout).
- **Code reuse: does a code stay valid after one successful use?** Try the
  same code in a fresh verification. NOTE (2026-11-23): this exact test was
  BLOCKED by the system on KuCoin (safety refusal on the email-change
  flow) — do not retry it there; record the block, do not work around it.
- Request 2 codes (accounts A and V); try A's code in V's account.
- Go directly to the dashboard URL without solving 2FA; try adding a
  referer header to the 2FA page URL.
- Search the 2FA code in responses / JS files.
- CSRF / clickjacking to disable 2FA.
- Enabling 2FA must expire previous sessions — check it does.
- Login via OAuth to bypass 2FA (usually N/A).
- No 2FA required to DISABLE 2FA.
- Password resettable via forgot-password without 2FA.
- Enter 000000 as the code.
- Request manipulation: null JSON response, flip `otprequired` true→false,
  remove the code / the parameter, email as an array.
- Code still valid after 4–5 hours.
- Backup codes: after login, replay the backup-code generation request
  with a victim email (own accounts only).
- Does the 2FA page leak sensitive info (e.g. phone number)?
- Perform authenticated requests (profile change, API create/delete)
  WITHOUT solving 2FA.
- OTP bypass in JSON: code as an array of many values.

## CSRF

- Remove the token parameter entirely, or just its value.
- Random token of the same length; of length-1 / length+1.
- Attacker's token in the victim session (own accounts only).
- Change POST → GET and drop the token.
- PUT/DELETE via POST with `_method=PUT`.
- If the token travels in a custom header, remove the header.
- Change Content-Type (application/json ↔ x-www-form-urlencoded ↔
  multipart ↔ text/xml).
- Double-submit token (cookie + header): try CRLF injection.
- Referer check bypass: `<meta name="referrer" content="never">`, or
  referer regex tricks (`https://attacker.com?target.com`,
  `https://target.com.attacker.com`, `https://target.com@attacker.com`, …).
- Steal the token via XSS / HTMLi / CORS (only if you already have one).
- JSON: change Content-Type to text/plain / urlencoded / multipart.
- Guessable token. Clickjacking to bypass a strong token. Type juggling.
  Set the token to `null` or add null bytes.
- Token sent over HTTP or to a third party.
- Generate many tokens; find the static part, play with the dynamic part.
- **First check whether CSRF applies at all:** if the app uses Bearer
  tokens (Auth0 / Authorization header), the browser does not send them
  automatically and CSRF is not applicable — confirm the auth mechanism
  before burning time on bypasses. (Bullish H-CSRF1 stayed inconclusive
  for exactly this reason.)

## Password reset

- Password reset poisoning (Host header).
- HTTP parameter pollution: `email=victim&email=attacker`, array forms,
  JSON duplicate keys — check if the reset link lands in the attacker's
  inbox. Own accounts only.
- IDOR in the reset link; use paramminer to find extra params.
- Broken crypto: guess how reset tokens are generated.
- Token leakage via Referer (open the reset link, click an external link).
- Token leakage in responses / JS files.
- Session/token not expiring after password reset.
- Weak password policy (space-only password).
- Request 2 reset links; try the OLDER one.
- Host-manipulation variants on the reset URL.
- SQLi / CRLF in the email field.
- Register a username identical to the victim's with surrounding spaces;
  reset your account, use the token on the victim's (own accounts only).
- App-level DoS via very long password — DO NOT TEST (DoS excluded).
- OTP for password reset → use the 2FA checklist.
- Homograph in email at registration, then password reset.
- Change method / content-type and observe.
- Null bytes after the email.
- XSS / SSTI in the email field; command injection via the email field.
- 2FA auto-disabled after password reset.
- User enumeration. Param value reflected in the email → HTMLi.
- XXE if the forgot-password request accepts XML.
- Missing rate limit on email triggering — observe only, do not flood.

## Rate limit (observe only)

- Observe: is there a cooldown / lockout / error after N attempts?
  (KuCoin: 60s cooldown IS the rate limit — 3 sends, no error, no
  bypass. That is normal behavior, not a flaw.)
- Do NOT rotate IPs, do NOT spoof X-Forwarded-For / X-Originating-IP /
  X-Client-IP to evade — programs explicitly exclude rate-limit bypass
  via IP/device rotation.
- Note the other vectors for the record (do not execute): null bytes
  after the endpoint/param, race condition, random extra params, change
  body format (form↔JSON↔XML), change method, change API version,
  Gmail +/. trick.

## JWT

- Signature not checked → modify the body.
- Signature checked only if present → modify the body, delete signature.
- Weak signature → brute-force.
- None algorithm (None/none/nONE/NONE).
- RS256 → find the public key via recon, switch to HS256, sign with it.
- Vulnerable `kid`: point it at your URL so the app verifies with your
  key; try public files (`../../public/main.css`); command injection and
  SQLi via `kid`.
- Tool: jwt_tool (`python3 jwt_tool.py -M at -t <url> -rh "Authorization: Bearer <token>"`).

## OAuth

- Open redirect to steal the token: change `redirect_uri`, IDN homograph.
- Change the Referer header while requesting the OAuth token.
- Pre-account-takeover: create a normal account with an email, then an
  OAuth account with the same email — own accounts only.
- OAuth token re-use. Missing/broken `state` parameter (see CSRF).
- Lack of origin check. Open redirect on another endpoint, reached via
  `redirect_uri`.
- Extra params: if the callback carries `email=`, try changing it.
- Remove email from scope, add another manually.
- `hd=company.com` → `hd=gmail.com` when only company emails are allowed.
- `client_secret` in Burp search / GitHub dorking (read-only).
- Token in browser history.

## Captcha

- Response manipulation; reuse a previous captcha value; any token of
  the same length (±1); remove the param / its value.
- Change POST → GET (or PUT) and drop the captcha; change body JSON↔form.
- OCR. Captcha value present in source (e.g. `2+2`).
- Same captcha value across different session IDs.
- X-Forwarded-* headers (observe only — do not use to evade).

## Support portal (own account only)

- **Legitimate angle:** test YOUR OWN support ticket content for stored
  XSS (benign canary, e.g. `<img src=x onerror=alert(document.domain)>`)
  — submit one ticket, verify in the browser whether it executes. If it
  renders as plain text, it is falsified. (KuCoin ticket 10776367:
  falsified this way.)
- **Do NOT do:** spoofing a victim's email to request an email change;
  creating accounts with someone else's email; reading others' tickets;
  the "ticket trick" to access third-party workspaces; using a support
  email to reset third-party accounts. All of that targets other people
  and is banned by every program.

## Attack surface prioritization — Orbis scoring (adapted)

The pipeline above produces a long list of live endpoints. Score each
one so the highest-value targets get tested first. Adapted from the
risk-scoring model in frogy2.0 ("Orbis", iamthefrogy/frogy2.0 — README
declares the project ABANDONED; the tool was never installed, only the
scoring model was adapted). Three capped buckets, max 100 per endpoint;
the report-level score is the mean of the top-5 endpoint scores.

| Bucket | Cap | Measures |
|--------|-----|----------|
| **Sensitivity** | 40 | Asset criticality, stack complexity, data handling |
| **Exposure** | 35 | Directly dangerous or reachable attack surfaces |
| **Hygiene** | 25 | Misconfigurations, certificate health, compliance gaps |

Highest-value signals (use these first; the full Orbis table has 70+):

- **Sensitivity:** employee/internal asset +12; admin/monitoring tool
  in stack (Kibana, Grafana, Jenkins, phpMyAdmin, k8s Dashboard) +12;
  crawl surface size (log-scaled unique pages) up to +12; identity/auth
  service (Keycloak, Okta, Auth0, LDAP, SAML) +8; object storage
  exposed +8; error/debug page public +8; non-prod environment in
  title (dev/staging/test/UAT) +6; API surface +6; CMS admin +6.
- **Exposure:** open public cloud storage bucket +20; login over HTTP
  +20; confirmed subdomain takeover +15–20; potential takeover
  (dangling CNAME) +8–12; management port exposed up to +15; DB port
  exposed up to +12; directory listing +10; admin tool in page title
  +10; cloud workload/CDN without WAF +8–12; authenticated surface
  (HTTPS login) +12.
- **Hygiene:** expired certificate +20; SSL 3.0 +18; NULL/anon cipher
  +18; RC4/DES/3DES +12; legacy TLS 1.0/1.1 +12; CORS wildcard `*` +12;
  CORS null-origin +10; end-of-life server +10; missing security
  headers up to +12; cert expiring within 7 days +12; self-signed +8;
  DMARC missing +6; SPF missing +4; server version disclosed +5.

How to use it: score every live endpoint from the response dump
(step 8) + nuclei/httpx output, sort descending, and test in that
order. An internal admin panel with an expired self-signed cert, a
wildcard CORS header, and an exposed DB port outranks a static
marketing page — start where the score is highest. Re-score after each
round; a finding on a high-score endpoint outranks the same class on a
low-score one when deciding what to validate first.

**The score orders work — it is NOT a severity rating and never a
finding on its own.** A high score means "test this first", not
"this is vulnerable". Every candidate still goes through the
validation gate (Claude + GPT + execution test) before filing.

# Validation Gate (the researcher's standing rule)

Before filing ANY candidate:

1. Validate with Claude AND GPT (use `agy` for Claude until 30 Nov 2026,
   else the vault Anthropic helper; Agentrouter gpt-5.6-sol in English
   only for general review).
2. **Validation alone is never final — execution test + recheck required.**
   A reflected string is not a finding; a WAF block is not a finding.
3. Humanize the report (`humanizer` skill) so it does not read as AI slop.
4. Combine multiple valid bugs into one report unless the program's rules
   require one report per bug.
5. Bank the report if the Bugcrowd 1-pending-review slot is occupied;
   submit only when the slot is free and the researcher approved that exact filing.
