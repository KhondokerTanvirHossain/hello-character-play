# Stage 1 — Release Roadmap

| | |
|---|---|
| **Version** | 0.5 (Release 0 split into R0a and R0b) |
| **Date** | 2026-10-23 |
| **Implements** | [PRD v1.0](../specs/2026-10-22-socrates-prd.md), [Architecture v1.1](../specs/2026-10-23-socrates-architecture.md) |
| **Status** | Awaiting validation |

## How stage 1 is delivered

- Stage 1 ships as **ten releases: R0a, R0b, R1a, R1b, R2–R7**. Every release is deployed to `https://philosopher.eliooo.org` and ends with **release notes and a user-perspective test script** in `docs/releases/` (PRD `P9`, `NFR-11`).
- **Each release gets its own detailed implementation plan** in `docs/superpowers/plans/`, written when the release starts, so that what earlier releases and spikes taught us shapes it. Only the R0a and R0b plans exist now.
- **Gate between releases:** the owner runs the release's test script on the live site and approves before the next release starts.
- **Rule: no capability ships ahead of its safety and privacy.** Whatever a release makes possible (storing text, examining people, remembering them) ships with the protections for it in the same release.
- **Informal hour-long sessions** by the owner and spouse start at R1b and continue every release; AC-4 at R7 is the formal one, not the first.
- Development is **test-first**. Every change reaches production through `main`, the CI checks, and the restricted deploy (Architecture §13.2).

## Releases

### R0a — Infrastructure and a "coming soon" page

**Goal:** prove the whole production path on the shared box with **no AI and no production key**.

- Gradle multi-project; CI; hardened GHCR image published by digest; restricted, detached, self-rolling-back deploy; host firewall with a self-recovering watchdog; 2 GB fixed-size data filesystem; Caddy; uptime and CPU-credit monitoring.
- A public "coming soon" page and privacy note at `https://philosopher.eliooo.org`; `/healthz` with `degraded` signals.
- A reboot test and a rollback drill on the live box.

**Covers:** P9 (public from day zero), NFR-4, NFR-5, NFR-8 (logging rules), NFR-11, AC-5, Architecture §12–§13, §19 (disk and restart checks).
**Plan:** [Release 0a](../superpowers/plans/2026-10-23-release-0a-infrastructure.md).
**Exit:** the owner's R0a test script passes on the live site, including the reboot test and the rollback drill.

### R0b — Socrates with fail-safe safety

**Goal:** a real, public Socrates on the proven infrastructure, safe from his first word.

- **Spike S0** (API facts).
- Chat page → signed device cookie → WebSocket → **one actor per conversation** → one streamed Haiku 5.5 reply with persona v0.
- **Layered, fail-safe safety**: the crisis backstop (English, Bangla, romanized Bangla) on arrival; then, in order, a Haiku safety classifier on every message before any gate or reply (all FR-10.1 categories and identification with his fate); check-in follow-through; sticky care per conversation and per device with a confirmed "return to Socrates" control; a persona care marker that turns Socrates' own step-out into care; fail closed whenever classification is unavailable; safety reserve with per-IP shares; a permanent help line.
- Budget ledger with caps, reserve and per-IP daily shares; rate limits; `Origin` checks; trusted client IPs; reconnect without loss; in-character language notice for non-English messages. **No conversation storage.**
- **Path-gated live safety smoke in CI** (eval key A-4) blocking deploys that change safety, persona, model or turn code.

**Covers:** FR-1.1–1.4, FR-10 (all detection paths, sticky care, check-in, FR-10.7, FR-10.8), FR-12.1, FR-12.2, FR-12.5, FR-13 (subset), NFR-1 (feedback), NFR-2, NFR-3 (public pool and safety reserve), NFR-6 (partial, Architecture E-8), NFR-7 (disclosure; no text stored), NFR-10, NFR-12 (reserved `lang` fields).
**Plan:** [Release 0b](../superpowers/plans/2026-10-23-release-0b-socrates.md).
**Exit:** the owner's R0b test script passes on the live site; S0 findings recorded.

### R1a — Storage with its privacy, the owner's tools

**Goal:** conversations are kept, safely, and the owner can see inside.

- Conversation storage (SQLite) with the storage notice enforced by the server; resuming anonymous conversations.
- **Privacy controls in the same release**: "Delete this device's conversations"; **per-turn care-mode flags and the persisted conversation care state** (care text excluded from resume, deleted at 30 days); retention jobs; `pending_task` queue with cancellation; backups and deploy snapshots with retention caps.
- **Shared-box checks with storage**: disk usage above 85% and restart count reflected as `degraded` on `/healthz` (the uptime monitor watches it).
- Admin port with password: the chat client plus the **debug panel** (turn, timing, cost, cache, safety verdicts).
- **Owner pool and invite tokens**, so the owner's and spouse's testing never uses the public budget.
- Persona v1: convictions, behaviour rules and the convictions/answers line, modesty about his feats, anachronism and future-revelation rules, content policy, biography bible (core), misquote denylist.
- Spike **S4** load test at realistic concurrency.

**Covers:** FR-1.5, FR-4.6 (device part), FR-8, FR-12.3a, FR-12.4, NFR-3 (all pools), NFR-7, NFR-8, PRD §5, Architecture §19 (disk and restart checks).
**Exit:** owner verifies deletion, retention and care-text exclusion on the live site; owner and spouse test via invites.

### R1b — The Socratic examination, with its safety routing

**Goal:** Socrates genuinely examines; safety routing is complete before he does.

- **Spikes S1, S2, S3, S5**.
- Plan call (structured output) → validation → notebook operations (verbatim concessions, normalised quote matching) → **route precedence** (care > ambiguous > minors > language notice > normal) → speak call with beats; refusal handling.
- **Safety routing moves into the plan**: the plan's `safety` field takes over the R0 classifier's role on the normal path (the classifier stays on gated and capped paths); the ambiguous check-in in the visitor's language; minors mode; identification-with-his-fate routing; `plan_json` reduced to the safety level for care-mode turns (§9 scrub), with a test.
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
- Monitoring (Architecture §19), remainder: spend and cache-ratio checks, the digest; live flags (advisory).
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
| R0a | A-1 DNS record; A-10 CPU credit mode; A-11 uptime keyword monitor and credit alarm; A-13 approve the host firewall rules; A-15 make the image package public; approve each one-time server step; reboot test and rollback drill |
| R0b | ~~A-14 development key~~ (done 2026-10-23); **A-4 eval workspace key** (CI safety smoke); A-5 confirm caps and safety reserve; A-8 production workspace + spend limit; review the persona and the Bangla crisis phrases, distress cues and replies |
| R1a | Invite your spouse; check deletion on the live site |
| R1b | Start informal hour-long sessions |
| R2 | A-3 judge key; first calibration round |
| R3 | Spot-check passage tags |
| R4 | Test being remembered on two devices |
| R7 | The one-hour test |
| Before public launch | A-9 off-box backup decision |
