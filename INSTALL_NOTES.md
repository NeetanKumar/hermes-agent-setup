# Hermes Agent (Nous Research) — Install Notes

Date: 2026-09-17
Machine: macOS 14.8.5, Apple Silicon (arm64), zsh

## Where things actually live

This project folder (`~/Desktop/Hermes-Nous`) is **not** where Hermes was
installed — it's just the sandbox workspace mounted into the Docker
terminal backend (see below) and the home for these notes.

| What | Path |
|---|---|
| Hermes source code | `~/.hermes/hermes-agent/` (git, commit `98f758ae7e8db83c2bb9214c3b35adf41df15f03`, branch `main`) |
| `hermes` binary | `~/.local/bin/hermes` (+ `hermes-agent`, `hermes-acp` launchers) |
| Config (non-secret) | `~/.hermes/config.yaml` |
| Secrets (API keys) | `~/.hermes/.env` (mode 600) |
| Memory | `~/.hermes/memories/` |
| Skills (58 bundled) | `~/.hermes/skills/` |
| Logs | `~/.hermes/logs/` |
| Cron jobs | `~/.hermes/cron/` |
| Sessions | `~/.hermes/sessions/` + `~/.hermes/state.db` |
| Docker sandbox workspace | `~/Desktop/Hermes-Nous` → `/workspace` in-container |

## Versions

- Hermes Agent v0.21.3 (2026.9.14), upstream commit `98f758ae`
- Python 3.12.13 (system Python, reused — installer's requested 3.11 was not needed since 3.12 satisfies `>=3.11,<3.14`)
- uv 0.12.15
- Node.js v26.5.1
- ripgrep 15.2.0
- ffmpeg 9.0.1
- Docker 29.6.1
- Sandbox image: `nikolaik/python-nodejs:python3.11-nodejs20` (2.21 GB, pulled)

## Commands run (chronological)

```bash
# Phase 0/1 — read-only research and preflight (see chat history for doc URLs read)
uname -a; sw_vers; df -h ~
ls ~/.hermes ~/.openclaw ~/dev/hermes-agent   # both absent, no migration needed

# Downloaded installer to scratchpad (not piped to bash), reviewed it in full
curl -fsSL -o install.sh https://hermes-agent.nousresearch.com/install.sh
shasum -a 256 install.sh   # 2de7a1d60a7c0edf685d3b7b875b93a99c8006473a55218688a0075a88637045

# Phase 2 — install (approved: --skip-setup only, full install incl. Chromium
# lazy-hook + Computer Use driver; installer allowed to edit ~/.zshrc)
bash install.sh --skip-setup
#   1st attempt: failed — transient DNS/Homebrew-lock issue mid-run
#     (github.com / ghcr.io unresolvable for ~18 min while brew waited on a
#     stale download lock from an unrelated process). No partial state left.
#   2nd attempt: succeeded. ripgrep+ffmpeg brew build got stuck on slow
#     sequential GNU readline patch checks; killed just that `brew install`
#     step (PID of `ruby .../brew.rb install ripgrep ffmpeg`) so install.sh's
#     own non-fatal fallback kicked in and the clone/venv/etc. continued.
#     ripgrep+ffmpeg were then rebuilt in a fully detached background job
#     (`brew install ripgrep ffmpeg`) that finished on its own — both now
#     installed via Homebrew bottles, no manual fallback needed.

# Manual step the installer told us to run (browser tools are lazy-installed
# on macOS, not part of the base install):
cd ~/.hermes/hermes-agent && npx playwright install chromium
# Not yet completed — playwright isn't a direct npm dependency (lazy_deps.py
# policy). Will trigger the first time the browser tool is actually enabled
# and used; see "Known open issues" below.

# Phase 3 — provider setup (user's own step, keys never seen by the assistant)
hermes model     # → provider: anthropic, model: claude-sonnet-5

# Phase 5 — safety defaults
cp ~/.hermes/config.yaml ~/.hermes/config.yaml.bak.<timestamp>
hermes config migrate                              # config v0 → v45
hermes config set terminal.backend docker
hermes config set terminal.docker_image "nikolaik/python-nodejs:python3.11-nodejs20"
hermes config set terminal.cwd "/workspace"
hermes config set skills.write_approval true
docker pull nikolaik/python-nodejs:python3.11-nodejs20
# + direct YAML edits (list values `hermes config set` doesn't take):
#   terminal.docker_forward_env: []
#   terminal.docker_volumes: ["/Users/neetan.kumar/Desktop/Hermes-Nous:/workspace"]
#   terminal.container_cpu: 2
#   approvals: { mode: smart, timeout: 300, cron_mode: deny,
#                single_query_mode: deny, unattended_mode: deny,
#                deny: ["git push --force*", "dd if=* of=/dev/*"] }

# Phase 4 — verification
hermes doctor
hermes status
hermes -z "Use the terminal tool to run 'uname -a' and tell me exactly what it printed."
#   → returned a Linux/aarch64 kernel string, confirming the command ran
#     inside the Docker container, not on the host.
hermes -z "Use the terminal tool to run 'pwd && ls -la / && ls /workspace' ..."
#   → cwd was /workspace; container root looks like a normal container
#     filesystem; /workspace correctly mirrors ~/Desktop/Hermes-Nous
#     (only a hidden .claude/ dir there, so ls without -a showed nothing).
```

## Current configuration summary

- **Provider/model:** Anthropic, `claude-sonnet-5` (set via `hermes model`).
- **Approvals:** `smart` mode (auxiliary-LLM risk assessment auto-approves
  low-risk commands, prompts for the rest). `cron_mode`, `single_query_mode`,
  and `unattended_mode` all `deny` (headless runs default to declining
  anything needing approval rather than hanging or auto-running).
  Explicit deny rules block force-pushes and raw `dd` writes regardless of
  mode.
- **Terminal backend:** Docker. Image `nikolaik/python-nodejs:python3.11-nodejs20`,
  2 CPUs, 5 GB RAM, 50 GB disk, persistent container. Only
  `~/Desktop/Hermes-Nous` is mounted (as `/workspace`); `docker_forward_env`
  is empty, so no host environment variables leak into the container.
  **Caveat:** only the `terminal` tool is sandboxed this way. Browser and
  Computer Use tools run on the host, not inside the container.
- **Skills:** `write_approval: true` — any skill the agent authors for
  itself is staged under `~/.hermes/pending/skills/` and needs your review
  before it takes effect.
- **Toolsets enabled (CLI):** web, browser, terminal, file, code_execution,
  vision, image_gen, tts, skills, todo, memory, session_search,
  connections, clarify, delegation, cronjob, computer_use. Disabled:
  video, video_gen, x_search, stt, kanban, context_engine, homeassistant,
  spotify, yuanbao, a2a.
- **Messaging gateway:** not configured. No Telegram/Discord/Slack/email
  set up in this pass (deferred by user decision).

## Known open issues / things I deliberately left alone

1. **Doc vs CLI discrepancy (resolved by trusting `--help`):** both
   `hermes -z "prompt"` (top-level, true one-shot) and
   `hermes chat -q "..." --oneshot` exist and work. The two doc pages each
   only mentioned one of them.
2. **Browser tools — resolved.** `npx playwright install chromium` was the
   wrong path (that's a legacy fallback; no `playwright` entry in
   `package.json`, which is why it warned about missing project deps).
   The default browser backend is Hermes's own "browser-use" managed CLI,
   which already bundled its own Chromium during the base install. Verified
   working via the correct, officially supported hook:
   `hermes tools post-setup agent_browser` → "Chromium browser already
   installed, nothing to do". Confirmed end-to-end with a real navigation:
   `hermes -z "Use the browser tool to navigate to https://example.com and
   tell me the exact page title."` → returned the live page title
   correctly. `hermes doctor` now shows `agent-browser` and `Playwright
   Chromium (browser engine)` both green.
3. **`hermes doctor` non-blocking warnings:**
   - npm audit: 2 vulnerabilities in the browser-tools workspace, 6 in the
     web workspace (Playwright/Node ecosystem transitive deps).
   - AWS Bedrock `AccessDeniedException` for IAM user
     `contract-ops-developer` — this is **not** something we configured.
     Hermes's connectivity probe found pre-existing AWS credentials already
     on this Mac (unrelated to this project) and tried them against
     Bedrock. Harmless since the active provider is explicitly `anthropic`,
     not `auto`/Bedrock, but worth knowing those credentials are ambient on
     this machine.
   - `image_gen` unavailable — expected, Anthropic doesn't do image
     generation; only relevant if you later add an image-gen-capable key.
4. **Unrelated pre-existing project found:** `~/Desktop/Hermes-agent`
   (no "Nous" in the name) is your own separate repo
   (`github.com/NeetanKumar/Hermes-agent`), with its own venv, `.env`, and
   a process that was running during this session. It is completely
   separate from `~/.hermes/hermes-agent` (the official Nous Research
   install) and was not touched.
5. **Cron/reminders + email gateway: deferred.** You decided to bring the
   gateway back in scope for email send/receive and reminders, but it
   needs a dedicated mailbox (not your personal one) with an app password
   before `hermes gateway setup` can be run. Not started — see "Next steps".
6. **Config backup:** pre-migration config saved at
   `~/.hermes/config.yaml.bak.<timestamp>` in case anything needs reverting.

## Next steps (your call)

- To finish browser tools: enable/use the `browser` tool once, or run
  `hermes tools` → Browser → its post-setup hook, then re-run
  `npx playwright install chromium` from `~/.hermes/hermes-agent`.
- To get email + reminders: create a dedicated mailbox with an app
  password, then run `hermes gateway setup` yourself (credentials never
  seen by the assistant), then `hermes gateway install` to run it as a
  launchd user service (`~/Library/LaunchAgents/ai.hermes.gateway.plist`,
  no sudo).
- Optional cleanup: `brew install --overwrite` is not needed; ripgrep and
  ffmpeg installed cleanly via the detached background build.
