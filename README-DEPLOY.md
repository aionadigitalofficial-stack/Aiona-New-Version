# Aiona Voice V5 — Deployment Guide (Hostinger KVM 8 + Coolify)

This deploys [Dograh AI](https://github.com/dograh-hq/dograh) (open-source
voice AI platform) under your own branding/infra as **Aiona Voice V5**, using
the official prebuilt Docker images — no build step, fast and reliable.

> **Note on branding:** this setup uses Dograh's official images as-is, so
> the app itself will still show "Dograh" in the UI (title, logo). "Aiona
> Voice V5" is the name of *your deployment/infra* (repo, containers, domain).
> Full UI white-labeling (changing the title/logo in the app itself) requires
> building from source instead of pulling prebuilt images — see the
> **"Full white-label build"** section at the bottom if you want that too.

---

## 0. What you're setting up

| Piece | Where |
|---|---|
| App (UI) | `https://app.yourdomain.com` |
| API (backend + WebSocket + telephony webhooks) | `https://api.yourdomain.com` |
| Storage (call recordings/transcripts) | `https://storage.yourdomain.com` |
| TURN server (WebRTC voice) | `<your VPS public IP>` on UDP/TCP 3478, 5349, 49152–49200 |

You'll need **3 DNS subdomains** pointed at your KVM 8 VPS's IP before you start.

---

## 1. Push this to GitHub

```bash
cd aiona-voice-v5
git init
git add .
git commit -m "Aiona Voice V5 - initial deployment config"
git branch -M main
git remote add origin https://github.com/<your-org>/aiona-voice-v5.git
git push -u origin main
```

**Do not commit your real `.env`** with production secrets — `.env.example`
is safe to commit (it has placeholder domains), but once you fill in real
values, keep that file local or paste the values directly into Coolify's
Environment Variables tab instead.

---

## 2. DNS

In your DNS provider, add three **A records** pointing at your VPS's public IP:

```
app.yourdomain.com      -> <VPS_IP>
api.yourdomain.com      -> <VPS_IP>
storage.yourdomain.com  -> <VPS_IP>
```

---

## 3. Server prerequisites (Hostinger KVM 8)

- Coolify already installed on the VPS (if not: `curl -fsSL https://cdn.coollabs.io/coolify/install.sh | bash`)
- Firewall/Hostinger panel: open these ports —
  - **TCP 80, 443** (Coolify/Traefik — HTTPS for app/api/storage)
  - **TCP 6001, 6002** (Coolify's own dashboard/realtime, if not already open)
  - **TCP 3478, 5349** and **UDP 3478, 5349, 49152–49200** (TURN/WebRTC — required for calls to actually carry audio)

KVM 8 (8 vCPU / ~32 GB RAM class) is comfortably above Dograh's stated
minimum of 8 GB RAM / 4 vCPUs, so you have real headroom for concurrent calls.

---

## 4. Create the resource in Coolify

1. **Projects → New Project** → name it `aiona-voice-v5`.
2. **New Resource → Docker Compose** (not "Application" — Compose, since this is a multi-container stack).
3. Point it at your GitHub repo (`aiona-voice-v5`, branch `main`) — connect via Coolify's GitHub App if not already linked.
4. Coolify will detect `docker-compose.yml` at the repo root.
5. Go to the resource's **Environment Variables** tab and paste in the contents of your filled-in `.env` (real domains + secrets, not the placeholders).

---

## 5. Assign domains per service

This is the key Coolify-specific step — each container gets its own public
domain instead of one nginx box doing path routing:

| Service | Coolify → Domains | Container port |
|---|---|---|
| `ui` | `app.yourdomain.com` | `3010` |
| `api` | `api.yourdomain.com` | `8000` |
| `minio` | `storage.yourdomain.com` | `9000` |

Coolify auto-provisions Let's Encrypt HTTPS for each once DNS resolves.
`postgres`, `redis`, and `coturn`'s config-render step get **no domain** —
they're internal-only (coturn instead gets raw host ports, see below).

---

## 6. Deploy

Click **Deploy** on the resource. Coolify pulls `dograhai/dograh-api` and
`dograhai/dograh-ui`, brings up Postgres/Redis/MinIO, runs the one-shot
`turn-config` job to render coturn's config, then starts `coturn`.

First boot takes a couple of minutes. Watch the logs per-service in Coolify
if anything looks stuck — `api` won't go healthy until Postgres/Redis/MinIO
are all up, and `ui` won't start until `api` is healthy.

Once green, open `https://app.yourdomain.com` — you should get the sign-up/login screen.

---

## 7. Verify TURN/WebRTC actually works

Coturn only binds correctly if `TURN_HOST` in your env is the VPS's **raw
public IP**, not a domain. Test:

1. Create a test agent in the UI, click **Web Call**.
2. If you hear audio both ways, TURN is fine.
3. If the call connects but there's no audio, check the browser console for
   `iceConnectionState: failed` — usually means UDP 49152–49200 isn't
   actually open on the Hostinger firewall (panel-level, separate from any
   in-VPS firewall like ufw).

---

## 8. Post-deploy configuration (in the app itself)

Once the app is up, configure it exactly the same way as any Dograh install —
none of this lives in the compose file:

- **Models** (`/model-configurations`): add your LLM key (OpenAI/Anthropic),
  STT (e.g. Deepgram), TTS (e.g. ElevenLabs/Cartesia) — or use a
  speech-to-speech realtime model instead.
- **Telephony** (`/telephony-configurations`): add your provider —
  Vobiz, Twilio, Vonage, Plivo, Cloudonix, or Asterisk ARI. For Vobiz
  specifically, the Answer URL Dograh needs is:
  `https://api.yourdomain.com/api/v1/telephony/inbound/run`
  (Dograh pushes this automatically when you assign an inbound workflow —
  just confirm it landed correctly on the Vobiz Application if inbound calls
  don't ring through).

---

## 9. Updating later

Since this uses prebuilt images (no local build), an update is just:

```bash
# In Coolify: resource -> Redeploy
# (pulls the latest dograhai/dograh-api:latest and dograh-ui:latest)
```

Pin `REGISTRY`/image tags in `docker-compose.yml` instead of `:latest` if you
want controlled, tested upgrades rather than auto-pulling newest on redeploy.

---

## Full white-label build (optional, more work)

If you want the app's own UI to say "Aiona Voice V5" instead of "Dograh"
(browser tab title, logo), you need to build from source instead of pulling
prebuilt images — Coolify supports this in the same Docker Compose resource:

1. Fork `dograh-hq/dograh` into your own GitHub org instead of just this
   deploy-config repo.
2. Edit `ui/src/app/layout.tsx` — change `title: "Dograh"` to
   `title: "Aiona Voice V5"`.
3. Replace logo/favicon assets under `ui/public/`.
4. In `docker-compose.yml`, replace the `image:` lines for `api` and `ui`
   with `build: ./api` and `build: ./ui` respectively (both directories
   already have Dockerfiles).
5. Redeploy in Coolify — first build takes several minutes since it compiles
   both images from scratch.

Since Dograh is BSD-2-Clause licensed, this is allowed — just keep the
original license/copyright file in the repo.
