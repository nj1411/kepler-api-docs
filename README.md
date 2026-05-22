# Kepler Insights API — Documentation source

This directory is the source content for the public docs site at **docs.keplerinsights.us**.

**Hosting:** Mintlify. Live since 2026-05-12 (API launch event). Content lives in the `nj1411/kepler-api-docs` GitHub repo — push from here to that repo to update the live site.

---

## File map

```
docs_site/
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
│   ├── async.mdx              Async cold-scoring narrative + polling guide
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
cd docs_site
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

## Sync mechanics + mirror divergence (added 2026-05-19)

The local `docs_site/` directory and the `nj1411/kepler-api-docs` GitHub repo are two source trees that must stay in sync. They drift any time someone hand-edits one without back-porting to the other.

**To push local edits to live:**

```bash
gh repo clone nj1411/kepler-api-docs /tmp/kepler-api-docs
rsync -av --exclude .git --exclude .github docs_site/ /tmp/kepler-api-docs/
# DO NOT add --delete — see "Known mirror-ahead files" below.
cd /tmp/kepler-api-docs
git status                                       # preview intended diff
git checkout endpoints/history.mdx mint.json     # revert mirror-ahead files
git diff                                          # final intended diff
git add -A && git commit -m "..." && git push origin main
```

Mintlify auto-rebuilds within ~60s of the push.

**Known mirror-ahead files (as of 2026-05-19).** These exist in the GitHub mirror at a more correct state than the local source and were never back-ported. `git checkout` them in the mirror clone before commit, OR do a one-time back-port into `docs_site/` to heal the divergence:

- `endpoints/history.mdx` — mirror has clean `<1KB`, local has HTML entity `&lt;1KB` (post-launch MDX-parsing fix on the mirror, commit `c6ccd81`)
- `mint.json` — mirror has a `Resources / changelog` nav group at the end of `navigation`; local does not (commit `c9746a9`)
- `changelog.mdx` — exists only on the mirror; no local source (commit `c9746a9`)

If you `rsync --delete` from local to mirror, you will erase the Changelog page from the live site. Don't.

## Deploy day checklist (historical — kept for reference)

This section captures the one-shot setup that happened at the 2026-05-12 launch event. Day-to-day updates are now just: edit MDX → push to `nj1411/kepler-api-docs` → Mintlify rebuilds (see "Sync mechanics" above for the modern recipe).

1. **Sign up for Mintlify.** Free tier covers a single site at a custom subdomain.
2. **Create a new project** pointing at this directory. Mintlify can deploy from a Git push, a CLI upload, or a manual web-upload zip. CLI upload (`mintlify deploy`) is the cleanest.
3. **Custom domain.** Set `docs.keplerinsights.us` in Mintlify's domain settings; add the CNAME at the DNS provider.
4. **Smoke check.** Visit `docs.keplerinsights.us/quickstart`. Verify code samples render with syntax highlighting + copy buttons. Click through every nav entry.
5. **Update the dev console.** The console at `console.keplerinsights.us` should link `Docs →` to the live docs URL.

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
