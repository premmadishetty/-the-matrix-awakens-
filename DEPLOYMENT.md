<div align="center">

```
╔══════════════════════════════════════════════════════╗
║   D E P L O Y M E N T   G U I D E                   ║
║   "Act without expectation of reward." — Gita 2.47   ║
╚══════════════════════════════════════════════════════╝
```

</div>

# Deployment Guide

> Full zero-cost edge deployment — Cloudflare Workers + Vercel. No servers. No VMs. No monthly bills.

---

## Prerequisites

| Tool | Version | Install |
|------|---------|---------|
| Node.js | ≥ 18 | [nodejs.org](https://nodejs.org) |
| npm | ≥ 9 | Bundled with Node |
| Git | Any | [git-scm.com](https://git-scm.com) |
| Wrangler CLI | v3+ | `npm install -g wrangler` |

---

## Step 1 — External Services Setup

### MongoDB → Not used. Cloudflare D1 replaces it entirely.

### Groq API (AI Inference)
1. Sign up at [console.groq.com](https://console.groq.com)
2. Create API Key → copy it
3. Free tier: 14,400 tokens/min — sufficient for portfolio traffic

### Resend (Email Alerts)
1. Sign up at [resend.com](https://resend.com)
2. Verify your domain OR use the sandbox `onboarding@resend.dev`
3. Create API Key → copy it

### Cloudflare Account
1. Sign up at [cloudflare.com](https://cloudflare.com) — free
2. No credit card required for Workers + D1 free tier

---

## Step 2 — Backend (Cloudflare Workers)

```bash
# Navigate to backend folder
cd backend

# Install dependencies (hono + wrangler)
npm install

# Authenticate with Cloudflare
npx wrangler login
# → Opens browser, sign in with your Cloudflare account

# Create D1 database
npx wrangler d1 create matrix-karma-db
# → Copy the database_id from the output
# → Paste it into wrangler.toml: database_id = "YOUR_ID_HERE"

# Initialize database schema (creates all 5 tables)
npm run db:init:remote
# → Should output: "Applied 1 migration"

# Set secrets (one by one — paste your real values when prompted)
npx wrangler secret put GROQ_API_KEY
npx wrangler secret put RESEND_API_KEY
npx wrangler secret put ADMIN_EMAIL
npx wrangler secret put ADMIN_SECRET

# Test locally first
npm run dev
# → Visit: http://localhost:8787/api/health
# → Expected: { "status": "[MATRIX_ONLINE]" }

# Deploy to Cloudflare edge
npm run deploy
# → Your Worker URL: https://the-matrix-awakens.YOUR_SUBDOMAIN.workers.dev
```

---

## Step 3 — Frontend (Vercel)

```bash
# Navigate to frontend
cd frontend

# Install dependencies
npm install

# Test locally (requires Worker running)
npm run dev
# → http://localhost:5173
```

**Vercel deployment:**
1. Push code to GitHub (see below)
2. Go to [vercel.com](https://vercel.com) → New Project → Import repo
3. Configure:
   - **Root Directory:** `frontend`
   - **Framework Preset:** Vite (auto-detected)
   - **Build Command:** `npm run build`
   - **Output Directory:** `dist`
4. Add Environment Variable:
   ```
   VITE_API_URL = https://the-matrix-awakens.YOUR_SUBDOMAIN.workers.dev
   ```
5. Deploy

---

## Step 4 — Final Wiring

After Vercel gives you your URL (e.g. `https://the-matrix-awakens.vercel.app`):

1. Open `backend/src/worker.js`
2. Update the CORS allowlist:
   ```js
   const allowed = [
     'https://your-real-vercel-url.vercel.app',
     'http://localhost:5173',
   ];
   ```
3. Redeploy:
   ```bash
   cd backend && npm run deploy
   ```

---

## Environment Variables Reference

### Backend (wrangler secrets — never committed)

| Secret | Description | Where to get |
|--------|-------------|-------------|
| `GROQ_API_KEY` | Groq inference API key | console.groq.com |
| `RESEND_API_KEY` | Transactional email | resend.com |
| `ADMIN_EMAIL` | Your email for alerts | Your email |
| `ADMIN_SECRET` | Admin dashboard password | Choose a strong one |

### Frontend (Vercel environment variables)

| Variable | Description |
|----------|-------------|
| `VITE_API_URL` | Your Worker base URL (no trailing slash) |

---

## Verify Everything Works

```bash
# 1. Worker health check
curl https://YOUR_WORKER.workers.dev/api/health
# Expected: { "status": "[MATRIX_ONLINE]", "runtime": "Cloudflare Workers" }

# 2. Admin dashboard
# Visit: https://YOUR_WORKER.workers.dev/admin
# Enter your ADMIN_SECRET

# 3. Frontend
# Visit your Vercel URL
# Open Sentinel chat → send a message → check admin dashboard for the log
```

---

## Ongoing Updates

```bash
# Frontend change → auto-deploys when pushed to GitHub main branch
git add . && git commit -m "update" && git push

# Backend change → requires manual redeploy
cd backend && npm run deploy

# Update resume
# Drop new resume.pdf into frontend/public/resume.pdf → git push
```

---

## Free Tier Limits

| Service | Free Allowance | Typical Portfolio Usage |
|---------|---------------|------------------------|
| Cloudflare Workers | 100,000 req/day | ~100 visitors/day = fine |
| Cloudflare D1 | 5M reads/day, 100K writes | Negligible |
| Groq API | 14,400 tokens/min | ~50 chat sessions/day |
| Resend | 3,000 emails/month | <100 contact forms |
| Vercel Hobby | 100GB bandwidth | Static site = fine |
| **Monthly cost** | **$0.00** | |

---

*"The Matrix has you. But the infrastructure costs you nothing."*
