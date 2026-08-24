# Sabzgin — uptime monitoring

External availability monitoring for [sabzgin.ir](https://sabzgin.ir), powered by
[Upptime](https://upptime.js.org): GitHub Actions runs the checks, GitHub Issues
records the incidents, GitHub Pages publishes the status page. No server.

This repository is deliberately public and deliberately contains **no secrets**.
Every URL it checks is already public. Notification credentials live in GitHub
Actions Secrets and are referenced by name only.

The design and its reasoning live in the private application repository, at
`docs/issues/external-production-monitoring.md`.

---

## What this layer does, and what it does not

Upptime answers **"can the outside world reach the shop?"** It runs on GitHub's
infrastructure, so it keeps answering when the VPS is a brick.

It cannot see a backup. That is the Healthchecks dead-man layer, which is
already live and lives in the private repo — the API pings it from
`BackupMonitoringListener`. The two layers are independent and share only a
Telegram chat.

---

## Why a separate PUBLIC repository

**Do not make the Village repository public. Do not change its visibility.**

| | Public | Private |
|---|---|---|
| GitHub Actions minutes | unlimited, free | 2,000/month free |
| Upptime's usage | ~3,000 min/month (per Upptime's docs) | **exceeds the free tier** |

Publishing a status page from a private repo also needs an authenticated API
proxy — extra moving parts on the thing whose job is to be simpler than what it
monitors.

Nothing secret goes in it. Every URL in `.upptimerc.yml` is already public: the
three hostnames are in DNS and answer unauthenticated HTTPS today.

---

## 🔴 Order matters

`/health/ready/db` and `/health/ready/redis` exist only from the deploy that
added them. **Deploy the API first.** Creating this repository beforehand puts
two checks permanently red on a perfectly healthy server — and a red check
nobody can fix is the fastest way to teach people to ignore the board.

Verify first: `curl -sS -o /dev/null -w '%{http_code}\n' https://api.sabzgin.ir/health/ready/db`
should answer `200`, not `404`.

## Creating it (manual, one time)

1. **New repo from the template** `upptime/upptime` → name `sabzgin-uptime`,
   visibility **public**, and tick **"Include all branches"** (required — the
   template ships `gh-pages` and the workflow set).
2. **Copy `.upptimerc.yml`** from this directory to the new repo's root. Nothing
   else from this repository is copied — see the checklist below.
3. **Add four Actions secrets** (Settings → Secrets and variables → Actions):

   | Secret | Value |
   |---|---|
   | `GH_PAT` | fine-grained PAT scoped to **this repo only**, read-write on Actions, Contents, Issues, Workflows |
   | `NOTIFICATION_TELEGRAM` | `true` |
   | `NOTIFICATION_TELEGRAM_BOT_KEY` | the BotFather token (same bot as Healthchecks) |
   | `NOTIFICATION_TELEGRAM_CHAT_ID` | the chat id |

4. **Enable Pages**: Settings → Pages → *Deploy from a branch* → `gh-pages` / `(root)`.
5. **Pin the actions.** The template references `upptime/uptime-monitor@master`.
   A `@master` reference is remote code execution on every scheduled run, holding
   a PAT. Pin to a tag or a commit SHA and update deliberately.
6. Set default workflow permissions to **read-only** and let each workflow
   escalate.

---

## 🔴 Never copy any of this into the public repository

`.env` · `.env.example` · anything under `docker/` (compose, Caddyfile, firewall
scripts and units) · anything under `docs/deployment/**` · any application source
· database, Redis, Mongo or Meilisearch credentials · the backup encryption key ·
S3/Arvan or Google Drive credentials · JWT or OTP secrets · Zibal, Tapin or SMS.ir
credentials · SMTP credentials · **Healthchecks ping URLs** · the Telegram bot
token or chat id · **the origin server's IP address or SSH username** · any log
excerpt.

The origin IP deserves its own line: the origin firewall admits only Cloudflare
ranges, so publishing it is not exploitable *today* — but it is the one address
that bypasses Cloudflare the moment that firewall has a gap, and it is a DDoS
target. Every check in `.upptimerc.yml` uses a hostname; the IP is never needed.

---

## Reading the checks

Names are prefixed on purpose, because the tiers mean different things:

- **`[page]` red** — customers cannot reach the site. This is the outage.
- **`[api]` red, `[page]` green** — the API process is down but Nitro is serving;
  expect the storefront to follow.
- **`[dep] Postgres` red** — the database is unreachable. Everything stops.
- **`[dep] Redis` red, everything else green** — **worse than it looks.**
  Browsing degrades gracefully (cache falls through to Postgres, throttler fails
  open) but OTP login and refresh-token rotation call Redis directly with no
  fallback. Customers can browse and **cannot buy**, while the status page reads
  green. Treat it as page-worthy.

The dependency checks are split into two URLs precisely so the alert can say
*which*. The 503 **body** still names nothing — that is deliberate, and it is why
the distinction had to move into the path.

**Detection latency is 10–25 minutes, not 5.** GitHub's scheduled workflows are
best-effort and routinely run late. Accepted trade-off; documented so nobody
believes the cron expression.

---

## The storefront canary

`[page] Storefront` asserts a string, not just a status code, because it is
**not known** whether Nuxt returns a non-2xx when the API is unreachable — it may
render a degraded page and still answer 200, which a status-only check would
report as healthy.

The asserted string is the site name from the `siteBranding` GraphQL query. The
build-time fallback in `nuxt.config.ts` is the shop's *old* name, so a failed
query changes the title and the string disappears. Verified against the live page:
the API-derived title is present and the fallback is absent.

**To settle it properly**, stop the API for ~30 seconds and record what `/`
returns. That is the one acceptance test still outstanding.
