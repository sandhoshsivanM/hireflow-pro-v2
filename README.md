# HireFlow Pro v2

Track job applications in one place and check how well your resume fits a role before you apply.

**.NET 10 · ASP.NET Core · EF Core · PostgreSQL · React 19 · TypeScript · Docker · Claude & Gemini APIs**

---

## What it does

- **Tracks applications** through Saved → Applied → Interview → Offer → Rejected → Ghosted, with a Kanban view, follow-up dates, contacts, a per-application timeline and CSV export.
- **Scores resume fit.** Paste a job description and get a 0–100 match score, the keywords you're missing, and suggestions.
- **Tailors a resume** to a role: summary, skills, bullet points and a cover-letter draft, exported as a PDF.
- **Finds jobs** from free boards (Remotive, Arbeitnow) and saves them to the tracker in one click.
- **Admin panel** with stats, user management and impersonation for support.
- **Chrome extension** (`hireflow-extension/`) saves jobs straight from LinkedIn, Indeed and Glassdoor.

## Architecture

```
hireflow-ui/                 React 19 + Vite SPA
        │  REST, JWT bearer
HireFlowPro.Api/             8 controllers, 45 endpoints, middleware, DI
HireFlowPro.Infrastructure/  EF Core DbContext, migrations, AI + job-board services
HireFlowPro.Core/            Entities, DTOs, interfaces — no dependencies
        │
PostgreSQL (prod) · SQLite (local dev)
```

Core has no outward dependencies, and Infrastructure implements Core's interfaces, so the AI provider or the database can be swapped without touching the API.

## Engineering decisions

**Two AI providers behind one interface.** `AIService` calls Claude by default and Gemini as the alternative. Prompts and response parsing are shared, so switching provider is a config change.

**AI spend is capped per user.** Every AI endpoint checks the user's monthly quota before calling the provider and records the use afterwards (`AIQuotaLimits` in `HireFlowPro.Core/Entities/AIUsage.cs`):

| Plan    | AI calls / month |
|---------|------------------|
| Free    | 5                |
| Pro     | 90               |
| Premium | Unlimited        |

**Passwords and sessions.** PBKDF2 password hashing, JWT bearer tokens, security headers on every response and a restricted CORS policy.

## API

| Area | Route | Endpoints |
|------|-------|-----------|
| Auth | `/api/auth` | 8 |
| Applications | `/api/applications` | 12 |
| AI (quota-enforced) | `/api/ai` | 5 |
| Resume profile | `/api/resume-profile` | 2 |
| Job discovery | `/api/jobs` | 1 |
| Billing | `/api/billing` | 4 |
| Admin (admin only) | `/api/admin` | 12 |
| Health | `/health` | 1 |

## Run it locally

Requirements: .NET 10 SDK and Node 20+. PostgreSQL is optional; without it the API uses SQLite.

```bash
git clone https://github.com/sandhoshsivanM/hireflow-pro-v2.git
cd hireflow-pro-v2

dotnet run --project HireFlowPro.Api          # API

cd hireflow-ui && npm install && npm run dev  # UI → http://localhost:5173
```

Set `AI__Claude__ApiKey` (or the Gemini key) to enable the AI features.

## Deploy

One multi-stage Dockerfile builds the React app and the .NET API into a single image that serves both on port 5000. `render.yaml` deploys it to Render with a managed PostgreSQL database, and every push to `main` redeploys.

```bash
docker build -t hireflowpro .
docker run -p 5000:5000 \
  -e DATABASE_URL="postgres://user:pass@host:5432/db" \
  -e Jwt__Key="<32+ char secret>" \
  -e AI__Claude__ApiKey="<key>" \
  hireflowpro
```

## Status

Billing endpoints and plan tiers are in place. Real Stripe checkout is not wired up yet; `POST /api/billing/checkout` currently changes the plan directly.
