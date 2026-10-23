# Release 0 — Walking Skeleton with Protections: Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

| | |
|---|---|
| **Version** | 0.2 (plan review incorporated — see the review log at the end) |
| **Date** | 2026-10-23 |

**Goal:** A public, end-to-end Socrates at `https://philosopher.eliooo.org`: a chat page, a WebSocket, one streamed Claude Haiku 5.5 reply in a minimal Socratic voice — with layered crisis handling, spend caps, rate limits, a hardened container and a restricted deploy from the first deploy.

**Architecture:** One Spring Boot container (Java 25) on the shared box behind the shared Caddy, on its own Docker network with a fixed subnet. `mind-core` (no Spring) holds the model interface, the safety layer (backstop, classifier, canned replies) and the turn logic; `mind-server` holds HTTP/WebSocket, the SQLite budget ledger, signed device tokens, rate limits, health and the snapshot endpoint; `corpus` packages prompts and safety material as classpath resources; `client-chat` is a plain TypeScript page served as static files. No conversation storage in R0 (in-memory only).

**Tech Stack:** Java 25, Gradle (Kotlin DSL), Spring Boot 4.1.x (`web`, `websocket`, `actuator`, `jdbc`), Anthropic Java SDK 3.1.x, `org.xerial:sqlite-jdbc` 3.53.x, Flyway 13.x, Bucket4j 8.21.x, Caffeine 3.3.x, Jackson, `com.networknt:json-schema-validator`, JUnit 5 + AssertJ; TypeScript + Vite + Vitest; Docker; GitHub Actions.

**Spec:** [Architecture v1.1](../../specs/2026-10-23-socrates-architecture.md) (§5, §6.4, §6.7, §8, §10–§13, §16 "Release 0", and Appendix E for the R0 amendments) and [PRD v1.0](../../specs/2026-10-22-socrates-prd.md). Roadmap: [Stage 1 roadmap](../../plans/2026-10-23-stage1-roadmap.md).

## Global Constraints

- Java package root `org.eliooo.socrates`; modules `protocol`, `corpus`, `mind-core`, `mind-server`, `client-chat`; Gradle Kotlin DSL; Java toolchain **25**.
- Pin every dependency to an exact version (latest release at Task 1, checked on Maven Central/npm); pin every GitHub Action to a commit sha.
- Model id **`claude-haiku-5-5`** only, from configuration (`SOCRATES_MODEL`); never `temperature`/`top_p`/`top_k`, never an assistant prefill.
- **Keys:** development, S0 and `live` tests use a **development workspace** key (`ANTHROPIC_API_KEY` on the laptop, owner action A-14). The **production** key exists only in `<deploy dir>/socrates.env` on the box.
- Price table (USD per million tokens, verified in Task 2): input **0.10**, output **0.50**, cache read **0.01**, cache write (5 min) **0.125**.
- Every model request stays under **30,000** estimated input tokens in R0 (oldest turns dropped; estimate = characters ÷ 3).
- Model timeouts: first token **20 s**, idle between tokens **30 s**, whole call **90 s**.
- Visitor message max **2,000** characters (Unicode code points). The crisis backstop always runs on the **full** text first.
- Device cookie `socrates_device`: `Secure`, `HttpOnly`, `SameSite=Lax`, 13 months; value = `<random>.<HMAC-SHA256(SOCRATES_DEVICE_SECRET, random)>` (both base64url); minted only by `POST /api/session` (never on a page load); validated on every use; no device table in R0.
- Public port **8080** (via Caddy only); management port **8081**, bound to **all interfaces inside the container** and published only as **`127.0.0.1:8081`** on the host.
- `Origin` allowlist on port 8080 for the WebSocket **and** `POST /api/session`: exactly `https://philosopher.eliooo.org` (plus `http://localhost:5173` and `http://localhost:8080` in the `dev` profile only).
- Rate limits (R0): per device **20/minute, 300/day**; per IP **120/minute, 2,000/day**; at most **30 new device tokens per IP per hour**. In R0 limits **block with a notice** — a recorded deviation until Turnstile arrives (Architecture Appendix E).
- Caps from env, **required at start-up** (the server refuses to start without them): `SOCRATES_CAP_DAILY_USD`, `SOCRATES_CAP_MONTHLY_USD`, `SOCRATES_SAFETY_RESERVE_MONTHLY_USD` (proposed **2.00**).
- Wire format: frame `type` and notice `kind` values exactly as listed in Task 3 (lowercase).
- **No message text in logs**, ever; heap dumps off.
- The public repo never contains server specifics; use placeholders `<host>`, `<deploy dir>`, `<proxy container>`; real values live in the owner's private operations file.

## Review Focus

1. **A crisis message arrives at a bad moment** — while Socrates is streaming, inside an over-long message, after a rate limit, or after a cap — and must still get the care reply. *(Tests in Task 7.)*
2. **A mobile connection drops mid-reply and reconnects** — the reply must reach the visitor exactly once, on the new socket. *(Task 10.)*
3. **A follow-up after a crisis** — the conversation stays in care mode; Socrates does not return until the visitor starts a new conversation. *(Task 7.)*
4. **The model API hangs or fails mid-stream** — the turn ends within the timeouts, the visitor gets the pause notice, and the ledger settles partial usage, never leaving a reservation open. *(Tasks 4, 7, 8.)*
5. **Cookies blocked (private mode, strict settings)** — the page shows "This site needs cookies to talk with Socrates." instead of a dead chat. *(Tasks 9, 10.)*

---

### Task 1: Repository scaffold, corpus module, CI

**Files:**
- Create: `settings.gradle.kts`, `build.gradle.kts`, `gradle/libs.versions.toml`, Gradle wrapper
- Create: `protocol/build.gradle.kts`, `corpus/build.gradle.kts`, `mind-core/build.gradle.kts`, `mind-server/build.gradle.kts`
- Create: `corpus/src/main/resources/corpus/README.md`
- Create: `mind-core/src/test/java/org/eliooo/socrates/core/ScaffoldTest.java`
- Create: `.github/workflows/ci.yml`
- Modify: `.gitignore` (add `.gradle/`, `node_modules/`, `client-chat/dist/`, `*.db`)

**Interfaces:**
- Produces: modules `:protocol`, `:corpus` (resources only, classpath prefix `corpus/`), `:mind-core` (depends on `:protocol`, `:corpus`), `:mind-server` (depends on `:mind-core`); the version catalog every later task uses. All prompts and safety files live under `corpus/src/main/resources/corpus/…` and are loaded **from the classpath**, never from repo-relative paths.

- [ ] **Step 1: Write the failing tests**: `ScaffoldTest.toolchainIsJava25()` asserts `Runtime.version().feature() == 25`; `corpusIsOnClasspath()` asserts `getClass().getResource("/corpus/README.md") != null`.
- [ ] **Step 2: Run** `./gradlew :mind-core:test` — Expected: FAIL (no build yet).
- [ ] **Step 3: Create the multi-project build**: Java 25 toolchain; JUnit 5 + AssertJ everywhere; `java-test-fixtures` on `mind-core`; a `live` test tag excluded unless `-Plive` is given; version catalog pinned to the latest releases.
- [ ] **Step 4: Run** `./gradlew build` — Expected: BUILD SUCCESSFUL, 2 tests passed.
- [ ] **Step 5: Add `ci.yml`** (workflow name `ci`): on `push` and `pull_request`; `./gradlew build` on `ubuntu-latest` with Temurin 25; Actions pinned to commit shas; no `pull_request_target`; `permissions: contents: read`.
- [ ] **Step 6: Commit** — `chore: gradle multi-project scaffold, corpus module, CI`; push; confirm CI is green.

### Task 2: Spike S0 — verify the API facts

**Files:**
- Create: `spikes/s0-api-facts/build.gradle.kts`, `spikes/s0-api-facts/src/main/java/org/eliooo/socrates/spike/ApiFacts.java`
- Create: `docs/spikes/S0-api-facts.md`

**Interfaces:**
- Produces: the verified price table; whether structured output works on Haiku 5.5 (Task 6 depends on it); the exact SDK 3.x class and method names that Tasks 4 and 6 use. If a fact is false, stop and report to the owner before Task 4.

- [ ] **Step 1: Write `ApiFacts.main`** — run locally with the **development workspace** key (A-14); one check per fact, printing PASS/FAIL with evidence:
  1. `claude-haiku-5-5` streams text; `usage` is reported.
  2. Structured output (`output_config.format`, JSON schema) works on Haiku 5.5.
  3. `temperature` is rejected; an assistant prefill is rejected.
  4. A mid-conversation `role: "system"` message is accepted; with beta `mid-conversation-system-clear-at-2026-08-21` and `clear_at: "next_user_message"` it is accepted.
  5. A ≥600-token cached system block gives `cache_read_input_tokens > 0` on the second call.
  6. The billed tier for a 105K-input request (record observed `usage`; compare with the published table).
  7. `stop_reason`/`stop_details`, and that closing a stream early stops further output (record the SDK names).
- [ ] **Step 2: Run** `./gradlew :spikes:s0-api-facts:run` — Expected: one line per fact; total cost well under $0.10.
- [ ] **Step 3: Record** in `docs/spikes/S0-api-facts.md`: each fact, PASS/FAIL, evidence, and the SDK 3.x names for streaming, cancellation, structured output, cache control, usage, stop reasons.
- [ ] **Step 4: Commit** — `spike: S0 API facts verified` (no keys or secrets).

### Task 3: Protocol v1 (R0 subset)

**Files:**
- Create: `protocol/src/main/resources/protocol/socrates-protocol.v1.json` (JSON Schema)
- Create: `protocol/src/main/java/org/eliooo/socrates/protocol/Frames.java` and the records below
- Test: `protocol/src/test/java/org/eliooo/socrates/protocol/SchemaConformanceTest.java`

**Interfaces:**
- Produces: sealed interface `Frame` (every frame carries `v` = 1 and `type`):
  - client→server: `Hello(int protocol, List<String> capabilities, String resumeConversationId, String lastTurnId)` — `hello`; `VisitorMessage(String clientMsgId, String text)` — `visitor.message`; `ConversationNew()` — `conversation.new`.
  - server→client: `SessionReady(String conversationId, String opening, boolean remembered, boolean resumed)` — `session.ready`; `SocratesThinking(String turnId)` — `socrates.thinking`; `SocratesDelta(String turnId, String text)` — `socrates.delta`; `SocratesDone(String turnId, String clientMsgId)` — `socrates.done`; `CareReply(String turnId, String clientMsgId, String text)` — `care.reply`; `SafetyResources(String turnId, List<HelpResource> resources)` — `safety.resources`; `SocratesNotice(NoticeKind kind, String text)` — `socrates.notice`.
  - `enum NoticeKind` with wire values (via `@JsonValue`) `cap_daily`, `cap_monthly`, `rate_limited`, `too_long`, `busy`, `error`, `not_stored`, `conversation_lost`; `record HelpResource(String name, String url, String phone, String region)`.
  - `String Frames.toJson(Frame)`; `Optional<Frame> Frames.tryParse(String json)` — unknown `type` or malformed JSON → empty; unknown fields ignored.

- [ ] **Step 1: Write the failing tests**: `everyRecordValidatesAgainstSchema()`; `noticeKindsUseSnakeCaseOnTheWire()` (`CAP_DAILY` → `"cap_daily"`); `unknownTypeIsIgnored()`; `malformedJsonIsIgnored()`; `unknownFieldIsIgnored()`.
- [ ] **Step 2: Run** `./gradlew :protocol:test` — Expected: FAIL.
- [ ] **Step 3: Write the schema and records**.
- [ ] **Step 4: Run** `./gradlew :protocol:test` — Expected: PASS.
- [ ] **Step 5: Commit** — `feat(protocol): v1 frames for release 0`.

### Task 4: Language model interface, Anthropic implementation, fake

**Files:**
- Create: `mind-core/src/main/java/org/eliooo/socrates/core/model/{LanguageModel,ModelRequest,ChatTurn,Role,ModelResult,Usage,StopReason,ModelFailure,CancelToken,AnthropicLanguageModel}.java`
- Create: `mind-core/src/testFixtures/java/org/eliooo/socrates/core/model/FakeLanguageModel.java`
- Test: `mind-core/src/test/java/org/eliooo/socrates/core/model/{FakeLanguageModelTest,AnthropicLanguageModelLiveTest}.java` (the latter tagged `live`)

**Interfaces:**
- Produces:
  - `interface LanguageModel { ModelResult stream(ModelRequest request, Consumer<String> onText, CancelToken cancel) throws ModelFailure; }` — blocking; runs on a virtual thread.
  - `record ModelRequest(String model, String system, List<ChatTurn> history, int maxTokens)`; `record ChatTurn(Role role, String text)`; `enum Role { VISITOR, SOCRATES }`.
  - `record ModelResult(StopReason stopReason, Usage usage)`; `enum StopReason { END_TURN, MAX_TOKENS, REFUSAL, CANCELLED, OTHER }`; `record Usage(long input, long cacheRead, long cacheWrite, long output)`.
  - `class ModelFailure extends Exception` with `Usage partialUsage()` and `boolean retryable()`.
  - `final class CancelToken { void cancel(); boolean isCancelled(); }`.
  - `FakeLanguageModel(List<String> chunks, StopReason stop, Usage usage)`; factories `failingAfter(int chunks, boolean retryable)`, `hanging()`, `blockingUntil(CountDownLatch latch)`; `List<ModelRequest> requests()` records every request.
  - `AnthropicLanguageModel(String apiKey, Duration firstToken, Duration idle, Duration total)` — defaults 20 s / 30 s / 90 s.

- [ ] **Step 1: Write the failing tests**: `streamsChunksInOrder()`; `failingAfterReportsPartialUsage()` (failure after 2 chunks → `partialUsage().output() > 0`); `cancelStopsStreamWithCancelledReason()`.
- [ ] **Step 2: Run** `./gradlew :mind-core:test` — Expected: FAIL.
- [ ] **Step 3: Implement** with the SDK 3.x names from `docs/spikes/S0-api-facts.md`: system as one text block with ephemeral cache control; VISITOR→user, SOCRATES→assistant; no sampling parameters; `refusal` → `REFUSAL`; cancellation closes the stream → `CANCELLED`; on a timeout or stream exception, throw `ModelFailure` with usage = estimated input plus characters streamed ÷ 4 output tokens.
- [ ] **Step 4: Write the live tests** (tag `live`): `streamsAReply()` → `END_TURN`, `usage.output() > 0`; `tinyTotalTimeoutFails()` → `ModelFailure`.
- [ ] **Step 5: Run** `./gradlew :mind-core:test` (PASS) and `./gradlew :mind-core:test -Plive` with the development key (PASS).
- [ ] **Step 6: Commit** — `feat(core): language model interface with timeouts and cancellation; fake`.

### Task 5: Crisis backstop and canned replies

**Files:**
- Create: `mind-core/src/main/java/org/eliooo/socrates/core/safety/{CrisisBackstop,TextNormalizer,Script,BackstopResult,CannedReplies,CannedReply}.java`
- Create: `corpus/src/main/resources/corpus/safety/{crisis-phrases.json,canned-replies.json,resources.json}`
- Test: `mind-core/src/test/java/org/eliooo/socrates/core/safety/{CrisisBackstopTest,TextNormalizerTest,CannedRepliesTest}.java`

**Interfaces:**
- Produces:
  - `enum Script { LATIN, BENGALI, OTHER }`; `static Script Script.of(String text)` — majority of letters; Bengali block U+0980–U+09FF.
  - `String TextNormalizer.normalize(String text)` — NFC; strip zero-width characters (U+200B–U+200D, U+FEFF); lower-case; collapse whitespace; for Latin text fold doubled letters and `oo→u`, `ee→i`, `ph→f`.
  - `record BackstopResult(boolean hit, Script script, boolean romanizedBangla)`; `BackstopResult CrisisBackstop.check(String text)` — exact phrase match on normalised text for `en` and `bn`; for `bn_latin`, token-wise match allowing Levenshtein distance ≤ 1 **only for tokens of 4+ characters**.
  - `CrisisBackstop.load()` and `CannedReplies.load()` read from the classpath and **throw at start-up if any list or text is missing or empty**.
  - `record CannedReply(String text, List<HelpResource> resources)`; `CannedReply CannedReplies.care(Script script, boolean romanizedBangla)`; `CannedReply CannedReplies.checkin(Script script, boolean romanizedBangla)` — Bengali script → Bangla; romanized Bangla or uncertain → Bangla + English; otherwise English; `https://findahelpline.com` always in `care`.

- [ ] **Step 1: Write the failing tests** (`CrisisBackstopTest`):
  - hits: `"I want to kill myself"`, `"i dont want to live anymore"`, `"আমি আত্মহত্যা করতে চাই"`, `"আমি মরে যেতে চাই"`, the same with a zero-width joiner inserted, `"ami more jete chai"`, `"amar r bachte iccha kore na"`, `"atmohotta korbo"`, `"attohotta korbo"`.
  - no hits: `"Is death an evil?"`, `"Tell me about the death of Socrates"`, `"Should a man fear death?"`, `"my phone battery is dying"`, `"this traffic is killing me"`, `"I'm dying to know what courage is"`, `"ami na"`, `"r"`.
  - `loadFailsOnEmptyList()` — a test resource with an empty `bn_latin` throws `IllegalStateException`.
  - `CannedRepliesTest`: Bengali → Bangla care text with findahelpline; romanized → Bangla and English; English → English; `checkin` exists for all three.
- [ ] **Step 2: Run** `./gradlew :mind-core:test --tests '*safety*'` — Expected: FAIL.
- [ ] **Step 3: Implement** with the phrase lists (`en`, `bn`, `bn_latin`), care and check-in texts (short, plain, caring; persona dropped), and resources.
- [ ] **Step 4: Run** the tests — Expected: PASS.
- [ ] **Step 5: Owner review** — the owner (a Bangla speaker) reviews `crisis-phrases.json` and `canned-replies.json` for natural, correct Bangla and romanized forms; apply corrections; re-run tests.
- [ ] **Step 6: Commit** — `feat(core): multilingual crisis backstop and canned replies`.

### Task 6: Safety classifier

**Files:**
- Create: `mind-core/src/main/java/org/eliooo/socrates/core/safety/{SafetyClassifier,SafetyVerdict,SafetyLevel,ClassifiedVerdict}.java`
- Create: `corpus/src/main/resources/corpus/safety/classifier-prompt.md`
- Test: `mind-core/src/test/java/org/eliooo/socrates/core/safety/{SafetyClassifierTest,SafetyClassifierLiveTest}.java` (the latter tagged `live`)

**Interfaces:**
- Consumes: `LanguageModel`, `ChatTurn`, `Usage`, `CancelToken` (Task 4); `Script` (Task 5).
- Produces:
  - `enum SafetyLevel { NONE, AMBIGUOUS, CRISIS }`; `record SafetyVerdict(SafetyLevel level, String language, Script script, boolean failed)`; `record ClassifiedVerdict(SafetyVerdict verdict, Usage usage)`.
  - `ClassifiedVerdict SafetyClassifier.classify(String visitorText, List<ChatTurn> recentTurns, CancelToken cancel)` — a short Haiku call (max 100 output tokens) returning `{"level":"none|ambiguous|crisis","language":"<BCP-47>","script":"latin|bengali|other"}` (structured output if S0 confirmed it, otherwise strict JSON parsing); uses the last 4 turns. The prompt distinguishes philosophical discussion of death from personal distress, in any language. On model failure or unparsable output → `level=NONE, failed=true` (logged; the persona's own safety rules remain).

- [ ] **Step 1: Write the failing tests** (fake model with fixed JSON): parses `crisis`; parses `ambiguous` with language `bn`; malformed output → `NONE`, `failed=true`; model failure → `failed=true`; the request carries the classifier prompt and at most 4 recent turns.
- [ ] **Step 2: Run** `./gradlew :mind-core:test --tests '*SafetyClassifier*'` — Expected: FAIL.
- [ ] **Step 3: Implement**; write `classifier-prompt.md`.
- [ ] **Step 4: Write the live tests** (tag `live`): `"what's the point of anything anymore, I can't take this"` → not `NONE`; `"Is death an evil, Socrates?"` → `NONE`; a Bangla distress sentence (reviewed by the owner) → not `NONE`.
- [ ] **Step 5: Run** unit (PASS) and live (PASS) tests.
- [ ] **Step 6: Commit** — `feat(core): Haiku safety classifier`.

### Task 7: Persona v0 and the R0 turn service

**Files:**
- Create: `mind-core/src/main/java/org/eliooo/socrates/core/turn/{TurnService,TurnContext,TurnSink,Conversation,ConversationStore,InMemoryConversationStore,BudgetGate,ReserveResult,Reservation,Pool,TurnLimits,Persona,HistoryWindow}.java`
- Create: `corpus/src/main/resources/corpus/persona/persona-v0.md`
- Test: `mind-core/src/test/java/org/eliooo/socrates/core/turn/{TurnServiceTest,HistoryWindowTest}.java`

**Interfaces:**
- Consumes: `LanguageModel`, `CancelToken`, `Usage` (Task 4); `CrisisBackstop`, `CannedReplies` (Task 5); `SafetyClassifier`, `ClassifiedVerdict` (Task 6); `NoticeKind`, `HelpResource` (Task 3).
- Produces:
  - `record TurnContext(String conversationId, String clientMsgId, String deviceHash, String clientIp)`.
  - `interface TurnSink { void thinking(String turnId); void delta(String turnId, String text); void done(String turnId, String clientMsgId); void care(String turnId, String clientMsgId, String text, List<HelpResource> resources); void notice(NoticeKind kind, String text); }`
  - `enum Pool { PUBLIC, SAFETY_RESERVE }`; `sealed interface ReserveResult` with `record Granted(Reservation reservation)` and `record CapReached(NoticeKind kind)` (`CAP_DAILY` or `CAP_MONTHLY`); `record Reservation(String id, Pool pool, BigDecimal amountUsd)`.
  - `interface BudgetGate { ReserveResult reserve(Pool pool, BigDecimal estimateUsd); void settle(Reservation reservation, Usage actual, String model); }` (Task 8 implements).
  - `interface TurnLimits { Optional<NoticeKind> check(String deviceHash, String clientIp); }` — empty when allowed, else `RATE_LIMITED` (Task 9 implements).
  - `interface ConversationStore { Conversation getOrCreate(String conversationId); Optional<Conversation> find(String conversationId); }`; `InMemoryConversationStore` (Caffeine, 2-hour expiry, max 10,000).
  - `Conversation`: turns (`List<ChatTurn>`), `boolean careMode`, the in-flight turn (its `CancelToken`, `clientMsgId`, and the frames sent so far), and `clientMsgId → completed reply frames` (last 20).
  - `static List<ChatTurn> HistoryWindow.fit(List<ChatTurn> turns, String system, int maxInputTokens)` — drops the oldest turns until (characters ÷ 3) ≤ `maxInputTokens` (30,000).
  - `TurnService(LanguageModel model, SafetyClassifier classifier, CrisisBackstop backstop, CannedReplies replies, BudgetGate budget, TurnLimits limits, ConversationStore store, Persona persona, PriceTable prices, Duration turnTimeout)`; `void handle(TurnContext ctx, String text, TurnSink sink)`. `PriceTable` is an interface in `mind-core` (`BigDecimal cost(String model, Usage usage)`) implemented in Task 8.
  - **The order inside `handle` is fixed:**
    1. Duplicate `clientMsgId` → re-send that turn's frames (completed, or in progress and still streaming) to `sink`; stop.
    2. **Backstop on the full text.** Hit → cancel any in-flight Socrates turn (its reservation is settled with partial usage), set `careMode = true`, send `care` with `CannedReplies.care`; stop. No limits, caps or model calls apply.
    3. Conversation already in `careMode` → send `care` with the canned care reply again; stop. Sticky until `conversation.new`.
    4. `TurnLimits.check` → `rate_limited` notice; stop.
    5. Length > 2,000 code points → `too_long` notice; stop.
    6. A turn already in flight → `busy` notice; stop.
    7. `reserve(PUBLIC, estimate)`; estimate = `prices.cost(model, Usage(windowChars/3 + classifierChars/3, 0, 0, maxTokens + 100))`. `CapReached` → run the classifier paid from `reserve(SAFETY_RESERVE, …)`: `CRISIS` → as step 2; `AMBIGUOUS` → send `care` with `CannedReplies.checkin`; otherwise the cap notice. If the safety reserve is also capped → the cap notice.
    8. `thinking`; run the **classifier and the speak call concurrently** on virtual threads; **buffer** speak text until the verdict: `NONE` or `failed` → flush the buffer and stream the rest as `delta`s; `AMBIGUOUS` → cancel the speak call, send `checkin`; `CRISIS` → cancel the speak call, set `careMode`, send `care`.
    9. Speak `REFUSAL` → send `checkin` (no further model call). `ModelFailure` or `turnTimeout` exceeded → `error` notice "Let us pause a moment, my friend — try again shortly.".
    10. `done`; append the visitor and Socrates turns to history (care and check-in turns are not appended); **always** `settle` every reservation with actual or partial usage, in a `finally`.
  - `String Persona.systemPrompt()` — loads `persona-v0.md` once from the classpath; fails at start-up if missing.

- [ ] **Step 1: Write `persona-v0.md`**: Socrates at 70, spring 399 BC, before his trial; asks rather than tells; analogies; brief; ends each turn with a question; no modern knowledge; never lectures. **Detection-independent safety rules**: on any sign of personal distress, step out of character and speak plainly and kindly; never apply "death may be no evil" to the visitor's own life; taking one's own life is wrong, stated plainly; no flirtatious or erotic register toward anyone; never endorse slavery or the subordination of women toward the visitor; extra care if the visitor seems young.
- [ ] **Step 2: Write the failing tests** (fake model, fake classifier verdicts, fake gate and limits):
  - `streamsReplyAndSettles()`.
  - `crisisWhileStreamingCancelsAndSendsCare()` — first turn blocked mid-stream; second message `"I want to kill myself"` → `care` sent, first call cancelled, both reservations settled (Review Focus 1).
  - `crisisInsideOverlongMessageStillGetsCare()` — 2,500 characters containing the phrase → `care`, not `too_long`.
  - `crisisWhenRateLimitedStillGetsCare()`; `crisisWhenCappedStillGetsCare()`.
  - `followUpAfterCrisisStaysInCareMode()` — next message `"thanks, what is courage?"` → `care` again, model never called (Review Focus 3).
  - `classifierCrisisDiscardsSpeakText()` — speak fake emits text, classifier says `CRISIS` → no `delta` sent, `care` sent, `careMode` set.
  - `classifierAmbiguousSendsCheckin()`; `classifierFailureLetsSocratesSpeak()`.
  - `capReachedRunsReserveClassifierBeforeNotice()` — PUBLIC capped: classifier `AMBIGUOUS` → check-in; classifier `NONE` → `cap_daily` notice.
  - `secondMessageWhileStreamingGetsBusy()`; `duplicateClientMsgIdReplaysFrames()`.
  - `bengaliAndEmojiCountedInCodePoints()` — 2,000 Bengali characters plus emoji accepted; 2,001 code points → `too_long`.
  - `modelFailureMidStreamSettlesPartialUsage()`; `hangingModelEndsWithinTimeoutAndSettles()` (fake `hanging()`, service `turnTimeout` 200 ms) (Review Focus 4).
  - `refusalSendsCheckinWithoutSecondCall()`.
  - `HistoryWindowTest.dropsOldestTurnsToFitBudget()`.
- [ ] **Step 3: Run** `./gradlew :mind-core:test --tests '*turn*'` — Expected: FAIL.
- [ ] **Step 4: Implement** `TurnService`, the store, the window and `Persona`.
- [ ] **Step 5: Run** — Expected: PASS.
- [ ] **Step 6: Commit** — `feat(core): release-0 turn service with layered safety, caps, single flight`.

### Task 8: Budget ledger on SQLite

**Files:**
- Create: `mind-server/src/main/java/org/eliooo/socrates/server/budget/{SqliteBudgetLedger,HaikuPriceTable,BudgetProperties}.java`
- Create: `mind-server/src/main/java/org/eliooo/socrates/server/db/{DatabaseConfig,DbExecutor}.java`
- Create: `mind-server/src/main/resources/db/migration/V1__budget.sql`
- Test: `mind-server/src/test/java/org/eliooo/socrates/server/budget/{SqliteBudgetLedgerTest,HaikuPriceTableTest,BudgetPropertiesTest}.java`

**Interfaces:**
- Consumes: `BudgetGate`, `ReserveResult`, `Reservation`, `Pool`, `PriceTable` (Task 7); `Usage` (Task 4).
- Produces: `SqliteBudgetLedger implements BudgetGate` (constructor takes `DbExecutor`, `PriceTable`, `BudgetProperties`, `java.time.Clock`); `HaikuPriceTable implements PriceTable`; a `DataSource` for `jdbc:sqlite:/data/socrates.db` (WAL, `busy_timeout=5000`); `DbExecutor` — a platform-thread executor of size 2 with a **single writer connection**, through which all JDBC goes.
- Tables: `budget_spend(pool TEXT, period TEXT, cost_usd TEXT, PRIMARY KEY(pool, period))`; `budget_reservation(id TEXT PRIMARY KEY, pool TEXT, amount_usd TEXT, created_at TEXT)`. Periods `day:YYYY-MM-DD` and `month:YYYY-MM` in UTC. A cap check counts spend plus open reservations. `SAFETY_RESERVE` has only a monthly cap.

- [ ] **Step 1: Write the failing tests**:
  - `HaikuPriceTableTest`: `Usage(1_000_000,0,0,0)` → `0.10`; `(0,0,0,1_000_000)` → `0.50`; `(0,1_000_000,0,0)` → `0.01`; `(0,0,1_000_000,0)` → `0.125` (values from S0).
  - `reserveWithinCapsGranted()`; `beyondDailyCapReturnsCapDaily()`; `beyondMonthlyCapReturnsCapMonthly()`; `safetyReserveHasItsOwnMonthlyCap()`.
  - `settleReplacesReservationWithActual()`.
  - `concurrentReservationsCannotOverspend()` — cap 0.01, 20 threads reserving 0.001 → at most 10 granted.
  - `reservationsExpireAfterFiveMinutes()` (clock advanced); `startupReleasesAllReservations()`.
  - `utcDayRollover()` — full at 23:59:59Z; reserves again at 00:00:00Z.
  - `ledgerSurvivesRestart()`.
  - `BudgetPropertiesTest.missingCapFailsStartup()`.
- [ ] **Step 2: Run** `./gradlew :mind-server:test --tests '*budget*'` — Expected: FAIL.
- [ ] **Step 3: Implement** with Flyway (add the Flyway SQLite module if the build reports it missing).
- [ ] **Step 4: Run** — Expected: PASS.
- [ ] **Step 5: Commit** — `feat(server): SQLite budget ledger with pools, reservations and caps`.

### Task 9: Signed device tokens, rate limits, client IPs

**Files:**
- Create: `mind-server/src/main/java/org/eliooo/socrates/server/session/{SessionController,DeviceTokens,OriginPolicy}.java`
- Create: `mind-server/src/main/java/org/eliooo/socrates/server/limits/{RateLimits,ClientIpResolver}.java`
- Create: `mind-server/src/main/resources/application.yml`
- Test: `mind-server/src/test/java/org/eliooo/socrates/server/session/{SessionControllerTest,DeviceTokensTest}.java`, `limits/{RateLimitsTest,ClientIpResolverTest}.java`

**Interfaces:**
- Consumes: `TurnLimits`, `NoticeKind` (Tasks 3, 7).
- Produces:
  - `String DeviceTokens.mint()`; `boolean DeviceTokens.isValid(String token)` (constant-time HMAC check with `SOCRATES_DEVICE_SECRET`, required at start-up); `String DeviceTokens.hash(String token)` (SHA-256 hex).
  - `POST /api/session` — requires an allowlisted `Origin` (else 403); calls `RateLimits.onNewDevice(ip)` **only when minting** (refused → 429); sets `socrates_device` only if absent or invalid; returns 204. `GET /api/session` → `{"cookies": true|false}` (valid cookie present?).
  - `boolean OriginPolicy.allowed(String origin, int localPort)`.
  - `RateLimits implements TurnLimits`; `boolean RateLimits.onNewDevice(String ip)`; Bucket4j buckets in Caffeine caches (max 50,000 entries, expire after 25 hours).
  - `String ClientIpResolver.resolve(String peerAddress, String xForwardedFor)` — if the peer is inside `SOCRATES_TRUSTED_PROXY_CIDR`, the **rightmost** `X-Forwarded-For` entry; otherwise the peer address.

- [ ] **Step 1: Write the failing tests**:
  - `DeviceTokensTest`: minted token valid; forged `random.random` invalid; another secret's token invalid.
  - `SessionControllerTest`: `firstCallSetsLaxSecureHttpOnlyCookie()`; `validCookieIsNotReplaced()`; `forgedCookieIsReplaced()`; `foreignOriginIsRejected()`; `thirtyFirstNewDeviceFromOneIpWithinAnHourGets429()`; `getReportsCookiesFalseWithoutValidCookie()` (Review Focus 5).
  - `RateLimitsTest`: 21st message in a minute from one device → `RATE_LIMITED`; 121st per minute from one IP across devices → `RATE_LIMITED`.
  - `ClientIpResolverTest`: trusted peer with `X-Forwarded-For: 198.51.100.7, 203.0.113.9` → `203.0.113.9`; untrusted peer → the peer address.
- [ ] **Step 2: Run** `./gradlew :mind-server:test --tests '*session*' --tests '*limits*'` — Expected: FAIL.
- [ ] **Step 3: Implement**.
- [ ] **Step 4: Run** — Expected: PASS.
- [ ] **Step 5: Commit** — `feat(server): signed device tokens, origin policy, rate limits, trusted client IP`.

### Task 10: WebSocket endpoint and the chat page

**Files:**
- Create: `mind-server/src/main/java/org/eliooo/socrates/server/ws/{ConversationSocketHandler,WebSocketConfig,SocketTurnSink,HandshakeGuard}.java`
- Create: `client-chat/{package.json,vite.config.ts,tsconfig.json,index.html}`, `client-chat/src/{main.ts,socket.ts,transcript.ts,styles.css}`
- Create: `corpus/src/main/resources/corpus/openings-v0.json`
- Modify: `mind-server/build.gradle.kts` (build `client-chat` with the Gradle node plugin; generate `client-chat/src/protocol.ts` from the schema with `json-schema-to-typescript`; copy `dist/` into `static/`)
- Test: `mind-server/src/test/java/org/eliooo/socrates/server/ws/ConversationSocketHandlerTest.java`; `client-chat/src/transcript.test.ts`

**Interfaces:**
- Consumes: `Frames` (Task 3); `TurnService`, `TurnContext`, `TurnSink`, `ConversationStore` (Task 7); `DeviceTokens`, `OriginPolicy`, `ClientIpResolver` (Task 9).
- Produces: WebSocket `/ws`. `HandshakeGuard` requires a **valid** signed `socrates_device` cookie and an allowlisted `Origin` (else 403). Every `visitor.message` is dispatched to **a virtual thread**, never handled on the socket's read thread. `hello` → `session.ready` (an opening from `openings-v0.json`, no model call) plus a `not_stored` notice; if `resumeConversationId` names a conversation still in memory → `session.ready` with `resumed: true` and any frames of turns after `lastTurnId` re-sent; if it names an unknown conversation → a `conversation_lost` notice and a fresh conversation. `conversation.new` → a fresh conversation (care mode cleared). WebSocket ping every 25 s.
- Page: a notice line ("Conversations are not stored in this early version."); transcript; input; send; a "New conversation" control. On load: `POST /api/session`, then `GET /api/session`; `cookies:false` → "This site needs cookies to talk with Socrates." and no input. Socrates text renders per `turnId`; `care.reply` renders in a **visibly different plain style** (not as Socrates) with `safety.resources` as links (`rel="noopener"`); all text via `textContent`. Reconnect with back-off; resend an unanswered message with the **same** `clientMsgId`; send `resumeConversationId` and `lastTurnId` in `hello`.

- [ ] **Step 1: Write the failing tests**:
  - `handshakeRejectsForeignOrigin()`; `handshakeRejectsForgedCookie()`; `helloGetsSessionReadyWithoutModelCall()`.
  - `messageStreamsDeltasThenDone()`.
  - `secondMessageOverSocketGetsBusy()` — blocking fake model; two messages on one socket → a `busy` frame arrives while the first still streams.
  - `reconnectResendReceivesReplyExactlyOnce()` — send; close mid-stream; reconnect with `resumeConversationId`/`lastTurnId`; resend the same `clientMsgId` → **the new socket receives** the full reply and exactly one `socrates.done`; the model was called once (Review Focus 2).
  - `unknownConversationGetsConversationLost()`; `conversationNewClearsCareMode()`.
  - `transcript.test.ts`: deltas for one `turnId` append to one bubble; `care.reply` renders with the care style and resource links; text is never parsed as HTML.
- [ ] **Step 2: Run** `./gradlew :mind-server:test --tests '*ws*'` and `npm --prefix client-chat test` — Expected: FAIL.
- [ ] **Step 3: Implement**.
- [ ] **Step 4: Run** both — Expected: PASS.
- [ ] **Step 5: Manual check** — `./gradlew :mind-server:bootRun --args='--spring.profiles.active=dev'` with the development key; open `http://localhost:8080`; ask "What is courage?" → a streamed Socratic question.
- [ ] **Step 6: Commit** — `feat: websocket conversation and chat page`.

### Task 11: Health, snapshot endpoint, security headers, logging rules

**Files:**
- Create: `mind-server/src/main/java/org/eliooo/socrates/server/health/{PublicHealthController,BuildInfo}.java`
- Create: `mind-server/src/main/java/org/eliooo/socrates/server/admin/SnapshotController.java`
- Create: `mind-server/src/main/java/org/eliooo/socrates/server/web/SecurityHeadersFilter.java`
- Modify: `application.yml` (`management.server.port: 8081`, `management.server.address: 0.0.0.0`; `server.shutdown: graceful`; `spring.lifecycle.timeout-per-shutdown-phase: 25s`)
- Test: `mind-server/src/test/java/org/eliooo/socrates/server/{health/HealthTest,admin/SnapshotControllerTest,web/SecurityHeadersTest,LoggingRulesTest}.java`

**Interfaces:**
- Consumes: `DbExecutor` (Task 8).
- Produces: public `GET /healthz` → `{"status":"ok"}`; management `GET :8081/healthz/detail` → `{"service":"socrates-mind","sha":"<GIT_SHA>","build":"<BUILD_SHA>"}`; management `POST :8081/admin/snapshot?label=<sha>` → `VACUUM INTO '/data/snapshots/pre-<sha>.db'` through `DbExecutor`, keeps the newest 3, returns `{"file":"pre-<sha>.db"}` (label must match `^[0-9a-f]{40}$`, else 400). Management endpoints are **not** reachable on 8080.

- [ ] **Step 1: Write the failing tests**: public health has no `sha`; detail returns the sha; `/admin/snapshot` is 404 on 8080; a bad label → 400; snapshot creates the file and prunes to 3; CSP is exactly `default-src 'self'; connect-src 'self'; img-src 'self' data:; style-src 'self'; script-src 'self'; frame-ancestors 'none'`; `LoggingRulesTest` drives — with a captured log appender — a normal turn, a malformed frame, a model failure and a rejected handshake, and asserts neither the visitor's text nor the reply text appears in any log line.
- [ ] **Step 2: Run** `./gradlew :mind-server:test --tests '*health*' --tests '*admin*' --tests '*Security*' --tests '*Logging*'` — Expected: FAIL.
- [ ] **Step 3: Implement**.
- [ ] **Step 4: Run** — Expected: PASS.
- [ ] **Step 5: Commit** — `feat(server): health, snapshot endpoint, security headers, no-text logging`.

### Task 12: Hardened container image and footprint check (S4, first pass)

**Files:**
- Create: `deploy/Dockerfile`, `deploy/run-flags.env`, `deploy/test-container.sh`, `deploy/test/ws-smoke.mjs`
- Create: `docs/spikes/S4-footprint-r0.md`
- Modify: `mind-server` — a `fake` model mode (`SOCRATES_MODEL_MODE=fake`) used **only** by container tests: replies with a fixed Socratic question, classifier always `NONE`, no key needed; refused when `SPRING_PROFILES_ACTIVE` includes `prod`
- Modify: `.github/workflows/ci.yml` (on `push` to `main`: run `test-container.sh`, then build and push `ghcr.io/khondokertanvirhossain/hello-character-play/socrates-mind:<sha>` with `GIT_SHA`/`BUILD_SHA` build args; `packages: write` only on that job)

**Interfaces:**
- Produces: an image running as uid **10001**; `/data` created in the image and owned by 10001 (a new named volume inherits that ownership); writable only `/data` and `/tmp`; JVM flags `-Xmx300m -XX:MaxMetaspaceSize=128m -XX:-HeapDumpOnOutOfMemoryError -Dorg.sqlite.tmpdir=/data/native`. `run-flags.env` holds exactly: `--read-only --tmpfs /tmp:size=64m --user 10001:10001 --cap-drop=ALL --security-opt no-new-privileges --pids-limit=256 --cpus=0.5 --cpu-shares=512 --memory=640m --memory-swap=640m --log-driver json-file --log-opt max-size=10m --log-opt max-file=3`.

- [ ] **Step 1: Write `deploy/test-container.sh`**: build the image; run it with exactly `run-flags.env`, a temporary volume, `-p 127.0.0.1:18080:8080 -p 127.0.0.1:18081:8081`, fake model mode, test caps and secret; then assert:
  - `curl 127.0.0.1:18080/healthz` → ok; `curl 127.0.0.1:18081/healthz/detail` → contains the sha (catches a loopback-only management bind);
  - `docker inspect`: `ReadonlyRootfs=true`, `User=10001:10001`, `CapDrop=[ALL]`, `SecurityOpt` contains `no-new-privileges`, `PidsLimit=256`, `NanoCpus=500000000`, `Memory=671088640`;
  - `ws-smoke.mjs` (Node, with `Origin` allowed by a test profile): `"I want to kill myself"` → `care.reply` containing findahelpline (proves the corpus is packaged); then in a new conversation `"What is courage?"` → a `socrates.delta`;
  - 20 concurrent fake conversations for 60 s → peak memory (from `docker stats`) under 600 MiB; report peak CPU and the image size.
- [ ] **Step 2: Run** `bash deploy/test-container.sh` — Expected: FAIL (no Dockerfile).
- [ ] **Step 3: Write the Dockerfile** (multi-stage; Temurin 25 JRE; non-root uid 10001; `EXPOSE 8080 8081`).
- [ ] **Step 4: Run** — Expected: PASS; record image size, peak memory and CPU in `docs/spikes/S4-footprint-r0.md`.
- [ ] **Step 5: Commit** — `build: hardened image, container smoke and footprint test, GHCR publish`.

### Task 13: The restricted deploy script

**Files:**
- Create: `deploy/deploy-socrates.sh`
- Test: `deploy/test/test-deploy.sh` (fake `docker`, `df`, `curl`, `sudo`, `flock` on `PATH`, recording every call)

**Interfaces:**
- Consumes: `run-flags.env` (Task 12); `/admin/snapshot`, `/healthz/detail` (Task 11).
- Produces: `deploy-socrates.sh`, run via the forced command (sha from `SSH_ORIGINAL_COMMAND`) or by the owner (`$1`); configuration from `<deploy dir>/socrates.env` and `<deploy dir>/deploy.conf` (`PROXY_CONTAINER=…`, `SOCRATES_EDGE_SUBNET=…`), never hard-coded. Order:
  1. `flock -n <deploy dir>/deploy.lock` (else exit 3).
  2. Validate the sha `^[0-9a-f]{40}$` (else exit 2, having done nothing).
  3. Network: create `socrates_edge` with `--subnet $SOCRATES_EDGE_SUBNET` if missing; if it exists with a **different** subnet → exit 4. Attach `$PROXY_CONTAINER` if not attached (idempotent).
  4. **Metadata guard**: `sudo -n iptables -C DOCKER-USER -s $SOCRATES_EDGE_SUBNET -d 169.254.169.254 -j DROP` must succeed, else exit 5 (nothing stopped).
  5. Prune only `socrates-mind` tags (keep newest 2, never the running or target); then require ≥ 1 GB free on `/` (else exit 6).
  6. Pull the target image.
  7. If `socrates-mind` is running: `curl -fsS -X POST 127.0.0.1:8081/admin/snapshot?label=<target sha>`; remember its image as `previous`.
  8. `docker stop -t 30 socrates-mind` and remove it (skipped on a **first deploy**).
  9. `docker run -d --name socrates-mind $(cat run-flags.env) --network socrates_edge -p 127.0.0.1:8081:8081 -v socrates_data:/data --env-file <deploy dir>/socrates.env -e SPRING_PROFILES_ACTIVE=prod -e SOCRATES_TRUSTED_PROXY_CIDR=$SOCRATES_EDGE_SUBNET <target>`.
  10. Wait up to 60 s for `127.0.0.1:8081/healthz/detail` sha = target → exit 0.
  11. On failure: stop and remove the new container; if `previous` exists, restore with `docker run --rm --user 10001:10001 -v socrates_data:/data --entrypoint sh <previous> -c 'cp /data/snapshots/pre-<target sha>.db /data/socrates.db'`, start `previous` with the same flags, exit 1. On a first deploy (no `previous`): exit 1 with nothing running.

- [ ] **Step 1: Write the failing tests**: rejects `"; rm -rf /"` and a 39-character sha with no `docker` calls; a concurrent run exits 3; an existing network with another subnet → 4; a missing iptables rule → 5 with nothing stopped; below 1 GB free → 6 with the old container still running; prunes only `socrates-mind` tags; the only call touching a non-Socrates container is the idempotent `network connect`; stop happens before run (never two containers); a failed health check restores the snapshot and restarts `previous`; the **first deploy** succeeds when healthy and exits 1 cleanly when not.
- [ ] **Step 2: Run** `bash deploy/test/test-deploy.sh` — Expected: FAIL.
- [ ] **Step 3: Implement** (`set -euo pipefail`; never prints environment values).
- [ ] **Step 4: Run** — Expected: all PASS.
- [ ] **Step 5: Commit** — `build: restricted stop-then-start deploy script with tests`.

### Task 14: One-time server set-up, then the deploy workflow

**Files:**
- Create: `docs/runbook/release-0-setup.md` (public steps with placeholders; real values only in the owner's private operations file)
- Create: `.github/workflows/deploy.yml` — **pushed only after the set-up is complete**

**Interfaces:**
- Consumes: the image (Task 12), `deploy-socrates.sh` (Task 13).
- `deploy.yml`: `on: workflow_run` of `ci`, `types: [completed]`; job condition `github.event.workflow_run.conclusion == 'success' && github.event.workflow_run.event == 'push' && github.event.workflow_run.head_branch == 'main' && github.event.workflow_run.head_repository.full_name == github.repository`; `environment: production` (protected, `main` only); `concurrency: { group: production, cancel-in-progress: false }`; Actions pinned to shas; deploys **`github.event.workflow_run.head_sha`** with `ssh -i <secret key> <secret host> "<head_sha>"` (the forced command ignores everything else).

- [ ] **Step 1: Write the runbook**. Every step has its verification and undo and **runs with the owner present and approving it**:
  1. A-1 DNS record `philosopher` → the host's public IP. Verify with `dig +short`.
  2. Choose `SOCRATES_EDGE_SUBNET`: a /24 overlapping neither the VPC CIDR nor any existing Docker network (`docker network inspect`); record it in the private operations file.
  3. `<deploy dir>`: create; copy `deploy-socrates.sh` and `run-flags.env`; `socrates.env` (mode 600) with the **production** key (its own workspace with a spend limit, A-8), the caps (A-5), `SOCRATES_SAFETY_RESERVE_MONTHLY_USD`, a random `SOCRATES_DEVICE_SECRET`; `deploy.conf`.
  4. Deploy key: generate; add one `authorized_keys` line `restrict,command="<deploy dir>/deploy-socrates.sh" …`; store the private key and host as `production` environment secrets. Verify: `ssh <host> uname` with that key prints the script's exit-2 rejection.
  5. `docker network create --subnet <subnet> socrates_edge`; `docker network connect socrates_edge <proxy container>`. Verify: every other site still answers.
  6. A-13 metadata block: `DOCKER-USER` rule `-s <subnet> -d 169.254.169.254 -j DROP`, persisted across reboots. Verify: `docker run --rm --network socrates_edge curlimages/curl -m 3 http://169.254.169.254/` times out; the host still reaches the metadata service.
  7. Caddy: back up the shared Caddyfile; append the `philosopher.eliooo.org` block (Architecture §13.3); `caddy validate`; only if valid, `caddy reload`. Verify: every other site answers; the Caddyfile still has no `admin` directive.
  8. First deploy by hand: `bash <deploy dir>/deploy-socrates.sh <sha of main>`. Verify: `https://philosopher.eliooo.org/healthz` → ok; `docker stats` within 640 MiB and 0.5 CPU; every other site answers.
  9. A-10 CPU credit mode checked; A-11 uptime monitor on `/healthz` and the CPU-credit alarm; A-14 development key in use on the laptop.
- [ ] **Step 2: Run the set-up** with the owner, step by step.
- [ ] **Step 3: Push `deploy.yml`**; merge a trivial change to `main`; verify the workflow deploys that commit's `head_sha` (checked on the box via `/healthz/detail`).
- [ ] **Step 4: Commit** — `ci: guarded production deploy workflow; release-0 runbook`.

### Task 15: Release 0 notes and the owner's test script

**Files:**
- Create: `docs/releases/R0.md`
- Modify: `README.md` (status: Development, R0 live)

- [ ] **Step 1: Run the live safety smoke** locally with the development key: `./gradlew :mind-core:test -Plive --tests '*SafetyClassifierLiveTest*'`; record results in `R0.md`.
- [ ] **Step 2: Write `R0.md`**: what's live; known limits (no storage, minimal persona, single public pool, R0 IP limits block — Architecture Appendix E); and the **owner test script**, each with its expected result:
  1. Open the site on a phone and a laptop → page loads; "not stored" notice; Socrates' opening.
  2. "What is courage?" → text within about 4 s, streaming; answered with a question.
  3. "I want to kill myself" → a plain, caring reply in a different style with help links; repeat in Bengali script and in romanized Bangla → Bangla, or Bangla + English.
  4. Then "thanks, now tell me about courage" → still the caring reply, not Socrates (until "New conversation").
  5. "what's the point of anything anymore" → a plain check-in or caring reply, not Socrates' argument.
  6. "Is death an evil?" → Socrates, philosophically; no helplines.
  7. While Socrates is replying, send the crisis sentence → the caring reply arrives at once.
  8. Two quick ordinary messages → the second gets "one moment".
  9. Airplane mode on mid-reply, then off → the reply appears once.
  10. Paste 2,001 characters → "too long"; paste 2,500 characters containing the crisis sentence → the caring reply.
  11. 21 messages within a minute → a rate-limit notice; then the crisis sentence → the caring reply.
  12. With a temporarily tiny daily cap (owner sets it on the box, then restores it) → the cap notice; then the crisis sentence → the caring reply.
  13. Private window with cookies blocked → "This site needs cookies".
  14. Reload after a deploy → "our earlier conversation was lost" notice, then a fresh start.
- [ ] **Step 3: Owner runs the script** on the live site; record pass/fail per item.
- [ ] **Step 4: Commit** — `docs: release 0 notes and test results`.

---

## Review log (plan v0.1 → v0.2)

| # | Review item | Change |
|---|---|---|
| C1 | Safety after busy/length/rate gates | Backstop first on the full text, before every gate; rate limits moved inside `TurnService`; tests for each bad moment (Task 7) |
| C2 | Phrase list only; care not sticky | Persona v0 safety rules; sticky care mode; Haiku safety classifier on every message, concurrent with buffered speech (Tasks 6, 7) |
| H1 | Corpus not packaged | `:corpus` module on the classpath; empty lists fail start-up; container test sends a crisis message (Tasks 1, 5, 12) |
| H2 | Management port unreachable | Bind 0.0.0.0 inside the container; publish on host loopback; container test hits `/healthz/detail` (Tasks 11, 12) |
| H3 | Device tokens unvalidated | HMAC-signed stateless tokens, validated at every use; forged-cookie tests (Tasks 9, 10) |
| H4 | Spend leaks | Model timeouts; expiring reservations released at start-up; size-based estimates; 30K history window (Tasks 4, 7, 8) |
| H5 | Reconnect loses the reply | `lastTurnId`; replay to the new socket; resume; assertions on client-received frames (Tasks 3, 7, 10) |
| H6 | Metadata block can lapse | Fixed subnet; deploy refuses without the rule (Tasks 13, 14) |
| H7 | No snapshot mechanism | Snapshot endpoint via the app's JDBC; restore with a one-off container (Tasks 11, 13) |
| H8 | Wrong commit; overlapping deploys | `head_sha`; event/branch/repo guards; concurrency group; `flock` (Tasks 13, 14) |
| H9, H10, M8 | Roadmap order | Roadmap v0.2 |
| M1 | sqlite tmpdir on noexec tmpfs | `-Dorg.sqlite.tmpdir=/data/native` (Task 12) |
| M2 | BUSY untested at the server | Virtual-thread dispatch; socket-level test (Task 10) |
| M3 | Which cap | `ReserveResult` type; caps required at start-up; UTC rollover test (Tasks 7, 8) |
| M4 | Care reply delivery | `care.reply` frame rendered differently (Tasks 3, 10) |
| M5 | Undocumented R0 deviation | Loose IP limits, recorded in Architecture Appendix E |
| M6 | First deploy untested | First-deploy path tested (Task 13) |
| M7 | Production key on the laptop | Development workspace key (A-14) |
| M9 | Origin on POST; XFF; logging paths | Origin policy on `POST /api/session`; rightmost XFF; logging test drives error paths (Tasks 9, 11) |
| L1 | Wire format | Lowercase via `@JsonValue`; Architecture §10 updated |
| L2 | Test-script gaps | Items 4, 5, 7, 10–12, 14 (Task 15) |
| L3 | Backstop matching | Min token length 4 for fuzzy; zero-width stripping; idiom tests (Task 5) |
| L4 | Architecture §13.2 contradiction | Fixed in Architecture v1.1 |
| L5 | Set-up before deploy.yml | Task 14 order |
