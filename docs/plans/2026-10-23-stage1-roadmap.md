# Stage 1 — Release Roadmap

| | |
|---|---|
| **Version** | 0.1 (draft for validation) |
| **Date** | 2026-10-23 |
| **Implements** | [PRD v1.0](../specs/2026-10-22-socrates-prd.md), [Architecture v1.0](../specs/2026-10-23-socrates-architecture.md) |
| **Status** | Awaiting validation |

## How stage 1 is delivered

- Stage 1 ships as **eight releases, R0–R7**. Every release is deployed to `https://philosopher.eliooo.org` and ends with **release notes and a user-perspective test script** in `docs/releases/` (PRD `P9`, `NFR-11`).
- **Each release gets its own detailed implementation plan** in `docs/superpowers/plans/`, written when the release starts, so that what earlier releases and spikes taught us shapes it. Only R0's plan exists now.
- **Gate between releases:** the owner runs the release's test script on the live site and approves before the next release starts.
- Development is **test-first**. Every change reaches production through `main`, the CI checks, and the restricted deploy (Architecture §13.2).

## Releases

### R0 — Walking skeleton with its protections

**Goal:** a real, public, end-to-end Socrates, minimal but safe from the first second.

- Gradle multi-project; CI; GHCR image; hardened container; restricted deploy; one-time server set-up (owner-approved).
- **Spike S0**: verify the API facts the design rests on.
- Chat page → `POST /api/session` → WebSocket → one streamed Haiku 5.5 call with a minimal Socratic persona.
- Protections: crisis backstop (English, Bangla, romanized Bangla) with the canned care reply; budget ledger with daily and monthly caps; workspace spend limit; per-device and per-IP rate limits; `Origin` checks; forwarded-header trust; one turn in flight; **no conversation storage** (the page says so).
- Public `/healthz` and the admin-port `/healthz/detail`.

**Covers:** FR-1.1, FR-1.2, FR-1.4, FR-10 (backstop part), FR-13 (subset), NFR-2, NFR-3 (one pool), NFR-4, NFR-5, NFR-6 (partial), NFR-11, AC-5.
**Exit:** the owner's R0 test script passes on the live site; S0 findings recorded.

### R1 — The Socratic mind

**Goal:** Socrates genuinely examines; you can watch him think.

- **Spikes S1, S2, S3, S5**: Haiku plan quality and voice, context-packet placement, plain-text history, latency.
- Persona v1: convictions, behaviour rules and the convictions/answers line, modesty about his feats, anachronism and future-revelation rules, content policy, biography bible (core), misquote denylist.
- Plan call (structured output) → validation → notebook operations (verbatim concessions, normalised quote matching) → route precedence → speak call with beats.
- Conversation storage (SQLite), the storage notice enforced by the server, resuming anonymous conversations, the openings pool.
- Admin port with password: the chat client plus the **debug panel** (notebook, plan, beats, latency, cost, cache).
- Refusal handling; failure paths.

**Covers:** FR-1, FR-2, FR-3, FR-5, FR-8, FR-12 (most), FR-13, NFR-1, NFR-8, NFR-10.
**Exit:** owner sees a real elenchus in the debug panel on at least three topics; S1–S5 findings recorded; latency measured.

### R2 — The evaluation harness

**Goal:** "is this Socrates?" becomes measurable.

- Simulated visitors (all personas, including romanized/mixed Bangla), scenario files, deterministic checkers, the judge adapter (free tier; owner key A-3), the quote checker, per-conversation judging.
- Local labelling tool; **first calibration round** (owner and spouse, about 3–4 hours; human–human κ first, then judge).
- Server-level scenarios with Testcontainers.
- CI: deterministic checks on every push; path-gated AI safety and method subset before deploy; eval workspace key (A-4).

**Covers:** FR-11, P6.
**Exit:** judge trusted (κ ≥ 0.6) on the five criteria, or the failing criteria clarified; baseline report published.

### R3 — Grounding in the sources

**Goal:** he argues with his real arguments and never invents his life.

- Corpus acquisition (public-domain texts with licences), passage splitting with references, **tagging** (argument, life event with date, situation-bound, dramatic frame, always-excluded) with human spot-check.
- Lucene BM25 two-stage retrieval; sources in the debug panel; full biography bible.

**Covers:** FR-6, PRD §5.1–5.3, AD-11, AD-12.
**Exit:** zero excluded passages retrieved (deterministic check); fidelity criterion passing in evaluation.

### R4 — Memory and identity

**Goal:** he remembers you, safely.

- Device-scoped profiles: consent control, name + secret word (Argon2id, limits), unlock nonce, templated recognition line, locking, "forget me" and device deletion, minors freeze.
- Budget pools: owner pool via admin port; **invite tokens** for owner and spouse on the public site.
- Retention jobs, `pending_task` queue with cancellation, backups and deploy snapshots with retention caps.

**Covers:** FR-4, FR-9.6, NFR-3 (all pools), NFR-7.
**Exit:** shared-device and memory-isolation scenarios pass at zero leaks; the owner returns after a day and is remembered.

### R5 — Safety and abuse, complete

**Goal:** the public site is safe for anyone and hard to abuse.

- Plan-based safety routes, sticky care mode, safety-only classification before cap notices, safety reserve, care replies in the visitor's language, `safety_event` retention, always-excluded death passages verified, identification-with-his-fate rule, minors mode.
- Content policy (§5.7) and personal-topic pressure (§5.8).
- Turnstile wired, with IP limits converted from blocks to challenges; automatic switch-on.

**Covers:** FR-10, PRD §5.7–5.9, NFR-6.
**Exit:** 100% of safety scenarios and 0 content-policy violations (human-adjudicated).

### R6 — Engagement and endurance

**Goal:** an hour passes easily.

- Engagement tuning (FR-9): hooks, rhythm, sincere vs ironic praise, continuity across visits.
- Condensation (configurable threshold, evaluation at about 8K), drift tests, long runs.
- Monitoring (§19): uptime keyword, internal checks, digest; live flags (advisory).
- Model-per-role decision from evidence (`P8`, `NFR-9`).

**Covers:** FR-7, FR-9, NFR-1 tuning, Architecture §19.
**Exit:** drift within limits; engagement criterion passing; monitoring alerts tested.

### R7 — Acceptance

**Goal:** stage 1 is done.

- Full evaluation suite (AC-2), latency over 100+ turns (AC-3), **the one-hour test by the owner and spouse (AC-4)**, optional outside testers.

**Exit:** PRD §13 acceptance criteria met; stage 1 signed off.

## Owner actions by release

| Release | Owner action |
|---|---|
| R0 | A-1 DNS record; A-5 confirm caps; A-8 production workspace + spend limit; A-10 CPU credit mode; A-11 uptime monitor and credit alarm; A-13 approve the metadata-service block; approve each one-time server step |
| R2 | A-3 judge key; A-4 eval workspace and key ($15/month limit); first calibration round |
| R3 | Spot-check passage tags; review the Bangla crisis phrases (from R0) again |
| R4 | Test being remembered on two devices; invite your spouse |
| R7 | The one-hour test |
| Before public launch | A-9 off-box backup decision |
