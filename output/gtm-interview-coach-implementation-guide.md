# AI GTM Interview Coach — Step-by-Step Implementation Guide

## Overview
- **6 modules**, **7 models**, **28 endpoints**, **14 pages**, **8 services**
- **3 phases**: Foundation → Core Modules → Quality

---

## Phase 1: Foundation (Do these in parallel)

### Step 1 — Set up your environment
```bash
mkdir gtm-interview-coach && cd gtm-interview-coach
cp .env.example .env  # Fill in all API keys
```

**Required API keys to get first:**
| Key | Where to get |
|-----|-------------|
| `ANTHROPIC_API_KEY` | console.anthropic.com |
| `OPENAI_API_KEY` | platform.openai.com |
| `ELEVENLABS_API_KEY` | elevenlabs.io |
| `EXA_API_KEY` | exa.ai |
| `STRIPE_SECRET_KEY` | dashboard.stripe.com |
| `AWS_ACCESS_KEY_ID/SECRET` | AWS IAM console |
| `SENDGRID_API_KEY` | sendgrid.com |
| `GOOGLE_CLIENT_ID/SECRET` | console.cloud.google.com |

---

### Step 2 — Database (DATABASE-AGENT)
Run: `/execute-prp PRPs/gtm-interview-coach-prp.md` and the DATABASE-AGENT will create:

- `backend/app/models/` — 5 model files (user, session, question, report, subscription)
- `alembic/versions/001_initial_schema.py` — all 7 tables
- `alembic/versions/002_seed_questions.py` — 30 seed questions

**Validate:**
```bash
alembic upgrade head
python -c "from app.models import *; print('OK')"
```

---

### Step 3 — Backend scaffold (BACKEND-AGENT)
Creates:
- `app/main.py` + `config.py` + `database.py` + `exceptions.py`
- All Pydantic schemas in `schemas/`
- Auth module: JWT + Google OAuth
- Service stubs for all 8 services

**Validate:**
```bash
ruff check backend/
uvicorn app.main:app --reload
```

---

### Step 4 — Frontend scaffold (FRONTEND-AGENT)
Creates:
- Vite + React + TypeScript + Tailwind + Framer Motion
- Base components: `GlassCard`, `GradientButton`, `AnimatedInput`
- `services/api.ts` — JWT interceptors + auto-refresh
- `context/AuthContext.tsx`
- React Router with protected routes

**Validate:**
```bash
npm install && npm run type-check
```

---

### Step 5 — Docker setup (DEVOPS-AGENT)
Creates:
- `docker-compose.yml` (postgres:16 + backend + frontend + redis)
- `backend/Dockerfile` + `frontend/Dockerfile`
- `.github/workflows/ci.yml`

**Validate:**
```bash
docker-compose config --quiet
docker-compose build
```

---

## Phase 2: Core Modules (Build in this order)

### Step 6 — Auth module
**Backend:** `routers/auth.py` + `services/auth_service.py`
- POST `/api/v1/auth/register` → bcrypt hash + JWT
- POST `/api/v1/auth/login` → access + refresh tokens
- GET `/api/v1/auth/google` → OAuth redirect
- GET `/api/v1/auth/google/callback` → token exchange

**Frontend:** `LoginPage.tsx` + `RegisterPage.tsx` + `ProfilePage.tsx`

---

### Step 7 — Question Bank
**Backend:** `routers/questions.py` + `routers/admin/questions.py`
- GET `/api/v1/questions` — filter by role/category/difficulty
- POST `/api/v1/admin/questions/bulk` — bulk import

**No frontend needed for MVP** (seed data covers it)

---

### Step 8 — Interview Session (core feature — most complex)

**Backend services to build in order:**

1. **`llm_service.py`** — Claude scoring first (most critical)
```python
# Claude claude-sonnet-4-6 → parse JSON score
# Fallback: GPT-4o same prompt
# Returns: score_label, score_numeric, feedback, pro_tip, improvement_area
```

2. **`exa_service.py`** — company context enrichment
```python
# Search: "{company} product GTM motion ICP"
# Return 500-char summary
```

3. **`voice_service.py`** — Whisper STT + ElevenLabs TTS
```python
# Whisper: base64 audio → transcript
# ElevenLabs: text → audio bytes → S3 → presigned URL
# Voice: Rachel (21m00Tcm4TlvDq8ikWAM), model: eleven_turbo_v2
```

4. **`interview_service.py`** — orchestrates everything
```python
# create_session(): check limits → Exa context → select 10 questions → TTS intro
# submit_answer(): STT → Claude score → store → next question → TTS feedback
# generate_report(): all answers → Claude report prompt → store PerformanceReport
```

**Frontend pages to build:**

1. **`SetupPage.tsx`** — role/company/difficulty/mode selectors
2. **`InterviewRoom.tsx`** — the core UI (state machine):
   - Text mode: question → textarea → submit → feedback panel → next
   - Voice mode: audio plays → mic button → recording → waveform → feedback → audio response

---

### Step 9 — Performance Reports
**Backend:** `routers/reports.py`
- GET `/api/v1/reports/{session_id}` — auto-generate if not exists
- GET `/api/v1/reports/{session_id}/pdf` — Pro only, WeasyPrint → S3
- GET `/api/v1/reports/public/{token}` — no auth

**Frontend:** `ReportPage.tsx` layout:
```
Overall Score circle → Summary → Category bar charts (Recharts)
→ Strength/Weakness cards → Action items → Share + PDF buttons
```

---

### Step 10 — Dashboard
**Backend:** `GET /api/v1/dashboard/stats` — aggregated scores + history

**Frontend:** `DashboardPage.tsx`:
- `StatsRow` — 4 metric cards with Framer Motion fade-in
- `ProgressLineChart` — Recharts LineChart (score over time)
- `CategoryRadarChart` — Recharts RadarChart
- `SessionHistory` — table with report links
- `QuickStartCard` — CTA to start next interview

---

### Step 11 — Subscriptions (Stripe)
**Backend:** `routers/subscription.py` + `services/stripe_service.py`

Stripe webhook events to handle:
```
customer.subscription.created → plan = pro/agency
customer.subscription.updated → update status + period_end
customer.subscription.deleted → status = cancelled
invoice.payment_failed → status = past_due + SendGrid email
```

Usage limit dependencies (inject into session/voice/PDF endpoints):
```python
check_session_limit()  # Free: max 3
check_voice_access()   # Free: blocked
check_pdf_access()     # Free: blocked
```

**Frontend:** `PricingPage.tsx` (3 cards) + `BillingPage.tsx` + `UpgradeModal.tsx`

---

## Phase 3: Quality

### Step 12 — Tests (80%+ coverage target)
```bash
# backend/tests/
conftest.py           # mock Claude, mock ElevenLabs, mock Whisper, test DB
test_auth.py          # register, login, refresh, Google OAuth
test_sessions.py      # create, submit answer, complete session
test_llm_service.py   # scoring, fallback to GPT-4o, JSON parse errors
test_voice_service.py # STT mock, TTS mock
test_reports.py       # report generation, PDF, share token
test_subscription.py  # webhook handler, usage limits
```

**Run:**
```bash
pytest --cov=app --cov-fail-under=80
```

---

### Step 13 — Security review checklist
- [ ] All API keys from env vars (no hardcoding)
- [ ] Stripe webhook signature verified (`stripe.Webhook.construct_event`)
- [ ] JWT secret minimum 32 chars
- [ ] Pydantic validates all inputs; base64 audio size limited
- [ ] N+1 queries fixed (eager load `SessionAnswers` with `joinedload`)
- [ ] Admin endpoints check `is_admin` flag

---

## Phase 4: Deployment

### Step 14 — Final validation
```bash
docker-compose up -d
curl localhost:8000/health       # → {"status": "ok"}
curl localhost:3000              # → React app loads
alembic upgrade head             # → all migrations applied
pytest --cov-fail-under=80      # → tests pass
npm run build                    # → no TS errors
```

---

## Execution Command

You already have everything set up. Run this to kick off the full build:

```bash
/execute-prp PRPs/gtm-interview-coach-prp.md
```

This activates the ORCHESTRATOR which dispatches DATABASE → BACKEND → FRONTEND → DEVOPS agents in Phase 1 parallel, then Module agents in Phase 2, then TEST agent in Phase 3.

---

## MVP Priority Order (if building manually)

| Priority | Feature | Time estimate |
|----------|---------|--------------|
| 1 | Auth (JWT + Google OAuth) | 2–3 hrs |
| 2 | Question bank + seed data | 1 hr |
| 3 | `llm_service.py` Claude scoring | 2 hrs |
| 4 | Interview session backend | 3–4 hrs |
| 5 | InterviewRoom frontend | 3–4 hrs |
| 6 | Performance report | 2–3 hrs |
| 7 | Dashboard | 2 hrs |
| 8 | Voice mode (ElevenLabs + Whisper) | 3 hrs |
| 9 | Stripe subscriptions | 3 hrs |
| 10 | Tests + Docker | 2–3 hrs |

**Total MVP (text mode only): ~15–18 hours**
