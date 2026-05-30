# Contributing to Health Chain Frontend

Welcome! This guide helps new contributors get up and running quickly and follow consistent practices when submitting issues and pull requests.

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Local Development Setup](#local-development-setup)
3. [Branch Naming Convention](#branch-naming-convention)
4. [Commit Message Format](#commit-message-format)
5. [Pull Request Process](#pull-request-process)
6. [Code Style](#code-style)
7. [Testing Expectations](#testing-expectations)
8. [Issue Reporting](#issue-reporting)

---

## Prerequisites

Before you start, make sure you have the following installed and configured.

### Node.js

- **Node.js** >= 18.x — [Download](https://nodejs.org/)
- **npm** >= 9.x (bundled with Node.js)

Verify your versions:

```bash
node --version   # should print v18.x.x or higher
npm --version    # should print 9.x.x or higher
```

### Freighter Wallet (Browser Extension)

Health Chain integrates with the [Freighter](https://www.freighter.app/) browser extension to sign Stellar transactions. Install it in Chrome or Firefox before testing any blockchain-related features.

### Stellar Testnet Account

You need a funded testnet account to test wallet and transaction flows locally.

1. Open Freighter and create or import a wallet.
2. Switch the network to **Testnet** inside Freighter settings.
3. Fund your testnet account using the [Stellar Testnet Faucet (Friendbot)](https://friendbot.stellar.org/?addr=YOUR_PUBLIC_KEY).

> Replace `YOUR_PUBLIC_KEY` with your Freighter public key. Friendbot sends 10,000 test XLM to any new testnet address.

---

## Local Development Setup

Follow these steps to go from a fresh clone to a running development server.

### 1. Clone the repository

```bash
git clone https://github.com/Healthy-Stellar/Healthy-Stellar-frontend.git
cd Healthy-Stellar-frontend
```

### 2. Navigate to the frontend app

```bash
cd frontend/health-chain
```

### 3. Install dependencies

```bash
npm install
```

### 4. Configure environment variables

```bash
cp .env.example .env
```

Open `.env` and fill in the required values:

| Variable | Description | Default |
|---|---|---|
| `NEXT_PUBLIC_API_URL` | Backend REST API base URL | `http://localhost:3001` |
| `NEXT_PUBLIC_API_PREFIX` | API route prefix | `api/v1` |
| `NEXT_PUBLIC_WS_URL` | WebSocket server URL | `http://localhost:3001` |

### 5. Start the development server

```bash
npm run dev
```

Open [http://localhost:3000](http://localhost:3000) in your browser.

### 6. Verify the setup

| URL | What you should see |
|---|---|
| `http://localhost:3000` | Public landing page |
| `http://localhost:3000/dashboard` | Admin dashboard (redirects to sign-in if unauthenticated) |
| `http://localhost:3000/transparency` | Public transparency dashboard |

---

## Branch Naming Convention

Use the following prefixes to keep branches organized:

| Prefix | When to use | Example |
|---|---|---|
| `feat/` | New features | `feat/blood-unit-search` |
| `fix/` | Bug fixes | `fix/token-refresh-loop` |
| `docs/` | Documentation only | `docs/architecture-guide` |
| `chore/` | Maintenance, dependency updates | `chore/update-tanstack-query` |
| `refactor/` | Code restructuring without behavior change | `refactor/api-service-layer` |
| `test/` | Adding or updating tests | `test/dashboard-hook-coverage` |

When working on a specific GitHub issue, include the issue number:

```
feat/21-contributing-guide
fix/42-wallet-connect-error
```

---

## Commit Message Format

This project follows [Conventional Commits](https://www.conventionalcommits.org/).

### Format

```
type(scope): short summary in present tense

Optional body explaining what changed and why.
```

### Types

| Type | When to use |
|---|---|
| `feat` | A new feature |
| `fix` | A bug fix |
| `docs` | Documentation changes only |
| `style` | Formatting, whitespace — no logic change |
| `refactor` | Code change that is neither a fix nor a feature |
| `test` | Adding or updating tests |
| `chore` | Build process, tooling, dependency updates |
| `perf` | Performance improvement |

### Examples

```
feat(dashboard): add real-time rider tracking map
fix(auth): resolve token refresh race condition on concurrent requests
docs(contributing): add Freighter setup instructions
chore(deps): bump @tanstack/react-query to 5.90.21
test(hooks): add coverage for useAuth logout flow
```

---

## Pull Request Process

### Before opening a PR

Run all checks locally and make sure they pass:

```bash
# Type checking
npm run type-check

# Linting
npm run lint

# Tests
npm run test

# Production build
npm run build
```

Do not open a PR if any of these fail.

### PR checklist

- [ ] Branch is based on the latest `main`
- [ ] `npm run type-check` passes with no errors
- [ ] `npm run lint` passes with no errors
- [ ] `npm run test` passes
- [ ] `npm run build` succeeds
- [ ] New features include tests where applicable
- [ ] No debug code, `console.log`, or commented-out code left in
- [ ] Commit messages follow Conventional Commits format

### PR template

When you open a PR, fill in:

- **Title** — concise, under 70 characters, following the commit format
- **Summary** — what changed and why
- **Issue link** — `Closes #<issue-number>`
- **Testing** — what you tested manually and/or automatically

### Review requirements

1. Open a PR against the `main` branch.
2. Reference the related issue (e.g., `Closes #21`).
3. Wait for CI checks to pass.
4. Address all review comments before requesting re-review.
5. A maintainer will merge once the PR is approved.

---

## Code Style

### TypeScript

- Strict mode is enabled (`tsconfig.json`). All types must be explicit — avoid `any`.
- Use named exports for components and functions.
- Co-locate types with the module that owns them (e.g., `lib/types/riders.ts` for rider types).

### ESLint & Prettier

The project uses ESLint with the Next.js config and Prettier for formatting.

```bash
# Check for lint errors
npm run lint

# Auto-fix lint errors where possible
npm run lint -- --fix
```

Configuration files:
- `eslint.config.mjs` — ESLint rules
- `.prettierrc` (if present) — Prettier formatting rules

### Naming conventions

| Thing | Convention | Example |
|---|---|---|
| React components | PascalCase | `BloodUnitCard` |
| Hooks | camelCase, `use` prefix | `useBloodBanks` |
| API functions | camelCase | `fetchDashboardStats` |
| Types / Interfaces | PascalCase | `DashboardStats` |
| Zustand stores | camelCase, `use` prefix | `useAuthStore` |
| Files | kebab-case | `blood-units.api.ts` |

### Component structure

Follow the existing pattern in `components/`:

```
components/
  <feature>/
    FeatureComponent.tsx   # Main component
    SubComponent.tsx       # Sub-components
    __tests__/             # Tests for this feature
```

---

## Testing Expectations

The project uses [Vitest](https://vitest.dev/) with [Testing Library](https://testing-library.com/).

### Running tests

```bash
# Run all tests once
npm run test

# Watch mode during development
npm run test:watch
```

### What tests are required before a PR can be merged

- **New hooks** (`lib/hooks/`) — unit tests covering the happy path and error states.
- **New API functions** (`lib/api/`) — unit tests mocking `fetch` and verifying request shape and response mapping.
- **New utility functions** (`lib/utils/`) — unit tests for all exported functions.
- **Bug fixes** — a regression test that would have caught the bug.

UI component tests are encouraged but not strictly required unless the component contains non-trivial logic.

### Test file location

Place test files next to the code they test, inside a `__tests__/` subdirectory:

```
lib/api/__tests__/dashboard.api.spec.ts
lib/hooks/__tests__/useDashboardData.spec.ts
```

---

## Issue Reporting

### Bug reports

When filing a bug, include:

1. **Steps to reproduce** — numbered, minimal steps
2. **Expected behavior** — what should happen
3. **Actual behavior** — what actually happens
4. **Environment** — browser, OS, Node.js version
5. **Screenshots or logs** — if applicable

Use the **Bug Report** issue template if one is available.

### Feature requests

When requesting a feature, include:

1. **Problem statement** — what problem does this solve?
2. **Proposed solution** — how you imagine it working
3. **Alternatives considered** — other approaches you thought about
4. **Additional context** — mockups, related issues, etc.

Use the **Feature Request** issue template if one is available.

### Security vulnerabilities

**Do not open public issues for security vulnerabilities.** Please refer to [SECURITY.md](../../SECURITY.md) for responsible disclosure instructions.

---

## Questions?

- Open a [GitHub Issue](https://github.com/Healthy-Stellar/Healthy-Stellar-frontend/issues) for bugs or feature requests.
- Check existing documentation in `docs/` and `lib/api/README.md`.
