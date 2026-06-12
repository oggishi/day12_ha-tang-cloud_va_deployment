# Day 12 Lab - Mission Answers

**Student Name:** Nguyen Thi Bao Tran
**Student ID:** 2A202600917
**Date:** 2026-06-12

> File này chứa đầy đủ code snippet, test output, log thực tế và bảng so sánh cho từng exercise. Bản gốc (kèm giải thích chi tiết hơn theo từng bước) nằm trong [CODE_LAB.md](CODE_LAB.md), bên trong các block `<details><summary>Trả lời</summary>...</details>`.

---

## Part 1: Localhost vs Production

### Exercise 1.1: Anti-patterns found

File `01-localhost-vs-production/develop/app.py`:

```python
"""
❌ BASIC — Agent "Kiểu Localhost" (Anti-patterns)
"""
import os

from fastapi import FastAPI
import uvicorn
from utils.mock_llm import ask

app = FastAPI(title="My Agent")

# ❌ Vấn đề 1: API key hardcode trong code
OPENAI_API_KEY = "sk-hardcoded-fake-key-never-do-this"
DATABASE_URL = "postgresql://admin:password123@localhost:5432/mydb"

# ❌ Vấn đề 2: Không có config management
DEBUG = True
MAX_TOKENS = 500


@app.get("/")
def home():
    return {"message": "Hello! Agent is running on my machine :)"}


@app.post("/ask")
def ask_agent(question: str):
    # ❌ Vấn đề 3: Print thay vì proper logging
    print(f"[DEBUG] Got question: {question}")
    print(f"[DEBUG] Using key: {OPENAI_API_KEY}")  # ❌ log ra secret!

    response = ask(question)

    print(f"[DEBUG] Response: {response}")
    return {"answer": response}


# ❌ Vấn đề 4: Không có health check endpoint
# ❌ Vấn đề 5: Port cố định — không đọc từ environment
if __name__ == "__main__":
    uvicorn.run("app:app", host="localhost", port=8000, reload=True)
```

**5 vấn đề tìm được:**

1. **API key & DB credentials hardcode trong code** — `OPENAI_API_KEY = "sk-hardcoded-fake-key-never-do-this"`, `DATABASE_URL = "postgresql://admin:password123@..."`. Nếu push lên GitHub, secret bị lộ ngay lập tức (và lộ vĩnh viễn trong git history).
2. **Không có config management** — `DEBUG = True`, `MAX_TOKENS = 500` hardcode trực tiếp trong code, không đọc từ environment variable → muốn đổi config phải sửa code và deploy lại.
3. **`print()` thay vì logging, kể cả log ra secret** — `print(f"[DEBUG] Using key: {OPENAI_API_KEY}")` log thẳng API key ra console/log file. Đây là lỗ hổng bảo mật nghiêm trọng vì log thường được lưu lại lâu dài và nhiều người có quyền đọc.
4. **Không có health check endpoint** — platform (Railway/Render/K8s) không có cách nào biết agent còn sống để tự restart khi crash.
5. **Port cố định & `host="localhost"`, `reload=True`** — `uvicorn.run("app:app", host="localhost", port=8000, reload=True)`. `localhost` chỉ nhận traffic nội bộ container → không hoạt động trên cloud (cloud platform inject `PORT` qua env var); `reload=True` là chế độ debug, tốn resource và không nên chạy trong production.

### Exercise 1.2: Chạy basic version

```bash
cd 01-localhost-vs-production/develop
pip install -r requirements.txt
python app.py
```

```bash
curl http://localhost:8000/ask -X POST \
  -H "Content-Type: application/json" \
  -d '{"question": "Hello"}'
```

**Kết quả:** server chạy thành công trên `localhost:8000`, request trả về `{"answer": "..."}` (mock LLM) — **chạy được nhưng không production-ready**, vì cả 5 vấn đề ở Exercise 1.1 đều còn nguyên: secrets lộ trong code, không health check, log ra secret, không graceful shutdown, không đọc `PORT` từ env (sẽ fail ngay khi deploy lên Railway/Render vì các platform này gán `PORT` động).

### Exercise 1.3: Comparison table

| Feature | Basic (develop) | Advanced (production) | Tại sao quan trọng? |
|---------|-------|----------|---------------------|
| Config | Hardcode (`OPENAI_API_KEY`, `DATABASE_URL` viết cứng trong code) | Env vars qua `Settings` dataclass (`config.py`), đọc từ `.env` | Hardcode secrets dễ bị lộ khi push lên GitHub; env vars cho phép đổi config theo từng môi trường (dev/staging/prod) mà không sửa code |
| Host/Port binding | `host="localhost"`, `port=8000` cố định | `host=settings.host` (0.0.0.0), `port=settings.port` từ env `PORT` | `localhost` chỉ nhận traffic nội bộ container → không hoạt động trên cloud; cloud platform (Railway/Render) tự inject `PORT` |
| Health check | Không có | `/health` trả về status, uptime, version | Platform cần endpoint này để biết container còn sống, nếu fail sẽ tự restart |
| Readiness check | Không có | `/ready` trả 503 nếu chưa sẵn sàng (dùng `is_ready` flag) | Load balancer chỉ route traffic vào instance đã sẵn sàng, tránh request bị lỗi khi app vừa start |
| Logging | `print()`, kể cả log ra API key (`OPENAI_API_KEY`) | Structured JSON logging qua `logging` module, không log secrets | `print()` không có level/timestamp, khó parse; log secret ra console là lỗ hổng bảo mật nghiêm trọng |
| Shutdown | Đột ngột (không có signal handler, `reload=True`) | Graceful qua `lifespan` context + `SIGTERM` handler — hoàn thành request đang chạy trước khi tắt | Khi container bị orchestrator kill (deploy mới, scale down), graceful shutdown tránh làm gián đoạn request của user |
| Debug mode | `DEBUG = True`, `reload=True` luôn bật | `debug=settings.debug` từ env, mặc định `false` | Debug/reload mode trong production tốn tài nguyên và có thể lộ thông tin nhạy cảm qua error pages |
| CORS | Không cấu hình | `CORSMiddleware` với `allowed_origins` từ env | Kiểm soát domain nào được gọi API, tránh bị abuse từ frontend không được phép |
| Metrics | Không có | `/metrics` endpoint (uptime, env, version) | Cho phép Prometheus hoặc monitoring tool scrape thông tin vận hành |
| Validation config | Không có | `Settings.validate()` — raise lỗi ngay nếu thiếu `AGENT_API_KEY` ở production | "Fail fast" — phát hiện lỗi config lúc khởi động, không phải lúc user gọi API mới crash |

### Checkpoint 1
- [x] Hiểu tại sao hardcode secrets là nguy hiểm
- [x] Biết cách dùng environment variables
- [x] Hiểu vai trò của health check endpoint
- [x] Biết graceful shutdown là gì

---

## Part 2: Docker Containerization

### Exercise 2.1: Dockerfile cơ bản

`02-docker/develop/Dockerfile`:

```dockerfile
FROM python:3.11

WORKDIR /app

COPY 02-docker/develop/requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY 02-docker/develop/app.py .

RUN mkdir -p utils
COPY utils/mock_llm.py utils/

EXPOSE 8000

CMD ["python", "app.py"]
```

**Trả lời:**

1. **Base image:** `python:3.11` — full Python distribution (~1 GB), bao gồm đầy đủ build tools, dễ debug nhưng image lớn (so với `python:3.11-slim` hoặc `python:3.11-alpine`).

2. **Working directory:** `/app` (set bằng `WORKDIR /app`) — mọi lệnh `COPY`, `RUN`, `CMD` sau đó sẽ chạy tương đối với thư mục này trong container.

3. **Tại sao COPY requirements.txt trước:** Để tận dụng **Docker layer caching**. Mỗi instruction trong Dockerfile tạo một layer; Docker cache layer nếu input không đổi. Nếu copy hết source code rồi mới `pip install`, thì chỉ cần sửa 1 dòng code là cache của layer `pip install` bị invalidate → phải install lại toàn bộ dependencies mỗi lần build (rất chậm). Bằng cách copy `requirements.txt` trước và `RUN pip install` ngay sau đó, layer install dependencies chỉ rebuild khi `requirements.txt` thay đổi — code thay đổi không ảnh hưởng tới layer này.

4. **CMD vs ENTRYPOINT:**
   - `CMD` định nghĩa lệnh mặc định khi container start, nhưng **có thể bị override** dễ dàng bằng cách truyền argument khi `docker run` (ví dụ `docker run image python other.py` sẽ thay thế toàn bộ `CMD`).
   - `ENTRYPOINT` định nghĩa lệnh **cố định** sẽ luôn chạy; argument truyền vào `docker run` sẽ được **append** vào sau `ENTRYPOINT` (không override).
   - Dockerfile này dùng `CMD ["python", "app.py"]` — phù hợp vì cho phép dễ dàng override khi debug (ví dụ `docker run image bash` để vào shell). Nếu muốn container luôn chạy đúng 1 chương trình bất kể argument, dùng `ENTRYPOINT`. Pattern phổ biến là kết hợp cả hai: `ENTRYPOINT` cho binary chính, `CMD` cho default arguments.

### Exercise 2.2: Build và run

```bash
docker build -f 02-docker/develop/Dockerfile -t my-agent:develop .
docker run -p 8000:8000 my-agent:develop

curl http://localhost:8000/ask -X POST \
  -H "Content-Type: application/json" \
  -d '{"question": "What is Docker?"}'
```

**Test output thực tế:**
```bash
$ curl -X POST "http://localhost:8000/ask?question=What%20is%20Docker%3F"
{"answer":"Container là cách đóng gói app để chạy ở mọi nơi. Build once, run anywhere!"}
```
(lưu ý: endpoint `/ask` của bản develop nhận `question` qua **query param**, không phải JSON body)

**Image size:**
```bash
$ docker images my-agent:develop
REPOSITORY    TAG       SIZE
my-agent      develop   424MB   (content) / 1.66GB (disk, gồm cache layers)
```

→ Image dùng base `python:3.11` (full distro) nên rất nặng.

### Exercise 2.3: Multi-stage build

`02-docker/production/Dockerfile`:

```dockerfile
# ──────────────────────────────────────────────────────────
# STAGE 1: Builder — cài đặt tất cả dependencies
# Image này KHÔNG được dùng để deploy
# ──────────────────────────────────────────────────────────
FROM python:3.11-slim AS builder

WORKDIR /app

RUN apt-get update && apt-get install -y \
    gcc \
    libpq-dev \
    && rm -rf /var/lib/apt/lists/*

COPY 02-docker/production/requirements.txt .
RUN pip install --no-cache-dir --user -r requirements.txt


# ──────────────────────────────────────────────────────────
# STAGE 2: Runtime — chỉ copy những gì cần để CHẠY
# ──────────────────────────────────────────────────────────
FROM python:3.11-slim AS runtime

RUN groupadd -r appuser && useradd -r -g appuser appuser

WORKDIR /app

COPY --from=builder /root/.local /home/appuser/.local

COPY 02-docker/production/main.py .

RUN mkdir -p /app/utils
COPY utils/mock_llm.py /app/utils/mock_llm.py

RUN chown -R appuser:appuser /app

USER appuser

ENV PATH=/home/appuser/.local/bin:$PATH
ENV PYTHONPATH=/app

EXPOSE 8000

HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8000/health')" || exit 1

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000", "--workers", "2"]
```

**Trả lời:**

- **Stage 1 (`builder`):** Base `python:3.11-slim`, cài build tools (`gcc`, `libpq-dev`) và `pip install --user -r requirements.txt` vào `/root/.local`. Image này KHÔNG dùng để deploy — chỉ để compile dependencies.
- **Stage 2 (`runtime`):** Base `python:3.11-slim` sạch, tạo non-root user `appuser`, `COPY --from=builder /root/.local /home/appuser/.local` — chỉ lấy package đã build sẵn, không có gcc/build tools/apt cache. Thêm `HEALTHCHECK`, chạy `uvicorn` với 2 workers.
- **Tại sao image nhỏ hơn:** Stage 1 chứa toolchain build (gcc, headers, apt lists...) chỉ cần lúc compile, không cần lúc runtime. Multi-stage cho phép loại bỏ hoàn toàn layer đó khỏi image cuối — chỉ giữ Python + site-packages cần thiết để chạy app.

**Kết quả build và so sánh thực tế:**

```bash
$ docker build -t my-agent:advanced .
$ docker images | grep my-agent
my-agent   advanced   56.6MB   (content) / 236MB (disk)
my-agent   develop    424MB    (content) / 1.66GB (disk)
```

| Image | Content size | Disk usage |
|---|---|---|
| `my-agent:develop` (single-stage, `python:3.11`) | 424 MB | 1.66 GB |
| `my-agent:advanced` (multi-stage, `python:3.11-slim`) | 56.6 MB | 236 MB |

→ Giảm ~**7.5x** content size (~87%).

**Test container advanced:**
```bash
$ docker run -p 8000:8000 -e ENVIRONMENT=production my-agent:advanced

$ curl http://localhost:8000/health
{"status":"ok","version":"2.0.0","uptime_seconds":12.3,"timestamp":"2026-06-12T..."}

$ curl -X POST http://localhost:8000/ask -H "Content-Type: application/json" -d '{"question": "What is multi-stage build?"}'
{"answer":"Đây là câu trả lời từ AI agent (mock). Trong production, đây sẽ là response từ OpenAI/Anthropic."}
```

Lưu ý: khác với develop (`/ask` nhận `question` qua **query param**), advanced nhận `question` qua **JSON body**.

### Exercise 2.4: Docker Compose stack (bonus)

`02-docker/production/docker-compose.yml` định nghĩa 4 services trong network `internal` (bridge): `nginx`, `agent`, `redis`, `qdrant`.

```bash
docker compose up -d --build
```

**Architecture:**

```
                         ┌─────────────┐
                         │   Client    │
                         └──────┬──────┘
                                │ :80 / :443
                                ▼
                       ┌─────────────────┐
                       │  nginx (alpine)  │  ← reverse proxy + load balancer
                       │  rate limiting   │     + security headers
                       └────────┬─────────┘
                                │ proxy_pass → agent:8000
                                ▼
                       ┌─────────────────┐
                       │  agent (FastAPI) │  ← multi-stage build, target "runtime"
                       │  ENVIRONMENT=    │     không expose port ra host
                       │    staging       │
                       └───┬─────────┬────┘
                  REDIS_URL │         │ QDRANT_URL
                            ▼         ▼
                     ┌─────────┐  ┌──────────┐
                     │  redis  │  │  qdrant   │
                     │ (cache, │  │ (vector   │
                     │ rate    │  │ database  │
                     │ limit)  │  │ for RAG)  │
                     └─────────┘  └──────────┘
```

**Cách communicate:**
- Chỉ **nginx** expose port ra host (`80:80`, `443:443`) — agent/redis/qdrant chỉ giao tiếp nội bộ qua network `internal`, không truy cập trực tiếp từ ngoài.
- Client gọi `http://localhost/...` → Nginx nhận, áp rate limit (`10r/s`, burst 20) + thêm security headers → `proxy_pass` sang `agent:8000` (DNS resolve qua Docker network).
- Agent kết nối `redis:6379` và `qdrant:6333` qua tên service (Docker DNS) — dùng cho cache/session và vector search.
- `depends_on` với `condition: service_healthy` đảm bảo agent chỉ start sau khi redis & qdrant pass healthcheck.

**Kết quả chạy thực tế** (`docker compose up -d --build`):

```
NAME                  STATUS                  PORTS
production-agent-1    Up (healthy)            (không expose port — chỉ nội bộ)
production-nginx-1    Up                      0.0.0.0:80->80, 0.0.0.0:443->443
production-qdrant-1   Up (healthy)            6333-6334/tcp (nội bộ)
production-redis-1    Up (healthy)            6379/tcp (nội bộ)
```

**Test:**
```bash
$ curl http://localhost/health
{"status":"ok","uptime_seconds":18.2,"version":"2.0.0","timestamp":"..."}

$ curl -X POST http://localhost/ask -H "Content-Type: application/json" -d '{"question": "Explain microservices"}'
{"answer":"Đây là câu trả lời từ AI agent (mock)..."}
```

Log Nginx ghi lại request `"POST /ask HTTP/1.1" 200`, log agent cho thấy request được proxy đúng và xử lý JSON logging.

### Checkpoint 2
- [x] Hiểu Dockerfile cơ bản và multi-stage build
- [x] Build và chạy được container
- [x] So sánh image size single-stage vs multi-stage
- [x] Hiểu docker-compose multi-service stack

---

## Part 3: Cloud Deployment

### Exercise 3.1: Deploy Railway

**URL:** https://secure-balance-production-6666.up.railway.app/

**Steps thực hiện:**
```bash
cd 03-cloud-deployment/railway
npm i -g @railway/cli
railway login
railway init                                  # auto-generate project name "secure-balance"
railway variables set PORT=8000
railway variables set AGENT_API_KEY=my-secret-key
railway up
railway domain
```

`railway.toml`:
```toml
[build]
builder = "NIXPACKS"

[deploy]
startCommand = "uvicorn app:app --host 0.0.0.0 --port $PORT"
healthcheckPath = "/health"
healthcheckTimeout = 30
restartPolicyType = "ON_FAILURE"
restartPolicyMaxRetries = 3
```

**Test trên public URL (live, re-test 2026-06-12):**
```bash
$ curl https://secure-balance-production-6666.up.railway.app/health
{"status":"ok","uptime_seconds":4919.6,"platform":"Railway","timestamp":"2026-06-12T10:04:28.116812+00:00"}

$ curl -X POST https://secure-balance-production-6666.up.railway.app/ask \
  -H "Content-Type: application/json" \
  -d '{"question": "What is deployment?"}'
{"question":"What is deployment?","answer":"Deployment là quá trình đưa code từ máy bạn lên server để người khác dùng được.","platform":"Railway"}
```

### Exercise 3.2: Deploy Render

**URL:** https://ai-agent-7keu.onrender.com

**Steps thực hiện:**
1. Push code lên GitHub
2. Render Dashboard → New → Blueprint, Blueprint Path = `03-cloud-deployment/render/render.yaml`
3. Render đọc `render.yaml`, tạo cả web service + Redis add-on

`render.yaml` (sau khi fix 2 lỗi build):
```yaml
services:
  - type: web
    name: ai-agent
    runtime: python
    region: singapore
    plan: free
    rootDir: 03-cloud-deployment/render     # ← fix: "requirements.txt not found"
    buildCommand: pip install -r requirements.txt
    startCommand: uvicorn app:app --host 0.0.0.0 --port $PORT
    healthCheckPath: /health
    autoDeploy: true
    envVars:
      - key: ENVIRONMENT
        value: production
      - key: PYTHON_VERSION
        value: 3.11.0
      - key: OPENAI_API_KEY
        sync: false
      - key: AGENT_API_KEY
        generateValue: true

  - type: redis
    name: agent-cache
    plan: free
    maxmemoryPolicy: allkeys-lru
    ipAllowList:                            # ← fix: "must specify IP allow list"
      - source: 0.0.0.0/0
        description: everywhere
```

**Test trên public URL (live, re-test 2026-06-12):**
```bash
$ curl -i https://ai-agent-7keu.onrender.com/health
HTTP/1.1 200 OK
Content-Type: application/json
...
{"status":"ok","uptime_seconds":83.4,"platform":"Render","timestamp":"2026-06-12T10:06:18.526115+00:00"}

$ curl -X POST https://ai-agent-7keu.onrender.com/ask \
  -H "Content-Type: application/json" \
  -d '{"question": "What is Docker?"}'
{"question":"What is Docker?","answer":"Container là cách đóng gói app để chạy ở mọi nơi. Build once, run anywhere!","platform":"Render"}
```

**So sánh `render.yaml` vs `railway.toml`:**

| Khía cạnh | `railway.toml` | `render.yaml` |
|---|---|---|
| **Phạm vi config** | Chỉ 1 service (agent) | **Multi-service** — khai báo cả `web` service và `redis` add-on trong cùng 1 file (Infrastructure as Code đầy đủ hơn) |
| **Build** | `builder = "NIXPACKS"` — Railway tự detect | `runtime: python` + `buildCommand: pip install -r requirements.txt` — khai báo rõ ràng |
| **Start command** | `startCommand = "uvicorn app:app --host 0.0.0.0 --port $PORT"` | Giống hệt: `startCommand: uvicorn app:app --host 0.0.0.0 --port $PORT` |
| **Health check** | `healthcheckPath = "/health"`, `healthcheckTimeout = 30` | `healthCheckPath: /health` (không có timeout config) |
| **Restart policy** | Khai báo rõ: `restartPolicyType = "ON_FAILURE"`, `restartPolicyMaxRetries = 3` | Không khai báo (Render tự quản lý) |
| **Region** | Không khai báo trong file (chọn qua dashboard/CLI) | `region: singapore` — khai báo trực tiếp trong YAML |
| **Plan/Pricing tier** | Không có trong config | `plan: free` — khai báo ngay trong service |
| **Env vars** | Đặt qua CLI/dashboard, không có trong file | Khai báo cấu trúc trong file: `sync: false` (set thủ công), `generateValue: true` (Render tự sinh secret) |
| **Auto-deploy** | Mặc định khi `railway up` hoặc kết nối GitHub | `autoDeploy: true` — khai báo rõ trong YAML |
| **Trigger deploy** | CLI (`railway up`) — push từ máy local | Chủ yếu qua GitHub push (Blueprint sync) |
| **Security config** | Không có | `ipAllowList` bắt buộc cho Redis — kiểm soát IP nào được kết nối vào database |

**Kết luận:** `render.yaml` mang tính **declarative/IaC** hơn — toàn bộ infrastructure (web service + Redis) được định nghĩa và version-control trong 1 file, Render tự sync khi push GitHub. `railway.toml` tập trung vào **build/deploy behavior của 1 service**, còn infrastructure khác (database, domain, env vars) quản lý qua CLI/dashboard riêng.

### Exercise 3.3: GCP Cloud Run (optional — đọc & phân tích)

**`cloudbuild.yaml` — CI/CD pipeline 4 bước tuần tự:**

```
push code → main branch
        │
        ▼
┌─────────────────┐
│ 1. test          │  python:3.11-slim → pip install + pytest tests/
│    (id: test)    │  Nếu test fail → pipeline dừng, không build/deploy
└────────┬─────────┘
         │ waitFor: [test]
         ▼
┌─────────────────┐
│ 2. build         │  docker build, tag theo $COMMIT_SHA và "latest"
│    (id: build)   │  --cache-from latest → tận dụng layer cache, build nhanh hơn
└────────┬─────────┘
         │ waitFor: [build]
         ▼
┌─────────────────┐
│ 3. push          │  push cả 2 tags lên Google Container Registry (gcr.io)
│    (id: push)    │
└────────┬─────────┘
         │ waitFor: [push]
         ▼
┌─────────────────┐
│ 4. deploy        │  gcloud run deploy — deploy image theo $COMMIT_SHA (immutable)
│    (id: deploy)  │  lên Cloud Run, region asia-southeast1
└─────────────────┘
```

Mỗi step có `waitFor` rõ ràng → đảm bảo thứ tự, và nếu 1 step fail thì các step sau không chạy ("fail fast").

**Cấu hình deploy đáng chú ý trong `cloudbuild.yaml`:**
- `--allow-unauthenticated`: endpoint public, không cần GCP IAM token
- `--min-instances=1`: giữ ít nhất 1 instance luôn chạy → tránh cold start (đánh đổi: tốn phí dù không có traffic)
- `--max-instances=10`: giới hạn scale để tránh chi phí vượt kiểm soát
- `--set-secrets=OPENAI_API_KEY=openai-key:latest`: lấy secret từ **Secret Manager**, không hardcode trong YAML hay image

**`service.yaml` — Knative spec:**
- **Autoscaling:** `minScale: 1`, `maxScale: 10`, `containerConcurrency: 80`
- **Resources:** request 0.5 CPU/256Mi, limit 1 CPU/512Mi
- **Secrets:** `OPENAI_API_KEY` và `AGENT_API_KEY` dùng `valueFrom.secretKeyRef` trỏ tới Secret Manager
- **2 loại health probe:**
  - `livenessProbe` → `/health`, check mỗi 30s sau 10s khởi động — Cloud Run **restart container** nếu fail
  - `startupProbe` → `/ready`, check mỗi 3s, tối đa 10 lần thất bại — đảm bảo container có thời gian khởi động trước khi nhận traffic

**So với Railway/Render:** Cloud Run yêu cầu setup phức tạp hơn (Docker image, Container Registry, Secret Manager, IAM) nhưng đổi lại có autoscaling chi tiết theo concurrency, immutable deploys theo commit SHA (dễ rollback), và CI/CD pipeline tích hợp test trước khi deploy.

### Checkpoint 3
- [x] Deploy thành công lên ít nhất 1 platform (Railway + Render, cả 2)
- [x] Có public URL hoạt động (cả 2 URL test live OK)
- [x] Hiểu cách set environment variables trên cloud
- [x] Biết cách xem logs

---

## Part 4: API Security

### Exercise 4.1: API Key authentication

`04-api-gateway/develop/app.py`:

```python
API_KEY = os.getenv("AGENT_API_KEY", "demo-key-change-in-production")
api_key_header = APIKeyHeader(name="X-API-Key", auto_error=False)


def verify_api_key(api_key: str = Security(api_key_header)) -> str:
    if not api_key:
        raise HTTPException(
            status_code=401,
            detail="Missing API key. Include header: X-API-Key: <your-key>",
        )
    if api_key != API_KEY:
        raise HTTPException(status_code=403, detail="Invalid API key.")
    return api_key


@app.post("/ask")
async def ask_agent(
    request: Request,
    _key: str = Depends(verify_api_key),  # ✅ require auth
):
    body = await request.json()
    question = body.get("question", "")
    if not question:
        raise HTTPException(status_code=422, detail="question is required")
    return {"question": question, "answer": ask(question)}


@app.get("/health")
def health():
    """Health check — public (platform cần access)"""
    return {"status": "ok"}
```

**1. API key được check ở đâu?**
Trong dependency function `verify_api_key()` (`app.py:39-54`):
- Đọc header `X-API-Key` qua `APIKeyHeader(name="X-API-Key", auto_error=False)`
- So sánh với `API_KEY = os.getenv("AGENT_API_KEY", "demo-key-change-in-production")`
- Dependency này được inject vào endpoint `/ask` qua `Depends(verify_api_key)` — endpoint `/` và `/health` **không** có dependency này nên public.

**2. Điều gì xảy ra nếu sai key?**
- **Không gửi header** `X-API-Key` → `401 Unauthorized`
- **Gửi key sai** → `403 Forbidden`
- **Key đúng** → `200 OK`, trả về answer từ mock LLM

(Phân biệt rõ 401 = thiếu credential, 403 = có credential nhưng không hợp lệ — đúng chuẩn HTTP semantics.)

**3. Làm sao rotate key?**
Key đọc từ env var `AGENT_API_KEY` (không hardcode) → rotate bằng cách set env var mới trên platform (`railway variables set AGENT_API_KEY=new-key`) rồi restart/redeploy. ⚠️ Hạn chế: chỉ hỗ trợ **1 key tại 1 thời điểm**, không có grace period — để rotate "mềm" (zero-downtime) cần hỗ trợ nhiều key hợp lệ song song (danh sách `VALID_KEYS`).

**Test thực tế:**
```bash
# Không có key
$ curl -X POST http://localhost:8000/ask -H "Content-Type: application/json" -d '{"question": "Hello"}'
HTTP/1.1 401 Unauthorized
{"detail":"Missing API key. Include header: X-API-Key: <your-key>"}

# Sai key
$ curl -X POST http://localhost:8000/ask -H "X-API-Key: wrong-key" -H "Content-Type: application/json" -d '{"question": "Hello"}'
HTTP/1.1 403 Forbidden
{"detail":"Invalid API key."}

# Đúng key
$ curl -X POST http://localhost:8000/ask -H "X-API-Key: demo-key-change-in-production" -H "Content-Type: application/json" -d '{"question": "Hello"}'
HTTP/1.1 200 OK
{"question":"Hello","answer":"Tôi là AI agent được deploy lên cloud. Câu hỏi của bạn đã được nhận."}
```

### Exercise 4.2: JWT authentication (Advanced)

`04-api-gateway/production/auth.py`:

```python
SECRET_KEY = os.getenv("JWT_SECRET", "super-secret-change-in-production-please")
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 60

DEMO_USERS = {
    "student": {"password": "demo123", "role": "user", "daily_limit": 50},
    "teacher": {"password": "teach456", "role": "admin", "daily_limit": 1000},
}

security = HTTPBearer(auto_error=False)


def create_token(username: str, role: str) -> str:
    payload = {
        "sub": username,
        "role": role,
        "iat": datetime.now(timezone.utc),
        "exp": datetime.now(timezone.utc) + timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES),
    }
    return jwt.encode(payload, SECRET_KEY, algorithm=ALGORITHM)


def verify_token(credentials: HTTPAuthorizationCredentials = Security(security)) -> dict:
    if not credentials:
        raise HTTPException(
            status_code=401,
            detail="Authentication required. Include: Authorization: Bearer <token>",
            headers={"WWW-Authenticate": "Bearer"},
        )
    try:
        payload = jwt.decode(credentials.credentials, SECRET_KEY, algorithms=[ALGORITHM])
        return {"username": payload["sub"], "role": payload["role"]}
    except jwt.ExpiredSignatureError:
        raise HTTPException(status_code=401, detail="Token expired. Please login again.")
    except jwt.InvalidTokenError:
        raise HTTPException(status_code=403, detail="Invalid token.")


def authenticate_user(username: str, password: str) -> dict:
    user = DEMO_USERS.get(username)
    if not user or user["password"] != password:
        raise HTTPException(status_code=401, detail="Invalid credentials")
    return {"username": username, "role": user["role"]}
```

⚠️ **Lưu ý:** Endpoint thực tế là **`/auth/token`** (không phải `/token`), demo users là **`student/demo123`** (role `user`) hoặc **`teacher/teach456`** (role `admin`) — không phải `admin/secret` như hướng dẫn ban đầu trong lab.

**JWT flow:**
1. `POST /auth/token` với `{username, password}` → `authenticate_user()` check trong `DEMO_USERS` dict → nếu đúng, `create_token()` tạo JWT chứa **username + role + thời hạn 60 phút**, ký bằng `JWT_SECRET` (HS256).
2. Client gửi token trong header `Authorization: Bearer <token>` cho các request sau.
3. `verify_token()` decode + verify signature bằng `SECRET_KEY` → trả về `{username, role}` để dùng cho rate limiting/cost guard/role check — **stateless**, không cần query DB mỗi request.

**Test thực tế:**
```bash
# 1. Lấy token
$ curl -X POST http://localhost:8000/auth/token -H "Content-Type: application/json" \
  -d '{"username": "student", "password": "demo123"}'
{"access_token":"eyJhbGc...","token_type":"bearer","expires_in_minutes":60,...}

# 2. Dùng token
$ curl -X POST http://localhost:8000/ask -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" -d '{"question": "Explain JWT"}'
HTTP/1.1 200 OK
{"answer":"...","usage":{"requests_remaining":9,"budget_remaining_usd":1.9e-05}}
```

| Trường hợp | HTTP | Response |
|---|---|---|
| Không có token | 401 | `{"detail":"Authentication required. Include: Authorization: Bearer <token>"}` |
| Token sai định dạng/signature | 403 | `{"detail":"Invalid token."}` |
| Token hết hạn | 401 | `{"detail":"Token expired. Please login again."}` |
| Token hợp lệ | 200 | trả answer + `usage` (remaining rate limit, budget còn lại) |

### Exercise 4.3: Rate limiting

`04-api-gateway/production/rate_limiter.py`:

```python
class RateLimiter:
    def __init__(self, max_requests: int = 10, window_seconds: int = 60):
        self.max_requests = max_requests
        self.window_seconds = window_seconds
        self._windows: dict[str, deque] = defaultdict(deque)

    def check(self, user_id: str) -> dict:
        now = time.time()
        window = self._windows[user_id]

        # Loại bỏ timestamps cũ (ngoài window)
        while window and window[0] < now - self.window_seconds:
            window.popleft()

        remaining = self.max_requests - len(window)
        reset_at = int(now) + self.window_seconds

        if len(window) >= self.max_requests:
            oldest = window[0]
            retry_after = int(oldest + self.window_seconds - now) + 1
            raise HTTPException(
                status_code=429,
                detail={
                    "error": "Rate limit exceeded",
                    "limit": self.max_requests,
                    "window_seconds": self.window_seconds,
                    "retry_after_seconds": retry_after,
                },
                headers={
                    "X-RateLimit-Limit": str(self.max_requests),
                    "X-RateLimit-Remaining": "0",
                    "X-RateLimit-Reset": str(reset_at),
                    "Retry-After": str(retry_after),
                },
            )

        window.append(now)
        return {"limit": self.max_requests, "remaining": remaining - 1, "reset_at": reset_at}


# Singleton instances cho các tiers khác nhau
rate_limiter_user = RateLimiter(max_requests=10, window_seconds=60)    # User: 10 req/phút
rate_limiter_admin = RateLimiter(max_requests=100, window_seconds=60)  # Admin: 100 req/phút
```

**1. Algorithm:** **Sliding Window Counter** (không phải token bucket). Mỗi user có 1 `deque` chứa timestamp các request gần đây. Mỗi lần check:
- Loại bỏ timestamp cũ hơn `window_seconds` (60s) khỏi đầu deque
- Nếu số timestamp còn lại ≥ `max_requests` → raise `429`
- Ngược lại → append timestamp hiện tại, cho qua

Khác với fixed window (reset theo mốc thời gian cố định, có thể bị burst ở ranh giới window), sliding window tính chính xác "N request trong 60s gần nhất tính từ bây giờ".

**2. Limit:**
- `rate_limiter_user` = **10 requests/phút** (role `user`, ví dụ `student`)
- `rate_limiter_admin` = **100 requests/phút** (role `admin`, ví dụ `teacher`)

**3. Bypass cho admin:** Không phải "bypass" hoàn toàn — `app.py:141` chọn limiter theo role:
```python
limiter = rate_limiter_admin if role == "admin" else rate_limiter_user
```
Admin vẫn bị giới hạn, nhưng với ngưỡng cao hơn (100 vs 10 req/phút) — **tiered rate limiting** theo role, lấy từ JWT payload.

**Test thực tế** — gọi `/ask` 13 lần liên tục với user `student` (limit 10/phút):

```bash
for i in {1..13}; do
  curl -s -o /dev/null -w "%{http_code} " http://localhost:8000/ask \
    -X POST -H "Authorization: Bearer $TOKEN" \
    -H "Content-Type: application/json" -d '{"question": "Test '$i'"}'
done
```

| Request | HTTP | `requests_remaining` |
|---|---|---|
| 1–10 | 200 | 9 → 0 |
| 11–13 | **429** | — |

Response khi vượt limit:
```json
{"detail":{"error":"Rate limit exceeded","limit":10,"window_seconds":60,"retry_after_seconds":56}}
```

Headers trả về:
```
x-ratelimit-limit: 10
x-ratelimit-remaining: 0
x-ratelimit-reset: <unix_timestamp>
retry-after: 44
```
→ Client có thể dùng `Retry-After` để biết chính xác khi nào nên thử lại, tuân theo chuẩn HTTP cho rate limiting.

### Exercise 4.4: Cost guard

`04-api-gateway/production/cost_guard.py`:

```python
PRICE_PER_1K_INPUT_TOKENS = 0.00015   # GPT-4o-mini: $0.15/1M input
PRICE_PER_1K_OUTPUT_TOKENS = 0.0006   # GPT-4o-mini: $0.60/1M output


@dataclass
class UsageRecord:
    user_id: str
    input_tokens: int = 0
    output_tokens: int = 0
    request_count: int = 0
    day: str = field(default_factory=lambda: time.strftime("%Y-%m-%d"))

    @property
    def total_cost_usd(self) -> float:
        input_cost = (self.input_tokens / 1000) * PRICE_PER_1K_INPUT_TOKENS
        output_cost = (self.output_tokens / 1000) * PRICE_PER_1K_OUTPUT_TOKENS
        return round(input_cost + output_cost, 6)


class CostGuard:
    def __init__(self, daily_budget_usd=1.0, global_daily_budget_usd=10.0, warn_at_pct=0.8):
        self.daily_budget_usd = daily_budget_usd
        self.global_daily_budget_usd = global_daily_budget_usd
        self.warn_at_pct = warn_at_pct
        self._records: dict[str, UsageRecord] = {}
        self._global_today = time.strftime("%Y-%m-%d")
        self._global_cost = 0.0

    def check_budget(self, user_id: str) -> None:
        record = self._get_record(user_id)

        # 1. Global budget check
        if self._global_cost >= self.global_daily_budget_usd:
            logger.critical(f"GLOBAL BUDGET EXCEEDED: ${self._global_cost:.4f}")
            raise HTTPException(503, "Service temporarily unavailable due to budget limits. Try again tomorrow.")

        # 2. Per-user budget check
        if record.total_cost_usd >= self.daily_budget_usd:
            raise HTTPException(402, {
                "error": "Daily budget exceeded",
                "used_usd": record.total_cost_usd,
                "budget_usd": self.daily_budget_usd,
                "resets_at": "midnight UTC",
            })

        # 3. Warning ở 80% — chỉ log, không block
        if record.total_cost_usd >= self.daily_budget_usd * self.warn_at_pct:
            logger.warning(f"User {user_id} at {record.total_cost_usd/self.daily_budget_usd*100:.0f}% budget")

    def record_usage(self, user_id, input_tokens, output_tokens) -> UsageRecord:
        record = self._get_record(user_id)
        record.input_tokens += input_tokens
        record.output_tokens += output_tokens
        record.request_count += 1
        cost = (input_tokens/1000*PRICE_PER_1K_INPUT_TOKENS + output_tokens/1000*PRICE_PER_1K_OUTPUT_TOKENS)
        self._global_cost += cost
        return record


cost_guard = CostGuard(daily_budget_usd=1.0, global_daily_budget_usd=10.0)
```

⚠️ **Lưu ý:** file này đã **implement đầy đủ** một `CostGuard` class (in-memory), khác với snippet TODO/Redis trong CODE_LAB.md (đó là phiên bản đơn giản hóa khác, dùng monthly budget). Hai cách tiếp cận bổ sung cho nhau — Redis là phần "Solution" giải quyết hạn chế in-memory bên dưới.

**3 lớp bảo vệ trong `check_budget()`:**
1. **Global budget** (`$10/ngày` tổng tất cả users) — vượt → `503 Service Unavailable`
2. **Per-user budget** (`$1/ngày`) — vượt → `402 Payment Required` kèm `used_usd`, `budget_usd`, `resets_at: "midnight UTC"`
3. **Warning ở 80%** — chỉ log `WARNING`, **không** block request

**Test thực tế** (gọi trực tiếp `CostGuard` vì $1/ngày cần ~50k request thật để chạm ngưỡng):

```python
>>> cg = CostGuard(daily_budget_usd=1.0, global_daily_budget_usd=10.0)
>>> cg.record_usage("student", input_tokens=1_500_000, output_tokens=1_500_000)
>>> cg.get_usage("student")
{'cost_usd': 1.125, 'budget_usd': 1.0, 'budget_remaining_usd': 0, 'budget_used_pct': 112.5}

>>> cg.check_budget("student")
HTTPException: 402 {"error":"Daily budget exceeded","used_usd":1.125,"budget_usd":1.0,"resets_at":"midnight UTC"}
```

**Test ngưỡng 80%** (`teacher`, 1.1M input + 1.1M output tokens → $0.825 = 82.5%):
```
WARNING:cost_guard:User teacher at 82% budget
```
→ `check_budget()` **vẫn pass**, request vẫn được xử lý — chỉ log cảnh báo.

**Test global budget** (giả lập `_global_cost` vượt `global_daily_budget_usd`):
```
CRITICAL:cost_guard:GLOBAL BUDGET EXCEEDED: $0.0020
```
→ raise **`503`**: `"Service temporarily unavailable due to budget limits. Try again tomorrow."`

**Hạn chế (đúng như comment trong file):** dữ liệu lưu **in-memory** (`dict`), mất khi restart server và **không share giữa nhiều instances** khi scale ngang.

**Solution mẫu trong lab (Redis, monthly budget):**
```python
import redis
from datetime import datetime

r = redis.Redis()

def check_budget(user_id: str, estimated_cost: float) -> bool:
    month_key = datetime.now().strftime("%Y-%m")
    key = f"budget:{user_id}:{month_key}"

    current = float(r.get(key) or 0)
    if current + estimated_cost > 10:
        return False

    r.incrbyfloat(key, estimated_cost)
    r.expire(key, 32 * 24 * 3600)  # 32 days
    return True
```
→ Đây là cách giải quyết hạn chế "in-memory" của `CostGuard`: lưu cost trong Redis (`INCRBYFLOAT` + `EXPIRE`), share được giữa nhiều instance, persist qua restart.

### Checkpoint 4
- [x] Implement API key authentication
- [x] Hiểu JWT flow
- [x] Implement rate limiting
- [x] Implement cost guard với Redis (đã có 2 cách: in-memory `CostGuard` + Redis solution)

---

## Part 5: Scaling & Reliability

### Exercise 5.1: Health checks

`05-scaling-reliability/develop/app.py`:

```python
START_TIME = time.time()
_is_ready = False
_in_flight_requests = 0


@app.get("/health")
def health():
    """LIVENESS PROBE — "Agent có còn sống không?" """
    uptime = round(time.time() - START_TIME, 1)

    checks = {}
    try:
        import psutil
        mem = psutil.virtual_memory()
        checks["memory"] = {
            "status": "ok" if mem.percent < 90 else "degraded",
            "used_percent": mem.percent,
        }
    except ImportError:
        checks["memory"] = {"status": "ok", "note": "psutil not installed"}

    overall_status = "ok" if all(v.get("status") == "ok" for v in checks.values()) else "degraded"

    return {
        "status": overall_status,
        "uptime_seconds": uptime,
        "version": "1.0.0",
        "environment": os.getenv("ENVIRONMENT", "development"),
        "timestamp": datetime.now(timezone.utc).isoformat(),
        "checks": checks,
    }


@app.get("/ready")
def ready():
    """READINESS PROBE — "Agent có sẵn sàng nhận request chưa?" """
    if not _is_ready:
        raise HTTPException(status_code=503, detail="Agent not ready. Check back in a few seconds.")
    return {"ready": True, "in_flight_requests": _in_flight_requests}
```

**Test thực tế** (chạy `python app.py` với `PORT=8005`, venv riêng + `pip install -r requirements.txt`, cần thêm `psutil==6.0.0`):

```bash
$ curl http://localhost:8005/health
{
  "status": "ok",
  "uptime_seconds": 0.9,
  "version": "1.0.0",
  "environment": "development",
  "timestamp": "2026-06-12T09:37:33.986948+00:00",
  "checks": { "memory": { "status": "ok", "used_percent": 81.6 } }
}

$ curl http://localhost:8005/ready
{ "ready": true, "in_flight_requests": 1 }

$ curl -X POST "http://localhost:8005/ask?question=What+is+Docker"
{ "answer": "Container là cách đóng gói app để chạy ở mọi nơi. Build once, run anywhere!" }
```

**So sánh `/health` vs `/ready`:**

| | `/health` (Liveness) | `/ready` (Readiness) |
|---|---|---|
| Trả lời câu hỏi | "Process có còn sống/chạy được không?" | "Có nên route traffic vào instance này không?" |
| Logic | Luôn trả `200`, kèm `status: ok/degraded` tuỳ check phụ (RAM) | Trả `503` nếu `_is_ready == False` (đang startup/shutdown) |
| Khi fail | Platform (Railway/Render/K8s) **restart container** | Load balancer **tạm ngừng gửi request mới**, không restart |
| Dùng trong lab này | Check RAM qua `psutil` — nếu RAM > 90% → `status: "degraded"` (vẫn 200, không restart) | Check biến global `_is_ready`, set bởi `lifespan` (startup → `True`, shutdown → `False`) |

### Exercise 5.2: Graceful shutdown

`05-scaling-reliability/develop/app.py`:

```python
_in_flight_requests = 0

@app.middleware("http")
async def track_requests(request, call_next):
    """Theo dõi số request đang xử lý."""
    global _in_flight_requests
    _in_flight_requests += 1
    try:
        return await call_next(request)
    finally:
        _in_flight_requests -= 1


@asynccontextmanager
async def lifespan(app: FastAPI):
    global _is_ready
    logger.info("Agent starting up...")
    time.sleep(0.2)
    _is_ready = True
    logger.info("✅ Agent is ready!")

    yield

    # ── Shutdown ──
    _is_ready = False
    logger.info("🔄 Graceful shutdown initiated...")

    timeout, elapsed = 30, 0
    while _in_flight_requests > 0 and elapsed < timeout:
        logger.info(f"Waiting for {_in_flight_requests} in-flight requests...")
        time.sleep(1)
        elapsed += 1

    logger.info("✅ Shutdown complete")


def handle_sigterm(signum, frame):
    logger.info(f"Received signal {signum} — uvicorn will handle graceful shutdown")

signal.signal(signal.SIGTERM, handle_sigterm)
signal.signal(signal.SIGINT, handle_sigterm)


if __name__ == "__main__":
    uvicorn.run(app, host="0.0.0.0", port=port, timeout_graceful_shutdown=30)
```

**Test thực tế** — vì Windows không gửi `SIGTERM` thật cho process Python (`taskkill` = `SIGKILL`), chạy app **trong container Linux** (PID 1 = `python app.py`, dùng `exec` để uvicorn nhận signal trực tiếp), rồi gửi 1 request và `docker stop` ngay sau đó:

```bash
$ docker run -d --name health-demo -v "$(pwd):/app" -w /app -p 8006:8000 \
  python:3.11-slim sh -c "pip install -q -r requirements.txt && exec python app.py"

$ curl -s -X POST "http://localhost:8006/ask?question=Long+task" &
$ docker stop -t 35 health-demo
```

**Log quan sát được:**
```
INFO:     Uvicorn running on http://0.0.0.0:8000 (Press CTRL+C to quit)
INFO:     Shutting down                                          ← SIGTERM nhận được
INFO:     172.17.0.1:54438 - "POST /ask?question=Long+task ..." 200 OK   ← request hoàn thành trước
INFO:     Waiting for application shutdown.
2026-06-12 09:42:05 INFO 🔄 Graceful shutdown initiated...
2026-06-12 09:42:05 INFO ✅ Shutdown complete
INFO:     Application shutdown complete.
INFO:     Finished server process [1]
2026-06-12 09:42:05 INFO Received signal 15 — uvicorn will handle graceful shutdown
```

→ **Request `/ask` đã hoàn thành (200 OK) trước khi container thực sự dừng** — đúng hành vi graceful shutdown.

**Cơ chế tổng thể:**

| Bước | Thành phần | Hành động |
|---|---|---|
| 1 | Platform (Railway/Render/K8s) | gửi `SIGTERM` tới container, đồng thời ngừng route traffic mới |
| 2 | `/ready` | trả `503` ngay (vì `_is_ready = False`) → load balancer biết để loại instance này |
| 3 | uvicorn | bắt `SIGTERM`, không nhận connection mới, chờ request hiện tại xong |
| 4 | `lifespan` shutdown | chờ `_in_flight_requests == 0`, tối đa `timeout=30s` |
| 5 | Nếu vượt 30s | platform gửi `SIGKILL` — request đang chạy bị cắt ngang |

### Exercise 5.3: Stateless design

`05-scaling-reliability/production/app.py`:

```python
try:
    import redis
    REDIS_URL = os.getenv("REDIS_URL", "redis://localhost:6379/0")
    _redis = redis.from_url(REDIS_URL, decode_responses=True)
    _redis.ping()
    USE_REDIS = True
except Exception:
    USE_REDIS = False
    _memory_store: dict = {}   # ⚠️ chỉ dùng khi không có Redis (dev/demo)


def save_session(session_id: str, data: dict, ttl_seconds: int = 3600):
    serialized = json.dumps(data)
    if USE_REDIS:
        _redis.setex(f"session:{session_id}", ttl_seconds, serialized)
    else:
        _memory_store[f"session:{session_id}"] = data


def load_session(session_id: str) -> dict:
    if USE_REDIS:
        data = _redis.get(f"session:{session_id}")
        return json.loads(data) if data else {}
    return _memory_store.get(f"session:{session_id}", {})


def append_to_history(session_id: str, role: str, content: str):
    session = load_session(session_id)
    history = session.get("history", [])
    history.append({"role": role, "content": content, "timestamp": datetime.now(timezone.utc).isoformat()})
    if len(history) > 20:
        history = history[-20:]
    session["history"] = history
    save_session(session_id, session)
    return history


@app.post("/chat")
async def chat(body: ChatRequest):
    session_id = body.session_id or str(uuid.uuid4())
    append_to_history(session_id, "user", body.question)
    answer = ask(body.question)
    append_to_history(session_id, "assistant", answer)
    return {
        "session_id": session_id,
        "question": body.question,
        "answer": answer,
        "served_by": INSTANCE_ID,   # ← chứng minh instance nào cũng đọc/viết được session
        "storage": "redis" if USE_REDIS else "in-memory",
    }
```

**Anti-pattern (trong đề bài) vs Correct (code thực tế):**

```python
#  Anti-pattern — State trong memory
conversation_history = {}

@app.post("/ask")
def ask(user_id: str, question: str):
    history = conversation_history.get(user_id, [])
    # ...

#  Correct — State trong Redis (đúng như production/app.py)
@app.post("/chat")
async def chat(body: ChatRequest):
    history = load_session(session_id).get("history", [])  # → từ Redis
    # ...
```

| | Anti-pattern (`conversation_history = {}`) | Code thực tế (`save_session`/`load_session` qua Redis) |
|---|---|---|
| Nơi lưu | RAM của process | Redis (shared, ngoài process) |
| Khi scale 3 instances | User A request 1 → Instance 1 (có history). Request 2 → Instance 2 (history rỗng!) → **mất context** | Bất kỳ instance nào đọc/viết cùng key `session:{id}` trong Redis → **context luôn nhất quán** |
| Khi instance restart/crash | Mất toàn bộ history đang có trong RAM | History vẫn còn trong Redis (TTL 1h) |
| `INSTANCE_ID` | không cần thiết | dùng để **chứng minh** (field `served_by`) — debug/demo |

### Exercise 5.4: Load balancing

`05-scaling-reliability/production/nginx.conf`:

```nginx
events { worker_connections 256; }

http {
    resolver 127.0.0.11 valid=10s;

    upstream agent_cluster {
        server agent:8000;   # Docker DNS "agent" → round-robin tự động giữa 3 container cùng service name
        keepalive 16;
    }

    server {
        listen 80;
        add_header X-Served-By $upstream_addr always;

        location / {
            proxy_pass http://agent_cluster;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_next_upstream error timeout http_503;   # tự chuyển sang instance khác nếu lỗi
            proxy_next_upstream_tries 3;
        }

        location /health {
            proxy_pass http://agent_cluster/health;
            access_log off;
        }
    }
}
```

**Setup thực tế:**
```bash
cd 05-scaling-reliability/production
touch .env.local
docker compose up -d --build --scale agent=3
```

```
NAME                  STATUS              PORTS
production-agent-1    Up (healthy)        8000/tcp   (không expose ra ngoài)
production-agent-2    Up (healthy)        8000/tcp
production-agent-3    Up (healthy)        8000/tcp
production-nginx-1    Up                  0.0.0.0:8080->80/tcp
production-redis-1    Up (healthy)        6379/tcp
```

**Test 6 request `/chat` (mỗi request tạo session mới):**

```bash
for i in 1 2 3 4 5 6; do
  curl -s -X POST http://localhost:8080/chat -H "Content-Type: application/json" -d '{"question":"ping '$i'"}'
done
```

| Request | served_by |
|---|---|
| 1 | instance-ddf9c5 |
| 2 | instance-91c43c |
| 3 | instance-4f4bd7 |
| 4 | instance-ddf9c5 |
| 5 | instance-91c43c |
| 6 | instance-4f4bd7 |

```bash
$ curl -sI http://localhost:8080/health | grep -i served
X-Served-By: 172.19.0.4:8000
```

→ Nginx round-robin đều đặn qua cả 3 instance.

**Test failover** — `docker stop production-agent-2` (giả lập 1 instance die), rồi gửi tiếp 4 requests với **cùng session_id**:

```
$ docker stop production-agent-2
production-agent-2

instance-4f4bd7
instance-ddf9c5
instance-4f4bd7
instance-ddf9c5
```

→ Sau khi `agent-2` bị dừng, **nginx tự động chỉ route tới 2 instance còn sống** (`proxy_next_upstream` retry sang upstream khác khi gặp lỗi/timeout), không có request nào bị lỗi.

### Exercise 5.5: Test stateless

```bash
PYTHONIOENCODING=utf-8 python test_stateless.py
```

**Output thực tế (5 câu hỏi, cùng 1 session):**

```
============================================================
Stateless Scaling Demo
============================================================

Session ID: 52fb52d7-524f-460f-a24c-e8c1392bc014

Request 1: [instance-91c43c]
  Q: What is Docker?
  A: Container là cách đóng gói app để chạy ở mọi nơi. Build once, run anywhere!...

Request 2: [instance-4f4bd7]
  Q: Why do we need containers?
  A: Tôi là AI agent được deploy lên cloud. Câu hỏi của bạn đã được nhận....

Request 3: [instance-ddf9c5]
  Q: What is Kubernetes?
  A: Agent đang hoạt động tốt! (mock response) Hỏi thêm câu hỏi đi nhé....

Request 4: [instance-91c43c]
  Q: How does load balancing work?
  A: Đây là câu trả lời từ AI agent (mock). Trong production, đây sẽ là response từ O...

Request 5: [instance-4f4bd7]
  Q: What is Redis used for?
  A: Agent đang hoạt động tốt! (mock response) Hỏi thêm câu hỏi đi nhé....

------------------------------------------------------------
Total requests: 5
Instances used: {'instance-4f4bd7', 'instance-ddf9c5', 'instance-91c43c'}
✅ All requests served despite different instances!

--- Conversation History ---
Total messages: 10
  [user]: What is Docker?...
  [assistant]: Container là cách đóng gói app để chạy ở mọi nơi. Build once...
  [user]: Why do we need containers?...
  [assistant]: Tôi là AI agent được deploy lên cloud. Câu hỏi của bạn đã đư...
  [user]: What is Kubernetes?...
  [assistant]: Agent đang hoạt động tốt! (mock response) Hỏi thêm câu hỏi đ...
  [user]: How does load balancing work?...
  [assistant]: Đây là câu trả lời từ AI agent (mock). Trong production, đây...
  [user]: What is Redis used for?...
  [assistant]: Agent đang hoạt động tốt! (mock response) Hỏi thêm câu hỏi đ...

✅ Session history preserved across all instances via Redis!
```

(Cần `PYTHONIOENCODING=utf-8` trên Windows vì console mặc định `cp1258` không encode được tiếng Việt trong mock response — không phải lỗi code.)

**Mở rộng — test "kill instance" thật** (đúng tinh thần bước 2 của script): sau khi có 10 messages trong history, `docker stop production-agent-2` rồi gửi tiếp 4 requests với cùng `session_id`:

```bash
$ curl -s http://localhost:8080/chat/52fb52d7.../history | jq '.count'
18   # 10 (trước) + 4*2 (sau khi 1 instance đã bị kill)
```

→ **History tăng từ 10 lên 18 messages liên tục, không mất dữ liệu**, dù request được phục vụ bởi `instance-4f4bd7` và `instance-ddf9c5` xen kẽ (instance-91c43c/agent-2 đã chết). Đây là minh chứng cho **stateless design**:
- Agent instance nào chết cũng không làm mất conversation
- Nginx tự động loại instance chết khỏi load balancing (`proxy_next_upstream`)
- User hoàn toàn không nhận biết được có instance bị restart ở phía sau

```bash
docker compose down   # dọn dẹp sau khi test xong
```

### Checkpoint 5
- [x] Implement health và readiness checks — `/health` (liveness, check RAM qua psutil) + `/ready` (readiness, dựa vào `_is_ready` flag)
- [x] Implement graceful shutdown — middleware đếm `_in_flight_requests` + lifespan shutdown chờ tối đa 30s + `timeout_graceful_shutdown=30`
- [x] Refactor code thành stateless — session/history lưu trong Redis (`save_session`/`load_session`), instance không giữ state trong RAM
- [x] Hiểu load balancing với Nginx — `upstream agent_cluster` round-robin qua Docker DNS, `proxy_next_upstream` tự failover khi 1 instance chết
- [x] Test stateless design — `test_stateless.py` + test failover thủ công (`docker stop` 1 instance, history vẫn nguyên vẹn qua Redis)

---

## Part 6: Final Project

### 🚀 Sản phẩm nộp chính (Exercise 6)

| | |
|---|---|
| **Source code** | https://github.com/oggishi/Day08_RAG_pipeline_cohort2 |
| **Live Demo (GitHub Pages)** | https://oggishi.github.io/Day08_RAG_pipeline_cohort2 |
| **Backend API (Render)** | ⏳ *sẽ điền sau khi deploy* — xem [DEPLOYMENT.md](DEPLOYMENT.md#4-final-project--sản-phẩm-nộp-exercise-6--part-6) |

Repo trên là RAG pipeline được dùng làm sản phẩm cuối cho Part 6, áp dụng các nguyên tắc production-readiness đã học ở Part 1-5 (config qua env, health check, containerization, deploy lên cloud...). Ảnh chụp màn hình deployment/test nằm trong thư mục [`screenshots/`](screenshots/) của repo này (Day12).

### Phiên bản tham khảo nội bộ (`06-lab-complete/`)

Ngoài ra, repo Day 12 này cũng có sẵn một production agent kết hợp **tất cả** concepts từ Part 1-5 (xem [DEPLOYMENT.md](DEPLOYMENT.md) để biết chi tiết public URL và bảng đối chiếu tính năng ↔ exercise đã test).

| Step | File | Trạng thái |
|---|---|---|
| 1. Project setup | `06-lab-complete/{app,Dockerfile,docker-compose.yml,requirements.txt,.env.example,.dockerignore}` | ✅ |
| 2. Config management | `app/config.py` — `Settings` dataclass đọc toàn bộ từ env + `validate()` fail-fast nếu thiếu secret ở production | ✅ |
| 3. Main application | `app/main.py` — `/`, `/ask`, `/health`, `/ready`, `/metrics` | ✅ |
| 4. Authentication | `verify_api_key()` — header `X-API-Key`, raise `401` nếu sai/thiếu | ✅ |
| 5. Rate limiting | `check_rate_limit()` — sliding window (deque), `RATE_LIMIT_PER_MINUTE` (mặc định 20/phút), raise `429` + `Retry-After` | ✅ |
| 6. Cost guard | `check_and_record_cost()` — tính cost theo token, `DAILY_BUDGET_USD`, raise `503` khi vượt | ✅ |
| 7. Dockerfile | Multi-stage (`builder` → `runtime`), non-root user `agent`, `HEALTHCHECK`, `python:3.11-slim` | ✅ |
| 8. Docker Compose | `agent` (build + healthcheck) + `redis` (7-alpine, `allkeys-lru`) | ✅ |
| 9. Test locally | xem kết quả test thực tế bên dưới | ✅ |
| 10. Deploy | `railway.toml` + `render.yaml` có sẵn — 2 public URL đã chạy (Railway + Render, Exercise 3.1/3.2) | ✅ |

### 🐛 2 bug phát hiện và fix khi build/run thực tế

**1. `Dockerfile` — sai đường dẫn user site-packages.**

`useradd -r -g agent -d /app agent` đặt **HOME của user `agent` = `/app`**, nên Python tìm package `pip install --user` tại `/app/.local`. Dockerfile cũ lại `COPY --from=builder /root/.local /home/agent/.local` và `ENV PATH=/home/agent/.local/bin:$PATH` → sai thư mục → container crash loop:

```
agent-1  | Traceback (most recent call last):
agent-1  |   File "/home/agent/.local/bin/uvicorn", line 5, in <module>
agent-1  |     from uvicorn.main import main
agent-1  | ModuleNotFoundError: No module named 'uvicorn'
```

**Fix** (`06-lab-complete/Dockerfile`):
```dockerfile
# Copy packages từ builder (HOME của user "agent" là /app, xem useradd -d /app)
COPY --from=builder /root/.local /app/.local
...
ENV PATH=/app/.local/bin:$PATH
```

**2. `app/main.py` — `response.headers.pop("server", None)` không hợp lệ.**

Starlette's `MutableHeaders` không có method `.pop()` → mọi request trả `500 Internal Server Error`:
```
agent-1  |   File "/app/app/main.py", line 148, in request_middleware
agent-1  |     response.headers.pop("server", None)
agent-1  |     ^^^^^^^^^^^^^^^^^^^^
agent-1  | AttributeError: 'MutableHeaders' object has no attribute 'pop'
```
(lỗi tương tự đã từng gặp và fix ở `04-api-gateway/production/app.py:85`)

**Fix:**
```python
response.headers["X-Frame-Options"] = "DENY"
if "server" in response.headers:
    del response.headers["server"]
```

**3.** Thư mục `06-lab-complete/utils/` còn thiếu (`COPY utils/ ./utils/` cần file trong build context) — đã `cp utils/mock_llm.py 06-lab-complete/utils/mock_llm.py`.

### ✅ Test thực tế sau khi fix — `docker compose up -d --build`

```bash
$ cd 06-lab-complete
$ cp .env.example .env.local
$ docker compose up -d --build
 Container 06-lab-complete-redis-1  Healthy
 Container 06-lab-complete-agent-1  Started
```

```bash
$ curl http://localhost:8000/health
{"status":"ok","version":"1.0.0","environment":"staging","uptime_seconds":16.5,"total_requests":1,"checks":{"llm":"mock"},"timestamp":"2026-06-12T10:31:16.660920+00:00"}

$ curl http://localhost:8000/ready
{"ready":true}

# Không có API key → 401
$ curl -i -X POST http://localhost:8000/ask -H "Content-Type: application/json" -d '{"question":"hi"}'
HTTP/1.1 401 Unauthorized
{"detail":"Invalid or missing API key. Include header: X-API-Key: <key>"}

# Có API key đúng → 200
$ curl -X POST http://localhost:8000/ask -H "X-API-Key: dev-key-change-me-in-production" \
    -H "Content-Type: application/json" -d '{"question": "What is Docker?"}'
{"question":"What is Docker?","answer":"Container là cách đóng gói app để chạy ở mọi nơi. Build once, run anywhere!","model":"gpt-4o-mini","timestamp":"2026-06-12T10:31:17.149255+00:00"}
```

**Rate limiting** (`RATE_LIMIT_PER_MINUTE=20`, gửi 22 request `/ask` liên tiếp, sau 1 request đã dùng trước đó = 23 tổng):

```bash
$ for i in $(seq 1 22); do
    curl -s -o /dev/null -w "%{http_code} " -X POST http://localhost:8000/ask \
      -H "X-API-Key: dev-key-change-me-in-production" -H "Content-Type: application/json" \
      -d '{"question":"test '$i'"}'
  done
200 200 200 200 200 200 200 200 200 200 200 200 200 200 200 200 200 200 200 429 429 429
```
→ Request thứ 20 (tính từ đầu) là request cuối được phép, từ 21 trở đi trả `429 Too Many Requests`.

**Cost guard / metrics:**
```bash
$ curl http://localhost:8000/metrics -H "X-API-Key: dev-key-change-me-in-production"
{"uptime_seconds":149.0,"total_requests":30,"error_count":0,"daily_cost_usd":0.0004,"daily_budget_usd":5.0,"budget_used_pct":0.0}
```

**Graceful shutdown** (gửi 1 request `/ask` rồi `docker compose stop -t 10 agent`):
```
{"event": "request", "method": "POST", "path": "/ask", "status": 200, "ms": 115.9}
INFO:     Received SIGTERM, exiting.
INFO:     Terminated child process [8]
INFO:     Terminated child process [9]
{"event": "signal", "signum": 15}
```
→ Request `/ask` hoàn thành `200 OK` **trước** khi 2 worker process bị terminate.

### ✅ `python check_production_ready.py` — 20/20 (100%)

```bash
$ cd 06-lab-complete
$ PYTHONIOENCODING=utf-8 python check_production_ready.py
```

```
=======================================================
  Production Readiness Check — Day 12 Lab
=======================================================

📁 Required Files
  ✅ Dockerfile exists
  ✅ docker-compose.yml exists
  ✅ .dockerignore exists
  ✅ .env.example exists
  ✅ requirements.txt exists
  ✅ railway.toml or render.yaml exists

🔒 Security
  ✅ .env in .gitignore
  ✅ No hardcoded secrets in code

🌐 API Endpoints (code check)
  ✅ /health endpoint defined
  ✅ /ready endpoint defined
  ✅ Authentication implemented
  ✅ Rate limiting implemented
  ✅ Graceful shutdown (SIGTERM)
  ✅ Structured logging (JSON)

🐳 Docker
  ✅ Multi-stage build
  ✅ Non-root user
  ✅ HEALTHCHECK instruction
  ✅ Slim base image
  ✅ .dockerignore covers .env
  ✅ .dockerignore covers __pycache__

=======================================================
  Result: 20/20 checks passed (100%)
  🎉 PRODUCTION READY! Deploy nào!
=======================================================
```

### Grading rubric tự đánh giá

| Criteria | Điểm | Ghi chú |
|---|---|---|
| Functionality (20) | 20 | `/ask` trả lời đúng, validate input (Pydantic `AskRequest`, min/max length) |
| Docker (15) | 15 | Multi-stage, non-root, slim, HEALTHCHECK — build + run thành công |
| Security (20) | 20 | API key auth (401), rate limit (429), cost guard (503 khi vượt budget) |
| Reliability (20) | 20 | `/health`, `/ready`, graceful shutdown (SIGTERM, request hoàn thành trước khi tắt) |
| Scalability (15) | 15 | Stateless (Redis-backed session đã chứng minh ở Exercise 5.3/5.5), `REDIS_URL` config sẵn |
| Deployment (10) | 10 | `railway.toml` + `render.yaml` có sẵn, 2 public URL đã chạy (Exercise 3.1/3.2) |
| **Total** | **100** | |

```bash
docker compose down   # dọn dẹp sau khi test
```

### Checkpoint 6
- [x] Build production-ready agent kết hợp tất cả concepts (config 12-factor, JSON logging, API key auth, rate limit, cost guard, health/ready, graceful shutdown)
- [x] Dockerize multi-stage, non-root, slim, HEALTHCHECK — build + run thành công sau khi fix 2 bug
- [x] `docker compose up -d --build` chạy agent + redis, test đủ `/health`, `/ready`, `/ask`, `/metrics`
- [x] Test rate limiting (429) và graceful shutdown (SIGTERM)
- [x] `check_production_ready.py` → 20/20 (100%)
- [x] Deploy lên cloud — 2 public URL hoạt động (Railway + Render)
