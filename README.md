# AI Campus Digital Twin 🏫

**AI-Powered Smart Campus Navigation System** — Team Kero

[![CI](https://github.com/YOUR_USERNAME/ai-campus-digital-twin/actions/workflows/ci.yml/badge.svg)](https://github.com/YOUR_USERNAME/ai-campus-digital-twin/actions)
[![Python](https://img.shields.io/badge/Python-3.11%2B-blue)](https://python.org)
[![FastAPI](https://img.shields.io/badge/FastAPI-0.111-green)](https://fastapi.tiangolo.com)

> A real-time 3D digital twin of a college campus — find any room, navigate with AI, check live facility status, and respond to emergencies.

---

## Live Demo

| File | Description | How to open |
|------|-------------|-------------|
| `demo.html` | Full frontend demo (map, AI chat, navigation) | Double-click — no server needed |
| `backend-demo.html` | Live API tester against real backend | Open after starting backend |

---

## Quick Start

### Run backend (Python 3.11+)

```bash
cd backend
pip install -r requirements.txt
python -m uvicorn app.main:app --reload --port 8000
```

**Swagger UI:** http://localhost:8000/docs

### Run full stack (Docker)

```bash
docker compose up --build
```

| Service | URL |
|---------|-----|
| API | http://localhost:8000 |
| Swagger | http://localhost:8000/docs |
| Mongo UI | http://localhost:8081 |

---

## Demo Credentials

| Role | Email | Password |
|------|-------|----------|
| Student | student@kero.edu | student123 |
| Faculty | faculty@kero.edu | faculty123 |
| **Admin** | **admin@kero.edu** | **admin123** |
| Security | security@kero.edu | security123 |

---

## API Endpoints

All prefixed with `/api/v1` | Full docs at `/docs`

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| POST | /auth/login | Public | Login → JWT |
| POST | /auth/guest | Public | Guest token |
| POST | /ai/chat | JWT | AI natural language query |
| POST | /nav/route | JWT | A* route (accessible routing) |
| GET | /emergency/nearest | Public | Nearest exits + first aid |
| GET | /search | JWT | Full-text + fuzzy search |
| GET | /facilities | JWT | Live occupancy status |
| GET | /faculty | JWT | Faculty directory |
| GET/POST | /events | JWT/Admin | Campus events + bookmark |
| POST | /admin/rooms | Admin | Room management |
| GET | /admin/audit-log | Admin | 90-day audit trail |
| POST | /emergency/alert | Admin | Campus-wide broadcast |

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Backend | Python 3.11 · FastAPI · Uvicorn |
| Navigation | NetworkX · A* algorithm |
| AI | OpenAI GPT-4o · keyword fallback (no key needed) |
| Database | MongoDB Atlas · Motor (async) |
| Cache | Redis |
| Auth | PyJWT · HMAC-SHA256 · RBAC |
| Container | Docker · Docker Compose |
| CI/CD | GitHub Actions |

---

## Project Structure

```
ai-campus-digital-twin/
├── demo.html                  # Frontend demo (no server needed)
├── backend-demo.html          # Live API tester
├── docker-compose.yml         # Full stack: API + MongoDB + Redis
├── .github/workflows/
│   ├── ci.yml                 # Lint + test on every push
│   └── deploy.yml             # Docker build + push to GHCR
├── backend/
│   ├── app/
│   │   ├── main.py            # FastAPI app entry point
│   │   ├── config.py          # Pydantic settings
│   │   ├── mock_data.py       # Seed data (no DB required for demo)
│   │   ├── auth/              # JWT creation + RBAC dependencies
│   │   ├── routers/           # 9 routers · 30 endpoints
│   │   └── services/          # A* pathfinding · AI assistant
│   ├── tests/                 # Unit tests (auth, navigation, search)
│   ├── Dockerfile
│   └── requirements.txt
└── .kiro/specs/               # Full PRD, design doc, task plan
    └── ai-campus-digital-twin/
        ├── requirements.md    # 14 requirements + acceptance criteria
        ├── design.md          # Architecture · APIs · algorithms
        └── tasks.md           # 15 task groups · effort estimates · risk register
```

---

## Environment Variables

Copy `backend/.env.example` → `backend/.env`

```env
SECRET_KEY=your-256-bit-random-key
MONGODB_URI=mongodb+srv://user:pass@cluster.mongodb.net/campus_twin
REDIS_URL=redis://localhost:6379/0
OPENAI_API_KEY=sk-...        # Optional — mock AI works without it
```

> The backend runs in **mock mode** without MongoDB/Redis — all endpoints work using in-memory data.

---

## Running Tests

```bash
cd backend
python -m pytest tests/ -v
```

---

## Deployment Options

| Platform | How |
|----------|-----|
| **Render** | Connect repo → set env vars → deploy |
| **Railway** | `railway up` from backend/ |
| **Fly.io** | `fly launch` from backend/ |
| **AWS ECS** | Use `Dockerfile` + task definitions from `design.md` |

---

## Spec Documents

Full product documentation in `.kiro/specs/ai-campus-digital-twin/`:

- **requirements.md** — 14 requirements with user stories and acceptance criteria
- **design.md** — System architecture, data models, API contracts, A* algorithm
- **tasks.md** — 15 implementation tasks, effort estimates, risk register

---

*Team Kero · Version 1.0.0 · AI Campus Digital Twin*
