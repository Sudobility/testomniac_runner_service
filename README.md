# testomniac_runner_service

`@sudobility/testomniac_runner_service` — shared scanning and test-generation
engine for Testomniac, published as a TypeScript library (ESM).

Despite the `_service` suffix this is a **library**, not a standalone server.
It is imported by Testomniac runner clients (the Chrome extension and
server-side runners) and provides the browser-automation abstraction plus the
logic that turns a URL or page state into persisted test coverage and findings.

## What It Does

- Orchestrates a test run against a browser (`runTestRun`, `runSequenceRun`)
- Executes persisted test surfaces, interactions, and actions
- Analyzes resulting page states and expands discovery coverage (`PageAnalyzer`)
- Evaluates seven expertise modules (tester, SEO, security, performance,
  content, UI, accessibility) and records findings
- Abstracts the browser behind `BrowserAdapter` and persistence behind `ApiClient`

The browser runtime, HTML decomposition, actionable-item/form extraction,
scaffold detection, and login detection are provided by the
`@sudobility/webgraph_parser` dependency and re-exported from this package.

## Install

This package and its `@sudobility/*` dependencies live on the GitHub Packages
registry (see `.npmrc`), so an authenticated token with read access is required.

```bash
bun install
```

Peer dependency: `@sudobility/testomniac_types`. Runtime dependency:
`@sudobility/webgraph_parser`.

## Build / Test / Run

The toolchain uses Bun to run scripts; the build itself is plain `tsc`.

```bash
bun run build        # tsc -p tsconfig.build.json -> dist/ (js + .d.ts)
bun run typecheck    # tsc --noEmit
bun run test:unit    # vitest run  (there is no `test` script; `bun test` will NOT work)
bun run test:watch   # vitest (watch mode)
bun run lint         # eslint src/
bun run lint:fix     # eslint src/ --fix
bun run format       # prettier --write "src/**/*.ts"
bun run verify       # typecheck + lint + test:unit + build (run before publishing)
```

There is no application to "run": consumers import the package. The root
`index.ts` (`console.log("Hello via Bun!")`) is a leftover scaffold stub and is
**not** the package entry point — the real entry is `src/index.ts`
(`dist/index.js` after build).

## Main Entry Points

- `src/index.ts` — package exports (the public API surface)
- `src/orchestrator/runner.ts` — `runTestRun()`, the primary execution loop
- `src/orchestrator/sequence-runner.ts` — `runSequenceRun()` for scenario sequences
- `src/orchestrator/test-interaction-executor.ts` — `executeTestInteraction()`,
  executes a single persisted interaction
- `src/analyzer/index.ts` (+ `src/analyzer/page-analyzer/`) — `PageAnalyzer`,
  discovery-time state interpretation and follow-up generation
- `src/api/client.ts` — `ApiClient` (`scanBegin` / `scanNext` / `scanEnd`)
- `src/expertise/` — the expertise evaluation system
- `BrowserAdapter` — browser runtime contract (from `@sudobility/webgraph_parser`)

## Coverage Model

Persistent coverage is a hierarchy of
`test_surface_bundle` → `test_surface` → `test_interaction` → `test_action`,
executed through matching run records
(`test_run`, `test_surface_bundle_run`, `test_surface_run`,
`test_interaction_run`, `test_run_finding`).

Coverage is created two ways: direct navigation coverage for every discovered
same-origin path, and stateful interaction coverage that `PageAnalyzer` derives
from observed page states. The discovery-time test-interaction generators now
run server-side in `testomniac_api`, not in this package.

## Related Repos

- `testomniac_extension` — Chrome extension runner client (uses `ChromeAdapter`)
- `testomniac_runner` — server-side polling worker (Puppeteer-backed adapter)
- `testomniac_api` — persistence and run-orchestration API; hosts the server-side generators
- `testomniac_types` — shared request, response, and domain types
- `webgraph_parser` — browser adapter contract, HTML decomposition, extractors

## License

BUSL-1.1
