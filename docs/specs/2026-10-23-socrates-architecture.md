# Socrates — Architecture & Technical Design

| | |
|---|---|
| **Version** | 1.0 (approved) |
| **Date** | 2026-10-23 |
| **Implements** | [PRD v1.0](2026-10-22-socrates-prd.md) — stage 1 in full; forward-compatible with stages 2–5 |
| **Product owner** | Tanvir |
| **Drafted by** | Claude |
| **Status** | **Approved** by the product owner on 2026-10-23 after three review rounds |

### How to read this document

Tags as in the PRD: **[D]** decided by the owner, **[P]** proposed by Claude and open to challenge, **[O]** open. Decisions are numbered `AD-n` (architecture decisions). Requirement references (`FR-`, `NFR-`, `AC-`) point to the PRD. Appendix A traces every PRD requirement to the component that implements it.

Facts about the server come from a read-only inspection on 2026-10-23. **Operational specifics** (host address, the other tenants, file paths, container and network names, DNS provider) are kept in a **private operations file outside this public repository** (A-12); this document uses placeholders such as `<host>`, `<deploy dir>`, `<proxy container>` and "the tenants' network". Model facts and prices come from Anthropic's published model table (cached 2026-10-06) and must be re-checked at build time.

---

## 1. Constraints that shape everything

| Constraint | Source | Consequence |
|---|---|---|
| Shared **burstable t3-class instance** (2 vCPU, 3.7 GB RAM, about 1.4 GB free, **no swap**), 20 GB disk with about 4.9 GB free | Server inspection | One small container with a **hard memory limit**; no extra database server; strict disk hygiene |
| **More capacity is available on request**: the owner will add memory or move Socrates to another EC2 instance if needed | Owner [D] | Memory limits protect the other tenants, not a ceiling on the product. The deployment is **portable**: the instance, Docker network, and reverse-proxy wiring are configuration, so moving to a new instance means a new target, not a redesign |
| Live tenants on the box (several production web apps, their databases, and an automation service; inventory in the private operations file) that **must never be harmed** | Owner [D] | Socrates is fully isolated: own image, own deploy script, own data volume, no host ports, prunes only its own images; the only shared touch-point is one new Caddy site block, applied with the owner's approval |
| Shared **Caddy** owns 80/443 and reaches the other apps by container name on the tenants' network | Server inspection | Caddy is also attached to Socrates' own network `socrates_edge` (AD-18) and reaches it as `socrates-mind:8080` |
| Existing deploy pattern: GitHub Actions → GHCR image tagged with git sha → SSH → deploy script with health probe | Server inspection | Socrates mirrors it exactly |
| Public GitHub repo: free Actions minutes, **public CI logs**. The repo **stays public for now**, including persona prompts and evaluation material; the owner may make it private later [D] | GitHub, owner | Nothing secret or tenant-specific is ever printed in CI. Nothing in the build or deploy assumes the repo is public, so making it private later is only a settings change (private repos get 2,000 free Actions minutes a month, enough for this project) |
| Public from day zero; testable by the owner on the live site | PRD `P9`, `NFR-11` | Continuous deployment of every green build to `https://philosopher.eliooo.org` |
| The box is a **burstable t3** instance (CPU credits) | Server inspection | Socrates gets a **CPU limit**; heavy CPU work (embeddings, frequent JVM warm-ups) is avoided; the credit balance is monitored (§19) |
| Socrates is a **public LLM app** next to tenants holding health data and credentials | Review | Socrates is **network-isolated** from them (AD-18); CI can only run one fixed deploy command (§13.2) |
| Owner is a veteran Java developer | PRD §12 | Mind in Java; frontend kept minimal |
| Budget: Claude API + existing EC2 only | PRD `P7` | Cheapest model first; in-process BM25 retrieval; SQLite |

## 2. System overview

```
 Visitor's browser                         Shared EC2 box (t3)      
 ┌──────────────────────┐        ┌─────────────────────────────────────────────────────────────┐
 │ Chat client (TS)     │ HTTPS  │  Caddy (shared)                                             │
 │ - transcript, input  │◄──────►│   philosopher.eliooo.org ──► socrates-mind:8080 (net: socrates_edge) │
 │ - memory controls    │  WSS   │                                                             │
 │ - debug panel (owner)│        │  ┌───────────────────── socrates-mind (one container) ───┐  │
 └──────────────────────┘        │  │ Spring Boot (Java 25, virtual threads), ≤640 MB, ≤1 CPU │  │
                                 │  │  ├─ static chat client                                 │  │
                                 │  │  ├─ WebSocket /ws  (conversation protocol v1)          │  │
                                 │  │  ├─ REST /api/memory/*, /healthz                       │  │
                                 │  │  ├─ admin port 8081 (127.0.0.1 only, via SSH tunnel)   │  │
                                 │  │  ├─ mind-core: turn pipeline, notebook, safety,        │  │
                                 │  │  │   retrieval (in-process BM25), condensation        │  │
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
| AD-1 | **Java 25 LTS, Spring Boot 4.x** (latest GA at build time), **virtual threads** on | Owner's language; long streaming conversations are I/O-bound; virtual threads keep thread memory low; Java 25 no longer pins virtual threads on `synchronized` (JEP 491) | [P] |
| AD-2 | **One container, one process** serves the client, the WebSocket, the REST API, and the mind | Memory: every extra container costs tens to hundreds of MB on a box with about 1.4 GB free | [P] |
| AD-3 | **WebSocket** (`/ws`, plain JSON text frames, no STOMP) for conversation; **REST** for memory, admin, health | Bidirectional and streaming; carries audio in stage 2 without a new transport; memory secrets stay off the conversation channel (`FR-4.3`) | [P] |
| AD-4 | **SQLite** (WAL mode) in a Docker volume; schema migrations with Flyway; **all database access on a small pool of platform threads with a single writer connection** and `busy_timeout` | No extra server or memory; single instance; trivially backed up. Never touch the box's existing Postgres. The SQLite driver uses native calls, which pin virtual threads to their only 2 carriers; a platform-thread pool keeps streaming unaffected | [P] |
| AD-5 | **Official Anthropic Java SDK** (`com.anthropic:anthropic-java`, version pinned at build) behind an internal `LanguageModel` interface; model per role in config | `NFR-9`: model and provider swappable; SDK, not raw HTTP | [P] |
| AD-6 | **Start with `claude-haiku-5-5` for every role** (planner, speaker, condenser, simulated visitor) | Owner decision [D]; upgrade per role by evaluation evidence (`P8`) | [D] |
| AD-7 | **Two calls per turn: Plan (structured JSON), then Speak (streamed text)** | The notebook is validated *before* Socrates speaks (`FR-3.3`); safety and language are decided before any reply (`FR-10.7`); the speaker gets a clear move to execute. **Fallback** if spike S5 misses p95: one streamed call emitting safety, then the move, then the speech, with the server holding text back until the safety part parses — the stage 2 design pulled forward only if needed | [P] |
| AD-8 | **History is stored and replayed as plain text turns**; thinking blocks are never replayed | Avoids the "preserved thinking" history-editing check (editing earlier turns invalidates replayed thinking and can return 400 on new accounts), which our own condensation (AD-9) must be free to do | [P] |
| AD-9 | **Own condensation, not server-side compaction** | Concessions and theses must stay verbatim (`FR-3.2`, `FR-7.2`); server compaction summarises arbitrarily | [P] |
| AD-10 | **Every request stays under 100K input tokens** (condensation triggers at about 70K) | Haiku 5.5 costs $0.10 / $0.50 per MTok up to 100K tokens and $0.50 / $2.50 above: a 5× cliff | [P] |
| AD-11 | **In-process retrieval with Lucene BM25** (pure Java) over the tagged passages, combined with **topic labels chosen by the plan**; index built at **build time** and shipped in the image. **Embeddings are added only if evaluation shows retrieval misses** | The corpus is small (about 1–2K passages); BM25 needs no native runtime, little memory and little CPU, and keeps the image small. (This supersedes the PRD's proposed embedding model, §12 [P]) | [P] |
| AD-12 | **Corpus is tagged per passage offline** and versioned in the repo | `§5.3` dating rule and always-excluded passages are enforced by filters, not by hoping the model obeys | [P] |
| AD-13 | **Secret word over REST**, hashed with **Argon2id**; the request body for that endpoint is excluded from all logging | `FR-4.3`: never reaches the model, transcript, notebook, logs | [P] |
| AD-14 | **Budget ledger in SQLite** with three pools; checked before every model call | `NFR-3` | [P] |
| AD-15 | **Evaluation runs off the box** (GitHub Actions on manual trigger, or the owner's laptop) | Memory and safety of the shared box; evaluation spend separated | [P] |
| AD-16 | **Chat client in plain TypeScript + Vite**, no UI framework; built into the server's static resources | Smallest possible thing that works; stage 3's world is a separate client on the same protocol | [P] |
| AD-17 | **Deployment mirrors the box's existing deploy pattern** but is entirely separate: `ghcr.io/khondokertanvirhossain/hello-character-play/socrates-mind:<sha>`, `<deploy dir>/deploy-socrates.sh` | Proven on this box; nothing shared except Caddy | [P] |
| AD-18 | **Own Docker network `socrates_edge`**, shared only by Caddy and `socrates-mind`; Socrates is **not** on the tenants' network | A compromised Socrates (or a malicious image) cannot reach the other tenants directly | [P] |
| AD-19 | **Admin and debug on a separate port (8081) bound to the box's 127.0.0.1 only**, reached by the owner through an SSH tunnel, **plus a password login on that port** (defence in depth); never routed by Caddy. The **chat client is also served on the admin port**, so the owner can hold a live conversation with the debug panel beside it (§11) | The panel can read every visitor's transcript; it must not be on the public internet, and loopback alone must not be the only lock (any local process on the box can reach 127.0.0.1) | [P] |
| AD-21 | **The container is hardened and cut off from the instance metadata service** | If the instance has an **IAM role** (whether it does is recorded in the private operations file), a compromised container that could reach the metadata service would obtain AWS credentials affecting every tenant | [P] |
| AD-20 | **CI can run exactly one command on the box**: a deploy key restricted in `authorized_keys` to `deploy-socrates.sh`, which accepts only a 40-character hex sha | SSH plus Docker is effectively root on a box holding other tenants' health data and credentials; a compromised Action or dependency in a public repo must not inherit that | [P] |

## 4. Components

Gradle multi-project build (Kotlin DSL) [P]:

| Module | What it is | Depends on |
|---|---|---|
| `protocol/` | The protocol v1 JSON Schema (single source of truth), generated TypeScript types, and Java records verified against the schema by a test | — |
| `corpus/` | Source texts with licence notes, passage files with tags and Stephanus/section references, the biography bible, the misquote denylist, and a build tool that produces the search index | — |
| `mind-core/` | Pure Java, no Spring: persona and prompt assembly, the turn pipeline, notebook, safety routing, retrieval, condensation, memory extraction, beat parsing, `LanguageModel` interface and its Anthropic implementation | `protocol`, `corpus` artefacts |
| `mind-server/` | Spring Boot application: WebSocket handler, REST endpoints, persistence (SQLite), identity, budget ledger, rate limiting, owner authentication, retention jobs, static client | `mind-core` |
| `client-chat/` | TypeScript chat page and debug panel | `protocol` (generated types) |
| `eval/` | Simulated visitors, scenarios, deterministic checkers, judge adapter, calibration and κ computation, reports | `mind-core` (runs the mind in-process) |
| `deploy/` | Dockerfile, `deploy-socrates.sh`, the Caddy site block, runbook | — |

## 5. The turn pipeline

For every visitor message:

| Step | What happens | Notes |
|---|---|---|
| 0 | **One turn in flight per conversation**: a second message while a turn runs is queued (or rejected with a notice); a resent message with an already-seen `clientMsgId` is ignored | Prevents two pipelines racing on one notebook and one ledger (double send, two tabs, reconnect) |
| 1 | Receive `visitor.message` on the WebSocket | **Input hygiene** (`NFR-6`): max 2,000 characters, control characters stripped, flood detection; empty or oversized messages get a notice |
| 2 | **Safety screen first**: the deterministic multilingual crisis backstop (§6.4) runs on every message **before** caps and rate limits. **A conversation already in care mode stays in care mode**: it bypasses caps and rate limits and is paid from the safety reserve, or gets the canned reply if the reserve is empty | A hit goes straight to care mode (§6.7), whatever the caps say |
| 3 | **Rate limit** (§11) and **budget reservation**: the estimated cost of the turn is reserved in the ledger before any call and settled from actual usage afterwards | **On a cap**: before any cap notice, a **cheap safety-only classification** (Haiku, paid from the safety reserve) runs on the message; distress of any level → care mode or the ambiguous check-in; otherwise the in-character cap notice (`NFR-3`) |
| 3a | Emit `socrates.thinking` | Within 1 s (`NFR-1`) |
| 4 | **Retrieve for the plan**: up to 6 passages by BM25 over the **visitor's message and the current thesis only**, filtered by tag: *argument*, and *dated life event* dated before spring 399; never *situation-bound*, *dramatic frame*, or *always-excluded* | In-process, about 10–20 ms |
| 5 | **Plan call** (Haiku 5.5, adaptive thinking, effort `medium`, structured output against the plan schema) | See §6.2 for its output |
| 6 | **Validate** the plan: schema, and consistency (cited turns exist, quoted premises are exact substrings of the cited Socrates turn, phase transitions are legal). One retry; if still invalid, continue from the last valid notebook and repair after the reply (`FR-3.3`). **The `safety` field is validated on its own**: if it parses, it is used even when the rest of the plan is invalid; if it does not, the turn is treated as `ambiguous` (§6.7) | |
| 7 | **Route** by the single precedence table in §6.4 (crisis > ambiguous > minors > language notice > normal), implemented in one function and unit-tested | Safety is decided on every language **before** any language handling (`FR-10.7`) |
| 7a | **Retrieve for the speaker**: a second BM25 query using the plan's **topic labels** and move, same tag filters | So a change of subject (courage → piety) is reflected in this turn's passages |
| 8 | **Speak call** (Haiku 5.5, streamed; thinking off or effort `low`, decided by spike S1) with the route's instructions | Text carries inline beat markers |
| 9 | **Stream**: the server strips beat markers out of the token stream and emits `socrates.delta` (text) and `socrates.beat` (with character position) | The client never sees raw markers |
| 10 | **Persist** the turn, notebook snapshot, sources, usage, cost, latency; update the budget ledger; emit `socrates.done` | Owner sessions also get `debug.turn` |
| 11 | **After the reply (async)**: write memory items for consented, unlocked profiles (from the plan's memory candidates, never from care-mode turns); run condensation if the request size crossed the threshold; repair the notebook if step 6 fell back | |

**Failure paths and safety** — whenever the normal turn cannot run, safety still holds:

| Situation | What the visitor gets |
|---|---|
| Caps or rate limit reached | Backstop already ran (step 2); a conversation already in care mode continues in care mode; otherwise the safety-only classification runs (step 3). Distress: care mode or ambiguous check-in, paid from the reserve. None: in-character cap notice |
| Plan invalid twice | Safety from the separately validated `safety` field; if unparsable, treated as `ambiguous` |
| Model refusal (`stop_reason: "refusal"`) | **On the plan call**: the turn takes the **ambiguous** route (a gentle, plain check-in), unless the backstop or a parsed `safety` field says crisis, in which case care mode. **On the speak call**: a **pre-written plain check-in** is sent, with **no further model call** (a second call may refuse again). Refusals are mostly triggered by jailbreak attempts and edgy philosophy, so sending helplines to every refusal would mislead and could be triggered on purpose |
| Anthropic API down or timing out | Backstop hit: canned care-mode reply. Otherwise: in-character "let us pause a moment" notice with retry (`NFR-10`) |
| Visitor disconnects mid-stream | The turn is completed and stored server-side; cost is settled from the usage reported, or estimated from streamed tokens if the stream was aborted |
| Deploy (shutdown) during a turn | Graceful shutdown waits up to 30 s for in-flight turns; asynchronous work (memory writes, notebook repair, condensation) is kept in a durable queue (`pending_task`) and resumed on start |

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

- The context packet is sent as a **mid-conversation system message** (supported on Haiku 5.5), ideally turn-scoped (`clear_at: "next_user_message"`, beta) so it costs nothing on later turns and never disturbs the cached history. **Spike S2** confirms this on Haiku 5.5. **Fallback**: the packet is appended as a text block to the latest user turn **for that request only** — it is never stored in history, and on the next turn that user turn is replayed without it; the automatic cache breakpoint therefore lands on the previous turn, and only the latest user turn plus the packet are uncached.
- **5-minute cache TTL**: conversation turns are seconds to minutes apart, so reads keep the cache warm. A visitor who pauses longer pays one cache rewrite, which costs very little at Haiku prices.
- Prompts contain **no timestamps, IDs or other per-request bytes** in the stable parts (they silently break caching). Cache hits are verified from `usage.cache_read_input_tokens` on every turn and shown in the debug panel.
- **No sampling parameters** (`temperature` and similar are rejected by the API since 2026-10-21) and **no assistant prefill** (rejected on current models). Output shape is controlled by structured outputs (plan) and instructions (speak).
- **Refusals**: Haiku 5.5 can stop with `stop_reason: "refusal"` and has no server-side fallback. Handling is defined once, in the failure table of §5; every refusal is logged for review.

### 6.2 The plan (structured output)

The plan call returns one JSON object [P]. **Field order is deliberate** — safety first, then the move, then the notebook changes — so that in stage 2 the speaker can start as soon as `move` has streamed (§6.6):

```json
{
  "input":  { "language": "en", "script": "latin | bengali | other", "mixed": false, "isEnglish": true },
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
- **A concession** (`FR-3.2`) is the pair *(Socrates' premise, the visitor's reply)*. `premiseQuote` is matched against Socrates' turn `socratesTurn` **after normalising** quotes, dashes and whitespace (with a small fuzzy tolerance), and the server then stores the **exact span** from the transcript; the visitor's reply is **copied verbatim by the server** from turn `visitorTurn`. **Every turn in the history the model sees carries its turn number** (e.g. `[T14 Visitor]`), so it can cite them. A bare "Yes" is therefore always stored with the premise it agreed to. Theses work the same way: the server copies the visitor's own words.
- **`requests.forgetMe`**: when the visitor asks Socrates to forget them (`FR-4.6`), the server shows the page-level "Forget me" confirmation (consent and deletion are always page-level controls, like `FR-4.2`); Socrates acknowledges in character. Nothing is deleted from chat text alone.

### 6.3 Beats in the spoken text

The speaker writes beats inline, for example `⟦mood=ironic action=leans_in tension=2⟧`, from the fixed vocabulary in `protocol/`. The server validates each marker against the vocabulary (unknown values are dropped and logged), removes it from the text, and emits `socrates.beat` with the character position (`FR-5.2`).

### 6.4 Routes

**Precedence** (first match wins; one function, unit-tested with every combination):

1. **Care mode** — backstop hit, `safety.level = crisis`, `identifiesWithFate`, or the conversation is already in care mode
2. **Ambiguous** — `safety.level = ambiguous`, or a plan refusal; the plain check-in is written **in the visitor's language**, even if it is not English
3. **Minors mode** — `minorCue` (may combine with the language notice below)
4. **Language notice** — not English
5. **Normal**

| Route | When | Speaker instructions |
|---|---|---|
| Normal | No crisis; English | Execute the plan's move as Socrates |
| Language notice | Not English, no crisis | In character: he cannot follow that tongue (`§5.9`); short; invites English |
| Care mode | `safety.level = crisis`, or `identifiesWithFate` | Persona dropped; plain, caring language **in the visitor's language**; help resources from `corpus/safety/resources.json` (international directory plus local emergency services) sent as a `safety.resources` event; conversation stays in care mode until the visitor clearly asks to return (`FR-10.4`) |
| Ambiguous | `safety.level = ambiguous` | Normal persona, but the move is a gentle, plain question whether the visitor speaks of themselves (`FR-10.2`) |
| Minors mode | `minorCue` | Normal persona with stricter rules; memory not offered; sensitive topics avoided (`§5.7`) |

A **deterministic backstop** runs alongside the plan: a short multilingual list of unambiguous crisis phrases — **English, Bangla in Bengali script, and romanized Bangla** (Bangla typed in Latin letters, often mixed with English) first — that forces care mode even if the plan missed it [P]. The plan's `input` reports `script` and `mixed` as well as `language`, so romanized or mixed Bangla is not mistaken for English. Because romanized spelling varies widely, backstop matching **normalises** text first (lower-case, vowel and doubled-letter folding, common spelling variants) and allows small fuzzy differences.

### 6.5 Condensation

When the next request would exceed a **configurable threshold** (default about **70K input tokens**), the condenser (Haiku 5.5) rewrites the oldest turns into a short narrative summary. Theses and concessions are **excluded from summarisation** and always present in the notebook verbatim. **Care-mode turns are never summarised** (they are replaced by "a private moment, not recorded"). The new history is `[summary] + recent turns`; one cache rewrite follows, then caching resumes. Because a real hour reaches only about 20K tokens, the default rarely fires; the **evaluation runs long conversations with the threshold set to about 8K**, so the condensation path and its verbatim guarantees are exercised.

### 6.6 Stage 2 latency (designed now, enabled later)

`NFR-1` requires stage 2 to reach about 2 s before speech without dropping the planning step. Stage 1 builds the pipeline so these can be switched on in stage 2 without rework (`P5`):

1. **Streamed plan, early speak**: the plan is streamed; as soon as `safety` and `move` are complete and valid, the speak call starts. `notebookOps` keep streaming and are validated in parallel; if they fail, the notebook is repaired after the reply (as in step 6). Safety is still decided before any speech.
2. **Planning while the visitor speaks**: with live speech recognition, a provisional plan starts on a pause in the visitor's speech and is discarded if they continue.
3. **The `thinks` beat**: the world client renders Socrates' absorbed stillness during the remaining gap, so a pause reads as character rather than lag.

Stage 1 does **not** build the streamed path: keeping the plan's field order (safety, move, then notebook changes) is enough to add it in stage 2 without rework (`P5`).

### 6.7 Care mode without a model

A **canned care-mode reply** exists for when no model call is possible (API down, refusal, caps with a backstop hit): a short, plain, caring message plus help resources, stored in `corpus/safety/` in **English and Bangla** to start. The language is chosen from the visitor's message: Bengali script → Bangla; **romanized or mixed Bangla, or uncertain → a bilingual Bangla + English reply**; otherwise English. The international directory is always included. More languages are added as material is reviewed.

## 7. Retrieval and the corpus

- **Texts** [P]: public-domain English translations. Plato from Project Gutenberg (Jowett) and, where a clean digital text exists, pre-1931 Loeb translations (Fowler, Lamb); Xenophon's *Memorabilia* (Marchant, Loeb 1923) for biography. Each source file records its licence and origin. Perseus texts are CC BY-SA and would need attribution and share-alike for the derived corpus; preferring Gutenberg and Wikisource avoids that.
- **Passages**: about 150–300 words each, with Stephanus (Plato) or book/chapter/section (Xenophon) references.
- **Tagging** (`§5.3`): `argument`, `life_event` (with a date), `situation_bound`, `dramatic_frame`, `always_excluded`, plus topic labels. Done **once, offline**, by Claude with the tagging rules, then **spot-checked by a human**; the always-excluded and situation-bound lists in the PRD are checked exhaustively.
- **Index**: a Lucene BM25 index built at build time and shipped in the image (a few MB), loaded at start-up. Retrieval runs twice per turn (§5 steps 4 and 7a): over the message for the plan, then with the plan's topic labels for the speaker. Embeddings are a later option (AD-11).
- **Biography bible and denylist**: versioned files in `corpus/`, used both in the persona prompt and by the evaluation checkers.

## 8. Identity, memory and privacy

| Element | Design |
|---|---|
| **Device token** | 256-bit random value in a cookie (`Secure`, `HttpOnly`, **`SameSite=Lax`**, 13-month expiry); only its SHA-256 hash is stored. **Minted only by a same-site call** (the WebSocket `hello` or an API call), **never on a page load** — so a visitor arriving from a link in WhatsApp or a search engine keeps their existing token. CSRF is covered by the `Origin` checks |
| **Storage notice** | Enforced **by the server**: no visitor message is accepted on a session that has not acknowledged the notice |
| **Consent** | Page-level "Remember me" control → `POST /api/memory/consent`; **rejected by the server** for a conversation in minors mode |
| **Secret word** | Page-level field → `POST /api/memory/secret` (set) or `/api/memory/unlock`; Argon2id (OWASP parameters, about 19 MB, 2 iterations), **at most 2 hashes computed at once** (a semaphore, so parallel requests cannot exhaust the heap); never on the WebSocket, never logged (endpoint excluded from request logging), never in the model's context. The page nudges visitors toward **two words** (at least 8 characters) |
| **Profiles** | Up to **4 per device**, each with its own secret word and memory. A visitor unlocks with **the name they gave plus their secret word**: the server finds the profile by name on that device and computes **one** Argon2 hash. Profile names are never listed. Setting a name already used on the device is refused |
| **Resuming on a shared device** | A conversation **bound to a profile** cannot be resumed, listed, or shown until that profile is unlocked in the current session. Only **anonymous** conversations (no profile) are resumable by device alone (`FR-1.5`) |
| **Unlocking mid-conversation** | The unlock request carries the **`conversationId` and a per-session nonce** (issued in `session.ready`); **only that one session and conversation** join the profile — another tab or person on the same device is unaffected. A page notice says so; that session receives `memory.status`. **Socrates does not speak on his own**: the page shows a **templated** recognition line filled with the stored name and the last open thread ("Ah, Tanvir! Last time we left courage unfinished.") — no model call — and his next reply, triggered by the visitor's next message, uses the memory (`FR-1.3`, `FR-4.4`) |
| **Locking** | Closing the page or starting a new conversation locks the profile; so does **2 hours** without activity (long enough for a pause in an hour-long conversation) |
| **Brute force and CPU** | 5 failed unlocks per device per 15 minutes; **every Argon2 endpoint is also rate-limited per IP and globally**, and when both hash slots are busy the request gets **429 immediately** (non-blocking acquire) rather than queueing; a forgotten word means starting fresh (`FR-4.8`) |
| **What is stored** | Memory items by kind (`name`, `occupation`, `thesis`, `thread`, `term`); sensitive beliefs only as examined theses (`FR-4.5`); nothing from care-mode turns. If a **minor cue** appears on an already-consented profile, **memory writes are frozen** for that profile |
| **Forget me** | `DELETE /api/memory` removes the profile, its memory and its conversations — **only after that profile is unlocked**, so one person on a shared device cannot wipe another's. A separate control, **"Delete this device's conversations"**, removes all anonymous conversations stored for the device. Deleting **cancels the profile's queued tasks in the same transaction** (§9) |
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
| `pending_task` | id, kind (memory_write, notebook_repair, condense, redact), profile_id, payload_ref, created_at, attempts — the durable queue for asynchronous work. **Deleting a profile or conversation cancels its tasks in the same transaction**, and every task **re-checks** that its profile still exists and consent still stands before acting |

Calibration labels and failure reviews live with the evaluation runner, not in production (§15).

Backups [P]: nightly `VACUUM INTO` a dated file in the volume, **keeping 3**, and only if at least 1 GB is free; **deploy snapshots: keeping the last 3**. Backups live on the same disk, so they protect against bad deploys and corruption, **not** against losing the disk; an off-box copy is open item A-9. Deleted data (forget me, care-mode text) can persist in these copies for up to 3 days or 3 deploys, which the privacy notice states.

**Scrubbing**: for care-mode turns, `turn_ops.plan_json` is reduced to the safety level only (signals and memory candidates can quote the visitor).

**Migrations**: the deploy is stop-then-start (§13.2), so only one version ever touches the database. Migrations are forward-only, applied by Flyway at start-up; Flyway is configured to **refuse to start** against a schema newer than the code knows. **Rollback is defined as restoring the snapshot taken before the deploy being rolled back, then starting the older image** — conversations from the interval are lost, explicitly. Where a change can be made backward-compatible (expand, then contract in a later release), it is, so most rollbacks lose nothing.

## 10. Protocol v1

Every **text** frame is JSON: `{"v": 1, "type": "...", ...}`. Within v1, changes are **additive only**; clients ignore unknown types and fields, and **v1 clients ignore binary frames** (reserved for stage 2 audio) (`FR-13.2`). The server only sends a kind of content a client has listed in its `hello` **capabilities**. A `lang` field is reserved on visitor and Socrates text events for stage 5 (`NFR-12`).

| Direction | Type | Payload |
|---|---|---|
| C→S | `hello` | `{ protocol: 1, capabilities: ["text"], resumeConversationId?, lastTurnId? }` (device token travels in the cookie; stage 2 clients add `"audio"`) |
| C↔S | ping / pong | WebSocket ping every **25 s** so mobile networks do not drop idle connections; the client reconnects and resumes from `lastTurnId` |
| C→S | `visitor.message` | `{ clientMsgId, text }` |
| C→S | `conversation.new` | `{}` |
| S→C | `session.ready` | `{ conversationId, sessionNonce, opening, remembered: false }` |
| S→C | `socrates.thinking` | `{ turnId }` |
| S→C | `socrates.delta` | `{ turnId, text }` |
| S→C | `socrates.beat` | `{ turnId, at, mood?, action?, tension? }` |
| S→C | `socrates.done` | `{ turnId }` |
| S→C | `socrates.notice` | `{ kind: "cap_daily" \| "cap_monthly" \| "rate_limited" \| "too_long" \| "error", text }` |
| S→C | `safety.resources` | `{ resources: [{ name, url, phone?, region }] }` |
| S→C | `memory.status` | `{ consented, unlocked, profileLabel?, recognitionLine? }` |
| S→C | `memory.forget_prompt` | `{}` — the page shows the "Forget me" confirmation (`requests.forgetMe`) |
| S→C | `debug.turn` (admin-port sessions only) | `{ turnId, plan, notebook, sources, beats, latency, usage, costUsd, cacheReadRatio }` |

Openings (`FR-1.3`) come from `corpus/openings.json`, a reviewed pool, so `session.ready` costs no model call.

## 11. Budget, rate limits, access

- **Pools** (`NFR-3`): `public`, `owner` (owner session or invite token), `eval` (separate Anthropic workspace and API key, enforced by the evaluation runner and by the workspace's own spend limit). Cost per call = usage × the price table in configuration.
- **Safety reserve**: a small separate allowance (**$2/month**) that pays for the safety-only classification, care-mode replies and ambiguous check-ins when a pool is capped. It is never blocked by the public caps; if it runs out, the canned care-mode reply is used.
- **Initial caps** [P — owner confirms before the first deploy, A-5]: public **$5/month, $0.50/day**; owner and testers **$10/month**; evaluation **$15/month** [D] (about two full runs plus one Sonnet comparison on a subset of scenarios, §14). At Haiku prices one hour-long conversation costs roughly $0.15–0.40 (§14), so the public cap allows about 15–30 hour-long conversations a month and the owner's about 30–60.
- **Real client IPs**: behind Caddy every request comes from Caddy's container address. The server trusts `X-Forwarded-For` / `X-Forwarded-Proto` **only from Caddy's address on `socrates_edge`**, so per-IP limits, `Secure` cookies and `Origin` checks see the real values and the headers cannot be spoofed.
- **Rate limits**: per device token 20 messages/minute and 300/day; per IP 60 messages/minute and 400/day, and **at most 5 new device tokens per IP per hour** — but **exceeding an IP limit triggers a Turnstile challenge, not a block**, because mobile carriers put many people behind one IP. Hard blocks apply only to device tokens and spend. Bucket4j with buckets in Caffeine caches (size-capped, with expiry). Limits reset on deploy; the persistent ledger still holds the caps.
- **Bot protection**: no model call before a human message; **Cloudflare Turnstile (free) is wired in from the start** and switches on automatically when traffic looks abnormal (e.g. new devices per hour or public spend per hour above a threshold).
- **Hard backstop**: the **production** Anthropic key lives in its own workspace with a **workspace spend limit** slightly above the sum of the app's caps, so a bug in the ledger cannot run up a bill (owner action A-8).
- **Owner session model**:
  - **Admin port** (AD-19): the owner opens an SSH tunnel (`ssh -L 8081:127.0.0.1:8081 <host>`), visits `http://localhost:8081`, and logs in with the owner password. There the owner gets the **chat client with the debug panel** (`debug.turn`), invites, digests and live-flag switch. Each port has its own `Origin` allowlist: the admin port accepts `http://localhost:8081`; the public port accepts only `https://philosopher.eliooo.org`. Admin-port conversations use the **owner pool**.
  - **Public site**: the owner and spouse use **self-issued invite tokens** (exchanged for a cookie, §12), so their AC-4 conversations on the real site use the **owner pool**, not the public $0.50/day.

## 12. Security

- HTTPS only (Caddy); **HSTS set once, in Caddy**; strict Content-Security-Policy (`self`, plus `https://challenges.cloudflare.com` for Turnstile's script and frame), set by the app; WebSocket `Origin` allowlist per port (§11).
- REST state changes require same-site cookies plus an `Origin` check.
- Visitor text only ever appears in the `user` role; the notebook and prompts never go to non-owner clients.
- **No extraction guard in the stream** (`FR-8.2`): a hold-back buffer would cost latency, and the prompts are public in the repo for now. Not revealing instructions or the notebook is handled by the persona instructions and tested by the character-breaker persona in evaluation.
- **Logs never contain message text** (visitor or Socrates), only IDs, sizes and metrics. **Heap dumps are disabled.**
- `/healthz` on the public port returns only `{ status }`. `/healthz/detail` on the admin port (sha, disk, ledger, cache) needs no login because it is reachable only from the box's loopback; the deploy script reads it at `127.0.0.1:8081`.
- **Invite tokens** are exchanged for a cookie on first use and stripped from the URL, so they never sit in logs or referrers.
- "Never reaches any API request" in `FR-4.3` means **the model API**: the secret word does travel, over HTTPS, in the page's own REST request to our server, and nowhere else.
- Secrets on the box in `<deploy dir>/socrates.env` (mode 600), the box's existing convention; CI never prints environment values.
- **Privacy notice** names Anthropic as the processor of conversation text and Cloudflare (Turnstile) when challenges are on, and states backup retention (§9).

## 13. Deployment

### 13.1 Container

| Setting | Value |
|---|---|
| Base image | Eclipse Temurin 25 JRE (minimal variant; final choice after measuring image size in S4) |
| CPU | `--cpus=0.5 --cpu-shares=512`: a t3's sustained baseline is well under one vCPU for the whole box, so half a vCPU at most, with lower priority than the tenants under contention. Costs a slightly slower JVM start |
| Hardening (AD-21) | `--read-only` root filesystem with a `tmpfs` for `/tmp`; a **non-root** `--user`; `--cap-drop=ALL`; `--security-opt no-new-privileges`; `--pids-limit=256` |
| Metadata service (AD-21) | Blocked from `socrates_edge` by a host firewall rule in the `DOCKER-USER` chain that drops traffic from that network's subnet to `169.254.169.254` (§13.3). Scoped to Socrates' network only, so other tenants are unaffected. (The alternative, an instance-wide IMDS hop limit of 1, would also cut off any tenant container that uses the instance role, so it is not used.) |
| Memory | `--memory=640m --memory-swap=640m` on the shared box (protects the other tenants); JVM `-Xmx300m`, `-XX:MaxMetaspaceSize=128m`. All values are deploy-time configuration, raised if the owner adds memory or moves the service |
| Network | `socrates_edge` only (AD-18); the public port 8080 is reachable only from Caddy; the admin port 8081 is published on **127.0.0.1 only** (AD-19). Caddy's own admin API must listen on its container's localhost only (verified at set-up; details in the private operations file), so Socrates cannot reconfigure the proxy |
| Data | Docker volume `socrates_data` → `/data` (SQLite and backups) |
| Logs | `json-file` driver, `max-size=10m`, `max-file=3` (30 MB ceiling on a nearly full disk) |
| Restart | `unless-stopped`; stopped with `docker stop -t 30` and Spring graceful shutdown |
| JVM | heap dumps off; container-aware memory settings |
| Health | public `GET /healthz` → `{ status }`; admin `GET /healthz/detail` → `{ service, sha, build, disk, ledger, cache }` |

### 13.2 Pipeline

1. **On every push to `main`**: unit and contract tests, the **deterministic** evaluation checks (no model calls), image build, push to GHCR as `socrates-mind:<sha>`. **Every deploy is a release** for `FR-11.6`. **Path-gated behaviour check**: if the change touches prompts, persona, routing, the plan schema or `corpus/safety/`, the **AI-judged safety and method smoke subset** (a few cents on Haiku, eval pool) runs **before deploy and blocks it** on regression; each scenario runs twice and fails only if both runs fail, to absorb run-to-run variation. Other changes are gated by the deterministic checks only. Work happens on branches; `main` receives merged, reviewed changes, so "every push to main" means every merged increment (`P9`).
2. **Deploy job** (SSH to the box) runs in a **protected GitHub Environment limited to `main`**, with every Action **pinned to a commit sha** and **no `pull_request_target` workflows**. Its key is restricted in `authorized_keys` (`restrict,command="…/deploy-socrates.sh"`); the script reads the sha from `SSH_ORIGINAL_COMMAND`, accepts **only 40 hex characters**, and does nothing else (AD-20). The script:
   - refuses to run unless at least **1 GB** is free on `/` after pruning;
   - prunes **only** `socrates-mind` tags, keeping the newest 2 and never the running or target image;
   - pulls the new image **while the old container keeps serving**;
   - takes a database snapshot (`VACUUM INTO`), keeping the last 3;
   - ensures Caddy is attached to `socrates_edge` (idempotent `docker network connect`, which does not restart Caddy; needed because recreating the proxy's compose stack would drop a manually attached network);
   - **stop-then-start**: stops the old container, starts the new one, waits for `127.0.0.1:8081/healthz/detail` to report the target sha. **Two Socrates containers never run at once**, so peak memory on the shared box never exceeds one container's limit, and only one version ever opens the database. Expected downtime: about 10–20 seconds per deploy (JVM start and model load); open conversations reconnect automatically and resume;
   - on a failed health check: stops the new container, restores the snapshot, starts the previous image again.
3. **Rollback**: run by the owner on the box (not by CI): restore the snapshot taken before the deploy being rolled back, then start the older image (§9).
4. **Full evaluation suite**: manual workflow (`workflow_dispatch`), never automatic.
5. **Evaluation in CI**: the **deterministic** checks (denylist, leakage, retrieval filters, beat vocabulary, memory isolation) run on every push and **block** the deploy. **AI-judged** checks never run per push: a small AI-judged smoke set runs **weekly or by hand**, and only **reports**, because model output varies from run to run.
6. **Secrets in GitHub** [P]: the restricted deploy key, the host, the **evaluation** Anthropic API key (its own workspace with its own spend limit) and the judge key. **The production API key never leaves the box.** GitHub masks secrets in logs and does not expose them to pull requests from forks.
7. Every release produces **release notes and a user-perspective test script** (`NFR-11`) in `docs/releases/`.

### 13.3 One-time set-up (each step needs the owner's approval when it is reached)

1. **DNS** (owner, at the DNS provider): A record `philosopher` → the host's public IP.
2. **Server folder**: `<deploy dir>` with `deploy-socrates.sh` and `socrates.env` (mode 600).
2a. **Deploy key**: one new line in the deploy user's `authorized_keys` with `restrict,command="<deploy dir>/deploy-socrates.sh"` (AD-20). Optionally a dedicated OS user for it (needs `sudo`; owner's choice).
2b. **Network**: `docker network create socrates_edge`, then `docker network connect socrates_edge <proxy container>` — attaches a network to the running Caddy without restarting it; other sites are unaffected. To make it survive a re-creation of the proxy's compose stack, the owner may later add `socrates_edge` as an external network in that compose file; until then the deploy script re-attaches it (§13.2).
2c. **Metadata block** (AD-21): one `DOCKER-USER` iptables rule dropping traffic from the `socrates_edge` subnet to `169.254.169.254`, made persistent across reboots. Affects only Socrates' network.
3. **Caddy** — add exactly one site block to the shared Caddyfile, after backing the file up:

   ```
   philosopher.eliooo.org {
   	encode zstd gzip
   	header {
   		Strict-Transport-Security "max-age=31536000"
   		X-Content-Type-Options "nosniff"
   		Referrer-Policy "strict-origin-when-cross-origin"
   		-Server
   	}
   	reverse_proxy socrates-mind:8080
   }
   ```

   Then `caddy validate` inside the Caddy container, and only if valid, `caddy reload` (graceful: other sites are not interrupted). Another tenant's CI drift check compares only its own blocks, so it is unaffected. Caddy obtains the TLS certificate automatically once DNS resolves.
4. **GitHub secrets**: deploy SSH key and host; GHCR uses the workflow token.

## 14. Cost model

Assumptions [P]: an hour-long conversation is about 80 turns; each turn is one plan call and one speak call; the cached system prefix is about 8K tokens; history grows to about 20K tokens by the end (about 10K on average); the context packet is about 3K tokens; plan output about 600 tokens including thinking; speak output about 200 tokens.

| Model (per MTok, input / output; cache read about 0.1× input) | One hour-long conversation |
|---|---|
| **Claude Haiku 5.5** ($0.10 / $0.50, under 100K-token requests) | **about $0.15–0.40** |
| Claude Sonnet 5.5 ($2 / $10) | about $2–3 |
| Claude Opus 5.5 ($4 / $20) | about $5–6 |

A full evaluation run (about 170 conversations of about 20 turns plus 10 runs of 50 turns, with a simulated visitor on Haiku) is estimated at **$3–8** on Haiku; the judge runs on another provider's free tier. Within the **$15/month** evaluation budget: about two full runs plus one Sonnet comparison on a subset (about 20 conversations), with deterministic checks free on every push.

**The public cap and abuse**: at roughly $0.0015 per turn, an unthrottled script could empty $0.50/day in minutes; the per-IP, new-device and Turnstile controls (§11) exist for this, and the workspace spend limit is the final backstop. Every figure is re-measured from real usage in the first builds.

## 15. Evaluation architecture

- **Runner** (`eval/`): runs the mind **in-process** (same code as production) against a **simulated visitor** (Haiku 5.5 with persona prompts: cooperative, stubborn, evasive, confused, hostile, playful, plays-along, character-breaker, apparent minor, non-English, **romanized or mixed Bangla**).
- **Scenarios and sizes**: as `FR-11.2` and `FR-11.6a`, stored as versioned files.
- **Server-level scenarios**: identity, memory, routing state, caps, care-mode stickiness and every secret-word path live in `mind-server`, so those categories (shared devices, memory isolation, secret-word leakage, caps with safety) run **against the real server** (Testcontainers) through REST and the WebSocket — not in-process.
- **Deterministic checkers** (Java): denylist phrases, modern-vocabulary list (anachronism), beat vocabulary validity, secret-word leakage (the word never appears anywhere), memory isolation between profiles, retrieval filter (no excluded passage ever retrieved).
- **AI judge**: another provider's free tier via a small adapter (`Judge` interface), **only on simulated conversations**. Rubric per `FR-11.3`, grouped into five criteria (`FR-11.5`). **One judge call per conversation** scores every turn at once (thousands of per-turn calls would exceed free-tier limits). The **judge model version is pinned**; changing it requires recalibration. A **fallback** judge (another free tier, or Claude Sonnet on the eval budget, marked as same-family) is ready if the free tier changes.
- **Live flags** (`FR-11.4`, `FR-12.4`): **advisory only — never used to gate a release**, since they use a same-family model. An owner-only switch in the debug panel, **off by default**. When on, each of the owner's own turns gets an asynchronous Claude Haiku 5.5 judgement, shown as flags; it runs only on owner and tester sessions, never on public visitors, and is paid from the owner pool. Built last in stage 1.
- **Quote checker**: extracts attributed claims and matches them against corpus, biography bible and denylist; calibrated like the judge.
- **Calibration** — a **local labelling tool** that runs next to the evaluation runner on the owner's laptop (`./gradlew :eval:label`, a small local web page), not on production: no upload of simulated transcripts, no second production login. The owner and spouse label on the same machine, each under their own labeller name. It presents balanced items (about 40 per criterion, half seeded failures), computes κ **first between the two human labellers** (if they disagree with each other, the criterion is clarified before any judge is calibrated), then between the humans and the judge; the judge is trusted only at κ ≥ 0.6. Labels are stored as versioned files in `eval/labels/`.
- **Human adjudication** — every zero-tolerance failure goes to a review queue in the same local tool before it counts.
- **Reports**: Markdown summary plus per-conversation transcripts as CI artefacts; trend of pass rates per release.

## 16. Development approach

- **Test-first** for `mind-core` and the server: unit tests; contract tests for the protocol; integration tests with a **recorded-response fake** of `LanguageModel` (no API cost); evaluation for behaviour.
- **Local run**: `./gradlew :mind-server:bootRun` with a local `.env`; or the same Docker image as production.
- **Release 0 — the walking skeleton with its protections** (`P9`, `NFR-11`, PRD §4): the first public deploy is end-to-end on `philosopher.eliooo.org` — a page, a WebSocket, a single Haiku call answering in a minimal Socrates voice — **plus**, from that first deploy: the crisis backstop and the canned care-mode reply; the production workspace spend limit (A-8) and a hard daily cap; per-device and per-IP rate limits and the `Origin` check; container hardening and the metadata block; and either the storage notice or no storage at all. Everything else deepens from there.

### Spikes to run first (each a day or less)

| # | Question | Decides |
|---|---|---|
| S0 | **Verify the API facts this design rests on** against current Anthropic documentation and real calls: Haiku 5.5 prices and the 100K price tier, structured outputs on Haiku 5.5, rejection of sampling parameters and prefill, mid-conversation system messages and `clear_at`, refusal stop reasons, cache minimums | AD-8, AD-10, §6.1 |
| S1 | Can Haiku 5.5 produce the plan JSON reliably and speak in a convincing Socratic voice? Speak with thinking off vs effort `low` | Model per role; speak settings |
| S2 | Do mid-conversation system messages with `clear_at` work on Haiku 5.5 with our caching layout? | Context-packet placement |
| S3 | Plain-text history with no replayed thinking: any API errors or quality loss? | AD-8 |
| S4 | Memory and CPU footprint of the container under load (Lucene, Argon2, streaming) and the **real image size** | The 640 MB and 1-CPU limits; disk planning |
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
| A-1 | Add the DNS A record `philosopher` → the host's public IP at the DNS provider | First deploy |
| A-2 | Approve the one-time Caddy change (§13.3) when it is reached | First deploy |
| A-3 | A free API key from another provider for the judge (e.g. Google AI Studio for Gemini) | Evaluation suite |
| A-4 | A separate Anthropic workspace and API key for evaluation (recommended, keeps the eval pool separate) | Evaluation suite |
| A-5 | Confirm initial budget caps (§11) | First deploy (release 0) |
| A-8 | Put the **production** Anthropic key in its own workspace with a **spend limit** (§11 hard backstop) | First deploy (release 0) |
| A-9 | **Off-box backup**: accept that backups live on the same disk (default for stage 1), or approve an encrypted off-box copy | First public release |
| A-10 | Check the instance's **CPU credit mode** (AWS console: *standard* or *unlimited*); unlimited mode can add charges when credits run out | First deploy |
| A-11 | Sign up for a **free uptime monitor** and set one CloudWatch alarm on CPU credit balance (§19) | First deploy (release 0) |
| A-13 | Approve the **metadata-service block** rule on the host firewall (§13.3 step 2c) | First deploy (release 0) |
| ~~A-12~~ | Public repo vs operational details — **resolved**: the repo stays public; operational specifics live in a private file outside the repo and public documents use placeholders. Details published in earlier commits remain in the history (accepted) | — |
| ~~A-6~~ | Evaluation budget — **resolved**: $15/month (§11) | — |
| ~~A-7~~ | Public repo — **resolved**: stays public for now, including prompts; the owner may make it private later (§1) | — |

## 19. Monitoring

All free:

- **Uptime**: an external free monitor (e.g. UptimeRobot) checks `https://philosopher.eliooo.org/healthz` every 5 minutes and e-mails the owner. The public health returns `degraded` (still HTTP 200, with a keyword the monitor watches) when an internal check fails, so one monitor covers several conditions.
- **Internal checks** (run by the app every few minutes, shown on the admin port, reflected in `degraded`): **disk usage above 85%** of the root filesystem; **public spend per hour** far above normal; **cache-read ratio** collapsing (cost would rise); **safety events** count; container restarts (crash loops).
- **CPU credits**: one CloudWatch alarm on the instance's `CPUCreditBalance` (owner action A-11).
- **Daily digest** on the admin port: turns, spend per pool, cache ratio, latency p50/p95, safety events, disk.

## Appendix A — PRD traceability

| PRD | Where implemented |
|---|---|
| FR-1 Conversation | §5, §10 (`session.ready`, openings pool), `mind-server` |
| FR-2 Examination | §6.2 notebook phases and move; persona prompt |
| FR-3 Notebook | §5 steps 5–6, §6.2, `turn_ops` |
| FR-4 Memory and identity | §8, §9, REST `/api/memory/*`. **Clarification of `FR-4.6`**: asking Socrates to forget you brings up the page-level confirmation, which then deletes — it does not delete from chat text alone (consent and deletion are page-level, like `FR-4.2`) |
| FR-5 Beats | §6.3, protocol `socrates.beat` |
| FR-6 Grounding | §7, AD-11, AD-12 |
| FR-7 Infinite conversation | §6.5, AD-9, AD-10 |
| FR-8 Robustness | Persona prompt; character-breaker persona in evaluation (§12, §15) |
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
| 1 | Safety when the normal turn cannot run | Fixed: backstop before caps; safety reserve; canned care mode without a model; `safety` validated on its own. *(Superseded in part: refusals now take the ambiguous route — Appendix C #20, D #14)* | §5, §6.7, §11 |
| 2 | Two containers during deploy; shared SQLite; migrations | Fixed: stop-then-start (10–20 s downtime), snapshot before deploy, forward-only migrations | §9, §13.2 |
| 3 | Evaluation spending | Owner decision: $15/month | §11 |
| 4 | API keys in GitHub; smoke evaluation blocking | Fixed: eval and judge keys as GitHub secrets, production key stays on the box; deterministic checks block, AI-judged checks report | §13.2 |
| 5 | Shared-device resume and unlocking | Fixed: profile conversations hidden until unlocked; unlock joins the current conversation; no unprompted model call; REST unlock notifies the WebSocket | §8, §10 |
| 6 | Concession definition; notebook size | Fixed: concession = premise quote + visitor reply copied by the server; model sends notebook operations only | §6.2 |
| 7 | Where labelling and review happen | Fixed: local tool next to the evaluation runner | §9, §15 |
| 8 | Retention of care-mode text | Fixed: deleted after 30 days; excluded from resume, condensation and memory | §6.5, §8, §9 |
| 9 | Stage 2's 2-second target | Fixed: streamed plan with early speak, planning during speech, `thinks` beat. *(Superseded in part: the streamed path is not built in stage 1 — Appendix C #22)* | §6.2, §6.6 |
| 10 | "Forget me" in chat; live judge flags; input moderation | Fixed: page-level confirmation; owner-only live flags, off by default; input hygiene plus the plan's safety assessment | §5, §6.2, §15 |
| 11 | Public repo exposes prompts; extraction guard vs streaming; meaning of "API request" | Owner decision: public for now, private later possible; "API request" clarified as the model API. *(Superseded in part: the hold-back guard was dropped — Appendix C #21)* | §1, §12 |

## Appendix C — External review (of v0.2) → v0.4

| # | Item | Disposition | Where |
|---|---|---|---|
| 1 | CI deploy key = root on a shared box | Fixed: forced-command key, sha-only input, pinned Actions, protected environment, no `pull_request_target` | AD-20, §13.2, §13.3 |
| 2 | Public LLM app on the tenants' network | Fixed: own `socrates_edge` network with Caddy only | AD-18, §13.3 |
| 3 | Public repo leaks reconnaissance data | Owner decision: keep public; specifics moved to a private operations file; placeholders in public docs; earlier history accepted | A-12 |
| 4 | No CPU limit on a burstable instance | Fixed: `--cpus=1.0`, lower CPU shares, credit alarm | §13.1, §19, A-10 |
| 5 | Disk exhaustion | Fixed: retention caps, free-space checks before backups, real image size measured, 85% alert; BM25 instead of ONNX shrinks the image | §9, §13.2, §19 |
| 6 | Rollback vs newer schema | Fixed: rollback = restore pre-deploy snapshot (data loss explicit); expand/contract where possible; Flyway refuses a newer schema | §9, §13.2 |
| 7 | Stop-then-start loses work | Fixed: graceful shutdown, `stop -t 30`, durable task queue, `clientMsgId` de-duplication | §5, §9, §13.1 |
| 8 | Every visitor has Caddy's IP | Fixed: forwarded headers trusted only from Caddy | §11 |
| 9 | Public cap drained in minutes | Fixed: per-IP daily cap, new-device limit, Turnstile wired with auto-switch, workspace spend limit | §11, A-8 |
| 10 | Unlock can merge the wrong conversation | Fixed: `conversationId` + session nonce | §8, §10 |
| 11 | Two turns at once | Fixed: one turn in flight per conversation | §5 |
| 12 | Virtual threads pinned by SQLite | Fixed: Java 25; database on a platform-thread pool; single writer | AD-1, AD-4 |
| 13 | Deleted data survives | Fixed: no text in logs, no heap dumps, `plan_json` scrubbed, backup retention capped and disclosed, Anthropic named as processor; off-box copy is an owner decision | §9, §12, A-9 |
| 14 | Admin panel on the public internet | Fixed: admin port on 127.0.0.1, SSH tunnel | AD-19, §11 |
| 15 | Evaluation budget contradiction | Fixed: $15/month (v0.3); AI-judged checks weekly or manual, never per push | §13.2, §14 |
| 16 | Free-tier judge throughput | Fixed: per-conversation judging, pinned model, fallback, human–human κ first | §15 |
| 17 | Romanized Bangla | Fixed: backstop phrases, bilingual canned reply, persona, script detection | §6.4, §6.7, §15 |
| 18 | ONNX is heavy for a small corpus | Fixed: Lucene BM25 + plan topic labels; embeddings only if evaluation shows misses | AD-11, §7 |
| 19 | Exact-substring quotes; turn numbers | Fixed: normalised/fuzzy match, server stores exact span; turn numbers in history | §6.2 |
| 20 | Refusal treated as crisis | Fixed: refusal → ambiguous unless backstop or safety says crisis | §5 |
| 21 | Hold-back extraction guard | Fixed: dropped; persona + evaluation instead | §12 |
| 22 | Streamed plan built in stage 1 | Fixed: not built; field order kept | §6.6 |
| 23 | No monitoring | Fixed: free uptime monitor with degraded keyword, internal checks, credit alarm, digest | §19 |
| 24 | Ledger accuracy | Fixed: reserve before call, settle after, estimate on abort | §5 |
| 25 | Argon2 and unbounded maps | Fixed: semaphore of 2; Caffeine caches with caps and expiry | §8, §11 |
| 26 | WebSocket keepalive | Fixed: 25 s ping, resume with `lastTurnId` | §10 |
| 27 | Live flags same-family | Fixed: advisory only, never gating | §15 |
| 28 | Context-packet fallback details | Fixed | §6.1 |
| 29 | Unverified API facts | Fixed: spike S0 before S1 | §16 |
| 30 | HSTS twice; public sha | Fixed: HSTS in Caddy only; public health returns status only | §12, §13.1 |
| 31 | Weak secret word | Fixed: nudge to two words, minimum 8 characters | §8 |
| 32 | Invite tokens in URLs | Fixed: exchanged for a cookie and stripped | §12 |
| 33 | Minor cue on existing profile | Fixed: memory writes frozen | §8 |

## Appendix D — Third review (of v0.5 against PRD v1.0) → v0.6

| # | Item | Disposition | Where |
|---|---|---|---|
| 1 | Skeleton public without protections | Fixed: release 0 includes backstop, canned care mode, spend limit, daily cap, rate limits, Origin check, hardening, notice; A-5/A-8/A-11 due at first deploy | §16, §18 |
| 2 | Safety degrades when capped; care mode not sticky | Fixed: care mode bypasses caps; safety-only classification before any cap notice; reserve $2 | §5, §11 |
| 3 | Route precedence | Fixed: one precedence table, one function, unit-tested | §5, §6.4 |
| 4 | Prompt changes deploy without behaviour check | Fixed: path-gated AI safety and method subset blocks deploy; every deploy is a release | §13.2 |
| 5 | Owner/debug redesign broken | Fixed: chat served on the admin port with debug panel; per-port Origin allowlists; password on the admin port; invite tokens for owner and spouse on the public site | AD-19, §10, §11 |
| 6 | Compromised container not contained | Fixed: hardening flags; metadata service blocked for Socrates' network only; Caddy admin verified localhost-only | AD-21, §13.1, §13.3, A-13 |
| 7 | `SameSite=Strict` cookie erases memory | Fixed: `Lax`; token minted only by same-site calls | §8 |
| 8 | IP limits vs carrier-grade NAT | Fixed: IP limits trigger Turnstile, not blocks | §11 |
| 9 | Retrieval before the plan that labels it | Fixed: two-stage retrieval | §5, §7 |
| 10 | Queued tasks resurrect deleted data | Fixed: cancel in the same transaction; re-check before acting | §8, §9 |
| 11 | "Forget me" gaps | Fixed: device anonymous delete; profile delete after unlock; FR-4.6 clarification recorded | §8, Appendix A |
| 12 | In-process eval misses server code | Fixed: server-level scenarios via Testcontainers | §15 |
| 13 | CPU credits; Argon2 floods | Fixed: `--cpus=0.5`; Argon2 rate limits per IP and global; immediate 429 | §8, §13.1 |
| 14 | Refusal handling specified twice | Fixed: one definition; speak refusal → pre-written check-in, no new call | §5, §6.1 |
| 15 | No slot for stage 2 audio | Fixed: binary frames ignored by v1; capabilities in `hello` | §10 |
| 16 | Stale appendices | Fixed: Appendix A updated; superseded rows marked in Appendix B | Appendices |
| 17 | Health-check contradiction | Fixed: deploy reads `/healthz/detail` on loopback | §12, §13.2 |
| 18 | Condensation never exercised | Fixed: configurable threshold; eval at about 8K | §6.5 |
| 19 | Multi-profile unlock | Fixed: name + word, one hash; max 4 profiles; duplicate names refused; 2-hour idle lock | §8 |
| 20 | Romanized Bangla under-specified | Fixed: normalised fuzzy backstop; schema fields | §6.2, §6.4 |
| 21 | Recognition line not contextual | Fixed: templated with name and last thread | §8 |
| 22 | CSP blocks Turnstile | Fixed: Cloudflare origin allowed; named in the notice | §12 |
| 23 | Two calls per turn | Kept, with a documented single-call fallback if S5 misses p95 | AD-7 |
| 24 | Server-side enforcement | Fixed: notice and minors-consent enforced by the server | §8 |
