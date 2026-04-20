# CI/CD

## Local Pre-commit Hook

`husky` + `lint-staged` runs on every commit:
- `prettier --write` on staged `src/**/*.{js,jsx,ts,tsx}`
- `eslint --fix` on staged `src/**/*.{js,jsx,ts,tsx}`

Hook installed via `npm run prepare` (`husky install && npm run build`).

## No CI Pipeline

No GitHub Actions or other CI configuration files exist in this repository. There is no automated build or test on pull requests.

## Release Process

Releases are fully manual, triggered by the maintainer:

```bash
npm version patch   # or minor / major
# This runs:
#   preversion: npm test && npm run lint && npm run build && git add dist/
#   version bump committed
#   postversion: git push && git push --tags
```

Then publish to npm:
```bash
npm publish
# This runs:
#   prepublishOnly: npm test && npm run lint
```

## Build Command

```bash
npm run build   # tsc --build tsconfig.release.json
```

Output goes to `dist/`. The `dist/` directory is committed and included in the npm package.

## Lint Command

```bash
npm run lint   # tsc --noemit && eslint 'src/**/*.{js,ts}'
```

Fails on TypeScript type errors or ESLint/Prettier violations.
