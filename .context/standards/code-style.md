# Code Style

## Formatter: Prettier

Config lives in `package.json` under `"prettier"`:

```json
{
  "printWidth": 120,
  "tabWidth": 2,
  "useTabs": false,
  "semi": true,
  "singleQuote": false,
  "trailingComma": "all",
  "bracketSpacing": true,
  "arrowParens": "always"
}
```

Enforced via `eslint-plugin-prettier`. Pre-commit hook (`husky` + `lint-staged`) auto-formats `src/**/*.{js,jsx,ts,tsx}`.

## Linter: ESLint + TypeScript

- `@typescript-eslint/eslint-plugin` + `@typescript-eslint/parser`
- `eslint-plugin-prettier` — prettier violations are ESLint errors
- `eslint-plugin-promise` — promise best practices

Run: `npm run lint` (also type-checks via `tsc --noemit`).

## TypeScript Compiler Options

From `tsconfig.json`:
- `target: "es5"`, `module: "commonjs"` — broad Node.js >=12 compatibility
- `noImplicitAny: true` — no implicit `any`
- `strictNullChecks: true` — nullability must be explicit
- `outDir: "./dist"`, `removeComments: false`

**Never use `as any`** — narrow types properly. Use type guards or explicit casts with a comment explaining why.

## Import Style

`prettier-plugin-import-sort` auto-sorts imports using the `module` style. Standard pattern from `src/index.ts`:

```typescript
import * as path from "path";

import { PromisePool } from "@supercharge/promise-pool";
import * as AdmZip from "adm-zip";
// ... other third-party
import Serverless from "serverless";

import GitRev from "./git-rev";
```

Node built-ins → third-party → local (separated by blank lines, alphabetical within groups).

## Null/Undefined Handling

Use optional chaining and nullish coalescing. Examples from `src/index.ts`:

```typescript
// Optional chaining into response error
(err as request.ResponseError)?.response?.text

// Nullish coalescing for defaults
this.options.stage ?? undefined

// Explicit undefined assignment over delete
this.sentry.authToken = undefined;
```

Never throw on missing optional config — log a warning and disable the feature.

## Error Handling

- Wrap Sentry API calls in try/catch; log `response.text` before re-throwing.
- Re-throw as `new Error("Sentry: <context> - " + err.toString())` — always prefix with `"Sentry: "`.
- 409 responses from source map upload are silently skipped (not an error).

## Logging

Use `this.serverless.cli.log(message, "sentry")` — always pass `"sentry"` as the prefix. Verbose logs gated by `process.env.SLS_DEBUG`.

```typescript
this.serverless.cli.log("Creating new release...", "sentry");
process.env.SLS_DEBUG && this.serverless.cli.log("Verbose message", "sentry");
```

## Class Structure

`SentryPlugin` follows Serverless Plugin interface:

```typescript
export class SentryPlugin implements Plugin {
  // Public state properties first
  sentry: Partial<SentryOptions>;
  serverless: Serverless;
  options: Serverless.Options;
  custom: Service.Custom;
  hooks: { [event: string]: (...rest: any[]) => any };
  provider: Aws;
  validated: boolean;
  isInstrumented: boolean;

  constructor(serverless: Serverless, options: Serverless.Options) { ... }

  // Public methods follow
  configPlugin(): void { ... }
  async validate(): Promise<void> { ... }
  // ...

  // Private/internal methods last, prefixed with _
  _apiParameters(): ApiParameters | undefined { ... }
  async _uploadSourceMap(...): Promise<void> { ... }
  async _resolveGitRefs(...): Promise<SentryRelease> { ... }
}

module.exports = SentryPlugin;  // CommonJS export for Serverless Framework
```

Note: `module.exports` (not `export default`) — required by the Serverless plugin system.
