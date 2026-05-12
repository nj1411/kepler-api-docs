# Kepler Insights API — Documentation source

This directory is the source content for the public docs site at **docs.keplerinsights.us**.

**Hosting:** Mintlify. Not signed up yet — provisioned at the coordinated API launch event.

---

## File map

```
Ki_dev/docs/
├── README.md                  ← this file (build / deploy notes)
├── mint.json                  ← Mintlify navigation + branding config
├── openapi.yaml               ← OpenAPI 3.1 spec (single source of truth)
├── introduction.mdx
├── quickstart.mdx
├── authentication.mdx
├── sandbox.mdx
├── tiers.mdx
├── economics.mdx
├── errors.mdx
├── endpoints/
│   ├── score.mdx              POST /v1/score
│   ├── score-get.mdx          GET  /v1/score/{domain}
│   ├── async.mdx              POST /v1/score?wait=false  (narrative)
│   ├── jobs.mdx               GET  /v1/jobs/{job_id}
│   ├── history.mdx            GET  /v1/score/{domain}/history
│   ├── cohort.mdx             GET  /v1/company/{domain}/cohort
│   ├── confidence.mdx         GET  /v1/company/{domain}/confidence
│   ├── distribution.mdx       GET  /v1/distribution
│   ├── movers.mdx             GET  /v1/movers
│   ├── signals.mdx            GET  /v1/signals
│   └── usage.mdx              GET  /v1/usage
└── operations/
    └── STATUS_PAGE.md         ← Instatus signup + monitoring plan (deploy-day runbook)
```

## Local preview

Mintlify ships a CLI for live preview without a paid account:

```bash
npm i -g mintlify
cd Ki_dev/docs
mintlify dev
# → http://localhost:3000
```

This validates `mint.json`, renders MDX, resolves `openapi.yaml` references, and hot-reloads on save. Run it before every push.

## Editing rules

1. **OpenAPI is the source of truth for shape.** Don't write request/response field names into MDX — Mintlify auto-generates that section from `openapi.yaml`. Use MDX for narrative ("when to use this", patterns, examples).
2. **Don't reference internals.** No Lambda names, no DynamoDB table names, no IAM roles. The docs site is public; if it leaks infra, fix the leak.
3. **Don't quote prices except in `tiers.mdx` and `economics.mdx`.** Single source of truth for pricing — when we change tiers, we change one place.
4. **Code examples use `ki_live_...` and `ki_test_...` placeholders.** Never paste a real key, even from a dead test account.
5. **MDX anchors:** when linking between docs, use root-relative paths (`/quickstart`, `/endpoints/score`) — not file names. Mintlify rewrites these.

## Deploy day checklist

When `api.keplerinsights.us` is being flipped live (post-Phase H):

1. **Sign up for Mintlify.** Free tier covers a single site at a custom subdomain.
2. **Create a new project** pointing at this directory. Mintlify can deploy from a Git push, a CLI upload, or a manual web-upload zip. CLI upload (`mintlify deploy`) is the cleanest.
3. **Custom domain.** Set `docs.keplerinsights.us` in Mintlify's domain settings; add the CNAME at the DNS provider.
4. **Smoke check.** Visit `docs.keplerinsights.us/quickstart`. Verify code samples render with syntax highlighting + copy buttons. Click through every nav entry.
5. **Update the dev console.** The console at `api.keplerinsights.us` should link `Docs →` to the live URL — update `Ki_dev/api_site/index.html` if it still points to a stub.
6. **Update `mint.json` topbar.** If the status page launched in the same coordinated event, the `https://status.keplerinsights.us` topbar link is already pointing at the live page.
7. **Crawler hygiene.** Mintlify lets browsers index by default — leave that on once we're public. Pre-launch: gate the project as "Private" in Mintlify settings.

## Spec validation

The OpenAPI spec parses with stock `yaml`:

```bash
python3 -c "import yaml; spec = yaml.safe_load(open('openapi.yaml')); print('paths:', len(spec['paths']), '· schemas:', len(spec['components']['schemas']))"
# paths: 10 · schemas: 17
```

For deeper validation (linting, breaking-change detection vs prior versions), use **Spectral** or **Redocly** locally:

```bash
npx @stoplight/spectral-cli lint openapi.yaml
```

## Versioning

The OpenAPI `info.version` field is the only source of truth for spec version. Bump it on any backward-incompatible change. SDK generators in Phase F1 / F2 will read this field.

Major version (v2) eventually gets its own `openapi-v2.yaml` and a parallel content tree. Don't plan for that pre-GA — premature.
