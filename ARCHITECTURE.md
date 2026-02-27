<div align="center">

```
╔══════════════════════════════════════════════════════╗
║   A R C H I T E C T U R E   D E E P   D I V E       ║
║   "The wise see action in inaction." — Gita 4.18     ║
╚══════════════════════════════════════════════════════╝
```

</div>

# Architecture Deep Dive

---

## Stage Machine — Entry Experience

The portfolio entrance is a **3-stage finite state machine**. Each stage is one-directional and irreversible per session.

```
┌──────────────────┐     onComplete()    ┌──────────────────┐     onComplete()    ┌──────────────────┐
│  STAGE 1         │ ──────────────────► │  STAGE 2         │ ──────────────────► │  STAGE 3         │
│  TerminalScreen  │                     │  MatrixTransition│                     │  Portfolio SPA   │
│                  │                     │                  │                     │                  │
│  Boot sequence   │                     │  Katakana rain   │                     │  Full site       │
│  Typewriter text │                     │  3D grid canvas  │                     │  All sections    │
│  ~3000ms         │                     │  ~2500ms         │                     │  Live forever    │
└──────────────────┘                     └──────────────────┘                     └──────────────────┘
```

---

## Theme Engine — Tri-Mode State Machine

```
                      ┌────────────────────────┐
                      │     MATRIX MODE        │
                      │  (transition ~800ms)   │
         ┌────────────│  #000 bg + #00ff41     │────────────┐
         │            │  Fira Code font        │            │
         │            │  Katakana rain         │            │
         │            └────────────────────────┘            │
         │                  ▲           ▲                    │
         │     yin-yang     │           │     yin-yang       │
         │      click       │           │      click         │
         ▼                  │           │                    ▼
┌─────────────────┐         │           │         ┌──────────────────┐
│   LIGHT MODE    │─────────┘           └─────────│   DARK MODE      │
│   (olha)        │                               │                  │
│  White bg       │◄──────────────────────────────│  #0a0a0a bg      │
│  Inter font     │      after 800ms flash        │  White text      │
│  Editorial UI   │                               │  Inter font      │
└─────────────────┘                               └──────────────────┘

BREACH TRIGGER (PM logo / nav click):
  any mode → matrix[~5000ms] → original mode   ← baseModeRef preserves state
```

**Key implementation:** `baseModeRef` tracks the last non-matrix mode. When `triggerBreach()` fires, it saves the current base state and restores it after the animation — preventing the common bug where dark mode would reset to light after clicking the nav.

---

## Request Lifecycle — POST /api/chat

```
CLIENT (Vercel)                          WORKER (Cloudflare Edge)
────────────────                         ──────────────────────────────────────────

SentinelChat.tsx                         [1] secureHeaders() middleware
  │                                      [2] CORS origin validation
  │  POST /api/chat                      [3] dharmaThrottle()
  │  { message, history,    ─────────►       └─ COUNT requests in D1 per IP/window
  │    sessionDuration,                       └─ Return 429 if > 100 / 15min
  │    clickCount,                        [4] Validate message (type, empty, trim)
  │    clickTargets }                     [5] extractEntityProfile()
  │                                           └─ cf-connecting-ip (true IP)
  │                                           └─ c.req.raw.cf.city / .country (FREE)
  │                                           └─ parseBrowser(userAgent)
  │                                           └─ parseDevice(userAgent)
  │                                       [6] Build transmission chain:
  │                                           [system: AGENT_DHARMA]
  │                                           [...history]
  │                                           [user: message]
  │                                       [7] invokeNeuralCore() → Groq API
  │                                           └─ llama-3.1-8b-instant
  │                                           └─ max_tokens: 1000, temp: 0.7
  │                                       [8] c.json({ response, timestamp })
  │  ◄─────────────────────────────────
  │  { response: string,                  [9] waitUntil: saveKarmaLog() → D1
  │    timestamp: string }                    (async, non-blocking)
  │                                      [10] waitUntil: isHighIntentSoul()?
                                              └─ sendOperatorAlert() → Resend
```

---

## Database Schema — Cloudflare D1

```
┌─────────────────────────────────────────────────────────────────────┐
│  DATABASE: matrix-karma-db (Cloudflare D1 / SQLite)                 │
└─────────────────────────────────────────────────────────────────────┘

  TABLE: visits
  ┌────────────────────┬──────────┬──────────────────────────────────┐
  │ id                 │ INTEGER  │ PK autoincrement                 │
  │ ip_address         │ TEXT     │ Visitor IP                       │
  │ city               │ TEXT     │ From Cloudflare cf header        │
  │ country            │ TEXT     │ From Cloudflare cf header        │
  │ browser            │ TEXT     │ Parsed from user-agent           │
  │ device             │ TEXT     │ Desktop / Mobile / Tablet        │
  │ referer            │ TEXT     │ Where they came from             │
  │ session_duration_ms│ INTEGER  │ Time on site (ms)                │
  │ click_count        │ INTEGER  │ Total clicks in session          │
  │ click_targets      │ TEXT     │ JSON: what they clicked          │
  │ timestamp          │ INTEGER  │ Unix ms                          │
  └────────────────────┴──────────┴──────────────────────────────────┘

  TABLE: karma_logs
  ┌──────────────────────┬──────────┬────────────────────────────────┐
  │ id                   │ INTEGER  │ PK autoincrement               │
  │ ip_address           │ TEXT     │ Visitor IP                     │
  │ city / country       │ TEXT     │ Geolocation                    │
  │ browser / device     │ TEXT     │ Client info                    │
  │ prompt_history       │ TEXT     │ JSON: full conversation        │
  │ neural_response      │ TEXT     │ Last Sentinel reply            │
  │ message_count        │ INTEGER  │ User messages in session       │
  │ conversation_summary │ TEXT     │ Auto: first→last message       │
  │ session_duration_ms  │ INTEGER  │ Time on site                   │
  │ is_high_intent       │ INTEGER  │ 0 or 1 — lead signal           │
  │ timestamp            │ INTEGER  │ Unix ms                        │
  └──────────────────────┴──────────┴────────────────────────────────┘

  TABLE: lead_souls       → Contact form submissions (name, email, message)
  TABLE: error_logs       → API/DB errors with type, endpoint, status code
  TABLE: rate_limits      → Per-IP request timestamps for throttle window
```

---

## The Sentinel — AI Persona Design

```
╔═══════════════════════════════════════════════════════════╗
║              SENTINEL BEHAVIORAL MATRIX                   ║
╠═══════════════════════════════════════════════════════════╣
║  INPUT TYPE          │  RESPONSE PROTOCOL                 ║
╠══════════════════════╪════════════════════════════════════╣
║  Technical question  │  Resume-sourced precise answer     ║
║  "hire/resume/CV"    │  Answer + lead alert fires         ║
║  Schedule meeting    │  Request email + log intent        ║
║  Personal/casual     │  [SECURITY_INTERCEPT]              ║
║  "how were you built"│  Edge architecture explanation     ║
║  Contact request     │  Authorized channels only          ║
╚══════════════════════╧════════════════════════════════════╝
```

**High-Intent Detection Logic:**
```
isHighIntentSoul(history, message) returns TRUE when:
  ├─ conversation depth > 4 messages  (sustained engagement)
  └─ message contains: resume | cv | contact | hire | connect | reach out

On TRUE:
  └─ sendOperatorAlert() fires via Resend (non-blocking via waitUntil)
     └─ Full conversation transcript
     └─ IP + geolocation
     └─ Browser + device
```

---

## Frontend Tracker — Visitor Intelligence

```
useTracker() — fires on Portfolio mount
  │
  ├─ [IMMEDIATE] POST /api/track
  │   └─ sessionDuration: 0
  │   └─ clickCount: 0          ← Guarantees visit is recorded
  │   └─ clickTargets: []
  │
  ├─ [ONGOING] document.addEventListener('click')
  │   └─ Records: link hrefs, button labels, section IDs
  │   └─ Capped at 50 targets per session
  │
  └─ [ON UNLOAD] window.beforeunload + pagehide
      └─ POST /api/track (keepalive: true)
      └─ sessionDuration: Date.now() - sessionStart
      └─ clickCount: final
      └─ clickTargets: final array
```

---

## Performance Profile

| Metric | Target | How |
|--------|--------|-----|
| AI Response Time | < 200ms | Groq Llama 3.1 8B Instant |
| Worker Cold Start | < 10ms | Cloudflare V8 isolates (no containers) |
| Frontend FCP | < 1.5s | Vercel edge CDN + Vite tree-shaking |
| DB Query | < 5ms | D1 co-located with Worker at edge |
| Monthly Cost | $0 | CF Workers free (100k req/day) + Vercel Hobby |

---

*"The Matrix cannot tell you who you are. The architecture can."*
