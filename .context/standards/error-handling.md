# Error Handling

## Pattern

All Sentry API calls are wrapped in try/catch. Errors are enriched with context and re-thrown. The caller (Serverless Framework) surfaces them to the user.

```typescript
try {
  await request.post(...).send(payload);
} catch (err) {
  if ((err as request.ResponseError)?.response?.text) {
    this.serverless.cli.log(
      `Received error response from Sentry:\n${String((err as request.ResponseError)?.response?.text)}`,
      "sentry",
    );
  }
  throw new Error(`Sentry: Error uploading release - ${(err as Error).toString()}`);
}
```

## Error Message Format

Always: `"Sentry: <verb phrase> - " + err.toString()`

Examples from `src/index.ts`:
- `"Sentry: Error uploading release - ..."`
- `"Sentry: Error uploading sourcemap file - ..."`
- `"Sentry: Error deploying release - ..."`
- `"Sentry: No Git available - ..."` (when `release: "git"` and git is missing)

## Soft Failures (Log + Continue)

Some conditions log a warning and disable the feature rather than throwing:

| Condition | Behavior |
|---|---|
| No `dsn` configured | `cli.log("DSN not set. Serverless Sentry plugin is disabled.")` — continues without error |
| `enabled: false` | `cli.log("Serverless Sentry is disabled from provided options.")` |
| `authToken` set but missing `organization`/`project` | Logs warning, sets `authToken = undefined` (disables API calls) |
| Git unavailable when `release: true` | Falls back to random UUID version |
| Git unavailable when `release: "git"` | Throws — explicit "git" means git is required |
| 409 on source map upload | Silently skipped — file already uploaded |

## Type Casting for superagent Errors

`superagent` errors have a `response` property that TypeScript doesn't know about. Cast explicitly:

```typescript
(err as request.ResponseError)?.response?.status   // HTTP status code
(err as request.ResponseError)?.response?.text      // Raw response body
```

## Validation Errors

Configuration validation uses the Serverless `configSchemaHandler` (AJV schema). Schema errors are surfaced by the framework before any plugin code runs.
