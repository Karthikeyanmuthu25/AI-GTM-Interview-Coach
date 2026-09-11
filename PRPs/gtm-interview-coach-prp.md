# PRP: GTM Interview Coach

> Implementation blueprint for parallel agent execution
> Generated from GTM_INITIAL.md

---

## METADATA

| Field | Value |
|-------|-------|
| **Product** | GTM Interview Coach |
| **Type** | SaaS (Freemium → Paid) |
| **Version** | 1.0 |
| **Created** | 2026-04-17 |
| **Complexity** | High |
| **Estimated Build Time** | 4–6 weeks (solo) · 2–3 weeks (parallel agents) |

---

## PRODUCT OVERVIEW

**Description:** AI-powered voice + text interview coaching platform for B2B SaaS professionals
targeting AI GTM Engineering, Product Marketing Manager, and RevOps roles. Users engage in
realistic mock interviews with an AI interviewer, receive Claude-scored feedback per answer,
and receive a structured performance report at session end.

**Value Proposition:**
- Real-time AI scoring eliminates the need for a human coach
- Voice mode (ElevenLabs TTS + Whisper STT) replicates actual interview conditions
- Company-specific context (Exa API) makes questions feel tailored to the target role
- Performance report with category scores + 30-day action plan is something no free tool offers

**MVP Scope:**
- [x] Text-mode interview sessions (Claude scoring)
- [x] JWT + Google OAuth authentication
- [x] Question bank: 30 seeded questions across 3 roles × 3 difficulties
- [x] Performance report at session end (category scores + action items)
- [x] Dashboard with score progression chart
- [ ] Voice mode: ElevenLabs TTS + Whisper STT
- [ ] Company context: Exa API enrichment
- [ ] PDF export + public share link
- [ ] Stripe subscription (Free / Pro / Agency)

---

## TECH STACK

| Layer | Technology | Skill Reference |
|-------|------------|--------------------|
| Backend | FastAPI + Python 3.11+ | skills/BACKEND.md |
| Frontend | React + TypeScript + Vite | skills/FRONTEND.md |
| Database | PostgreSQL + SQLAlchemy | skills/DATABASE.md |
| Auth | JWT + bcrypt + Google OAuth | skills/BACKEND.md |
| UI | Tailwind CSS + Framer Motion | skills/FRONTEND.md |
| LLM Primary | Claude claude-sonnet-4-6 (Anthropic) | services/llm_service.py |
| LLM Fallback | OpenAI GPT-4o | services/llm_service.py |
| Voice TTS | ElevenLabs API | services/voice_service.py |
| Voice STT | OpenAI Whisper API | services/voice_service.py |
| Search | Exa API | services/exa_service.py |
| Payments | Stripe | services/stripe_service.py |
| Storage | AWS S3 | services/storage_service.py |
| Email | SendGrid | services/email_service.py |
| Testing | pytest + React Testing Library | skills/TESTING.md |
| Deployment | Docker + GitHub Actions | skills/DEPLOYMENT.md |

---

## DATABASE MODELS

### User (built-in)
```python
id, email, hashed_password, full_name, is_active, is_verified,
oauth_provider, oauth_id, created_at, updated_at
```

### InterviewSession
```python
id, user_id (FK→User),
target_company (str), target_role (enum: AI_GTM_Engineer|PMM|RevOps),
difficulty (enum: Beginner|Intermediate|Advanced),
mode (enum: text|voice), status (enum: active|completed|abandoned),
total_questions (int=10), current_index (int=0),
overall_score (float nullable), overall_rating (str nullable),
company_context (text nullable),
started_at (datetime), completed_at (datetime nullable),
created_at, updated_at
```

### SessionAnswer
```python
id, session_id (FK→InterviewSession), user_id (FK→User),
question_id (FK→Question), question_index (int),
answer_text (text), audio_s3_key (str nullable),
score_label (enum: Strong|Good|Needs_Work),
score_numeric (int: 10|7|4),
feedback (text), pro_tip (text), improvement_area (str),
created_at
```

### Question
```python
id,
role (enum: AI_GTM_Engineer|PMM|RevOps|General),
category (enum: GTM_Strategy|Tools_Technical|Behavioural|Metrics|Vision|Product_Marketing|Competitive),
difficulty (enum: Beginner|Intermediate|Advanced),
question_text (text), ideal_answer_hint (text), scoring_rubric (text),
is_active (bool=True), created_at, updated_at
```

### PerformanceReport
```python
id, session_id (FK→InterviewSession, unique), user_id (FK→User),
overall_score (float), overall_rating (str),
summary (text), top_strength (str), top_weakness (str),
category_scores (JSON), action_items (JSON),
ready_to_interview (bool), recommended_next_steps (JSON),
pdf_s3_key (str nullable), share_token (str unique),
created_at
```

### Subscription
```python
id, user_id (FK→User, unique),
stripe_customer_id (str), stripe_subscription_id (str),
plan (enum: free|pro|agency), status (enum: active|cancelled|past_due|trialing),
current_period_end (datetime), created_at, updated_at
```

### UsageLog
```python
id, user_id (FK→User), session_id (FK nullable),
event_type (str: session_created|voice_used|pdf_downloaded),
created_at
```

---

## MODULES

### Module 1: Authentication
**Agents:** DATABASE-AGENT + BACKEND-AGENT + FRONTEND-AGENT

**Backend Endpoints:**
| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| POST | /api/v1/auth/register | None | Register with email + password |
| POST | /api/v1/auth/login | None | Login → access + refresh tokens |
| POST | /api/v1/auth/refresh | None | Refresh access token |
| POST | /api/v1/auth/logout | JWT | Revoke refresh token |
| GET | /api/v1/auth/me | JWT | Current user profile |
| GET | /api/v1/auth/google | None | Initiate Google OAuth |
| GET | /api/v1/auth/google/callback | None | OAuth callback → tokens |

**Frontend Pages:**
| Route | Page | Key Components |
|-------|------|----------------|
| /login | LoginPage | EmailPasswordForm, GoogleOAuthButton, AnimatedInput |
| /register | RegisterPage | RegisterForm, PasswordStrengthBar |
| /profile | ProfilePage | ProfileCard, EditProfileForm |
| /settings | SettingsPage | SettingsTabs (profile, billing, notifications) |

---

### Module 2: Interview Session
**Agents:** BACKEND-AGENT + FRONTEND-AGENT (core logic handled by services)

**Backend Endpoints:**
| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| POST | /api/v1/sessions | JWT | Create session → Exa context → first question + audio |
| GET | /api/v1/sessions | JWT | List sessions (paginated, sorted by created_at desc) |
| GET | /api/v1/sessions/{id} | JWT | Session detail + all answers |
| POST | /api/v1/sessions/{id}/answer | JWT | Submit answer → Claude score → next question + audio |
| POST | /api/v1/sessions/{id}/complete | JWT | Force complete → trigger report generation |
| DELETE | /api/v1/sessions/{id} | JWT | Soft delete session |

**Request/Response shapes:**

POST /api/v1/sessions:
```json
Request:  { "target_company": "Rocketlane", "target_role": "AI_GTM_Engineer", "difficulty": "Intermediate", "mode": "text" }
Response: { "session_id": 1, "question": { "id": 4, "text": "...", "category": "GTM_Strategy" }, "audio_url": null, "question_index": 0 }
```

POST /api/v1/sessions/{id}/answer:
```json
Request:  { "answer_text": "I would start by...", "audio_base64": null }
Response: { "score": "Strong", "feedback": "...", "pro_tip": "...", "next_question": {...}, "audio_url": "...", "is_last": false, "session_complete": false }
```

**Frontend Pages:**
| Route | Page | Key Components |
|-------|------|----------------|
| /dashboard | DashboardPage | StatsRow, ProgressChart, CategoryRadar, SessionHistory, QuickStartCard |
| /interview/setup | SetupPage | RoleSelector, CompanyInput, DifficultyPicker, ModeToggle, StartButton |
| /interview/{id} | InterviewRoom | QuestionBubble, AnswerInput|MicButton, AudioPlayer, FeedbackPanel, ProgressBar, TimerDisplay |
| /interview/{id}/report | ReportPage | ScoreSummary, CategoryBarChart, ActionItemList, ShareButton, DownloadPDFButton |

**InterviewRoom component logic:**
```
State: { currentQuestion, currentIndex, isRecording, isLoading, feedback, audioUrl, sessionComplete }

Text mode flow:
  1. Display question text in QuestionBubble
  2. User types in AnswerTextarea
  3. Click Submit → POST /sessions/{id}/answer → show loading
  4. Response: display FeedbackPanel (score badge + feedback + pro_tip)
  5. Play audio_url if voice mode
  6. Click "Next Question" → update state with next_question
  7. If session_complete → redirect to /interview/{id}/report

Voice mode flow:
  1. AudioPlayer plays question audio_url on mount
  2. User clicks MicButton → MediaRecorder starts (WebM/OGG)
  3. On stop → convert to base64 → POST with audio_base64
  4. Backend: Whisper STT → Claude score → ElevenLabs TTS → return
  5. Display transcript + feedback → play audio response
```

**Services (backend/app/services/):**

`interview_service.py`:
```python
async def create_session(db, user_id, data) -> SessionCreateResponse:
    # 1. Check usage limits (free: max 3 sessions)
    # 2. Exa API: fetch company_context for data.target_company
    # 3. Select 10 questions: distribute across categories by difficulty
    # 4. Create InterviewSession row
    # 5. If voice mode: ElevenLabs TTS for intro + first question
    # 6. Return first question + audio_url

async def submit_answer(db, session_id, user_id, answer_text, audio_base64) -> AnswerResponse:
    # 1. Get session + current question
    # 2. If audio_base64: Whisper STT → answer_text
    # 3. Claude scoring: build prompt with question + answer + rubric + company_context
    # 4. Parse JSON response: score_label, feedback, pro_tip, improvement_area
    # 5. Store SessionAnswer
    # 6. Determine next_question (or trigger report if last)
    # 7. ElevenLabs TTS: synthesise feedback + next_question text
    # 8. Return full AnswerResponse

async def generate_report(db, session_id, user_id) -> PerformanceReport:
    # 1. Fetch all SessionAnswers for session
    # 2. Claude: analyse all answers → category_scores, summary, strengths, action_items
    # 3. Create PerformanceReport row with share_token = uuid4()
    # 4. Return report
```

`llm_service.py`:
```python
CLAUDE_SCORING_PROMPT = """
You are a senior GTM Engineering interviewer at an AI-native B2B SaaS startup.
Score the candidate's answer. Return ONLY valid JSON:
{
  "score_label": "Strong|Good|Needs Work",
  "score_numeric": 10|7|4,
  "feedback": "2-3 sentences of direct feedback",
  "pro_tip": "1 actionable improvement tip",
  "improvement_area": "short label"
}

Scoring criteria:
- Strong (10): Specific example with metric OR clear structured thinking with tool knowledge
- Good (7): Solid understanding but lacks specifics or metrics
- Needs Work (4): Too vague, too short, or misses the core of what was asked

QUESTION: {question_text}
SCORING RUBRIC: {scoring_rubric}
COMPANY CONTEXT: {company_context}
CANDIDATE ANSWER: {answer_text}
"""

async def score_answer(question_text, scoring_rubric, company_context, answer_text) -> dict:
    # Primary: Claude claude-sonnet-4-6
    # Fallback: OpenAI GPT-4o (same prompt)
    # Parse JSON. Handle malformed response with default scores.

CLAUDE_REPORT_PROMPT = """
Generate a final performance report. Return ONLY valid JSON:
{
  "overall_score": float (0-10),
  "overall_rating": "Excellent|Strong|Developing|Early Stage",
  "summary": "2-3 sentence assessment",
  "top_strength": "string",
  "top_weakness": "string",
  "category_scores": {"GTM Strategy": int, "Tools Technical": int, "Behavioural": int, "Metrics": int, "Vision": int},
  "action_items": ["item1", "item2", "item3"],
  "ready_to_interview": bool,
  "recommended_next_steps": ["step1", "step2", "step3"]
}
"""
```

`voice_service.py`:
```python
async def transcribe_audio(audio_base64: str) -> str:
    # Decode base64 → temp file → Whisper API → return transcript

async def synthesise_speech(text: str) -> str:
    # ElevenLabs TTS → audio bytes → S3 upload → return presigned URL
    # Voice: Rachel (21m00Tcm4TlvDq8ikWAM), model: eleven_turbo_v2
    # Stability: 0.55, similarity_boost: 0.80
```

`exa_service.py`:
```python
async def get_company_context(company_name: str) -> str:
    # Exa search: "{company_name} product GTM motion ICP customers"
    # Return first 500 chars of top result as context string
    # Cache in Redis or DB to avoid repeated API calls for same company
```

---

### Module 3: Question Bank
**Agents:** DATABASE-AGENT + BACKEND-AGENT + FRONTEND-AGENT (admin)

**Backend Endpoints:**
| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| GET | /api/v1/questions | JWT | List questions (filter: role, category, difficulty) |
| GET | /api/v1/questions/{id} | JWT | Get one question |
| POST | /api/v1/admin/questions | Admin JWT | Create question |
| PUT | /api/v1/admin/questions/{id} | Admin JWT | Update question |
| DELETE | /api/v1/admin/questions/{id} | Admin JWT | Soft delete |
| POST | /api/v1/admin/questions/bulk | Admin JWT | Bulk import JSON array |

**Seed migration:** Create `alembic/versions/002_seed_questions.py` with all 30 questions from GTM_INITIAL.md

**Frontend Pages:**
| Route | Page | Key Components |
|-------|------|----------------|
| /admin/questions | QuestionBankPage | QuestionTable, FilterBar, AddQuestionButton |
| /admin/questions/new | AddQuestionPage | QuestionForm (role/category/difficulty selectors + text areas) |
| /admin/questions/{id}/edit | EditQuestionPage | QuestionForm (pre-filled) |

---

### Module 4: Performance Reports
**Agents:** BACKEND-AGENT + FRONTEND-AGENT

**Backend Endpoints:**
| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| GET | /api/v1/reports/{session_id} | JWT | Get report (auto-generate if not exists) |
| GET | /api/v1/reports/{session_id}/pdf | JWT (Pro) | Generate + download PDF |
| GET | /api/v1/reports/public/{token} | None | Public shared view |
| POST | /api/v1/reports/{session_id}/share | JWT | Generate/return share_token |
| GET | /api/v1/reports | JWT | All my reports |

**Frontend Pages:**
| Route | Page | Key Components |
|-------|------|----------------|
| /interview/{id}/report | ReportPage | OverallScoreCircle, CategoryBarChart, StrengthWeaknessCards, ActionItemList, NextStepsCard, ShareButton, PDFDownloadButton |
| /reports | ReportsListPage | ReportCard (mini summary per session) |
| /r/{token} | PublicReportPage | ReadOnly version of ReportPage |

**ReportPage layout:**
```
┌─────────────────────────────────────────┐
│  Overall Score: 7.4/10  · "Strong"      │
│  Ready to interview: YES / NOT YET       │
├─────────────────────────────────────────┤
│  Summary paragraph (2-3 sentences)      │
├─────────────────────────────────────────┤
│  Category Scores (horizontal bar chart) │
│  GTM Strategy ████████░░  8/10          │
│  Tools & Tech  ██████░░░░  6/10         │
│  Behavioural   ███████░░░  7/10         │
│  Metrics       ███████░░░  7/10         │
│  Vision        █████████░  9/10         │
├─────────────────────────────────────────┤
│  Top Strength      │  Top Weakness      │
│  [strength text]   │  [weakness text]   │
├─────────────────────────────────────────┤
│  3 Action Items                         │
│  1. [action]  2. [action]  3. [action]  │
├─────────────────────────────────────────┤
│  [Share Report]  [Download PDF]         │
│  [Practice Again]                       │
└─────────────────────────────────────────┘
```

---

### Module 5: Subscription & Usage
**Agents:** BACKEND-AGENT + FRONTEND-AGENT

**Backend Endpoints:**
| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| GET | /api/v1/subscription | JWT | Current plan + status + usage |
| POST | /api/v1/subscription/checkout | JWT | Create Stripe checkout session |
| POST | /api/v1/subscription/portal | JWT | Stripe billing portal URL |
| POST | /api/v1/webhooks/stripe | None (signature verify) | Handle subscription events |
| GET | /api/v1/usage | JWT | Sessions used, voice calls, PDF downloads |

**Stripe webhook events to handle:**
```
customer.subscription.created  → create/update Subscription row, plan=pro/agency
customer.subscription.updated  → update status, period_end
customer.subscription.deleted  → set status=cancelled
invoice.payment_failed         → set status=past_due, send email
```

**Usage limit enforcement (middleware/dependencies):**
```python
async def check_session_limit(current_user, db):
    if current_user.subscription.plan == "free":
        session_count = count_sessions(db, current_user.id)
        if session_count >= 3:
            raise HTTPException(402, "Free plan limit reached. Upgrade to Pro.")

async def check_voice_access(current_user):
    if current_user.subscription.plan == "free":
        raise HTTPException(402, "Voice mode requires Pro plan.")

async def check_pdf_access(current_user):
    if current_user.subscription.plan == "free":
        raise HTTPException(402, "PDF export requires Pro plan.")
```

**Frontend Pages:**
| Route | Page | Key Components |
|-------|------|----------------|
| /pricing | PricingPage | PricingCard × 3, FeatureComparisonTable, CTAButton |
| /settings/billing | BillingPage | CurrentPlanCard, UsageBar, ManageBillingButton, UpgradeButton |

**Upgrade prompt component (shown inline when limit hit):**
```
UpgradeModal: "You've used all 3 free sessions. Upgrade to Pro for ₹499/month."
  [Upgrade Now]  [Not Now]
```

---

### Module 6: Dashboard
**Agents:** FRONTEND-AGENT + BACKEND-AGENT (stats endpoint)

**Backend Endpoint:**
```
GET /api/v1/dashboard/stats → {
  total_sessions, completed_sessions, avg_score,
  best_category, current_streak,
  score_history: [{ date, score }],         # last 10 sessions
  category_averages: { GTM_Strategy: float, ... }
}
```

**Frontend Components:**
```typescript
// StatsRow: 4 stat cards with Framer Motion fade-in
{ total_sessions, avg_score, best_category, streak }

// ProgressChart: Recharts LineChart
{ data: score_history, xAxis: date, yAxis: score (0-10) }

// CategoryRadar: Recharts RadarChart
{ data: category_averages, shape: polygon }

// SessionHistory: table with columns
{ date, role, difficulty, overall_score, status, actions: [View Report] }

// QuickStartCard: prominent CTA
{ "Ready for your next interview? → [Start Interview]" }
```

---

## PHASE EXECUTION PLAN

### Phase 1: Foundation (4 agents in parallel — estimated 3 hours)

**DATABASE-AGENT tasks:**
- Create all 7 models with correct relationships and enums
- Write alembic migration `001_initial_schema.py`
- Write seed migration `002_seed_questions.py` (all 30 questions)
- Add database indexes: (session.user_id), (answer.session_id), (question.role, question.difficulty)
- Reference: skills/DATABASE.md

**BACKEND-AGENT tasks:**
- FastAPI project structure: main.py, config.py, database.py, exceptions.py
- All Pydantic schemas for request/response shapes (schemas/)
- Service file stubs: llm_service.py, voice_service.py, exa_service.py, storage_service.py
- Auth module: JWT + Google OAuth (from skills/BACKEND.md)
- CORS configuration for Vite dev server + production domain
- Reference: skills/BACKEND.md

**FRONTEND-AGENT tasks:**
- Vite + React + TypeScript project scaffold
- Tailwind CSS + Framer Motion setup
- Folder structure: components/, pages/, hooks/, services/, context/, types/
- Base components: GlassCard, GradientButton, AnimatedInput, PageWrapper, LoadingSpinner
- API client (services/api.ts) with JWT interceptors + auto-refresh
- AuthContext provider
- React Router setup with protected routes
- Reference: skills/FRONTEND.md

**DEVOPS-AGENT tasks:**
- docker-compose.yml (postgres:16, backend:uvicorn, frontend:nginx, redis:alpine)
- Dockerfile for backend (python:3.11-slim, multi-stage)
- Dockerfile for frontend (node:20-alpine build + nginx serve)
- .env.example with all required variables
- GitHub Actions: `.github/workflows/ci.yml` (lint + test on PR)
- Reference: skills/DEPLOYMENT.md

**Validation Gate 1:**
```bash
alembic upgrade head
pytest backend/tests/ -x  # smoke test
npm install && npm run type-check
docker-compose config --quiet
```

---

### Phase 2: Core Modules (backend + frontend in parallel per module)

**Module 2A — Interview Session Backend (BACKEND-AGENT):**
- `routers/sessions.py`: all 6 endpoints with proper auth dependencies
- `services/interview_service.py`: create_session, submit_answer, generate_report
- `services/llm_service.py`: score_answer (Claude primary + GPT-4o fallback) + generate_report
- `services/voice_service.py`: transcribe_audio (Whisper), synthesise_speech (ElevenLabs)
- `services/exa_service.py`: get_company_context with basic caching
- Error handling: API timeouts, LLM parse failures, S3 upload failures

**Module 2B — Interview Session Frontend (FRONTEND-AGENT):**
- `pages/SetupPage.tsx`: RoleSelector + CompanyInput + DifficultyPicker + ModeToggle
- `pages/InterviewRoom.tsx`: full state machine (loading → question → answering → feedback → next)
  - Voice recording: MediaRecorder API → WebM → base64 encoding
  - Audio playback: HTML5 Audio element + custom controls
  - Typing indicator animation while AI scores
  - FeedbackPanel with score badge (Strong=green, Good=amber, Needs Work=red)
- `hooks/useInterview.ts`: session state management, API calls, audio handling
- `hooks/useMediaRecorder.ts`: mic recording with waveform animation

**Module 3 — Question Bank (BACKEND-AGENT):**
- `routers/questions.py`: list + get endpoints
- `routers/admin/questions.py`: CRUD + bulk import
- Admin role middleware
- Seed data validation

**Module 4 — Performance Reports (BACKEND-AGENT + FRONTEND-AGENT):**
- `routers/reports.py`: get, pdf, public, share endpoints
- PDF generation: WeasyPrint or reportlab → S3 upload
- `pages/ReportPage.tsx`: full report layout with Recharts CategoryBarChart
- `pages/PublicReportPage.tsx`: no-auth version
- Share link copy component

**Module 5 — Subscription (BACKEND-AGENT + FRONTEND-AGENT):**
- `routers/subscription.py`: checkout, portal, webhook
- `services/stripe_service.py`: Stripe SDK integration
- Usage check dependencies (inject into session + voice + PDF endpoints)
- `pages/PricingPage.tsx`: 3-tier pricing cards with feature comparison
- `pages/BillingPage.tsx`: current plan + usage bars + manage button
- `components/UpgradeModal.tsx`: shown when limit hit

**Module 6 — Dashboard (FRONTEND-AGENT + BACKEND-AGENT):**
- `routers/dashboard.py`: GET /dashboard/stats
- `pages/DashboardPage.tsx`: StatsRow + ProgressChart + CategoryRadar + SessionHistory + QuickStart
- Recharts integration: LineChart (score over time) + RadarChart (category averages)

**Validation Gate 2:**
```bash
ruff check backend/ && mypy backend/
npm run lint && npm run type-check
pytest backend/tests/ --cov=app --cov-fail-under=60
```

---

### Phase 3: Quality (3 agents in parallel)

**TEST-AGENT tasks:**
- `tests/test_auth.py`: register, login, refresh, OAuth flow
- `tests/test_sessions.py`: create session, submit answer (mock Claude), complete session
- `tests/test_llm_service.py`: Claude scoring (mock API), fallback to GPT-4o, JSON parse error handling
- `tests/test_voice_service.py`: Whisper STT (mock), ElevenLabs TTS (mock)
- `tests/test_reports.py`: report generation, PDF, share token
- `tests/test_subscription.py`: Stripe webhook handler, usage limits
- Frontend: RTL tests for InterviewRoom state machine (text mode + voice mode)
- Target: 80%+ coverage
- Reference: skills/TESTING.md

**REVIEW-AGENT tasks:**
- Security: SQL injection check, JWT secret strength, Stripe webhook signature verification
- Performance: N+1 queries in session list, eager loading SessionAnswers
- API key security: all keys from env vars, no hardcoding
- Input validation: all request bodies validated with Pydantic, base64 size limits for audio

**Final Validation:**
```bash
pytest --cov --cov-fail-under=80
npm test -- --coverage
docker-compose up -d
curl localhost:8000/health
curl localhost:3000
```

---

## VALIDATION GATES SUMMARY

| Gate | Trigger | Commands |
|------|---------|---------|
| 1 | After Phase 1 | `alembic upgrade head` · `docker-compose config` · `npm install` |
| 2 | After Phase 2 | `ruff check backend/` · `mypy backend/` · `npm run type-check` |
| 3 | After Phase 3 | `pytest --cov --cov-fail-under=80` · `npm test` |
| Final | Before deploy | `docker-compose up -d` · health checks · smoke test all endpoints |

---

## ENVIRONMENT VARIABLES

```env
# Core
DATABASE_URL=postgresql://user:pass@localhost:5432/gtm_interview_coach
SECRET_KEY=your-super-secret-key-min-32-chars
ALGORITHM=HS256
ACCESS_TOKEN_EXPIRE_MINUTES=30
REFRESH_TOKEN_EXPIRE_DAYS=7
ENVIRONMENT=development

# OAuth
GOOGLE_CLIENT_ID=xxx.apps.googleusercontent.com
GOOGLE_CLIENT_SECRET=xxx

# AI / Voice / Search
ANTHROPIC_API_KEY=sk-ant-xxx
OPENAI_API_KEY=sk-xxx
ELEVENLABS_API_KEY=xxx
ELEVENLABS_VOICE_ID=21m00Tcm4TlvDq8ikWAM
EXA_API_KEY=xxx

# Payments
STRIPE_SECRET_KEY=sk_test_xxx
STRIPE_WEBHOOK_SECRET=whsec_xxx
STRIPE_PRO_PRICE_ID=price_xxx
STRIPE_AGENCY_PRICE_ID=price_xxx

# Storage
AWS_ACCESS_KEY_ID=xxx
AWS_SECRET_ACCESS_KEY=xxx
AWS_S3_BUCKET=gtm-interview-coach
AWS_REGION=ap-south-1

# Email
SENDGRID_API_KEY=xxx
FROM_EMAIL=coach@gtminterviewcoach.com

# Frontend
VITE_API_URL=http://localhost:8000
VITE_STRIPE_PUBLISHABLE_KEY=pk_test_xxx
VITE_APP_NAME=GTM Interview Coach
```

---

## OUTPUT DIRECTORY STRUCTURE

```
gtm-interview-coach/
├── backend/
│   ├── app/
│   │   ├── main.py
│   │   ├── config.py
│   │   ├── database.py
│   │   ├── exceptions.py
│   │   ├── models/
│   │   │   ├── __init__.py
│   │   │   ├── user.py
│   │   │   ├── session.py       # InterviewSession + SessionAnswer
│   │   │   ├── question.py
│   │   │   ├── report.py        # PerformanceReport
│   │   │   └── subscription.py  # Subscription + UsageLog
│   │   ├── schemas/
│   │   │   ├── auth.py
│   │   │   ├── session.py
│   │   │   ├── question.py
│   │   │   ├── report.py
│   │   │   └── subscription.py
│   │   ├── routers/
│   │   │   ├── auth.py
│   │   │   ├── sessions.py
│   │   │   ├── questions.py
│   │   │   ├── reports.py
│   │   │   ├── subscription.py
│   │   │   ├── dashboard.py
│   │   │   └── admin/
│   │   │       └── questions.py
│   │   ├── services/
│   │   │   ├── auth_service.py
│   │   │   ├── interview_service.py
│   │   │   ├── llm_service.py      # Claude + GPT-4o
│   │   │   ├── voice_service.py    # ElevenLabs + Whisper
│   │   │   ├── exa_service.py      # Company context
│   │   │   ├── stripe_service.py
│   │   │   ├── storage_service.py  # S3
│   │   │   └── email_service.py    # SendGrid
│   │   └── auth/
│   │       ├── jwt.py
│   │       └── oauth.py
│   ├── alembic/
│   │   └── versions/
│   │       ├── 001_initial_schema.py
│   │       └── 002_seed_questions.py
│   ├── tests/
│   │   ├── conftest.py
│   │   ├── test_auth.py
│   │   ├── test_sessions.py
│   │   ├── test_llm_service.py
│   │   ├── test_voice_service.py
│   │   ├── test_reports.py
│   │   └── test_subscription.py
│   ├── requirements.txt
│   └── Dockerfile
├── frontend/
│   └── src/
│       ├── components/
│       │   ├── ui/
│       │   │   ├── GlassCard.tsx
│       │   │   ├── GradientButton.tsx
│       │   │   ├── AnimatedInput.tsx
│       │   │   ├── ScoreBadge.tsx       # Strong|Good|Needs Work
│       │   │   ├── AudioPlayer.tsx
│       │   │   ├── WaveformRecorder.tsx
│       │   │   └── UpgradeModal.tsx
│       │   ├── layout/
│       │   │   ├── PageWrapper.tsx
│       │   │   ├── Navbar.tsx
│       │   │   └── Sidebar.tsx
│       │   └── charts/
│       │       ├── ProgressLineChart.tsx
│       │       └── CategoryRadarChart.tsx
│       ├── pages/
│       │   ├── LoginPage.tsx
│       │   ├── RegisterPage.tsx
│       │   ├── DashboardPage.tsx
│       │   ├── SetupPage.tsx
│       │   ├── InterviewRoom.tsx
│       │   ├── ReportPage.tsx
│       │   ├── PublicReportPage.tsx
│       │   ├── ReportsListPage.tsx
│       │   ├── PricingPage.tsx
│       │   ├── BillingPage.tsx
│       │   └── admin/
│       │       └── QuestionBankPage.tsx
│       ├── hooks/
│       │   ├── useAuth.ts
│       │   ├── useInterview.ts
│       │   └── useMediaRecorder.ts
│       ├── services/
│       │   ├── api.ts
│       │   ├── sessions.ts
│       │   ├── reports.ts
│       │   └── subscription.ts
│       ├── context/
│       │   └── AuthContext.tsx
│       ├── types/
│       │   └── index.ts
│       └── App.tsx
├── docker-compose.yml
├── .env.example
├── .github/
│   └── workflows/
│       ├── ci.yml
│       └── deploy.yml
├── GTM_INITIAL.md
└── PRPs/
    └── gtm-interview-coach-prp.md   ← this file
```

---

## AGENT DISPATCH INSTRUCTIONS

### DATABASE-AGENT
```yaml
TO: DATABASE-AGENT
TASK: Create all models, migrations, and seed data
READ: skills/DATABASE.md
CREATE:
  - backend/app/models/user.py
  - backend/app/models/session.py      (InterviewSession + SessionAnswer)
  - backend/app/models/question.py
  - backend/app/models/report.py
  - backend/app/models/subscription.py (Subscription + UsageLog)
  - alembic/versions/001_initial_schema.py
  - alembic/versions/002_seed_questions.py  (all 30 questions from INITIAL.md)
VALIDATION: alembic upgrade head && python -c "from app.models import *; print('OK')"
```

### BACKEND-AGENT
```yaml
TO: BACKEND-AGENT
TASK: Build all FastAPI endpoints and services
READ: skills/BACKEND.md
DEPENDS ON: DATABASE-AGENT output
CREATE:
  - app/main.py, config.py, database.py, exceptions.py
  - All routers (auth, sessions, questions, reports, subscription, dashboard, admin)
  - All services (interview, llm, voice, exa, stripe, storage, email, auth)
  - All Pydantic schemas
VALIDATION: ruff check backend/ && uvicorn app.main:app --host 0.0.0.0 --port 8000
```

### FRONTEND-AGENT
```yaml
TO: FRONTEND-AGENT
TASK: Build all React pages, components, and hooks
READ: skills/FRONTEND.md
DEPENDS ON: BACKEND-AGENT (API contract from schemas)
CREATE:
  - All pages (LoginPage, RegisterPage, DashboardPage, SetupPage, InterviewRoom,
               ReportPage, PublicReportPage, PricingPage, BillingPage)
  - All UI components (GlassCard, GradientButton, ScoreBadge, AudioPlayer,
                       WaveformRecorder, ProgressLineChart, CategoryRadarChart, UpgradeModal)
  - All hooks (useAuth, useInterview, useMediaRecorder)
  - API services layer
VALIDATION: npm run lint && npm run type-check && npm run build
```

### DEVOPS-AGENT
```yaml
TO: DEVOPS-AGENT
TASK: Docker + CI/CD setup
READ: skills/DEPLOYMENT.md
CREATE:
  - docker-compose.yml (postgres, backend, frontend, redis)
  - backend/Dockerfile
  - frontend/Dockerfile
  - .env.example
  - .github/workflows/ci.yml
  - .github/workflows/deploy.yml
VALIDATION: docker-compose config --quiet && docker-compose build
```

### TEST-AGENT
```yaml
TO: TEST-AGENT
TASK: Write tests achieving 80%+ coverage
READ: skills/TESTING.md
DEPENDS ON: All agents complete
CREATE:
  - tests/conftest.py (fixtures, mock Claude, mock ElevenLabs, mock Whisper)
  - tests/test_auth.py
  - tests/test_sessions.py
  - tests/test_llm_service.py
  - tests/test_voice_service.py
  - tests/test_reports.py
  - tests/test_subscription.py
VALIDATION: pytest --cov=app --cov-report=term-missing --cov-fail-under=80
```

---

## NEXT STEP

```bash
/execute-prp PRPs/gtm-interview-coach-prp.md
```

This will activate the ORCHESTRATOR which dispatches all agents in the correct phase order.

---

## PRP GENERATION REPORT

```
═══════════════════════════════════════════════════════════
                   PRP GENERATED
═══════════════════════════════════════════════════════════

Product:    GTM Interview Coach
File:       PRPs/gtm-interview-coach-prp.md
Complexity: High

Modules:    6
Models:     7  (User, InterviewSession, SessionAnswer, Question,
                PerformanceReport, Subscription, UsageLog)
Endpoints:  28
Pages:      14
Services:   8  (interview, llm, voice, exa, stripe, storage, email, auth)
Components: 12+
Seed Data:  30 questions (3 roles × 3 difficulties × ~3-4 questions each)

Phases:
  1. Foundation:  4 agents parallel  (DATABASE + BACKEND + FRONTEND + DEVOPS)
  2. Modules:     6 modules          (auth, sessions, questions, reports, subscription, dashboard)
  3. Quality:     3 agents parallel  (TEST + REVIEW + RESEARCH)

External APIs:
  - Anthropic Claude claude-sonnet-4-6  (primary LLM scoring)
  - OpenAI GPT-4o              (fallback LLM)
  - OpenAI Whisper             (STT — voice mode)
  - ElevenLabs eleven_turbo_v2 (TTS — voice mode)
  - Exa Search                 (company context enrichment)
  - Stripe                     (subscriptions)
  - AWS S3                     (audio + PDF storage)
  - SendGrid                   (transactional email)

Monetisation:
  Free:   3 sessions, text mode only
  Pro:    ₹499/month — unlimited + voice + PDF
  Agency: ₹1999/month — white-label + custom questions

NEXT: /execute-prp PRPs/gtm-interview-coach-prp.md
═══════════════════════════════════════════════════════════
```
