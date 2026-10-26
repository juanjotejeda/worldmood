# World Mood Widget: Build Spec for Claude Code

Read this whole file before writing code. Build in the task order below. Stop at each checkpoint and report results before moving on.

## What you are building

A Node.js service on Railway that:

1. Streams public Bluesky posts from Jetstream.
2. Keeps only first-person feeling statements in English, German and Spanish.
3. Classifies each kept post into joy, fear, anger, sadness or neutral with a local model.
4. Stores hourly counts in Postgres (never post text, never account identifiers).
5. Once a day computes emotion shares, z-scores against a 28-day baseline, a mood label, and one generated sentence via Claude Haiku.
6. Serves the result as JSON to an existing personal dashboard widget.

Two slices: `global` (en + de + es) and `de` (German-language posts only, also counted in global).

## Hard constraints

- Never store post text, handles or DIDs in the database or logs.
- Keep it lean: one service, no queue systems, no Redis, no frameworks beyond what is listed.
- No paid data sources.
- All times in Europe/Berlin for scheduling; store timestamps in UTC.
- Every config value comes from environment variables.

## Stack

- Node.js 22, ESM, TypeScript optional (ask if unsure; default plain JS with JSDoc).
- `ws` for the websocket.
- `@huggingface/transformers` for the classifier, run in a `worker_threads` worker.
- `pg` for Postgres.
- `node-cron` for scheduled jobs.
- `fastify` for the API.
- `@anthropic-ai/sdk` for the daily sentence.
- Deploy target: Railway, one service, Railway Postgres plugin.

## Environment variables

```
DATABASE_URL=
ANTHROPIC_API_KEY=
ANTHROPIC_MODEL=claude-haiku-4-5        # confirm the current Haiku model id before use
JETSTREAM_URLS=wss://jetstream2.us-east.bsky.network/subscribe,wss://jetstream1.us-east.bsky.network/subscribe
HOURLY_CAP_PER_LANG=600
CONFIDENCE_MIN=0.5
BOT_DAILY_MAX=50
BASELINE_DAYS=28
CORS_ORIGIN=                            # dashboard frontend origin
PORT=3000
```

## Project structure

```
/src
  index.js            # boot: db migrate, start stream, worker, cron, api
  config.js
  db/
    migrate.js
    schema.sql
  stream/
    jetstream.js      # connect, cursor, failover
    filter.js         # language + feeling patterns + drop rules
    sampler.js        # per-language hourly cap
    botGuard.js       # in-memory daily per-account counter (hashed, never persisted)
  classify/
    worker.js         # model load + inference in worker thread
    client.js         # promise wrapper around the worker
  aggregate/
    hourly.js         # flush in-memory counts to hourly_counts every hour
    terms.js          # in-memory term frequencies per slice
    daily.js          # 05:00 job: shares, z-scores, label, Haiku call
  api/
    server.js
/test
  filter.test.js
  daily.test.js
  fixtures/labeled_posts.csv   # created in Task 2
```

## Database schema

```sql
CREATE TABLE IF NOT EXISTS ingest_state (
  key TEXT PRIMARY KEY,
  cursor BIGINT NOT NULL,
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE IF NOT EXISTS hourly_counts (
  hour TIMESTAMPTZ NOT NULL,
  slice TEXT NOT NULL CHECK (slice IN ('global','de')),
  emotion TEXT NOT NULL CHECK (emotion IN ('joy','fear','anger','sadness')),
  count INT NOT NULL,
  total_kept INT NOT NULL,
  total_seen INT NOT NULL,
  PRIMARY KEY (hour, slice, emotion)
);

CREATE TABLE IF NOT EXISTS daily_mood (
  date DATE NOT NULL,
  slice TEXT NOT NULL CHECK (slice IN ('global','de')),
  shares JSONB NOT NULL,
  zscores JSONB,
  label TEXT NOT NULL,
  sentence TEXT,
  driver TEXT,
  top_terms JSONB,
  post_count INT NOT NULL,
  calibrating BOOLEAN NOT NULL,
  PRIMARY KEY (date, slice)
);
```

Retention: a daily job deletes `hourly_counts` rows older than 90 days. `daily_mood` is kept.

## Behavior spec

### Ingestion

- Connect to the first URL in `JETSTREAM_URLS` with `?wantedCollections=app.bsky.feed.post`.
- Append `&cursor=<time_us>` from `ingest_state` when present. Jetstream cursors are microsecond timestamps.
- Process only events where `kind === "commit"` and `commit.operation === "create"`.
- Post text: `commit.record.text`. Languages: `commit.record.langs` (array, may be missing; missing means drop).
- Save the latest `time_us` to `ingest_state` every 30 seconds.
- Reconnect with exponential backoff (1s, 2s, 4s, max 60s). After 3 failed attempts on one URL, switch to the next.
- Track `last_event_at` for the health endpoint.

### Filter

Keep a post if its first matching language is en, de or es and the text matches a keep pattern for that language and no drop pattern.

| Language | Keep (case-insensitive, word boundaries) | Drop |
| --- | --- | --- |
| en | `i feel`, `i'm feeling`, `im feeling`, `i am feeling`, `feeling so` | `feel free`, `i feel like` |
| de | `ich fühle mich`, `fühle mich`, `mir geht es`, `mir geht's`, `mir gehts` | `fühlt sich an wie` |
| es | `me siento`, `me estoy sintiendo` | `siento que` |

Also drop:
- Posts under 4 words.
- Posts with more than 2 links (count facets of type link, or URLs in text).
- Accounts whose matched-post count today exceeds `BOT_DAILY_MAX`. Count in memory with a salted hash of the DID; reset at midnight Berlin; never persist.

Count every en/de/es post seen as `total_seen` before the feeling filter.

### Sampling

Per language, cap classified posts at `HOURLY_CAP_PER_LANG` per clock hour using reservoir sampling or a probabilistic gate. Count everything that passed the filter as `total_kept` regardless of sampling.

### Classification

- Load the model once in the worker. Batch inputs (size 16 to 32).
- Output label + confidence. Map model labels to joy, fear, anger, sadness, neutral.
- Discard neutral and anything under `CONFIDENCE_MIN`.
- Increment in-memory counters `[slice][emotion]`. A `de` post increments both `global` and `de`.

### Terms

For each classified non-neutral post, tokenize the text, drop stopwords for its language, drop the feeling phrases themselves, and add term counts to an in-memory map per slice for the current day. Keep the map bounded (top 5,000 terms). Never write post text anywhere.

### Hourly flush

At minute 0 every hour, upsert the in-memory counters into `hourly_counts` for the previous hour and reset them.

### Daily job (05:00 Europe/Berlin)

For each slice:

1. Sum the previous 24 hours of `hourly_counts`. `share[e] = count[e] / sum(counts)`.
2. Load the previous `BASELINE_DAYS` of `daily_mood.shares`. If fewer than `BASELINE_DAYS` rows exist, set `calibrating = true`, `label = "Calibrating"`, `zscores = null`.
3. Otherwise `z[e] = (share[e] - mean[e]) / stdev[e]` (guard stdev = 0 → z = 0).
4. Label: find the emotion with the highest z. If it is above +1: joy → Upbeat, anger → Tense, fear → Anxious, sadness → Heavy. Otherwise Steady.
5. Top terms: rank today's terms by frequency relative to a rolling baseline (store a compact baseline of term document frequency in a JSON file or table; simple ratio is enough). Keep top 10.
6. Call Claude Haiku with shares, z-scores, label, top terms and slice. Prompt: return JSON `{ "sentence": string under 20 words, "driver": one term from the list }`. Plain language, no metaphors, no em dashes. Parse defensively; on failure store `sentence = null`.
7. Upsert `daily_mood`. Reset the term map.

### API

```
GET /api/mood/today?slice=global|de
→ { date, slice, label, color, sentence, driver,
    shares: { joy, fear, anger, sadness },
    zscores: { joy, fear, anger, sadness } | null,
    post_count, calibrating }

GET /api/mood/history?slice=global|de&days=7     (max 31)
→ [ { date, label, zscores } ]

GET /health
→ { stream_connected, last_event_age_s, posts_last_hour, current_url }
```

Colors by label (hex for the widget, muted): Steady `#8A9A8E`, Upbeat `#D9A441`, Tense `#C0603A`, Anxious `#6B6FA8`, Heavy `#5E6B7A`, Calibrating `#A0A0A0`.

Enable CORS for `CORS_ORIGIN` only.

## Tasks and checkpoints

### Task 1: Stream spike
Ingestion, filter, sampler, bot guard. No classifier yet. Log per hour: seen, kept, sampled per language.
Checkpoint: run for 30 minutes locally. Report posts seen and kept per language per hour, and memory and CPU usage.

### Task 2: Classifier
Find a multilingual emotion model that runs in `@huggingface/transformers` (ONNX weights available). Start with `MilaNLProc/xlm-emo-t`; if no ONNX export exists, check for a converted version or convert it with Optimum; if none of that works, propose a Python sidecar and stop for approval.
Build `test/fixtures/labeled_posts.csv` with 60 captured matched posts per language (text kept only in this local fixture, gitignored). I will label them.
Checkpoint: report inference time per batch, memory footprint, and precision against my labels.

### Task 3: Storage and hourly flush
Migrations, hourly flush, retention job.
Checkpoint: show one hour of rows in `hourly_counts`.

### Task 4: API and dashboard widget
Endpoints as specified. Return Calibrating state until 28 days exist.
Checkpoint: curl output for all three endpoints.

### Task 5: Daily job
Shares, z-scores, label, terms, Haiku call. Add `daily.test.js` with synthetic data covering: calibrating, Steady, each of the four labels, stdev zero.
Checkpoint: run the job manually against test data and show the stored row.

### Task 6: Deploy
Railway service with Postgres plugin. `npm start` runs everything. Add a README with env setup and a Railway section.
Checkpoint: health endpoint live on Railway for 24 hours with `stream_connected = true` above 95% of checks.

## Out of scope

Market fear, Wikipedia attention, disaster feeds, X data, geolocation, a public page, per-topic slices. Do not build these.

## Open decisions (defaults in brackets; ask me if a task depends on one)

- Database: Railway Postgres [default] or existing Supabase.
- Separate Railway service [default] or inside the dashboard backend.
- Widget frontend contract: ask me for the dashboard repo before Task 4 so the response shape matches its widget pattern.

## Resolved decisions

- This file in the repo is the source of truth for the spec.
- Plain JS with JSDoc. Tests use `node:test`; dev uses `node --watch`.
- Bot guard salt: random per process, generated at boot, held in memory only. No env var.
- Language: use the first entry in `commit.record.langs` that is en, de or es. If none, drop.
- Hour buckets come from the event's `time_us`, not wall-clock arrival time. This keeps cursor replays in the right hour.
- English drop list includes `i feel like` (usually an opinion, not a feeling).
- Task 1 spike logs seen, kept and sampled per language every 5 minutes; hourly rates are extrapolated for the checkpoint.
