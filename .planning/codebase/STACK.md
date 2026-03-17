# Technology Stack

**Analysis Date:** 2026-03-17

## Languages

**Primary:**

- TypeScript 5.8.3 - Action source code and build tooling

## Runtime

**Environment:**

- Node.js 20.19.4 (defined in volta configuration)
- GitHub Actions Node20 runtime (action.yml line 45)

**Package Manager:**

- Bun 1.2.19 - Primary package manager and build tool
- Lockfile: bun.lock present (uses Bun's native lockfile format)

## Frameworks

**Core:**

- GitHub Actions - Action framework for CI/CD integration

**Build/Dev:**

- TypeScript 5.8.3 - Static type checking and transpilation
- Bun - Bundles TypeScript source to executable JavaScript (`bun build` in package.json line 8)
- Prettier 3.6.2 - Code formatting

**Git Hooks:**

- Husky 9.1.7 - Git hook management
- lint-staged 16.1.2 - Pre-commit linting and formatting

## Key Dependencies

**Critical:**

- `@actions/core` 1.11.1 - GitHub Actions toolkit for core functions (logging, inputs, outputs, environment variables)
  - Used for: `core.getInput()`, `core.setOutput()`, `core.setFailed()`, `core.exportVariable()`
  - Located in: `nx-set-shas.ts` lines 1, 12-20, 78, 128, 152-161

- `@actions/github` 6.0.1 - GitHub API client and context
  - Used for: GitHub REST API calls, workflow run queries, commit verification
  - Provides: `github.context`, `github.getOctokit()`
  - Located in: `nx-set-shas.ts` lines 2, 11, 185
  - Transitive dependency on: `@octokit/core`, `@octokit/plugin-rest-endpoint-methods`, `@octokit/request`

**Development:**

- `@types/node` 20.19.9 - TypeScript type definitions for Node.js
- `yoctocolors` 2.1.1 - Terminal color output for pre-commit script
- `is-ci` 4.1.0 - Detects CI/CD environment for conditional Husky installation (line 9)

## Configuration

**Environment:**

- GitHub token passed via action input `gh-token` (action.yml line 5)
- Set as `process.env.GITHUB_TOKEN` in `nx-set-shas.ts` line 12
- Used by Octokit for GitHub REST API authentication

**Build:**

- `tsconfig.json` - TypeScript compiler configuration
  - Target: ES2023
  - Module: nodenext
  - Lib: ES2023
- `bun.lock` - Dependency lockfile
- `.prettierrc` - Prettier formatting configuration (2-space tabs, single quotes, LF line endings)

**Versioning:**

- Semantic versioning managed in package.json (currently 4.4.0)
- Version is for the published GitHub Action

## Platform Requirements

**Development:**

- Node.js >= 20 (package.json engines line 17)
- Bun >= 1.2.19 (package.json packageManager line 22)
- Git (for `git grep`, `git rev-parse`, `git merge-base`, `git hash-object`, `git cat-file` commands)

**Production:**

- GitHub Actions environment (runner with Node.js 20)
- Git available on the runner
- GitHub token with appropriate permissions for reading workflow runs and commits
- Network access to GitHub API (api.github.com)

## Build Process

**Build Command:** `bun build ./nx-set-shas.ts --outdir ./dist --target node`

**Output:**

- Transpiles TypeScript to JavaScript
- Produces: `dist/nx-set-shas.js` (referenced in action.yml line 46)
- Target: Node.js (not browser)

**Pre-commit Hook:**

- Runs `npm run build` to validate TypeScript compilation
- Adds modified files in `dist/` to git commit
- Runs `npm run format` to auto-fix formatting
- Located in: `tools/pre-commit.ts`

---

_Stack analysis: 2026-03-17_
