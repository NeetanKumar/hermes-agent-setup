# Emergency Stop Guide

Jobs: `job-autoapply` (`9d00ac003645`), `github-contributions-6h` (`c30e35113061`).

## Quick reference

| Goal | Command | Undo |
|---|---|---|
| Halt all cron dispatch and new gateway turns (gateway and email stay up) | `hermes pause` | `hermes resume` |
| Stop the whole gateway service (cron + email) | `hermes gateway stop` | `hermes gateway start` |
| Stop one job | `hermes cron pause <job_id>` | `hermes cron resume <job_id>` |
| Remove the service so it stays off after reboot/re-login | `hermes gateway uninstall` | `hermes gateway install` |

## Notes

- Stopping doesn't kill runs in flight cleanly or undo anything already submitted.
- Killing the Docker container stops nothing; the scheduler runs in the gateway on the host.
- Uninstall only removes the launchd plist. Jobs, config and credentials in `~/.hermes` stay.
- After a reinstall, resume only the jobs you want with `hermes cron resume <job_id>`.
- Emergency sequence: `hermes pause`, then `hermes gateway stop`, then check `ps aux | grep -i hermes` and `docker ps`.
