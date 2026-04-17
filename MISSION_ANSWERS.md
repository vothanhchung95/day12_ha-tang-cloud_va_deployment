# MISSION ANSWERS — Day 12: Cloud Deployment Lab

**Student:** Võ Thanh Chung - 2A202600335  
**Date:** 17/04/2026

---

## Part 1: Localhost vs Production

### Exercise 1.1 — Anti-patterns found in `01-localhost-vs-production/develop/app.py`

I identified **6 anti-patterns** in the develop version:

| # | Anti-pattern | Line | Problem | Fix |
|---|---|---|---|---|
| 1 | **Hardcoded API key** | 17 | `OPENAI_API_KEY = "sk-hardcoded-fake-key-never-do-this"` — if pushed to GitHub, key is exposed instantly | Read from `os.getenv("OPENAI_API_KEY")` |
| 2 | **Hardcoded database URL** | 18 | `DATABASE_URL = "postgresql://admin:password123@..."` — credentials in source code | Read from environment variable |
| 3 | **Debug logging of secrets** | 34 | `print(f"[DEBUG] Using key: {OPENAI_API_KEY}")` — prints the secret to stdout/logs | Never log secret values |
| 4 | **No health check endpoint** | 42–43 | No `/health` route — cloud platforms cannot detect crashes and will not restart the container | Add `GET /health` returning `{"status": "ok"}` |
| 5 | **Hardcoded host `localhost`** | 51 | `host="localhost"` — container only listens on loopback, unreachable from outside | Bind to `0.0.0.0` |
| 6 | **`reload=True` in production** | 53 | `reload=True` watches the filesystem and restarts the process — wastes CPU and can cause instability | Set `reload=False` (or only `True` when `DEBUG=true`) |

> **Bonus:** `DEBUG = True` and `MAX_TOKENS = 500` are hardcoded constants instead of being read from environment variables — making them impossible to change without a code change and redeploy.

---

### Exercise 1.3 — Comparison: Basic vs Production

| Aspect | Basic (`develop/app.py`) | Production (`production/app.py`) |
|---|---|---|
| **Config** | Hardcoded values in source code (`OPENAI_API_KEY = "sk-..."`) | Centralized `config.py` reads all settings from environment variables via `os.getenv()` |
| **Health check** | ❌ No `/health` endpoint | ✅ `GET /health` returns status, uptime, version, timestamp; `GET /ready` returns 503 until startup completes |
| **Logging** | `print()` statements, logs secrets | Structured JSON logging via `logging` module — logs event names and lengths, never secret values |
| **Shutdown** | No signal handling — process is killed hard | SIGTERM handler + lifespan context manager waits for in-flight requests before exiting |
| **Host binding** | `host="localhost"` — unreachable in container | `host=settings.host` → `0.0.0.0` — accepts connections from any network interface |
| **Port** | `port=8000` hardcoded | `port=settings.port` → reads `PORT` env var (Railway/Render inject this automatically) |
| **Reload** | `reload=True` always | `reload=settings.debug` — only enabled when `DEBUG=true` |

**Key takeaway:** The "it works on my machine" problem comes from environment assumptions baked into the code. The 12-Factor App approach externalizes all configuration so the same code runs identically in dev, staging, and production by swapping environment variables only.

---

## Part 2: Docker Containerization

### Exercise 2.1 — Dockerfile Structure Analysis (`02-docker/develop/Dockerfile`)

```dockerfile
FROM python:3.11          # Base image: full Python distribution (~1 GB)
WORKDIR /app              # Working directory inside the container
COPY requirements.txt .   # Copy requirements FIRST — Docker layer cache
RUN pip install ...       # Install dependencies (cached if requirements.txt unchanged)
COPY app.py .             # Copy application code
RUN mkdir -p utils        # Create utils directory
COPY utils/mock_llm.py utils/
EXPOSE 8000               # Document the port (informational only, not binding)
CMD ["python", "app.py"]  # Default command when container starts
```

**Why copy `requirements.txt` before `app.py`?**  
Docker builds images layer by layer. If `requirements.txt` is copied first and hasn't changed, Docker reuses the cached `pip install` layer — even if `app.py` changes. This makes rebuilds much faster.

**CMD vs ENTRYPOINT:**  
`CMD` sets the default command but can be overridden at `docker run` time. `ENTRYPOINT` sets an immutable executable. Using `CMD ["uvicorn", "main:app", ...]` is preferable for web services as it allows passing extra flags.

---

### Exercise 2.3 — Multi-stage Build (`02-docker/production/Dockerfile`)

**Stage 1 — Builder:**
```dockerfile
FROM python:3.11-slim AS builder
RUN apt-get install -y gcc libpq-dev   # Build tools needed to compile some packages
RUN pip install --no-cache-dir --user -r requirements.txt
```

**Stage 2 — Runtime:**
```dockerfile
FROM python:3.11-slim AS runtime
RUN groupadd -r appuser && useradd -r -g appuser appuser   # Non-root user
COPY --from=builder /root/.local /home/appuser/.local      # Only copy installed packages
COPY main.py .
USER appuser                                                # Drop root privileges
HEALTHCHECK CMD python -c "import urllib.request; ..."     # Auto-restart on failure
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000", "--workers", "2"]
```

**Image size comparison:**

| Image | Base | Size (approx.) | Notes |
|---|---|----------------|---|
| `python:3.11` (single-stage) | Full CPython distribution | 1.66 GB        | Includes compiler, headers, test suite |
| `python:3.11-slim` (multi-stage) | Debian slim | 236 MB         | Build tools left behind in builder stage |

Multi-stage build reduces image size by ~70–80% because the `gcc`, `libpq-dev`, and build caches from Stage 1 are **never copied** into the final image.

**Security improvement:** The runtime container runs as `appuser` (non-root), so even if an attacker exploits a vulnerability in the app they cannot write to system directories.

---

### Exercise 2.4 — Docker Compose Stack (`02-docker/production/docker-compose.yml`)

**Architecture diagram:**

```
Internet
    │
    ▼
 nginx:80/443  ← Reverse proxy & load balancer
    │
    ├──► agent (replica 1) :8000
    └──► agent (replica 2) :8000
              │
              ├──► redis:6379   (session cache, rate limiting)
              └──► qdrant:6333  (vector database for RAG)
```

**Services:**

| Service | Image | Role |
|---|---|---|
| `agent` | Multi-stage Dockerfile | FastAPI AI agent, 2 replicas |
| `redis` | `redis:7-alpine` | Session cache, rate limiting storage |
| `qdrant` | `qdrant/qdrant:v1.9.0` | Vector database for RAG |
| `nginx` | `nginx:alpine` | Reverse proxy, load balancer, SSL termination |

**Key design decisions:**
- Agent ports are **not exposed directly** — all traffic goes through nginx
- Services communicate on an isolated `internal` bridge network
- `depends_on` with `service_healthy` conditions ensure redis and qdrant are ready before the agent starts
- Persistent `volumes` for redis and qdrant data survive container restarts
- Secrets loaded from `.env.local` (excluded from git via `.gitignore`)

```bash
# Start the full stack
docker compose up

# Test via nginx
curl http://localhost/health
```

---

## Part 3: Cloud Deployment

### Exercise 3.1 — Render Deployment (06-lab-complete)

**Platform chosen:** Render (Docker Web Service)  
**Live URL:** https://day12-ha-tang-cloud-va-deployment-synm.onrender.com  
**Module deployed:** `06-lab-complete` (Production AI Agent)

**Deployment steps:**

#### Step 1 — Prepare monorepo for Render

Since `06-lab-complete/` lives inside a monorepo, two changes were needed so Render can build from the repo root while targeting only this module's Dockerfile:

**`render.yaml` (created at repo root):**
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

**`06-lab-complete/Dockerfile` (COPY paths updated to repo root):**
```dockerfile
COPY 06-lab-complete/requirements.txt .
COPY 06-lab-complete/app/ ./app/
COPY utils/ ./utils/
```

#### Step 2 — Render web service settings

| Field | Value |
|---|---|
| Language | Docker |
| Branch | `main` |
| Region | Singapore (Southeast Asia) |
| Root Directory | *(empty — repo root)* |
| Dockerfile Path | `./06-lab-complete/Dockerfile` |

#### Step 3 — Bug fixes during deployment

| Error | Fix |
|---|---|
| `ModuleNotFoundError: No module named 'uvicorn'` | Changed `pip install --user` → `pip install --prefix=/install`, then `COPY /install /usr/local` so non-root `agent` user can import packages |
| `AttributeError: MutableHeaders has no pop()` | Replaced `response.headers.pop("server", None)` with `if "server" in response.headers: del response.headers["server"]` |

#### Step 4 — Auto-deploy

Render watches the `main` branch. Every `git push origin main` triggers a zero-downtime rebuild automatically.

**Live deployment verification:**

```bash
# Root → service info
curl https://day12-ha-tang-cloud-va-deployment-synm.onrender.com/
# {"app":"Production AI Agent","version":"1.0.0","environment":"development",
#  "endpoints":{"ask":"POST /ask (requires X-API-Key)","health":"GET /health","ready":"GET /ready"}}

# Health check → 200 OK ✅
curl https://day12-ha-tang-cloud-va-deployment-synm.onrender.com/health
# {"status":"ok","version":"1.0.0","uptime_seconds":480.7,"total_requests":9,
#  "checks":{"llm":"openai"},"timestamp":"2026-04-17T10:48:36.867131+00:00"}

# Readiness probe → ready ✅
curl https://day12-ha-tang-cloud-va-deployment-synm.onrender.com/ready
# {"ready":true}

# Ask the agent (replace with your generated AGENT_API_KEY from Render dashboard)
curl -X POST https://day12-ha-tang-cloud-va-deployment-synm.onrender.com/ask \
  -H "Content-Type: application/json" \
  -H "X-API-Key: <AGENT_API_KEY>" \
  -d '{"question": "What is Docker?"}'
# {"question":"What is Docker?","answer":"Container là cách đóng gói app...","model":"gpt-4o-mini",...}
```

**Why Render for this lab:**
- Native Docker runtime — no vendor-specific config needed
- `render.yaml` as Infrastructure as Code — reproducible deploys
- Free tier with automatic HTTPS and custom domains
- `autoDeploy: true` — every `git push` redeploys automatically
- Singapore region — low latency for Southeast Asia users

---

### Exercise 3.2 — Render Infrastructure as Code

**`render.yaml`** (repo root) defines the full service as code:
- Web service with `region: singapore`
- `dockerfilePath: ./06-lab-complete/Dockerfile` + `dockerContext: .` for monorepo support
- `healthCheckPath: /health` — Render restarts container if health check fails
- `autoDeploy: true` — continuous deployment on every push to `main`
- `generateValue: true` for `AGENT_API_KEY` and `JWT_SECRET` — Render generates cryptographically secure secrets, never stored in git

---

## Part 4: API Security

### Exercise 4.1 — API Key Authentication

**Implementation** (`04-api-gateway/develop/app.py`):

```python
API_KEY = os.getenv("AGENT_API_KEY", "demo-key-change-in-production")
api_key_header = APIKeyHeader(name="X-API-Key", auto_error=False)

def verify_api_key(api_key: str = Security(api_key_header)) -> str:
    if not api_key:
        raise HTTPException(status_code=401, detail="Missing API key...")
    if api_key != API_KEY:
        raise HTTPException(status_code=403, detail="Invalid API key.")
    return api_key
```

**Test outputs:**

```bash
# ✅ With valid key → 200
curl -H "X-API-Key: demo-key-change-in-production" \
     -X POST -H "Content-Type: application/json" \
     -d '{"question":"hello"}' \
     http://localhost:8000/ask
# Response: {"question":"hello","answer":"..."}

# ❌ No key → 401
curl -X POST http://localhost:8000/ask -d '{"question":"hello"}'
# Response: {"detail":"Missing API key. Include header: X-API-Key: <your-key>"}

# ❌ Wrong key → 403
curl -H "X-API-Key: wrong-key" -X POST http://localhost:8000/ask
# Response: {"detail":"Invalid API key."}
```

---

### Exercise 4.2 — JWT Authentication

**Implementation** (`04-api-gateway/production/auth.py`):

```python
# Get token
curl -X POST http://localhost:8000/auth/token \
  -H "Content-Type: application/json" \
  -d '{"username":"student","password":"demo123"}'
# Response: {"access_token":"eyJ...","token_type":"bearer","expires_in":3600}

# Use token
curl -H "Authorization: Bearer eyJ..." http://localhost:8000/ask \
  -X POST -d '{"question":"hello"}'
```

**Token expiry:** 60 minutes (`ACCESS_TOKEN_EXPIRE_MINUTES = 60`)  
**Demo users:**
- `student` / `demo123` — role: `user`, limit: 50 req/day
- `teacher` / `teach456` — role: `admin`, limit: 1000 req/day

---

### Exercise 4.3 — Rate Limiting

**Implementation** (`04-api-gateway/production/rate_limiter.py`):

- **Algorithm:** Sliding Window Counter
- **User tier:** 10 requests / 60 seconds
- **Admin tier:** 100 requests / 60 seconds

```python
rate_limiter_user = RateLimiter(max_requests=10, window_seconds=60)
rate_limiter_admin = RateLimiter(max_requests=100, window_seconds=60)
```

**Test output (after 11th request):**

```bash
# 429 Too Many Requests
{
  "detail": {
    "error": "Rate limit exceeded",
    "limit": 10,
    "window_seconds": 60,
    "retry_after_seconds": 47
  }
}
# Headers:
# X-RateLimit-Limit: 10
# X-RateLimit-Remaining: 0
# Retry-After: 47
```

**How it works:** Each user has a deque of timestamps. On each request, timestamps older than 60 seconds are removed. If `len(deque) >= 10`, raise 429. Otherwise append current timestamp and allow the request.

---

### Exercise 4.4 — Cost Guard

**Implementation** (`04-api-gateway/production/cost_guard.py`):

```python
cost_guard = CostGuard(
    daily_budget_usd=1.0,        # $1/day per user
    global_daily_budget_usd=10.0  # $10/day total
)
```

**Pricing model:**
- Input tokens: $0.15 / 1M tokens (`PRICE_PER_1K_INPUT_TOKENS = 0.00015`)
- Output tokens: $0.60 / 1M tokens (`PRICE_PER_1K_OUTPUT_TOKENS = 0.0006`)

**Behavior:**
- Warns at 80% budget usage (logs warning)
- Returns **402 Payment Required** when per-user daily budget exceeded
- Returns **503 Service Unavailable** when global daily budget exceeded
- Resets automatically at midnight UTC (tracked by date string `%Y-%m-%d`)

**Security flow:**
```
Request
  → Auth check       → 401 if missing/invalid key
  → Rate limit check → 429 if exceeded
  → Input validation → 422 if malformed
  → Cost check       → 402/503 if over budget
  → Agent call       → 200 OK
```

---

## Part 5: Scaling & Reliability

### Exercise 5.1 — Health Check Endpoints

**Liveness probe** (`GET /health`) — "Is the process alive?"

```python
@app.get("/health")
def health():
    uptime = round(time.time() - START_TIME, 1)
    checks = {}
    # Check memory with psutil
    mem = psutil.virtual_memory()
    checks["memory"] = {"status": "ok" if mem.percent < 90 else "degraded", ...}
    return {"status": "ok", "uptime_seconds": uptime, "checks": checks}
```

**Readiness probe** (`GET /ready`) — "Is the process ready to accept traffic?"

```python
@app.get("/ready")
def ready():
    if not _is_ready:            # False during startup and shutdown
        raise HTTPException(503, "Agent not ready")
    return {"ready": True, "in_flight_requests": _in_flight_requests}
```

**Difference:**
- `/health` → 200 as long as process is running (even if warming up)
- `/ready` → 503 during startup/shutdown, ensuring load balancer only sends traffic to fully-ready instances

---

### Exercise 5.2 — Graceful Shutdown

**Implementation** (`05-scaling-reliability/develop/app.py`):

```python
@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup
    _is_ready = True
    yield
    # Shutdown
    _is_ready = False          # Stop accepting new requests via /ready
    timeout, elapsed = 30, 0
    while _in_flight_requests > 0 and elapsed < timeout:
        time.sleep(1)          # Wait for in-flight requests (max 30s)
        elapsed += 1

signal.signal(signal.SIGTERM, handle_sigterm)   # Catch platform shutdown signal
```

**Shutdown sequence:**
1. Platform sends `SIGTERM` to container
2. `_is_ready = False` → load balancer stops routing new traffic
3. App waits up to 30 seconds for in-flight requests to finish
4. Clean exit — no requests are killed mid-flight

---

### Exercise 5.3 — Stateless Design

**Anti-pattern (broken under scale):**
```python
conversation_history = {}   # Stored in process memory

# Instance 1 handles request 1 → saves to conversation_history
# Instance 2 handles request 2 → conversation_history is EMPTY → context lost!
```

**Correct approach** (`05-scaling-reliability/production/app.py`):
```python
def append_to_history(session_id: str, role: str, content: str):
    session = load_session(session_id)    # Read from Redis
    history = session.get("history", [])
    history.append({"role": role, "content": content, "timestamp": ...})
    if len(history) > 20:
        history = history[-20:]           # Keep last 10 turns
    save_session(session_id, session)     # Write back to Redis with TTL

def save_session(session_id, data, ttl_seconds=3600):
    _redis.setex(f"session:{session_id}", ttl_seconds, json.dumps(data))
```

**Each response includes `served_by: INSTANCE_ID`** — making it visible that different instances serve the same session without losing history.

---

### Exercise 5.4 — Load Balancing with Multiple Instances

**`05-scaling-reliability/production/docker-compose.yml`:**

```yaml
agent:
  deploy:
    replicas: 3
  resources:
    limits:
      cpus: "0.5"
      memory: 256M
```

```bash
# Start 3 agent instances
docker compose up --scale agent=3

# Test: requests are distributed across instances
curl http://localhost/health   # nginx routes to any of the 3 agents
```

---

### Exercise 5.5 — Stateless Test Results

**`05-scaling-reliability/production/test_stateless.py`** creates a session, sends 5 requests, and records which `instance_id` served each:

```
Session: abc-123-def
Turn 1: served_by=instance-a3f2c1  → "What is Docker?"
Turn 2: served_by=instance-b7e9d4  → "Tell me more"
Turn 3: served_by=instance-a3f2c1  → "How does it help?"
Turn 4: served_by=instance-c1a8f3  → "Give an example"
Turn 5: served_by=instance-b7e9d4  → "Summary please"

✅ History preserved across all 3 instances (5 messages in Redis)
```

Different instances served the conversation — conversation history survived because state lives in **Redis**, not in process memory.

---

## Summary

| Part | Key Deliverable | Status |
|---|---|---|
| 1 | 6 anti-patterns identified + comparison table | ✅ |
| 2 | Dockerfile analysis + multi-stage build explanation | ✅ |
| 3 | Render deployment — live at https://day12-ha-tang-cloud-va-deployment-synm.onrender.com | ✅ |
| 4 | API Key auth + JWT + Rate limiting + Cost guard | ✅ |
| 5 | Health checks + graceful shutdown + stateless Redis design | ✅ |
