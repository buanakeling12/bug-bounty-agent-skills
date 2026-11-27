---
name: ai-bug-bounty
description: AI-assisted bug bounty workflow adapted from Beacon's "Bug bounty with AI" guide — a 5-level tool map (recon → URLs/JS → AI triage → in-scope testing → agents/source review) plus a 7-step pipeline, prompt templates, and report-writing rules, all re-routed through the researcher's scope gate, validators, and evidence rules.
category: security
risk: offensive
source: https://aibugbounty.beacon.host/
source_type: community
date_added: '2026-11-26'
license: unknown
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

> **Adaptation note (2026-11-26)**
> Adapted from Beacon's "Bug bounty with AI" (aibugbounty.beacon.host).
> Changes vs the original: (1) the site's "read the policy" step is replaced
> by the researcher's hard gate — frozen SCOPE.md + bounty_gate.py check BEFORE any
> tool touches a target; (2) the site's Ollama/local-model assumption is
> replaced by the researcher's real validators — Claude Opus 5.5 (vault helper) +
> BeastFarmer gpt-6-luna / Junie Lite (free) — because he does not run
> Ollama; Gemini is NOT used for sensitive bounty data (it trains on data);
> (3) added the validation gate (Claude + GPT + execution test + recheck)
> and the evidence gate (screenshot/video + PoC attached) before anything is
> filed; (4) agent tools (PentestGPT, Strix, Shannon, hackingBuddyGPT) are
> restricted to lab / staging / local source-available work — the researcher's hunts
> run against live programs, so agents never drive a live target; (5) Burp
> extensions (ReconAIzer, BurpGPT, Burp AT) are marked optional — the researcher
> works in browser + curl + playwright, not Burp; (6) tool availability is
> noted per tool against hermes-wb (nuclei v3.11.1 + ffuf 2.1.0 confirmed
> there); (7) filing always needs the researcher's per-finding approval, and
> reports are humanized before submit.

# AI-Assisted Bug Bounty Workflow (adapted)

The core idea, taken from the source: **AI sits on top of a boring, careful
pipeline.** If you skip mapping and triage and jump to an agent, you waste
money, leave scope, or file junk. Work the levels in order. Every command
below assumes a written allowlist.

The 5 levels:

| Level | Job | Tools |
|-------|-----|-------|
| 1 | Map the surface — find what is actually in scope | Subfinder, assetfinder, Amass, dnsx, httpx, naabu |
| 2 | Collect URLs, params and JS — see the app, not just the host | katana, waybackurls, gau, waymore, ParamSpider, Arjun, gf, LinkFinder |
| 3 | AI at your desk — triage, templates, help in the proxy | Claude Opus + gpt-6-luna (the researcher's validators), Nuclei -ai, Burp tools (optional) |
| 4 | Hunt in scope — fuzz, screenshot, prove carefully | ffuf, Dalfox, gowitness, interactsh |
| 5 | AI agents & source review — labs, staging, source-available only | PentestGPT, hackingBuddyGPT, Strix, Shannon, Vulnhuntr |

Then the 7-step pipeline (below), the prompt templates, and the
report-writing rules.

## The 7-step pipeline

### 1. Read the policy and freeze the scope
Before any tool, open the program brief. Write down the in-scope roots,
the out-of-scope list, rate limits, and whether crawling, automated
scanning, out-of-band callbacks or credentialed testing are allowed.
**Freeze it as SCOPE.md** (`~/workspace/hunts/<program>/SCOPE.md`) and run
the bounty gate (`bounty_gate.py check`) before touching anything.
If a subdomain is related but not listed, it is out until the program
says otherwise.

Do not hand a model a target and ask it to "go hunt." Ask it to help you
stay organised only after the allowlist exists.

### 2. Map the surface
Enumerate names for each in-scope root with Subfinder (and optionally
assetfinder / Amass). Resolve with dnsx. Probe HTTP with httpx and keep
one live-host file. Use naabu only if the policy clearly allows port
scanning.

This step is not optional just because you have an AI assistant. Models
invent hosts. Your lists have to come from tools and from the program
scope.

### 3. Collect URLs, parameters and JavaScript
Crawl live hosts with katana. Pull historical URLs with waybackurls, gau
and, if you want a deeper pass, waymore. Mine parameterized URLs with
ParamSpider, discover live parameters with Arjun (rate-limited), grep the
pile with gf, and extract endpoints from JavaScript with LinkFinder.

Filter every URL back to the allowlist. Historical links love to wander
into other brands, expired apps and third-party CDNs.

### 4. Triage with AI, preferably local
This is the highest-value AI step. Feed a model a slice of your URL list
or a JavaScript file and ask it to group, label and flag leftovers.
Use the researcher's validators (Claude Opus via the vault helper + BeastFarmer
gpt-6-luna), never a cloud chatbot for sensitive program data.
Use Nuclei's `-ai` flag for a specific hypothesis on an in-scope host,
and read the generated template before a wider run. Do not paste session
cookies, passwords, or customer data into any model.

### 5. Test, still in scope
Fuzz unlinked paths with ffuf at a sensible rate. Run Nuclei templates
against the live-host file, not against the internet. Pipe parameterized
URLs into Dalfox if XSS testing is allowed. Screenshot interesting hosts
with gowitness. Use interactsh only when the program allows out-of-band
checks and you already have a reason.

Record the exact command, the time, and the response that mattered. You
will need that for the report, and a model cannot invent it later.

### 6. Agents last, and only in a safe place
PentestGPT, hackingBuddyGPT, Strix and Shannon can plan and act. Point
them at a lab, a CTF, or a staging / local copy of an app you are
authorized to test. Shannon's own documentation says it executes real
exploits and must not be run against production. Give the agent the
allowlist, watch the log, and stop it if it drifts.

If a program gives you source, Vulnhuntr (Python) or Strix / Shannon on
a local checkout can help you read it. Reproduce any claimed bug
yourself.

**For the researcher's live-program hunts: agents do NOT drive the target.**
They are a lab/staging tool only. Live hunting stays manual-style with
the validators + execution test.

### 7. Write the report like a human who used a typist
Reproduce the issue. Save the request, the response and the impact.
Then let a model help you phrase the title, the summary and the fix,
using only the notes you already verified. You file the report; the
model does not. Details are in the report-writing section below.
Then humanize the report (no em dashes, no bold-label lists, no triads,
no staged openers, sentence-case headings) and attach the evidence
(screenshot/video + PoC) before filing — and only with the researcher's
per-finding approval.

## Level 1 — Map the surface

Start here, before any model. Read the program policy, write down every
host you are allowed to touch, then enumerate subdomains, resolve them
and see which ones answer. AI cannot invent a valid attack surface.

### Subfinder (ProjectDiscovery) — Tried
Fast passive subdomain enumeration: queries many public sources and
returns names for a domain without sending traffic to the target itself.
For anyone starting recon on an in-scope root domain.
Heads-up: passive only. It does not tell you which names are live; pair
it with dnsx and httpx.
```
go install -v github.com/projectdiscovery/subfinder/v2/cmd/subfinder@latest
subfinder -d in-scope.example -o subs.txt
```
Tip: keep a clean wordlist of program roots and run Subfinder per root,
not against a sibling brand that is out of scope.

### assetfinder (tomnomnom) — Tried
Finds domains and subdomains related to a given name using sources such
as crt.sh, the Wayback Machine and others. A quick second opinion beside
Subfinder, especially on smaller scopes.
Heads-up: some sources need API keys (the README lists them). Related
names are not automatically in scope; you still filter.
```
go get -u github.com/tomnomnom/assetfinder
assetfinder --subs-only in-scope.example
```
Tip: the README also offers platform releases if you do not want to build
from source. Always drop anything that is not in the program scope.

### Amass (OWASP) — Tried
OWASP's attack-surface mapping framework: much more than a subdomain
tool, with passive and active techniques and a long-term asset database.
For hunters who want deeper mapping than a single passive enum, and teams
tracking a program over time.
Heads-up: active modes send traffic. Read the program rules before any
active enum, and start with a narrow, explicit scope.
```
CGO_ENABLED=0 go install -v github.com/owasp-amass/amass/v5/cmd/amass@main
amass enum -d in-scope.example
```
Tip: official docs also cover Homebrew and Docker.

### dnsx (ProjectDiscovery) — Tried
A fast DNS toolkit: take a list of names and see which resolve, with
records and responses you can keep for later. For anyone who just pulled
a long subdomain list and needs to know what is real.
Heads-up: it resolves names, it does not probe HTTP or find
vulnerabilities.
```
go install -v github.com/projectdiscovery/dnsx/cmd/dnsx@latest
cat subs.txt | dnsx -silent -a -resp -o resolved.txt
```
Tip: resolve first, probe second. A name that does not resolve is not a
live target.

### httpx (ProjectDiscovery) — Tried
Fast HTTP toolkit: feed it hosts and it reports which answer, with status
codes, titles, tech detection and more. For anyone with a list of
in-scope hosts to sort before crawling or scanning.
Heads-up: it probes, it does not find vulnerabilities.
```
go install -v github.com/projectdiscovery/httpx/cmd/httpx@latest
cat resolved.txt | httpx -title -status-code -tech-detect -o live.txt
```
Tip: save one live-host file and reuse it. Do not point later scanners
at unresolved or out-of-scope names.

### naabu (ProjectDiscovery) — Tried
A fast port scanner written in Go, focused on reliability and simplicity.
For hunters whose program allows port scanning, when HTTP on 80/443 is
not the whole surface.
Heads-up: port scans are noisy and often restricted. Check the policy
first. Needs libpcap (libpcap-dev on Linux, Npcap on Windows).
```
go install -v github.com/projectdiscovery/naabu/v2/cmd/naabu@latest
naabu -list resolved.txt -o ports.txt
```
Tip: if the program forbids port scanning, skip this tool. When you do
use it, keep the host list inside scope and avoid default-wide internet
ranges.

#### Level 1 checklist
- I have the program's written scope and I know what is out of bounds
- I have a single allowlist of in-scope roots and I do not scan anything else
- I can resolve and probe live hosts without spraying the whole internet

## Level 2 — Collect URLs, params and JS

Most bounty bugs live on pages and parameters nobody linked on the
marketing site. Crawl what is live, pull historical URLs, mine parameters
and read the JavaScript. This is also the raw material you will later hand
to a model.

### katana (ProjectDiscovery) — Tried
A next-generation crawling and spidering framework that walks a site,
including JavaScript-heavy pages in headless mode, and lists the
endpoints it finds. For testers building a full in-scope URL list before
scanning or asking a model to triage.
Heads-up: a crawler only sees what is linked or loaded. Pair it with
historical URL tools and a fuzzer for hidden paths.
```
CGO_ENABLED=1 go install github.com/projectdiscovery/katana/cmd/katana@latest
katana -u https://in-scope.example -o crawl.txt
```
Tip: stay on the in-scope host; do not follow links off into a different
brand.

### waybackurls (tomnomnom) — Tried
Fetches all the URLs that the Wayback Machine knows about for a domain.
For anyone who wants historical paths, old admin panels and forgotten
files.
Heads-up: historical URLs may be dead, or may point at hosts that are no
longer in scope. Filter before you visit them.
```
go install github.com/tomnomnom/waybackurls@latest
cat domains.txt | waybackurls > wayback.txt
```
Tip: run it on in-scope roots only, then keep URLs whose host still
matches your allowlist.

### gau (lc) — Tried
Fetches known URLs from AlienVault OTX, the Wayback Machine, Common
Crawl and URLScan. For hunters who want more history than the Wayback
Machine alone.
Heads-up: provider APIs can be slow or rate-limited. Be considerate, and
still filter to in-scope hosts.
```
go install github.com/lc/gau/v2/cmd/gau@latest
gau in-scope.example --subs
```
Tip: --subs includes subdomains of the target domain. Drop any name that
is not in the program scope before you crawl or scan.

### waymore (xnl-h4ck3r) — Tried
Pulls way more from the Wayback Machine, Common Crawl, AlienVault OTX,
URLScan and VirusTotal: URL lists, and optionally downloaded archived
responses. For hunters who want a deeper historical pass than gau or
waybackurls, including old response bodies.
Heads-up: downloading responses (mode R or B) is slow and heavy. Archived
pages can contain secrets; keep them private.
```
pip install waymore
waymore -i in-scope.example -mode U
```
Tip: mode U collects URLs only and does not download responses.

### ParamSpider (devanshbatham) — Tried
Mines URLs with parameters from web archives for a domain or a list of
domains, and filters out the boring ones. For hunters building a list of
parameterized endpoints to fuzz or to review with a model.
Heads-up: it mines archives, it does not discover live hidden parameters
(use Arjun for that).
```
git clone https://github.com/devanshbatham/paramspider
cd paramspider && pip install .
paramspider -d in-scope.example
```
Tip: use -l domains.txt for several in-scope roots. Review the output
hosts before you send them to a fuzzer.

### Arjun (s0md3v) — Tried
HTTP parameter discovery suite: finds query and body parameters on a URL,
with GET, POST, JSON and XML support and a large default wordlist. For
anyone staring at an in-scope endpoint that looks like it takes more
input than it shows.
Heads-up: it sends many requests. Use --ratelimit or --stable, and only
on endpoints you are allowed to test.
```
pipx install arjun
arjun -u https://in-scope.example/endpoint --ratelimit 2
```
Tip: the official wiki documents -u, import with -i, and export to Burp,
text or JSON. Prefer --ratelimit or --stable on public programs.

### gf (tomnomnom, patterns by 1ndianl33t) — Tried
A wrapper around grep with named pattern files. Community Gf-Patterns
add ready-made patterns for things like interesting parameters in a URL
list. For anyone sorting a huge URL or JS dump before a human or a model
looks at it.
Heads-up: patterns are hints, not findings. A match is a place to look,
not a vulnerability.
```
go install github.com/tomnomnom/gf@latest
cat urls.txt | gf xss
```
Tip: copy gf's example patterns into ~/.gf, then add
1ndianl33t/Gf-Patterns (git clone, then mv *.json ~/.gf).

### LinkFinder (GerbenJavado) — Tried
A Python script that finds endpoints and URLs inside JavaScript files,
so you can see APIs the HTML never linked to. For hunters (and later, a
local model) reviewing in-scope JavaScript.
Heads-up: older project. It extracts links, it does not test them. Some
URLs it finds may be out of scope or third-party.
```
git clone https://github.com/GerbenJavado/LinkFinder.git
cd LinkFinder && pip3 install -r requirements.txt
python3 linkfinder.py -i https://in-scope.example/app.js -o cli
```
Tip: collect JS from your crawl first, run LinkFinder on those files,
then keep only endpoints on in-scope hosts. That list is excellent
context for a model.

#### Level 2 checklist
- My URL list only contains hosts in the written scope
- I have a separate file of JavaScript and parameterized URLs
- I set a rate limit before crawling or fuzzing parameters

## Level 3 — AI at your desk

This is where AI earns its keep in a bounty: sorting huge URL and JS
piles, drafting Nuclei templates, and commenting on traffic you already
captured. Prefer a local model when the data is sensitive. Verify every
suggestion by hand.

**The researcher's validators (replace the site's Ollama assumption):**
- **Claude Opus 5.5** — paid key, via the vault helper
  `~/workspace/custom-connectors/custom.anthropic/bin/anthropic.py`
  (`ask --prompt-file <file> --model claude-opus-5-5 --max-tokens 12000`).
  Exploit-framed briefs get refused — use a defensive-audit reframe.
  For SSRF topics Opus refuses ALL framings — do not retry, do not
  fabricate a verdict.
- **BeastFarmer gpt-6-luna** — free, via
  `~/workspace/custom-connectors/custom.beastfarmer/bin/beastfarmer.py`
  (`ask --model gpt-6-luna --prompt "<text>"`). Use for the GPT side of
  validation.
- **Junie Lite** — free until 31 Dec 2026, via
  `~/workspace/custom-connectors/custom.junie/bin/junie.py`
  (`ask --prompt "..." --model junie-lite`). Another free GPT-validation
  route.
- **Never** send sensitive bounty data through Gemini (it trains on
  data). Never paste tokens, session cookies or customer data into a
  third-party chat.

### Nuclei (ProjectDiscovery) — Tried
Template-based vulnerability scanner. You can run the community
templates, write your own, or use the -ai / -prompt flag to generate and
run a template from a plain-English prompt. For bug bounty hunters and
AppSec teams checking an in-scope surface for known issues and for a
specific hypothesis.
Heads-up: it only finds what a template covers. The AI template feature
talks to ProjectDiscovery's cloud (described as free); plain template
scans do not need that. Treat every hit as unverified.
**Confirmed on hermes-wb: nuclei v3.11.1 + templates.**
```
go install -v github.com/projectdiscovery/nuclei/v3/cmd/nuclei@latest
nuclei -l live.txt -rate-limit 30 -o nuclei-hits.txt
```
Tip: for a single in-scope idea:
`nuclei -u https://in-scope.example -ai "look for exposed env or git files"`.
Read the generated template before you trust a wider run. Set rate
limits. Never scan hosts outside the allowlist.

### ReconAIzer (hisxo) — Tried — OPTIONAL, Burp only
A Burp Suite extension that sends a request or response to GPT and helps
suggest endpoints, parameters, URLs and subdomains for bug-bounty recon.
For hunters already living in Burp who want a second pair of eyes on
captured traffic.
Heads-up: older project (last updated 2023) and uses the OpenAI API, so
request data leaves your machine. Review suggestions against scope.
**The researcher does not use Burp — this is optional, not core.** Load the
extension in Burp's Extender tab, set your OpenAI API key on the
ReconAIzer Config tab. Strip cookies and tokens before anything goes to
a cloud API.

### BurpGPT (aress31) — Tried — OPTIONAL, Burp only
A Burp Suite extension that adds a GPT-powered passive scan and lets you
run custom prompts over captured traffic. For testers who want bespoke,
prompt-driven checks on in-scope traffic they already have.
Heads-up: older project (last updated 2024). Passive analysis only.
Traffic you include in a prompt is sent to the model provider you
configure. **Optional for the researcher (no Burp).** Clone the repo, build with
Gradle, load the burpgpt-all jar from lib/build/libs. Write prompts that
ask for questions and leads, not for exploits. Keep cookies and secrets
out of the prompt.

### Burp AT (PortSwigger) — Tried — OPTIONAL, Burp only
PortSwigger's agentic assistant in Burp Suite Professional: it can
explain selected text, explore HTTP messages in Repeater and investigate
Scanner issues, using Burp's own tools under your approval settings. For
web testers on Burp Suite Professional 2026.7 or later.
Heads-up: paid; not in Burp Community. Needs a PortSwigger account and
outbound access to PortSwigger's AI services. You are still responsible
for scope and for verifying results. **Optional for the researcher (no Burp).**
Start with tight permissions (approve actions yourself) until you trust
the task.

#### Level 3 checklist
- I know which data I will send to a cloud model and which stays local
- I never paste tokens, session cookies or customer data into a third-party chat
- I treat every model suggestion as unverified until I reproduce it

## Level 4 — Hunt in scope

Now you test. Keep rate limits on, stay inside the allowlist, and record
every command. AI can help you choose what to try next; it does not get
to fire the fuzzer for you on a whim.

### ffuf — Tried
Fast web fuzzer written in Go: swap the FUZZ keyword anywhere in a URL,
header or body with a wordlist to find hidden files, folders, parameters
and virtual hosts. For anyone hunting for unlinked content on an
in-scope host.
Heads-up: only as good as your wordlist, and it is loud. Set a rate
limit. Watch for wildcard responses so you do not record 10,000 fake
hits.
**Confirmed on hermes-wb: ffuf 2.1.0.**
```
go install github.com/ffuf/ffuf/v2@latest
ffuf -u https://in-scope.example/FUZZ -w wordlist.txt -rate 20
```
Tip: filter by status and size, and stop if the host is clearly blocking
you or if you leave scope.

### Dalfox (hahwul) — Tried
Open-source XSS scanner and utility focused on automation: it analyses
parameters and checks reflected, stored and DOM XSS, and it is easy to
pipe after a crawler. For hunters running XSS checks across a list of
in-scope parameterized URLs.
Heads-up: finds XSS only. It sends payloads to the target, so the program
must allow this kind of testing. Official warning: download only from the
project's GitHub Releases or the official install guide; fake download
pages have distributed malware.
```
brew install dalfox
cat urls.txt | dalfox scan
```
Tip: official alternatives include GitHub Releases, snap, and the
installation guide on dalfox.hahwul.com. Point it at in-scope URLs only,
not at a raw internet-wide list.

### gowitness (sensepost) — Tried
A Go website-screenshot utility that uses headless Chrome, with a report
viewer. Useful for visually triaging a pile of live hosts (the idea
behind the older, now-archived Aquatone). For hunters who want to glance
at many in-scope sites before spending time in the proxy.
Heads-up: screenshots can include sensitive data from authenticated
sessions. Keep the report private. Aquatone itself is archived; use
gowitness instead.
```
go install github.com/sensepost/gowitness@latest
gowitness scan single --url "https://in-scope.example" --write-db
```
Tip: the README covers list and file scan types as well. Only screenshot
hosts on your allowlist.

### interactsh (ProjectDiscovery) — Tried
An out-of-band interaction gathering server and client: you get a unique
callback host and can see if an in-scope app later fetches it (useful for
some SSRF, XSS and similar checks). For hunters whose program allows
out-of-band testing, and who already have a hypothesis that needs a
callback.
Heads-up: OOB testing is sometimes disallowed. Do not embed callbacks in
places that could hit other customers. You still have to prove impact in
scope.
```
go install -v github.com/projectdiscovery/interactsh/cmd/interactsh-client@latest
interactsh-client
```
Tip: start the client, use the hostname it prints only in authorized
tests, and keep the session log for your report. Self-hosting the server
is documented if you do not want the public service.
**The researcher note:** webhook.site is the proven OOB alternative in his hunts
(used for the Sophos SSRF PoC) — same role, no install needed.

#### Level 4 checklist
- Every target in this level is in my written scope
- I have a rate limit set on fuzzers and scanners
- I can reproduce a finding myself before I write it up

## Level 5 — AI agents & source review

Agents can plan, run tools and write a draft report. Treat them like a
junior on day one: give them a sandbox, a hard scope and an approval
step, and read everything they did. Several of these tools say themselves
they are not for production.

**The researcher rule: labs, staging, and source-available programs ONLY. Never
point an agent at a live program target.**

### PentestGPT (GreyDGL) — Tried
An agentic penetration-testing framework powered by LLMs, from the
research behind the USENIX Security 2024 paper of the same name. The
current v1 agent can drive Claude Code or Codex through staged phases; a
legacy interactive mode supports more providers, including local Ollama.
For learners and hunters working through labs, CTFs and other authorized
practice targets.
Heads-up: the default examples are lab and CTF style. Real programs need
a human in charge, a written scope, and no unsupervised run against
production.
```
git clone https://github.com/GreyDGL/PentestGPT.git
cd PentestGPT && make install
pentestgpt --target 10.10.11.234
```
Tip: that usage line is the project's own CTF-style example. Point it
only at a lab or an authorized target. make install runs uv sync; you
also need Python 3.12+ and the CLI backends the README lists.

### hackingBuddyGPT (TU Wien, ipa-lab) — Tried
A research framework for LLM hacking agents in a small amount of code,
with use cases such as Linux privilege escalation and web testing. The
installed command is wintermute. For researchers and builders who want to
write or study their own agent, and testers running a published use-case
against a lab.
Heads-up: a framework to build with, not a push-button bounty scanner.
Lab and authorized targets only.
```
git clone https://github.com/ipa-lab/hackingBuddyGPT.git
cd hackingBuddyGPT && uv sync
wintermute
```
Tip: running wintermute with no arguments lists registered use-cases;
wintermute <UseCase> --help shows options.

### Strix — Tried
Open-source AI penetration-testing agents that run against an app, look
for vulnerabilities and try to validate them with proofs of concept. It
can hook into CI and has a local results viewer. For developers and
hunters testing an app they own, or a copy of a source-available target
they are authorized to test.
Heads-up: needs an LLM API key. Not a replacement for a scoped human
test. Do not point it at production or at someone else's repo without
permission.
```
curl -sSL https://strix.ai/install | bash
strix --target ./app-directory
```
Tip: that usage line is the official 'scan a local codebase' example.
Prefer a local checkout of an in-scope app. The installer also mentions
pipx, uv and pip (package name strix-agent). Review every proof yourself.

### Shannon (Keygraph) — Tried
A white-box AI pentester for web apps and APIs: it analyses your source,
then executes real exploits against the running app. Its own tagline is
'No exploit, no report.' For teams and hunters with source access to an
app they are authorized to test, usually in staging or a local
environment.
Heads-up: the official README warns — Shannon actively executes
exploits; run it only against applications you own or have written
authorization to test; do not run it against production. Findings still
need human review.
```
npx @keygraph/shannon@latest
npx @keygraph/shannon@latest start -u https://your-app.com -r /path/to/your/repo
```
Tip: run npx @keygraph/shannon@latest setup first. Needs Docker,
Node.js 22+ and your own model key.

### Vulnhuntr (Protect AI) — Tried
LLM-driven static analysis that follows call chains through a Python
codebase to look for remotely exploitable bugs. Can use Claude, GPT or a
local Ollama model. For people reviewing an in-scope, source-available
Python project, or their own code.
Heads-up: Python codebases only, and the project says it strictly
requires Python 3.10. Do not scan untrusted repos lightly: a malicious
codebase can try prompt injection.
```
pipx install git+https://github.com/protectai/vulnhuntr.git --python python3.10
vulnhuntr -r /path/to/in-scope/repo/
```
Tip: add -l ollama to keep analysis local (the researcher: use the Claude /
gpt-6-luna validators instead — he has no Ollama). Treat every report as
a lead and reproduce it in a safe environment before you file.

#### Level 5 checklist
- The agent runs in a lab, VM or staging app I am allowed to test
- I have written authorization and a hard allowlist before it starts
- I review the full log and reproduce any finding myself

## Example prompts that stay useful

Good prompts give the model your scope, your evidence and a hard rule:
do not invent hosts, steps or severity. Swap in-scope.example for a host
on your allowlist. Do not paste secrets.

**Send these to Claude Opus (vault helper) or gpt-6-luna (BeastFarmer),
not to a cloud chatbot. For SSRF topics Opus refuses — use gpt-6-luna
and treat its verdict as provisional.**

### Triage a URL list
Use after gau, waybackurls or katana. Paste only in-scope URLs.
```
You are helping me triage recon for an authorized bug bounty program. The only in-scope hosts are listed below. Group these URLs by likely function (auth, upload, admin, API, debug, static). Flag leftovers such as env, backup, git, or old admin paths. Quote the URL you used for each flag. Do not add hosts or paths that are not in the list. Do not tell me how to exploit anything; just sort and explain why a path looks interesting.

Allowlist:

in-scope.example

URLs:

[paste]
```

### Read JavaScript for endpoints
Use after LinkFinder, or on a JS file you already fetched from an
in-scope host.
```
This JavaScript was loaded from an in-scope host I am authorized to test. List API endpoints, hidden parameters, interesting comments, and any hardcoded URLs. For each item, quote the snippet you used. Mark anything that is clearly a third-party host so I can ignore it if it is out of scope. Do not invent endpoints that are not in the file. Do not write exploit payloads.

[paste JS]
```

### Draft a Nuclei idea, then you review it
Use with Nuclei's -ai flag only after you have a hypothesis, or ask a
local model to draft a template you will read first.
```
I have written authorization to test https://in-scope.example. I want a Nuclei template that checks for an exposed .env or .git/HEAD on that host only. Explain the requests and the matchers in plain language first. I will review the template and run it myself. Do not widen the scan to other hosts. Do not add destructive requests.
```

### Explain a captured request in Burp
For Burp AT, BurpGPT or a local model. Strip cookies and tokens first
if the model is hosted in the cloud.
```
Here is one HTTP request and response from an in-scope app I am authorized to test. Explain what the endpoint appears to do, which parameters look sensitive, and whether anything suggests an access-control or IDOR lead I should verify by hand. Stay inside this request. Do not invent other endpoints. Do not give me a ready-made exploit.

[paste redacted request and response]
```
**The researcher note:** he does not use Burp. The same prompt works on a
curl-captured request/response pair, or on a playwright-cli snapshot.

### Turn verified notes into a report draft
Only after you have already reproduced the bug. See the report section.
```
Turn my verified notes into a bug bounty report. Use only what I wrote. Do not add reproduction steps, hosts, or impact I did not record. Do not invent a CVSS number. Write: a short title, a summary, the affected URL, step-by-step reproduction copied from my notes, impact in plain language, and a practical fix. Flag any place where my notes are too thin to support a claim.

Notes:

[paste your verified notes]
```

### Keep an agent on a leash
If you start PentestGPT, Strix or a similar agent in a lab or staging app.
```
You are a junior tester. You may only touch this allowlist: [hosts]. This is a lab / staging app I am authorized to test. Before each command, state the command and wait if I have asked you to approve actions. Do not scan other hosts, do not use destructive payloads, and do not exfiltrate data. After you stop, give me a log of every command you ran and any lead I should verify myself. If you are unsure whether something is in scope, stop and ask.
```

## Writing the report with AI, without inventing it

Programs pay for clear, reproducible bugs, not for a confident
paragraph. Use a model as a typist and an editor. You remain the
researcher of record.

### What belongs in the report
- A short, specific title (what is wrong, where, and why it matters).
- The exact in-scope URL or endpoint, and the account role you used, if any.
- Step-by-step reproduction that you have already walked through successfully.
- Evidence: request, response, screenshot, or a short video, with secrets redacted.
- Impact in plain language, tied to this program, not a generic worst case.
- A practical fix the team can actually ship.

### How to use the model
- Paste verified notes only. If a step is missing, the model must say so, not fill it in.
- Ask it to tighten language, not to raise severity. You choose the severity after you understand impact.
- Do not let it invent CVSS, CWE IDs, or "likely also vulnerable" endpoints.
- Match the platform's format if the program uses one (they often say so on the brief).
- Read the draft out loud against your evidence before you click submit.

### What not to file
- A scanner hit you have not reproduced, including Nuclei, Burp Scanner, Dalfox or an agent "proof."
- A model monologue with no request and response.
- Anything on a host that is out of scope, even if a tool or a model suggested it.
- Destructive or out-of-band activity the policy forbids, described as if it were fine.
- Customer data, live session tokens, or other people's accounts in your attachments.

### A simple drafting loop
1. Reproduce once more and save the raw evidence. 2. Write messy notes
in your own words. 3. Use the report prompt above to get a clean draft.
4. Delete any sentence you cannot back up. 5. Submit. If the triage team
asks a question, answer from evidence, not from a fresh chat that might
improvise.

### The researcher's gates on top (non-negotiable)
- **Validation gate:** every candidate gets Claude Opus + gpt-6-luna
  validation AND an execution test + recheck before it is filed.
  Validation alone is never final.
- **Evidence gate:** a validated finding is NOT submittable without (1)
  exact reproduction steps, (2) evidence ATTACHED (screenshot/video +
  page/request log), (3) PoC/test page persisted and reachable. Never
  submit a text-only report. (This is the lesson of the Sophos #13
  rejection — Klaus closed it as Not Applicable for exactly this.)
- **Humanize:** no em dashes, no bold-label lists, no triads, no staged
  openers, sentence-case headings. Combine valid bugs into one report
  unless the program requires separate reports.
- **Approval:** no filing without the researcher's explicit per-finding approval.

## Safety principles

**Scope first**
Only point a tool at systems you own or have written permission to test.
Stay inside the program allowlist. Labs, CTFs, staging copies and your
own apps are the place to learn.

**Keep agents on a leash**
Fuzzers, XSS scanners and AI agents send traffic and can change data.
Shannon and similar tools say not to run them against production.
Sandbox them.

**Verify by hand**
Scanners and models find candidates, not facts. Reproduce every finding
yourself before it goes in a report. Do not file a draft you have not
walked through.

## Related skills
- `bb-methodology` — the 16-step recon pipeline this workflow's Levels
  1–2 feed into.
- `bb-recon-pipeline` — response-dump recon + per-area test checklists.
- `bb-workflow-0xkonde` — the full hunt workflow this AI layer plugs
  into.
- `bug_bounty_hunt` — vulnerability-class testing + the validation gate
  before reporting.
- `gate_validasi_hipotesis` — the anti-premature-rejection validation
  gate (CONFIRMED / FALSIFIED / INCONCLUSIVE).
- `reconkit` — scope-gated recon modules with canary validation.
