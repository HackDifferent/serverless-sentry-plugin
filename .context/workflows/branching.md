# Branching

## Integration Branch

`master` is the only integration branch. All work merges here.

## Feature Branch Naming

No consistent feature branch naming is visible in git history. The project appears to be primarily maintained by a single author (`arabold`) with PRs from external contributors. External contributors use their own conventions (e.g., `feature/any-respository-provider`, `fix/validation-schema`, `asb/use-enabled-var`).

No enforced naming convention — use descriptive names.

## Release / Tag Format

No `git tag` output available, but based on commit messages, releases follow `X.Y.Z` semver (e.g., commit `2.5.3`, `2.5.2`, `2.5.1`). Tags are likely `2.5.3` (not `v2.5.3`) based on version bump commit style.

## Merge Strategy

PRs from contributors are merged via GitHub's merge commit (`Merge pull request #NN from ...`). The author's own commits are pushed directly to `master`.

## Release Workflow

From `package.json` scripts:
1. `npm version <patch|minor|major>` triggers `preversion`: `npm test && npm run lint && npm run build && git add dist/`
2. Version bump committed
3. `postversion`: `git push && git push --tags`

The `dist/` directory must be built and staged before the version commit.
