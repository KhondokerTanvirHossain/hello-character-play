# Socrates — Architecture & Technical Design

| | |
|---|---|
| **Version** | 0.2 (draft for validation; builder review incorporated — see Appendix B) |
| **Date** | 2026-10-23 |
| **Implements** | [PRD v1.0](2026-10-22-socrates-prd.md) — stage 1 in full; forward-compatible with stages 2–5 |
| **Product owner** | Tanvir |
| **Drafted by** | Claude |
| **Status** | Awaiting validation |

### How to read this document

Tags as in the PRD: **[D]** decided by the owner, **[P]** proposed by Claude and open to challenge, **[O]** open. Decisions are numbered `AD-n` (architecture decisions). Requirement references (`FR-`, `NFR-`, `AC-`) point to the PRD. Appendix A traces every PRD requirement to the component that implements it.

Facts about the server come from a read-only inspection on 2026-10-23. Model facts and prices come from Anthropic's published model table (cached 2026-10-06) and must be re-checked at build time.

---

## 1. Constraints that shape everything

| Constraint | Source | Consequence |
|---|---|---|
| Shared **t3.medium** (2 vCPU, 3.7 GB RAM, about 1.4 GB free, **no swap**), 20 GB disk with about 4.9 GB free | Server inspection | One small container with a **hard memory limit**; no extra database server; strict disk hygiene |
| **More capacity is available on request**: the owner will add memory or move Socrates to another EC2 instance if needed | Owner [D] | Memory limits protect the other tenants, not a ceiling on the product. The deployment is **portable**: the instance, Docker network, and reverse-proxy wiring are configuration, so moving to a new instance means a new target, not a redesign |
| Live tenants on the box (medscribe-ai, medioo-web, bddrug, n8n, healthbuddy) that **must never be harmed** | Owner [D] | Socrates is fully isolated: own image, own deploy script, own data volume, no host ports, prunes only its own images; the only shared touch-point is one new Caddy site block, applied with the owner's approval |
| Shared **Caddy** owns 80/443 and reaches apps by container name on network `n8n_edge` | Server inspection | Socrates joins `n8n_edge` and is reached as `socrates-mind:8080` |
| Existing deploy pattern: GitHub Actions → GHCR image tagged with git sha → SSH → deploy script with health probe | Server inspection | Socrates mirrors it exactly |
| Public GitHub repo: free Actions minutes, **public CI logs** | GitHub | Nothing secret or tenant-specific is ever printed in CI |
| Public from day zero; testable by the owner on the live site | PRD `P9`, `NFR-11` | Continuous deployment of every green build to `https://philosopher.eliooo.org` |
| Owner is a veteran Java developer | PRD §12 | Mind in Java; frontend kept minimal |
| Budget: Claude API + existing EC2 only | PRD `P7` | Cheapest model first; in-process embeddings; SQLite |

## 2. System overview

```
 Visitor's browser                         Shared EC2 box (t3.medium)
 ┌──────────────────────┐        ┌─────────────────────────────────────────────────────────────┐
 │ Chat client (TS)     │ HTTPS  │  Caddy (shared, n8n-caddy-1)                                │
 │ - transcript, input  │◄──────►│   philosopher.eliooo.org ──► socrates-mind:8080             │
 │ - memory controls    │  WSS   │                                                             │
 │ - debug panel (owner)│        │  ┌───────────────────── socrates-mind (one container) ───┐  │
 └──────────────────────┘        │  │ Spring Boot (Java 21, virtual threads), mem ≤ 640 MB   │  │
                                 │  │  ├─ static chat client                                 │  │
                                 │  │  ├─ WebSocket /ws  (conversation protocol v1)          │  │
                                 │  │  ├─ REST /api/memory/*, /admin/*, /healthz             │  │
                                 │  │  ├─ mind-core: turn pipeline, notebook, safety,        │  │
                                 │  │  │   retrieval (in-process embeddings), condensation  │  │
                                 │  │  ├─ budget ledger, rate limits                         │  │
                                 │  │  └─ SQLite (volume socrates_data)                      │  │
                                 │  └────────────────────────────────┬───────────────────────┘  │
                                 └───────────────────────────────────┼──────────────────────────┘
                                                                     │ HTTPS
                                                         Anthropic API (Claude Haiku 5.5)

 Off the box:  GitHub Actions (build, test, deploy)  ·  Evaluation runner (CI or laptop) ──► Anthropic API
                                                                                         └─► judge (other provider, free tier,
                                                                                              simulated conversations only)
```

## 3. Architecture decisions

| # | Decision | Rationale | Tag |
|---|---|---|---|
| AD-1 | **Java 21 LTS, Spring Boot 4.x** (latest GA at build time), **virtual threads** on | Owner's language; long streaming conversations are I/O-bound; virtual threads keep thread memory low | [P] |
| AD-2 | **One container, one process** serves the client, the WebSocket, the REST API, and the mind | Memory: every extra container costs tens to hundreds of MB on a box with about 1.4 GB free | [P] |
| AD-3 | **WebSocket** (`/ws`, plain JSON text frames, no STOMP) for conversation; **REST** for memory, admin, health | Bidirectional and streaming; carries audio in stage 2 without a new transport; memory secrets stay off the conversation channel (`FR-4.3`) | [P] |
| AD-4 | **SQLite** (WAL mode) in a Docker volume; schema migrations with Flyway | No extra server or memory; single instance; trivially backed up. Never touch the box's existing Postgres | [P] |
| AD-5 | **Official Anthropic Java SDK** (`com.anthropic:anthropic-java`, version pinned at build) behind an internal `LanguageModel` interface; model per role in config | `NFR-9`: model and provider swappable; SDK, not raw HTTP | [P] |
| AD-6 | **Start with `claude-haiku-5-5` for every role** (planner, speaker, condenser, simulated visitor) | Owner decision [D]; upgrade per role by evaluation evidence (`P8`) | [D] |
| AD-7 | **Two calls per turn: Plan (structured JSON), then Speak (streamed text)** | The notebook is validated *before* Socrates speaks (`FR-3.3`); safety and language are decided before any reply (`FR-10.7`); the speaker gets a clear move to execute | [P] |
| AD-8 | **History is stored and replayed as plain text turns**; thinking blocks are never replayed | Avoids the "preserved thinking" history-editing check (editing earlier turns invalidates replayed thinking and can return 400 on new accounts), which our own condensation (AD-9) must be free to do | [P] |
| AD-9 | **Own condensation, not server-side compaction** | Concessions and theses must stay verbatim (`FR-3.2`, `FR-7.2`); server compaction summarises arbitrarily | [P] |
| AD-10 | **Every request stays under 100K input tokens** (condensation triggers at about 70K) | Haiku 5.5 costs $0.10 / $0.50 per MTok up to 100K tokens and $0.50 / $2.50 above: a 5× cliff | [P] |
| AD-11 | **In-process retrieval**: a quantised all-MiniLM-L6-v2 embedding model (ONNX, via LangChain4j) and an in-memory vector index built at **build time** and shipped in the image | No external service or cost; about 60 MB of memory; no start-up indexing | [P] |
| AD-12 | **Corpus is tagged per passage offline** and versioned in the repo | `§5.3` dating rule and always-excluded passages are enforced by filters, not by hoping the model obeys | [P] |
| AD-13 | **Secret word over REST**, hashed with **Argon2id**; the request body for that endpoint is excluded from all logging | `FR-4.3`: never reaches the model, transcript, notebook, logs | [P] |
| AD-14 | **Budget ledger in SQLite** with three pools; checked before every model call | `NFR-3` | [P] |
| AD-15 | **Evaluation runs off the box** (GitHub Actions on manual trigger, or the owner's laptop) | Memory and safety of the shared box; evaluation spend separated | [P] |
| AD-16 | **Chat client in plain TypeScript + Vite**, no UI framework; built into the server's static resources | Smallest possible thing that works; stage 3's world is a separate client on the same protocol | [P] |
| AD-17 | **Deployment mirrors medscribe's pattern** but is entirely separate: `ghcr.io/khondokertanvirhossain/hello-character-play/socrates-mind:<sha>`, `~/socrates/deploy-socrates.sh` | Proven on this box; nothing shared except Caddy | [P] |

## 4. Components

Gradle multi-project build (Kotlin DSL) [P]:

| Module | What it is | Depends on |
|---|---|---|
| `protocol/` | The protocol v1 JSON Schema (single source of truth), generated TypeScript types, and Java records verified against the schema by a test | — |
| `corpus/` | Source texts with licence notes, passage files with tags and Stephanus/section references, the biography bible, the misquote denylist, and a build tool that produces the embedding index | — |
| `mind-core/` | Pure Java, no Spring: persona and prompt assembly, the turn pipeline, notebook, safety routing, retrieval, condensation, memory extraction, beat parsing, `LanguageModel` interface and its Anthropic implementation | `protocol`, `corpus` artefacts |
| `mind-server/` | Spring Boot application: WebSocket handler, REST endpoints, persistence (SQLite), identity, budget ledger, rate limiting, owner authentication, retention jobs, static client | `mind-core` |
| `client-chat/` | TypeScript chat page and debug panel | `protocol` (generated types) |
| `eval/` | Simulated visitors, scenarios, deterministic checkers, judge adapter, calibration and κ computation, reports | `mind-core` (runs the mind in-process) |
| `deploy/` | Dockerfile, `deploy-socrates.sh`, the Caddy site block, runbook | — |

## 5. The turn pipeline

For every visitor message:

| Step | What happens | Notes |
|---|---|---|
| 1 | Receive `visitor.message` on the WebSocket | **Input hygiene** (`NFR-6`): max 2,000 characters, control characters stripped, flood detection; empty or oversized messages get a notice |
| 2 | **Safety screen first**: the deterministic multilingual crisis backstop (§6.4) runs on every message **before** caps and rate limits | A hit goes straight to care mode (§6.7), whatever the caps say |
| 3 | **Rate limit** (per device token, loose per-IP backstop) and **budget check** for the caller's pool | On a cap: in-character notice (`NFR-3`), no model call. The safety reserve (§11) is never blocked by caps |
| 3a | Emit `socrates.thinking` | Within 1 s (`NFR-1`) |
| 4 | **Retrieve** up to 6 passages for (visitor message + current thesis), filtered by tag: *argument*, and *dated life event* dated before spring 399; never *situation-bound*, *dramatic frame*, or *always-excluded* | In-process, about 10–20 ms |
| 5 | **Plan call** (Haiku 5.5, adaptive thinking, effort `medium`, structured output against the plan schema) | See §6.2 for its output |
| 6 | **Validate** the plan: schema, and consistency (cited turns exist, quoted premises are exact substrings of the cited Socrates turn, phase transitions are legal). One retry; if still invalid, continue from the last valid notebook and repair after the reply (`FR-3.3`). **The `safety` field is validated on its own**: if it parses, it is used even when the rest of the plan is invalid; if it does not, the turn is treated as `ambiguous` (§6.7) | |
| 7 | **Route**: (a) safety crisis → *care mode*; (b) non-English, not a crisis → *language notice* in character; (c) otherwise → *normal* | Safety is decided on every language **before** any language handling (`FR-10.7`) |
| 8 | **Speak call** (Haiku 5.5, streamed; thinking off or effort `low`, decided by spike S1) with the route's instructions | Text carries inline beat markers |
| 9 | **Stream**: the server strips beat markers out of the token stream and emits `socrates.delta` (text) and `socrates.beat` (with character position) | The client never sees raw markers |
| 10 | **Persist** the turn, notebook snapshot, sources, usage, cost, latency; update the budget ledger; emit `socrates.done` | Owner sessions also get `debug.turn` |
| 11 | **After the reply (async)**: write memory items for consented, unlocked profiles (from the plan's memory candidates, never from care-mode turns); run condensation if the request size crossed the threshold; repair the notebook if step 6 fell back | |

**Failure paths and safety** — whenever the normal turn cannot run, safety still holds:

| Situation | What the visitor gets |
|---|---|
| Caps or rate limit reached | Backstop already ran (step 2). No hit: in-character cap notice. Hit: care mode, paid from the safety reserve |
| Plan invalid twice | Safety from the separately validated `safety` field; if unparsable, treated as `ambiguous` |
| Model refusal (`stop_reason: "refusal"`) on the plan or speak call | Treated as a **possible safety signal**: the canned care-mode reply (§6.7), not an in-character deflection |
| Anthropic API down or timing out | Backstop hit: canned care-mode reply. Otherwise: in-character "let us pause a moment" notice with retry (`NFR-10`) |

**Latency estimate** [P, verified by spike S5]: Plan call about 1.5–3 s (short structured output plus thinking), Speak first token about 0.5–1 s after → first visible text about **2.5–4 s p50**, within `NFR-1` (4 s p50, 8 s p95). If the plan call is too slow, the first lever is a lower effort on the plan; the second is moving the plan to Sonnet 5.5, which is faster per token but costs more.

## 6. Prompts and model usage

### 6.1 Prompt layout and caching

Each call type has its own **stable, cached prefix** and a **volatile tail**:

```
[system — stable, cached (explicit breakpoint, 5-minute TTL)]
  persona & voice · canonical convictions · behaviour rules & the convictions/answers line
  content policy · safety rules · biography bible · misquote denylist · output format (beats)
[messages — conversation history as plain text turns (automatic caching moves forward each turn)]
  …condensed summary of early conversation (if any)…
  visitor / Socrates turns
[latest user turn]
  the visitor's message
[per-turn context packet — volatile, never cached]
  notebook (thesis, verbatim concessions, target inconsistency, phase, mode)
  remembered facts about this visitor (if unlocked)
  retrieved passages with references
  (Speak only) the plan's chosen move and the route
```

- The context packet is sent as a **mid-conversation system message** (supported on Haiku 5.5), ideally turn-scoped (`clear_at: "next_user_message"`, beta) so it costs nothing on later turns and never disturbs the cached history. **Spike S2** confirms this on Haiku 5.5; the fallback is a text block appended to the latest user turn.
- **5-minute cache TTL**: conversation turns are seconds to minutes apart, so reads keep the cache warm. A visitor who pauses longer pays one cache rewrite, which costs very little at Haiku prices.
- Prompts contain **no timestamps, IDs or other per-request bytes** in the stable parts (they silently break caching). Cache hits are verified from `usage.cache_read_input_tokens` on every turn and shown in the debug panel.
- **No sampling parameters** (`temperature` and similar are rejected by the API since 2026-10-21) and **no assistant prefill** (rejected on current models). Output shape is controlled by structured outputs (plan) and instructions (speak).
- **Refusals**: Haiku 5.5 can stop with `stop_reason: "refusal"` and has no server-side fallback. The server treats it like a failed call: a graceful in-character line, logged for review (`NFR-10`).

### 6.2 The plan (structured output)

The plan call returns one JSON object [P]. **Field order is deliberate** — safety first, then the move, then the notebook changes — so that in stage 2 the speaker can start as soon as `move` has streamed (§6.6):

```json
{
  "input":  { "language": "en", "isEnglish": true },
  "safety": { "level": "none | ambiguous | crisis", "signals": ["..."], "identifiesWithFate": false,
              "minorCue": false, "personalTopic": false },
  "move":   { "intent": "...", "useAnalogy": "...", "endHook": "...", "sincerityReminder": false,
              "mode": "examination | encouragement | play | story" },
  "notebookOps": [
    { "op": "setThesis", "text": "...", "visitorTurn": 12 },
    { "op": "addConcession", "premiseQuote": "...", "socratesTurn": 13, "visitorTurn": 14, "sincere": true },
    { "op": "setTarget", "text": "..." },
    { "op": "setPhase", "phase": "draw_out | premises | inconsistency | revision | aporia | begin_again" }
  ],
  "memoryCandidates": [{ "kind": "name | occupation | thesis | thread | term", "content": "..." }],
  "requests": { "forgetMe": false },
  "beatsOutline": { "mood": "curious", "tension": 1 }
}
```

- **The notebook is held by the server**; the model sends only **changes** (`notebookOps`), so plan output stays small and roughly constant over an hour. The server applies the operations and sends the full current notebook in the next context packet.
- **A concession** (`FR-3.2`) is the pair *(Socrates' premise, the visitor's reply)*. `premiseQuote` must be an **exact substring** of Socrates' turn `socratesTurn` (checked by the server); the visitor's reply is **copied verbatim by the server** from turn `visitorTurn`. A bare "Yes" is therefore always stored with the premise it agreed to. Theses work the same way: the server copies the visitor's own words.
- **`requests.forgetMe`**: when the visitor asks Socrates to forget them (`FR-4.6`), the server shows the page-level "Forget me" confirmation (consent and deletion are always page-level controls, like `FR-4.2`); Socrates acknowledges in character. Nothing is deleted from chat text alone.

### 6.3 Beats in the spoken text

The speaker writes beats inline, for example `⟦mood=ironic action=leans_in tension=2⟧`, from the fixed vocabulary in `protocol/`. The server validates each marker against the vocabulary (unknown values are dropped and logged), removes it from the text, and emits `socrates.beat` with the character position (`FR-5.2`).

### 6.4 Routes

| Route | When | Speaker instructions |
|---|---|---|
| Normal | No crisis; English | Execute the plan's move as Socrates |
| Language notice | Not English, no crisis | In character: he cannot follow that tongue (`§5.9`); short; invites English |
| Care mode | `safety.level = crisis`, or `identifiesWithFate` | Persona dropped; plain, caring language **in the visitor's language**; help resources from `corpus/safety/resources.json` (international directory plus local emergency services) sent as a `safety.resources` event; conversation stays in care mode until the visitor clearly asks to return (`FR-10.4`) |
| Ambiguous | `safety.level = ambiguous` | Normal persona, but the move is a gentle, plain question whether the visitor speaks of themselves (`FR-10.2`) |
| Minors mode | `minorCue` | Normal persona with stricter rules; memory not offered; sensitive topics avoided (`§5.7`) |

A **deterministic backstop** runs alongside the plan: a short multilingual list of unambiguous crisis phrases (English and Bangla first) that forces care mode even if the plan missed it [P].

### 6.5 Condensation

When the next request would exceed about **70K input tokens**, the condenser (Haiku 5.5) rewrites the oldest turns into a short narrative summary. Theses and concessions are **excluded from summarisation** and always present in the notebook verbatim. **Care-mode turns are never summarised** (they are replaced by "a private moment, not recorded"). The new history is `[summary] + recent turns`; one cache rewrite follows, then caching resumes.

### 6.6 Stage 2 latency (designed now, enabled later)

`NFR-1` requires stage 2 to reach about 2 s before speech without dropping the planning step. Stage 1 builds the pipeline so these can be switched on in stage 2 without rework (`P5`):

1. **Streamed plan, early speak**: the plan is streamed; as soon as `safety` and `move` are complete and valid, the speak call starts. `notebookOps` keep streaming and are validated in parallel; if they fail, the notebook is repaired after the reply (as in step 6). Safety is still decided before any speech.
2. **Planning while the visitor speaks**: with live speech recognition, a provisional plan starts on a pause in the visitor's speech and is discarded if they continue.
3. **The `thinks` beat**: the world client renders Socrates' absorbed stillness during the remaining gap, so a pause reads as character rather than lag.

Stage 1 ships with (1) behind a feature flag, off, and measured in spike S5.

### 6.7 Care mode without a model

A **canned care-mode reply** exists for when no model call is possible (API down, refusal, caps with a backstop hit): a short, plain, caring message plus help resources, stored in `corpus/safety/` in **English and Bangla** to start. The language is chosen from the script of the visitor's message (Bengali script → Bangla; otherwise English), with the international directory always included. More languages are added as material is reviewed.

## 7. Retrieval and the corpus

- **Texts** [P]: public-domain English translations. Plato from Project Gutenberg (Jowett) and, where a clean digital text exists, pre-1931 Loeb translations (Fowler, Lamb); Xenophon's *Memorabilia* (Marchant, Loeb 1923) for biography. Each source file records its licence and origin. Perseus texts are CC BY-SA and would need attribution and share-alike for the derived corpus; preferring Gutenberg and Wikisource avoids that.
- **Passages**: about 150–300 words each, with Stephanus (Plato) or book/chapter/section (Xenophon) references.
- **Tagging** (`§5.3`): `argument`, `life_event` (with a date), `situation_bound`, `dramatic_frame`, `always_excluded`, plus topic labels. Done **once, offline**, by Claude with the tagging rules, then **spot-checked by a human**; the always-excluded and situation-bound lists in the PRD are checked exhaustively.
- **Index**: embeddings computed at build time and stored as a binary file in the image (a few MB); loaded into memory at start-up.
- **Biography bible and denylist**: versioned files in `corpus/`, used both in the persona prompt and by the evaluation checkers.

## 8. Identity, memory and privacy

| Element | Design |
|---|---|
| **Device token** | 256-bit random value set as a cookie (`Secure`, `HttpOnly`, `SameSite=Strict`, 13-month expiry); only its SHA-256 hash is stored |
| **Consent** | Page-level "Remember me" control → `POST /api/memory/consent` |
| **Secret word** | Page-level field → `POST /api/memory/secret` (set) or `/api/memory/unlock`; Argon2id (OWASP parameters, about 19 MB, 2 iterations); never on the WebSocket, never logged (endpoint excluded from request logging), never in the model's context |
| **Profiles** | Several per device, each with its own secret word and memory |
| **Resuming on a shared device** | A conversation **bound to a profile** cannot be resumed, listed, or shown until that profile is unlocked in the current session. Only **anonymous** conversations (no profile) are resumable by device alone (`FR-1.5`) |
| **Unlocking mid-conversation** | The current anonymous conversation joins the unlocked profile (a page notice says so). The REST unlock notifies the open WebSocket session of the same device (server-side session registry), which receives `memory.status`. **Socrates does not speak on his own**: the page shows a pre-written recognition line (no model call), and his next reply, triggered by the visitor's next message, uses the memory (`FR-1.3`, `FR-4.4`) |
| **Locking** | Closing the page, starting a new conversation, or 30 minutes idle locks the profile again |
| **Brute force** | 5 failed unlocks per device per 15 minutes; a forgotten word means starting fresh (`FR-4.8`) |
| **What is stored** | Memory items by kind (`name`, `occupation`, `thesis`, `thread`, `term`); sensitive beliefs only as examined theses (`FR-4.5`); nothing from care-mode turns |
| **Forget me** | `DELETE /api/memory` removes the profile, its memory, and its conversations |
| **Retention jobs** (nightly) | Profiles and conversations idle 12 months: deleted. Operational fields of turn logs (notebook snapshot, sources, latency): 90 days. Safety events: 30 days, minimal fields. **Text of care-mode turns** (both sides): deleted after 30 days and replaced by a placeholder; never part of resumed history, condensation, or memory |

## 9. Data model (SQLite)

| Table | Key columns |
|---|---|
| `device` | id, token_hash, created_at, last_seen_at |
| `profile` | id, device_id, secret_hash, consented_at, last_visit_at, minors_mode |
| `conversation` | id, device_id, profile_id (nullable), started_at, last_turn_at, condensed_summary, mode |
| `turn` | id, conversation_id, idx, role, text, created_at, route, **care_mode**, model, input_tokens, cache_read_tokens, cache_write_tokens, output_tokens, cost_usd, latency_first_text_ms |
| `turn_ops` (90-day) | turn_id, notebook_json, plan_json, sources_json, beats_json |
| `memory_item` | id, profile_id, kind, content, source_conversation_id, created_at |
| `budget_ledger` | pool, period (day or month), cost_usd |
| `invite` | id, token_hash, label, created_at, revoked_at |
| `owner_session` | id, created_at, expires_at |
| `safety_event` (30-day) | id, conversation_id, turn_idx, level, created_at |

Calibration labels and failure reviews live with the evaluation runner, not in production (§15).

Backups [P]: nightly `VACUUM INTO` a dated file in the volume, keeping 7 (the database is expected to stay in the tens of MB).

**Migrations**: the deploy is stop-then-start (§13.2), so only one version ever touches the database. Every migration is forward-only and is applied by Flyway at start-up; the deploy script takes a `VACUUM INTO` snapshot first, so a rollback restores the snapshot taken before the failed version.

## 10. Protocol v1

Every frame is JSON: `{"v": 1, "type": "...", ...}`. Within v1, changes are **additive only**; clients ignore unknown types and fields (`FR-13.2`). A `lang` field is reserved on visitor and Socrates text events for stage 5 (`NFR-12`).

| Direction | Type | Payload |
|---|---|---|
| C→S | `hello` | `{ protocol: 1, resumeConversationId? }` (device token travels in the cookie) |
| C→S | `visitor.message` | `{ clientMsgId, text }` |
| C→S | `conversation.new` | `{}` |
| S→C | `session.ready` | `{ conversationId, opening, remembered: false }` |
| S→C | `socrates.thinking` | `{ turnId }` |
| S→C | `socrates.delta` | `{ turnId, text }` |
| S→C | `socrates.beat` | `{ turnId, at, mood?, action?, tension? }` |
| S→C | `socrates.done` | `{ turnId }` |
| S→C | `socrates.notice` | `{ kind: "cap_daily" \| "cap_monthly" \| "rate_limited" \| "too_long" \| "error", text }` |
| S→C | `safety.resources` | `{ resources: [{ name, url, phone?, region }] }` |
| S→C | `memory.status` | `{ consented, unlocked, profileLabel?, recognitionLine? }` |
| S→C | `memory.forget_prompt` | `{}` — the page shows the "Forget me" confirmation (`requests.forgetMe`) |
| S→C | `debug.turn` (owner only) | `{ turnId, plan, notebook, sources, beats, latency, usage, costUsd, cacheReadRatio }` |

Openings (`FR-1.3`) come from `corpus/openings.json`, a reviewed pool, so `session.ready` costs no model call.

## 11. Budget, rate limits, access

- **Pools** (`NFR-3`): `public`, `owner` (owner session or invite token), `eval` (separate Anthropic workspace and API key, enforced by the evaluation runner and by the workspace's own spend limit). Cost per call = usage × the price table in configuration.
- **Safety reserve**: a small separate allowance (**$1/month**) that pays for safety screening and care-mode replies when a pool is capped. It is never blocked by the public caps.
- **Initial caps** [P — owner sets]: public **$5/month, $0.50/day**; owner and testers **$10/month**; evaluation **$5/month**. At Haiku prices one hour-long conversation costs roughly $0.15–0.40 (§14), so the public cap allows about 15–30 hour-long conversations a month and the owner's about 30–60.
- **Rate limits**: per device token 20 messages/minute and 300/day; per IP 120 messages/minute (loose, for shared mobile IPs). Bucket4j, in memory.
- **Bot protection**: no model call before a human message; Cloudflare Turnstile (free) is prepared but switched off until abuse appears.
- **Owner access**: one owner account (Argon2id password, secure session cookie) for the debug panel, invites, labelling; protected by the same rate limits.

## 12. Security

- HTTPS only (Caddy); HSTS; strict Content-Security-Policy (`self` only); WebSocket `Origin` must be `https://philosopher.eliooo.org`.
- REST state changes require same-site cookies plus an `Origin` check.
- Visitor text only ever appears in the `user` role; the notebook and prompts never go to non-owner clients.
- **Extraction guard** (`FR-8.2`): see open item A-7. Mechanism if kept: a short hold-back buffer in the stream (about 200 characters) so a reply that starts reproducing a long run of the system prompt or notebook can be cut before it reaches the visitor and replaced by an in-character deflection.
- "Never reaches any API request" in `FR-4.3` means **the model API**: the secret word does travel, over HTTPS, in the page's own REST request to our server, and nowhere else.
- Secrets on the box in `~/socrates/socrates.env` (mode 600), the same convention as medscribe; GitHub holds only the SSH deploy key and host; CI never prints environment values.

## 13. Deployment

### 13.1 Container

| Setting | Value |
|---|---|
| Base image | Eclipse Temurin 21 JRE (Ubuntu-based, needed for the ONNX runtime's glibc) |
| Memory | `--memory=640m --memory-swap=640m` on the shared box (protects the other tenants); JVM `-Xmx300m`, `-XX:MaxMetaspaceSize=128m`; the embedding model uses native memory. All values are deploy-time configuration, raised if the owner adds memory or moves the service |
| Network | `n8n_edge` only; **no host ports published** |
| Data | Docker volume `socrates_data` → `/data` (SQLite and backups) |
| Logs | `json-file` driver, `max-size=10m`, `max-file=3` (30 MB ceiling on a nearly full disk) |
| Restart | `unless-stopped` |
| Health | `GET /healthz` → `{ service: "socrates-mind", sha, build }` |

### 13.2 Pipeline

1. **On every push to `main`**: unit and contract tests, the evaluation smoke subset (small, on the eval budget), image build, push to GHCR as `socrates-mind:<sha>`.
2. **Deploy job** (SSH to the box): `IMAGE_TAG=<sha> bash ~/socrates/deploy-socrates.sh`, which:
   - refuses to run unless at least **1 GB** is free on `/` after pruning;
   - prunes **only** `socrates-mind` tags, keeping the newest 2 and never the running or target image;
   - pulls the new image **while the old container keeps serving**;
   - takes a database snapshot (`VACUUM INTO`);
   - **stop-then-start**: stops the old container, starts the new one, waits for `/healthz` to report the target sha. **Two Socrates containers never run at once**, so peak memory on the shared box never exceeds one container's limit, and only one version ever opens the database. Expected downtime: about 10–20 seconds per deploy (JVM start and model load); open conversations reconnect automatically and resume;
   - on a failed health check: stops the new container, restores the snapshot, starts the previous image again.
3. **Rollback**: `IMAGE_TAG=<older sha> bash ~/socrates/deploy-socrates.sh`.
4. **Full evaluation suite**: manual workflow (`workflow_dispatch`), never automatic.
5. **Smoke evaluation on every push**: the **deterministic** checks (denylist, leakage, retrieval filters, beat vocabulary, memory isolation) **block** the deploy. The **AI-judged** smoke checks only **report**, because model output varies from run to run and a random failure must not stop deployment.
6. **Secrets in GitHub** [P]: the SSH deploy key, the host, the **evaluation** Anthropic API key (its own workspace with its own spend limit) and the judge key. **The production API key never leaves the box.** GitHub masks secrets in logs and does not expose them to pull requests from forks.
7. Every release produces **release notes and a user-perspective test script** (`NFR-11`) in `docs/releases/`.

### 13.3 One-time set-up (each step needs the owner's approval when it is reached)

1. **DNS** (owner, at Hostinger): A record `philosopher` → `13.205.14.249`.
2. **Server folder**: `~/socrates/` with `deploy-socrates.sh` and `socrates.env` (mode 600).
3. **Caddy** — add exactly one site block to `~/n8n/Caddyfile`, after backing the file up:

   ```
   philosopher.eliooo.org {
   	encode zstd gzip
   	header {
   		Strict-Transport-Security "max-age=31536000"
   		X-Content-Type-Options "nosniff"
   		Referrer-Policy "strict-origin-when-cross-origin"
   	}
   	reverse_proxy socrates-mind:8080
   }
   ```

   Then `caddy validate` inside the Caddy container, and only if valid, `caddy reload` (graceful: other sites are not interrupted). The medioo drift check compares only its own blocks, so it is unaffected. Caddy obtains the TLS certificate automatically once DNS resolves.
4. **GitHub secrets**: deploy SSH key and host; GHCR uses the workflow token.

## 14. Cost model

Assumptions [P]: an hour-long conversation is about 80 turns; each turn is one plan call and one speak call; the cached system prefix is about 8K tokens; history grows to about 20K tokens by the end (about 10K on average); the context packet is about 3K tokens; plan output about 600 tokens including thinking; speak output about 200 tokens.

| Model (per MTok, input / output; cache read about 0.1× input) | One hour-long conversation |
|---|---|
| **Claude Haiku 5.5** ($0.10 / $0.50, under 100K-token requests) | **about $0.15–0.40** |
| Claude Sonnet 5.5 ($2 / $10) | about $2–3 |
| Claude Opus 5.5 ($4 / $20) | about $5–6 |

A full evaluation run (about 170 conversations of about 20 turns plus 10 runs of 50 turns, with a simulated visitor on Haiku) is estimated at **$3–8** on Haiku; the judge runs on another provider's free tier. Every figure is re-measured from real usage in the first builds.

## 15. Evaluation architecture

- **Runner** (`eval/`): runs the mind **in-process** (same code as production) against a **simulated visitor** (Haiku 5.5 with persona prompts: cooperative, stubborn, evasive, confused, hostile, playful, plays-along, character-breaker, apparent minor, non-English).
- **Scenarios and sizes**: as `FR-11.2` and `FR-11.6a`, stored as versioned files.
- **Deterministic checkers** (Java): denylist phrases, modern-vocabulary list (anachronism), beat vocabulary validity, secret-word leakage (the word never appears anywhere), memory isolation between profiles, retrieval filter (no excluded passage ever retrieved).
- **AI judge**: another provider's free tier via a small adapter (`Judge` interface), **only on simulated conversations**. Rubric per `FR-11.3`, grouped into five criteria (`FR-11.5`).
- **Live flags** (`FR-11.4`, `FR-12.4`): an owner-only switch in the debug panel, **off by default**. When on, each of the owner's own turns gets an asynchronous Claude Haiku 5.5 judgement, shown as flags; it runs only on owner and tester sessions, never on public visitors, and is paid from the owner pool. Built last in stage 1.
- **Quote checker**: extracts attributed claims and matches them against corpus, biography bible and denylist; calibrated like the judge.
- **Calibration** — a **local labelling tool** that runs next to the evaluation runner on the owner's laptop (`./gradlew :eval:label`, a small local web page), not on production: no upload of simulated transcripts, no second production login. The owner and spouse label on the same machine, each under their own labeller name. It presents balanced items (about 40 per criterion, half seeded failures), computes κ per criterion, and the judge is trusted only at κ ≥ 0.6. Labels are stored as versioned files in `eval/labels/`.
- **Human adjudication** — every zero-tolerance failure goes to a review queue in the same local tool before it counts.
- **Reports**: Markdown summary plus per-conversation transcripts as CI artefacts; trend of pass rates per release.

## 16. Development approach

- **Test-first** for `mind-core` and the server: unit tests; contract tests for the protocol; integration tests with a **recorded-response fake** of `LanguageModel` (no API cost); evaluation for behaviour.
- **Local run**: `./gradlew :mind-server:bootRun` with a local `.env`; or the same Docker image as production.
- **Walking skeleton first** (`NFR-11`): the first release is end-to-end on `philosopher.eliooo.org` — a page, a WebSocket, a single Haiku call answering in a minimal Socrates voice — before the pipeline is deepened.

### Spikes to run first (each a day or less)

| # | Question | Decides |
|---|---|---|
| S1 | Can Haiku 5.5 produce the plan JSON reliably and speak in a convincing Socratic voice? Speak with thinking off vs effort `low` | Model per role; speak settings |
| S2 | Do mid-conversation system messages with `clear_at` work on Haiku 5.5 with our caching layout? | Context-packet placement |
| S3 | Plain-text history with no replayed thinking: any API errors or quality loss? | AD-8 |
| S4 | Memory footprint of the container under load with the ONNX model | The 640 MB limit |
| S5 | First-text latency p50 / p95 over 100 turns | `NFR-1`, plan effort |

## 17. Forward compatibility

| Stage | What this design already provides | What the stage adds |
|---|---|---|
| 2 Voice | WebSocket transport; protocol additive; beats with positions | Audio events; speech recognition (likely in the browser); speech synthesis location decided then. A self-hosted voice needs roughly 300–500 MB more; the owner will add memory or move to another instance if so (§1) |
| 3–4 World | Beats vocabulary and positions; the director lives in the new client | A three.js client on the same protocol; Caddy route for its static files |
| 5 Any language | `lang` fields reserved; prompts and corpus structured per language; safety already multilingual | Persona and evaluation material per language |

## 18. Open items for the owner

| # | Item | Needed by |
|---|---|---|
| A-1 | Add the DNS A record `philosopher` → `13.205.14.249` at Hostinger | First deploy |
| A-2 | Approve the one-time Caddy change (§13.3) when it is reached | First deploy |
| A-3 | A free API key from another provider for the judge (e.g. Google AI Studio for Gemini) | Evaluation suite |
| A-4 | A separate Anthropic workspace and API key for evaluation (recommended, keeps the eval pool separate) | Evaluation suite |
| A-5 | Confirm initial budget caps (§11) | First public release |
| A-6 | **Evaluation budget**: how many full runs and model-comparison runs per month (§14) | Development plan |
| A-7 | **Public repo**: may the persona prompts, crisis phrase list and evaluation scenarios be public, or must they live outside the repo? | Development plan |

## Appendix A — PRD traceability

| PRD | Where implemented |
|---|---|
| FR-1 Conversation | §5, §10 (`session.ready`, openings pool), `mind-server` |
| FR-2 Examination | §6.2 notebook phases and move; persona prompt |
| FR-3 Notebook | §5 steps 5–6, §6.2, `turn_ops` |
| FR-4 Memory and identity | §8, §9, REST `/api/memory/*` |
| FR-5 Beats | §6.3, protocol `socrates.beat` |
| FR-6 Grounding | §7, AD-11, AD-12 |
| FR-7 Infinite conversation | §6.5, AD-9, AD-10 |
| FR-8 Robustness | Persona prompt; §12 extraction guard |
| FR-9 Engagement | Plan `move.endHook`, modes; metrics in `debug.turn`; rubric |
| FR-10 Safety | §6.4 routes, deterministic backstop, `safety_event` retention |
| FR-11 Evaluation | §15, AD-15 |
| FR-12 Chat client | `client-chat/`, AD-16 |
| FR-13 Protocol | §10, `protocol/` |
| NFR-1 Latency | §5 estimate, spike S5 |
| NFR-2 Streaming | §5 step 9 |
| NFR-3 Cost | §11, §14, AD-14 |
| NFR-4 Hosting | §13 |
| NFR-5 Secrets | §12 |
| NFR-6 Access and abuse | §11, §12 |
| NFR-7 Privacy | §8 |
| NFR-8 Observability | §9 `turn`/`turn_ops`, §13.1 logs |
| NFR-9 Model flexibility | AD-5, AD-6 |
| NFR-10 Reliability | §5 step 6 fallback, §6.1 refusals |
| NFR-11 Testability | §13.2 step 5, §16 walking skeleton |
| NFR-12 Language readiness | §10 `lang`, §17 |

## Appendix B — Builder review (of v0.1) → v0.2

| # | Question | Disposition | Where |
|---|---|---|---|
| 1 | Safety when the normal turn cannot run | Fixed: backstop before caps; safety reserve; refusal treated as a safety signal; canned care mode without a model; `safety` validated on its own | §5, §6.7, §11 |
| 2 | Two containers during deploy; shared SQLite; migrations | Fixed: stop-then-start (10–20 s downtime), snapshot before deploy, forward-only migrations | §9, §13.2 |
| 3 | Evaluation spending | Owner decision | A-6 |
| 4 | API keys in GitHub; smoke evaluation blocking | Fixed: eval and judge keys as GitHub secrets, production key stays on the box; deterministic checks block, AI-judged checks report | §13.2 |
| 5 | Shared-device resume and unlocking | Fixed: profile conversations hidden until unlocked; unlock joins the current conversation; no unprompted model call; REST unlock notifies the WebSocket | §8, §10 |
| 6 | Concession definition; notebook size | Fixed: concession = premise quote + visitor reply copied by the server; model sends notebook operations only | §6.2 |
| 7 | Where labelling and review happen | Fixed: local tool next to the evaluation runner | §9, §15 |
| 8 | Retention of care-mode text | Fixed: deleted after 30 days; excluded from resume, condensation and memory | §6.5, §8, §9 |
| 9 | Stage 2's 2-second target | Fixed: streamed plan with early speak, planning during speech, `thinks` beat; (1) built behind a flag in stage 1 | §6.2, §6.6 |
| 10 | "Forget me" in chat; live judge flags; input moderation | Fixed: page-level confirmation; owner-only live flags, off by default; input hygiene plus the plan's safety assessment | §5, §6.2, §15 |
| 11 | Public repo exposes prompts; extraction guard vs streaming; meaning of "API request" | Owner decision on exposure (A-7); hold-back buffer if the guard is kept; "API request" clarified as the model API | §12, A-7 |
