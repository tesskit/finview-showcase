# FinView — Services & Dependencies

A quick reference for every external service FinView uses, what it costs, and where to manage it.

---

## Paying / Could Be Paying

| Service | URL | Cost | Notes |
|---|---|---|---|
| Render | dashboard.render.com | $25/month (Standard) | Backend + PostgreSQL. |
| Anthropic Claude API | console.anthropic.com | Pay per token | Every AI tab call hits the API. Per-user budget caps enforced in app. |
| Claude.ai Pro | claude.ai | Monthly subscription | Development and planning use. |

---

## Free Tiers — Monitor Limits

| Service | URL | Limit | Notes |
|---|---|---|---|
| Vercel | vercel.com | Free hobby tier | Frontend deployment. Auto-deploys on push. |
| yfinance | pypi.org/project/yfinance | Free (scraping) | Supplemental price data. Unofficial — subject to breakage. |

---

## Free — No Limits

| Service | URL | Notes |
|---|---|---|
| Pyth Network | pyth.network | Real-time decentralized price feeds. Free API, no key required for public feeds. |
| Google Finance | — | Extended-hours fallback. Fetched via HTTP, no official API. Subject to rate limits. |
| GitHub | github.com/tesskit | Source control. Private repo: tesskit/finview. Public showcase: tesskit/finview-showcase. |

---

## Open Source — No Account Needed

| Package | URL | Notes |
|---|---|---|
| FastAPI | fastapi.tiangolo.com | Python web framework. Backend routing layer. |
| Next.js | nextjs.org | React framework. Frontend. |
| psycopg2 | psycopg.org | PostgreSQL adapter for Python. |

---

## Notes

- The only service that costs real money every month is **Render**.
- **Claude API** cost scales with usage — per-user budget caps in the superadmin dashboard keep this controlled.
- Google Finance and Pyth are unofficial/free — the price feed waterfall is designed so either can fail without breaking the dashboard.
