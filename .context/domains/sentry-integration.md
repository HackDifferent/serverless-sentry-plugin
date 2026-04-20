# Domain: Sentry Integration

## Overview

This plugin manages the deploy-time side of Sentry integration for Serverless Framework projects. It does not capture errors itself — that is the job of `serverless-sentry-lib` at runtime. This plugin: validates config, resolves the release version, injects env vars into Lambda functions, and calls the Sentry Management API.

## Key Entities

| Entity | Where defined | Key fields |
|---|---|---|
| `SentryOptions` | `src/index.ts:26` | `dsn`, `authToken`, `organization`, `project`, `release`, `enabled`, `filterLocal`, `sourceMaps`, `autoBreadcrumbs`, `captureErrors`, `captureUnhandledRejections`, `captureUncaughtException`, `captureMemoryWarnings`, `captureTimeoutWarnings` |
| `SentryRelease` | `src/index.ts:15` | `version: string \| boolean`, `refs?: { repository, commit, previousCommit }[]` |
| `ApiParameters` | `src/index.ts:68` | `authToken`, `organization`, `project?`, `refs?`, `version` |
| `FunctionDefinitionWithSentry` | `src/index.ts:63` | Extends Serverless `FunctionDefinition` with optional `sentry?: boolean \| SentryOptions` |

## API Endpoints Used

All calls go to `https://sentry.io/api/0/`. All require `Authorization: Bearer <authToken>`.

| Method | Endpoint | Purpose | Method in plugin |
|---|---|---|---|
| POST | `/organizations/{org}/releases/` | Create release | `createSentryRelease()` |
| POST | `/organizations/{org}/releases/{version}/files/` | Upload source map file | `_uploadSourceMap()` |
| POST | `/organizations/{org}/releases/{version}/deploys/` | Record deploy | `deploySentryRelease()` |

## Workflow: Full Deploy with Source Maps

```
1. validate()          — check DSN, validate config, set environment default
2. setRelease()        — resolve version string from git/random/explicit
3. instrumentFunctions() — inject SENTRY_* env vars into every Lambda function
4. [Serverless deploys Lambda functions]
5. createSentryRelease() — POST release with version + git refs to Sentry
6. uploadSentrySourcemaps() — extract .js/.js.map from artifact ZIPs, upload 50 at a time
7. deploySentryRelease() — POST deploy record to Sentry
```

Steps 5–7 are skipped if `authToken` is not configured. Steps 6–7 are skipped if `sourceMaps` is falsy.

## Business Rules

| Rule | Location | Behavior |
|---|---|---|
| `dsn` is required | `src/index.ts:99` (AJV schema) | Schema validation error if missing |
| Missing `dsn` at runtime → plugin disabled | `src/index.ts:214` | Logs warning, skips instrumentation |
| `enabled: false` → plugin disabled | `src/index.ts:218` | Logs warning, skips instrumentation |
| `authToken` without `organization`/`project` → API disabled | `src/index.ts:227` | Logs warning, clears `authToken` |
| `release: true` → git tag, fallback to short SHA | `src/index.ts:368` | `exactTag()` then `short()` |
| `release: "git"` → git required | `src/index.ts:387` | Throws if git unavailable |
| `release: "random"` → UUID | `src/index.ts:397` | `uuid()` without dashes |
| Per-function `sentry: false` → skip instrumentation | `src/index.ts:310` | Function excluded from env var injection |
| 409 on source map upload → silent skip | `src/index.ts:489` | Already uploaded, not an error |
| Release `refs[].repository === "git"` → resolve from `git remote origin` | `src/index.ts:326` | Extracts `owner/repo` from origin URL |
| GitLab repositories get spaces around slash | `src/index.ts:329` | `owner / repo` format for GitLab |

## Environment Variables Injected into Lambda

See `standards/naming-conventions.md` for the full mapping. All env vars are string-coerced from their config values.

## Gotchas

- `release.version` type is `string | boolean`. Always `String(version)` before use — do not assume it's a string even if the user passes a string-looking value.
- `validate()` runs multiple times (different hooks call it). The `this.validated` guard is essential.
- `instrumentFunctions()` also runs multiple times. The `this.isInstrumented` guard prevents double-injection, but is bypassed when `setEnv=true` (local invoke needs `process.env` set).
- `deploySentryRelease()` only fires if `sourceMaps` is truthy — even though you might expect it to fire for any release. This is by design (a deploy record in Sentry is tied to source map context).
- The `refs` array on `SentryRelease` uses sentinel strings `"git"` and `"git"` for repository and commit — these are resolved by `_resolveGitRefs()`. Do not pass literal `"git"` expecting it to pass through.
