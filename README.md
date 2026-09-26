# TAI Skill Checker

AI-powered workforce skill assessment platform. Administrators design scenario-based voice assessments, employees complete them in a real-time conversation with an AI interviewer, and an independent evaluator produces evidence-backed competency reports.

## How it works

1. **Build** – An admin creates an assessment manually or generates scenarios, competencies, and rubrics from a brief and uploaded PDFs, then publishes it as an immutable version.
2. **Invite** – The admin invites employees by email. Each invitation carries a single-use, time-limited magic link.
3. **Assess** – The employee prepares, passes a microphone check, and joins a LiveKit room where a Gemini Live interviewer works through the fixed scenarios with adaptive follow-ups.
4. **Evaluate** – When the interview completes, a queued evaluator scores each competency 1–5, citing specific transcript turns as evidence.
5. **Review** – Admins read reports, replay private recordings, and can listen in on live sessions in listen-only mode.

Employees never see rubrics, scores, or evaluation output.

## Repository layout

```
.
├── backend/    Express 5 API, LiveKit agent worker, evaluation worker, Drizzle migrations
└── frontend/   React 19 + Vite admin workspace and employee assessment app
```

`backend/` and `frontend/` are git submodules pointing at their own repositories ([tai-skill-checker](https://github.com/pasiduR/tai-skill-checker) and [tai-skill-checker-web](https://github.com/pasiduR/tai-skill-checker-web)). Clone with submodules:

```sh
git clone --recurse-submodules https://github.com/pasiduR/tai-skill-checker-mono.git
# or, in an existing clone:
git submodule update --init
```

Commit and push changes inside `backend/` or `frontend/` to their own repos first, then commit the updated submodule pointer here.

Each package has its own README with detailed setup, and its own `AGENT.md` with coding conventions:

- [backend/README.md](backend/README.md) – environment, migrations, Docker/Dokploy deployment, API reference, domain guarantees
- [frontend/README.md](frontend/README.md) – local dev, Boneyard skeletons, checks

## Tech stack

| Area | Technology |
| --- | --- |
| Frontend | React 19, Vite 7, TypeScript, Tailwind CSS 4, Radix UI, TanStack Query, React Router 7 |
| Backend | Node.js 22+, Express 5, TypeScript, Zod, Pino |
| Database | Neon PostgreSQL, Drizzle ORM, pgvector |
| Auth | Better Auth (admin password, employee magic links) |
| Realtime voice | LiveKit rooms, agents, and audio egress |
| AI | Gemini Live (interviewer), Gemini (content generation, embeddings), Anthropic Claude (evaluation) |
| Storage | Cloudflare R2 (private recordings) |
| Email | Resend |

## Prerequisites

- Node.js 22 or newer
- A Neon (or other PostgreSQL) database with the `vector` extension available
- Accounts and keys for LiveKit, Google Gemini, Anthropic, Cloudflare R2, and Resend

## Quick start

**1. Backend**

```sh
cd backend
cp .env.example .env        # fill in database, auth, admin, and provider keys
npm install
npm run db:migrate
npm run dev                  # API on http://localhost:3000
```

In a second terminal, start the LiveKit interviewer worker:

```sh
cd backend
npm run agent:dev
```

Set a unique `LIVEKIT_AGENT_NAME` for local development so LiveKit does not dispatch your test sessions to another worker.

**2. Frontend**

```sh
cd frontend
cp .env.example .env        # VITE_API_URL defaults to http://localhost:3000
npm install
npm run dev                  # app on http://localhost:5173
```

**3. Sign in**

Open `http://localhost:5173/admin/sign-in` and use the `ADMIN_EMAIL` / `ADMIN_PASSWORD` from `backend/.env`. The admin account and organization are provisioned automatically on first API start.

## Key routes

| Route | Who | Purpose |
| --- | --- | --- |
| `/admin` | Admin | Operations overview |
| `/admin/assessments` | Admin | Create, edit, publish, and invite |
| `/admin/evaluations` | Admin | Evaluation queue and reports |
| `/admin/live` | Admin | Monitor and listen to active sessions |
| `/admin/settings` | Admin | Voice agent configuration and test calls |
| `/admin/docs` | Admin | In-app user manual |
| `/employee/sign-in` | Employee | Request a fresh magic link |
| `/employee` | Employee | Assigned assessments |

## Checks

Run from either `backend/` or `frontend/`:

```sh
npm run typecheck
npm test
npm run build
```

The frontend also has `npm run lint`.

## Deployment

- **Backend** – One Docker image runs both the API and the agent worker. See `backend/compose.yaml` and the Docker section of [backend/README.md](backend/README.md). Run `npm run db:migrate:prod` once per release before starting the new API.
- **Frontend** – Static Vite build; `frontend/vercel.json` configures SPA rewrites for Vercel.

## Documentation

- User manual: sign in as admin and open **User manual** in the sidebar (`/admin/docs`).
- Product requirements: [REQUIREMENT.md](REQUIREMENT.md)
"# tai-skill-checker-mono" 
