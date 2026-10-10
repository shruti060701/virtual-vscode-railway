# Deploy and Host code-server (VS Code in the Browser) on Railway

[code-server](https://github.com/coder/code-server) runs full VS Code in a container and serves it over HTTP — open it from a tablet, a Chromebook, or any machine with a browser, and you're in a real development environment, not a toy. This template deploys a fixed fork of the `deploy-code-server` project that Railway's own official `virtual-vscode` template is built on, and fixes three real issues found by actually reading its Dockerfile, entrypoint script, and running it live.

## About Hosting code-server

This template builds from `shruti060701/virtual-vscode`, a fork of `coder/deploy-code-server` (MIT), which wraps the official `coder/code-server` (MIT) image with a Railway-friendly entrypoint: password auth from an environment variable, and an optional `GIT_REPO` to auto-clone into your workspace on first boot. The fork changes two things from upstream: the pinned code-server version, and how the workspace directory is initialized so it actually survives a redeploy on Railway's volumes.

The version pin matters more than it sounds. Railway's reference template builds from a Dockerfile that still reads `FROM codercom/code-server:4.9.0` — a release from 2022/2023 — despite the template's own listing being labeled "[Updated Oct '26]". That's roughly 130 releases of bug fixes, security patches, and features sitting unapplied. This fork pins `4.141.0`, the latest stable release as of this writing.

## Common Use Cases

- **A portable dev environment**: code from any device, pick up exactly where you left off, no local setup
- **Pairing and teaching**: share a password-protected URL instead of walking someone through a local install
- **Working around a weak local machine**: run the actual compute on Railway, keep a Chromebook or tablet as the terminal
- **A disposable sandbox**: spin up a clean environment per project or experiment, point `GIT_REPO` at the repo, and start immediately

## Dependencies for code-server Hosting

Running this template needs nothing beyond the container and one persistent volume — no external database, no separate services. You bring the repo you want to work on (if any) via `GIT_REPO`, and a password.

### Deployment Dependencies

- [code-server's Source Code](https://github.com/coder/code-server) — MIT
- [code-server's Official Documentation](https://coder.com/docs/code-server/latest)
- [This Template's Fork](https://github.com/shruti060701/virtual-vscode) — MIT, forked from [`coder/deploy-code-server`](https://github.com/coder/deploy-code-server)

### Implementation Details

Three issues surfaced in this template, each one found by actually deploying and reading the resulting logs rather than trusting the reference's configuration or the upstream project's assumptions.

**No real persistence.** The upstream project predates widespread container-platform volume support and says so directly in its own README: storage "may not be persistent," and the documented fix is configuring an [rclone](https://rclone.org/) remote by hand — generating a config, base64-encoding it, pasting it into an environment variable. Railway's reference template doesn't attach a volume either, so a redeploy silently resets your workspace to whatever `GIT_REPO` last pointed at. This template mounts a Railway volume at `/home/coder/project` instead — no rclone, no manual config, it just persists.

Making that volume actually work took two more fixes, both found live:

1. A freshly-mounted Railway volume is ext4, and ext4 ships its own `lost+found` directory. The entrypoint's "is the workspace already initialized" check used a plain `ls -A`, which saw `lost+found` as content and skipped `GIT_REPO` cloning on literally the first boot, before anything had ever been written. Excluding `lost+found` from the check fixed the detection — but then:
2. `git clone` independently refuses to clone into any directory that isn't completely empty, and `lost+found` alone is enough to trip that too. The fix clones into a scratch directory first and moves the result into place, sidestepping git's requirement entirely. Testing that revealed a third, more serious issue: the volume mount comes back **owned by root**, while code-server itself runs as the unprivileged `coder` user — so every file copy into the workspace failed with `Permission denied`, and the original script didn't even notice, because it only checked `git clone`'s exit code, not the copy step's. It logged "Cloned successfully" while silently writing nothing. The entrypoint now `chown`s the volume to the `coder` user on boot, and the success log line depends on every step actually succeeding.

**No healthcheck.** code-server ships a real, built-in, unauthenticated status endpoint at `/healthz` (`src/node/routes/health.ts` in its own source), registered before any auth middleware. The obvious alternative, `/`, redirects unauthenticated visitors to `/login` with a `302` — and Railway's healthcheck prober doesn't follow redirects, so pointing it at `/` would fail every single deploy regardless of whether the app is healthy. This template points the healthcheck at `/healthz` instead.

## Why Deploy code-server on Railway?

Railway removes everything around the container that code-server itself doesn't handle: TLS, a public domain, and — once configured correctly, which took the three fixes above — storage that survives a restart without you thinking about it. For a tool whose entire purpose is "give me a real dev environment I can reach from anywhere," that's the whole value proposition; losing your workspace on every redeploy defeats the point.

This template specifically closes the gap between "the Composer accepted my config" and "my files are actually still here tomorrow." Every fix here was driven by a real failure surfaced during deployment, not a guess about what might go wrong.

## What Was Verified

Every claim above was checked against a live deployment, not inferred from reading the code alone. `GET /healthz` returns `200 {"status":"alive"/"expired","lastHeartbeat":...}` with no authentication. `GET /` returns a `302` to `/login`, confirming why it's the wrong healthcheck target. `GET /login` returns a real `200` code-server login page. Deploy logs confirm `code-server 4.141.0` is what's actually running.

The persistence fix took three iterations to get right, and each one was caught by actually watching the deploy logs rather than assuming the previous fix had worked: the `lost+found` detection bug, `git clone`'s independent empty-directory requirement, and the root-owned volume mount all showed up as concrete log lines (`already has content, skipping init` on an empty volume; `fatal: destination path ... already exists`; `Permission denied`) before being fixed and re-verified.

## Frequently Asked Questions

### Why isn't `/` used as the healthcheck path?
Because code-server redirects unauthenticated requests to `/login` with a `302`, and Railway's healthcheck doesn't follow redirects — pointing it at `/` marks every deploy unhealthy regardless of the app's actual state. `/healthz` is code-server's own dedicated, unauthenticated status endpoint.

### Will my files survive a redeploy?
Yes — the workspace lives on a persistent Railway volume at `/home/coder/project`, and the entrypoint only initializes it once; on every subsequent boot, if the volume already has content, nothing touches it.

### Does this template modify code-server itself?
No. It only changes the surrounding fork — `coder/deploy-code-server`'s Dockerfile and entrypoint script — not code-server's own source. The version pin is bumped to the latest stable release, and the entrypoint is patched to actually work with a mounted volume.

### What if I don't set `GIT_REPO`?
The workspace opens with a single placeholder file instead of a cloned repo. You can `git clone` your own repo manually from the integrated terminal once you're in, or set `GIT_REPO` and redeploy.

### Can I still add custom tools, like the upstream project supports?
Yes — the fork's `Dockerfile` has the same extension points as upstream (install extensions, apt packages, copy files). Edit it, push to your own fork, and point this template's source at your fork.

### Where can I find code-server?
Source is on GitHub at [github.com/coder/code-server](https://github.com/coder/code-server). Use this template to deploy a fixed fork with real persistence and a working healthcheck, in one click.

Source for this template's docs: https://github.com/shruti060701/virtual-vscode-railway
