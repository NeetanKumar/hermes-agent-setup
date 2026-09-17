# Hermes Agent Setup

A record of my [Hermes Agent](https://hermes-agent.nousresearch.com/) install
and configuration — the settings, safety posture, and notes, **not** the
Hermes Agent source code itself (that lives upstream at
[NousResearch/hermes-agent](https://github.com/NousResearch/hermes-agent) and
is installed separately on my machine, not tracked here).

## What's in this repo

- **`INSTALL_NOTES.md`** — full log of the install: commands run, versions,
  file locations, issues hit along the way, and what's still open.
- **`hermes-config.yaml`** — a copy of my non-secret `~/.hermes/config.yaml`.
  Contains model/provider choice, approval mode, and the Docker sandbox
  settings. **No API keys or secrets** — those live in `~/.hermes/.env`,
  which is never copied here (see `.gitignore`).

## Safety posture

- Command approval: `smart` mode (auto-approves low-risk commands, prompts
  for anything risky), with explicit deny rules for force-pushes and raw
  disk writes.
- Terminal sandbox: Docker (`nikolaik/python-nodejs:python3.11-nodejs20`),
  with only one folder on the host mounted in, no host environment
  variables forwarded.
- Agent-authored skills require human review before they persist
  (`skills.write_approval: true`).

See `INSTALL_NOTES.md` for the full picture.
