code-server — VS Code in your browser

## Composer description field (paste this in)

Browser VS Code: latest code-server, real healthcheck, persistence verified live.

## Longer version (for context)

Full VS Code, running in a container, reachable from any device. This template deploys a fixed fork of the reference `deploy-code-server` project: bumped from a 2022-era code-server release to the latest stable, with a persistent volume so your workspace actually survives a redeploy instead of depending on manual rclone setup, and a real healthcheck at `/healthz` instead of none at all.

## Why this description
This category is crowded (13+ competing code-server templates), and "persistent files" is already claimed by at least one rival. The differentiator here isn't claiming persistence — several others do too — it's that persistence was actually proven: three live-caught bugs (ext4 `lost+found`, git's empty-dir requirement, a root-owned volume mount) were found and fixed by deploying and reading logs, then re-verified across a real service restart.
