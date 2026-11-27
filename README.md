# Bug Bounty Agent Skills

A curated collection of **86 agent skills** for authorized bug bounty hunting
and security research, maintained by Muhammad Zainur ([@buanakeling12](https://github.com/buanakeling12)).
Each skill is a `SKILL.md` playbook (plus references and helper scripts where
noted) that an AI coding agent loads when working a specific phase of a hunt:
scoping, recon, vulnerability-class hunting, validation, triage, and reporting.

Built from real hunting work: every playbook is grounded in public bug bounty
reports and the author's own validated findings across web, API, mobile,
cloud, and smart-contract programs.

## Install

For Claude Code / Muse and other agents that read skills from disk:

```bash
git clone https://github.com/buanakeling12/bug-bounty-agent-skills.git
cp -r bug-bounty-agent-skills/* ~/.claude/skills/   # or your agent's skills dir
```

Or copy only the skills you need. Each skill is self-contained in its own
directory.

## How to use

1. Start with `bb-methodology` — the master orchestrator for a hunting session.
2. Run `reconkit` for scoped, canary-safe recon before touching a target.
3. Pick the playbook for the vulnerability class you are hunting (`hunt-*`).
4. Validate every candidate twice (independent validators + execution test)
   and record evidence with `evidence-hygiene` before writing the report
   (`report-writing`, `reporting-security-findings`).

## Rules of this collection

- **Authorized targets only.** Every skill requires an explicit scope gate:
  a live bug bounty program, a frozen scope document, and a passed
  authorization check before any active testing. No scope, no test.
- **Validate before you file.** A candidate is not a finding until it is
  reproduced with an execution test and re-checked. Validated findings are
  combined into one report unless the program requires separate reports.
- **No public disclosure.** Report through the program's private channel only.

## Catalog

### Core workflow

| Skill | What it does |
| --- | --- |
| `ai-bug-bounty` | AI-assisted bug bounty workflow adapted from Beacon's "Bug bounty with AI" guide — a 5-level tool map (recon → URLs/JS → AI triage → in-scope testing → agent... |
| `bb-methodology` | Use at the START of any bug bounty hunting session, when switching targets, or when feeling lost about what to do next. Master orchestrator that combines the... |
| `bb-recon-pipeline` | Adapted recon pipeline + per-area test checklists for authorized bug bounty hunts. Use at the START of recon on a new target (after scope is frozen) and when... |
| `bb-local-toolkit` | Use the full bug bounty workflow when you also need the local path of a tool, wordlist, or clone (SecLists, jhaddix, ffuf, dalfox, ghauri, trufflehog). Cover... |
| `bb-workflow-0xkonde` |  |
| `bug-bounty` | Guide a bug bounty task from recon and HackerOne scope through research, vulnerability testing, finding validation, and reporting. Use for subdomain and asse... |
| `bug_bounty_hunt` | Test vulnerability classes on an authorized, recon-mapped target before gate validation and reporting. Adapts public bug-bounty hunt patterns per class; ever... |
| `bug-bounty-programs` | Find bug bounty / vulnerability disclosure programs before choosing a hunting target. Sources: local lissy93/bug-bounties database (3,385 programs) or its MC... |
| `reconkit` | Authorized bug bounty recon with a scope gate, sequential recon modules, and canary-based safe validation from C0 to C4. |
| `gate_validasi_hipotesis` | Validate security hypotheses with an anti-premature-rejection gate, from recon to a CONFIRMED, FALSIFIED, or INCONCLUSIVE verdict, before any report. |
| `evidence-hygiene` | Evidence-capture and PoC-redaction discipline for bug-bounty submissions: cookie redaction protocol (which fields to mask, Preview annotation / Burp panel hi... |
### Vulnerability playbooks

| Skill | What it does |
| --- | --- |
| `hunt-api-misconfig` | Hunt API security misconfiguration — mass assignment, prototype pollution, HTTP verb tampering. Mass assignment: send {is_admin:true, role:admin, verified:tr... |
| `hunt-aspnet` | Hunt ASP.NET-specific surface — ViewState deserialization (signed-only vs encrypted), machineKey recovery, dual-parser MAC-bypass anti-pattern, request-valid... |
| `hunt-ato` | Hunt account takeover taxonomy — 9 distinct paths to ATO, plus chains. Paths: (1) password reset flaws (host-header injection redirects token, predictable/nu... |
| `hunt-auth-bypass` | Hunting skill for auth bypass vulnerabilities. Built from 12 public bug bounty reports across SAML XSW / parser-differential (GitHub Enterprise CVE-2025-2529... |
| `hunt-brute-force` | Hunt Missing/Weak Rate Limiting — login brute force, OTP/2FA brute force (10^6 keyspace), password-reset-token brute, credential stuffing, username/email enu... |
| `hunt-business-logic` | Hunting skill for business logic vulnerabilities. Built from 12 public bug bounty reports. Covers coupon-race-stacking (Instacart, Stripe, Reverb), negative-... |
| `hunt-cache-poison` | Hunting skill for cache poison vulnerabilities. Built from 10 public bug bounty reports including X-Forwarded-Host poisoning, X-HTTP-Method-Override / GCS ca... |
| `hunt-captcha-bypass` | Hunt CAPTCHA Bypass |
| `hunt-cicd` | Hunt CI/CD pipeline vulnerabilities — GitHub Actions workflow injection (pull_request_target Pwnrequest + ${{ }}-into-shell), self-hosted runner poisoning, O... |
| `hunt-clickjacking` | Hunt Clickjacking — missing X-Frame-Options / CSP frame-ancestors lets an attacker embed the target page in an invisible iframe and trick victims into clicki... |
| `hunt-cloud-misconfig` | Hunt cloud / infrastructure misconfigurations. AWS: public S3 buckets (s3:GetObject anonymous), permissive bucket policies (PutObjectAcl public-write), expos... |
| `hunt-cors` | Hunt CORS Misconfiguration — origin-reflection with credentials, null-origin trust, subdomain-regex bypass (unanchored vs unescaped-dot vs prefix-only), pre-... |
| `hunt-csrf` | Hunting skill for csrf vulnerabilities. Built from 15 public bug bounty reports including modern variants — SameSite=Lax sibling-subdomain bypass (Argo CD CV... |
| `hunt-deserialization` | Hunt Insecure Deserialization — Java gadget chains (ysoserial), PHP object injection (phpggc), Python pickle RCE, .NET BinaryFormatter, Ruby Marshal.load, JN... |
| `hunt-dispatch` | Skill-set loader for /hunt orchestrator. Fingerprints the target, picks the right platform attack skills, and loads the Red Team or WAPT skill set. Use when... |
| `hunt-dom` | Hunt client-side DOM vulnerabilities — DOM Clobbering (overwrite JS globals via HTML injection), PostMessage hijacking (missing origin check), Service Worker... |
| `hunt-exceptional-conditions` | Hunt mishandling of exceptional conditions — feed an endpoint malformed/unexpected input (wrong type, broken JSON, oversized field, null byte) and make it fa... |
| `hunt-file-upload` | Hunt file upload bugs — RCE via webshell, XSS via SVG/HTML, SSRF via XXE in DOCX, path traversal via filename. Bypass tables (10 techniques): double extensio... |
| `hunt-fintech-graphql` | Hunt fintech-specific GraphQL vulnerabilities: money-movement mutations (transfers, redemptions, withdrawals, card top-ups), ledger/balance/portfolio query I... |
| `hunt-forgot-password` | Hunt Forgot Password / Account Recovery Authentication Flaws — 5 distinct patterns: (1) username enumeration via different responses for valid vs invalid ema... |
| `hunt-graphql` | Hunting skill for graphql vulnerabilities. Built from 12 public bug bounty reports across IDOR via node() / GID, mutation IDOR including AI/LLM features, cro... |
| `hunt-grpc` | Hunt gRPC vulnerabilities — server reflection enabled (enumerate all services/methods), missing authentication / metadata-stripping on internal endpoints, pl... |
| `hunt-host-header` | Hunt Host Header Injection — password reset poisoning → ATO, web cache poisoning via unkeyed Host/X-Forwarded-Host, routing-based SSRF (Host picks upstream →... |
| `hunt-html-injection` | Hunt HTML Injection — user-supplied input is rendered as raw HTML in the response without sanitisation, allowing an attacker to inject arbitrary HTML tags (b... |
| `hunt-http-smuggling` | Hunt HTTP request smuggling (CL.TE, TE.CL, H2.CL, H2.TE). Cause: front-end proxy and back-end server disagree on where one request ends and the next begins (... |
| `hunt-idor` | Hunting skill for idor vulnerabilities. Built from 26 public bug bounty reports. Use when hunting idor on any target. |
| `hunt-jwt-crypto` | Hunt JWT cryptographic failures — alg:none signature-stripping and RS256→HS256 key-confusion that let an attacker forge a token for any identity (e.g. an adm... |
| `hunt-k8s` | Hunt Kubernetes & Docker — API anonymous access, kubelet 10250 exec (SPDY/WebSocket, NOT plain POST) and the simpler /run primitive, etcd 2379 unauth, dashbo... |
| `hunt-laravel` | Hunt Laravel specific vulnerabilities — Debug mode leakage (APP_DEBUG=true exposes full stack trace + env vars), Laravel Telescope/Horizon dashboard unauthor... |
| `hunt-ldap` | Hunt LDAP Injection and XPath Injection — authentication bypass, blind char-by-char attribute exfiltration, AD user/group enumeration, XML-store XPath bypass... |
| `hunt-lfi` | Hunt Local File Inclusion (LFI), Remote File Inclusion (RFI), and Path Traversal — /etc/passwd read, log poisoning → RCE, PHP filter-chain RCE (no upload nee... |
| `hunt-llm-ai` | Hunt LLM/AI feature bugs — prompt injection, indirect injection, exfiltration via tool-use/markdown, ASCII smuggling, agentic AI security (OWASP Agentic Apps... |
| `hunt-mfa-bypass` | Hunt MFA / 2FA bypass — 7 distinct patterns. (1) MFA not enforced on sensitive endpoints (password change, email change accept without MFA challenge), (2) MF... |
| `hunt-misc` | Hunting skill for misc vulnerabilities. Built from 225 public bug bounty reports. Use when hunting misc on any target. |
| `hunt-nextjs` | Hunt Next.js specific vulnerabilities — Server Actions arbitrary function execution, Middleware auth bypass via static asset paths, ISR cache poisoning, Imag... |
| `hunt-nodejs` | Hunt Node.js specific vulnerabilities |
| `hunt-nosqli` | Hunt NoSQL Injection — MongoDB operator injection ($where, $regex, $gt, $ne), CouchDB, Redis command injection, auth bypass via NoSQLi, data dump. Use when t... |
| `hunt-ntlm-info` | Hunt NTLM/Negotiate information disclosure on internet-reachable IIS/SharePoint/Exchange. Anonymous NTLM Type-2 challenge capture leaks NetBIOS domain, inter... |
| `hunt-oauth` | Hunting skill for oauth vulnerabilities. Built from 19 public bug bounty reports. Use when hunting oauth on any target. |
| `hunt-open-redirect` | Hunt Open Redirect — all types including low-impact, chained to OAuth token theft → ATO, phishing chains. URL parameter manipulation, JavaScript redirect, me... |
| `hunt-race-condition` | Hunting skill for race condition vulnerabilities. Built from 12 public bug bounty reports including modern HTTP/2 single-packet attack cases (James Kettle DE... |
| `hunt-rag-vector` | Hunt vector-store / embedding-layer weaknesses in RAG pipelines (OWASP LLM08 Vector and Embedding Weaknesses) — persistent corpus poisoning that survives acr... |
| `hunt-rate-limit` | Hunt rate-limit bypass — the 12-vector cheat sheet: IP rotation, X-Forwarded-For / X-Originating-IP / X-Remote-IP / X-Client-IP / X-Host header spoofing (inc... |
| `hunt-rce` | Hunting skill for rce vulnerabilities. Built from 67 public bug bounty reports. Use when hunting rce on any target. |
| `hunt-saml` | Hunt SAML / SSO attacks. Patterns: XML Signature Wrapping (XSW) — modify Assertion while keeping Signature valid by relocating signed element, comment inject... |
| `hunt-session` | Hunt Session Management vulnerabilities — session fixation (no regeneration on login), insufficient invalidation on logout / password-change / email-change,... |
| `hunt-shadow-api` | Hunt shadow / zombie / undocumented API surface (OWASP API9 Improper Inventory Management) — enumerate the full API version history (v1/v2/beta/legacy paths,... |
| `hunt-sharepoint` | Hunt Microsoft SharePoint Server (2013/2016/2019/Subscription Edition) on-prem farms — anonymous endpoint enumeration, version disclosure, legacy SOAP login... |
| `hunt-source-leak` | Hunt source code and build artifact leakage — JavaScript source maps (.js.map) reconstructing TypeScript/ES6 source, Swagger/OpenAPI JSON endpoint discovery,... |
| `hunt-spa-api` | Discover a single-page-app's hidden backend API from its public JS bundle, then test that API for broken access control / missing authentication. One of the... |
| `hunt-springboot` | Hunt Spring Boot specific vulnerabilities — Actuator endpoints (heapdump, env, loggers, mappings, shutdown), Spring Expression Language (SpEL) injection → RC... |
| `hunt-sqli` | Hunting skill for sqli vulnerabilities. Built from 12 public bug bounty reports including modern NoSQL injection (Rocket.Chat CVE-2021-22911 MongoDB $regex,... |
| `hunt-ssrf` | Hunting skill for ssrf vulnerabilities. Built from 15 public bug bounty reports including AWS metadata SSRF (HackerOne $25k Analytics PDF, Shopify Exchange $... |
| `hunt-ssti` | Hunt server-side template injection (SSTI) across Jinja2 (Flask/Django), Twig (Symfony), Freemarker (Java), ERB (Rails), Spring, Velocity, Mako, Thymeleaf, S... |
| `hunt-subdomain` | Hunting skill for subdomain takeover vulnerabilities. Includes modern provider fingerprints — Microsoft Azure DevOps `cloudapp.azure.com` regional-pool re-is... |
| `hunt-tls-network` | Hunt TLS/SSL and DNS misconfigurations — missing HSTS (downgrade attack), weak cipher suites, expired/invalid certificates, mTLS bypass, missing SPF/DKIM/DMA... |
| `hunt-websocket` | Hunt WebSocket vulnerabilities — Cross-Site WebSocket Hijacking (CSWSH), missing/weak Origin validation on the WS handshake, no per-message authentication, m... |
| `hunt-xss` | Hunting skill for xss vulnerabilities. Built from 174 public bug bounty reports. Use when hunting xss on any target. For markup injection that reflects raw H... |
| `hunt-xxe` | Hunting skill for xxe vulnerabilities. Built from 10 public bug bounty reports including SVG-upload XXE, Office-doc (PPTX/DOCX) XXE, SOAP XXE, SAML Assertion... |
### Smart-contract & Web3 audit

| Skill | What it does |
| --- | --- |
| `evm-foundry` | Foundry development for EVM chains including Celo. Use when working with forge, cast, anvil, writing Solidity contracts, testing, deploying, or verifying con... |
| `evm-hardhat` | Hardhat development for EVM chains including Celo. Use when setting up Hardhat projects, writing Solidity contracts, compiling, testing, deploying, or verify... |
| `bsc-contract-audit` | Evidence-based due-diligence audit for BNB Smart Chain (chain 56) token contracts, NFT mints, and deposit-refund "lock-to-allocate" escrows — verification st... |
| `launch-curve-audit` | Method and tooling to audit a token launch curve on BSC — bonding-curve or tranche IEO, Agent-ID or allowlist gating, escrowed bids, graduation into a pool,... |
| `contract-verification` | Verify smart contracts on Celo. Use when publishing contract source code to Celoscan or Blockscout. |
### Recon & fuzzing

| Skill | What it does |
| --- | --- |
| `ffuf-mastery` | Master ffuf (Fuzz Faster you Fool) for bug bounty recon and pen testing — directory brute force, recursion, multi-keyword fuzzing, calibration, auth, match/f... |
| `api-recon` | 收集网站API接口时调用该skill。 |
| `playwright-cli` | Automate browser interactions, test web pages and work with Playwright tests. |
| `artex-scopesentry` | Manage a ScopeSentry security-scan platform via MCP (projects, tasks, templates, assets, nodes). Use when the user mentions ScopeSentry, MCP, API key, scan t... |
### Triage & reporting

| Skill | What it does |
| --- | --- |
| `security-audit` | Security guidance and vulnerability review for codebases, APIs, services, CLI tools, libraries, and daemons. Use for security questions, focused reviews, vul... |
| `security-bounty-hunter` | Hunt for exploitable, bounty-worthy security issues in repositories. Focuses on remotely reachable vulnerabilities that qualify for real reports instead of n... |
| `vulnerability-triage-brocards` | >- |
| `triage-validation` | Finding validation before writing any report — 7-Question Gate (all 7 questions), 4 pre-submission gates, always-rejected list, conditionally valid with chai... |
| `cvss-scoring` | Produces CVSS v4.0 vector strings from plain-text vulnerability descriptions by systematically evaluating base metrics. Use when scoring a vulnerability, gen... |
| `report-writing` | Bug bounty report writing for H1/Bugcrowd/Intigriti/Immunefi — report templates, human tone guidelines, impact-first writing, CVSS 3.1 scoring, title formula... |
| `reporting-security-findings` | Write security findings and assessment reports — severity scoring with CVSS and business impact, reproducible proof of concept, remediation guidance, executi... |

## Attribution

This collection mixes original work with adaptations of public methodology.
Each skill file records its source in its frontmatter where adapted:

- `hunt-*` playbooks, `bb-methodology`, and several core skills are adapted
  from [Claude-BugHunter](https://github.com/elementalsouls/Claude-BugHunter)
  (MIT) — playbooks built from public bug bounty reports.
- `ai-bug-bounty` is adapted from Beacon's
  [Bug bounty with AI](https://aibugbounty.beacon.host/) guide.
- `bb-workflow-0xkonde` is adapted from 0xkonde's bug bounty workflow.
- `bb-recon-pipeline` adapts the public 16-step recon methodology
  (tuhin1729 / yogsec).
- `reconkit` adapts the public ReconKit methodology.
- Everything else is original work by the maintainer.

Third-party licenses travel with their skills; see each skill's frontmatter
(`source`, `license`, `license_source`). This repository as a whole is MIT
licensed — see [LICENSE](LICENSE).

## Contributing

Found a wrong offset, a stale payload, or a playbook that no longer matches
reality? Open an issue or a PR with the public report or PoC that proves the
correction. No unverified claims, no invented data.

## License

MIT — see [LICENSE](LICENSE).
