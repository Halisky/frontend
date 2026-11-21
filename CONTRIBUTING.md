# Contributing to SynapsVault Frontend

A Stellar-powered marketplace UI built with React, TypeScript, Vite, and Tailwind CSS.

## Local Development

### Prerequisites
- Node.js >=24.15.0 (see `engines` in `package.json`)
- npm (or pnpm)

The repo records its Node version in `.nvmrc`. If you use nvm, see
[Selecting a Node version with nvm](./docs/tooling-nvm.md) for how to install
that version and switch to it with `nvm use` before installing dependencies.

### Setup

```bash
# Install dependencies
npm install

# Configure environment
cp .env.example .env
# Edit .env — leave VITE_API_URL empty for local development and set
# VITE_DEV_PROXY_TARGET to your backend (e.g. http://localhost:3000)

# Start dev server
npm run dev
# Open http://localhost:5173
```

The Vite dev server provides hot module reload, but it does **not** typecheck your
code — Vite strips types without checking them. Run `npm run typecheck`
(`tsc --noEmit`) separately to catch type errors, and run it before opening a PR.

### API Environment Variables

For local development, keep `VITE_API_URL` **empty** and point
`VITE_DEV_PROXY_TARGET` at your backend instead. The Vite dev server proxies
API requests to that target, which avoids CORS issues and keeps the browser
talking to a single origin. See `.env.example` for the full list of variables.

`.env.example` lists only variables the application actually reads. There are
no debug or verbose-logging flags (`VITE_DEBUG` / `VITE_VERBOSE_LOGGING` were
removed because nothing in `src/` consumed them). When you add a new `VITE_*`
variable, add its example entry in the same PR as the code that reads it.

```bash
# .env (local development)
VITE_API_URL=
VITE_DEV_PROXY_TARGET=http://localhost:3000
```

Only set `VITE_API_URL` when the frontend must call a backend directly (for
example, a deployed environment with a fictional host such as
`https://api.example.test`).

## Docker Development

```bash
# Build image
npm run docker:build

# Run container
npm run docker:run
# Open http://localhost:8080
```

See [Local Docker Smoke Checks](#local-docker-smoke-checks) to verify the
container is healthy before pushing. If port 8080 is already in use, see
[Port 8080 already in use?](#port-8080-already-in-use).

## Local Docker Smoke Checks

CI runs three checks against the container on port 8080 (see
[`.github/workflows/ci.yml`](./.github/workflows/ci.yml), job `docker`).
Run the same checks locally to catch regressions before they hit CI.

### 1. Start the container

```bash
docker build -t synapsvault-frontend:local .
docker run -d --name web -p 8080:8080 synapsvault-frontend:local
```

### 2. Health endpoint

The image exposes a `/health` route. Poll it until the server is ready (it
may take a few seconds to start), then assert a 200 response:

```bash
for i in $(seq 1 15); do
  curl -fsS http://localhost:8080/health && break
  sleep 1
done
curl -fsS http://localhost:8080/health
```

A successful response means the server is up and the health route is reachable.

### 3. App-shell check

The root path must return the HTML app shell — specifically the `<div
id="root">` mount point that React attaches to:

```bash
curl -fsS http://localhost:8080/ | grep -q '<div id="root">'
```

### 4. Deep-link / SPA fallback check

The server must serve the same app shell for any path, so that client-side
routes such as `/marketplace/listing/abc123` work after a hard refresh or
direct navigation:

```bash
curl -fsS http://localhost:8080/marketplace/listing/abc123 | grep -q '<div id="root">'
```

If this check fails, the server is not configured to fall back unknown paths to
`index.html`. Fix the Nginx (or equivalent) `try_files` rule in the
`Dockerfile`.

### 5. Docker HEALTHCHECK status

The image defines its own `HEALTHCHECK` instruction. After the checks above,
verify that Docker itself reports the container as `healthy`:

```bash
for i in $(seq 1 20); do
  status=$(docker inspect --format '{{.State.Health.Status}}' web)
  [ "$status" = healthy ] && break
  sleep 2
done
echo "container health: $status"
[ "$status" = healthy ]
```

If the status stays `starting` or flips to `unhealthy`, see
[Container stays unhealthy or won't start](#container-stays-unhealthy-or-wont-start).

### Cleanup

```bash
docker stop web && docker rm web
```

## Branch Strategy

| Branch | Purpose | Deploy Target |
|--------|---------|----------------|
| `main` | Production-ready | Vercel → production |
| `dev` | Staging/integration | Vercel preview |
| `feat/*` | Feature branches | PR to `dev` |
| `fix/*` | Bug fixes | PR to `dev` |
| `chore/*` | Maintenance | PR to `dev` |

CI runs only on pushes to `main`/`dev` and on PRs targeting `main`/`dev` — see
[CI Branch Triggers](#ci-branch-triggers) for why a push to a feature branch may
produce no run.

## CI Branch Triggers

CI only runs on specific branches. The workflow's `on` block (see
[`.github/workflows/ci.yml`](./.github/workflows/ci.yml)) filters both event
types:

```yaml
on:
  push:
    branches: [main, dev]
  pull_request:
    branches: [main, dev]
```

- **Push** runs CI automatically on direct pushes to `main` or `dev`.
- **Pull request** runs CI when a PR's target (base) branch is `main` or `dev`.

**Why a push may produce no run.** A push to any other branch — such as
`feat/login`, `fix/typo`, or `chore/deps` — triggers nothing, because those
branches are not in the `push.branches` list. CI starts for such a branch only
once you open a PR targeting `main` or `dev`: the `pull_request` filter matches
the PR's base branch, so `feat/login` → `dev` runs CI, while a standalone push
to `feat/login` does not.

This keeps CI focused on the branches that matter for integration and release
(`main`, `dev`) and on PRs seeking to merge into them, rather than running on
every feature-branch commit.

## Development Workflow

1. **Create feature branch from dev:**
   ```bash
   git checkout -b feat/my-feature dev
   ```

2. **Make changes and test locally:**
   ```bash
   npm run dev
   npm run test:watch
   ```

3. **Before committing, verify:**
   ```bash
   npm run lint:fix    # Auto-fix style issues, then review the diff
   npm run lint        # Confirm no remaining lint errors (CI enforces it)
   npm run typecheck   # Type safety
   npm run test        # All tests pass
   npm run build       # Production build works
   # or simply: npm run check
   node scripts/scan-secrets.mjs   # No credentials in built assets
   ```

   To verify the built output locally, see
   [Previewing a Production Build](#previewing-a-production-build).
   For details on reviewing auto-fix changes, see [Linting](#linting).
   To understand what the secret scanner checks and how to interpret its
   output, see [Secret Scanner](#secret-scanner). `npm run check` also
   verifies English/Spanish key parity — see
   [Translation Key Parity](#translation-key-parity).

   CI also enforces a production dependency audit that `npm run check` does not
   include — see [Production Dependency Audit](#production-dependency-audit) to
   run it locally before opening a PR.

4. **Commit with conventional format (see below)**

5. **Push and create PR to `dev`**

   Pushing a new commit to an open PR (or to `main`/`dev`) cancels any CI run
   still in progress for that same ref — see
   [CI Concurrency and Cancelled Runs](#ci-concurrency-and-cancelled-runs).

6. **After review + merge to `dev`:**
   - Staging deploy to Vercel preview
   - Create release PR to `main`

## Testing

```bash
npm run test           # Run all tests once
npm run test:watch    # Watch mode for development
npm run test:ui       # Visual UI for test runner
```

Tests use Vitest + React Testing Library. Focus on user behavior, not implementation details.

Vitest packages are bumped by Dependabot in separate PRs — see
[Dependabot Updates](#dependabot-updates) for keeping them in step.

### Vitest UI Peer Dependency
For hooks that schedule timers, use Vitest's fake-timer API instead of real delays.
See [docs/testing-fake-timers.md](./docs/testing-fake-timers.md) for a worked example with `useDebouncedValue`, including timer cleanup.

More focused guides:

- [docs/testing-catalog-stale-banner.md](./docs/testing-catalog-stale-banner.md): testing localized, timestamped status messages in English and Spanish
- [docs/testing-language-effects.md](./docs/testing-language-effects.md): testing `applyLanguageEffects` when `localStorage` throws
- [docs/tooling-vitest-ui.md](./docs/tooling-vitest-ui.md): making `npm run test:ui` work from a clean install
- [docs/tooling-bundle-analyzer.md](./docs/tooling-bundle-analyzer.md): opening `stats.html` on any operating system
- [docs/session-expiry.md](./docs/session-expiry.md): how session tokens are validated, cleared on expiry, and used during wallet sign-in

### Test setup and automatic cleanup

`src/test/setup.ts` runs before every test file (configured in `vite.config.ts` via `setupFiles`). It does two things automatically so individual tests do not have to.

```bash
npm install -D @vitest/ui@^5.0.5
```

Then run `npm run test:ui` to open the visual test runner.

## Previewing a Production Build

Build the app and serve the generated assets locally:

```bash
npm run build     # Outputs to dist/
npm run preview   # Serves the built assets (default: http://localhost:4173)
```

`npm run preview` serves the files produced by `npm run build` — it does not
run the dev server, hot module reload, or the dev proxy. Use it to check the
production bundle before opening a PR.

## Production Build (build:prod)

```bash
npm run build:prod
```

This runs the production build with `NODE_ENV=production` and writes the
generated assets to the `dist/` directory. Deploy or inspect `dist/` after the
command completes. CI's `build` job also uploads its own `dist/` output as a
downloadable artifact — see [CI dist Artifact](#ci-dist-artifact).

### build:prod on Windows

The script is currently defined as:

```json
"build:prod": "NODE_ENV=production vite build"
```

The `NODE_ENV=production <command>` prefix is POSIX shell syntax. It works in
bash, zsh, Git Bash, and WSL, but fails in the default Windows command shell
(`cmd.exe`) and in PowerShell, which do not support inline environment
assignment — npm reports that `NODE_ENV` is not recognized as a command.

**Portable alternative.** `vite build` already runs in `production` mode by
default, so plain `npm run build` emits the same production assets to `dist/`
on every platform:

```bash
npm run build
```

Use `npm run build` on Windows, or run `npm run build:prod` from Git Bash or
WSL. Either way, check that `dist/` contains hashed, minified assets under
`dist/assets/`. Do not add a dependency (such as `cross-env`) solely to set
`NODE_ENV` for this script.

## CI dist Artifact

The `build` job (see [`.github/workflows/ci.yml`](./.github/workflows/ci.yml),
step `Upload dist artifact`) uploads its build output as a named artifact:

```yaml
- name: Upload dist artifact
  uses: actions/upload-artifact@v7
  with:
    name: dist
    path: dist/
    retention-days: 7
```

**Where it appears.** Once the run finishes, the `dist` artifact is listed in
the **Artifacts** section of the workflow run's summary page — open the
repository's Actions tab, click the CI run, and scroll to "Artifacts". It
holds the contents of the `dist/` directory produced by `npm run build` in the
`build` job. Download it as a zip to inspect that exact build without
rebuilding locally.

**Retention.** `retention-days: 7` keeps the artifact downloadable for seven
days; GitHub then deletes it automatically. Once it is gone it can't be
recovered — re-run the workflow or rebuild locally to produce a fresh `dist/`.

## Production Dependency Audit
**React Testing Library cleanup**

After each test, `cleanup()` from `@testing-library/react` unmounts every component tree that was rendered during that test and removes its DOM nodes. This is registered with Vitest's `afterEach`:

```ts
afterEach(() => {
  cleanup();
});
```

Without this, rendered components would accumulate across tests in the same file, causing false query matches, event-listener leaks, and unpredictable failures.

**jsdom environment shims**

jsdom does not implement every browser API. The setup file patches two gaps that several hooks depend on:

| Shim | Why it is needed |
|------|-----------------|
| `window.matchMedia` | `useTheme()` calls `window.matchMedia('(prefers-color-scheme: dark)')` unconditionally on mount. jsdom leaves this undefined, throwing a `TypeError`. The shim returns a minimal object (`matches: false`, no-op `addEventListener`/`removeEventListener`). |
| `globalThis.localStorage` | `useTheme()` and `useWalletConnection()` read and write `localStorage` on mount. Some jsdom configurations leave it undefined. The shim provides a `Map`-backed in-memory `Storage` object. |

The shims apply globally for the entire test run, so you do not need to mock either API in individual test files.

**Restoring test-specific mocks**

When a test overrides a global or module export with `vi.spyOn` or `vi.fn()`, restore it after the test so it does not affect others. The simplest approach is `vi.restoreAllMocks()` in an `afterEach`:

```ts
import { afterEach, vi } from "vitest";

afterEach(() => {
  vi.restoreAllMocks();
});
```

Alternatively, pass `restoreMocks: true` in the Vitest config to restore every spy automatically. If you only need to reset call counts and return values without removing the spy, use `vi.clearAllMocks()` instead.

**Example: overriding `localStorage` for one test**

The global shim covers the happy path. When you need to test error handling or a specific stored value, replace the shim locally and restore it afterwards:

```ts
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { render, screen } from "@testing-library/react";
import { ThemeToggle } from "./ThemeToggle";

## CI Job Timeouts

None of the CI jobs (`test`, `build`, `docker` in
[`.github/workflows/ci.yml`](./.github/workflows/ci.yml)) currently set
`timeout-minutes`. GitHub Actions therefore applies its platform default of
**360 minutes (6 hours)** per job.

**What this means in practice.** A stalled command — a hung `npm ci`, a test
that never resolves, or a container that never becomes healthy — keeps the
runner busy until the default limit is hit, rather than failing fast. Normal
runs finish in a few minutes, so a job still running far beyond that is almost
certainly stuck.

**If a job appears stuck**

1. Open the run in the repository's Actions tab and check the last log line of
   the running step to see which command stalled.
2. Click **Cancel workflow** to free the runner — don't wait for the 6-hour
   limit. (Pushing a new commit to the same branch or PR also cancels it; see
   [CI Concurrency and Cancelled Runs](#ci-concurrency-and-cancelled-runs).)
3. Reproduce the stalled step locally (for example `npm test` or the
   [Local Docker Smoke Checks](#local-docker-smoke-checks)) before re-running.

When adding a timeout to a job, set `timeout-minutes` at the job level (next to
`runs-on`), choose a value comfortably above the job's typical duration, and
leave its steps unchanged.

## Secret Scanner
describe("ThemeToggle", () => {
  let getItemSpy: ReturnType<typeof vi.spyOn>;

  beforeEach(() => {
    getItemSpy = vi.spyOn(globalThis.localStorage, "getItem");
  });

  afterEach(() => {
    vi.restoreAllMocks(); // removes the spy, restores the shim's original getItem
  });

  it("defaults to light theme when nothing is stored", () => {
    getItemSpy.mockReturnValue(null);
    render(<ThemeToggle />);
    expect(screen.getByRole("switch")).toHaveAttribute("aria-checked", "false");
  });

  it("restores dark theme from storage", () => {
    getItemSpy.mockReturnValue("dark");
    render(<ThemeToggle />);
    expect(screen.getByRole("switch")).toHaveAttribute("aria-checked", "true");
  });
});
```

**Clean output** (no matches):

```
Built assets: no recognized private-key patterns found.
```

**Failure output** (match found):

```
dist/assets/index-Dq3kL8mN.js: possible Stellar private key
```

The scanner prints only the file path and pattern label — never the matched
value itself. The process exits with code 1 when any match is found, so it
blocks CI the same way a failing test does.

### Where the scan runs

The scanner reads `dist/`, so it must always run **after** a build. It is
currently invoked directly with `node` in two places (there is no dedicated
npm script for it yet):

| Where | Invocation | Runs after |
|-------|-----------|------------|
| `npm run check` (`package.json`) | `... && npm run bundle:check && node scripts/scan-secrets.mjs` | `bundle:check`, which runs `npm run build` |
| CI `build` job, step `Scan for leaked secrets` | `node scripts/scan-secrets.mjs` | `Build` and `Enforce bundle size budgets` |

In both places a non-zero exit fails the run: `npm run check` stops at the
`&&` chain, and the CI step fails the `build` job (so the `dist` artifact is
not uploaded). If you change how the scanner is invoked, update both places
together and keep it after the build step.

If you get a false positive from a test fixture or generated stub, check that
no real credential was accidentally inlined into the bundle. If the match is
genuinely a non-secret (for example, a base32 string that happens to match the
Stellar key regex), move the value out of the bundled source or rename it so
it no longer matches.

## Dependabot Updates

Dependabot is configured in
[`.github/dependabot.yml`](./.github/dependabot.yml):

| Ecosystem | Directory | Schedule | Open PR limit |
|-----------|-----------|----------|---------------|
| `npm` | `/` | weekly | 5 |
| `github-actions` | `/` | weekly | default (5) |

No `groups` are defined, so Dependabot opens **one pull request per
dependency**. Security updates are raised independently of this weekly
schedule and are not affected by the PR limit.

**Coupled Vitest tooling.** `vitest` and its companion packages — such as
`@vitest/ui` (needed for `npm run test:ui`, see
[Vitest UI Peer Dependency](#vitest-ui-peer-dependency)) and any
`@vitest/coverage-*` provider — must stay on the same version. Because they
arrive as separate PRs, merging only one of them can leave the toolchain
mismatched. When reviewing a Vitest-related Dependabot PR:

1. Check for sibling PRs bumping other `vitest` / `@vitest/*` packages.
2. Merge them together (or combine them into one branch) so the versions stay
   aligned.
3. Run `npm test` — and `npm run test:ui` if you have `@vitest/ui` installed —
   before merging.

If you add a `groups` entry to `dependabot.yml`, keep the existing weekly
schedule and leave unrelated dependencies ungrouped, then validate the YAML
before committing.

## Linting

The project uses ESLint with the TypeScript and React Hooks plugins. CI runs
`npm run lint` and fails the build on any error, so lint must pass before you
push.

```bash
npm run lint        # Report all lint errors (read-only)
npm run lint:fix    # Auto-fix what ESLint can correct in place
```

`lint:fix` rewrites files directly — it does not stage or commit anything.
After running it, review the diff before you stage so you understand every
change:

```bash
npm run lint:fix
git diff            # Review all edits made by the auto-fixer
git add -p          # Stage interactively if you want fine-grained control
```

Not every error is auto-fixable. Formatting changes (quote style, semicolons,
trailing commas) and simple import ordering are fixed automatically; logic
errors flagged by the React Hooks plugin (missing dependencies in `useEffect`,
conditional hook calls) require manual correction. After `lint:fix`, re-run
`npm run lint` to confirm no errors remain.

### Linting a single file

When you only want to check one file — for example while iterating on a
component — pass the path directly to `npx eslint`:

```bash
npx eslint src/components/ResourceCard.tsx
```

To auto-fix that file alone:

```bash
npx eslint --fix src/components/ResourceCard.tsx
```

The config (`eslint.config.js`) lints `**/*.{ts,tsx}` source files plus
`*.config.{js,ts}` and `scripts/**/*.{js,mjs}`. Passing a path outside those
patterns (for example a plain `.js` file in `src/`) produces no output because
no rule block matches it — not because the file is error-free.

Example output for a file with errors:

```
/workspaces/frontend/src/components/ResourceCard.tsx
  12:7  error  'setItems' is assigned a value but never used  @typescript-eslint/no-unused-vars
  28:3  error  React Hook useEffect has a missing dependency: 'fetchData'  react-hooks/exhaustive-deps

✖ 2 problems (2 errors, 0 warnings)
  0 errors and 0 warnings potentially fixable with the `--fix` option.
```

The first column is the line:column of the finding, the second is the rule
name. Rules from `react-hooks/exhaustive-deps` are not auto-fixable and need
manual correction; `@typescript-eslint/no-unused-vars` errors are also not
auto-fixed (rename or prefix unused variables with `_` to suppress them, as
configured in `eslint.config.js`).
The `cleanup()` registered in the setup file unmounts the rendered tree after each test, and `vi.restoreAllMocks()` removes the `getItem` spy, so neither the DOM state nor the mock leaks into the next test.

## Code Standards

- **TypeScript**: Strict mode, no `any` without comment
- **Components**: Functional components with hooks
- **Styling**: Use semantic Tailwind tokens (`bg-surface`, `text-fg-muted`, `border-line`) — see [DESIGN_SYSTEM.md](./DESIGN_SYSTEM.md)
- **Accessibility**: ARIA labels, keyboard navigation
- **Performance**: Code-split routes, memoize expensive renders
- **Errors**: Use Sentry for tracking, user-friendly error messages

## Design System

Tokens live in `src/styles/tokens.css` and are exposed to Tailwind as semantic
classes that adapt to light/dark automatically. Read
[DESIGN_SYSTEM.md](./DESIGN_SYSTEM.md) before adding UI.

## Adding a New Page

Pages are rendered as tabs from `src/App.tsx`; there is no `src/pages` directory.

1. **Create component:** `src/components/MyPage.tsx`
2. **Add tests:** `src/components/MyPage.test.tsx`
3. **Wire it into the tab navigation in `src/App.tsx`:**
   - Extend the `Tab` union with your new tab id (e.g. `'my-page'`)
   - Add a nav entry for the tab in the navigation list
   - Render `<MyPage />` in the tab switch for that id
4. **Test:** `npm run dev` and verify the tab renders and switches correctly

## Commit Format (Conventional Commits)

```
feat(ui): add subscription plan selector
fix(wallet): handle Freighter disconnect race condition
test(catalog): add search filter tests
docs: update component library
refactor(theme): extract color variables to custom properties
chore: upgrade vite to 6.0
perf(catalog): memoize ResourceCard component
```

**Types:** feat, fix, test, docs, refactor, chore, perf, style

## Code Review Checklist

- [ ] Lint passes (`npm run lint`)
- [ ] TypeScript compiles (`npm run typecheck`)
- [ ] Tests pass (`npm run test`)
- [ ] Secret scanner passes (`node scripts/scan-secrets.mjs` after `npm run build`)
- [ ] Bundle size acceptable (check Vite output)
- [ ] No hardcoded colors/spacing (use CSS vars)
- [ ] Accessible (keyboard nav, ARIA labels)
- [ ] Mobile responsive (tested at 320px+)
- [ ] Follows conventional commit format
- [ ] PR description explains "why", not "what"

## Wallet Integration

Uses Stellar Freighter for wallet connection.

- [`src/lib/freighter.ts`](./src/lib/freighter.ts) — Freighter helpers such as
  `loadFreighter`, `signWithFreighter`, and `isFreighterInstalled`
- [`src/hooks/useWalletConnection.ts`](./src/hooks/useWalletConnection.ts) — connection
  state, `connect`/`disconnect`, and restore-on-mount behavior

Example public key used in tests and docs: `GABCD...` (placeholder only — never commit
real keys or secrets).

For testing wallet-dependent components without triggering a real signature request, see
[Testing wallet interactions](#testing-wallet-interactions) below.

## Testing wallet interactions

Components that call Freighter should never reach the extension during unit tests.
Mock the hook that wraps the wallet call instead of the extension itself.

`BuyModal` delegates all payment logic to `useBuyResource`. The test suite in
`src/components/BuyModal.test.tsx` mocks that hook at the module boundary:

```ts
// At the top of the test file
vi.mock("../hooks/useBuyResource.js", () => ({
  useBuyResource: vi.fn(),
}));
```

Then each test drives the component into the desired state by configuring the mock's
return value with `vi.mocked`:

```ts
// Wallet connected — idle state, pay button enabled
vi.mocked(useBuyResource).mockReturnValue({
  status: "idle",
  result: null,
  error: null,
  buy: mockBuy,
  reset: mockReset,
});
render(<BuyModal
  resourceTitle="Premium Dataset"
  price="1.50"
  recipient="GBUYER1XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX"
  accessUrl="https://api.example.com/resources/abc"
  walletAddress="GWALLET1XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX"
  onClose={vi.fn()}
  onCopyUrl={vi.fn()}
/>);
// pay button is now enabled
expect(screen.getByRole("button", { name: /pay 1\.50 usdc/i })).toBeEnabled();
```

**Key patterns**

| Scenario | `status` to set | What to assert |
|---|---|---|
| No wallet connected | `"idle"` + `walletAddress={null}` | Pay button disabled, "connect your Freighter wallet" notice visible |
| Payment in flight | `"paying"` | Spinner has `aria-busy="true"`, Close button disabled |
| Payment succeeded | `"success"` + `result.explorerUrl` | Explorer link rendered with correct `href` |
| Payment failed | `"error"` + `error` message | Error text visible, "Try again" button calls `buy` |

Because the mock controls all state transitions, tests run offline and never prompt
the Freighter browser extension for a signature.

## Payment Integration

Uses x402 protocol via `@x402/fetch` and `@x402/stellar`. See `src/api/payments.ts`.

## Troubleshooting

**Port 5173 already in use?**
```bash
npm run dev -- --port 5174
```

**Port 8080 already in use?**

`npm run docker:run` always binds host port 8080 (it runs
`docker run -p 8080:8080 …`). The script accepts no port argument, so call
`docker run` directly to map a different host port to the container's fixed
port 8080:

```bash
docker run -p 9090:8080 synapsvault-frontend:latest
# Open http://localhost:9090
```

Replace `9090` with any free port. The container always listens on 8080
internally; only the host side of `-p HOST:8080` changes.

**Container stays unhealthy or won't start?**

The `Dockerfile` configures the health check as:

```
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --start-interval=2s
  CMD wget -qO- http://127.0.0.1:8080/health || exit 1
```

Docker probes `/health` every 2 seconds during the 10-second start period, then
every 30 seconds after that. A probe must respond within 5 seconds. After three
consecutive failures the container is marked `unhealthy`.

**Read the health-check log** to see exactly which probes passed or failed and
what output they returned:

```bash
docker inspect --format '{{json .State.Health}}' web
```

Example output (fictional):

```json
{
  "Status": "unhealthy",
  "FailingStreak": 3,
  "Log": [
    {
      "Start": "2024-11-01T12:00:00.000Z",
      "End":   "2024-11-01T12:00:05.001Z",
      "ExitCode": 1,
      "Output": "wget: can't connect to remote host (127.0.0.1): Connection refused\n"
    }
  ]
}
```

`ExitCode: 0` means the probe succeeded; any other value means it failed.
`Output` contains whatever `wget` (or the error) printed to stdout/stderr.

**Read the container logs** to see nginx startup errors and request logs:

```bash
docker logs web
```

The image runs nginx in the foreground (`daemon off`), so all nginx output —
including startup errors, access logs, and error logs — goes straight to
stdout/stderr and appears in `docker logs`. Add `--tail 50` to limit output or
`-f` to follow in real time:

```bash
docker logs --tail 50 web   # last 50 lines
docker logs -f web           # stream live
```

Common things to look for:

- `[emerg]` lines indicate nginx failed to start (usually a config syntax error).
- `"GET /health HTTP/1.1" 200` confirms health probes are reaching nginx.
- A complete absence of request lines while the status is `unhealthy` means
  the port binding failed before nginx could accept connections.

**Build too large?**
```bash
npm run bundle:stats
# Opens stats.html showing bundle breakdown
```

**Tests failing?**
```bash
npm run test:watch
# Rerun on file changes
```

## Questions?

- Open a GitHub issue
- Check existing PRs for similar work
- Ask in Stellar Dev Discord #synapsvault
