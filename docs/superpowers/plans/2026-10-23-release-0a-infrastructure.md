# Release 0a — Infrastructure and a "Coming Soon" Page: Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

| | |
|---|---|
| **Version** | 1.0 (split from the Release 0 plan v0.7 after its final external review — see the history at the end) |
| **Date** | 2026-10-23 |

**Goal:** Prove the whole production path on the shared box — hardened container, fenced network, fixed-size data disk, digest-pinned self-repairing deploys, Caddy, monitoring — with a public "coming soon" page at `https://philosopher.eliooo.org` and **no model calls and no production API key** anywhere.

**Architecture:** A minimal Spring Boot app (Java 25) serving a static page, `/healthz`, and an admin connector (health detail, SQLite snapshot) runs as one hardened container on its own Docker network with a fixed IP, fenced by a host firewall and a watchdog, with its data on a 2 GB loop-mounted filesystem. CI builds, tests and publishes an image by digest; a restricted SSH forced command deploys it with stop-then-start and automatic rollback. Release 0b adds Socrates on top of exactly this.

**Tech Stack:** Java 25, Gradle (Kotlin DSL), Spring Boot 4.1.x (`web`, `jdbc`, Flyway starter), `org.xerial:sqlite-jdbc` 3.53.x, Flyway 13.x, JUnit 5 + AssertJ; Docker 25 / standalone docker-compose on Amazon Linux 2023; iptables; systemd; GitHub Actions; GHCR.

**Spec:** [Architecture v1.1](../../specs/2026-10-23-socrates-architecture.md) — §1, §12, §13, §19 and **Appendix E** (later entries override earlier ones; E-37+ record this split) — and [PRD v1.0](../../specs/2026-10-22-socrates-prd.md) `P9`, `NFR-4`, `NFR-5`, `NFR-8`, `NFR-11`, `AC-5`. Roadmap: [Stage 1 roadmap](../../plans/2026-10-23-stage1-roadmap.md).

## Global Constraints

- Java package root `org.eliooo.socrates`; modules `protocol`, `corpus`, `mind-core`, `mind-server` (R0a uses `mind-server` and the empty scaffolds of the others); Gradle Kotlin DSL; Java toolchain **25**.
- Pin every dependency (latest release at Task 1); pin every GitHub Action to a commit sha.
- **No model calls, no Anthropic SDK, no API key** in R0a.
- Ports: public **8080** (reached only via Caddy); admin connector **8081** inside the container, published on the host as `127.0.0.1:<ADMIN_HOST_PORT>`.
- Database file `<data-dir>/socrates.db` (`socrates.data-dir`, default `/data`); snapshots in `<data-dir>/snapshots/`; start-up refuses unless the marker `<data-dir>/.socrates-fs` exists.
- On the box, `/data` is a **fixed-size 2 GB filesystem**; nothing Socrates writes can fill the shared disk.
- The container never reaches the metadata service, private networks, other tenants, host services, or anything on the internet except HTTPS and its two DNS resolvers.
- **No request bodies or message text in logs**; heap dumps off.
- The public repo never contains server specifics; placeholders `<host>`, `<deploy dir>`, `<proxy container>`, `<subnet>`, `<ip range>`, `<socrates ip>`, `<NETWORK_NAME>`, `<DATA_MOUNT>`; real values live in the owner's private operations file.
- **Every step that touches the shared box runs with the owner present and approving it.**

## Review Focus

1. **The box reboots (AWS maintenance)** — every tenant and Socrates come back without anyone touching the box, with the firewall rules present before Socrates starts. *(Tasks 5, 6.)*
2. **A deploy fails half-way** (bad image, SSH drop, failed health) — the previous version is running again with its data. *(Task 4.)*
3. **A compromised image** cannot reach the metadata service, private networks, other tenants or arbitrary internet ports, cannot fill the shared disk, and is the first thing the kernel kills under memory pressure. *(Tasks 3–5.)*
4. **The other tenants** see no change: their sites answer, their memory, CPU and disk headroom remain, Caddy's routing is unchanged. *(Tasks 5, 6.)*
5. **Something goes wrong unnoticed** — disk filling, a crash loop, the watchdog stopping Socrates — and the owner is alerted. *(Tasks 2, 5, 6.)*

---

### Task 1: Repository scaffold, corpus module, CI

**Files:**
- Create: `settings.gradle.kts`, `build.gradle.kts`, `gradle/libs.versions.toml`, Gradle wrapper
- Create: `protocol/build.gradle.kts`, `corpus/build.gradle.kts`, `mind-core/build.gradle.kts`, `mind-server/build.gradle.kts`
- Create: `corpus/src/main/resources/corpus/README.md`; `mind-core/src/test/java/org/eliooo/socrates/core/ScaffoldTest.java`
- Create: `.github/workflows/ci.yml`
- Modify: `.gitignore` (add `.gradle/`, `node_modules/`, `client-chat/dist/`, `*.db`)

**Interfaces:**
- Produces: `:protocol` (java-library); `:corpus` (resources only, classpath prefix `corpus/`); `:mind-core` (java-library, `api(project(":protocol"))`, `implementation(project(":corpus"))`, `java-test-fixtures`); `:mind-server` (Spring Boot app, `implementation(project(":mind-core"))`, `testImplementation(testFixtures(project(":mind-core")))`).

- [ ] **Step 1: Write the failing tests**: `ScaffoldTest.toolchainIsJava25()`; `corpusIsOnClasspath()`.
- [ ] **Step 2: Run** `./gradlew :mind-core:test` — Expected: FAIL.
- [ ] **Step 3: Create the build**: Java 25 toolchain; JUnit 5 + AssertJ; a `live` tag excluded unless `-Plive`; version catalog pinned.
- [ ] **Step 4: Run** `./gradlew build` — Expected: BUILD SUCCESSFUL, 2 tests passed.
- [ ] **Step 5: Add `ci.yml`** (workflow name `ci`): on `push` and `pull_request`; job `build` (`permissions: contents: read`) runs `./gradlew build` on Temurin 25; `concurrency: { group: ci-${{ github.ref }}, cancel-in-progress: true }`; Actions pinned; no `pull_request_target`.
- [ ] **Step 6: Commit** — `chore: gradle multi-project scaffold, corpus module, CI`; push; CI green.

### Task 2: The minimal server — page, health, admin connector, snapshots

**Files:**
- Create: `mind-server/src/main/java/org/eliooo/socrates/server/{SocratesApplication,SocratesProperties,DataDirGuard}.java`
- Create: `mind-server/src/main/java/org/eliooo/socrates/server/db/{DatabaseConfig,DbExecutor,Migrations}.java`
- Create: `mind-server/src/main/java/org/eliooo/socrates/server/admin/{AdminConnectorConfig,PortScopeFilter,SnapshotController,HealthDetailController}.java`; `.../health/{PublicHealthController,BuildInfo,HealthSignals}.java`; `.../web/SecurityHeadersFilter.java`
- Create: `mind-server/src/main/resources/{application.yml,application-dev.yml,static/index.html,static/privacy.html,static/styles.css}`, `mind-server/src/main/resources/db/migration/V1__meta.sql`, `mind-server/src/test/resources/application-test.yml`
- Test: `mind-server/src/test/java/org/eliooo/socrates/server/{DataDirGuardTest,ApplicationContextTest,admin/AdminPortTest,admin/SnapshotControllerTest,health/HealthTest,web/SecurityHeadersTest,LoggingRulesTest}.java`

**Interfaces:**
- Produces:
  - `SocratesProperties` (`socrates.*`): `data-dir`; `static final String DB_FILE = "socrates.db"`.
  - `DataDirGuard` — start-up fails unless `<data-dir>/.socrates-fs` exists (tests create it in a temp dir).
  - `DbExecutor` (platform-thread executor of size 2, single writer connection, WAL, `busy_timeout=5000`); `Migrations.apply(DataSource)`; Flyway starter with `spring.flyway.ignore-migration-patterns=` (empty) so a newer schema fails start-up. `V1__meta.sql` creates `meta(key TEXT PRIMARY KEY, value TEXT)` (used by the snapshot round-trip tests).
  - **Second Tomcat connector on 8081** (`WebServerFactoryCustomizer`), bound to all interfaces inside the container; `PortScopeFilter` — `/healthz/detail` and `/admin/**` only on 8081; public paths 404 on 8081.
  - Public `GET /` → `index.html`: "Socrates is coming soon." with a short line on what the site will be, and a link to `privacy.html` (which states that nothing is collected yet).
  - Public `GET /healthz` → `{"status":"ok"}` or `{"status":"degraded"}` (HTTP 200 in both) from `HealthSignals`: `degraded` when `/data` is over 85% full or the container has restarted more than 3 times in the last hour (restart count from a counter file in `/data`). R0b adds the safety-reserve signal.
  - Admin `GET :8081/healthz/detail` → `{"service":"socrates-mind","sha":"<GIT_SHA>","build":"<BUILD_SHA>","dataUsedPct":<n>}`.
  - Admin `POST :8081/admin/snapshot?label=<sha>` (label `^[0-9a-f]{40}$`, else 400) → `mkdir -p <data-dir>/snapshots`; `VACUUM INTO` a temporary file, renamed to `pre-<sha>-<UTC yyyyMMdd'T'HHmmss>.db`; keep the newest 3; returns `{"file":"<name>"}`.
  - Admin `POST :8081/admin/meta?key=<k>&value=<v>` and `GET :8081/admin/meta?key=<k>` (admin port only; used by the container snapshot tests).
  - CSP exactly `default-src 'self'; connect-src 'self'; img-src 'self' data:; style-src 'self'; script-src 'self'; frame-ancestors 'none'`. `server.shutdown: graceful`; `spring.lifecycle.timeout-per-shutdown-phase: 25s`.

- [ ] **Step 1: Write the failing tests**: missing marker fails start-up; the context starts with `application-test.yml`; admin paths unreachable on 8080 and public paths 404 on 8081; snapshot label validation, distinct names for the same label, pruning to 3; `HealthTest` — `ok`, then `degraded` for data over 85% and for a restart storm; CSP exact; `LoggingRulesTest` — no request body or query value of `/admin/meta` appears in any log line.
- [ ] **Step 2: Run** `./gradlew :mind-server:test` — Expected: FAIL.
- [ ] **Step 3: Implement**.
- [ ] **Step 4: Run** — Expected: PASS.
- [ ] **Step 5: Commit** — `feat(server): coming-soon page, health, admin connector, snapshots`.

### Task 3: Hardened container image, footprint check, least-privilege publishing

**Files:**
- Create: `deploy/Dockerfile`, `deploy/run-flags.env`, `deploy/test-container.sh`
- Create: `docs/spikes/S4-footprint-r0a.md`
- Modify: `.github/workflows/ci.yml`

**Interfaces:**
- Produces:
  - Image: Temurin 25 JRE (Ubuntu-based, with `/bin/sh` and `cp`); non-root uid **10001**; `/data` created and owned by 10001; **only `/data` declared as a volume**; exec-form `ENTRYPOINT`; JVM flags `-Xmx300m -XX:MaxMetaspaceSize=128m -XX:MaxDirectMemorySize=64m -XX:ReservedCodeCacheSize=64m -Xss512k -XX:ActiveProcessorCount=1 -XX:+ExitOnOutOfMemoryError -XX:-HeapDumpOnOutOfMemoryError -Dorg.sqlite.tmpdir=/data/native`.
  - `run-flags.env` — one argument per line, **always `--flag=value`**: `--restart=unless-stopped`, `--read-only`, `--tmpfs=/tmp:size=64m`, `--user=10001:10001`, `--cap-drop=ALL`, `--security-opt=no-new-privileges`, `--pids-limit=256`, `--cpus=0.3`, `--cpu-shares=512`, `--memory=640m`, `--memory-swap=640m`, `--oom-score-adj=1000`, `--dns=1.1.1.1`, `--dns=8.8.8.8`, `--log-driver=json-file`, `--log-opt=max-size=10m`, `--log-opt=max-file=3`.
  - CI on `push` to `main`: job `build` (contents: read) builds, runs `test-container.sh`, saves the tested image as an artifact; job `publish` (`packages: write`; no Gradle or npm; `needs: build`) loads that exact image, pushes `ghcr.io/khondokertanvirhossain/hello-character-play/socrates-mind:<sha>`, uploads an artifact `deploy-ref` containing `<sha> <sha256 digest>`. (R0b adds `safety-smoke` to `publish`'s `needs`.)

- [ ] **Step 1: Write `test-container.sh`**: build; run with exactly `run-flags.env` (read line by line into an array), a temporary volume holding the marker file, `-p 127.0.0.1:18080:8080 -p 127.0.0.1:18081:8081`; assert:
  - `/` serves the page; `/healthz` ok; `/healthz/detail` has the sha;
  - `docker inspect`: `ReadonlyRootfs`, `User=10001:10001`, `CapDrop=[ALL]`, `no-new-privileges`, `PidsLimit=256`, `NanoCpus=300000000`, `Memory=671088640`, `RestartPolicy.Name=unless-stopped`, `OomScoreAdj=1000`, `Dns=[1.1.1.1 8.8.8.8]`; `Config.Volumes` only `/data`; no argv element contains a space;
  - snapshot round trip: write meta row A; `POST /admin/snapshot`; write row B; stop; restore with the Task 4 restore command; start → A present, B absent. Repeat with a **file-level** snapshot (Task 4 step 6) of an exited container;
  - with `/data` on a 64 MB loop-mounted test filesystem, filling it fails **inside** the container, `/healthz` reports `degraded`, and the host is unaffected;
  - start without the marker file → the container exits; record peak memory and CPU at idle and under 50 requests/s for 60 s, start-up time at 0.3 CPU, and image size.
- [ ] **Step 2: Run** `bash deploy/test-container.sh` — Expected: FAIL.
- [ ] **Step 3: Write the Dockerfile**.
- [ ] **Step 4: Run** — Expected: PASS; record figures in `docs/spikes/S4-footprint-r0a.md`.
- [ ] **Step 5: Update `ci.yml`** with the build/publish split; push; verify the digest artifact.
- [ ] **Step 6: Commit** — `build: hardened image, container tests, least-privilege publish with digest`.

### Task 4: The restricted deploy script

**Files:**
- Create: `deploy/deploy-socrates.sh`
- Test: `deploy/test/test-deploy.sh` (fakes for `docker`, `df`, `curl`, `sudo`, `flock`, `setsid`, `findmnt`, `/proc/meminfo`; can fail any call, kill the script, or close stdout)

**Interfaces:**
- Consumes: `run-flags.env` (Task 3); admin endpoints (Task 2); `/usr/local/sbin/socrates-fw-check` and `/etc/socrates/fw.conf` (Task 5).
- Produces: `deploy-socrates.sh`. Resolves files from its own directory; reads `<subnet>`, `<ip range>`, `<socrates ip>`, `NETWORK_NAME` **only** from `/etc/socrates/fw.conf` (root-owned, world-readable — the single source); reads `PROXY_CONTAINER`, `ADMIN_HOST_PORT`, `PROXY_DEFAULT_ROUTE`, `DATA_MOUNT`, `MAX_IMAGE_MB` (default 600) from `$DIR/deploy.conf`.
  - **Entry (forced command)**: `SSH_ORIGINAL_COMMAND` must match `^[0-9a-f]{40} sha256:[0-9a-f]{64}$` (else exit 2, nothing done); the script re-execs itself detached (`setsid nohup "$0" --run <sha> <digest> >>"$DIR/logs/deploy-<sha>.log" 2>&1 </dev/null &`) and tails the log until a `RESULT=<code>` line or 15 minutes, exiting with that code. The owner may run `--run <sha> <digest>` directly.
  - **`--run` order** (`set -euo pipefail`; `trap '' HUP INT PIPE`):
    1. `flock -n $DIR/deploy.lock` (else `RESULT=3`). Keep only the newest 20 files in `$DIR/logs`.
    2. Network: create `$NETWORK_NAME` with `--subnet <subnet> --ip-range <ip range>` (IPv4 only) if missing; existing with different IPAM, **IPv6 enabled**, or any non-Socrates endpoint holding `<socrates ip>` → `RESULT=4`. Refuse (`RESULT=7`) if `$NETWORK_NAME` no longer sorts after every network of `$PROXY_CONTAINER` (Docker 25 has no `--gw-priority`). If the proxy is not attached: `docker network connect`, then compare `docker exec $PROXY_CONTAINER ip route show default` with `PROXY_DEFAULT_ROUTE`; mismatch → disconnect, `RESULT=7`, nothing stopped.
    3. Data volume: `socrates_data` is a bind volume onto `DATA_MOUNT`, which is mounted (`findmnt`), fully allocated, and holds `.socrates-fs` — else `RESULT=10`.
    4. Firewall: `sudo -n /usr/local/sbin/socrates-fw-check` exits 0 — else `RESULT=5`, nothing stopped.
    5. Memory: `MemAvailable + (current socrates-mind usage from its cgroup, or 0) ≥ 640 MiB + 256 MiB` — else `RESULT=8`.
    6. Image: the manifest's **compressed** layer total × 4 ≤ `MAX_IMAGE_MB` (else `RESULT=9`); prune Socrates images **by repository and image id** (`docker images --digests`, including untagged), keeping the newest 2 plus running, previous and target; free disk on `/` minus `MAX_IMAGE_MB` ≥ 1 GB (else `RESULT=6`); pull **by digest**; `Config.Volumes` only `/data` (else `RESULT=9`).
    7. If a `socrates-mind` container **exists in any state**, record its image id as `previous`. Running → `POST 127.0.0.1:$ADMIN_HOST_PORT/admin/snapshot?label=<sha>`, recording the file name; not running or failed → after step 8, a **file-level snapshot** in a one-off `--network=none --cap-drop=ALL --security-opt=no-new-privileges --user=10001:10001` container from `previous`, copying `socrates.db` and `socrates.db-wal` (if any) into `snapshots/pre-<sha>-<timestamp>-file/`.
    8. Arm `trap rollback ERR EXIT`. `docker stop -t 30 socrates-mind` (if running); `docker rm socrates-mind` (if it exists).
    9. `docker run -d --name=socrates-mind <run-flags.env lines> --network=$NETWORK_NAME --ip=<socrates ip> -p 127.0.0.1:$ADMIN_HOST_PORT:8081 -v socrates_data:/data --env-file=$DIR/socrates.env -e SPRING_PROFILES_ACTIVE=prod -e SOCRATES_TRUSTED_PROXY_CIDR=<subnet> <image>@<digest>`.
    10. `docker inspect` IPAddress on `$NETWORK_NAME` equals `<socrates ip>` (else rollback); wait up to **120 s** for `/healthz/detail` sha = target → disarm the trap; clear the watchdog's stop marker (Task 5); `RESULT=0`.
  - **`rollback`** (`trap - ERR EXIT; set +e`; every step idempotent): save `docker logs` of the new container to `$DIR/logs/failed-<sha>-<timestamp>.log`; stop and remove it; if `previous` exists, restore in a one-off hardened `--network=none` container from `previous` — file snapshot: `rm -f /data/socrates.db-wal /data/socrates.db-shm && cp <file> /data/socrates.db.tmp && mv /data/socrates.db.tmp /data/socrates.db`; directory snapshot: the same plus copying its `-wal` back — then start `previous` with the same flags; `RESULT=1`. First deploy: `RESULT=1` with nothing running.
  - Re-running with the **running** sha and digest is the supported way to apply a changed `socrates.env`.

- [ ] **Step 1: Write the failing tests**: bad command → 2 with no docker calls; detached re-exec and `RESULT` relay; concurrent run → 3; IPAM mismatch, IPv6 enabled or IP held → 4; network name order or proxy route → 7; firewall → 5; data volume not on the mount / not allocated / no marker → 10; memory (500 MiB running + 900 MiB available passes; 300 MiB available and not running → 8); image size or extra volumes → 9; low disk → 6; pruning by id including untagged; pull and run by digest; no argv element with a space; exited container → file snapshot; failed health → logs saved, WAL/SHM removed, `previous` restarted; IP mismatch → rollback; script killed or stdout closed after the stop → `previous` running; `docker run` failing → `previous` running; rollback runs once; first deploy healthy → 0, unhealthy → 1 with nothing running; same-sha redeploy works; the only call touching a non-Socrates container is the idempotent proxy `network connect`.
- [ ] **Step 2: Run** `bash deploy/test/test-deploy.sh` — Expected: FAIL.
- [ ] **Step 3: Implement** (never prints environment values).
- [ ] **Step 4: Run** — Expected: all PASS.
- [ ] **Step 5: Commit** — `build: restricted, detached, digest-pinned, self-rolling-back deploy script`.

### Task 5: Host firewall, watchdog and data filesystem units

**Files:**
- Create: `deploy/host/{socrates-fw-apply,socrates-fw-check,socrates-watch,socrates-fw.service,socrates-watch.service,socrates-watch.timer,fw.conf.example,sudoers-socrates,fstab-line.example}`
- Test: `deploy/test/test-fw.sh` (fake `iptables` with an in-memory rule table); `deploy/test/test-watch.sh` (fakes for the check, `findmnt`, `docker`)

**Interfaces:**
- Produces:
  - `/etc/socrates/fw.conf` (root-owned, 644): `SUBNET`, `IP_RANGE`, `SOCRATES_IP`, `NETWORK_NAME`, `DATA_MOUNT`.
  - `socrates-fw-apply` (root, idempotent; `--expected` loads the same rules into a scratch chain `SOCRATES-EGRESS-EXPECTED`): creates chain `SOCRATES-EGRESS`, **flushes it, then appends in order**: `-m conntrack --ctstate ESTABLISHED,RELATED -j RETURN`; DNS (`udp` and `tcp` port 53) to `1.1.1.1` and `8.8.8.8` → `RETURN`; `DROP` to `169.254.0.0/16`, `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`, `100.64.0.0/10`; `-p tcp --dport 443 -m conntrack --ctstate NEW -j RETURN`; `-j DROP`. Ensures `DOCKER-USER` exists and holds exactly one `-I DOCKER-USER 1 -s <socrates ip> -j SOCRATES-EGRESS`; one `INPUT` rule `-s <socrates ip> -m conntrack --ctstate NEW -j DROP`.
  - `socrates-fw-check` (root, read-only): compares `iptables -S` of `SOCRATES-EGRESS` and the scratch chain (chain names normalised), confirms the single jump in `DOCKER-USER`, the `INPUT` rule, and that `FORWARD` jumps to `DOCKER-USER`.
  - `socrates-fw.service`: oneshot, `Before=docker.service`, runs `socrates-fw-apply` at boot.
  - `socrates-watch.timer` (every minute) → `socrates-watch`: checks the firewall (re-applies once on failure) and that `DATA_MOUNT` is mounted with its marker. **Stops Socrates only after 3 consecutive failures**, writes a stop marker (`/var/lib/socrates/watch-stopped`) and logs; **once checks pass again and the marker exists, runs `docker start socrates-mind`** and removes the marker. Never touches Docker itself or any other container.
  - `sudoers-socrates`: the deploy user may run only `/usr/local/sbin/socrates-fw-check` without a password.
  - fstab line: `<file> <DATA_MOUNT> ext4 loop,nofail,x-systemd.device-timeout=10s,x-systemd.before=docker.service,X-fstrim.notrim 0 0` — ordering only; Docker does **not** require the mount, so a failed mount never takes the tenants down.
  - **Never** use `iptables-save`, `iptables-restore` or `iptables-services` on this box.

- [ ] **Step 1: Write the failing tests**: `test-fw.sh` — apply produces exactly the ordered chain, one jump, one `INPUT` rule; apply twice changes nothing; check fails on a missing rule, a reordered chain, a duplicate jump, a missing `FORWARD → DOCKER-USER`; check passes on iptables' own rewritten text. `test-watch.sh` — one or two failures do nothing; the third stops `socrates-mind` and writes the marker; recovery with the marker starts it and removes the marker; recovery without the marker does nothing; no command ever names another container.
- [ ] **Step 2: Run** both — Expected: FAIL.
- [ ] **Step 3: Implement**.
- [ ] **Step 4: Run** — Expected: PASS.
- [ ] **Step 5: Commit** — `build: host firewall, watchdog with auto-recovery, data filesystem units`.

### Task 6: One-time server set-up, the deploy workflow, and the R0a release

**Files:**
- Create: `docs/runbook/release-0-setup.md` (placeholders only; real values in the private operations file)
- Create: `.github/workflows/deploy.yml` — **pushed only after the set-up is complete**
- Create: `docs/releases/R0a.md`; Modify: `README.md`

**Interfaces:**
- `deploy.yml`: `on: workflow_run` of `ci` (`completed`); job `gate` (no environment): success, `event == 'push'`, `head_branch == 'main'`, `head_repository.full_name == github.repository`, and the tip of `main` (API) equals `workflow_run.head_sha` → output `go`; job `deploy` (`needs: gate`, `if: go == 'true'`, `environment: production`, `concurrency: { group: production, cancel-in-progress: false }`, `timeout-minutes: 20`): downloads `deploy-ref` from that run; SSH with the restricted key, `StrictHostKeyChecking=yes`, known-hosts secret, sending `<sha> <digest>`; **only after `RESULT=0`** creates a deployment record `task: confirmed-deploy`, status `success`.

- [ ] **Step 1: Write the runbook**. Each step has a verification and an undo; **every step runs with the owner present and approving it**:
  1. **A-1** DNS record `philosopher` → host public IP; verify with `dig +short`.
  2. Choose and record (private operations file): `NETWORK_NAME` (sorting after every network the proxy is attached to, e.g. `zz-…`), `<subnet>`, `<ip range>` (upper half), `<socrates ip>` (outside the range) — after reviewing `ip route`, the VPC CIDR and route tables, every Docker network's subnet and the daemon's default address pools; a free `ADMIN_HOST_PORT` (`ss -ltnp`, tenants' compose files and scripts); `PROXY_DEFAULT_ROUTE` from `docker exec <proxy container> ip route show default`.
  3. Deploy user and files: `<deploy dir>` owned by the deploy user; `deploy-socrates.sh`, `run-flags.env`, `deploy.conf`, `logs/`; `socrates.env` (mode 600) — **empty of secrets in R0a**.
  4. **Fixed-size data filesystem**: `fallocate -l 2G` a file under a root-owned directory; `mkfs.ext4 -E nodiscard`; verify fully allocated; bare mountpoint `root:root`, mode 000, `chattr +i`; the fstab line; `findmnt --verify`; `mount -a`; `chown 10001:10001` the mounted root; create `/.socrates-fs`; `docker volume create --driver local --opt type=none --opt o=bind --opt device=<DATA_MOUNT> socrates_data`. Verify `findmnt` and that tenants are unaffected.
  5. **Firewall (A-13)**: write `/etc/socrates/fw.conf`; install scripts to `/usr/local/sbin/` (root, 0755) and units; **sudoers**: `visudo -cf ./sudoers-socrates`, then `install -m 0440 -o root -g root sudoers-socrates /etc/sudoers.d/socrates`, then `visudo -c` **with a second root shell kept open** until `sudo -n -l` as the deploy user shows only the check; `systemctl enable --now socrates-fw.service socrates-watch.timer`; verify the check exits 0.
  6. **Network**: `docker network create --subnet <subnet> --ip-range <ip range> <NETWORK_NAME>`; `docker network connect` the proxy; its default route unchanged; every other site answers.
  7. **Isolation proof** from `docker run --rm --network <NETWORK_NAME> --ip <socrates ip> --dns 1.1.1.1 curlimages/curl`: resolving `api.anthropic.com` works and `https://api.anthropic.com` is reachable; the metadata service, a tenant's published port on the host IP, a tenant container IP, the VPC resolver, `tcp/22` and `tcp/25` to a public host, and the proxy's own IP on the Socrates network on port 2019 and every port it listens on → all time out. Verify directly that the proxy's admin API listens only on loopback (`docker exec <proxy container> sh -c 'ss -ltn || netstat -ltn'`).
  8. **Deploy key**: one `authorized_keys` line `restrict,command="<deploy dir>/deploy-socrates.sh" …`; private key, host and known-hosts as `production` environment secrets; verify `ssh <host> uname` prints the exit-2 rejection.
  9. **Caddy**: back up the shared Caddyfile; append the `philosopher.eliooo.org` block; `caddy validate`; only if valid, `caddy reload`; every other site answers; still no `admin` directive.
  10. **A-15**: after the first `publish`, the owner sets the GHCR package to **public** (irreversible); verify an anonymous `docker pull …@<digest>`.
  11. **First deploy by hand**: `bash <deploy dir>/deploy-socrates.sh --run <sha> <digest>`; verify `https://philosopher.eliooo.org/` shows the page; `docker stats` (≤ 640 MiB, ≤ 0.3 CPU); every other site.
  12. **Reboot test after the first deploy** (quiet window, owner's choice): reboot the instance; then verify, without touching anything: every tenant's site, the firewall check, the data mount, **Socrates running**, the isolation probes of step 7, and no watchdog stop marker.
  13. **A-10** CPU credit mode; **A-11** uptime monitor as a **keyword check alerting when `degraded` appears** on `/healthz` (verify by briefly filling a test file on `/data` past 85%, then removing it) and the CPU-credit alarm.
  - Updating host copies later: copy, compare `sha256sum` with the repo, record in the operations file.
- [ ] **Step 2: Run the set-up** with the owner.
- [ ] **Step 3: Push `deploy.yml`**; merge a trivial change → the deployed sha on `/healthz/detail` equals its `head_sha`; push two commits quickly → only the tip deploys; merge a change that fails `test-container.sh` → nothing is published or deployed.
- [ ] **Step 4: Rollback drill** (owner present): deploy a deliberately unhealthy build (a flag that makes `/healthz/detail` report a wrong sha) → the deploy reports `RESULT=1` and the previous version is serving again with its data (meta row intact).
- [ ] **Step 5: Write `R0a.md`** — what is live, and the **owner test script**:
  1. `https://philosopher.eliooo.org/` on a phone and a laptop → the coming-soon page over HTTPS; the privacy page.
  2. Every other site on the box answers as before.
  3. `/healthz` → `ok`; the uptime monitor shows green and alerted during the `degraded` drill.
  4. The reboot test (step 12) passed.
  5. The rollback drill (step 4) passed.
  6. The isolation probes (step 7) passed before and after the reboot.
  Owner runs it; record pass/fail per item; update `README.md` (R0a live).
- [ ] **Step 6: Commit** — `ci: guarded digest deploys; release-0 runbook; R0a notes`.

---

## History

This plan and the [Release 0b plan](2026-10-23-release-0b-socrates.md) replace the single Release 0 plan, whose last version (v0.7) went through one external plan review and five multi-agent verification rounds (review logs in that file's git history). Its final external review recommended splitting Release 0 so the infrastructure is proven on its own, with nothing unsafe public until Socrates arrives; the owner agreed on 2026-10-23. Infrastructure changes from that review folded into this plan: the watchdog stops Socrates only after three consecutive failures and restarts it on recovery; the data mount is ordered before Docker without being required; a reboot test runs after the first deploy; IPv6 is refused on the Socrates network; the image-size guard uses compressed sizes with a safety factor instead of decompressing every layer.
