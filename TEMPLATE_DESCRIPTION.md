code-server — VS Code in your browser

Full VS Code, running in a container, reachable from any device. This template deploys a fixed fork of the reference `deploy-code-server` project: bumped from a 2022-era code-server release to the latest stable, with a persistent volume so your workspace actually survives a redeploy instead of depending on manual rclone setup, and a real healthcheck at `/healthz` instead of none at all.
