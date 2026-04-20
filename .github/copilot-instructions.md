# serverless-sentry Plugin

## What This Is

A [Serverless Framework](https://serverless.com) plugin that integrates [Sentry](https://sentry.io) into AWS Lambda deployments. At deploy time it: injects Sentry env vars into Lambda functions, creates a Sentry release, uploads source maps, and records a deploy. It does **not** capture errors at runtime — that is `serverless-sentry-lib` (a peer dependency, installed by users separately).

## Tech Stack

- **Language**: TypeScript 4.x, compiles to ES5 CommonJS
- **Runtime target**: Node.js >=12
- **Framework**: Serverless Framework plugin interface (>=2)
- **HTTP client**: `superagent` (Sentry Management API)
- **Build**: `tsc --build tsconfig.release.json` → `dist/`
- **Linting**: ESLint + `@typescript-eslint` + Prettier (120 char line width, double quotes, 2-space indent)
- **Pre-commit**: husky + lint-staged (format and lint `src/**/*.ts` before commit)
- **Package manager**: npm

## Key Commands

```bash
npm run build        # Compile TypeScript → dist/
npm run lint         # Type check + ESLint/Prettier
npm run prepare      # Install husky hooks + build (runs on npm install)
npm version patch    # Bump version, build, commit dist/, push + push tags
npm publish          # Publish to npm (runs lint + test first)
```

## Structure

```
src/index.ts     # SentryPlugin class — all plugin logic
src/git-rev.ts   # GitRev — git shell command helpers
dist/            # Compiled output — committed, included in npm package
```

## High-Level Conventions

- All plugin logic is in `SentryPlugin` class; `module.exports = SentryPlugin` (CommonJS required by Serverless)
- `dist/` is committed and must be rebuilt before version bumps
- No automated tests — `npm test` is `exit 0`
- Error messages always prefixed: `"Sentry: <context> - " + err.toString()`
- Logging via `this.serverless.cli.log(msg, "sentry")`

## Detailed Documentation

See `.context/` for full architectural patterns, domain rules, and coding standards.
