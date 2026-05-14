# Status page + synthetic monitoring plan

> **🟡 DEFERRED to v1.5** (2026-05-12). v1.0 ships an inline real-time health widget on the dev console (`console.keplerinsights.us` top-right corner, 30-second poll via Netlify same-origin proxy to `/health` → API Gateway) plus a `noah@keplerinsights.us` support-email footer. Cost saved: ~$180/yr Instatus subscription. Trigger to revisit: ≥10 paying customers OR a customer explicitly asks "where's your status page?"

**This document remains the runbook for when we DO want a dedicated status page.** Originally planned for launch, now reserved for the moment we actually need it.

---

**Provider chosen: Instatus** ($15/mo Personal tier).
**URL on launch:** `https://status.keplerinsights.us`.
**Wired up at:** when v1.5 trigger fires (see top of doc).

Signup, DNS, and synthetic check provisioning are deferred. This document is the runbook for doing it.

---

## 1. Why Instatus

| Provider | Entry tier | Why we picked / passed |
|---|---|---|
| **Instatus** | **$15/mo** | Cheapest. Components, subscribers, incident history, public + private pages. Built-in 5-min synthetic checks on the higher tier ($30/mo) — defer until volume justifies. |
| Statuspage.io | $29/mo | Atlassian's. Most mature, integrates with PagerDuty. Twice the cost for features we don't need yet. |
| BetterStack | $25/mo | Bundles status + uptime monitoring + log mgmt. Right answer when ops scales; premature for a solo-founder API launch. **Path to upgrade documented below.** |

We can swap to BetterStack later by re-pointing `status.keplerinsights.us` DNS and re-creating the components. Subscriber list does not migrate automatically — plan a transition email if you swap once we have real subscribers.

---

## 2. Components

The status page surfaces health per component, not per Lambda. End users care about routes, not internals.

| Component | Backing Lambdas | "Operational" means |
|---|---|---|
| **Scoring API** | kepler-api-score, kepler-api-read, kepler-fetcher-orchestrator | All endpoints respond 2xx to the synthetic check; cold pipeline P95 < 90s. |
| **Developer console** | kepler-api-key-mgmt, api_site (Netlify) | Console loads + magic-link round-trip works. |
| **Documentation** | Mintlify hosting (docs.keplerinsights.us) | Docs site returns 200 on the homepage. |
| **Scoring engine** | kepler-scoring-engine, fetchers | Engine processes a real domain successfully end-to-end every 6 hours (validation cron). |

Group components on the status page as "API platform" (Scoring API, Scoring engine) and "Developer experience" (Developer console, Documentation).

---

## 3. Synthetic monitoring

### Primary check — `POST /v1/score` (cached path, every 5 min)

Pings `POST /v1/score` with a known-already-scored domain so the response is always cached. Confirms:
- Authorizer is online (responds to `X-API-Key`)
- API Gateway routing is healthy
- DynamoDB read path is fast
- Lambda cold-start latency hasn't regressed

```bash
curl -X POST https://api.keplerinsights.us/v1/score \
  -H "X-API-Key: ki_live_MONITORING_KEY" \
  -H "Content-Type: application/json" \
  -d '{"domain": "stripe.com"}' \
  -w "%{http_code} %{time_total}\n"
```

Pass criteria: HTTP 200, total time < 2.5s, response body contains `"ki_rating"`.

### Secondary check — `GET /v1/usage` (every 5 min)

Faster + cheaper than the score check. Provides a backstop signal: if `/v1/usage` is up but `/v1/score` is down, the problem is in the score Lambda specifically, not the gateway / authorizer.

### Cold-pipeline validation check — every 6 hours

Run `POST /v1/score` against a domain whose freshness window has expired (force_fresh on Enterprise, or pick a domain we haven't refreshed today). Confirms the cold pipeline + fetchers are healthy. Lambda timeout 70s; alert if p95 over 7 days exceeds 90s.

This is the only check that costs real money — ~$3/day at one cold call per 6 hours. Acceptable canary spend.

### Where to run them

**Instatus Pro tier ($30/mo)** has built-in monitors. Cheaper alternative: a tiny EventBridge cron → Lambda in this AWS account that does the checks and writes to Instatus via its incidents API. Defer the decision until we know the operational rhythm.

---

## 4. Setup checklist (deploy day)

### One-time

- [ ] Sign up for Instatus on the Personal tier ($15/mo)
- [ ] Provision two API keys at `api.keplerinsights.us` labeled `monitoring-prod` and `monitoring-cold-canary` (live mode, lowest-tier sufficient — Starter)
- [ ] Add `status.keplerinsights.us` as a custom domain on Instatus; copy the CNAME target
- [ ] Add the CNAME record at the DNS provider (Netlify DNS / wherever `keplerinsights.us` is hosted)
- [ ] Add an `X-Robots-Tag: noindex` header at the DNS layer if the status page should not be indexed pre-launch (revisit at GA)
- [ ] Create components per § 2
- [ ] Create the three monitoring checks per § 3
- [ ] Subscribe `noah@keplerinsights.us` to the page so you get email + SMS on incidents
- [ ] Link the status page from `mint.json.topbarLinks` (already configured pre-launch)

### Per-release

- [ ] Update `kepler-api-cost-log` weekly review — if cold-pipeline P95 has crept above 70s, the canary will start tripping; pre-empt by raising Lambda timeout or tuning the cold pipeline.
- [ ] Post a planned-maintenance incident on Instatus 24h before any breaking change goes live.

---

## 5. Incident response

Severity 1 (full outage, multiple components red):
1. Acknowledge in Instatus within 5 min ("we're investigating")
2. Email Kepler customers on the API in addition to the status page subscribers
3. First update within 15 min — what's broken, what we're doing
4. Post-incident: write a short post-mortem in `docs_site/operations/incidents/YYYY-MM-DD-slug.md`

Severity 2 (degraded, one component yellow):
1. Acknowledge within 15 min
2. Update every 30 min until resolved

Severity 3 (single canary trip, intermittent):
1. Don't post unless the canary is sustained > 15 min — Instatus posting noise erodes trust

---

## 6. When to upgrade

Promote off the Personal tier when **any** of:
- We have > 50 status-page subscribers (Personal tier caps at 100)
- We need on-call routing / paging (BetterStack or PagerDuty integration via Statuspage.io)
- We need uptime monitoring + log management bundled

For now: $15/month, manual response, solo-founder ops.
