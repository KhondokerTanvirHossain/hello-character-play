# Stage 1 — Release Roadmap

| | |
|---|---|
| **Version** | 0.2 (plan review incorporated: safety and privacy ship with capability; R1 split) |
| **Date** | 2026-10-23 |
| **Implements** | [PRD v1.0](../specs/2026-10-22-socrates-prd.md), [Architecture v1.1](../specs/2026-10-23-socrates-architecture.md) |
| **Status** | Awaiting validation |

## How stage 1 is delivered

- Stage 1 ships as **nine releases: R0, R1a, R1b, R2–R7**. Every release is deployed to `https://philosopher.eliooo.org` and ends with **release notes and a user-perspective test script** in `docs/releases/` (PRD `P9`, `NFR-11`).
- **Each release gets its own detailed implementation plan** in `docs/superpowers/plans/`, written when the release starts, so that what earlier releases and spikes taught us shapes it. Only R0's plan exists now.
- **Gate between releases:** the owner runs the release's test script on the live site and approves before the next release starts.
- **Rule: no capability ships ahead of its safety and privacy.** Whatever a release makes possible (storing text, examining people, remembering them) ships with the protections for it in the same release.
- **Informal hour-long sessions** by the owner and spouse start at R1b and continue every release; AC-4 at R7 is the formal one, not the first.
- Development is **test-first**. Every change reaches production through `main`, the CI checks, and the restricted deploy (Architecture §13.2).

## Releases

### R0 — Walking skeleton with its protections

**Goal:** a real, public, end-to-end Socrates, minimal but safe from the first second.

- Gradle multi-project with a packaged corpus; CI; hardened GHCR image; restricted, guarded deploy; one-time server set-up (owner-approved).
- **Spike S0** (API facts) and a first **S4** footprint check.
- Chat page → signed device cookie → WebSocket → one streamed Haiku 5.5 reply with persona v0 (which carries detection-independent safety rules).
- **Layered safety**: crisis backstop (English, Bangla, romanized Bangla) first on every message; Haiku safety classifier on every message, concurrent with buffered speech; sticky care mode; canned care and check-in replies; safety reserve.
- Budget ledger with daily and monthly caps; workspace spend limit; rate limits; `Origin` checks; trusted client IPs; one turn in flight; reconnect without loss. **No conversation storage.**

**Covers:** FR-1.1, FR-1.2, FR-1.4, FR-10 (backstop, classifier, sticky care, canned replies, every language), FR-13 (subset), NFR-2, NFR-3 (public pool and safety reserve), NFR-4, NFR-5, NFR-6 (partial, Architecture Appendix E), NFR-10, NFR-11, AC-5.
**Exit:** the owner's R0 test script passes on the live site; S0 and S4 findings recorded.

### R1a — Storage with its privacy, the owner's tools

**Goal:** conversations are kept, safely, and the owner can see inside.

- Conversation storage (SQLite) with the storage notice enforced by the server; resuming anonymous conversations.
- **Privacy controls in the same release**: "Delete this device's conversations"; 30-day deletion of care-mode text; care-mode turns never stored in plan logs; retention jobs; `pending_task` queue with cancellation; backups and deploy snapshots with retention caps.
- Admin port with password: the chat client plus the **debug panel** (turn, timing, cost, cache, safety verdicts).
- **Owner pool and invite tokens**, so the owner's and spouse's testing never uses the public budget.
- Persona v1: convictions, behaviour rules and the convictions/answers line, modesty about his feats, anachronism and future-revelation rules, content policy, biography bible (core), misquote denylist.
- Spike **S4** load test at realistic concurrency.

**Covers:** FR-1.5, FR-4.6 (device part), FR-8, FR-12.4, NFR-3 (all pools), NFR-7, NFR-8, PRD §5.
**Exit:** owner verifies deletion and retention on the live site; owner and spouse test via invites.

### R1b — The Socratic examination, with its safety routing

**Goal:** Socrates genuinely examines; safety routing is complete before he does.

- **Spikes S1, S2, S3, S5**.
- Plan call (structured output) → validation → notebook operations (verbatim concessions, normalised quote matching) → **route precedence** (care > ambiguous > minors > language notice > normal) → speak call with beats; refusal handling.
- **Core safety moves here from R5**: plan-based safety routing, the ambiguous check-in in the visitor's language, minors mode, identification-with-his-fate rule, care-mode flags on turns (so FR-4.5 can rely on them later).
- **Route-assertion smoke test** in CI (live Haiku calls on crisis, ambiguous, minor and non-English probes; asserts the route; no judge needed), **path-gated** to changes in prompts, persona, routing, plan schema and `corpus/safety/` — blocking deploy.
- Debug panel shows the notebook, plan and beats. Informal hour-long sessions begin.

**Covers:** FR-2, FR-3, FR-5, FR-10 (routing), FR-13, NFR-1, PRD §5.7–5.9.
**Exit:** a real elenchus visible in the debug panel on at least three topics; the route-assertion smoke passes; latency measured.

### R2 — The evaluation harness

**Goal:** "is this Socrates?" becomes measurable.

- Simulated visitors (all personas, including romanized/mixed Bangla), scenario files, deterministic checkers, the judge adapter (free tier; owner key A-3), the quote checker, per-conversation judging.
- Local labelling tool; **first calibration round** (owner and spouse, about 3–4 hours; human–human κ first, then judge).
- Server-level scenarios with Testcontainers.
- CI: the AI-judged safety and method subset joins the path-gated pre-deploy check; eval workspace key (A-4).

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

- Device-scoped profiles: consent control (server-enforced, refused in minors mode), name + secret word (Argon2id with limits), unlock nonce, templated recognition line, locking, "forget me" (after unlock), minors freeze; sensitive beliefs only as theses; nothing from care-mode turns.

**Covers:** FR-4, FR-9.6.
**Exit:** shared-device and memory-isolation scenarios pass at zero leaks; the owner returns after a day and is remembered.

### R5 — Abuse protection and content tuning

**Goal:** the public site is hard to abuse, and the content policy holds under pressure.

- Turnstile wired; IP limits converted from blocks to challenges (ending the R0 deviation); automatic switch-on.
- Content-policy and personal-topic-pressure tuning against the evaluation suite (§5.7, §5.8).

**Covers:** NFR-6 (complete), PRD §5.7–5.8 tuning.
**Exit:** 100% of safety scenarios and 0 content-policy violations (human-adjudicated).

### R6 — Engagement and endurance

**Goal:** an hour passes easily.

- Engagement tuning (FR-9): hooks, rhythm, sincere vs ironic praise, continuity across visits.
- Condensation (configurable threshold, evaluation at about 8K), drift tests, long runs.
- Monitoring (Architecture §19): uptime keyword, internal checks, digest; live flags (advisory).
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
| R0 | A-1 DNS record; A-5 confirm caps and safety reserve; A-8 production workspace + spend limit; A-10 CPU credit mode; A-11 uptime monitor and credit alarm; A-13 approve the metadata-service block; **A-14 development workspace key** for the laptop; review the Bangla crisis phrases and replies; approve each one-time server step |
| R1a | Invite your spouse; check deletion on the live site |
| R1b | Start informal hour-long sessions |
| R2 | A-3 judge key; A-4 eval workspace and key ($15/month limit); first calibration round |
| R3 | Spot-check passage tags |
| R4 | Test being remembered on two devices |
| R7 | The one-hour test |
| Before public launch | A-9 off-box backup decision |
