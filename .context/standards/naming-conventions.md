# Naming Conventions

## Classes

| Role | Convention | Example |
|---|---|---|
| Plugin entry point | `<Name>Plugin` | `SentryPlugin` |
| Utility helper | Descriptive noun | `GitRev` |
| Exported types | PascalCase | `SentryOptions`, `SentryRelease`, `ApiParameters` |

## Methods

- Public lifecycle methods: camelCase verb phrases — `validate()`, `setRelease()`, `instrumentFunctions()`, `createSentryRelease()`, `uploadSentrySourcemaps()`, `deploySentryRelease()`
- Private/internal methods: prefixed with `_` — `_apiParameters()`, `_uploadSourceMap()`, `_resolveGitRefs()`
- Private helpers on `GitRev`: prefixed with `_` — `_command()`

## Types

- `SentryOptions` — all plugin configuration fields
- `SentryRelease` — shape of the `release` config object
- `ApiParameters` — internal parameters extracted before API calls
- `FunctionDefinitionWithSentry` — extends Serverless `FunctionDefinition` with optional `sentry` field

## Environment Variable Names (injected into Lambda)

All uppercase with `SENTRY_` prefix:

| Field | Env Var |
|---|---|
| `dsn` | `SENTRY_DSN` |
| `release.version` | `SENTRY_RELEASE` |
| `environment` | `SENTRY_ENVIRONMENT` |
| `autoBreadcrumbs` | `SENTRY_AUTO_BREADCRUMBS` |
| `sourceMaps` | `SENTRY_SOURCEMAPS` |
| `filterLocal` | `SENTRY_FILTER_LOCAL` |
| `captureErrors` | `SENTRY_CAPTURE_ERRORS` |
| `captureUnhandledRejections` | `SENTRY_CAPTURE_UNHANDLED` |
| `captureUncaughtException` | `SENTRY_CAPTURE_UNCAUGHT` |
| `captureMemoryWarnings` | `SENTRY_CAPTURE_MEMORY` |
| `captureTimeoutWarnings` | `SENTRY_CAPTURE_TIMEOUTS` |

## Files

| File | Purpose |
|---|---|
| `src/index.ts` | Plugin entry point — `SentryPlugin` class |
| `src/git-rev.ts` | Git command helper — `GitRev` class |
| `dist/index.js` | Compiled output — what npm consumers use |
| `tsconfig.json` | TypeScript config for development |
| `tsconfig.release.json` | TypeScript config for `npm run build` |

## Serverless YAML Config Key

The plugin reads `custom.sentry` in `serverless.yml`. Schema is registered via `configSchemaHandler.defineCustomProperties()` in the constructor.
