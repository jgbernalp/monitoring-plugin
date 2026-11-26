# Proposal

## Why

To let automated CVE PRs merge on their own and let agents change code without regressions, the monitoring plugin needs faster, more trustworthy tests (OU-1645, OU-1542; see the team doc *Monitoring plugin: Testing enhancements*). The Cypress suite has about 52 specs and about 27k lines including support code. It is slow, flaky, and duplicated across admin, namespaced, and perspective variants. It mixes cluster setup into test hooks, and it re-checks component behavior through a live cluster. E2E should cover only what needs a real cluster, such as RBAC and console integration, and the same commands should run locally and in CI.

## What Changes

- **Test conventions:**
  - Unit tests sit next to the code they test (`*.spec.ts(x)`).
  - Component tests sit next to their component (`*.browser.spec.tsx`).
  - E2E tests live in `web/e2e/<suite>/<flow>.e2e.spec.ts(x)`.
  - The README documents a placement policy saying what belongs in E2E.
- **Environment setup in `hack/env/`**, in bash with no TypeScript build:
  - **One command** (`hack/env/setup.sh`) for a personal cluster, a local frontend, DevSpace, and CI. It detects what it needs (console URL, auth, user source) instead of asking for an environment name.
  - **Plan before apply:** it checks every needed step, prints a plan, refuses to start if any step is blocked, then applies only what is missing.
  - **Scoped to the selected suites:** each suite declares the steps and roles it needs, and setup prepares only those. For example, `metrics` never touches user workload monitoring.
  - **Six small steps, one file each:** `plugin-image`, `users`, `namespaces`, `rbac`, `uwm`, and `alert-rules`. Each has a check, an apply, and a teardown, and each can run on its own. The order is listed in one place.
  - **Full or partial teardown** through recorded ownership, so different setups can be swapped on the same cluster.
  - **Test users**, first match wins:
    1. a provided pool, auto-detected from the CI files of `hypershift-hostedcluster-htpasswd` or `idp-htpasswd`;
    2. a local credentials file;
    3. users created on clusters with mutable OAuth, whose passwords stay on the runner's machine.
- **Playwright suite in `web/e2e/`:**
  - One test per user flow, written as steps.
  - Login once per role, with automatic identity provider selection.
  - Page Objects only for pages shared by two or more specs.
  - Test data through the console API.
  - Lint-enforced waiting rules (no fixed sleeps).
  - Traces on failure, plus JUnit and HTML reports.
- **First coverage, the doc's monitoring flows:** admin and non-admin alerting (silence lifecycle and visibility), metrics across permitted and forbidden namespaces, legacy dashboard permissions, and console navigation entry points. Component-level Cypress checks aren't recreated.
- **Selection:**
  - Suites are folders, selected with `E2E_SUITE=<suites>`, or `affected`, which maps changed paths to suites.
  - `@smoke` is the only tag.
  - Known failures are annotated, not hidden.
  - A suite is skipped with a reason when setup didn't prepare what it needs.
- **Entry points:**
  - Make targets `make test-e2e-ui` (setup then tests), `make env-plan`, and `make test-shell` (bash unit tests).
  - npm scripts `test:e2e*`.
  - A CI image, `web/e2e/Dockerfile`.
- **Cypress untouched:** `web/cypress/` keeps running as the reference until each area is migrated. There's no 1:1 port.
- **Depends on `add-dev-tooling`**, which provides `make tools` and the shell and Markdown linters used for the new files.

## Capabilities

### New Capabilities

- `e2e/environment-setup`: One bash command that adapts to the environment, plans and applies only the steps the selected suites need, resolves test users, and tears down fully or partially. It emits the configuration the tests consume.
- `e2e/test-harness`: Conventions and shared infrastructure for Playwright E2E tests: placement, layout, Page Objects, authentication, isolation, synchronization, selection, and reporting.
- `e2e/monitoring-user-flows`: The cluster-dependent user flows for the core monitoring plugin: admin and non-admin alerting, namespace-scoped metrics, legacy dashboard permissions, and console navigation entry points.

### Modified Capabilities

<!-- None: there are no existing specs in openspec/specs/. -->

## Impact

- **New:** `hack/env/` (about 12 files), `web/e2e/` (config, `support/`, 4 suite folders, `suites.sh`, `run-ci.sh`, `Dockerfile`, README).
- **Modified:**
  - `web/package.json`: `@playwright/test` and `eslint-plugin-playwright`, plus the `test:e2e*`, `e2e:browsers`, and `lint:tsc:e2e` scripts, and `./e2e` added to `lint`.
  - `web/eslint.config.ts`: E2E rules.
  - `Makefile`: the `test-e2e-ui`, `env-plan`, and `test-shell` targets, and `lint:tsc:e2e` added to `lint-frontend`.
  - `.gitignore`.
  - `AGENTS.md`.
  - `data-test.ts` and components, only where a flow needs a missing test ID.
- **Unchanged:** `web/cypress/**`, the Cypress scripts and jobs, the existing `hack/*.sh` scripts, and `make test-e2e` (Go).
- **Cluster effects:** every change is decided by detection and by the selected suites, is idempotent and labeled or recorded, and is reversible by teardown. That covers role bindings, test namespaces and PrometheusRules, user workload monitoring, the CMO override and plugin image, and htpasswd users (mutable-OAuth clusters only).
- **Follow-ups:**
  - A Prow job: the `hypershift-hostedcluster-htpasswd` workflow, or `idp-htpasswd` on a claimed cluster.
  - More than 5 CI users for the later Perses RBAC suite.
  - Operator steps and the COO, Incidents, Logging, Tracing, Troubleshooting, Perses, and Virtualization suites.
  - The group-based namespace-admin case.
  - Moving Cypress-only component checks into Vitest.
  - Retiring Cypress specs as each area is migrated.
