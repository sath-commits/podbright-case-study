# PodBright — Case Study

**PodBright converts emails, newsletters, PDFs, and web articles into personalized podcast episodes, delivered to any podcast app via a private RSS feed.**

Live product: [podbright.ai](https://podbright.ai) · Core repo: private · Case study: public

---

## Problem

Knowledge workers subscribe to 10–20 newsletters but read fewer than half. The content they care about gets buried in email, saved to read-later apps that never get opened, or skimmed on a screen when they have five minutes and no cognitive bandwidth.

The constraint isn't interest — it's attention format. People have 40+ minutes of commute or exercise time that is audio-compatible but not screen-compatible. Existing solutions (podcasts, audiobooks) don't cover the long-tail content that matters most to any individual.

The gap: personal, high-volume text content with no audio equivalent.

---

## Solution

PodBright is a full-stack pipeline that converts any text content into a podcast episode indistinguishable from professionally produced audio.

**Inputs:** Email forward, PDF attachment, Chrome extension (web article)  
**Output:** Audio episode in the user's private RSS feed — available in Apple Podcasts, Overcast, Pocket Casts, AntennaPod, and any RSS-capable app

The critical insight is that text-to-speech alone produces bad audio. Reading a newsletter aloud sounds like a robot reading a webpage because newsletters are *written* to be *read*, not heard. PodBright runs an LLM rewriting step before synthesis: removing email chrome (headers, footers, ad copy, tracking artifacts), restructuring bullet lists into prose, expanding abbreviations, and smoothing transitions. The result sounds edited, not extracted.

---

## Demo

> **Live:** [podbright.ai](https://podbright.ai)

| Dashboard | Settings | Episode |
|---|---|---|
| *(screenshot)* | *(screenshot)* | *(screenshot)* |

---

## Traction

| Metric | Value |
|---|---|
| Registered users | 20 |
| Episodes generated | 185|
| Weekly active users | 16 |
| Avg episodes/user/week | 2|
| 30-day retention | 13 |

*Metrics presented are from production analytics (Supabase). Placeholders to be filled before publishing.*

> **Why these metrics matter:** Episode count demonstrates actual pipeline utilization, not just signups. 30-day retention validates whether the habit forms. Avg episodes/user/week shows engagement depth.

---

## Product Flow

```
1. User signs up
   └── Gets private inbox address (inbox+<token>@podbright.ai)
   └── Gets private RSS feed URL (/api/feed/<feed_token>)
   └── Adds RSS feed once to their podcast app

2. User sends content (any of three paths)
   ├── Email forward → inbound email webhook
   ├── PDF attachment → email webhook + PDF parser
   └── Chrome extension → direct API call with scraped article text

3. System processes content
   ├── Validates sender token from To: address
   ├── Extracts clean text (strips HTML, email headers, tracking pixels)
   ├── Checks rate limits and monthly episode cap
   ├── Creates episode record (status: queued)
   └── Triggers async processing

4. Processing pipeline (async)
   ├── Loads user preferences (voice, mode, difficulty, target duration)
   ├── Rewrites content for audio via LLM
   │   ├── Verbatim mode: cleans and structures only
   │   └── Summary mode: condenses to key points at target duration
   ├── Generates audio via TTS API
   ├── Uploads audio to Cloudflare R2
   ├── Generates chapter markers (for long content)
   ├── Fetches cover image from Unsplash
   └── Updates episode to status: ready

5. Delivery
   └── Podcast app polls RSS feed
       └── New episode appears automatically
```

---

## Architecture

See [`docs/architecture.md`](docs/architecture.md) for full diagram.

### High-level components

```
┌─────────────────────────────────────────────────────────────┐
│                        Inputs                               │
│  Email/PDF Webhook    Chrome Extension    (future: API)     │
└───────────────┬───────────────┬───────────────────────────-─┘
                │               │
                ▼               ▼
┌─────────────────────────────────────────────────────────────┐
│                   Next.js API Routes                        │
│   /api/webhook/email    /api/episodes/article               │
│   • Token validation    • Extension auth                    │
│   • Content extraction  • Rate limiting                     │
│   • Episode creation    • Monthly cap check                 │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                   Processing Pipeline                       │
│   Supabase episodes table (status machine)                  │
│   queued → processing → ready / failed                      │
│                                                             │
│   LLM Rewriter → TTS Generator → R2 Upload                 │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                     Delivery                                │
│   /api/feed/:token → RSS XML → Podcast App                  │
│   Cloudflare R2 → Audio file (direct CDN URL)               │
└─────────────────────────────────────────────────────────────┘
```

### Data stores

| Store | Purpose |
|---|---|
| Supabase Postgres | Episodes, profiles, queue state, analytics |
| Cloudflare R2 | Audio file storage and CDN delivery |
| Supabase Auth | User authentication |

---

## Technical Decisions

See [`docs/decisions.md`](docs/decisions.md) for full write-up.

### LLM rewriting before TTS (not after)

The rewriting step is the core product differentiator. The decision was to run a full LLM rewrite — not a light cleanup — before synthesis. This costs more per episode (~2–4¢ in LLM tokens) but the audio quality difference is large enough to be immediately obvious. Users who heard early versions without the rewrite described the output as "robotic" and "hard to follow." The rewrite step restructures not just words but sentence rhythm, handles enumeration (converting bullet lists to spoken sequences), and removes content that is visually meaningful but aurally noisy (e.g., "Click here", link text, image captions).

The alternative — post-processing audio — is not viable because TTS errors caused by bad input text cannot be corrected after synthesis without re-generating.

### TTS provider selection

Evaluated: OpenAI TTS, ElevenLabs, Unreal Speech, Google Cloud TTS.

Chose Unreal Speech for production:
- ~10x lower cost per character than OpenAI TTS at the same audio quality tier
- Accepts long-form input without chunking (OpenAI has a 4096-token limit requiring stitching)
- Low latency (<30s for a 10-minute episode)
- Multiple neural voices with distinct character

OpenAI TTS was ruled out not on quality but on economics: at scale, the per-character cost becomes the dominant infrastructure cost. ElevenLabs produces the best output but at a price point that requires a premium tier the product wasn't ready to charge for at launch.

### RSS delivery with token-in-URL (not authenticated)

Podcast apps do not support OAuth, session tokens, or standard HTTP auth in a consistent way. Apple Podcasts in particular silently drops feeds that require authentication. The design decision was to use a long random token embedded in the feed URL as the access control mechanism. This trades off theoretical security (anyone with the URL can access the feed) against practical compatibility (works with every podcast app, zero friction for users). Users are warned not to share their feed URL.

### Atomic episode claiming (race condition prevention)

The processing queue is implemented directly in Postgres with a status column rather than a dedicated queue service (SQS, BullMQ). The risk with this pattern is that two concurrent requests claim the same episode. The fix: the status update from `queued` → `processing` is conditional on the current status still being `queued`:

```sql
UPDATE episodes SET status = 'processing'
WHERE id = $1 AND status = 'queued'
RETURNING id;
```

If this returns zero rows, the claim failed — another worker already claimed it. This is a single round-trip, uses Postgres row-level locking, and requires no external coordination. For the current traffic volume, Postgres as a queue is entirely sufficient. The threshold for migrating to a dedicated queue is roughly 100+ concurrent users processing simultaneously.

### Cloudflare R2 over S3

R2 has zero egress fees. Audio files are 5–25 MB each and are fetched by podcast apps on every episode play (apps do not reliably cache). S3 egress at scale becomes a material cost. R2's S3-compatible API required no code changes to the upload layer.

### Fire-and-forget analytics

All analytics writes (page views, events, episode jobs) use a fire-and-forget pattern — wrapped in `try/catch`, never awaited on the critical path, never capable of causing a 500. The reasoning: analytics data is valuable but never worth failing a user-facing request. The cost of a missed event is low; the cost of a failed episode creation because an analytics insert timed out is unacceptable.

---

## Challenges and Tradeoffs

### Email variability

Inbound emails arrive in every conceivable format. HTML newsletters use undocumented layout conventions, inline styles, comment blocks, and sometimes base64-encode their content. The parser must handle:
- Multi-part MIME (HTML + plain text; prefer plain when available)
- Malformed HTML that `cheerio` can parse but produces garbage text
- Forwarded email threads (quoted replies must be stripped)
- PDF attachments treated differently from inline content
- Tracking pixels and link redirectors that pollute extracted text

The solution is a layered extraction: try plain text → fall back to HTML stripping → validate minimum length → reject with error if insufficient content found. Each failure mode produces a specific error code surfaced to the user.

### Processing latency

End-to-end processing (LLM rewrite + TTS + R2 upload) takes 15–45 seconds depending on content length. The challenge is user expectation: users who forward an email and immediately open their podcast app see nothing for 30 seconds.

Mitigation:
- Episode appears immediately as "Queued" in the dashboard
- Status updates to "Processing" as soon as the job starts
- Processing is fully async; the email webhook returns 200 in <2s
- Podcast app polling interval is separate from processing — apps poll every 15–30 minutes anyway, so the 30s processing delay is irrelevant in steady-state usage

### Cost scaling

At low volume, costs are trivial. The cost inflection point is TTS — it scales linearly with character count, not user count. A user who forwards 10 long newsletters/week costs 5–10x more to serve than a user who sends 2 short ones.

Mitigations:
- Monthly episode cap (enforced pre-processing)
- Content length cap (120,000 characters)
- Summary mode reduces TTS input by 60–80%
- Target duration setting (3/5/10 min) limits TTS output length directly

---

## What I Would Build Next

**Short term (product completeness)**
- Email digest mode: batch multiple inputs from the same day into a single episode
- Playback position sync across devices (requires a player, not just RSS)
- Per-episode settings override (current settings are account-wide)

**Medium term (distribution)**
- Zapier/Make integration for non-Gmail email clients
- Slack and Notion as input sources
- Shareable public episodes (opt-in)

**Infrastructure (for 10x users)**
- Separate the processing pipeline from Next.js API routes into a dedicated worker service — the current architecture runs processing inside the web process, which works but limits horizontal scaling
- Introduce a proper job queue (BullMQ or SQS) with retries, dead-letter queues, and priority lanes
- Pre-warm LLM connections to reduce cold-start latency on the rewriting step
- Per-user cost tracking and automated alerts for high-cost accounts

**Product intelligence**
- Listening analytics: which episodes get completed vs abandoned
- Auto-discovery of forwarding rules (detect newsletter patterns in inbox)
- Voice quality personalization based on content type (technical vs narrative)

---

## Scaling to 10x: Technical Architecture

See [`docs/decisions.md#scaling`](docs/decisions.md#scaling) for full analysis.

### Current bottlenecks

| Bottleneck | Threshold | Mitigation |
|---|---|---|
| LLM rewrite | ~50 concurrent jobs | Async queue; no blocking |
| TTS API | Rate limits per account | Retry with backoff; provider-level limit negotiation |
| Postgres as queue | ~100 concurrent workers | Migrate to BullMQ/SQS at this threshold |
| R2 upload | None observed | R2 scales horizontally |
| RSS feed generation | Per-request DB query | Add Redis cache on feed token; TTL 60s |

### Redesign for 10x

```
Current:  Next.js API → inline processEpisode() → Postgres status updates

10x:      Next.js API → enqueue job (SQS/BullMQ)
                            ↓
                    Dedicated worker fleet (auto-scaling)
                            ↓
                    Parallel: LLM rewrite + image fetch
                            ↓
                    TTS generation (with provider fallback)
                            ↓
                    R2 upload → Postgres update → push notification
```

Key additions:
- **Worker autoscaling** based on queue depth, not request rate
- **Provider fallback**: if primary TTS fails, retry on secondary provider
- **Parallel enrichment**: chapter generation and image fetch run concurrently with TTS rather than sequentially
- **Feed caching**: RSS XML cached in Redis, invalidated on new episode, eliminates per-request DB query for high-traffic feed URLs

### Cost control at scale

- Tiered pricing enforces caps before they hit infrastructure
- LLM rewrite cost is the second-largest cost center after TTS; optimizing prompt token count (shorter system prompt, trimmed context) reduces it 20–30% with no quality impact
- Summary mode is a cost-control lever as well as a UX feature — shorter TTS input = cheaper episode
- Per-user cost tracking enables identifying outlier accounts before they cause margin problems

---

## Failure Handling

### Input failures (graceful)
- Content too short → reject immediately, inform user
- PDF encrypted → detected pre-processing, specific error message
- Content over limit → truncated with warning, not rejected
- HTML-only email with no extractable text → fallback chain, then reject

### Processing failures (recoverable)
- LLM timeout → episode marked failed with `error_type: timeout`; user can resubmit
- TTS API error → same pattern; error surfaced in dashboard
- R2 upload failure → episode fails; audio not partially stored
- All failures log structured error type + message for analytics

### Idempotency
- Episode IDs are UUIDs generated at creation time
- Duplicate email forwards (Gmail sometimes sends twice) create duplicate episodes — known issue, mitigated by displaying episode list with title so users can identify and delete duplicates
- Processing claim is atomic: double-processing the same episode is not possible

---

## Why This Exists

Most AI product demos generate text. This project uses AI at the processing layer of a real distribution system: content arrives unpredictably, in uncontrolled formats, must be transformed to a different medium, and delivered reliably to third-party clients (podcast apps) with no retry mechanism.

The interesting problems are not in the AI calls — they're in the plumbing: email parsing reliability, audio latency UX, cost scaling, RSS compatibility across 20+ podcast clients, and building a system that has to work on the first try for every input because there's no "undo" in a podcast app.

This is the kind of problem that differentiates AI product engineering from AI demos. The model is one component. The system is the product.

---

## Stack

| Layer | Technology | Why |
|---|---|---|
| Frontend + API | Next.js (App Router) | Single deployment, RSC for dashboard, API routes for webhooks |
| Database + Auth | Supabase | Postgres + Auth in one; RLS for data isolation; no separate auth service |
| Audio storage | Cloudflare R2 | Zero egress fees; S3-compatible API |
| TTS | Unreal Speech | Best cost/quality ratio for long-form; no chunking required |
| LLM rewriting | OpenAI / Claude | Prompt-swappable; model selection based on cost and output quality |
| Email ingestion | Inbound webhook | Decoupled from MX; works with any forwarding setup |
| Mobile | React Native (Expo) | Shared type definitions with web; iOS + Android from one codebase |

---

*This is a case study repository. The core codebase is private. All code samples are illustrative and sanitized.*
