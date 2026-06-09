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
  → User taps "Approve & post" (via draft/inbox mode)
  → Clip lands in creator's TikTok inbox for final posting
```

The signal-detection step is the cost lever: instead of sending an entire
multi-hour VOD to the model, only the windows that *look* eventful (loud audio,
rapid scene changes, chat going off) get sent to Claude. On a 3-hour stream this
is the difference between a few cents and several dollars per analysis.

## Launch Strategy: The Safe Path

This repo is built for **human-in-the-loop posting**, not fully autonomous public publishing. 
The approved approach:

1. **Clips auto-generate** (via signal detection + Claude ranking)
2. **Creator reviews in approval queue** (fast—just click titles/hashtags)
3. **Posted to TikTok draft/inbox mode** (`asDraft: true`)
   - Clip lands in the creator's own TikTok inbox
   - Creator can preview, tweak captions, and publish in-app
   - No TikTok audit required
   - Zero risk of platform violations

This pattern:
- ✅ Passes TikTok's policy (you're not auto-posting public)
- ✅ Protects quality (creator can edit before going live)
- ✅ Works immediately (no audit wait)
- ✅ Builds confidence and usage data for future auditing

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
| Auth | **⚠️ Dev shim in `middleware.ts` — must replace before shipping** |
| TikTok publish (`lib/tiktok.ts`) | Working API calls with `asDraft: true` |

## Pre-Launch Checklist

**Security & Compliance:**
- [ ] Replace auth dev shim in `middleware.ts` with real auth (OAuth / SSO / email magic link)
- [ ] Add rate limiting on `/api/process` and `/api/approve` endpoints
- [ ] Sanitize user inputs (captions, hashtags) before storing
- [ ] Verify Redis + Postgres credentials are environment-only (not in repo)

**Infrastructure:**
- [ ] Deploy Redis (Upstash / local)
- [ ] Deploy Postgres (Vercel Postgres / local / Docker)
- [ ] Host workers on a separate machine (or Render / Railway)
- [ ] Verify `ffmpeg` and Whisper CLI are installed on worker machine
- [ ] Set up environment variables (`.env`) — see `.env.example`
- [ ] Test end-to-end: upload → detect → approve → post to TikTok inbox

**TikTok Setup:**
- [ ] Register app at [TikTok Developer Console](https://developer.tiktok.com/)
- [ ] Generate OAuth app credentials (`CLIENT_ID`, `CLIENT_SECRET`)
- [ ] Add redirect URI: `https://yourdomain.com/auth/tiktok/callback`
- [ ] Ensure `asDraft: true` is set in `lib/tiktok.ts` publishing logic
- [ ] **Do not request `video.publish` scope** at launch (you won't get it until audit)

**Testing:**
- [ ] Upload a short test VOD
- [ ] Verify signal detection fires correctly
- [ ] Verify Claude generates reasonable clip titles/captions
- [ ] Approve a clip in queue
- [ ] Confirm clip lands in TikTok inbox (check creator's TikTok app)
- [ ] Creator manually publishes from TikTok inbox

## The TikTok Policy (Important)

TikTok's Content Posting API **does not allow unaudited apps to post public videos**:
- Posts are forced to `SELF_ONLY` (private) visibility unless you pass their audit
- The `asDraft: true` workaround avoids this entirely—you're just filling the creator's inbox

**Don't ship "fully automatic public posting" as a headline feature** until you hold the 
audited `video.publish` scope. Building auto-public from day one is the fastest way to get 
rejected or rate-limited.

Once you have usage + retention data (2–3 months), apply for audit to unlock public posting.

## Setup

```bash
npm install
cp .env.example .env        # fill in keys (see checklist above)
npm run db:push             # initialize Postgres schema
npm run dev                 # frontend (http://localhost:3000)
npm run worker:process      # in a second terminal (clip detection)
npm run worker:publish      # in a third (TikTok posting)
```

Requires:
- `ffmpeg` on worker machine
- Whisper CLI on worker machine (OpenAI's local whisper, or swap for API call in `lib/whisper.ts`)
- Redis (connection string in `.env`)
- Postgres (connection string in `.env`)

## Debugging

**Clips not generating?**
- Check worker:process logs for signal detection output
- Verify Claude is being called (check `ANTHROPIC_API_KEY`)
- Look at the `clips` table—are rows being created with status `PENDING_APPROVAL`?

**TikTok posts not landing in inbox?**
- Verify OAuth token is valid (check `accounts` table)
- Confirm `asDraft: true` in the publish logic
- Check TikTok API response in worker:publish logs

**Auth failing?**
- Replace the dev shim in `middleware.ts` before shipping
- Test with real OAuth credentials
