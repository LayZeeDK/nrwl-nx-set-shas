# Codebase Structure

**Analysis Date:** 2026-03-17

## Directory Layout

```
nrwl-nx-set-shas/
├── .github/                    # GitHub configuration
│   ├── workflows/              # CI/CD workflow definitions
│   └── assets/                 # Documentation images
├── .husky/                     # Git hooks (Husky)
├── .planning/                  # Planning and documentation
│   └── codebase/               # Codebase analysis documents (GENERATED)
├── dist/                       # Compiled output (GENERATED)
│   └── nx-set-shas.js          # Bundled executable
├── tools/                      # Development utilities
│   └── pre-commit.ts           # Pre-commit hook implementation
├── action.yml                  # GitHub Action definition
├── nx-set-shas.ts              # Main entry point
├── package.json                # Dependencies and scripts
├── tsconfig.json               # TypeScript configuration
├── .prettierrc                 # Code formatting config
├── .prettierignore             # Prettier ignore rules
├── .gitignore                  # Git ignore rules
├── README.md                   # User-facing documentation
└── CONTRIBUTING.md             # Contribution guidelines
```

## Directory Purposes

**.github/**

- Purpose: GitHub-specific configuration and automation
- Contains: Workflow files (CI/CD), branding assets
- Key files: `workflows/test.yml`, `workflows/publish.yml`

**.github/workflows/**

- Purpose: CI/CD pipeline definitions
- Contains: YAML workflow files for testing, linting, publishing
- Key files:
  - `test.yml` - Runs tests and formatting checks
  - `integration-test-workflow.yml` - Integration test pipeline
  - `publish.yml` - Release publishing workflow
  - `format.yml` - Code formatting check
  - `test-integration.yml` - Additional integration tests

**.husky/**

- Purpose: Git hook automation via Husky
- Contains: Shell scripts for pre-commit hooks
- Key files: `pre-commit` - Validates build and formatting before committing

**.planning/codebase/**

- Purpose: Generated codebase analysis and architecture documentation
- Contains: Markdown documents describing architecture, structure, conventions
- Generated: Yes (via GSD mapping tool)
- Committed: Yes (checked in to version control)

**dist/**

- Purpose: Compiled and bundled output
- Contains: JavaScript bundle from TypeScript compilation
- Key files: `nx-set-shas.js` - Entry point for GitHub Actions runtime
- Generated: Yes (via `npm run build`)
- Committed: Yes (checked in for GitHub Actions distribution)

**tools/**

- Purpose: Development and build-time utilities
- Contains: TypeScript scripts for pre-commit hooks
- Key files: `pre-commit.ts` - Validates build integrity and code formatting

## Key File Locations

**Entry Points:**

- `nx-set-shas.ts`: Main Action logic (TypeScript source)
- `dist/nx-set-shas.js`: Compiled Action entry point (used by GitHub Actions runtime via action.yml)
- `action.yml`: GitHub Action metadata and input/output definitions

**Configuration:**

- `package.json`: Dependencies, scripts, version, engine requirements
- `tsconfig.json`: TypeScript compiler options
- `.prettierrc`: Code formatter settings (2-space indentation, single quotes, LF line endings)
- `action.yml`: Action inputs (gh-token, main-branch-name, remote, etc.) and outputs (base, head, noPreviousBuild)

**Core Logic:**

- `nx-set-shas.ts`: Entire application logic (283 lines)
  - Input extraction (lines 7-24)
  - Event routing and SHA resolution (lines 28-162)
  - Utility functions for Git/API (lines 164-282)

**Development:**

- `tools/pre-commit.ts`: Ensures build artifacts and formatting are in sync (45 lines)
- `.husky/pre-commit`: Git hook runner

**Documentation:**

- `README.md`: User-facing documentation with examples and configuration options
- `CONTRIBUTING.md`: Contribution guidelines for developers

## Naming Conventions

**Files:**

- Main entry point: `<action-name>.ts` (e.g., `nx-set-shas.ts`)
- Supporting scripts: `<script-purpose>.ts` (e.g., `pre-commit.ts`)
- Compiled output: Same base name as input (e.g., `nx-set-shas.js`)
- Config files: Standard names (`.prettierrc`, `.gitignore`, `action.yml`, `package.json`, `tsconfig.json`)

**Directories:**

- Internal tooling: `tools/`
- GitHub integration: `.github/`
- Version control hooks: `.husky/`
- Planning/docs: `.planning/codebase/`
- Build output: `dist/`

**Functions:**

- Exported/main functions: camelCase with descriptive verb-noun pattern
  - `findSuccessfulCommit` - queries GitHub API for successful workflow run
  - `findExistingCommit` - searches for valid commit in list
  - `commitExists` - validates single commit
- Helper functions: camelCase
  - `reportFailure` - logs error and fails action
  - `stripNewLineEndings` - string utility

**Variables:**

- Constants/SHAs: SCREAMING_SNAKE_CASE (e.g., `BASE_SHA`, `HEAD_SHA`)
- Local variables: camelCase (e.g., `runId`, `mainBranchName`, `workingDirectory`)
- Configuration parameters: kebab-case in action.yml (converted to camelCase in code)

## Where to Add New Code

**New Feature:**

- Primary code: `nx-set-shas.ts` (add function above main async block if helper, or extend event routing logic)
- Build/tooling: `tools/<feature-name>.ts`

**New Test/Integration Test:**

- Test definitions: `.github/workflows/<test-name>.yml`

**New Git Hook:**

- Hook implementation: `tools/<hook-name>.ts`
- Hook registration: `.husky/<hook-name>` (shell script calling the TypeScript file)

**Configuration Changes:**

- Action inputs/outputs: Update `action.yml`
- Package/runtime config: Update `package.json`, `tsconfig.json`, `.prettierrc`

## Special Directories

**.git/**

- Purpose: Git version control metadata
- Generated: No (standard for all repos)
- Committed: No

**dist/**

- Purpose: Compiled JavaScript for distribution
- Generated: Yes (via `bun build` command)
- Committed: Yes (required for GitHub Actions to execute without build step)

**.planning/codebase/**

- Purpose: Architecture and codebase analysis documents
- Generated: Yes (via GSD tool)
- Committed: Yes

**node_modules/** (not shown in listing)

- Purpose: Installed npm/bun dependencies
- Generated: Yes (via `bun install`)
- Committed: No (in .gitignore)

---

_Structure analysis: 2026-03-17_
