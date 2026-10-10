# Template Composer setup checklist

## Service
- **Source**: GitHub repo `shruti060701/virtual-vscode`, branch `main` (built from its `Dockerfile` — not a prebuilt image)
- **Icon**: code-server's own logo, e.g. `https://raw.githubusercontent.com/coder/code-server/main/src/browser/media/favicon.svg` (reference template shipped with no icon at all)

## Volume
- Mount path: `/home/coder/project`
- Must match the `START_DIR` variable if you ever change it from the default
- This is the actual fix over the reference template, which has no volume and documents a manual rclone setup as the only way to persist files

## Networking
- Generate a domain, **target port 8080** (code-server's bind address is hardcoded in the entrypoint to `0.0.0.0:8080`, not driven by `$PORT` — do not rely on auto-detection)

## Healthcheck
- Path: `/healthz`
- **Do not use `/`** — code-server redirects unauthenticated requests to `/login` (`302`), and Railway's healthcheck prober doesn't follow redirects, so every deploy would be marked unhealthy. `/healthz` is code-server's own built-in status endpoint (`src/node/routes/health.ts`), registered with no auth required.
- Restart policy: `ALWAYS`

## Variables

| Name | Default Value | Description (paste into Composer) | Optional |
|---|---|---|---|
| `PASSWORD` | `${{ secret(20, 'abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789') }}` | Password to log into code-server. | No |
| `GIT_REPO` | _(empty)_ | A public git repo URL to auto-clone into the workspace on first boot. Leave empty to start with a blank workspace. | Yes |

Optional, power-user variables (document in README, don't need to be in the Composer's primary variable list): `HASHED_PASSWORD`, `DOTFILES_REPO`, `START_DIR`.

**Important**: this template's fork patches `deploy-container/entrypoint.sh` so that `$START_DIR` (the volume mount) is only initialized once — on every boot after the first, if the volume already has content, cloning is skipped entirely so your work isn't touched. The original upstream script re-attempts `git clone` into the directory on every single boot, which — once a volume is attached — fails outright after the first successful clone.

## Verified live (2026-10-10)
- `GET /healthz` → `200 {"status":"alive"/"expired","lastHeartbeat":...}`, no auth
- `GET /` → `302` to `/login` (expected — confirms `/` is the wrong healthcheck path)
- `GET /login` → `200`, real code-server login page (`<title>code-server login</title>`)
- Deploy logs confirm `code-server 4.141.0` running (vs. reference's pinned `4.9.0`)
- First-boot volume initialization took three live-caught fixes before it actually worked:
  1. A fresh Railway (ext4) volume ships its own `lost+found` directory, which a naive "is this directory empty" check sees as content — excluded explicitly
  2. `git clone` independently refuses a non-empty destination even after that — now clones to a scratch directory and moves the result into `$START_DIR`
  3. The volume mount comes back **root-owned**, while code-server runs as the unprivileged `coder` user — every file copy into it failed with `Permission denied`, silently, because the script only checked `git clone`'s exit code and logged "success" regardless. Fixed by `chown`-ing the volume to `coder` on boot and gating the success log on the copy step too.
- **Persistence re-verified after a real restart**: restarted the live service and confirmed the deploy logs show `/home/coder/project already has content, skipping init` on the second boot — the cloned repo was not re-cloned or wiped.
