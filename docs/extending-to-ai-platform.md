# Extending PodBright into an AI Platform

This section describes how the current architecture could evolve from a single-purpose pipeline into a composable AI platform for content transformation.

---

## Current state

PodBright is a linear pipeline:

```
Input → Parse → Rewrite → Synthesize → Deliver
```

Each step is deterministic (same input = same output), sequential, and purpose-built for the email → podcast use case.

---

## What "AI platform" means here

Not: a marketplace or generic API.  
Yes: a system where the transformation pipeline is configurable per-user, per-content-type, and per-output-format — and where new input sources and output formats can be added without changing the core processing engine.

---

## Extension 1: Agent-based content routing

Instead of a fixed pipeline, introduce a routing agent that decides how to process content based on its type and the user's context:

```
Input arrives
    ↓
Router Agent
    ├── Is this a technical paper? → use Technical difficulty, full length
    ├── Is this a news briefing? → use 3min summary
    ├── Is this a long-form essay? → use Standard difficulty, chapters
    └── Is this a product changelog? → skip (user set filter)
```

The agent uses lightweight classification (content type, sender, length) to select pipeline configuration automatically. Users can override but rarely need to.

This transforms PodBright from "one set of preferences applied to everything" to "content-aware processing."

---

## Extension 2: Multi-modal output

The rewriting step currently produces audio-optimized text. The same step could branch to multiple output formats:

```
LLM Rewriter output
    ├── → TTS (audio, current)
    ├── → Summary card (email digest, new)
    ├── → Structured notes (Notion export, new)
    └── → Key quotes (Twitter/X thread, new)
```

A user sends one newsletter. They get a podcast episode AND a summary card AND a highlights export. The transformation cost is paid once; the outputs are cheap to generate from the rewritten text.

---

## Extension 3: Orchestration across sources

Current: each input source is a separate API endpoint with duplicated logic.

Platform: a unified ingestion layer that normalizes all sources into a `ContentItem` with known schema, then routes to shared processing logic:

```typescript
interface ContentItem {
  id: string;
  source: "email" | "pdf" | "web" | "api" | "slack" | "notion";
  title: string;
  raw_text: string;
  metadata: Record<string, string>;
  user_id: string;
  received_at: Date;
}
```

Adding a new source (Slack DM, Notion page, RSS feed subscription) means writing an adapter that produces a `ContentItem` — the rest of the pipeline is unchanged.

---

## Extension 4: Feedback loop and personalization

Current: user preferences are static (set once in Settings).

Platform: the pipeline logs what content was generated, and a lightweight fine-tuning or retrieval layer learns from user behavior:

- Which episodes did the user listen to completion?
- Which voices correlate with higher completion rates for this user?
- Which content types does this user consistently summarize vs. hear in full?

Over time, the router agent's defaults per user become personalized. This is not fine-tuning a model — it's using behavioral signals to inform pipeline configuration, which is fast and cheap.

---

## What this requires technically

| Capability needed | Implementation |
|---|---|
| Content-type classification | Small model or rule-based heuristics (fast, cheap) |
| Branching pipeline | Replace linear function chain with a DAG executor |
| Source adapters | Interface-based plugin pattern |
| Behavioral logging | Already implemented (episode_jobs, analytics_events) |
| Personalization | Aggregate behavioral signals per user; feed into router |

The foundation — structured content items, a processing queue, analytics instrumentation, and a modular rewriting step — is already in place. The extension is primarily architectural (DAG over linear pipeline) and product (deciding which output formats to prioritize).
