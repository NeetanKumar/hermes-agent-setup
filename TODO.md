# Open Follow-ups

## Email attachments may fail to deliver

**Status:** Not fixed. Text-only email works fine; this only affects files.

**What's wrong:** The gateway logged this warning on restart:

> Docker backend is enabled for the messaging gateway but no explicit
> host-visible output mount (for example
> `/home/user/.hermes/cache/documents:/output`) is configured. This is
> fine if the model already emits host-visible paths, but MEDIA file
> delivery can fail for container-local paths like `/workspace/...` or
> `/output/...`.

**Why it happens:** The terminal tool runs inside the Docker sandbox
(`terminal.backend: docker`). If the agent generates a file — an image,
a document, an exported report — and tries to email it to you, that file
physically exists only inside the container's filesystem. The email
adapter runs on the host and can't reach a path like `/workspace/report.pdf`
or `/output/chart.png`, because those paths don't exist outside the
container.

**Fix:** Add an explicit output volume mount in `~/.hermes/config.yaml`
so a specific host folder is shared with the container at a known path,
and point the agent's file-output convention at that path:

```yaml
terminal:
  docker_volumes:
    - "/Users/neetan.kumar/Desktop/Hermes-Nous:/workspace"
    - "/Users/neetan.kumar/.hermes/cache/documents:/output"   # add this
```

Then `mkdir -p ~/.hermes/cache/documents` on the host so the mount target
exists, and restart the gateway (`hermes gateway restart`) for it to
pick up the change.

**When to do this:** Only matters once you actually ask the agent to
send you a file over email or another platform. Not urgent for
text-only use.
