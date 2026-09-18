# Job Search Automation — Naukri Attempt and Pivot

A record of trying to build a daily auto-apply job on Naukri.com through
Hermes's real-profile browser feature, why it didn't work, and where it's
headed instead.

## The ask

Automate job applications with these criteria:

- Roles: AI Engineer, Backend Engineer, Backend Developer
- Location: remote preferred, otherwise any
- Experience: 0-4 years
- Exclude: ML Engineer roles
- Salary floor: skip listings under 23 LPA (no salary listed is still fair
  game)
- Volume: up to 30 applications/day, once a day
- Full autonomy, but a report after every run and an easy stop switch
- Login via Hermes's `browser.use_real_profile` (reuse the already
  logged-in Chrome session — no password stored)

## What got built

**1. Enabled `browser.use_real_profile`.**
`hermes config set browser.use_real_profile true` — this alone wasn't
blocked by anything.

**2. Copied the real Chrome profile into Hermes's sandbox.**
Detecting *which* browser to snapshot depends on the macOS default-browser
setting (`defaults read com.apple.LaunchServices/... LSHandlers`), which
wasn't set to any Chromium browser on this machine — it failed closed with
`default browser: None`. Fix: temporarily set Chrome as the macOS default
(System Settings → Desktop & Dock → Default web browser), fully quit
Chrome (Cmd+Q — a running browser holds a lock on the cookie DB), then
snapshot it directly:

```python
from hermes_cli.browser_connect import snapshot_real_profile
dst, err = snapshot_real_profile('chrome')
# dst: /Users/neetan.kumar/.hermes/browser-profile/chrome, err: None
```

This copies `Local State`, the active profile's cookies/login DBs, etc.
into a Hermes-owned, owner-only-permissioned directory. The real Chrome
profile is never touched or locked afterward — the agent drives the copy,
not the original.

Note: calling `hermes -z "..." --yolo -t browser` to test-drive this got
denied by Claude Code's auto-mode classifier under **"Create Unsafe
Agents"** (spinning up a fully autonomous, auto-approving agent). Rather
than work around that, the snapshot function was called directly — it's a
pure file-copy operation, not an agent, so no classifier conflict.

**3. Built the cron job.**
`hermes cron create` with a schedule of `0 10 * * *`, `--deliver email`
(routes to the address in `~/.hermes/channel_directory.json`), and a
prompt encoding all the criteria above. Dedup tracking lives in a plain
JSON file rather than the built-in cron notepad, because the notepad's
write path is `hermes cron notepad <id> set ...` run *through the
terminal tool*, and the terminal tool executes inside the Docker sandbox —
not guaranteed to have the host `hermes` CLI or `HERMES_HOME` on its PATH.
A file under the already-mounted workdir sidesteps that entirely:

- `naukri-autoapply/state.json` — `{"mode": "dry_run" | "live"}`, checked
  every run before anything is clicked
- `naukri-autoapply/applied_jobs.json` — array of `{job_id, title,
  company, applied_at}`, read-modify-written after every successful apply
  so nothing gets double-applied

First run was forced into `dry_run` mode on purpose (list matches, don't
click Apply) so the criteria could be sanity-checked against real listings
before trusting it to submit anything.

**4. Stop mechanism.**
No custom kill-switch needed — Hermes already ships `hermes cron pause
<job_id>` / `hermes cron resume <job_id>` per-job, plus a global `hermes
pause` emergency stop. Used the per-job one throughout.

## Where it broke

The very first triggered run (`hermes cron run 65ec7222b6f1`) hit a wall
immediately: Naukri's homepage and the job-search URL both returned an
**Akamai Edgesuite "Access Denied"** — a WAF block, not a login page and
not a CAPTCHA. No search was ever performed.

This matched the stop condition written into the job's own prompt
("if you hit anything that looks like automated-access blocking, stop
immediately, don't retry aggressively, report exactly what you saw") —
the agent did exactly that, correctly, and left `state.json` /
`applied_jobs.json` untouched.

Diagnosis: this isn't an IP-reputation or proxy issue — `hermes egress
status` confirmed no proxy is active, `browser.backend` is empty (plain
local Chrome), so the request left from the same residential IP as any
normal manual browsing on this machine. The block is almost certainly
Naukri's Akamai Bot Manager fingerprinting the CDP-driven Chrome session
itself (e.g. `navigator.webdriver` and related automation signals that
CDP exposes even on a real, cookie-authenticated profile).

**Decision: don't chase this.** Getting past an Akamai Bot Manager
signature means stealth-patching the browser to hide automation signals —
i.e., building tooling specifically to defeat a site's anti-bot security
control. That's a different thing from "automating my own logged-in
session," and it's not a line worth crossing for a job-search convenience
feature. The job was paused (`hermes cron pause 65ec7222b6f1`) rather than
retried with evasion techniques.

## Current state

- Cron job `naukri-autoapply` (`65ec7222b6f1`) exists but is **paused**.
- Real-profile Chrome snapshot is in place and reusable for whichever
  site comes next.
- Decision in progress: drop Naukri, broaden to other job sources.
  AngelList/Wellfound confirmed as one target so far; still deciding
  between "shortlist-only everywhere" (Hermes finds and filters, human
  clicks Apply — zero ToS/bot-defense risk, works uniformly) vs.
  "auto-apply where a site technically tolerates it, shortlist elsewhere."
  LinkedIn is expected to have similar bot-detection to Naukri if
  auto-apply is attempted there.

## Lessons for next time

- Any site with real anti-bot infrastructure (Akamai, PerimeterX,
  Cloudflare Bot Management) should be assumed to block CDP automation
  outright, regardless of a valid logged-in cookie — test with a single
  dry-run request before designing a whole pipeline around a site.
- Real-profile browsing needs the target Chromium browser to be the
  macOS default at snapshot time; it can be switched back immediately
  after.
- Prefer plain files in the already-mounted workdir over the cron
  notepad for state that the *agent* (not just the operator) needs to
  read/write mid-run, since the notepad's write path assumes host CLI
  access that a Docker-sandboxed terminal tool may not have.
