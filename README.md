# Twitch → Short-form Clip Pipeline

Upload a Twitch VOD → auto-detect viral moments → cut clips → generate copy →
review and publish to TikTok.

## Pipeline

```
Upload (UploadThing → S3/R2)
  → Whisper transcript
  → Signal detection (audio spikes + scene cuts + chat spikes)   ← cheap pre-filter
  → Claude ranks candidates + writes title/hook/caption/hashtags
  → FFmpeg cuts + crops to 9:16
  → Clips land in an APPROVAL QUEUE (status: PENDING_APPROVAL)
  → User taps "Approve & post"
  → TikTok Content Posting API
```

The signal-detection step is the cost lever: instead of sending an entire
multi-hour VOD to the model, only the windows that *look* eventful (loud audio,
rapid scene changes, chat going off) get sent to Claude. On a 3-hour stream this
is the difference between a few cents and several dollars per analysis.

## What's wired vs. what you must finish

| Piece | Status |
|-------|--------|
| Signal detection (`lib/signals.ts`) | Working logic, runs against real ffmpeg |
| FFmpeg cut + 9:16 crop (`lib/ffmpeg.ts`) | Working |
| Claude clip detection (`lib/claude.ts`) | Working, needs `ANTHROPIC_API_KEY` |
| Whisper (`lib/whisper.ts`) | Assumes a local whisper CLI — swap for your install |
| Approval queue + workers | Working logic, needs Redis + Postgres |
| `lib/db.ts` | Done — Prisma singleton |
| `lib/storage.ts` | Done — S3/R2 upload + signed URLs |
| API routes (process, jobs, clips, approve, reject) | Done |
| Upload page → UploadThing → enqueue | Done |
| TikTok OAuth (connect + callback) | Done — needs client key/secret + audit |
| Auth | Dev shim in `middleware.ts` — **replace before shipping** |
| TikTok publish (`lib/tiktok.ts`) | Working API calls — but see the caveat below |

## The TikTok caveat (read before promising auto-posting)

TikTok's Content Posting API does **not** let an un-audited app post public videos.
Until your app passes TikTok's audit:

- posts are forced to `SELF_ONLY` (private) visibility, and
- only to the authenticated creator's own account.

For a real product, the dependable pattern is the **approval queue** this repo is
built around: clips generate automatically, but a human clicks "Approve & post."
You can also use draft/inbox mode (`asDraft: true`) which drops the clip into the
creator's TikTok inbox to finish posting in-app. That path needs the least review
and aligns with platform policy.

Don't ship "fully automatic public posting" as a headline feature until you hold
the audited `video.publish` scope. Building it as auto-public from day one is the
fastest way to get an app rejected or rate-limited.

## Setup

```bash
npm install
cp .env.example .env        # fill in keys
npm run db:push
npm run dev                 # frontend
npm run worker:process      # in a second terminal
npm run worker:publish      # in a third
```

Requires `ffmpeg` and a Whisper CLI on the host running the workers.
