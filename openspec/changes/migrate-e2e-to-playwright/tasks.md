# Tasks

Prerequisite: `add-dev-tooling` is merged (`make lint-shell`, `make lint-markdown`).

## 1. Scaffolding

- [ ] 1.1 Add `@playwright/test@1.59.1` (matching the existing `playwright`) and `eslint-plugin-playwright` to `web/package.json`. Verify: `npm install` succeeds, and `npm ls playwright @playwright/test` shows a single `1.59.1` playwright-core.
- [ ] 1.2 Create the `hack/env/` and `web/e2e/` trees from D1, and `web/e2e/tsconfig.json` extending `../tsconfig.json`. Verify: a scratch file in `web/e2e/support/` importing `DataTestIDs` from `@/shared/constants/data-test` passes `npm run lint:tsc:e2e`.
- [ ] 1.3 Add the npm scripts `test:e2e`, `test:e2e:ui`, `e2e:browsers`, and `lint:tsc:e2e`; add `./e2e` to `lint`; and add `.gitignore` entries for `web/e2e/.env.e2e`, `.e2e-users`, `.auth/`, `test-results/`, and `playwright-report/`. Verify:
  - `git diff web/package.json` touches no Cypress lines.
  - `npm run test:unit` runs no `.e2e.spec` file.
  - `git check-ignore` matches each ignored path.

## 2. Environment setup in `hack/env/` (`environment-setup` spec)

- [ ] 2.1 Implement `lib/common.sh` (log, die, require_tool, wait_until, read_secret, labels, state ConfigMap get/set/delete, env file writer with mode 600). Verify: `make lint-shell` passes, and it can be sourced in `env -i bash --noprofile` without node.
- [ ] 2.2 Implement `web/e2e/suites.sh` (`resolve`, `affected`; D11), plus a `requires.env` in each suite folder, with `suites.test.sh`. Verify that the test covers:
  - `metrics` → `namespaces role:METRICS_TWO_NS`
  - an empty selection → every suite
  - an unknown suite → error listing the suites
  - a folder without `requires.env` → error naming it
  - affected mapping for feature-only, shared, docs-only, and unknown paths, with `navigation` added for UI changes
- [ ] 2.3 Implement `lib/steps.sh` (D3: `STEPS`, context from D4, needed-step expansion, plan printing, `step_main` with `--check`/`--teardown`), plus `setup.sh` and `teardown.sh`, with `steps.test.sh` for the pure logic. Verify the test covers:
  - `role:X` → `users` + `rbac`, and `rbac` → `namespaces`
  - `alert-rules` → `uwm` + `namespaces`, ordered by `STEPS`
  - teardown of `uwm` → adds `alert-rules`, in reverse order
  - a plan with one blocked check → non-zero exit and no apply calls (using stub step functions)

  Verify against a cluster that `hack/env/setup.sh --plan` changes nothing: no labeled objects exist afterwards, and no credentials file is written.
- [ ] 2.4 Implement the `users` step and `create-users.sh` (D5: source precedence, CI file detection, pool parsing, role assignment, one creation path, refusal on `External`), with `users.test.sh` for parsing and precedence. Verify that the test covers:
  - comma, `USERS=`, and per-line formats
  - the exact hosted-cluster fixture (5 lines with `/+=` passwords)
  - both CI files present → `users` wins
  - a malformed line → blocked, with no password printed
  - a credentials file for the same or a different API URL

  Verify on a classic cluster:
  - The first run creates users who can log in.
  - A re-run leaves `oauth/cluster`'s `resourceVersion` unchanged.
  - After deleting the credentials file, the passwords are regenerated and work.
  - No plaintext password appears in any cluster object.

  Verify on HyperShift with no pool: the step is blocked with the remedy.
- [ ] 2.5 Implement the `namespaces` and `rbac` steps (D5, D6), with teardowns. Verify through the console tenancy APIs, logged in as each role:
  - `ALERTING_NS_ADMIN` sees alerts only in its namespace.
  - `METRICS_TWO_NS` can read `metrics-a` and `metrics-b` and is denied `metrics-c`.
  - `DASHBOARDS_VIEWER` can read its namespace.

  Add `monitoring-rules-view` if needed and note the result in the README. `--teardown` removes only labeled objects.
- [ ] 2.6 Implement the `uwm` and `alert-rules` steps (D6), with teardowns. Verify all four `uwm` branches with `--check`: absent, on, present with it off, and CMO scaled down after `hack/scale-down-cmo.sh`. Verify that `alert-rules` finishes only once all three alerts are active, that a re-run is a no-op, and that `E2E_SUITE=metrics hack/env/setup.sh` never reads `cluster-monitoring-config` (check with `bash -x`).
- [ ] 2.7 Implement the `plugin-image` step (D7). Verify:
  - It deploys a given image, and a second run reports `satisfied` without restarting pods.
  - After `hack/scale-down-cmo.sh`, it is blocked with the remedy.
  - An existing override is reused and left in place by teardown.
  - `--teardown` restores the previous image.
- [ ] 2.8 Implement `preflight.sh` (API, console, plugin enabled and manifest served or local manifest, each user's login, prepared alerts firing). Verify: it passes on a prepared cluster, and fails with a named message for a wrong password, for a closed local plugin port, and for a DevSpace pod whose `web/dist` is missing.
- [ ] 2.9 Verify full and partial teardown end to end:
  - After setup for all suites, `hack/env/teardown.sh uwm` removes `alert-rules` then `uwm`, and leaves users, namespaces, and RBAC in place.
  - A following full `hack/env/teardown.sh` restores `oauth/cluster`, `clusterversion.spec.overrides`, `cluster-monitoring-config`, and the plugin image to their pre-setup state, and deletes the credentials file.
  - Overrides or configs that existed before setup are untouched.
- [ ] 2.10 Write `hack/env/README.md`:
  - how it works
  - the step contract and the `operators/README.md` template
  - user sources
  - HyperShift and DevSpace notes
  - the D14 "Common scenarios" with both sample plans

  Verify: `make lint-markdown` passes, and examples 9 and 10 run as written.

## 3. Playwright harness (`test-harness` spec)

- [ ] 3.1 Implement `support/env.ts` (D2) and `playwright.config.ts` (D10). Verify: `npm run test:e2e -- --list` lists only `.e2e.spec` files, and an exported `E2E_BASE_URL` overrides the env file's value.
- [ ] 3.2 Implement `support/auth.setup.ts` and `useRole()` (D8). Verify:
  - The OAuth page is hit once per role across a full run.
  - Single-provider, one-test-provider, and multi-provider-without-`E2E_IDP` logins behave as specified.
  - An unassigned role skips with its name.
  - With auth disabled, only `ADMIN` is used.
- [ ] 3.3 Implement `support/test.ts` (runId, cleanup stack, console log on failure, suite-level skip from `requires.env` against `E2E_PREPARED`) and `support/alertmanager.ts` (D9). Verify with a temporary spec:
  - A silence created through the API is expired by cleanup, even after a thrown error.
  - The alerting tests are skipped with the step named after setup scoped to `metrics`.
- [ ] 3.4 Implement `support/navigation.ts` and the six Page Objects (D12). Add any missing `data-test` IDs to `data-test.ts` and the components. Verify: `make lint-frontend` and `npm run test:unit` pass.
- [ ] 3.5 Configure ESLint for `web/e2e/**` (D10). Verify: scratch specs using `waitForTimeout`, a `../../cypress` import, or a `.pf-` locator each fail `npm run lint`, and a tag other than `@smoke` fails if `valid-test-tags` is available. Then remove the scratch specs.
- [ ] 3.6 Write `web/e2e/README.md`:
  - the placement decision table and naming conventions
  - how to write a flow
  - roles, suite needs, and skips
  - the selection model and known failures

  Verify: `make lint-markdown` passes, and the rules match the ESLint config.

## 4. Monitoring user flows (`monitoring-user-flows` spec)

- [ ] 4.1 `alerting/admin-silence-lifecycle.e2e.spec.ts` (`ADMIN`): filter for `MPE2EAdminAlert` → silence → confirm Silenced → expire → confirm Firing, with cleanup by marker. Verify: it passes with `--repeat-each=3 --workers=1`, and in parallel with 4.2.
- [ ] 4.2 `alerting/non-admin-alerting.e2e.spec.ts` (`ALERTING_NS_ADMIN`; `private-silence` created through the admin API):
  - select the user's namespace
  - `MPE2EPrivateAlert` is not visible
  - silence, confirm, expire, and confirm `MPE2EUserAlert`
  - `private-silence` is not listed

  Verify: it passes with `--repeat-each=3 --workers=1`, and skips when the role has no user.
- [ ] 4.3 `metrics/namespace-metrics.e2e.spec.ts` (`METRICS_TWO_NS`):
  - in `metrics-a`, every series of `kube_pod_info` belongs to `metrics-a`
  - after switching to `metrics-b`, the results change to `metrics-b`
  - a query or URL for `metrics-c` shows no `metrics-c` series, and shows the backend's error or empty state

  Verify: it passes with `--repeat-each=3`.
- [ ] 4.4 `legacy-dashboards/non-admin-dashboards.e2e.spec.ts` (`DASHBOARDS_VIEWER`): a namespace dashboard renders data, cluster-only dashboards aren't offered, and a URL for a forbidden namespace renders no data. Verify: it passes on the reference OCP version.
- [ ] 4.5 `navigation/console-entry-points.e2e.spec.ts` (`@smoke`, `ADMIN`, locators inline per D12), with one test per entry point:
  - Overview → Alerting
  - Project → Alerting
  - Topology → dashboard
  - Topology → Inspect → Metrics
  - Pod Metrics tab → Metrics page
  - Notification drawer → alert details
  - Alert details → Inspect
  - Rule details → Inspect

  Verify: it passes on a local console and on a cluster console, and fails naming the entry point when the plugin is disabled.
- [ ] 4.6 Add the README coverage map (D15) for every file under `web/cypress/e2e/monitoring/`. Verify: every file is listed.

## 5. Entry points and CI image

- [ ] 5.1 Add the Make targets `test-e2e-ui`, `env-plan`, and `test-shell`, and add `lint:tsc:e2e` to `lint-frontend` (D13). Verify:
  - `make test-e2e-ui` exits non-zero on a forced failure and zero on success.
  - `make env-plan E2E_SUITE=metrics` matches the D14 sample plan.
  - `make test-shell` runs every `*.test.sh`.
- [ ] 5.2 Implement `web/e2e/run-ci.sh` and `web/e2e/Dockerfile` (D13). Verify:
  - `podman build -f web/e2e/Dockerfile .` succeeds.
  - Running the image with a mounted kubeconfig and a `SHARED_DIR` fixture holding a `users` file produces `junit.xml` in the artifacts directory, once with `E2E_SUITE=alerting` and once with `E2E_BASE_REF=origin/main`.
  - The README documents the image contract.
- [ ] 5.3 Update `AGENTS.md` with the test conventions and a Playwright section (setup, commands, roles, examples link), and note that Cypress is frozen as a reference. Verify: `make lint-frontend` and `make test-translations` pass.

## 6. Integration verification

- [ ] 6.1 Cypress and the existing hack scripts are untouched: `git diff --stat main -- web/cypress web/cypress.config.ts 'hack/*.sh'` is empty, and `npm run lint:tsc:cypress` passes.
- [ ] 6.2 Runtime budget on a prepared cluster with 2 workers: `@smoke` finishes in under 5 min and the full suite in under 15 min. Record the numbers in the PR.
- [ ] 6.3 Stability: 10 consecutive full runs with `--retries=0` and zero failures.
- [ ] 6.4 Run the same `make test-e2e-ui` in each environment, and check that the plan matches D4/D5:
  - a hosted-cluster htpasswd workflow cluster (`SHARED_DIR/users`, no OAuth change, `user1` as admin)
  - HyperShift with an `idp-htpasswd` `runtime_env`
  - a fresh personal classic cluster, where users are created
  - a re-run, where every step is satisfied
  - HyperShift with no pool, where users are blocked and nothing is applied
- [ ] 6.5 Suite-scoped setup: on a cluster without UWM, `make test-e2e-ui E2E_SUITE=metrics` passes without creating `cluster-monitoring-config` or any PrometheusRule.
- [ ] 6.6 Run every D14 example as written: 1–5 and 7–10 on a personal cluster, 6 with a local backend and console, 11 with `devspace dev` (including the blocked-`uwm` plan), and 12 in the CI image. Fix the README or the code where they differ.
- [ ] 6.7 AGENTS.md pre-PR checks: `make lint-frontend`, `make lint-backend`, `make test-translations`, `make test-backend`, `make test-frontend`, `make lint-shell`, `make lint-markdown`, and `make test-shell` all pass.

## Workflow follow-up

- Add an optional Prow job (the hosted-cluster htpasswd workflow, or `idp-htpasswd`), and make it required once stable.
- Plan the follow-up suites with their operator steps under `hack/env/operators/`, each retiring its Cypress counterpart.
- Add the group-based namespace-admin case.
- Archive this change after review.
