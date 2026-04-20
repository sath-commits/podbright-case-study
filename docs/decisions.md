# Technical Decisions

A log of non-obvious architectural and product decisions, why each was made, and what the tradeoffs are.

---

## 1. LLM rewriting as a first-class pipeline step

**Decision:** Run a full LLM rewrite of input content before TTS synthesis. This is not optional cleanup — it is the core transformation.

**Why:** TTS APIs convert text to speech phonetically. They cannot compensate for input that was written to be read, not heard. Newsletters contain:
- Bullet points (spoken as "dash item one, dash item two")
- Link text ("click here", "read more")
- HTML artifacts that survive naive stripping
- Email footers, legal disclaimers, unsubscribe notices
- Sentence structures that read well on screen but are hard to parse aurally

The LLM rewrite step removes all of this and restructures content into spoken prose. The difference in output quality is immediately perceptible.

**Cost:** ~2–4¢ per episode in LLM tokens. Acceptable at current scale; addressable via prompt optimization at higher volume.

**Alternative considered:** Post-processing audio. Rejected — errors in TTS output caused by bad input cannot be fixed without re-generating.

---

## 2. TTS provider: Unreal Speech over OpenAI TTS

**Decision:** Use Unreal Speech as the primary TTS provider.

**Evaluation matrix:**

| Provider | Cost/1M chars | Max input | Quality | Notes |
|---|---|---|---|---|
| OpenAI TTS | ~$15 | 4,096 tokens | High | Requires chunking for long content |
| ElevenLabs | ~$30–$60 | 5,000 chars | Best | Too expensive at launch pricing |
| Unreal Speech | ~$1–$2 | Unlimited | High | Best cost/quality for long-form |
| Google Cloud TTS | ~$4–$16 | 5,000 bytes | Medium | Voices less natural |

**Why Unreal Speech:**
- 10x cheaper than OpenAI TTS
- Accepts long-form input without chunking — no stitching artifacts
- 6 high-quality neural voices with distinct character
- Latency acceptable for async processing (<30s for 10-min episode)

**OpenAI TTS rejection reason:** Not quality — economics. Long newsletters average 30,000–60,000 characters. At OpenAI's pricing and with chunking overhead, cost per episode is 5–8x higher than Unreal Speech with no perceptible quality gain for conversational content.

**Risk:** Single provider dependency. Mitigation: TTS provider is abstracted behind a single function; swapping requires changing one file.

---

## 3. RSS feed with token-in-URL access control

**Decision:** Access control for the private podcast feed is a long random token embedded in the URL path (`/api/feed/<feed_token>`), not HTTP authentication.

**Why:** Podcast apps are not browsers. They don't support OAuth, session cookies, or even consistent Basic Auth behavior. Apple Podcasts silently drops feeds that require non-standard authentication. Building auth on top of RSS would make the product incompatible with the primary delivery channel.

**Security model:** The feed token is a 32-character random string. Guessing it is computationally infeasible. The risk is token leakage (user shares URL). Mitigation: UI warns against sharing, token regeneration is possible.

**Alternative considered:** Requiring users to use a dedicated app. Rejected — the product's core value proposition is working with podcast apps users already have. A proprietary player is a distribution problem.

---

## 4. Postgres as the job queue

**Decision:** Use the `episodes` table status column as the queue mechanism rather than a dedicated queue service (SQS, BullMQ, Redis).

**Why:** At current traffic volume, Postgres is sufficient. A dedicated queue adds operational complexity (another service, separate failure modes, credentials) with no benefit until concurrent processing demand exceeds what Postgres can handle with row-level locking.

**Race condition protection:** The claim operation is:
```sql
UPDATE episodes SET status = 'processing'
WHERE id = $1 AND status = 'queued'
RETURNING id;
```
This is atomic at the Postgres level. Only one concurrent transaction can successfully claim a given episode.

**Threshold for migration:** ~100+ simultaneous users processing episodes concurrently. At that point, queue depth visibility, dead-letter handling, and worker autoscaling become meaningful enough to justify BullMQ or SQS.

---

## 5. Cloudflare R2 for audio storage

**Decision:** Store audio files in Cloudflare R2, not AWS S3.

**Why:** Egress fees. Audio files are 5–25 MB each. Podcast apps fetch them on every play and do not reliably cache (each app handles caching differently). At volume, egress from S3 becomes the largest infrastructure cost line item. R2 has zero egress fees with an S3-compatible API — zero code changes required.

**Tradeoff:** R2 has fewer features than S3 (no event notifications, limited lifecycle policies). These features are not required for this use case.

---

## 6. Supabase over self-hosted Postgres + separate auth

**Decision:** Use Supabase for both database and authentication.

**Why:** Speed of development at launch. Supabase provides Postgres, Auth (with JWT, email OTP, OAuth), Row Level Security, and a dashboard in one managed service. The alternative — self-hosted Postgres + NextAuth + a separate admin interface — would have taken 2–3x longer to set up and added multiple failure surfaces.

**RLS usage:** Analytics tables use service-role key only (no RLS needed — never accessed from browser). User data tables use RLS to enforce `user_id = auth.uid()` at the DB level, so even a misconfigured API route cannot leak cross-user data.

**Exit strategy:** Supabase exposes standard Postgres. Migration to self-hosted Postgres is a connection string change.

---

## 7. Fire-and-forget analytics

**Decision:** All analytics writes are fire-and-forget — wrapped in `try/catch`, never awaited on the critical request path, never capable of blocking or failing a user-facing response.

**Why:** The cost of a missed analytics event is low (slightly degraded reporting). The cost of a failed episode creation because an analytics insert timed out or threw is unacceptable (lost user content, frustrated user, broken trust in the product).

**Pattern:**
```typescript
// Never throws, never blocks
trackEvent(ANALYTICS_EVENT.EMAIL_RECEIVED, userId, props)
  .catch(err => console.error("Analytics write failed:", err));
```

**Tradeoff:** Under-counting is possible during analytics service degradation. Acceptable for a product at this stage; not acceptable for billing or usage caps (those use synchronous checks).

---

## 8. Difficulty and duration as user-controlled settings

**Decision:** Expose LLM rewriting complexity and output length as first-class user settings rather than hiding them.

**Options considered:**
1. Fixed pipeline (no user control)
2. Simple/Full toggle
3. Full settings panel (voice, mode, difficulty, duration)

**Why option 3:** The product serves radically different use cases. A software engineer forwarding technical papers wants technical vocabulary preserved and full content. A commuter forwarding casual newsletters wants a 5-minute summary in plain English. These needs cannot be served by a single pipeline configuration. Exposing these as settings lets users adapt the product to their context rather than the product imposing a single output style.

**UX risk:** Too many settings overwhelm new users. Mitigation: sensible defaults (Standard difficulty, Full length, Verbatim mode) that work well for most content. Advanced users discover the settings page.

---

## Scaling Analysis {#scaling}

### Current architecture limits

| Component | Current approach | Limit | Upgrade path |
|---|---|---|---|
| Processing | Inline async in Next.js process | ~20 concurrent | Dedicated worker service |
| Queue | Postgres status column | ~100 concurrent | BullMQ / SQS |
| RSS feed | DB query per request | ~1000 req/min | Redis cache (TTL 60s) |
| TTS | Single provider | Provider rate limit | Provider-level negotiation or multi-provider |
| LLM rewrite | Single provider | Provider rate limit | Same |

### Redesign for 10x users

**Decouple processing from the web process:**

```
Current:
  Webhook handler → createEpisode() → processEpisode() [async in same process]

10x:
  Webhook handler → createEpisode() → enqueue(episodeId)
  Worker fleet (separate service) → dequeue → processEpisode()
```

Benefits: web process never blocks on processing, worker fleet scales independently, failures in processing don't affect ingestion.

**Add provider fallback:**

```typescript
async function synthesize(text: string): Promise<Audio> {
  try {
    return await unrealSpeech.synthesize(text);
  } catch (err) {
    if (isRateLimitError(err)) {
      return await openaiTTS.synthesize(text); // fallback
    }
    throw err;
  }
}
```

**Cache RSS feeds:**

```
GET /api/feed/:token
  → check Redis cache (key: feed_token, TTL: 60s)
  → if hit: return cached XML
  → if miss: query DB, generate XML, write to cache, return
  → on new episode ready: invalidate cache for that user's feed_token
```

Eliminates per-request DB query for feed polling. Most podcast apps poll every 15–30 minutes anyway; 60s TTL means near-real-time delivery with dramatically lower DB load.

**Cost control mechanisms at scale:**

| Mechanism | Implementation |
|---|---|
| Monthly episode cap | Checked synchronously before processing |
| Content length cap | 120,000 chars; enforced at ingestion |
| Summary mode | Reduces TTS input by 60–80% |
| Target duration | Caps TTS output length directly |
| Per-user cost tracking | episode_jobs table enables per-user LLM + TTS cost attribution |
| Anomaly alerts | Flag users exceeding 3x average cost/week for review |
