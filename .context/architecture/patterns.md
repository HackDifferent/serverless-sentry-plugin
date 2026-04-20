# Architecture Patterns

## Primary Pattern: Serverless Plugin

The `SentryPlugin` class implements the Serverless Framework plugin interface:

```typescript
export class SentryPlugin implements Plugin {
  hooks: { [event: string]: (...rest: any[]) => any };
  // ...
  constructor(serverless: Serverless, options: Serverless.Options) {
    this.hooks = {
      "after:deploy:deploy": async () => { ... },
      // ...
    };
  }
}
module.exports = SentryPlugin;  // Must be CommonJS export
```

Key constraint: `module.exports`, not `export default`. Serverless Framework requires CommonJS plugin exports.

## State Machine via Instance Flags

Two boolean flags prevent double-execution across repeated hook firings:

- `this.validated` — set to `true` after first successful `validate()`. Subsequent calls return early.
- `this.isInstrumented` — set to `true` after `instrumentFunctions()`. Prevents re-instrumenting unless `setEnv=true`.

This matters because multiple hooks (e.g., `before:package:initialize` and `before:deploy:deploy`) both call `validate()`.

## Config Merging Pattern

Per-function Sentry overrides merge into the global config:

```typescript
instrumentFunction(originalDefinition, setEnv) {
  const sentryConfig = { ...this.sentry };          // Start with global config
  const localConfig = newDefinition.sentry;
  if (typeof localConfig === "object") {
    Object.assign(sentryConfig, localConfig);        // Override with per-function config
  }
  // Then inject env vars from merged sentryConfig
}
```

Functions can set `sentry: false` to opt out entirely — checked as `(functionObject.sentry ?? true) !== false`.

## Version Resolution Chain (setRelease)

```
release config value
       │
       ├── string → wrap as { version: string }
       ├── false / undefined → disabled, return
       ├── true / "true" / "git" → resolve from git
       │       ├── exactTag() → short SHA fallback
       │       └── git unavailable:
       │               ├── "git" → throw
       │               └── true → random UUID
       ├── "random" → UUID
       └── other string → use as-is
```

## Parallel Upload Pattern

Source maps use `PromisePool` to avoid unbounded concurrency:

```typescript
await PromisePool.withConcurrency(50)
  .for(results)
  .process(async (nextArtifact) => await nextArtifact());
```

`results` is an array of `() => Promise<void>` closures — deferred so they only execute inside the pool.

## GitRev Shell-Out Pattern

`GitRev` wraps `child_process.exec` for each git command:

```typescript
async _command(cmd: string): Promise<string> {
  return new Promise((resolve, reject) => {
    exec(cmd, this.opts, function (err, stdout) {
      return err ? reject(err) : resolve(stdout.replace(/\n/g, "").trim());
    });
  });
}
```

Each public method (`short()`, `long()`, `tag()`, `exactTag()`, `origin()`) delegates to `_command()`. `exactTag()` catches errors and returns `undefined` — callers should not treat a missing tag as fatal.
