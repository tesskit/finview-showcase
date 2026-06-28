# FinView — Architecture Document

**Version:** 1.0
**Status:** Production

---

## 1. Overview

FinView is a full-stack financial dashboard with a React/Next.js frontend deployed on Vercel and a Python/FastAPI backend deployed on Render. PostgreSQL stores user session data, prompt configurations, and AI usage logs. No personally identifiable information is collected or stored. Six tabs surface live market data and AI-powered analysis in a single interface.

---

## 2. System Architecture

```
Browser (Next.js / Vercel)
    |
    |-- HTTP requests --> FastAPI (Render)
    |                         |
    |                         |-- Claude API (Anthropic)
    |                         |-- Pyth Network (price feeds)
    |                         |-- Google Finance (price feeds)
    |                         |-- yfinance (supplemental)
    |                         |-- PostgreSQL (Render)
    |
    |<-- SSE stream <-- FastAPI (cache refresh events)
```

---

## 3. Frontend

**Stack:** React / Next.js, deployed on Vercel

**Six tabs:**
- Watchlist — live quotes, extended-hours prices
- Market Snapshot — AI market summary
- Market Outlook — AI forward-looking analysis
- Hype Detector — AI-generated custom HTML dashboard
- World Events — AI geopolitical and macro analysis
- Catalyst Calendar — pure data feed, no AI

**SSE client:**
The frontend opens a persistent SSE connection to the backend on load. When the backend refreshes its data cache, it pushes an event to all connected clients. The frontend updates the relevant tab without a page reload or polling interval.

---

## 4. Backend

**Stack:** Python + FastAPI, deployed on Render

### Key Endpoints

| Endpoint | Method | Purpose |
|---|---|---|
| /watchlist | GET | Live quotes + extended-hours prices |
| /snapshot | POST | Market Snapshot AI analysis |
| /outlook | POST | Market Outlook AI analysis |
| /hype | POST | Hype Detector HTML dashboard |
| /events | POST | World Events AI analysis |
| /catalyst | GET | Catalyst Calendar data feed |
| /cache/refresh | POST | Trigger cache refresh, emit SSE event |
| /admin/usage | GET | Per-user AI usage and budget data |

### Prompt Management
All Claude prompts are stored in PostgreSQL, not hardcoded. The backend loads the active prompt for each tab at request time. Prompts can be updated via the admin UI without a redeploy.

### AI Usage Tracking
Every Claude API call logs token counts and estimated cost to PostgreSQL, keyed by user. The superadmin dashboard queries this table to show per-user usage and enforce budget caps.

---

## 5. Real-Time: SSE Architecture

Server-Sent Events (SSE) is a one-way protocol: the server pushes events to the browser over a persistent HTTP connection. FinView uses SSE for live cache refresh notifications.

**Flow:**
```
1. Frontend connects to /events/stream on load (persistent SSE connection)
2. User or scheduler triggers /cache/refresh
3. Backend refreshes data, writes to cache
4. Backend emits SSE event: { type: "cache_updated", tab: "snapshot", timestamp: ... }
5. Frontend receives event, re-fetches the relevant tab data
6. User sees updated data without page reload
```

**Why SSE over WebSockets:**
SSE is simpler and sufficient for one-way server-to-client push. WebSockets add bidirectional complexity that FinView does not need. SSE works over standard HTTP and is supported natively by the browser EventSource API.

---

## 6. Extended-Hours Price Feed

Standard quote APIs do not provide pre-market and after-hours prices at low cost. FinView uses a multi-source waterfall:

| Source | Role | Notes |
|---|---|---|
| Pyth Network | Primary | Real-time decentralized price feeds, strong crypto and equity coverage |
| Google Finance | Fallback | Extended-hours data for tickers not covered by Pyth |
| yfinance | Supplemental | Historical and supplemental data, not real-time |

The waterfall tries Pyth first. If the ticker is not available or the feed is stale, it falls back to Google Finance. yfinance fills gaps for historical context.

---

## 7. AI Pipeline Per Tab

Each AI tab calls Claude with a different prompt contract loaded from PostgreSQL.

| Tab | Output Format | Notes |
|---|---|---|
| Market Snapshot | Prose | 2-3 paragraph market read |
| Market Outlook | Prose with key levels | Explicitly forward-looking, not recap |
| Hype Detector | Raw HTML | Strict output contract — see Section 8 |
| World Events | Prose with sector tags | Maps headlines to market exposure |

---

## 8. Prompt Engineering: Hype Detector

The Hype Detector tab renders a custom HTML dashboard generated entirely by Claude. This created a non-determinism problem that required explicit prompt engineering to solve.

### The Problem

Claude's output format was inconsistent. The same semantic request produced different output formats across runs — valid raw HTML in some, prose descriptions in others, HTML wrapped in markdown code fences in others. The frontend rendered the first correctly and failed silently on the rest.

### The Fix: Prompt Contract

The system prompt was rewritten as an explicit output contract with four components:

1. **Format specification:** Absolute instruction that the entire response must be valid HTML starting with `<div` and ending with `</div>`. No markdown, no prose, no code fences, no explanation.
2. **Structure specification:** Exact HTML structure required — wrapper element, card structure, required CSS classes, required data attributes.
3. **Negative examples:** Each disallowed output format shown explicitly with a "do not do this" label.
4. **Output anchor:** The prompt ends at the start of the expected output, priming Claude to continue in the correct format rather than choose one.

### Result

Output is now deterministic across runs. No retry logic, no format validation, no fallback handling needed.

Full case study: [docs/PROMPT_ENGINEERING.md](./docs/PROMPT_ENGINEERING.md)

---

## 9. Catalyst Calendar: Intentional AI Removal

Originally designed to use Claude for generating upcoming catalyst events: earnings dates, FDA decision windows, economic data releases.

### Why AI Was Removed

AI-generated catalyst data failed in two ways:
- **Hallucinated dates:** Plausible-sounding but incorrect earnings dates, especially for smaller-cap tickers
- **Fabricated events:** FDA windows and economic releases sometimes invented when Claude lacked current data

These failures are silent. A hallucinated earnings date looks identical to a correct one. For financial data, silent failure is worse than no data.

### The Rebuild

Rebuilt as a pure data feed from authoritative sources. Simpler, faster, and trustworthy.

### The Principle

Not every feature benefits from AI. Evaluating output quality honestly and removing AI where it introduces more risk than value is harder than leaving it in — and it is the right call.

---

## 10. Database Schema (Summary)

| Table | Purpose |
|---|---|
| users | Session tokens and roles only — no personally identifiable information stored |
| prompts | Per-tab prompt text, version, active flag |
| ai_usage | Per-request token counts, cost, user, tab, timestamp |
| cache | Cached AI responses per tab, TTL |

---

## 11. Deployment

| Component | Platform | Notes |
|---|---|---|
| Frontend | Vercel | Auto-deploys on push to main |
| Backend | Render Standard | Managed PostgreSQL on same platform |

FinView does not run any ML models on the server. The backend is FastAPI + Claude API calls + data fetching. Standard instance RAM is sufficient — a meaningful contrast to Pivot, which requires Standard specifically to hold pre-computed embeddings in RAM.
