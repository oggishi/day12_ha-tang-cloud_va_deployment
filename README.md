# Day 12 — Deployment: Đưa Agent Lên Cloud


## 📝 Bài Nộp 

> **Student:** Nguyen Thi Bao Tran — **ID:** 2A202600917

| Thông tin | Vị trí |
|-----------|--------|
| **Báo cáo đầy đủ** (code, test output, log, bảng so sánh từng exercise) | [MISSION_ANSWERS.md](MISSION_ANSWERS.md) |
| **Thông tin deployment & public URL** | [DEPLOYMENT.md](DEPLOYMENT.md) |
| **Ảnh chụp màn hình deployment/test** | [screenshots/](screenshots/) |
| **Sản phẩm cuối ( Part 6) — Source code** | https://github.com/oggishi/Day08_RAG_pipeline_cohort2 |
| **Sản phẩm cuối — Live Demo (GitHub Pages)** | https://oggishi.github.io/Day08_RAG_pipeline_cohort2 |
| **Sản phẩm cuối —  (Render)** | https://luatmatuy-api.onrender.commd|
| **Public URL Exercise 3.1 (Railway)** | https://secure-balance-production-6666.up.railway.app/ |
| **Public URL Exercise 3.2 (Render)** | https://ai-agent-7keu.onrender.com |


## Cấu Trúc Project

```
day12_ha-tang-cloud_va_deployment/
├── 01-localhost-vs-production/     # Section 1: Dev ≠ Production
│   ├── develop/                      #   Agent "đúng kiểu localhost"
│   └── production/                   #   12-Factor compliant agent
│
├── 02-docker/                      # Section 2: Containerization
│   ├── develop/                      #   Dockerfile đơn giản
│   └── production/                   #   Multi-stage + Docker Compose stack
│
├── 03-cloud-deployment/            # Section 3: Cloud Options
│   ├── railway/                    #   Deploy Railway (< 5 phút)
│   ├── render/                     #   Deploy Render + render.yaml
│   └── production-cloud-run/         #   GCP Cloud Run + CI/CD
│
├── 04-api-gateway/                 # Section 4: Security
│   ├── develop/                      #   API Key authentication
│   └── production/                   #   JWT + Rate Limiting + Cost Guard
│
├── 05-scaling-reliability/         # Section 5: Scale & Reliability
│   ├── develop/                      #   Health check + graceful shutdown
│   └── production/                   #   Stateless + Redis + Nginx LB
│
├── 06-lab-complete/                # Lab 12: Production-ready agent
│   └── (full project kết hợp tất cả)
│
└── utils/                          # Mock LLM dùng chung (không cần API key)
```

