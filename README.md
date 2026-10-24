# muse2api

**Use muse.ai through a standard OpenAI-compatible API.**

muse2api wraps the muse.ai web interface into an OpenAI-compatible REST API. It drives a pool of headless Chromium sessions (via the Chrome DevTools Protocol) that automate the muse.ai web UI, so you can call chat, image generation, and video generation through the same endpoints your code already knows — `/v1/chat/completions`, `/v1/images/generations`, `/v1/videos`.

- **Version:** v1.6.3 · **License:** MIT · **Stack:** Python 3.11 + FastAPI + Chromium (CDP) inside Docker
- **Dashboard:** built-in web admin console at `/admin` (Chinese/English switch)

> ⚠️ **Disclaimer** — this project automates the muse.ai *website*, not an official API. That likely violates muse.ai's Terms of Service and your account can be restricted or banned. Use a secondary account, keep volume reasonable, and understand the risk before connecting a real account. Everything (cookies, tasks, media) stays on your own server.

---

## Table of contents

1. [Features](#features)
2. [How it works](#how-it-works)
3. [Requirements](#requirements)
4. [Quick start (Docker)](#quick-start-docker)
5. [Configuration reference](#configuration-reference)
6. [Connect your muse.ai account](#connect-your-museai-account)
7. [Admin console](#admin-console)
8. [API reference & examples](#api-reference--examples)
9. [Using it with clients & gateways](#using-it-with-clients--gateways)
10. [Testing](#testing)
11. [Troubleshooting](#troubleshooting)
12. [Operations & maintenance](#operations--maintenance)

---

## Features

| Area | What you get |
|---|---|
| **Chat** | `/v1/chat/completions` and `/v1/responses`, SSE streaming (~2–3 s first token), multi-turn context, system prompts, model aliasing |
| **Images** | `/v1/images/generations`, `/v1/images/edits`, multiple aspect ratios, URL or `b64_json` output, watermark-free; async task mode (`/v1/images/tasks`) for use behind proxies/CDNs |
| **Video** | Text-to-video and image-to-video (first-frame reference), 5–10 s clips, async task creation + status polling, local MP4/WebP serving |
| **Account pool** | Unlimited account imports, warm-tab affinity with LRU scheduling, isolated Chromium contexts (default 2 concurrent, configurable), automatic failover when a quota is exhausted |
| **Session keep-alive** | Background heartbeat renews session cookies (+48 h) and wakes cloud VMs, so you rarely need to re-login |
| **Import tooling** | Bundled Chrome extension and a Python helper script that extract login cookies (including HttpOnly) and push them into the pool |
| **Admin console** | Dark-themed dashboard: account health/quota, task progress, media library, one-click online upgrade, API key rotation |

## How it works

```
Your client (curl / SDK / OmniRoute / anything OpenAI-compatible)
        │  Bearer m2a_...
        ▼
muse2api (FastAPI, port 18610)
        │  routes request through the account pool
        ▼
Headless Chromium (CDP, isolated profile per account)
        │  automates the muse.ai web UI
        ▼
muse.ai  ──► generated media saved to ./data/media and served locally
```

muse.ai's login cookies (`hatch_sess`, `hatch_gw`, `hatch_vml`, `hatch_native_auth_device`) are all `HttpOnly`, so they can't be grabbed with `document.cookie`. That's why the project ships a browser extension / helper script that reads them from the browser layer and uploads them to the account pool.

## Requirements

- **Docker Desktop** (or any Docker Engine + Compose v2) — recommended
- A **muse.ai account** you can log into in Chrome/Edge
- ~4 GB RAM free for the container (Chromium rendering; `shm_size: 2gb` is set in compose)
- Linux host note: only x86_64 is practical — Chromium + the upstream page expect it

## Quick start (Docker)

```bash
# 1. Get the code
git clone https://github.com/czg86389-hub/muse2api.git muse-ai-to-api
cd muse-ai-to-api

# 2. Create your .env (a strong key is generated for you below)
KEY="m2a_$(openssl rand -hex 24)"
sed "s|^MUSE2API_KEY=.*|MUSE2API_KEY=$KEY|" .env.example > .env
echo "Your API key: $KEY"        # save it — the admin page and all API calls need it

# 3. Build and start
docker compose build
docker compose up -d

# 4. Verify
curl http://localhost:18610/healthz
# → {"status":"ok",...}
```

Then connect a muse.ai account ([next section](#connect-your-museai-account)) — until you do, generation endpoints return *"no available accounts, please import cookies via the admin page"*.

Useful compose commands:

```bash
docker compose logs -f          # follow logs
docker compose restart          # restart
docker compose down             # stop (data in ./data is kept)
docker compose up -d --build    # rebuild after pulling new code
```

The service listens on **port 18610** (`http://localhost:18610`). All state persists in `./data/` (accounts, tasks, media) because the compose file mounts it as a volume.

## Configuration reference

Everything is configured in `.env` (copy of `.env.example`). Compose passes these into the container.

| Variable | Default | Meaning |
|---|---|---|
| `MUSE2API_KEY` | *(auto-generated)* | Admin + API auth key, must start with `m2a_`. If left empty, a random one is generated and printed to the container log on first start |
| `MUSE2API_HOST` / `MUSE2API_PORT` | `0.0.0.0` / `18610` | Listen address. Keep `0.0.0.0` inside Docker |
| `MUSE2API_PUBLIC_BASE` | *(empty)* | Public base URL (e.g. `https://api.yourdomain.com`) shown in the admin UI; leave empty to auto-detect from the browser URL |
| `MUSE2API_CHROMIUM` | `/usr/bin/chromium` | Chromium binary path (set automatically in the Docker image) |
| `MUSE2API_CDP_PORT` | `19210` | Internal CDP debug port |
| `MUSE2API_IMAGE_TIMEOUT` | `240` | Image generation timeout, seconds |
| `MUSE2API_VIDEO_TIMEOUT` | `600` | Video generation timeout, seconds |
| `MUSE2API_CHAT_TIMEOUT` | `300` | Chat timeout, seconds |
| `MUSE2API_CONCURRENCY` | `2` | Max parallel generation tasks (raise freely on a bigger box) |
| `MUSE2API_KEEPALIVE_DISABLED_ACCOUNTS` | `0` | `1` = also keep sessions alive for disabled accounts |

## Connect your muse.ai account

Log into **https://muse.ai** first (until you can see the chat UI), then pick **one** of the three methods below. The server needs 4 cookies: `hatch_sess`, `hatch_gw`, `hatch_vml`, `hatch_native_auth_device` — all `HttpOnly`, hence the tooling.

### Method A — Chrome extension (easiest)

1. Open `chrome://extensions` (Edge: `edge://extensions`) → enable **Developer mode** → **Load unpacked** → select the `extension/` folder of this repo.
2. Log into muse.ai in that browser.
3. Click the extension icon → fill in **Base URL** (`http://localhost:18610`) and **API Key** (your `m2a_...` key) — only needed once, it remembers.
4. Click **Read & import** → "✓ import success".

### Method B — Python helper script (no extension)

Runs on the host, needs Python 3.8+ and local Chrome. It opens a dedicated Chrome window with a temp profile, waits for you to log in, captures the cookies and uploads them automatically:

```bash
python tools/get_muse_cookie.py \
  --base http://localhost:18610 \
  --key  m2a_your_key_here \
  --label acc-01
```

### Method C — Manual paste in the admin console

F12 on muse.ai → **Application → Cookies → https://muse.ai** (DevTools can see HttpOnly values) → copy the values, then in the admin console's account dialog paste:

```
hatch_sess=VALUE; hatch_gw=VALUE; hatch_vml=VALUE
```

Multiple accounts at once, one per line:

```
acc-01 | hatch_sess=...; hatch_gw=...
acc-02 | hatch_sess=...; hatch_gw=...
```

### Verify the account

Open `/admin` → **Account Pool**: the account should show **healthy** with a cookie expiry date. Press its **Test** button to confirm the session works and quota can be read. Or check headlessly:

```bash
curl -s http://localhost:18610/readyz -H "Authorization: Bearer $KEY"
# → {"status":"ok","accounts":1,...}
```

## Admin console

Open **http://localhost:18610/admin** and enter your `m2a_...` key in the **API Key** box (top-right) — it is stored in your browser's localStorage.

- **Account Pool** — add/remove accounts, import cookies, per-account test, re-login, quota read, enable/disable, keep-alive controls
- **Tasks** — live progress of image/video jobs, cancel or clear
- **Media** — browse/delete generated files
- **API Key** — rotate the server key (all clients must be updated afterwards)
- **Update** — check the upstream repo and upgrade with one click
- Language switch (中文 / English) in the header

Tip: `http://localhost:18610/admin?key=m2a_...` logs you in via URL. Note the key is saved **per origin** — `localhost:18610` and `127.0.0.1:18610` have separate localStorage.

## API reference & examples

All endpoints require `Authorization: Bearer m2a_...` and speak OpenAI JSON dialect. Base URL: `http://localhost:18610`.

### Models

```bash
curl -s http://localhost:18610/v1/models -H "Authorization: Bearer $KEY"
# muse-spark  — chat / code
# muse-image  — text-to-image / image editing
# muse-video  — text-to-video / image-to-video
```

### Chat (non-streaming)

```bash
curl -s -X POST http://localhost:18610/v1/chat/completions \
  -H "Authorization: Bearer $KEY" -H "Content-Type: application/json" \
  -d '{"model":"muse-spark","messages":[{"role":"user","content":"Say hi from muse2api"}]}'
```

### Chat (streaming, SSE)

```bash
curl -s -N -X POST http://localhost:18610/v1/chat/completions \
  -H "Authorization: Bearer $KEY" -H "Content-Type: application/json" \
  -d '{"model":"muse-spark","stream":true,"messages":[{"role":"user","content":"Tell me a joke"}]}'
```

### Image generation

```bash
curl -s -X POST http://localhost:18610/v1/images/generations \
  -H "Authorization: Bearer $KEY" -H "Content-Type: application/json" \
  -d '{"model":"muse-image","prompt":"a cute corgi wearing a red scarf","n":1}'
# → {"data":[{"url":"..."}]}  (or b64_json)
```

Async variant for long runs behind proxies: `POST /v1/images/tasks` then poll `GET /v1/images/tasks/{id}`.

### Video (async task + polling)

```bash
# create
curl -s -X POST http://localhost:18610/v1/videos \
  -H "Authorization: Bearer $KEY" -H "Content-Type: application/json" \
  -d '{"model":"muse-video","prompt":"a corgi running on the beach","duration":5}'
# → {"task_id":"...","status":"queued"}

# poll until status is "completed" (progress 0-100), then download the mp4 url
curl -s http://localhost:18610/v1/videos/<task_id> -H "Authorization: Bearer $KEY"
```

Media files are served from `/v1/media/{name}`.

### Health / readiness

```bash
curl -s http://localhost:18610/healthz    # liveness
curl -s http://localhost:18610/readyz -H "Authorization: Bearer $KEY"   # account pool status
```

## Using it with clients & gateways

Any tool that accepts a custom OpenAI-compatible base URL works. Point it at:

```
Base URL:  http://localhost:18610/v1
API Key:   m2a_...        (your MUSE2API_KEY)
Models:    muse-spark, muse-image, muse-video
```

**OpenAI Python SDK:**

```python
from openai import OpenAI
client = OpenAI(base_url="http://localhost:18610/v1", api_key="m2a_...")
r = client.chat.completions.create(model="muse-spark",
    messages=[{"role": "user", "content": "Hello"}])
print(r.choices[0].message.content)
```

**OmniRoute / other AI gateways** — add a provider of type *OpenAI Compatible*:

- Name: `Muse2API`, Prefix: `muse`
- API Type: `Chat Completions`
- Base URL: `http://localhost:18610/v1` — **unless OmniRoute itself runs in Docker**, then use `http://host.docker.internal:18610/v1` (`localhost` inside a container points at the container, not your host)
- Store your `m2a_` key in the provider's *connection*, and register the model IDs (`muse-spark`, ...) as custom models. Clients then call `muse/muse-spark`.

## Testing

### Quick end-to-end test script

Two helpers live in the repo root and read the key from `.env` automatically:

```bash
./test-api.sh            # Git Bash / Linux: health, models, auth, chat
./test-api.sh --stream   # + SSE streaming
./test-api.sh --image    # + image generation (uses quota, ~1-4 min)
./test-api.sh --video    # + video generation (uses quota, ~5-10 min)
./test-api.sh --all      # everything

# Windows double-click: test-api.bat (quick tests only)
```

Each check prints `[PASS]` / `[FAIL]` / `[SKIP]` with the reason; exit code is non-zero if anything failed.

### Offline regression suite (upstream)

The repo ships self-contained regression tests that don't touch real accounts or quota. They are standalone scripts (not pytest) and expect to run from the repo root:

```bash
python tests/test_media_selection.py
python tests/test_async_images.py
python tests/test_vm_wait.py engine.py --assert
python tests/test_session_health.py
# ... one python tests/test_<name>.py per file in tests/
```

Inside the container you must first `docker cp tests muse2api:/app/tests` (the production image doesn't copy `tests/`), and `test_muse_video_delivery.py` additionally needs `node`.

## Troubleshooting

| Symptom | Cause & fix |
|---|---|
| *"没有可用账号 / no available accounts"* from chat/image/video | No account in the pool yet — connect one via [Method A/B/C](#connect-your-museai-account) |
| **"API Key is missing or incorrect"** in the admin page | The page's saved key doesn't match the server. Re-paste the `m2a_` key into the API Key box (top-right) and press Save. Beware `localhost` vs `127.0.0.1` have separate saved keys |
| Extension says *"API Key 不对 (401)"* | Same key mismatch — paste the key from the admin console / `.env` into the extension popup |
| Chat returns **401** | Your client's key is wrong. Rotate-safe fix: read the current key from `.env` (or `GET /admin/apikey`) and update clients |
| Account shows **401/403** | Session invalid/blocked — press **Test**, then **Re-login** or re-import fresh cookies |
| Account shows **429** | Quota exhausted on that account — add another account to the pool or wait for reset |
| *"There was an error parsing the body"* | Malformed JSON in the request — usually an encoding/quoting issue in the shell; prefer single-quoted ASCII payloads (see the examples above) |
| Image/video hangs for minutes | Normal — generation takes 1–4 min (images) / 5–10 min (videos); use the async task endpoints behind strict proxies |
| Gateway (OmniRoute etc.) can't reach the API | Gateway runs in Docker? Use `http://host.docker.internal:18610/v1` instead of `localhost` |
| Chromium crashes in container | Ensure `shm_size: '2gb'` is present in `docker-compose.yml` and the host has enough RAM |

## Operations & maintenance

```bash
docker compose ps                 # status
docker compose logs -f muse2api   # logs (account events, task results, keep-alive)
docker compose restart            # restart service
docker compose down && docker compose up -d --build   # upgrade after git pull
```

- **Data lives in `./data/`** — accounts, task history and media survive container rebuilds. Back it up if you care about the media library.
- **Keep-alive** runs every 15 minutes and extends sessions by +48 h, so an imported account stays usable without re-login as long as the server stays up.
- **Rotation**: if you ever expose the service beyond localhost, put it behind an HTTPS reverse proxy (Nginx config samples are in `deploy/`), and rotate the key from the admin console.

---

*Upstream project & docs (Chinese): see [README.upstream-zh.md](README.upstream-zh.md) · upstream repo: [czg86389-hub/muse2api](https://github.com/czg86389-hub/muse2api) · MIT License.*
