# Deployment Information

**Student Name:** Nguyen Thi Bao Tran
**Student ID:** 2A202600917

Lab này được deploy lên **2 platform** (Exercise 3.1 và 3.2 trong [CODE_LAB.md](CODE_LAB.md)):

---

## 1. Railway

### Public URL
https://secure-balance-production-6666.up.railway.app/

### Platform
Railway (deployed via Railway CLI: `railway init` → `railway up`)

### Source
`03-cloud-deployment/railway/`

### Test Commands

**Health check:**
```bash
curl https://secure-balance-production-6666.up.railway.app/health
```
Kết quả thực tế:
```json
{"status":"ok","uptime_seconds":4919.6,"platform":"Railway","timestamp":"2026-06-12T10:04:28.116812+00:00"}
```

**API Test:**
```bash
curl -X POST https://secure-balance-production-6666.up.railway.app/ask \
  -H "Content-Type: application/json" \
  -d '{"question": "What is deployment?"}'
```
Kết quả thực tế:
```json
{"question":"What is deployment?","answer":"Deployment là quá trình đưa code từ máy bạn lên server để người khác dùng được.","platform":"Railway"}
```

### Environment Variables Set
- `PORT` (Railway tự inject)
- `AGENT_API_KEY` (set qua `railway variables set`)

---

## 2. Render

### Public URL
https://ai-agent-7keu.onrender.com

### Platform
Render (deployed via Blueprint — `render.yaml`)

### Source
`03-cloud-deployment/render/`

### Test Commands

**Health check:**
```bash
curl https://ai-agent-7keu.onrender.com/health
```
Kết quả thực tế:
```json
{"status":"ok","uptime_seconds":83.4,"platform":"Render","timestamp":"2026-06-12T10:06:18.526115+00:00"}
```

**API Test:**
```bash
curl -X POST https://ai-agent-7keu.onrender.com/ask \
  -H "Content-Type: application/json" \
  -d '{"question": "What is Docker?"}'
```
Kết quả thực tế:
```json
{"question":"What is Docker?","answer":"Container là cách đóng gói app để chạy ở mọi nơi. Build once, run anywhere!","platform":"Render"}
```

### Environment Variables Set
- `PORT` (Render tự inject)
- `ENVIRONMENT=production`
- `PYTHON_VERSION=3.11.0`
- `AGENT_API_KEY` (`generateValue: true` — Render tự sinh)
- `OPENAI_API_KEY` (để trống — dùng mock LLM)
- Redis add-on `agent-cache` (free plan, `ipAllowList: 0.0.0.0/0`)

---

## 3. Full Production Agent (`06-lab-complete/`) — chạy & kiểm thử local

Phiên bản kết hợp đầy đủ tất cả tính năng (API Key auth, rate limiting, cost guard, health/readiness, graceful shutdown, structured JSON logging, multi-stage Dockerfile, docker-compose + Redis) nằm ở `06-lab-complete/`. Các tính năng này đã được implement và **test trực tiếp** ở các exercise tương ứng:

| Tính năng | Đã test ở | Kết quả |
|---|---|---|
| API Key authentication | Exercise 4.1 | 401 (thiếu key) / 403 (sai key) / 200 (đúng key) |
| JWT authentication | Exercise 4.2 | `/auth/token` trả JWT, `/ask` verify Bearer token |
| Rate limiting (sliding window) | Exercise 4.3 | 10 req/phút (user) — request 11+ trả 429 |
| Cost guard (per-user + global budget) | Exercise 4.4 | 402 (vượt budget user) / 503 (vượt budget global) / warning ở 80% |
| Health + Readiness checks | Exercise 5.1 | `/health` (liveness, check RAM), `/ready` (readiness, `_is_ready` flag) |
| Graceful shutdown (SIGTERM) | Exercise 5.2 | request hoàn thành 200 OK trước khi shutdown, log đầy đủ |
| Stateless design (Redis session) | Exercise 5.3, 5.5 | session/history lưu Redis, `served_by` đổi giữa 3 instance nhưng history nguyên vẹn |
| Load balancing (Nginx, 3 instances) | Exercise 5.4 | round-robin qua 3 instance, tự failover khi 1 instance chết |

Production readiness check:
```bash
cd 06-lab-complete
python check_production_ready.py
```

> Riêng `06-lab-complete/` chưa được deploy lên public URL riêng (do trùng tính năng với 2 deployment ở Exercise 3.1/3.2, vốn đã có public URL hoạt động). Toàn bộ tính năng của nó đã được kiểm thử end-to-end qua các exercise trong Part 4 và Part 5.

---

## 4. Final Project — Sản phẩm nộp (Exercise 6 / Part 6)

### Source code
https://github.com/oggishi/Day08_RAG_pipeline_cohort2

### Live Demo (GitHub Pages)
https://oggishi.github.io/Day08_RAG_pipeline_cohort2

### Backend API (Render)
> ⏳ TODO — sẽ điền link Render sau khi deploy:
>
> `https://<TÊN-SERVICE-CỦA-BẠN>.onrender.com`

### Test Commands (điền sau khi có link Render)

```bash
curl https://<TÊN-SERVICE-CỦA-BẠN>.onrender.com/health
```

```bash
curl -X POST https://<TÊN-SERVICE-CỦA-BẠN>.onrender.com/ask \
  -H "Content-Type: application/json" \
  -d '{"question": "..."}'
```

---

## Screenshots

- [Deployment dashboard](screenshots/dashboard.png)
- [Service running](screenshots/running.png)
- [Test results](screenshots/test.png)

> Ảnh chụp màn hình đặt trong thư mục [`screenshots/`](screenshots/) theo đúng 3 tên file trên (xem [screenshots/README.md](screenshots/README.md) để biết nội dung cần chụp).
