# ReachInbox Scheduler — Full-Stack Email Job Scheduler

A production-shaped email scheduling service + dashboard: schedule emails via
API, send them through Ethereal SMTP at exact times using BullMQ delayed jobs
(no cron), enforce per-sender rate limits safely across workers, notify
Slack live when a limit is hit, and search all emails via Elasticsearch.

## Architecture overview

```
frontend (Next.js + Tailwind)  ──HTTP──>  backend (Express + TypeScript)
                                              │
                                              ├── Postgres  (source of truth: users, senders,
                                              │              campaigns, emails, rate_limit_counters)
                                              ├── Redis     (BullMQ delayed-job queue +
                                              │              atomic per-sender hourly counters)
                                              ├── Ethereal  (fake SMTP send)
                                              ├── Elasticsearch (searchable index of every email)
                                              └── Slack API (live webhook/chat.postMessage on rate-limit hit)

worker process (separate from the API process) pulls jobs off the BullMQ
queue and does the actual send + rate-limit check + Slack notify.
```

- **API process** (`npm run dev` in `backend/`): accepts schedule requests,
  writes rows to Postgres, and enqueues one BullMQ delayed job per recipient.
- **Worker process** (`npm run worker` in `backend/`): a separate long-running
  process that consumes the queue. Running API and worker separately means
  you can scale workers independently and a server restart never loses queued
  work (jobs live in Redis, not in either process's memory).

## How scheduling works (no cron)

Each recipient becomes one row in `emails` **and** one BullMQ job, added with
`delay = sendAt - now` and `jobId = emailId`. BullMQ persists delayed jobs in
Redis and fires them itself when the delay elapses — there is no cron job or
polling loop anywhere. Using the email's own UUID as the BullMQ `jobId` makes
scheduling idempotent: if the schedule logic ever ran twice for the same
email, BullMQ would reject the duplicate job, so **no email can be
double-queued**.

On restart: Redis still holds every not-yet-due delayed job, so pending
emails fire at their original time. Sent/failed emails are never re-queued
because their Postgres status is only ever set to `scheduled` once, at
creation time.

## Rate limiting design

- **Per-sender, per-hour** counters, keyed as `ratelimit:<senderId>:<hourBucket>`
  in Redis, incremented with `INCR` (atomic — safe across every worker
  process/instance) and mirrored into `rate_limit_counters` in Postgres for
  auditing. We deliberately do **not** rely on in-memory counters.
- Limits are checked **at send time** (inside the worker), not at schedule
  time — this is what lets 1000+ emails scheduled for the same instant be
  handled correctly: the worker reserves a slot, and if the hourly cap is
  already spent, it **reschedules** that email into the next hour window
  (via a fresh delayed job, same `emailId`) instead of dropping or failing it.
- `MAX_EMAILS_PER_HOUR` (global fallback) and each sender's
  `max_emails_per_hour` (per-sender/tenant) are both configurable via
  env/DB — never hardcoded.
- **Minimum delay between sends**: chosen as **2000ms (2 seconds)** between
  individual sends, configurable via `MIN_DELAY_MS_BETWEEN_SENDS`. Enforced
  inside the worker itself (a `setTimeout` before each send) so it holds even
  when running with multiple concurrent workers.
- **Worker concurrency** is configurable via `WORKER_CONCURRENCY` (default 5).

## Slack notifications

"Connect Slack" triggers a real Slack OAuth v2 `authorize` redirect; the
callback exchanges the code for a token/incoming-webhook and stores it per
user in `slack_connections`. The worker looks this row up fresh on every
rate-limit hit (no caching), so:
- if the user hasn't connected Slack yet, notification is a silent no-op —
  never a crash;
- if they connect Slack later, the very next rate-limit hit notifies them,
  no redeploy needed.

## Repo layout

```
backend/    Express + TypeScript API, BullMQ queue + worker, Postgres, Elasticsearch, Slack/Google OAuth
frontend/   Next.js + Tailwind dashboard
docker-compose.yml   Redis + Postgres + Elasticsearch for local dev
```

## Running it locally

### 1. Start infra
```bash
docker compose up -d
```

### 2. Backend
```bash
cd backend
cp .env.example .env      # fill in GOOGLE_CLIENT_ID/SECRET and SLACK_CLIENT_ID/SECRET
npm install
npm run migrate           # creates tables in Postgres
npm run dev                # API on http://localhost:4000
npm run worker             # in a second terminal — the BullMQ worker
```

Live BullMQ dashboard: `http://localhost:4000/admin/queues`

### 3. Frontend
```bash
cd frontend
cp .env.local.example .env.local
npm install
npm run dev                # http://localhost:3000
```

### 4. Ethereal Email
No setup needed — the backend auto-creates a fresh Ethereal test SMTP account
for each user's first login (via `nodemailer.createTestAccount()`) and stores
its credentials in the `senders` table. Sent-message preview URLs are logged
by the worker.

### 5. Google OAuth
Create an OAuth 2.0 Client ID in Google Cloud Console (Web application),
add `http://localhost:4000/auth/google/callback` as an authorized redirect
URI, and put the client ID/secret in `backend/.env`.

### 6. Slack OAuth
Create a Slack App at api.slack.com/apps with the `incoming-webhook` and
`chat:write` scopes, set the redirect URL to
`http://localhost:4000/slack/oauth/callback`, and put the client ID/secret
in `backend/.env`.

### 7. Elasticsearch
Provided via docker-compose (single-node, security disabled, for local dev
only). The backend creates the `emails` index automatically on boot.

## Environment variables

See `backend/.env.example` and `frontend/.env.local.example` — every limit,
delay, and credential is configurable, nothing is hardcoded.

## Testing without real OAuth

`POST /dev/login { "userId": "<uuid>" }` sets the session directly, bypassing
real Google OAuth — useful for integration tests or sandboxes with no
internet access to `accounts.google.com`. It's compiled in but **inert
unless `ALLOW_TEST_AUTH=true`** is explicitly set in `backend/.env` (default
is unset/false). Leave it unset in any real deployment.

## What's been verified end-to-end vs. not

I ran this locally (Postgres + Redis installed directly, via the dev-login
bypass above) and confirmed:
- Schedule → per-recipient rows created, staggered by `delayMs`, one
  BullMQ delayed job per email.
- Rate limiting: with `hourlyLimit=2` and 4 recipients, exactly 2 were
  gated and correctly **rescheduled** to the next hour window (not dropped).
- Idempotency: calling `enqueueEmail` twice with the same `emailId` does
  **not** overwrite the existing Redis job — confirmed by inspecting the raw
  Redis hash before/after.
- Restart persistence: killed the worker process mid-delay, confirmed the
  delayed jobs were still sitting in Redis (untouched by the process death),
  restarted the worker, and all 4 jobs fired at their original scheduled
  times with no duplication.
- Graceful degradation: Elasticsearch and Slack calls fail silently
  (logged, non-fatal) when those services aren't reachable — the API never
  500s because of them.
- CSV lead parsing and count detection, empty-state responses.

Not verified in this sandbox (no outbound network to these hosts):
- Actual email delivery through Ethereal SMTP (connections to
  `smtp.ethereal.email` time out here — the send code path runs, but I
  couldn't confirm a real message lands).
- Real Google OAuth and Slack OAuth round-trips (needs real client
  credentials + reachable `accounts.google.com` / `slack.com`).
- Elasticsearch itself running (indexing/search code paths are exercised
  against a *down* ES to confirm graceful failure, but not against a live
  index).

Recommend re-running this checklist once with real credentials and
`docker compose up` before you submit, to confirm those three.

## Known simplifications (given assignment scope)

- Single Ethereal sender per user is auto-provisioned on first login; the
  schema (`senders` table) already supports multiple senders per user if
  multi-sender UI were added.
- The 1000+ emails / "would exceed rate limit" behavior is implemented and
  will correctly reschedule under real load, but the demo does not attempt
  to actually send thousands of emails through Ethereal.
