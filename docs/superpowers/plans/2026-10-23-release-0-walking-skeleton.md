# Release 0 — Walking Skeleton with Protections: Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** A public, end-to-end Socrates at `https://philosopher.eliooo.org`: a chat page, a WebSocket, one streamed Claude Haiku 5.5 reply in a minimal Socratic voice — with crisis handling, spend caps, rate limits, a hardened container and a restricted deploy from the first deploy.

**Architecture:** One Spring Boot container (Java 25) on the shared box behind the shared Caddy, on its own Docker network. `mind-core` (no Spring) holds the model interface, the crisis backstop and the turn logic; `mind-server` holds HTTP/WebSocket, the SQLite budget ledger, rate limits and health; `client-chat` is a plain TypeScript page served as static files. No conversation storage in R0.

**Tech Stack:** Java 25, Gradle (Kotlin DSL), Spring Boot 4.1.x (`spring-boot-starter-web`, `-websocket`, `-actuator`, `-jdbc`), Anthropic Java SDK 3.1.x, `org.xerial:sqlite-jdbc` 3.53.x, Flyway 13.x, Bucket4j 8.21.x, Caffeine 3.3.x, JUnit 5 + AssertJ; TypeScript + Vite + Vitest; Docker; GitHub Actions.

**Spec:** [Architecture v1.0](../../specs/2026-10-23-socrates-architecture.md) (§5, §6.4, §6.7, §8, §10–§13, §16 "Release 0") and [PRD v1.0](../../specs/2026-10-22-socrates-prd.md). Roadmap: [Stage 1 roadmap](../../plans/2026-10-23-stage1-roadmap.md).

## Global Constraints

- Java package root `org.eliooo.socrates`; modules `protocol`, `mind-core`, `mind-server`, `client-chat`; Gradle Kotlin DSL; Java toolchain **25**.
- Pin every dependency to an exact version (latest release at Task 1, checked on Maven Central/npm); pin every GitHub Action to a commit sha.
- Model id **`claude-haiku-5-5`** only, from configuration (`SOCRATES_MODEL`); never `temperature`/`top_p`/`top_k`, never an assistant prefill.
- Price table (USD per million tokens, verified in Task 2): input **0.10**, output **0.50**, cache read **0.01**, cache write (5 min) **0.125**; requests must stay under **100,000** input tokens.
- Visitor message max **2,000** characters (Unicode code points).
- Device cookie `socrates_device`: `Secure`, `HttpOnly`, `SameSite=Lax`, 13 months; minted only by `POST /api/session`, never on a page load; only its SHA-256 hash is kept server-side.
- Public port **8080** (via Caddy only); admin/management port **8081** bound to `127.0.0.1`.
- WebSocket `Origin` allowlist on port 8080: exactly `https://philosopher.eliooo.org` (plus `http://localhost:5173` in the `dev` profile only).
- Rate limits: per device 20 messages/minute and 300/day; per IP 60/minute and 400/day; at most 5 new device tokens per IP per hour. In R0 an IP limit **blocks with a notice** (converted to a Turnstile challenge in R5).
- Caps (owner-set via env, A-5): `SOCRATES_CAP_DAILY_USD`, `SOCRATES_CAP_MONTHLY_USD`; one pool (`public`) in R0.
- **No message text in logs**, ever; heap dumps off.
- The public repo must never contain server specifics (host, tenants, paths, container or network names other than `socrates_edge`/`socrates-mind`); use placeholders `<host>`, `<deploy dir>`, `<proxy container>`.

## Review Focus

1. **A mobile connection drops mid-reply and reconnects** — the visitor should not get a duplicate turn or a half-lost reply; a resent `clientMsgId` is ignored and the client shows the reply once. *(Test in Task 9.)*
2. **The visitor sends a second message while Socrates is still replying** — it must not start a second model call; the visitor gets a "one moment" notice. *(Task 6.)*
3. **Bengali script, emoji, or very long input** — length is counted in code points, Bengali is never mangled, and an oversized message gets a `too_long` notice instead of an error. *(Task 6.)*
4. **The model API fails or rate-limits mid-stream** — the visitor gets the in-character pause notice, and the ledger settles the partial usage (or the estimate), never leaving a reservation open. *(Task 6.)*
5. **Cookies blocked (private mode, strict browser settings)** — `POST /api/session` succeeds but the cookie never comes back; the page shows a plain "this site needs cookies" message instead of a dead chat. *(Tests in Tasks 8 and 9.)*

---

### Task 1: Repository scaffold and CI

**Files:**
- Create: `settings.gradle.kts`, `build.gradle.kts`, `gradle/libs.versions.toml`, `gradlew` (wrapper)
- Create: `protocol/build.gradle.kts`, `mind-core/build.gradle.kts`, `mind-server/build.gradle.kts`
- Create: `mind-core/src/test/java/org/eliooo/socrates/core/ScaffoldTest.java`
- Create: `.github/workflows/ci.yml`
- Modify: `.gitignore` (add `.gradle/`, `node_modules/`, `client-chat/dist/`, `*.db`)

**Interfaces:**
- Produces: modules `:protocol`, `:mind-core`, `:mind-server`; version catalog aliases used by every later task.

- [ ] **Step 1: Write the failing test** — `ScaffoldTest.toolchainIsJava25()` asserts `Runtime.version().feature()` is `25`.
- [ ] **Step 2: Run** `./gradlew :mind-core:test` — Expected: FAIL (no build yet).
- [ ] **Step 3: Create the multi-project build** with the Java 25 toolchain, JUnit 5 + AssertJ for all modules, and the version catalog pinned to the latest releases (verify each on Maven Central).
- [ ] **Step 4: Run** `./gradlew build` — Expected: BUILD SUCCESSFUL, 1 test passed.
- [ ] **Step 5: Add `ci.yml`**: on push and pull request, `./gradlew build` on `ubuntu-latest` with Temurin 25; Actions pinned to commit shas; no `pull_request_target`; `permissions: contents: read`.
- [ ] **Step 6: Commit** — `chore: gradle multi-project scaffold and CI`; push; confirm the CI run is green.

### Task 2: Spike S0 — verify the API facts

**Files:**
- Create: `spikes/s0-api-facts/build.gradle.kts`, `spikes/s0-api-facts/src/main/java/org/eliooo/socrates/spike/ApiFacts.java`
- Create: `docs/spikes/S0-api-facts.md`

**Interfaces:**
- Produces: the verified price table and SDK class names that Tasks 4 and 5 use. If a fact is false, stop and report to the owner before Task 4.

- [ ] **Step 1: Write `ApiFacts.main`**, run locally with `ANTHROPIC_API_KEY` from the owner's **production workspace**, one check per fact, printing PASS/FAIL with the evidence:
  1. Model `claude-haiku-5-5` streams text; `usage` is reported.
  2. Structured output (`output_config.format`, JSON schema) works on Haiku 5.5.
  3. `temperature` is rejected; an assistant prefill is rejected.
  4. A mid-conversation `role: "system"` message is accepted; with beta `mid-conversation-system-clear-at-2026-08-21` and `clear_at: "next_user_message"` it is accepted.
  5. A ≥600-token cached system block gives `cache_read_input_tokens > 0` on the second call.
  6. A request with 105K input tokens is billed at the higher tier (compare `usage` with the published table; record the observed values).
  7. `stop_reason` and `stop_details` fields exist on the SDK's message type (record class/method names).
- [ ] **Step 2: Run** `./gradlew :spikes:s0-api-facts:run` — Expected: one line per fact. Total cost: well under $0.10.
- [ ] **Step 3: Record results** in `docs/spikes/S0-api-facts.md`: each fact, PASS/FAIL, evidence, the exact SDK 3.x class and method names for streaming, structured output, cache control, and usage.
- [ ] **Step 4: Commit** — `spike: S0 API facts verified` (no keys, no outputs containing secrets).

### Task 3: Protocol v1 (R0 subset)

**Files:**
- Create: `protocol/src/main/resources/protocol/socrates-protocol.v1.json` (JSON Schema)
- Create: `protocol/src/main/java/org/eliooo/socrates/protocol/Frames.java`
- Test: `protocol/src/test/java/org/eliooo/socrates/protocol/SchemaConformanceTest.java`
- Create: `client-chat/src/protocol.ts` (generated by `json-schema-to-typescript` in Task 9's build)

**Interfaces:**
- Produces: sealed interface `Frame` with records (all carry `int v` = 1 and `String type`):
  - client→server: `Hello(int protocol, List<String> capabilities, String resumeConversationId)`, `VisitorMessage(String clientMsgId, String text)`, `ConversationNew()`
  - server→client: `SessionReady(String conversationId, String opening, boolean remembered)`, `SocratesThinking(String turnId)`, `SocratesDelta(String turnId, String text)`, `SocratesDone(String turnId)`, `SocratesNotice(NoticeKind kind, String text)`, `SafetyResources(List<HelpResource> resources)`
  - `enum NoticeKind { CAP_DAILY, CAP_MONTHLY, RATE_LIMITED, TOO_LONG, BUSY, ERROR, NOT_STORED }`; `record HelpResource(String name, String url, String phone, String region)`
  - `Frames.toJson(Frame)` / `Frames.fromJson(String) -> Frame` (Jackson; unknown fields ignored; unknown `type` → `Optional.empty()` from `Frames.tryParse`).

- [ ] **Step 1: Write the failing tests**: `everyRecordValidatesAgainstSchema()` serialises one instance of each record and validates it against the schema (use `com.networknt:json-schema-validator`); `unknownTypeIsIgnored()` asserts `Frames.tryParse("{\"v\":1,\"type\":\"future.thing\"}")` is empty; `unknownFieldIsIgnored()` parses a `SocratesDelta` with an extra field.
- [ ] **Step 2: Run** `./gradlew :protocol:test` — Expected: FAIL.
- [ ] **Step 3: Write the schema and records** so the tests pass; `type` strings exactly: `hello`, `visitor.message`, `conversation.new`, `session.ready`, `socrates.thinking`, `socrates.delta`, `socrates.done`, `socrates.notice`, `safety.resources`.
- [ ] **Step 4: Run** `./gradlew :protocol:test` — Expected: PASS.
- [ ] **Step 5: Commit** — `feat(protocol): v1 frames for release 0`.

### Task 4: Language model interface and the Anthropic implementation

**Files:**
- Create: `mind-core/src/main/java/org/eliooo/socrates/core/model/{LanguageModel,ModelRequest,ModelResult,Usage,StopReason,ModelFailure}.java`
- Create: `mind-core/src/main/java/org/eliooo/socrates/core/model/AnthropicLanguageModel.java`
- Create: `mind-core/src/testFixtures/java/org/eliooo/socrates/core/model/FakeLanguageModel.java`
- Test: `mind-core/src/test/java/org/eliooo/socrates/core/model/AnthropicLanguageModelLiveTest.java` (tag `live`, excluded by default)
- Test: `mind-core/src/test/java/org/eliooo/socrates/core/model/FakeLanguageModelTest.java`

**Interfaces:**
- Produces:
  - `interface LanguageModel { ModelResult stream(ModelRequest request, java.util.function.Consumer<String> onText) throws ModelFailure; }` — blocking (runs on a virtual thread), calls `onText` per text delta.
  - `record ModelRequest(String model, String system, List<ChatTurn> history, int maxTokens)`; `record ChatTurn(Role role, String text)`; `enum Role { VISITOR, SOCRATES }`.
  - `record ModelResult(StopReason stopReason, Usage usage)`; `enum StopReason { END_TURN, MAX_TOKENS, REFUSAL, OTHER }`; `record Usage(long input, long cacheRead, long cacheWrite, long output)`.
  - `class ModelFailure extends Exception` with `Usage partialUsage()` (may be estimated) and `boolean retryable()`.
  - `FakeLanguageModel(List<String> chunks, StopReason stop, Usage usage)` plus `FakeLanguageModel.failingAfter(int chunks, boolean retryable)`.

- [ ] **Step 1: Write the failing tests**: `FakeLanguageModelTest.streamsChunksInOrder()`; `failingAfterReportsPartialUsage()` (failure after 2 chunks → `ModelFailure.partialUsage().output() > 0`).
- [ ] **Step 2: Run** `./gradlew :mind-core:test` — Expected: FAIL.
- [ ] **Step 3: Implement** the records, the fake, and `AnthropicLanguageModel` with the SDK 3.x streaming call (names from `docs/spikes/S0-api-facts.md`): system as one text block with ephemeral cache control; history mapped VISITOR→user, SOCRATES→assistant; no sampling parameters; map `refusal` to `StopReason.REFUSAL`; on a stream exception, throw `ModelFailure` with usage estimated as `chars streamed / 4` output tokens.
- [ ] **Step 4: Write `AnthropicLanguageModelLiveTest.streamsAReply()`** (tag `live`): a 1-turn request returns `END_TURN` and `usage.output() > 0`.
- [ ] **Step 5: Run** `./gradlew :mind-core:test` (PASS) and `./gradlew :mind-core:test -Plive` with the key (PASS).
- [ ] **Step 6: Commit** — `feat(core): language model interface, Anthropic streaming implementation, fake`.

### Task 5: Crisis backstop and the canned care reply

**Files:**
- Create: `mind-core/src/main/java/org/eliooo/socrates/core/safety/{CrisisBackstop,TextNormalizer,Script,BackstopResult,CareReplies,CareReply}.java`
- Create: `corpus/safety/crisis-phrases.json`, `corpus/safety/care-replies.json`, `corpus/safety/resources.json`
- Test: `mind-core/src/test/java/org/eliooo/socrates/core/safety/CrisisBackstopTest.java`, `CareRepliesTest.java`

**Interfaces:**
- Produces:
  - `enum Script { LATIN, BENGALI, OTHER }`; `Script Script.of(String text)` (majority of letters; Bengali block U+0980–U+09FF).
  - `record BackstopResult(boolean hit, Script script, boolean romanizedBangla)`; `BackstopResult CrisisBackstop.check(String text)`.
  - `String TextNormalizer.normalize(String text)` — lower-case, Unicode NFC, collapse whitespace; for Latin text also fold doubled letters and vowel variants (`oo→u`, `ee→i`, `w→o` before a vowel, `ph→f`).
  - `CareReply CareReplies.forResult(BackstopResult)`; `record CareReply(String text, List<HelpResource> resources)` — Bengali script → Bangla; romanized Bangla or uncertain → Bangla + English; otherwise English; the international directory (`https://findahelpline.com`) is always included.

- [ ] **Step 1: Write the failing tests** (`CrisisBackstopTest`):
  - hits: `"I want to kill myself"`, `"i dont want to live anymore"`, `"আমি আত্মহত্যা করতে চাই"`, `"আমি মরে যেতে চাই"`, `"ami more jete chai"`, `"amar r bachte iccha kore na"`, `"atmohotta korbo"` (and spelling variant `"attohotta korbo"`).
  - no hits (philosophy must stay philosophy): `"Is death an evil?"`, `"Tell me about the death of Socrates"`, `"Should a man fear death?"`, `"my phone battery is dying"`.
  - `CareRepliesTest`: Bengali input → reply text in Bangla, resources contain findahelpline; romanized hit → reply contains both Bangla and English; English → English.
- [ ] **Step 2: Run** `./gradlew :mind-core:test --tests '*safety*'` — Expected: FAIL.
- [ ] **Step 3: Implement** with phrase lists in `crisis-phrases.json` (`en`, `bn`, `bn_latin` arrays; matching on normalised text, plus Levenshtein ≤ 1 per token for `bn_latin`), care texts in `care-replies.json` (short, plain, caring; persona dropped), resources in `resources.json`.
- [ ] **Step 4: Run** the tests — Expected: PASS.
- [ ] **Step 5: Owner review** — the owner (a Bangla speaker) reviews `crisis-phrases.json` and `care-replies.json` for natural, correct Bangla and romanized forms; apply corrections; re-run tests.
- [ ] **Step 6: Commit** — `feat(core): multilingual crisis backstop and canned care reply`.

### Task 6: The R0 turn service

**Files:**
- Create: `mind-core/src/main/java/org/eliooo/socrates/core/turn/{TurnService,TurnSink,Conversation,ConversationStore,BudgetGate,Reservation,Persona}.java`
- Create: `corpus/persona/persona-v0.md`
- Test: `mind-core/src/test/java/org/eliooo/socrates/core/turn/TurnServiceTest.java`

**Interfaces:**
- Consumes: `LanguageModel`, `CrisisBackstop`, `CareReplies` (Tasks 4–5); `Frames`' `NoticeKind`, `HelpResource` (Task 3).
- Produces:
  - `interface TurnSink { void thinking(String turnId); void delta(String turnId, String text); void done(String turnId); void notice(NoticeKind kind, String text); void safety(List<HelpResource> resources); }`
  - `interface BudgetGate { Optional<Reservation> reserve(BigDecimal estimateUsd); void settle(Reservation r, Usage actual, String model); }` — empty means a cap was reached (implemented in Task 7).
  - `interface ConversationStore { Conversation getOrCreate(String conversationId); }` — R0 implementation is in-memory (Caffeine, 2-hour expiry, max 10,000 conversations): **nothing is persisted**.
  - `final class TurnService { void handle(String conversationId, String clientMsgId, String text, TurnSink sink); }`
  - `String Persona.systemPrompt()` — loads `persona-v0.md` once.
- Order inside `handle` (Architecture §5, R0 subset): de-duplicate `clientMsgId` → one turn in flight (else `BUSY`) → length check (> 2,000 code points → `TOO_LONG`) → backstop (hit → care reply + `safety(...)`, **no cap check**) → `reserve` (empty → `CAP_DAILY`/`CAP_MONTHLY`) → `thinking` → `LanguageModel.stream` → `delta`s → `done` → `settle`. On `ModelFailure`: `notice(ERROR, "Let us pause a moment, my friend — try again shortly.")` and `settle` with partial usage. On `REFUSAL`: send the pre-written plain check-in from `care-replies.json` (`checkin` entry), no further model call.

- [ ] **Step 1: Write the failing tests** (all with `FakeLanguageModel` and a fake `BudgetGate`):
  - `streamsReplyAndSettles()` — sink receives `thinking`, deltas in order, `done`; `settle` called once with the fake usage.
  - `crisisMessageBypassesCapAndModel()` — gate always empty; message `"I want to kill myself"` → `safety` called with findahelpline; model never called.
  - `capReachedSendsNoticeWithoutModelCall()`.
  - `secondMessageWhileStreamingGetsBusy()` — a blocking fake model; second `handle` on the same conversation → `notice(BUSY, …)`, model called once.
  - `duplicateClientMsgIdIsIgnored()` — same `clientMsgId` twice → one model call.
  - `bengaliAndEmojiCountedInCodePoints()` — 2,000 Bengali characters plus emoji accepted; 2,001 code points → `TOO_LONG`.
  - `modelFailureMidStreamSettlesPartialUsage()` — `failingAfter(2, true)` → `ERROR` notice and `settle` with partial usage (Review Focus 4).
  - `refusalSendsCheckinWithoutSecondCall()`.
- [ ] **Step 2: Run** `./gradlew :mind-core:test --tests '*TurnService*'` — Expected: FAIL.
- [ ] **Step 3: Implement** `TurnService` and the in-memory store; write `persona-v0.md` (a short Socrates: 70, spring 399 BC; asks rather than tells; analogies; brief; no modern knowledge; never lectures; in-character, warm; ends each turn with a question).
- [ ] **Step 4: Run** the tests — Expected: PASS.
- [ ] **Step 5: Commit** — `feat(core): release-0 turn service with backstop, caps, single flight`.

### Task 7: Budget ledger on SQLite

**Files:**
- Create: `mind-server/src/main/java/org/eliooo/socrates/server/budget/{SqliteBudgetLedger,PriceTable,BudgetProperties}.java`
- Create: `mind-server/src/main/java/org/eliooo/socrates/server/db/DatabaseConfig.java`
- Create: `mind-server/src/main/resources/db/migration/V1__budget_ledger.sql`
- Test: `mind-server/src/test/java/org/eliooo/socrates/server/budget/SqliteBudgetLedgerTest.java`, `PriceTableTest.java`

**Interfaces:**
- Consumes: `BudgetGate`, `Reservation`, `Usage` (Tasks 4, 6).
- Produces: `SqliteBudgetLedger implements BudgetGate`; `BigDecimal PriceTable.cost(String model, Usage usage)`; Spring `DataSource` for `jdbc:sqlite:/data/socrates.db` (WAL, `busy_timeout=5000`), all access through a dedicated platform-thread executor of size 2 with a single writer connection (Architecture AD-4).
- Table `budget_ledger(pool TEXT, period TEXT, cost_usd TEXT, reserved_usd TEXT, PRIMARY KEY(pool, period))`; periods `day:YYYY-MM-DD` and `month:YYYY-MM` in UTC.

- [ ] **Step 1: Write the failing tests**:
  - `PriceTableTest.haikuCost()` — `Usage(1_000_000, 0, 0, 0)` → `0.10`; `Usage(0,0,0,1_000_000)` → `0.50`; `Usage(0,1_000_000,0,0)` → `0.01`; `Usage(0,0,1_000_000,0)` → `0.125` (values from S0).
  - `reserveWithinCapsSucceeds()`; `reserveBeyondDailyCapIsEmpty()`; `reserveBeyondMonthlyCapIsEmpty()`.
  - `settleReplacesReservationWithActual()` — reserved 0.01, actual 0.004 → day total 0.004, reserved 0.
  - `concurrentReservationsCannotOverspend()` — cap 0.01, 20 threads each reserving 0.001 → at most 10 succeed.
  - `ledgerSurvivesRestart()` — new ledger instance on the same file sees the same totals.
- [ ] **Step 2: Run** `./gradlew :mind-server:test --tests '*budget*'` — Expected: FAIL.
- [ ] **Step 3: Implement** with Flyway (add the Flyway SQLite database module if the core build reports it missing); caps from `SOCRATES_CAP_DAILY_USD` / `SOCRATES_CAP_MONTHLY_USD`; reservation estimate per turn: 0.005 USD (configurable).
- [ ] **Step 4: Run** the tests — Expected: PASS.
- [ ] **Step 5: Commit** — `feat(server): SQLite budget ledger with reservations and caps`.

### Task 8: Session cookie, rate limits, client IPs

**Files:**
- Create: `mind-server/src/main/java/org/eliooo/socrates/server/session/{SessionController,DeviceTokens}.java`
- Create: `mind-server/src/main/java/org/eliooo/socrates/server/limits/{RateLimits,LimitDecision}.java`
- Create: `mind-server/src/main/resources/application.yml` (forwarded headers, ports, Origin allowlist)
- Test: `mind-server/src/test/java/org/eliooo/socrates/server/session/SessionControllerTest.java`, `limits/RateLimitsTest.java`, `ForwardedHeadersTest.java`

**Interfaces:**
- Produces:
  - `POST /api/session` → `204`, sets `socrates_device` if absent (never replaces an existing valid one); response body none. `GET /api/session` → `{ "cookies": true|false }` so the page can detect blocked cookies (Review Focus 5).
  - `String DeviceTokens.mint()` (32 random bytes, base64url); `String DeviceTokens.hash(String token)` (SHA-256 hex).
  - `LimitDecision RateLimits.onMessage(String deviceHash, String clientIp)`; `LimitDecision RateLimits.onNewDevice(String clientIp)`; `enum LimitDecision { ALLOW, DEVICE_LIMITED, IP_LIMITED }` — Bucket4j buckets in Caffeine caches (max 50,000 entries, expire 25 hours).
  - Client IP from `X-Forwarded-For` **only** when the direct peer is inside the `socrates_edge` subnet (`SOCRATES_TRUSTED_PROXY_CIDR`, set by the deploy script); otherwise the peer address.

- [ ] **Step 1: Write the failing tests**:
  - `firstCallSetsLaxSecureHttpOnlyCookie()`; `existingCookieIsNotReplaced()`; `getReportsCookiesFalseWhenNoCookieSent()`.
  - `sixthNewDeviceFromOneIpWithinAnHourGets429()` — `POST /api/session` calls `RateLimits.onNewDevice` before minting.
  - `RateLimitsTest`: 21st message in a minute from one device → `DEVICE_LIMITED`; 6th new device from one IP within an hour → `IP_LIMITED`; 61st message per minute from one IP across devices → `IP_LIMITED`.
  - `ForwardedHeadersTest`: from a trusted proxy address, `X-Forwarded-For: 203.0.113.9` → client IP `203.0.113.9`; from an untrusted address the header is ignored.
- [ ] **Step 2: Run** `./gradlew :mind-server:test --tests '*session*' --tests '*limits*' --tests '*Forwarded*'` — Expected: FAIL.
- [ ] **Step 3: Implement** the controller, token helper, limits, and the trusted-proxy resolution.
- [ ] **Step 4: Run** the tests — Expected: PASS.
- [ ] **Step 5: Commit** — `feat(server): device session cookie, rate limits, trusted client IP`.

### Task 9: WebSocket endpoint and the chat page

**Files:**
- Create: `mind-server/src/main/java/org/eliooo/socrates/server/ws/{ConversationSocketHandler,WebSocketConfig,SocketTurnSink}.java`
- Create: `client-chat/{package.json,vite.config.ts,tsconfig.json,index.html}`, `client-chat/src/{main.ts,socket.ts,transcript.ts,styles.css}`
- Modify: `mind-server/build.gradle.kts` (build `client-chat` with the Gradle node plugin and copy `dist/` into `static/`)
- Test: `mind-server/src/test/java/org/eliooo/socrates/server/ws/ConversationSocketHandlerTest.java`; `client-chat/src/transcript.test.ts`

**Interfaces:**
- Consumes: `Frames` (Task 3), `TurnService`/`TurnSink` (Task 6), `RateLimits`, `DeviceTokens` (Task 8).
- Produces: WebSocket at `/ws`; handshake requires the `socrates_device` cookie and an allowlisted `Origin` (else 403); `hello` → `session.ready` with an opening from `corpus/openings-v0.json` (no model call) and a `NOT_STORED` notice; `conversation.new` → a fresh conversation id and a new `session.ready`; `visitor.message` → rate limits → `TurnService.handle`; WebSocket ping every 25 s.
- Page: a notice line ("Conversations are not stored in this early version."), transcript, input, send; on load calls `POST /api/session`, then `GET /api/session` — if `cookies:false`, shows "This site needs cookies to talk with Socrates." and no input (Review Focus 5); renders deltas into one Socrates bubble per `turnId`; renders `safety.resources` as plain links; reconnects with back-off and resends an unanswered message with the **same** `clientMsgId`.

- [ ] **Step 1: Write the failing tests**:
  - `handshakeRejectsForeignOrigin()`; `handshakeRejectsMissingCookie()`.
  - `helloGetsSessionReadyWithoutModelCall()` (fake model call count 0).
  - `conversationNewStartsEmptyConversation()` — after `conversation.new`, the next model request has an empty history.
  - `messageStreamsDeltasThenDone()` (fake model).
  - `reconnectResendIsNotDuplicated()` — send, drop the socket mid-stream, reconnect, resend same `clientMsgId` → model called once, final transcript has one Socrates turn (Review Focus 1).
  - `transcript.test.ts`: deltas for one `turnId` append to one bubble; a `done` closes it; `safety.resources` renders links with `rel="noopener"`.
- [ ] **Step 2: Run** `./gradlew :mind-server:test --tests '*ws*'` and `npm --prefix client-chat test` — Expected: FAIL.
- [ ] **Step 3: Implement** the handler, sink, config, and the page (plain TypeScript, no framework; text inserted with `textContent`, never `innerHTML`).
- [ ] **Step 4: Run** both test commands — Expected: PASS.
- [ ] **Step 5: Manual check** — `./gradlew :mind-server:bootRun --args='--spring.profiles.active=dev'` with a local key; open `http://localhost:8080`; ask "What is courage?" → a streamed Socratic question back.
- [ ] **Step 6: Commit** — `feat: websocket conversation and chat page`.

### Task 10: Health, security headers, logging rules

**Files:**
- Create: `mind-server/src/main/java/org/eliooo/socrates/server/health/{PublicHealthController,BuildInfo}.java`
- Create: `mind-server/src/main/java/org/eliooo/socrates/server/web/SecurityHeadersFilter.java`
- Modify: `mind-server/src/main/resources/application.yml` (management port 8081 on 127.0.0.1; `server.shutdown=graceful`; `spring.lifecycle.timeout-per-shutdown-phase=25s`)
- Test: `mind-server/src/test/java/org/eliooo/socrates/server/health/HealthTest.java`, `web/SecurityHeadersTest.java`, `LoggingRulesTest.java`

**Interfaces:**
- Produces: public `GET /healthz` → `{"status":"ok"}` (or `degraded`); management `GET http://127.0.0.1:8081/healthz/detail` → `{"service":"socrates-mind","sha":"<GIT_SHA>","build":"<BUILD_SHA>"}` (from env baked at build).

- [ ] **Step 1: Write the failing tests**: public health has no `sha` field; detail health on the management port returns the sha; CSP header is exactly `default-src 'self'; connect-src 'self'; img-src 'self' data:; style-src 'self'; script-src 'self'; frame-ancestors 'none'`; `LoggingRulesTest` runs a turn through `TurnService` with a captured log appender and asserts the visitor's text and the reply text appear in **no** log line.
- [ ] **Step 2: Run** `./gradlew :mind-server:test --tests '*health*' --tests '*Security*' --tests '*Logging*'` — Expected: FAIL.
- [ ] **Step 3: Implement**; JVM flag `-XX:-HeapDumpOnOutOfMemoryError` set in the Dockerfile (Task 11).
- [ ] **Step 4: Run** — Expected: PASS.
- [ ] **Step 5: Commit** — `feat(server): health endpoints, security headers, no-text logging`.

### Task 11: Hardened container image

**Files:**
- Create: `deploy/Dockerfile`, `deploy/run-flags.env` (the `docker run` flags in one place), `deploy/test-container.sh`
- Modify: `.github/workflows/ci.yml` (on `main`: build and push `ghcr.io/khondokertanvirhossain/hello-character-play/socrates-mind:<sha>` with `GIT_SHA`/`BUILD_SHA` build args)

**Interfaces:**
- Produces: image running as a non-root user (uid 10001), writable only `/data` (volume) and `/tmp` (tmpfs), listening on 8080 and 8081; `run-flags.env` defines: `--read-only --tmpfs /tmp:size=64m --user 10001:10001 --cap-drop=ALL --security-opt no-new-privileges --pids-limit=256 --cpus=0.5 --cpu-shares=512 --memory=640m --memory-swap=640m --log-driver json-file --log-opt max-size=10m --log-opt max-file=3`; JVM `-Xmx300m -XX:MaxMetaspaceSize=128m -XX:-HeapDumpOnOutOfMemoryError`.

- [ ] **Step 1: Write `deploy/test-container.sh`**: builds the image, runs it with exactly the flags from `run-flags.env` plus a temp volume and a dummy key, waits for `/healthz` = ok, asserts `docker exec` `id -u` is `10001`, asserts writing to `/app` fails, reports image size.
- [ ] **Step 2: Run** `bash deploy/test-container.sh` — Expected: FAIL (no Dockerfile).
- [ ] **Step 3: Write the Dockerfile** (multi-stage; Temurin 25 JRE runtime; copy the boot jar; create uid 10001; `EXPOSE 8080 8081`).
- [ ] **Step 4: Run** `bash deploy/test-container.sh` — Expected: PASS; record the image size in the commit message (disk planning, S4).
- [ ] **Step 5: Commit** — `build: hardened container image and GHCR publish`.

### Task 12: The restricted deploy script

**Files:**
- Create: `deploy/deploy-socrates.sh`
- Test: `deploy/test/test-deploy.sh` (runs the script with a fake `docker`, `df` and `curl` on `PATH`, recording calls)

**Interfaces:**
- Consumes: `run-flags.env` (Task 11).
- Produces: `deploy-socrates.sh`, invoked only via the forced command; reads the sha from `SSH_ORIGINAL_COMMAND` (or `$1` when run by the owner); configuration from `<deploy dir>/socrates.env` and `<deploy dir>/deploy.conf` (`PROXY_CONTAINER=…`), never hard-coded.
- Behaviour, in order: validate sha (`^[0-9a-f]{40}$`, else exit 2 having done nothing) → `df` check ≥ 1 GB free on `/` after pruning only `socrates-mind` tags (keep newest 2, never the running or target) → pull → SQLite snapshot via `VACUUM INTO` (keep last 3) → `docker network create socrates_edge` if missing and `docker network connect socrates_edge $PROXY_CONTAINER` if not attached → `docker stop -t 30 socrates-mind` → `docker run` with `run-flags.env`, `--network socrates_edge`, `-p 127.0.0.1:8081:8081`, `-v socrates_data:/data`, `--env-file socrates.env`, `-e SOCRATES_TRUSTED_PROXY_CIDR=<socrates_edge subnet>` → wait up to 60 s for `127.0.0.1:8081/healthz/detail` sha = target → on failure: stop new, restore snapshot, start previous image, exit 1.

- [ ] **Step 1: Write the failing tests** in `test-deploy.sh`: rejects `"; rm -rf /"` and a 39-char sha with no docker calls; refuses below 1 GB free and leaves the old container running; prunes only `socrates-mind` tags; never calls `docker` on any container other than `socrates-mind` except the one idempotent `network connect`; stops before it starts (no two containers at once); on failed health restores the snapshot and restarts the previous image.
- [ ] **Step 2: Run** `bash deploy/test/test-deploy.sh` — Expected: FAIL.
- [ ] **Step 3: Implement** `deploy-socrates.sh` (`set -euo pipefail`; prints no environment values).
- [ ] **Step 4: Run** — Expected: all tests PASS.
- [ ] **Step 5: Commit** — `build: restricted stop-then-start deploy script with tests`.

### Task 13: Deploy workflow and one-time server set-up

**Files:**
- Create: `.github/workflows/deploy.yml`
- Create: `docs/runbook/release-0-setup.md` (public steps with placeholders; real values only in the owner's private operations file)

**Interfaces:**
- Consumes: the image (Task 11), `deploy-socrates.sh` (Task 12).
- `deploy.yml`: after `ci.yml` succeeds on `main`, job in GitHub Environment `production` (protected, `main` only), Actions pinned to shas, runs `ssh -i <secret key> <secret host> "$GITHUB_SHA"` — the forced command ignores anything else.

- [ ] **Step 1: Write the runbook** with each one-time step, its verification, and its undo. **Every step is run with the owner present and approving it**:
  1. A-1: DNS A record `philosopher` → host IP (owner, at the DNS provider). Verify: `dig +short philosopher.eliooo.org`.
  2. `<deploy dir>`: create; copy `deploy-socrates.sh`; create `socrates.env` (mode 600) with the production key (workspace with spend limit, A-8) and caps (A-5); `deploy.conf`.
  3. Deploy key: generate; add one `authorized_keys` line `restrict,command="<deploy dir>/deploy-socrates.sh" …`; store the private key and host as GitHub `production` environment secrets. Verify: `ssh <host> "uname"` with that key runs the deploy script (which rejects `uname` and exits 2).
  4. `docker network create socrates_edge`; `docker network connect socrates_edge <proxy container>`. Verify: other sites still answer.
  5. A-13: `DOCKER-USER` rule dropping traffic from the `socrates_edge` subnet to `169.254.169.254`, persisted across reboots. Verify: `docker run --rm --network socrates_edge curlimages/curl -m 3 http://169.254.169.254/` times out, while the host itself still reaches it.
  6. Caddy: back up the shared Caddyfile; append the `philosopher.eliooo.org` block (Architecture §13.3); `caddy validate`; only if valid, `caddy reload`. Verify: all other sites answer; `curl -I https://philosopher.eliooo.org/healthz` → 200 once Socrates runs. Also confirm the shared Caddyfile still has no `admin` directive, so the proxy's admin API stays on its own container's localhost (Architecture §13.1).
  7. A-10: owner checks the CPU credit mode; A-11: owner sets up the uptime monitor on `/healthz` and the CPU-credit alarm.
- [ ] **Step 2: Write `deploy.yml`** and push to `main`.
- [ ] **Step 3: Run the one-time set-up** with the owner, step by step.
- [ ] **Step 4: Verify the first deploy**: the Actions run is green; `https://philosopher.eliooo.org/healthz` → `{"status":"ok"}`; the other tenants' sites still answer; `docker stats` shows `socrates-mind` under 640 MB and 0.5 CPU.
- [ ] **Step 5: Commit** — `ci: production deploy workflow; release-0 runbook`.

### Task 14: Release 0 notes and the owner's test script

**Files:**
- Create: `docs/releases/R0.md`
- Modify: `README.md` (status: Development, R0 live)

- [ ] **Step 1: Write `R0.md`**: what's live, known limits (no storage, minimal persona, one pool, IP limits block), and the **owner test script**, each with the expected result:
  1. Open `https://philosopher.eliooo.org` on a phone and on a laptop → the page loads; the "not stored" notice shows; Socrates' opening line appears.
  2. Type "What is courage?" → text begins within about 4 seconds and streams; Socrates answers with a question.
  3. Type "I want to kill myself" → a plain, caring reply (not Socrates) with help links; repeat in Bengali script and in romanized Bangla → the reply comes in Bangla, or Bangla + English.
  4. Type "Is death an evil?" → Socrates answers philosophically (no helplines).
  5. Send two messages quickly → the second gets "one moment".
  6. Turn on airplane mode mid-reply, then off → the reply appears once, not twice.
  7. Paste 2,001 characters → a "too long" notice.
  8. Send 21 messages within a minute → a rate-limit notice.
  9. Open in a private window with cookies blocked → "This site needs cookies".
  10. Reload → a new conversation (nothing remembered — expected in R0).
- [ ] **Step 2: Owner runs the script** on the live site; record pass/fail per item in `R0.md`.
- [ ] **Step 3: Commit** — `docs: release 0 notes and test results`.
