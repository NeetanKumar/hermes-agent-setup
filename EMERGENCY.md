# Emergency Stop Guide

How to stop Hermes fast. Job IDs at time of writing: `job-autoapply`
(`9d00ac003645`), `github-contributions-6h` (`c30e35113061`).

## Quick reference

| Goal | Command | Undo |
|---|---|---|
| Halt all cron dispatch and new gateway turns (gateway and email stay up) | `hermes pause` | `hermes resume` |
| Stop the whole gateway service (cron + email) | `hermes gateway stop` | `hermes gateway start` |
| Stop one job | `hermes cron pause <job_id>` | `hermes cron resume <job_id>` |
| Remove the service so it stays off after reboot/re-login | `hermes gateway uninstall` | `hermes gateway install` |

## Details

- **`hermes pause`** is the built-in emergency stop. It halts cron dispatch,
  kanban dispatch and new gateway turns. Runs already in flight are not killed.
- **`hermes gateway stop`** stops the service. The gateway is a launchd service
  (`ai.hermes.gateway`) that starts at login, so it comes back after a reboot or
  re-login unless you run `hermes gateway uninstall`.
- **`hermes cron pause <id>`** pauses a single job. Use `hermes cron list` to see IDs.

## Caveats

- Stopping does not undo anything a run already did. Applications already
  submitted to a site cannot be recalled.
- In-flight runs are cut off, not finished. The plist allows a 25-second graceful
  drain, then launchd force-kills. Leftover browser processes or the Docker
  container may need a manual check afterwards.
- `hermes gateway stop` also takes down the email gateway (send and receive via
  trialbasis44@gmail.com), so no reports or replies while it is stopped.
- Killing the gateway process by hand (`kill`) does not hold: launchd has
  `KeepAlive` and respawns it within about 30 seconds.
- Killing the Docker container does NOT stop anything. It is only the terminal
  tool's sandbox; the scheduler runs in the gateway process on the host, and a
  new container is created on the next terminal call.
- `cron.catch_up_missed` was set to `false` so jobs missed while stopped do not
  all fire on restart. Check with `hermes config get cron.catch_up_missed`;
  the gateway needs a restart to pick up a change.
- `hermes gateway stop` staying stopped (no respawn) is inferred from the plist
  (`SuccessfulExit=false`), not tested. Verify with `launchctl list | grep hermes`.

## What uninstall removes, and how to bring it back

`hermes gateway uninstall` only unloads the launchd job and deletes
`~/Library/LaunchAgents/ai.hermes.gateway.plist` (read from
`launchd_uninstall` in `hermes_cli/gateway.py`). It does not touch:

- Cron jobs, their schedules, prompts and paused state
- `~/.hermes/config.yaml` (including `cron.catch_up_missed`)
- Email credentials in `~/.hermes/.env`
- The Chrome profile snapshot, sessions, logs and state DBs
- This repo and the `job-autoapply/` / `naukri-autoapply/` state files

Reinstalling is about one command:

1. `hermes gateway install` regenerates the plist and loads the service
   (`hermes gateway start` also regenerates a missing plist).
2. `hermes cron resume <job_id>` for each job you actually want running.

Only deleting `~/.hermes` itself would mean real re-setup. Running commands in
this repo does not start anything: it has no `package.json`, and only the
launchd service starts the gateway and scheduler.

## Recommended emergency sequence

1. `hermes pause`
2. `hermes gateway stop`
3. Check nothing is left: `ps aux | grep -i hermes`, `docker ps --filter name=hermes`,
   `launchctl list | grep hermes`
4. To resume: `hermes gateway start`, then `hermes resume`.
