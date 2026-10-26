# Milestones

Status of the World Mood build. Spec: [world-mood-spec.md](world-mood-spec.md).

Paused 2026-10-26: Railway was down. Resume at Task 1.

## Done

- [x] Repo set up: Node `.gitignore`, `.env.example`, spec committed
- [x] Spec reviewed; answers recorded under "Resolved decisions" in the spec

## Tasks

- [ ] **Task 1: Stream spike** (next, not started; needs no Railway, runs locally)
  Ingestion, filter, sampler, bot guard. Checkpoint: 30 min local run, seen/kept per language, memory and CPU.
- [ ] **Task 2: Classifier** - find ONNX build of `MilaNLProc/xlm-emo-t`; build labeled fixture (60 posts per language).
- [ ] **Task 3: Storage and hourly flush** - migrations, flush, 90-day retention.
- [ ] **Task 4: API** - needs the dashboard repo from you first.
- [ ] **Task 5: Daily job** - shares, z-scores, label, terms, Haiku sentence; `daily.test.js`.
- [ ] **Task 6: Deploy** - Railway service + Postgres; README; 24h health check.

## Open questions (decide at the task that needs them)

- Task 3: totals (`total_seen`, `total_kept`, sampled) are repeated on every emotion row; may need a separate per-hour totals table.
- Task 4: `/today` response when no `daily_mood` row exists yet; meaning of `posts_last_hour` in `/health`.
- Task 5: daily window (05:00 to 05:00 vs. previous calendar day) and which date the row gets; flush before the daily job; `post_count` definition; term baseline needs a table (Railway disk is ephemeral); Haiku model id (spec says `claude-haiku-4-5`, current is `claude-haiku-5-5`).
- Task 2: `xlm-emo-t` likely has no neutral label, so `CONFIDENCE_MIN` is the only neutral filter.
