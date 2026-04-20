# Testing

## Current State: No Automated Tests

`package.json` `scripts.test` is `exit 0`. There are no test files in this repository. This is tracked as technical debt in ADR-006.

## Manual Testing

The plugin must be tested against a real Serverless project:

1. `npm run build` to compile `dist/`
2. Link the plugin locally: `npm link` in this repo, then `npm link serverless-sentry` in a test project
3. Configure `custom.sentry` in the test project's `serverless.yml`
4. Run `serverless package`, `serverless deploy`, or `serverless invoke local`

Key scenarios to verify manually:
- `dsn` missing → plugin disabled without error
- `enabled: false` → plugin disabled without error
- `release: true` → version resolved from git tag or short SHA
- `release: "git"` → fails if no git available
- `release: "random"` → UUID assigned
- `sourceMaps: true` → `.js`/`.js.map` files uploaded from artifact ZIP
- Per-function `sentry: false` → function skipped in instrumentation
- Per-function `sentry: { ... }` → overrides applied

## Adding Tests

If automated tests are added, recommended approach:
- Jest with `ts-jest` (consistent with TypeScript project)
- Mock the Serverless Framework instance (pass a minimal mock object to `new SentryPlugin(mockServerless, options)`)
- Mock `superagent` to capture API calls without network
- Mock `child_process.exec` via `jest.mock` for `GitRev`
- Test naming: `*.test.ts` co-located in `src/` or in a top-level `test/` directory
