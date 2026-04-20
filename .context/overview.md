# Architecture Overview

## Purpose of This Documentation

This `.context` directory provides comprehensive project-specific knowledge that enables AI agents and human developers to work effectively with this codebase.

---

## What This System Does

`serverless-sentry` is a [Serverless Framework](https://serverless.com) plugin that integrates [Sentry](https://sentry.io) error monitoring into AWS Lambda functions. It operates at deploy time (not runtime), injecting Sentry configuration as Lambda environment variables and calling the Sentry API to register releases and upload source maps.

The plugin has two sister projects:
- **This plugin** (`serverless-sentry`) — deploy-time instrumentation, release management, source map uploads.
- **[serverless-sentry-lib](https://github.com/arabold/serverless-sentry-lib)** — runtime error capture (wraps Lambda handler with `withSentry`). This is a peer dependency; do not confuse the two.

## Core Business Concepts

| Concept | Definition |
|---|---|
| **Release** | A Sentry release tied to a version string (from git tag, short SHA, `random`, or explicit). Created via Sentry API. |
| **Instrumentation** | Setting Lambda env vars (`SENTRY_DSN`, `SENTRY_RELEASE`, `SENTRY_ENVIRONMENT`, etc.) so the runtime lib reads them without hardcoding. |
| **Source Maps** | `.js` and `.js.map` files extracted from Lambda deployment ZIPs and uploaded to Sentry for readable stack traces. |
| **Deploy** | Sentry-side record of a deployment, created after source maps upload. |

## System Architecture

```
serverless.yml (custom.sentry config)
        │
        ▼
SentryPlugin (src/index.ts)   ←── Serverless Framework lifecycle hooks
        │
        ├── validate()          — reads config, sets defaults
        ├── setRelease()        — resolves version (git tag / SHA / random)
        │       └── GitRev (src/git-rev.ts) — shells out to git commands
        ├── instrumentFunctions() — injects SENTRY_* env vars into each Lambda
        ├── createSentryRelease() — POST to Sentry API
        ├── uploadSentrySourcemaps() — reads .zip artifacts, uploads .js/.js.map (50 parallel)
        └── deploySentryRelease()  — POST deploy record to Sentry API
```

## Module Structure

```
src/
├── index.ts       # SentryPlugin class — all plugin logic
└── git-rev.ts     # GitRev helper — shells out git commands

dist/              # Compiled output (tsc → ES5 CommonJS); committed for npm publish
```

## Lifecycle Hooks Wired

| Hook | Actions |
|---|---|
| `before:package:initialize` | `validate()` |
| `after:package:initialize` | `setRelease()`, `instrumentFunctions()` |
| `before:deploy:deploy` | `validate()`, `setRelease()`, `instrumentFunctions()` |
| `after:deploy:deploy` | `createSentryRelease()`, `uploadSentrySourcemaps()`, `deploySentryRelease()` |
| `before:deploy:function:deploy` | same as deploy |
| `after:deploy:function:deploy` | same as after deploy |
| `before:invoke:local:invoke` | validate, setRelease, `instrumentFunctions(true)` (also sets `process.env`) |
| `before:offline:start` / `before:offline:start:init` | validate, setRelease, instrumentFunctions |

## Key External Dependencies

| Dependency | Version | Purpose |
|---|---|---|
| `serverless` | >=2 (peer) | Plugin host framework |
| `serverless-sentry-lib` | 2.x.x (peer) | Runtime companion — users install separately |
| `@supercharge/promise-pool` | ^2.3.0 | Concurrency-limited parallel uploads (50 at a time) |
| `adm-zip` | ^0.5.9 | Read Lambda deployment ZIP files to extract source maps |
| `superagent` | ^7.1.6 | HTTP client for Sentry API calls |
| `semver` | ^7.3.2 | Validates minimum Serverless framework version (>=1.12.0) |
| `uuid` | ^8.0.0 | Generates random release versions when git is unavailable |

## Deployment Target

This is an **npm package**, not a deployed service. Published as `serverless-sentry` on npm. The compiled `dist/` directory is included in the published package alongside `package.json` and `README.md`.

## Entry Points for New Feature Work

- **New Sentry config option**: add to `SentryOptions` type (`src/index.ts:26`), update AJV schema in `constructor` (`src/index.ts:96`), inject env var in `instrumentFunction()` (`src/index.ts:237`).
- **New API interaction**: add a method on `SentryPlugin`, wire into the appropriate hook in `this.hooks`.
- **New git metadata**: extend `GitRev` (`src/git-rev.ts`) with new `git` shell command wrappers.
