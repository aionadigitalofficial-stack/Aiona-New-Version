# Aiona Voice V5

Self-hosted voice AI agent platform, built on [Dograh AI](https://github.com/dograh-hq/dograh)
(BSD-2-Clause), deployed via Coolify on a Hostinger KVM 8 VPS.

👉 **See [README-DEPLOY.md](./README-DEPLOY.md) for the full setup guide.**

## Contents

- `docker-compose.yml` — Coolify-ready compose stack (Postgres, Redis, MinIO, coturn TURN server, API, UI)
- `.env.example` — environment variable template (copy to `.env`, fill in your domains + IP)
- `deploy/turnserver.conf.template` — coturn config template, rendered automatically at deploy time
- `README-DEPLOY.md` — step-by-step: GitHub → Coolify → DNS → deploy → post-config
