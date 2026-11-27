# bb-workflow-0xkonde — Bug Bounty Workflow (0xkonde, adapted)

> Adaptasi dari `bugbounty-workflow-0xkonde_1_k41i.md` (0xkonde // ZERO-DAY HUNTER).
> Owner: the researcher. Agent: the assistant. SOUL tetap the assistant, tidak ditimpa.
> Doktrin: PROVEN Critical/High only. Medium/Low = fuel, bukan hasil.
> Yang TIDAK diadopsi: "full autonomy tanpa tanya" (otonomi the assistant dibatasi gate the researcher:
> filing, deposit, KYC, spend, dan on-chain tx butuh go-ahead eksplisit per langkah;
> lane yang safety-flagged tetap TUTUP). Immunefi di-drop (the researcher skip program Immunefi).
> Tool di workflow asli (subfinder, katana, gau, dnsx, httpx) tidak diasumsikan ada —
> pakai yang tersedia (nuclei/ffuf/nmap/sqlmap di hermes-wb, curl di sandbox, reconkit).

## 0. Aturan main
1. Otonomi penuh DALAM gate the researcher. Gas tanpa tanya untuk recon + testing dalam scope.
   Stop dan lapor: (a) Critical/High PROVEN via validation gate, atau (b) semua vector habis.
2. Urutan serang: UNAUTH dulu, akun test PALING AKHIR.
3. Tiap klaim wajib PoC real dari terminal. Kata "possibly" = HARAM.
4. Chain anomaly kecil jadi 1 serangan lethal. Critical lahir dari chain, bukan single bug.
5. Hemat resource: OTP 1x/akun, captcha max 2x STOP, proxy residential cuma kalau diblokir/geo.
   Money-path butuh deposit real -> STOP, lapor dulu (aturan the researcher, bukan saran).
6. **5-MINUTE RULE:** semua 401/403/404 selama 5 menit -> MOVE ON ke surface lain.
7. **1-HOUR RULE:** 1 jam no progress -> SWITCH CONTEXT.
8. Min 3 iterasi per vector sebelum drop. Gagal -> tulis 1 baris LESSON, ganti payload, RUN AGAIN.

## 1. Urutan fase
```
0 scope -> cek scope deny-wins SEBELUM sentuh HTTP (scope-check + bounty gate)
1 recon -> subdomain, live host, crawl, JS, API spec
2 surface -> ranking attack surface + threat model + crown jewel
3 hunt -> 1 bug class at a time, deep bukan spray
4 ideation -> baca writeup, pola CVE lama -> pola baru
5 chain -> A->B->C escalation
6 validate -> 7-Question Gate + 4 gates (di bawah)
7 report -> email/H1/Bugcrowd (bukan Immunefi), HANYA atas go-ahead the researcher per finding
8 remember -> catat lesson
```

## 2. Recon (wajib sebelum testing)
- Subdomain: crt.sh (curl) + subfinder bila ada. Live host: dnsx/httpx bila ada, else curl + getent.
- URL: katana/waybackurls/gau bila ada, else crawl JS + sitemap + robots.
- Nuclei critical/high/medium di live host (rate-limit rendah, sesuai policy program).
- API spec duluan (sering no-auth): `/swagger/v1/swagger.json`, `/swagger.json`,
  `/api/swagger.json`, `/openapi.json`, `/api/openapi.json`, `/v1/api-docs`,
  `/api-docs`, `/swagger-ui.html`, `/redoc`, `/graphql` (introspection).
- JS secrets: kumpulin .js, grep manual (SecretFinder/jsluice bila ada).
- Quick wins: takeover (github.io, heroku, azure, s3 NoSuchBucket), `/.git`, `/.env`,
  `/actuator/env`, `/actuator/heapdump`, open redirect (`?redirect= ?next= ?url=`),
  CORS `Origin: https://evil.com` + creds, S3/GCS listable, GraphQL introspection,
  Firebase `https://APP.firebaseio.com/.json`.
- Fingerprint: `XSRF-TOKEN` + `*_session` = Laravel -> coba `/horizon /telescope /storage/logs/laravel.log`;
  `PHPSESSID` = PHP; `X-Powered-By: Express` = Node; `wp-json` = WordPress ->
  `/wp-json/wp/v2/users /xmlrpc.php /?author=1`.

## 3. Surface + threat model
```
TARGET:
CROWN JEWELS: 1.___ 2.___ 3.___
SURFACE:
 [ ] unauth: login, register, reset, public API
 [ ] auth: upload, export, API calls
 [ ] cross-tenant: org/team/workspace ID
 [ ] admin: /admin /internal /debug
PRIORITAS: crown jewel x entry termudah
```
- Pake app 15 menit kayak user asli sebelum ngetes.
- Baca 3 disclosed report program itu. Siapin 2 akun (attacker + victim).

## 4. Hunt — satu class, deep (checklist per class)

### IDOR (10 varian — cek semua)
1. Direct `/api/users/123` -> 456
2. Body `{"user_id":456}`
3. GraphQL `node(id: base64(OtherType:123))`
4. Batch `?ids=1,2,3,4,5`
5. Nested `/orgs/{org}/users/{user}`
6. File `?path=../other-user/file.pdf`
7. Predictable (seq int, timestamp)
8. Method swap (GET 403? coba PUT/PATCH/DELETE)
9. Version rollback (/api/v2 blocked? coba /api/v1/)
10. Header `X-User-ID: victim` / `X-Org-ID: victim_org`
- Wajib 2 akun. Test GET+PUT+DELETE+PATCH semua. Chain: read PII=Med, write=High, admin=Critical.

### Auth / JWT / OAuth
- Tanpa token, token expired, token user lain.
- JWT: `alg:none`, weak secret, kid injection.
- OAuth: missing `state`, `redirect_uri` wildcard, missing PKCE, implicit flow.
- Open redirect -> OAuth code theft (bypass: `//evil.com`, `https://target.com@evil.com`,
  `https://target.com\\@evil.com`, `%252F%252F`, `///evil.com`, `%09`, `#`, `%00`,
  param pollution, `..%2F`, unicode).

### SSRF
- Target: `http://169.254.169.254/latest/meta-data/`, `http://127.0.0.1:6379/` `:9200` `:27017`.
- Bypass IP: `2130706433` `0x7f000001` `0177.0.0.1` `127.1` `[::1]` `[::ffff:127.0.0.1]`,
  redirect chain, DNS rebinding, `%2523@`, unicode `①②⑦.⓪.⓪.①`, `gopher://`.
- Impact: DNS-only = Info, JANGAN submit. Metadata readable = High. + exfil keys = Critical.
- (Catatan the researcher: SSRF metadata via plugin/endpoint yang safety-flagged tetap TUTUP;
  pakai attribution test yang aman sebagai gantinya.)

### XSS
- Sinks: `innerHTML outerHTML document.write eval setTimeout(string) new Function`.
- Chain: stored + admin panel = High. + CSRF token theft = Critical.
- (Catatan: XSS probe ke form publik yang safety-flagged tetap TUTUP; functional test benign OK.)

### SQLi
- JANGAN manual exploit. Tanda -> langsung sqlmap:
  `sqlmap -u 'https://TARGET/api/search?q=test' --batch --level=3 --risk=2`

### GraphQL
- Introspection doang = Info. `node()` bypass per-object auth = bug.
- Batching 100x login = rate limit bypass.

### Race
- `seq 20 | xargs -P 20 -I {} curl -s -X POST https://TARGET/redeem -H "Authorization: Bearer $TOKEN" -d 'code=PROMO10' &`
- Target: kupon, gift, transfer/withdraw, vote, OTP.

### SSTI detect
`{{7*7}}` `${7*7}` `<%= 7*7 %>` `#{7*7}` `*{7*7}` `{{7*'7'}}`
- Jinja2 RCE: `{{config.__class__.__init__.__globals__['os'].popen('id').read()}}`

### ATO taxonomy (7 path)
1. Reset poisoning via Host/X-Forwarded-Host
2. Reset token leak via Referer
3. Token predictable -> ffuf `seq -w 000000 999999`
4. Token reuse / no expiry
5. Email change tanpa password
6. OAuth linking abuse
7. Session fixation (session ID gak ganti habis login)

### File upload bypass
`file.php.jpg` `file.pHp` `.phtml .phar .shtml`, Content-Type spoof, `GIF89a; <?php`,
`.htaccess`, `<svg onload=alert(1)>`, race upload+exec, polyglot, zip-slip.

### LLM/AI
- Wajib chain ke exfil/RCE/ATO. Prompt injection / ASI doang = Info.
- Tool misuse (fetch URL=SSRF, code tool=RCE), IDOR chat history, system prompt extract,
  indirect injection via doc/URL, excessive agency.

### Cache poisoning / smuggling
- Unkeyed header `X-Forwarded-Host X-Original-URL X-Rewrite-URL`, param cloaking `?a=v;poison=xss`,
  fat GET, deception `/account/settings.css`.
- CL.TE / TE.CL / H2.CL, `Transfer-Encoding: xchunked` + tab/space prefix.

## 5. Chain (A->B->C — bayaran 3-10x)
| A | B | C |
|---|---|---|
| IDOR read | PUT/DELETE same endpoint | full manipulate |
| SSRF | metadata | IAM exfil -> RCE |
| stored XSS | HttpOnly miss | hijack -> ATO |
| open redirect | OAuth redirect_uri | code theft -> ATO |
| S3 listing | JS bundle secret | OAuth chain |
| no rate limit | OTP brute | ATO |
| GraphQL introspection | missing field auth | mass PII |
| CORS reflect | +credentials | data theft |
| host header | reset poisoning | ATO |

Protokol: CONFIRM A -> MAP SIBLINGS -> TEST SIBLINGS -> CHAIN -> QUANTIFY (N users / $X / N records) -> 1 report per chain.

## 6. Validate — 7-Question Gate (semua YES sebelum report)
1. Bisa exploit SEKARANG dgn PoC real?
2. Kena user real yg gak ngapa-ngapain?
3. Impact konkret (duit, PII, ATO, RCE)?
4. In-scope sesuai policy?
5. Udah cek duplicate (hacktivity/changelog/issues)?
6. Bukan di always-rejected list (atau udah di-chain)?
7. Triager capek jam 5 sore bakal bilang "yes real bug"?

4 gates: Reality (bug real, in-scope, repro, evidence) -> Impact (attacker bisa apa yg
tadinya gak bisa? ada victim real?) -> Dedup (5 disclosed terakhir dibaca) ->
Quality (judul formula, req copy-paste, screenshot impact, CVSS, fix 1 kalimat, <600 kata).

Validasi the researcher tetap berlaku di atas ini: Claude + GPT + execution test + recheck.
Evidence HARUS ada DI DALAM submission (screenshot/video + req log), bukan cuma di file lokal.

## 7. Selalu ditolak (jangan submit, kecuali sudah di-chain ke impact real)
CSP/HSTS/SPF header doang, introspection doang, banner version tanpa CVE exploit jalan,
clickjacking halaman non-sensitif, tabnabbing, CSV injection, CORS wildcard tanpa PoC exfil creds,
logout CSRF, self-XSS, open redirect doang, client_secret di mobile app, SSRF DNS-ping doang,
host header doang, no rate limit form non-kritis, session gak invalid logout, concurrent session,
IP internal, mixed content, cipher lemah, cookie flag doang, broken link, autocomplete password.

## 8. Report
- Formula judul: `[Class] in [endpoint] allows [actor] to [impact] [scope]`
  Bagus: `IDOR in /api/v2/invoices/{id} allows auth user to read any customer invoice`
  Jelek: `IDOR vulnerability found`
- Template: Title / Summary (2-3 kalimat: apa, dimana, attacker bisa apa) /
  Steps To Reproduce (1. login attacker 2. req exact 3. observe 4. confirm gain) /
  Supporting Material (screenshot/video/req-resp) / Impact (attacker bisa [aksi] -> [harm].
  N users / Y data / $Z) / Severity CVSS 3.1 X.X / fix 1 kalimat.
- 60-detik checklist: judul formula, kalimat pertama impact, req copy-paste, bukti impact
  (bukan cuma 200), 2 akun, CVSS, fix 1 kalimat, no typo path, <600 kata,
  severity = impact (jangan overclaim).
- Counter downgrade: "Requires auth" -> "cuma butuh akun gratis, no role khusus";
  "Limited impact" -> "kena N users / tipe PII / $ amount";
  "Known" -> "minta nomor report, gue search gak ada";
  "By design" -> "minta docs yg bilang intended".
- Humanize first-person, active voice "I found", tanpa em-dash / leverage / seamless / ensure.
- CVSS patokan: IDOR read 6.5 Med, IDOR write 7.5 High, auth bypass admin 9.8 Crit,
  SQLi exfil 8.6 High, SSRF metadata 9.1 Crit, race double-spend 7.5 High, JWT none 9.1 Crit.

## 9. Severity map (Bugcrowd/H1)
P1 Crit: RCE, SQLi, auth bypass full, SSRF metadata+exfil
P2 High: IDOR sensitif, stored XSS B2B, priv esc
P3 Med: stored XSS B2C, SSRF internal, mass assignment
P3/P4: reflected XSS, info disclosure keys, CORS+creds
P4/Info: open redirect, no rate limit, self-XSS

## 10. Note-taking (jangan hunt tanpa ini)
```markdown
# TARGET: x.com -- SESSION 1
## Leads
- [14:22] /api/v2/invoices/{id} no auth visible, testing...
## Dead Ends
- /admin IP restricted (15+ bypass header gagal)
## Anomalies
- GET /api/export 200 tanpa cookie
- POST /api/check-user 150ms vs 8ms (user enum?)
## Rabbit Holes (max 15 mnt)
- [ ] 10 mnt: JWT kid injection
## Confirmed
- [15:10] IDOR /api/invoices/{id} read+write
```

## 11. STATE (tulis tiap beberapa step)
```
STATE @ <target> <HH:MM>
findings: <proven + kandidat + tier>
invariants: <yg harus hold>
tested-safe: <vector -> kenapa gagal>
untouched: <belum disentuh>
next: <3 move konkret>
```

Scaffold: `targets/<target>/scope.md findings/ evidence/ submissions.txt recon/`
Ingat: teori = buang waktu. "Could" bukan bug. Prove it atau drop it.
