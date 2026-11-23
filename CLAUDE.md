# Mallakhamb — Spring Boot migration repo

This repository exists for one job: **port the Mallakhamb backend from
Node/Express/Mongoose (`Server/`) to Spring Boot (`mallakhamb-backend/`)**,
keeping MongoDB, the React app and every wire contract unchanged.

Production still runs from the old repo,
`github.com/angreatharva/mallakhamb` (local clone `D:\Mallakhamb-Repo\Mallakhamb-master`).
Nothing in this repo deploys anywhere.

## Read first

1. [mallakhamb-backend/TRACKER.md](mallakhamb-backend/TRACKER.md): **the phase-wise task list.** Pick up the `[~]` task, tick it when done, and add a Log row.
2. [mallakhamb-backend/MIGRATION-STATUS.md](mallakhamb-backend/MIGRATION-STATUS.md): open decisions, findings, drift, blockers.
3. [docs/springboot-migration/README.md](docs/springboot-migration/README.md): the plan (revision 4) and decisions D1–D9.
4. [docs/springboot-migration/06-ROADMAP.md](docs/springboot-migration/06-ROADMAP.md): stage detail, gates, and the per-module checklist.

## Layout

| Path | Role here |
|---|---|
| `mallakhamb-backend/` | **The product.** Gradle (Groovy) project, Spring Boot 4.1.2, Java 25, War run with `java -jar`. Group `com.mallakhamb`. |
| `mallakhamb-backend/tools/node/` | Port kit, written in Node: generators (`gen`), golden engine vectors (`golden`), the twin-run recorder and replayer (`twinrun`). It `require`s code from `Server/src`. |
| `docs/springboot-migration/` | The plan, ADRs, API matrix, and drafts of the agent kit (skills and agents move into `.claude/` in Stage 2). |
| `Server/` | **The reference Node backend, read-only.** It is a snapshot of old-repo `master` at `1ccade8` (2026-11-23). It is the oracle for the golden vectors and the twin-run. Do not fix bugs here. Fixes go to the old repo and are re-synced (see below). `Server/MEMORY/` is the agent map of it. |
| `Web/` | React SPA, snapshot from the same day. Out of scope: its contract must not change. `Web/MEMORY/` maps it. |
| `deploy/`, `ocr/`, `render.yaml` | Production ops and the OCR sidecar, kept for reference (cutover runbook, systemd, Caddy). Read-only here. |

## Branches

- `master`: the migration's lead branch (it replaces the old repo's `migration/springboot`).
- `stage/<n>-<name>`: one per stage, merged into `master` when the stage gate passes.
- `module/<name>`: one per ported module, a normal feature branch.
- **No git worktrees** (owner, 2026-11-23). All work happens in this checkout, on `master` or a feature branch. Modules are ported one branch at a time, not in parallel worktrees.
- Old-repo branches `docs/springboot-migration-plan`, `migration/springboot` and `migration/stage-1-discovery` are imported here and frozen. Don't add to them.

## Rules that don't bend

- **MongoDB stays** (D1). Spring Data MongoDB + `MongoTemplate`, no JPA. Same collections, fields, indexes and `__v`.
- **Wire values stay byte-identical**: the `{ success, data | error }` envelope, `/api/v1/...` and `/api/...` routes, per-role cookies, CSRF, and Socket.IO events and rooms.
- **The engines must match the golden vectors exactly** (`tools/node/golden`). JS number semantics apply (`JsNumber`). Finals use `toFixed(3)`.
- **Before 21 Dec 2026 nothing shared changes** (D2). No VM, CI, deploy, real database or push to the old repo. The December meets (12–13 and 19–20 Dec) run on Node.
- The twin-run only talks to `mongodb://127.0.0.1/twin_test_*`. `Server/.env` points at real Atlas data: never use it from tools.
- Decide local, reversible choices yourself and record them in `MIGRATION-STATUS.md`. Pushing, deploying, VMs, real data and anything that changes a user flow need the owner.

## Commands

```bash
cd mallakhamb-backend && ./gradlew build                 # JDK 25 (Temurin 25.0.5) via toolchain
cd Server && npm ci                                       # needed by the port kit
cd mallakhamb-backend/tools/node && npm ci && npm test    # generators + twin-run tests
npm run gen            # vocabulary / catalogue reports -> mallakhamb-backend/generated/
npm run golden:verify  # committed vectors still match Node
npm run golden:full    # rebuild the 1M-case set, check hashes
npm run twin:replay    # replay a recorded corpus (Stage 2: --url at Java)
```

## Re-syncing the Node reference

When the old repo's `master` moves (December fixes, for example), copy `Server/`
and `Web/` across from it in one commit named `Sync Node reference to <sha>`,
update the sha above and in `MIGRATION-STATUS.md`, then re-run `npm run gen`,
the golden vectors and a twin-run replay. Any difference is a finding.
