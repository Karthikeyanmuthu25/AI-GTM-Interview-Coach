# INITIAL.md — AI GTM Interview Coach

> AI-powered voice interview coach for GTM engineering and product marketing roles.
> Run: /generate-prp GTM_INITIAL.md

---

## PRODUCT

**Name:** GTM Interview Coach

**Tagline:** Practice like a pro. Land the GTM role.

**Description:**
An AI-powered interview coaching SaaS for B2B SaaS professionals targeting AI GTM Engineering,
Product Marketing, and RevOps roles. Users select a target company + role, then engage in a live
voice or text mock interview. An AI interviewer asks real questions, scores each answer via Claude,
speaks feedback via ElevenLabs TTS, and generates a structured performance report with
per-category scores, improvement areas, and a 30-day action plan.

**Who it's for:**
- Mid-career B2B SaaS marketers targeting GTM Engineering / PMM roles (10-14 LPA range)
- Job seekers with skills gaps wanting structured feedback before real interviews
- Founders wanting to practice for investor or sales conversations

**Type:** SaaS (Freemium → Paid)

**Monetisation:**
- Free: 3 interview sessions total, text-only mode
- Pro (₹499/month): Unlimited sessions, voice mode, full PDF reports, saved history
- Agency (₹1999/month): White-label for recruiters, custom question banks, bulk candidate management

---

## TECH STACK

| Layer | Choice |
|-------|--------|
| Backend | FastAPI + Python 3.11+ |
| Frontend | React + TypeScript + Vite |
| Database | PostgreSQL + SQLAlchemy |
| Auth | JWT + Google OAuth |
| UI | Tailwind + Framer Motion |
| Payments | Stripe |
| LLM Primary | Claude (claude-sonnet-4-6) via Anthropic API |
| LLM Fallback | OpenAI GPT-4o |
| Voice TTS | ElevenLabs API (Rachel voice, eleven_turbo_v2) |
| Voice STT | OpenAI Whisper API |
| Search | Exa API (company context enrichment) |
| Email | SendGrid |
| Storage | AWS S3 (audio recordings, PDF reports) |

---

## MODULES

### Module 1: Authentication (Built-in)

**Models:** User, RefreshToken

**Endpoints:**
- POST /auth/register
- POST /auth/login
- POST /auth/refresh
- GET  /auth/me
- GET  /auth/google
- POST /auth/logout

**Pages:** /login, /register, /profile, /settings

---

### Module 2: Interview Session

**Description:**
Core product module. User configures role, company, difficulty, mode. Backend creates session,
fetches company context via Exa, pulls 10 questions. Each submitted answer is scored by Claude,
with ElevenLabs TTS for voice responses. Full session state persisted in DB.

**Models:**
```
InterviewSession:
  - id, user_id (FK)
  - target_company: str
  - target_role: str          # "AI_GTM_Engineer" | "PMM" | "RevOps"
  - difficulty: enum          # "Beginner" | "Intermediate" | "Advanced"
  - mode: enum                # "text" | "voice"
  - status: enum              # "active" | "completed" | "abandoned"
  - total_questions: int = 10
  - current_index: int = 0
  - overall_score: float (nullable)
  - overall_rating: str (nullable)
  - company_context: text (Exa-fetched)
  - started_at, completed_at, created_at, updated_at

SessionAnswer:
  - id, session_id (FK), user_id (FK), question_id (FK)
  - question_index: int
  - answer_text: text
  - audio_s3_key: str (nullable)
  - score_label: enum         # "Strong" | "Good" | "Needs Work"
  - score_numeric: int        # 10 | 7 | 4
  - feedback: text
  - pro_tip: text
  - improvement_area: str
  - created_at
```

**Endpoints:**
```
POST   /api/v1/sessions                     - Create session → returns first question + audio_url
GET    /api/v1/sessions                     - List user sessions (paginated)
GET    /api/v1/sessions/{id}                - Get session detail + all answers
POST   /api/v1/sessions/{id}/answer         - Submit answer (text or base64 audio)
POST   /api/v1/sessions/{id}/complete       - Force complete session
DELETE /api/v1/sessions/{id}                - Delete session
```

**Pages:**
```
/dashboard             - Session history + stats
/interview/setup       - Role + company + difficulty + mode selector
/interview/{id}        - Live interview room
/interview/{id}/report - Post-session performance report
```

**Key service logic (services/interview_service.py):**
1. Session create: call Exa for company_context, pull 10 questions by difficulty distribution
2. Answer submit voice: Whisper STT → transcript. Answer submit any: Claude scoring API call
3. Claude prompt includes: question + answer + company_context + scoring rubric → JSON response
4. ElevenLabs TTS: synthesise feedback + next question → stream audio → store S3 key
5. LLM fallback: Claude timeout/error → retry once → fallback to GPT-4o same prompt
6. Final answer: trigger PerformanceReport generation via Claude → store in DB

---

### Module 3: Question Bank

**Description:**
Admin-managed question database. 30 seed questions across 3 roles × 3 difficulties.
Questions carry scoring rubric used to guide Claude's evaluation prompt.

**Models:**
```
Question:
  - id
  - role: enum        # "AI_GTM_Engineer" | "PMM" | "RevOps" | "General"
  - category: enum    # "GTM_Strategy" | "Tools_Technical" | "Behavioural" | "Metrics" | "Vision" | "Product_Marketing" | "Competitive"
  - difficulty: enum  # "Beginner" | "Intermediate" | "Advanced"
  - question_text: text
  - ideal_answer_hint: text
  - scoring_rubric: text
  - is_active: bool = True
  - created_at, updated_at
```

**Endpoints:**
```
GET    /api/v1/questions                 - List (filter: role, category, difficulty)
GET    /api/v1/questions/{id}            - Get one
POST   /api/v1/admin/questions           - Create (admin only)
PUT    /api/v1/admin/questions/{id}      - Update (admin only)
DELETE /api/v1/admin/questions/{id}      - Soft delete (admin only)
POST   /api/v1/admin/questions/bulk      - Bulk import JSON (admin only)
```

**Pages:**
```
/admin/questions           - Question management table
/admin/questions/new       - Add question
/admin/questions/{id}/edit - Edit question
```

---

### Module 4: Performance Reports

**Description:**
AI-generated report at session end. Category scores, top strength/weakness, 3 action items,
readiness flag. PDF downloadable. Shareable via public token link.

**Models:**
```
PerformanceReport:
  - id, session_id (FK), user_id (FK)
  - overall_score: float
  - overall_rating: str
  - summary: text
  - top_strength: str
  - top_weakness: str
  - category_scores: json       # {"GTM Strategy": 8, "Tools": 6, ...}
  - action_items: json          # ["action1", "action2", "action3"]
  - ready_to_interview: bool
  - recommended_next_steps: json
  - pdf_s3_key: str (nullable)
  - share_token: str (unique)
  - created_at
```

**Endpoints:**
```
GET    /api/v1/reports/{session_id}        - Get report
GET    /api/v1/reports/{session_id}/pdf    - Download PDF (Pro only)
GET    /api/v1/reports/public/{token}      - Public view (no auth)
POST   /api/v1/reports/{session_id}/share  - Generate share link
```

**Pages:**
```
/interview/{id}/report    - Full report page
/reports                  - All my reports
/r/{token}                - Public report (no login)
```

---

### Module 5: Subscription & Usage

**Description:**
Stripe-powered subscription management. Free: 3 sessions, text only. Pro: unlimited + voice + PDF.
Usage limits enforced per API call with clear upgrade prompts in UI.

**Models:**
```
Subscription:
  - id, user_id (FK)
  - stripe_customer_id, stripe_subscription_id
  - plan: enum           # "free" | "pro" | "agency"
  - status: enum         # "active" | "cancelled" | "past_due"
  - current_period_end: datetime
  - created_at, updated_at

UsageLog:
  - id, user_id (FK), session_id (FK nullable)
  - event_type: str      # "session_created" | "voice_used" | "pdf_downloaded"
  - created_at
```

**Endpoints:**
```
GET    /api/v1/subscription              - Get current plan + usage
POST   /api/v1/subscription/checkout     - Stripe checkout session
POST   /api/v1/subscription/portal       - Stripe billing portal
POST   /api/v1/webhooks/stripe           - Webhook handler (no auth)
GET    /api/v1/usage                     - Usage stats
```

**Pages:**
```
/pricing              - Plans comparison
/settings/billing     - Subscription management
```

---

### Module 6: Dashboard

**Description:**
User home. Session history table, score progression chart (Recharts line chart),
category radar chart, stats row, quick-start CTA.

**Pages:**
```
/dashboard
  Components:
    - StatsRow          (total sessions, avg score, best category, streak)
    - ProgressChart     (score over time — Recharts LineChart)
    - CategoryRadar     (radar chart of 5 category scores — Recharts RadarChart)
    - SessionHistory    (recent sessions table with status + score + report link)
    - QuickStartCard    (prominent CTA: "Start New Interview")
```

---

## QUESTION SEED DATA (30 questions — use in alembic seed migration)

### AI GTM Engineer — Beginner (3 questions)
1. "Tell me about yourself and why you are interested in AI GTM engineering roles." | Category: Introduction
2. "What do you understand by ICP — Ideal Customer Profile? How would you define one?" | Category: GTM_Strategy
3. "Have you used any sales or marketing automation tools? Walk me through one." | Category: Tools_Technical

### AI GTM Engineer — Intermediate (4 questions)
4. "Walk me through how you would build an AI-powered outbound sequence from scratch for a B2B SaaS product." | Category: GTM_Strategy
5. "You join a startup and their outbound response rate is 1.5%. What is your diagnosis and first 30-day plan?" | Category: GTM_Strategy
6. "Have you worked with Clay, Apollo, or similar GTM automation tools? Walk me through a specific workflow." | Category: Tools_Technical
7. "How do you measure the success of a GTM engineering initiative?" | Category: Metrics

### AI GTM Engineer — Advanced (3 questions)
8. "Design a full GTM motion for an AI dev-tools product targeting senior engineers at Series B+ startups." | Category: GTM_Strategy
9. "How would you architect a Clay-based lead enrichment pipeline using LinkedIn, Crunchbase, and job postings?" | Category: Tools_Technical
10. "What is your view on the future of SDRs given AI automation? Where do humans still win?" | Category: Vision

### PMM — Beginner (3 questions)
11. "What is positioning and why does it matter for a new product?" | Category: Product_Marketing
12. "How would you describe a messaging hierarchy? Give an example." | Category: Product_Marketing
13. "Tell me about a time you wrote copy that converted. What made it work?" | Category: Behavioural

### PMM — Intermediate (4 questions)
14. "How do you approach competitive analysis? Walk me through your process for a new product entering a crowded market." | Category: Competitive
15. "Walk me through how you would position an AI product for a non-technical buyer." | Category: Product_Marketing
16. "How do you enable a sales team to use a new product launch story effectively?" | Category: GTM_Strategy
17. "Tell me about a product launch you owned or contributed to. What was your role and what was the outcome?" | Category: Behavioural

### PMM — Advanced (3 questions)
18. "You are launching an AI-native product into a market with one dominant player. Build the full positioning and launch strategy." | Category: Product_Marketing
19. "How do you measure whether your messaging is resonating with your ICP? What signals do you track?" | Category: Metrics
20. "How does product marketing change when the buyer is also the user versus when they are different people?" | Category: Product_Marketing

### RevOps — Beginner (2 questions)
21. "What is RevOps and how does it differ from sales ops?" | Category: GTM_Strategy
22. "What CRM tools have you worked with? What did you use them for?" | Category: Tools_Technical

### RevOps — Intermediate (3 questions)
23. "How do you build a revenue attribution model? What are the trade-offs between first-touch and multi-touch?" | Category: Metrics
24. "Walk me through how you would diagnose and fix a leaky sales funnel." | Category: GTM_Strategy
25. "What does a healthy sales pipeline look like? What leading indicators would you track?" | Category: Metrics

### RevOps — Advanced (2 questions)
26. "Design a RevOps function from scratch for a 30-person B2B SaaS company scaling from $1M to $5M ARR." | Category: GTM_Strategy
27. "How would you build a unified data model across marketing, sales, and CS to enable true revenue attribution?" | Category: Metrics

### General — Behavioural (3 questions)
28. "Tell me about a time you had to learn a new skill under pressure. How did you do it?" | Category: Behavioural
29. "Describe a situation where you disagreed with a decision. How did you handle it?" | Category: Behavioural
30. "Where do you see AI GTM engineering going in the next two years? What shift are you most excited about?" | Category: Vision

---

## MVP SCOPE

Must Have (Week 1–2):
- [x] User registration / login (JWT + Google OAuth)
- [x] Text-mode interview session with Claude scoring
- [x] Question bank seeded with 30 questions
- [x] Performance report generated at session end
- [x] Dashboard with session history and score chart

Phase 2 (Week 3–4):
- [ ] Voice mode (ElevenLabs TTS + Whisper STT)
- [ ] Company context enrichment via Exa API
- [ ] PDF report export to AWS S3
- [ ] Public shareable report link

Phase 3 (Week 5–6):
- [ ] Stripe subscription (Free / Pro / Agency)
- [ ] Usage limits enforcement
- [ ] Admin question bank UI
- [ ] SendGrid onboarding email sequence

---

## ACCEPTANCE CRITERIA

- [ ] User registers, logs in, views profile (email + Google OAuth)
- [ ] User starts text interview for any of 3 roles at any difficulty
- [ ] Claude scores each answer and returns feedback within 5 seconds
- [ ] Session completes after 10 answers and shows performance report
- [ ] Voice mode: mic → Whisper transcript → Claude score → ElevenLabs audio response
- [ ] Exa company context is included in Claude scoring prompt
- [ ] Dashboard shows progress chart and session history
- [ ] Free users blocked after 3 sessions with upgrade prompt
- [ ] 80%+ pytest coverage on backend services
- [ ] docker-compose up -d starts all services cleanly
- [ ] All API endpoints return proper error shapes (code, message, detail)

---

## RUN

```bash
/generate-prp GTM_INITIAL.md
/execute-prp PRPs/gtm-interview-coach-prp.md
```
