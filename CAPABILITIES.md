# Extra Hermes Agent Capabilities (not yet used)

Everything below is available in this install but not yet turned on or
tried. None of it required setup we skipped — it's either built in
already, or a one-line command away. Grouped by what it's for.

## Already confirmed working on this install

For reference — see `INSTALL_NOTES.md` for full detail:
terminal (Docker-sandboxed), browser, file, code execution, vision,
image generation, memory, session search, todo, cron/reminders,
computer use, skills, TTS, email send/receive.

## Working smarter within a session

- **Persistent Goals** — `/goal <task>` in a chat session. Gives the
  agent a standing objective it keeps working toward across turns on
  its own, checking its own progress after each step, until done, paused,
  or a 20-turn safety cap. Already explained in earlier chat; nothing to
  install. Docs: `/docs/user-guide/features/goals`
- **Recurring Loops** — like cron, but for repeating a task on a
  schedule *within* a live session rather than as a background job.
  Docs: `/docs/user-guide/features/loops`
- **Hooks** — shell scripts that run automatically at points in the
  agent's lifecycle (e.g. before a risky command). Docs:
  `/docs/user-guide/features/hooks`
- **Delegation / subagents** — the `delegate_task` tool is already
  enabled; it lets the agent spin off a subagent for a piece of work
  instead of doing everything in one thread.
- **Kanban multi-agent** — `hermes kanban`. A task board for running
  several agent workers on related tasks in parallel, with worker lanes
  and multi-gateway deployment for bigger setups. Currently disabled in
  `hermes tools list`. Docs: `/docs/user-guide/features/kanban`

## Connecting external tools and services

- **MCP servers** — `hermes mcp` (interactive catalog), or
  `hermes mcp install <name>` directly. Nous ships a curated,
  security-reviewed catalog: GitHub, Linear, Stripe, Figma, Playwright,
  n8n, and more. Config lives under `mcp_servers:` in `config.yaml`.
  Docs: `/docs/user-guide/features/mcp`
- **Skills Hub** — `hermes skills browse` / `hermes skills search <query>`
  / `hermes skills install <id>`. 58 bundled skills are already
  installed (`~/.hermes/skills/`); this pulls more from GitHub, ClawHub,
  skills.sh, and other sources, each security-scanned on install.
- **Home Assistant** — smart-home control, `homeassistant` toolset.
  Currently disabled; needs a Home Assistant instance to point at.
- **Webhooks** — `hermes webhook subscribe`. Lets external services
  trigger the agent (e.g. a GitHub push, a form submission) instead of
  the agent only acting on a schedule or a message.
- **A2A (Agent-to-Agent)** — lets this Hermes talk to other
  agent instances directly. Plugin toolset, currently disabled.

## More messaging platforms

Email is the only one connected so far. The same `hermes gateway setup`
pattern (or manual `hermes config set` for env-style keys, since we
found the interactive wizard doesn't prompt for every platform) applies
to: Telegram, Discord, Slack, WhatsApp, Signal, SMS, Matrix, Mattermost,
Microsoft Teams, Google Chat, and about a dozen regional platforms
(WeCom, Feishu/Lark, LINE, DingTalk, QQ, Yuanbao). Telegram is the
simplest to add next — just a bot token from @BotFather and your
Telegram user ID, no dedicated-account concerns like email had.

## Model / cost / reliability tuning

- **Mixture of Agents** — `hermes moa`. Runs a task across multiple
  models/providers and combines their answers, rather than relying on
  one model.
- **Fallback Providers** — `hermes fallback`. Automatic failover to a
  backup provider if the primary one is down or rate-limited. Also
  configurable via `fallback_model:` in `config.yaml` (commented out by
  default — see the bottom of `hermes-config.yaml` in this repo).
- **Credential Pools** — round-robin or load-balance across multiple
  API keys for the same provider, useful if you hit rate limits often.

## Personality and self-maintenance

- **SOUL.md** — `~/.hermes/SOUL.md`. A file you can edit to give the
  agent a defined identity/personality that persists across sessions.
- **Context Files** — AGENTS.md / CLAUDE.md / .cursorrules style files
  that get auto-injected as project-specific instructions when the agent
  works in a given folder.
- **Curator** — `hermes curator`. Background process that reviews and
  prunes the agent's own memories and self-authored skills over time so
  they don't accumulate cruft. Config under `curator:` in `config.yaml`
  (added by the v0→v45 migration, not yet configured).

## Not relevant to this install

- **Voice Mode** and premium TTS voices (ElevenLabs) — `tts` toolset is
  already enabled with the built-in voice; ElevenLabs needs its own key.
- **X (Twitter) Search** — needs `XAI_API_KEY`, not configured.
- **Spotify** — needs its own OAuth setup, not configured.

---

None of this needs to happen — the agent is fully functional without
any of it. This file exists so you know what's on the table if you want
to expand what it can do later.
