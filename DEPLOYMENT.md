# Deployment Report — Day 12: Production AI Agent on Render

**Student:** Võ Thanh Chung - 2A202600335  
**Date:** 17/04/2026

---

## Live Service

| | |
|---|---|
| **URL** | https://day12-ha-tang-cloud-va-deployment-synm.onrender.com |
| **Platform** | Render (Web Service — Docker runtime) |
| **Region** | Singapore (Southeast Asia) |
| **Plan** | Starter |
| **Status** | ✅ Live |

---

## Architecture

```
GitHub (main branch)
        │
        │  git push  →  auto-deploy
        ▼
   Render Build
        │
        │  Docker build (repo root context)
        │  Dockerfile: 06-lab-complete/Dockerfile
        │  Multi-stage: builder → runtime (python:3.11-slim)
        │  Non-root user: agent
        ▼
   Render Container
        │
        │  uvicorn app.main:app --host 0.0.0.0 --port 8000 --workers 2
        ▼
   Public HTTPS endpoint  (Render TLS termination)
        │
        ├── GET  /          → service info
        ├── GET  /health    → liveness probe
        ├── GET  /ready     → readiness probe
        ├── POST /ask       → AI agent (requires X-API-Key)
        └── GET  /metrics   → usage stats (requires X-API-Key)
```

---

## Deployment Steps

### 1. Prepare monorepo for Render

The repo is a monorepo. Two files were created/modified so Render can build `06-lab-complete` from the repo root:

**`render.yaml` (repo root — new):**
```yaml
services:
  - type: web
    name: ai-agent-production
    runtime: docker
    dockerfilePath: ./06-lab-complete/Dockerfile
    dockerContext: .
    region: singapore
    plan: starter
    healthCheckPath: /health
    autoDeploy: true
    envVars:
      - key: ENVIRONMENT
        value: production
      - key: AGENT_API_KEY
        generateValue: true
      - key: JWT_SECRET
        generateValue: true
```

**`06-lab-complete/Dockerfile` (updated COPY paths):**
```dockerfile
# Paths relative to repo root (dockerContext: .)
COPY 06-lab-complete/requirements.txt .
COPY 06-lab-complete/app/ ./app/
COPY utils/ ./utils/
```

### 2. Key Dockerfile fixes applied

| Issue | Fix |
|---|---|
| `pip install --user` → packages in `/root/.local`, not importable by non-root `agent` user | Changed to `pip install --prefix=/install`, then `COPY --from=builder /install /usr/local` |
| `response.headers.pop("server", None)` → `MutableHeaders` has no `.pop()` | Changed to `if "server" in response.headers: del response.headers["server"]` |

### 3. Render web service settings (manual form)

| Field | Value |
|---|---|
| Language | Docker |
| Branch | main |
| Region | Singapore (Southeast Asia) |
| Root Directory | *(empty — repo root)* |
| Dockerfile Path | `./06-lab-complete/Dockerfile` |

### 4. Auto-deploy

Render watches the `main` branch. Every `git push origin main` triggers a new build and zero-downtime deployment automatically.

---

## Live API Verification

### GET / — Service info
```
GET https://day12-ha-tang-cloud-va-deployment-synm.onrender.com/
```
```json
{
  "app": "Production AI Agent",
  "version": "1.0.0",
  "environment": "development",
  "endpoints": {
    "ask": "POST /ask (requires X-API-Key)",
    "health": "GET /health",
    "ready": "GET /ready"
  }
}
```

### GET /health — Liveness probe ✅
```
GET https://day12-ha-tang-cloud-va-deployment-synm.onrender.com/health
```
```json
{
  "status": "ok",
  "version": "1.0.0",
  "environment": "development",
  "uptime_seconds": 480.7,
  "total_requests": 9,
  "checks": { "llm": "openai" },
  "timestamp": "2026-04-17T10:48:36.867131+00:00"
}
```

### GET /ready — Readiness probe ✅
```
GET https://day12-ha-tang-cloud-va-deployment-synm.onrender.com/ready
```
```json
{ "ready": true }
```

### POST /ask — AI agent (requires API key)
```bash
curl -X POST https://day12-ha-tang-cloud-va-deployment-synm.onrender.com/ask \
  -H "Content-Type: application/json" \
  -H "X-API-Key: <AGENT_API_KEY>" \
  -d '{"question": "What is Docker?"}'
```
```json
{
  "question": "What is Docker?",
  "answer": "Container là cách đóng gói app để chạy ở mọi nơi. Build once, run anywhere!",
  "model": "gpt-4o-mini",
  "timestamp": "2026-04-17T10:48:37.000000+00:00"
}
```

---

## Production Features Deployed

| Feature | Implementation | Status |
|---|---|---|
| 12-Factor config | All settings via env vars (`config.py`) | ✅ |
| Structured JSON logging | `logging` module with JSON format | ✅ |
| API Key authentication | `X-API-Key` header, `APIKeyHeader` | ✅ |
| Rate limiting | Sliding window, 20 req/min | ✅ |
| Cost guard | Daily budget $5 USD, auto-reset | ✅ |
| Input validation | Pydantic `AskRequest` (max 2000 chars) | ✅ |
| Health check (`/health`) | Liveness probe, returns uptime + checks | ✅ |
| Readiness probe (`/ready`) | 503 until app fully started | ✅ |
| Graceful shutdown | SIGTERM handler via lifespan context | ✅ |
| Security headers | `X-Content-Type-Options`, `X-Frame-Options` | ✅ |
| CORS | Configurable `ALLOWED_ORIGINS` | ✅ |
| Non-root container | User `agent` (no root privileges) | ✅ |
| Multi-stage Docker build | Builder + slim runtime, ~250 MB image | ✅ |
| Auto-deploy | `git push` → Render rebuilds automatically | ✅ |
| HTTPS | Render TLS termination (free) | ✅ |

---

## Troubleshooting Log

| Error | Root Cause | Fix |
|---|---|---|
| `ModuleNotFoundError: No module named 'uvicorn'` | `pip install --user` installs to `/root/.local`; non-root `agent` user can't find packages there | `pip install --prefix=/install` → `COPY --from=builder /install /usr/local` |
| `AttributeError: 'MutableHeaders' object has no attribute 'pop'` | Starlette's `MutableHeaders` does not implement `.pop()` | `del response.headers["server"]` with guard |
| Build fails: `COPY app/` not found | Docker context was `06-lab-complete/` but `utils/` lives at repo root | Set `dockerContext: .` (repo root), update all COPY paths to be relative to root |

---

## Environment Note

The `/health` response shows `"environment": "development"`. This is because the service was created via Render's manual web form — the `render.yaml` env vars are only applied when deploying via **Blueprint**. To fix, set `ENVIRONMENT=production` in the Render dashboard under **Environment → Environment Variables**.
