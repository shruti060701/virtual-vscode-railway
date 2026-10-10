# code-server (VS Code in the browser) on Railway

Deploy [code-server](https://github.com/coder/code-server) — full VS Code, running in the browser, reachable from any device — using a fixed fork of the `deploy-code-server` project that Railway's own `virtual-vscode` template is built on.

This template fixes two real gaps found in the reference template by reading its actual Dockerfile and entrypoint script rather than trusting its variable names and version pin.

## What's fixed vs. the official Railway template

| Issue | Reference template | This template |
|---|---|---|
| **code-server is years out of date** | Pins `codercom/code-server:4.9.0` in its `Dockerfile` — a release from 2022/2023. The template is labeled "[Updated Oct '26]" but the actual image tag was never bumped. | Pins `codercom/code-server:4.141.0`, the latest stable release as of this writing (Oct 2026) — ~130 releases of fixes and features ahead. |
| **No real persistence** | No volume is attached. The upstream project's own docs admit storage "may not be persistent" unless you set up an [rclone](https://rclone.org/) remote yourself — a non-trivial, fully-manual setup most people deploying a one-click template will never do. | A Railway volume is mounted at `/home/coder/project` (where code-server opens and clones your repo), and the entrypoint was patched to detect existing content and skip re-cloning — your files and edits survive every redeploy, with zero extra setup. |
| **No healthcheck configured** | Ships with no `healthcheckPath`, so Railway can't detect a broken boot before routing traffic to it. | Sets `healthcheckPath=/healthz` — code-server's own built-in, unauthenticated status endpoint, verified to return `200 {"status":"alive",...}`. |

### Why not just add a healthcheck at `/`?

`/` serves the VS Code UI itself, and code-server redirects unauthenticated requests to `/login` (`302`). Railway's healthcheck prober doesn't follow redirects, so pointing it at `/` would mark every deploy unhealthy regardless of whether the app is actually fine — the same mistake that shows up across several reference templates we've checked. code-server ships a real `/healthz` endpoint (confirmed in its own source, `src/node/routes/health.ts`) specifically for this purpose, registered with no authentication required. That's what this template uses.

## What you get

- The latest stable code-server, deployed from a maintained fork — no Dockerfile editing required to get a current version
- A persistent volume at `/home/coder/project` — your workspace survives redeploys and restarts
- Optional `GIT_REPO` auto-clone on first boot — point it at any public git repo and it opens ready to go
- A real, working healthcheck, so Railway actually knows when your environment is up

## After deploying

1. Open the service's public domain — you'll be redirected to `/login`.
2. Log in with the `PASSWORD` variable (auto-generated, visible in the Variables tab).
3. If you set `GIT_REPO`, your repo is cloned into the workspace on first boot. From then on, your edits — and anything you `git clone`, `npm install`, or otherwise create — persist across redeploys.

## Adding your own tools

This image is built from a Dockerfile you can extend, same as upstream: add `apt-get install`, `code-server --install-extension`, or `COPY` lines to the bottom of [the `Dockerfile`](https://github.com/shruti060701/virtual-vscode/blob/main/Dockerfile), push, and Railway rebuilds automatically.

## Variables

| Variable | Default | Purpose |
|---|---|---|
| `PASSWORD` | random | Login password for code-server |
| `GIT_REPO` | _(optional)_ | A public git repo URL to auto-clone into the workspace on first boot |
| `HASHED_PASSWORD` | _(optional)_ | SHA-256 hash of a password — overrides `PASSWORD` if set |
| `DOTFILES_REPO` | _(optional)_ | A [dotfiles](https://dotfiles.github.io/) repo to apply on boot |
| `START_DIR` | `/home/coder/project` | The directory code-server opens — matches the volume's mount path, don't change unless you also move the volume |

## License

This template deploys a fork of [`coder/deploy-code-server`](https://github.com/coder/deploy-code-server) (MIT), which itself builds on the official [`coder/code-server`](https://github.com/coder/code-server) (MIT) image. The fork only changes the base image version pin and makes the init script volume-aware — see [the diff](https://github.com/shruti060701/virtual-vscode/commit/69b88a1dd179f0ec0d68a0555bb1a34d232f202a).
