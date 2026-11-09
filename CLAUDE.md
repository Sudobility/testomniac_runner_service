# Testomniac Runner Service

> **Git policy — never auto-commit or auto-push.** Leave work in the working tree.
> Run `git commit`, `git push`, `gh pr create`, or `scripts/push_all.sh` **only when
> the user explicitly asks in that turn.** Approval for an earlier change does not carry
> forward, and finishing a task is not permission to commit it.
> Do not use feature branches — stay on the current branch.

`@sudobility/testomniac_runner_service` is a TypeScript **library** (ESM, not a
server despite the name). It is the shared execution + discovery engine imported
by Testomniac runner clients. Public API: `src/index.ts` → `dist/index.js`.

## Stack & Commands

- Bun runs scripts and installs deps; the build is plain `tsc`. Tests: vitest.
- `@sudobility/*` deps resolve from GitHub Packages (`.npmrc`) — install needs an
  authed read token.

```bash
bun install
bun run build       # tsc -p tsconfig.build.json -> dist/
bun run typecheck   # tsc --noEmit
bun run test:unit   # vitest run
bun run lint        # eslint src/
bun run verify      # typecheck + lint + test:unit + build (pre-publish gate)
```

**Gotcha:** there is no `test` script. `bun run test` fails and `bun test` runs
Bun's own runner (not vitest). Always use `bun run test:unit` / `test:watch`.

## webgraph_parser boundary (important)

`BrowserAdapter`, HTML decomposition (`decomposeHtml`), actionable-item/form
extraction, scaffold detection (`detectScaffoldRegions`), login detection
(`detectLoginPage`), and hashing all come from `@sudobility/webgraph_parser` and
are re-exported by `src/index.ts`. There is **no** local `src/adapter.ts`,
`src/extractors/`, `src/scanner/component-detector.ts`,
`src/scanner/pattern-detector.ts`, or `src/scanner/login-detector.ts` — do not
look for them here. The runner never touches Puppeteer/Playwright/Chrome APIs
directly; it only calls `BrowserAdapter`.

## Core model & terminology

Persistent coverage: `test_surface_bundle` → `test_surface` → `test_interaction`
→ `test_action` (+ `test_step`). Run records: `test_run`,
`test_surface_bundle_run`, `test_surface_run`, `test_interaction_run`,
`test_run_finding`. The code uses **interaction**, not "element" or "case" —
`executeTestInteraction`, `TestInteractionResponse`, etc.

## Execution flow

1. A client calls `runTestRun()` (`src/orchestrator/runner.ts`).
2. Runner claims the `test_run`, loads the active bundle run, iterates pending
   surface runs, then pending interaction runs whose dependencies are ready.
3. `executeTestInteraction()` (`src/orchestrator/test-interaction-executor.ts`)
   navigates, replays dependency setup, executes actions, gathers runtime
   artifacts, and runs expertises.
4. It calls `api.scanNext()` — one call that completes the interaction, persists
   page state, runs generators server-side, persists findings, and returns the
   next interaction. `scanBegin()` starts the scan; `scanEnd()` triggers persona
   + scenario detection.
5. Surface runs, bundle run, then root run are completed.

`runSequenceRun()` (`src/orchestrator/sequence-runner.ts`) is a second entry that
replays an ordered interaction list for a scenario, reusing the same executor.

## Scan lifecycle & generators (both server-side now)

The executor makes a single `scanNext()` call per interaction instead of many
sequential API calls. The old client methods (`ensurePageStateCombined`,
`generateSurfaceInteractions`, `completeInteractionRunCombined`,
`detectPersonas`, etc.) are gone. The 12 discovery-time test-interaction
generators moved to `testomniac_api`'s `src/generators/` (run inside
`/scan/next`); `src/analyzer/page-analyzer/generators/` was deleted here.

## Replaying dependency setup (learned by breaking it)

- **Replay hover chains in full.** A hover opening a popup menu is the setup for
  the hover onto an item inside it; skipping the opener removes what the second
  step needs. Full model is in `testomniac_api`'s CLAUDE.md ("How a hover becomes
  interactions") — read it before optimising step counts here.
- **A failing setup step is expected — fail it fast.** Setup replays against a
  moved-on page; a missing control is caught, logged as skipped, stepped over.
  `SETUP_REPLAY_TIMEOUT_MS` is 1s (not the adapter's 5s): a control already
  rendered is there or it is not. Removing "already-failing" steps is not free —
  the work they were meant to do simply stops happening.

## Discovery rules

- Every actionable element starts with a hover interaction.
- After hover: no new actionable items → dependent click interaction; new items
  → dependent hover interactions from the hover target page state.
- `PageAnalyzer` (`src/analyzer/index.ts`, ~573 lines) owns target-page-state
  creation and follow-up generation; it does finding dedup, expectation
  generation, and hover-to-click only. Structural decomposition is deterministic,
  not AI (the old `src/ai/` dir is deleted; AI generation is server-side).
- Expertises evaluate observed results; they do **not** generate coverage.

## Expertise system (`src/expertise/`)

`createDefaultExpertises()` builds seven; `tester` is always included even if
not requested. Each is one file in `src/expertise/` (`*-expertise.ts`):

- `tester` — 37+ expectation types via checker modules in `src/expertise/tester/`
  (page load, console/network, form validation, input values, selection,
  commerce, network intent, search, navigation, dialog, feedback, keyboard,
  persistence, list).
- `content`, `ui`, `security`, `performance`, `seo`, `accessibility` — page-level
  quality/correctness checks.
- `src/scanner/page-health-evaluator.ts` runs browser-side DOM checks via
  `adapter.evaluate()` during each interaction (broken images, element overlap,
  cart math, grammar/count mismatches, empty/broken links, missing `noopener`,
  etc.).

## Login & scope

- `LoginManager` (`src/orchestrator/login-manager.ts`): state machine; while in a
  login flow it suspends the scope boundary. Max 3 re-login attempts.
- `src/orchestrator/sso-handler.ts`: provider flows (Google, Microsoft, GitHub,
  Facebook, Apple, Twitter, LinkedIn, Okta, SAML) with a generic fallback.
- `isWithinScopePath()` (`src/crawler/scope-checker.ts`): same-origin + path
  prefix; enforced at link extraction, pre-execution, and generation; suspended
  during login via `LoginManager.isInLoginFlow()`.

## Removed / not here

- `runScan()` legacy orchestration — removed; `runTestRun()` is the entry point.
- `src/ai/`, `src/analyzer/page-analyzer/generators/` — deleted.
- Dropped deps: `openai`, `react`, `zustand`.

## Notes

- `docs/ARCHITECTURE.md` and `docs/HOVER.md` are partly stale (old
  `test_case`/`executeTestCase`/`src/adapter.ts` terminology) — trust code and
  this file over them.
- CI (`.github/workflows/ci-cd.yml`) calls a reusable workflow and publishes to
  GitHub Packages on push to `main`/`develop` when secrets are set.
