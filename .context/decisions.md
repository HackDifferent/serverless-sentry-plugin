# Architecture Decision Records (ADRs)

## Template

```markdown
## ADR-[NUMBER]: [TITLE]

**Date**: [YYYY-MM-DD]
**Status**: [Proposed | Accepted | Deprecated | Superseded by ADR-XXX]

### Context
### Decision
### Consequences
### Alternatives Considered
```

---

## ADR-001: TypeScript with Strict Null Checks

**Date**: ~2020
**Status**: Accepted

### Context
Plugin originally written in JavaScript; migrated to TypeScript for safety.

### Decision
Use TypeScript with `noImplicitAny: true` and `strictNullChecks: true` (see `tsconfig.json`). Compile target is ES5 CommonJS for broad Node.js compatibility.

### Consequences
- All optional Sentry config fields use `Partial<SentryOptions>` throughout.
- `release.version` type is `string | boolean` — code must narrow before use.
- `dist/` is committed to the repo and included in npm publish (no build step for consumers).

---

## ADR-002: Single-Class Plugin Architecture

**Date**: ~2020
**Status**: Accepted

### Context
Serverless plugins must export a class with a `hooks` object.

### Decision
All logic lives in one `SentryPlugin` class in `src/index.ts`. The only helper class is `GitRev` in `src/git-rev.ts`. No service layer, no DI container.

### Consequences
- Easy to navigate — everything is in one file.
- `SentryPlugin` carries all state as instance properties (`this.sentry`, `this.validated`, `this.isInstrumented`).
- Adding features means adding methods directly to `SentryPlugin`.

---

## ADR-003: dist/ Committed to Git

**Date**: ~2020
**Status**: Accepted

### Context
npm consumers need `dist/` at install time without running a build step.

### Decision
`dist/` is git-tracked and updated as part of the version release process (see `scripts.preversion` in `package.json`).

### Consequences
- `git add dist/` is part of the version bump workflow.
- PRs that change `src/` should also update `dist/` or note it's intentional.
- Do not gitignore `dist/`.

---

## ADR-004: Sentry API via superagent (Not Official SDK)

**Date**: ~2021
**Status**: Accepted

### Context
Sentry provides a management API but no official Node.js management SDK.

### Decision
Use `superagent` for direct REST calls to `https://sentry.io/api/0/`. The Sentry DSN-based SDK is not used in the plugin itself (only in the runtime lib).

### Consequences
- API endpoints are hardcoded. Changes to Sentry API require manual updates.
- Error handling inspects `response.text` from `superagent` `ResponseError`.

---

## ADR-005: Source Map Upload Concurrency at 50

**Date**: 2022 (PR #53)
**Status**: Accepted

### Context
Lambda deployment ZIPs can contain many `.js`/`.js.map` files. Sequential upload is slow.

### Decision
Use `@supercharge/promise-pool` with concurrency 50 — matching the AWS JavaScript SDK default for parallel socket connections.

### Consequences
- `PromisePool.withConcurrency(50)` in `uploadSentrySourcemaps()`.
- 409 (already uploaded) responses are silently skipped.

---

## ADR-006: No Automated Tests

**Date**: ongoing
**Status**: Accepted (technical debt)

### Context
`scripts.test` is `exit 0`. No test files exist.

### Decision
Testing is done manually against real Serverless projects.

### Consequences
- `npm test` always passes — do not rely on it.
- Changes to `instrumentFunction()` or API calls require manual validation.
- Adding tests would require mocking the Serverless Framework internals.

---

## Decision Log Summary

| ADR | Title | Status |
|-----|-------|--------|
| 001 | TypeScript with Strict Null Checks | Accepted |
| 002 | Single-Class Plugin Architecture | Accepted |
| 003 | dist/ Committed to Git | Accepted |
| 004 | Sentry API via superagent | Accepted |
| 005 | Source Map Upload Concurrency at 50 | Accepted |
| 006 | No Automated Tests | Accepted (tech debt) |
