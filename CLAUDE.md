# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Architecture

Ledger CFO is a two-part system with a split deployment:

- **Frontend** (`/Users/aldenhoemann/Desktop/Ledger CFO/`) — Plain HTML/CSS/JS static files, deployed to Netlify via drag-and-drop. No build step, no framework.
- **Backend** (`/Users/aldenhoemann/Desktop/backend/`) — Node.js + Express, deployed to Railway via GitHub auto-deploy (`https://github.com/aldenaureh-crypto/ledger-cfo-backend`).

The frontend calls the backend at `https://ledger-cfo-backend-production.up.railway.app`. This URL is hardcoded as `const API=...` at the top of each HTML file's `<script>` block.

## Backend

### Running locally
```bash
cd /Users/aldenhoemann/Desktop/backend
npm run dev   # uses nodemon for auto-restart
```

### Required env vars (Railway + local .env)
```
ANTHROPIC_API_KEY
SUPABASE_URL
SUPABASE_SERVICE_KEY   # admin client, bypasses RLS
SUPABASE_ANON_KEY      # auth client, for token verification
STRIPE_SECRET_KEY
STRIPE_WEBHOOK_SECRET
PORT                   # Railway sets this automatically
```

### Key architectural decisions
- Two Supabase clients: `supabase` (service key, admin) for DB operations, `supabaseAuth` (anon key) for verifying user JWT tokens
- Stripe webhook route (`/api/stripe/webhook`) uses `express.raw()` and must be registered **before** `express.json()` — order matters
- All auth-protected routes call `getUser(req)` which verifies the Bearer token via `supabaseAuth.auth.getUser(token)`

### API routes
- `POST /api/auth/signup` — creates Supabase user + profiles row
- `POST /api/auth/login` — returns session token
- `GET/POST /api/profile` — get or update business profile
- `POST /api/chat` — main AI endpoint; checks message limits for starter plan, builds system prompt from profile + finances_manual, calls Claude
- `POST /api/stripe/checkout` — creates Stripe checkout session, returns redirect URL
- `POST /api/stripe/portal` — opens Stripe billing portal
- `POST /api/stripe/webhook` — handles subscription lifecycle, updates plan in profiles table

## Frontend

### Deploying
Select all files in `/Users/aldenhoemann/Desktop/Ledger CFO/` **except** the `backend` folder, drag onto Netlify deploy dropzone. No build step needed.

### Auth pattern
Every page checks `localStorage.getItem('lcfo_token')` and redirects to `login.html` if missing. Profile data is fetched fresh from `/api/profile` on load — localStorage (`lcfo_plan`, `lcfo_business`, `lcfo_revenue`) is only used for fast initial render.

### Feature gating
Plan is read from the API response (`profile.plan`), not trusted from localStorage alone. Gating logic:
- `starter` → chat only, 15 msg/month, dashboard shows upgrade gate
- `pro` → dashboard unlocked, unlimited chat
- `scale` → reports unlocked (pro also sees reports upgrade gate)

### Financial data
`finances_manual` is a JSONB column in `profiles`. Structure: `{ expenses: [{name, amount}], cash_balance, total_expenses, outstanding_invoices }`. The dashboard health score and stat cards only display when this data exists — nothing is estimated or faked.

## Supabase schema
```
profiles: id, business_name, owner_name, industry, monthly_revenue, employees, goal, plan, trial_ends_at, finances_manual (JSONB), stripe_customer_id, stripe_subscription_id
conversations: id, user_id, title, created_at
messages: id, conversation_id, role, content, created_at
waitlist: email
```

## Stripe
Currently in **sandbox mode**. Price IDs:
- Starter: `price_1TekZyLd5E4MDHh9EVPdhfrs`
- Pro: `price_1TekaRLd5E4MDHh9XGCjyoOQ`
- Scale: `price_1Tekb7Ld5E4MDHh9YiqgTepY`

`payment_method_collection: 'if_required'` is set intentionally — no card required during trial (demo phase). Remove this line when going live.

## AI system prompt
The chat endpoint builds a dynamic system prompt per user including: business profile, industry-specific terminology (e.g. restaurant → food cost %, table turns; SaaS → MRR, churn, CAC), and the full `finances_manual` data broken down into calculated metrics (margin %, net profit). The prompt is in `server.js` around the `systemPrompt` variable.

## Test accounts
- `test.starter@ledgercfo.dev` / `TestLedger123!` — Starter plan
- `test.pro@ledgercfo.dev` / `TestLedger123!` — Pro plan, has finances_manual set
- `test.scale@ledgercfo.dev` / `TestLedger123!` — Scale plan
