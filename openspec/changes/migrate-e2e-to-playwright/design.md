# Design

## Context

See proposal.md for the motivation and the team doc *Monitoring plugin: Testing enhancements* (OU-1645). These facts shape the design:

- **Tooling dependency.** `add-dev-tooling` lands first and provides `Makefile.tools`, `make lint-shell` (shellcheck + shfmt), and `make lint-markdown`. This change only adds files under the paths those linters cover.
- **Playwright already present.** `web/package.json` has `playwright@1.59.1`, used by Vitest browser mode with `testIdAttribute: 'data-test'`. Test IDs live in `web/src/shared/constants/data-test.ts`.
- **Cypress today.** Cluster setup is mixed into `before()` hooks, configuration comes from ~30 `CYPRESS_*` variables, and users come from `CYPRESS_LOGIN_USERS`.
- **Local development.** The plugin backend runs on `:9443` (`make start-backend`). The console bridge runs on `:9000` (`make start-console`) with `BRIDGE_USER_AUTH=disabled` and user settings in localStorage.
- **DevSpace.** `devspace dev` (choosing `mp`) runs `hack/scale-down-cmo.sh`, which adds a ClusterVersion override for CMO, scales CMO to 0, and replaces the plugin pod with a dev pod syncing `web/dist`. `devspace purge` scales CMO back up and removes **all** overrides.
- **Console proxies.** The console exposes `/api/prometheus[-tenancy]`, `/api/alertmanager[-tenancy]`, and `/api/kubernetes`. Mutations need `X-CSRFToken` matching the `csrf-token` cookie. The tenancy endpoints enforce RBAC by namespace.
- **CI user steps** (openshift/release), neither of which changes OAuth from inside the cluster:
  - `hypershift-hostedcluster-htpasswd` workflow: a fresh HyperShift cluster, an IDP named `htpasswd`, five users `user1`…`user5`, written one `user:pass` per line to `${SHARED_DIR}/users`.
  - `idp-htpasswd`: works on HyperShift and on classic clusters, and writes `export USERS=u1:p1,...` to `${SHARED_DIR}/runtime_env`. Adding an IDP on HyperShift can remove kubeadmin.
- **Clusters are never shared.** They are ephemeral CI clusters or personal developer clusters.

## Goals / Non-Goals

**Goals:**
- One setup command and one test command that work the same locally, on a personal cluster, under DevSpace, and in CI. Only detected facts and the generated env file differ.
- Setup that prepares only what the selected suites need, shows a plan before changing anything, and can tear down fully or partially, so different setups can be swapped on the same cluster.
- Small files with one job each, and no machinery the current scope doesn't need.

**Non-Goals:** operator install steps (only a README template), mocked backends (they belong in Vitest), visual regression, browsers other than Chromium, kubeadmin login, and the group-based namespace-admin case.

## Decisions

### D1. Layout

```
hack/env/
├── README.md               # how it works, common scenarios (D14), step template
├── setup.sh                # context → needed steps → plan → apply → preflight → env file
├── teardown.sh             # [step...] → add dependents → step_teardown in reverse order
├── preflight.sh            # read-only checks, usable standalone
├── lib/
│   ├── common.sh           # log, die, require_tool, wait_until, read_secret, labels, state ConfigMap, env file writer
│   └── steps.sh            # STEPS list, context, dependency expansion, plan printing, step_main
├── users-rbac/
│   ├── users.sh            # step: users
│   ├── create-users.sh     # helper used by users.sh (also runnable alone); clusters with mutable OAuth only
│   └── rbac.sh             # step: rbac
├── operators/README.md     # template for future operator steps
├── patching/plugin-image.sh  # step: plugin-image (includes the CMO unmanaged override)
└── fixtures/
    ├── namespaces.sh       # step: namespaces
    ├── uwm.sh              # step: uwm
    └── alert-rules.sh      # step: alert-rules

web/e2e/
├── playwright.config.ts
├── tsconfig.json
├── README.md               # placement policy, conventions, coverage map, CI image contract
├── Dockerfile
├── suites.sh               # resolve <E2E_SUITE> | affected [base]   (bash)
├── run-ci.sh               # container entrypoint
├── support/
│   ├── env.ts              # env file + process.env → typed config
│   ├── test.ts             # extended test: config, runId, cleanup, useRole(), suite-needs auto-skip
│   ├── auth.setup.ts       # one login per assigned role → .auth/<role>.json
│   ├── alertmanager.ts     # silences through the console proxy
│   ├── navigation.ts       # menu › submenu › tab, selectNamespace
│   └── pages/              # Page Objects for pages used by 2+ specs:
│                           # alerts, silences, alert-details, rule-details, metrics, dashboards
├── alerting/      { requires.env, admin-silence-lifecycle.e2e.spec.ts, non-admin-alerting.e2e.spec.ts }
├── metrics/       { requires.env, namespace-metrics.e2e.spec.ts }
├── legacy-dashboards/ { requires.env, non-admin-dashboards.e2e.spec.ts }
└── navigation/    { requires.env, console-entry-points.e2e.spec.ts }
```

Pure-logic bash functions get `*.test.sh` files next to them: pool parsing in `users.sh`, dependency expansion in `steps.sh`, and resolution and affected mapping in `suites.sh`. `make test-shell` runs them all.

*Why two homes:* `hack/env/` also serves local development without tests, and sits next to the existing `hack/*.sh`. `web/e2e/` reuses `node_modules` and the `@/` alias for `DataTestIDs`.

### D2. Environment contract
`setup.sh` writes `web/e2e/.env.e2e` (git-ignored, mode 600, path overridable with `E2E_ENV_FILE`). `support/env.ts` parses it with Node's `util.parseEnv` and merges it under `process.env`, so exported variables win.

| Variable | Set by | Meaning |
|---|---|---|
| `E2E_BASE_URL` | caller / detected | console URL. Default: the cluster's `console` route |
| `E2E_AUTH_DISABLED` | caller / detected | the local bridge runs without auth |
| `E2E_PLUGIN_URL` | caller | local plugin base, default `http://localhost:9443` |
| `E2E_USERS`, `E2E_USERS_FILE` | caller / CI | provided user pool (D5) |
| `E2E_IDP` | caller | only needed when several non-`kube:admin` providers exist (D8) |
| `E2E_CREDENTIALS_FILE` | — | created-user credentials, default `web/e2e/.e2e-users` |
| `E2E_PLUGIN_IMAGE` | caller | plugin image to deploy (D7) |
| `E2E_NS_PREFIX` | caller | test namespace prefix, default `mp-e2e` |
| `E2E_SUITE`, `E2E_GREP`, `E2E_BASE_REF` | caller | selection (D11), read by setup and by the runner |
| `E2E_ARTIFACTS_DIR` | caller / CI | reports, default `web/e2e/test-results`; `$ARTIFACT_DIR` in CI |
| `E2E_ROLE_<ROLE>_USER/_PASSWORD` | setup | roles: `ADMIN`, `ALERTING_NS_ADMIN`, `METRICS_TWO_NS`, `DASHBOARDS_VIEWER` |
| `E2E_PREPARED` | setup | steps that are satisfied, e.g. `users,namespaces,rbac` |

The names differ from the `CYPRESS_*` ones, so the two suites can't interfere.

### D3. Setup runner and step contract
**Steps.** Each step is one file with this shape:

```bash
STEP_NAME=alert-rules
STEP_REQUIRES="uwm namespaces"
step_check()    { …; }   # read-only; prints: satisfied | apply <detail> | blocked <reason> -- <remedy>
step_apply()    { …; }   # idempotent; records cluster-wide changes in the state ConfigMap; waits until ready
step_teardown() { …; }   # reverts only what the state ConfigMap or labels say this step created
[[ ${BASH_SOURCE[0]} == "$0" ]] && { source "$(dirname "$0")/../lib/steps.sh"; step_main "$@"; }
```

**Order is declared once** in `lib/steps.sh`, which maps each name to its file:

```bash
STEPS=(plugin-image users namespaces rbac uwm alert-rules)
```

Adding a step means one file plus one entry in this list. The list doubles as readable documentation of the order.

| Step | Requires | Does |
|---|---|---|
| `plugin-image` | — | CMO unmanaged override + image patch (D7) |
| `users` | — | resolve or create users, assign roles (D5) |
| `namespaces` | — | every test namespace, with a pause pod (D6) |
| `rbac` | users, namespaces | bindings per assigned role (D5) |
| `uwm` | — | user workload monitoring (D6) |
| `alert-rules` | uwm, namespaces | PrometheusRules, wait until firing (D6) |

**`setup.sh [--plan]`:**
1. **Context** (D4) and **needed steps**. `suites.sh resolve` turns `E2E_SUITE` into suite folders. Their `requires.env` entries (`E2E_REQUIRES="<step>… role:<ROLE>…"`) are combined: `role:X` adds `X` to the needed roles and needs `users` and `rbac`. `plugin-image` is added when `E2E_PLUGIN_IMAGE` is set. `STEP_REQUIRES` is expanded transitively, and the result is filtered through `STEPS` to get the order.
2. **Plan.** `step_check` runs for each needed step and the table is printed. If any step is blocked, setup exits non-zero without applying anything. `--plan` stops here.
3. **Apply.** `step_apply` runs for each `apply` step in order, then `step_check` again, which must report `satisfied`.
4. **Preflight** (`preflight.sh`), then the env file is written, with `E2E_PREPARED` set to the satisfied steps.

**Direct run.** `hack/env/<cat>/<step>.sh [--check|--teardown]` calls `step_main`. It loads the context, requires the step's dependencies to be satisfied (naming the ones that aren't), then checks and applies, checks only, or tears down that step.

**`teardown.sh [step...]`.** With no arguments it covers every step recorded as prepared. With arguments, it adds every prepared step that depends on a named one, for example `uwm` pulls in `alert-rules`. It then runs `step_teardown` in reverse `STEPS` order, expires test-marked silences, and on a full teardown deletes the credentials file for that cluster.

**Ownership.** A labeled `monitoring-plugin-e2e-state` ConfigMap in `openshift-monitoring` records, per step, what that step created and couldn't otherwise mark: the UWM config it created, the CMO override it added and the plugin's previous image, and the IDP and secret it added. Namespaced objects carry `app.kubernetes.io/part-of=monitoring-plugin-e2e` and `monitoring-plugin-e2e/step=<name>`. Teardown only removes what is recorded or labeled. Something recorded but already gone counts as reverted, for example an override removed by `devspace purge`.

**Bash tests** cover only pure logic: dependency expansion, dependents for teardown, and plan formatting from fake check outputs. Cluster behavior is covered by the integration runs in tasks group 6.

*Alternatives considered:*
- *Discovering steps by scanning folders and ordering them with `tsort`.* Rejected because six steps don't justify discovery, cycle detection, and their tests. An explicit list is easier to read.
- *A central detection script plus a requirement registry.* Rejected because every step change would touch several shared files.
- *Driving the step graph with Make* (phony targets like `env-alert-rules: env-uwm env-namespaces` running `script check || script apply`). Rejected for four reasons:
  1. Make decides from file timestamps, while step state lives in the cluster.
  2. The needed steps depend on detection and on the suites, which Make expresses poorly.
  3. Make has no read-only plan phase, so the "refuse if anything is blocked" guarantee would be lost.
  4. Every image and standalone use would need Make.
  
  Make is still used for the entry points (D13).
- *Per-step flags (`--skip`, `--only`, `--interactive`).* Rejected. Running a step file directly replaces `--only`, and nothing needs the others.

### D4. Environment detection, with no "target" setting
Each behavior follows its own detected fact. The plan header prints a descriptive label (`local`, `cluster`, `ci`, `devspace`) only to help people read it.

| Fact | Detection | Used by |
|---|---|---|
| Console URL | `E2E_BASE_URL`, else the `console` route in `openshift-console` | preflight, env file |
| Local console | `E2E_BASE_URL` is a localhost URL | preflight checks `E2E_PLUGIN_URL/plugin-manifest.json` instead of the console operator and console proxy |
| Auth disabled | `E2E_AUTH_DISABLED`, else probe whether the local bridge redirects `/` to an OAuth login | env file → only `ADMIN` is usable |
| CI user file | `${SHARED_DIR}/users`, then `${SHARED_DIR}/runtime_env` | `users` step |

Facts that only one step needs stay inside that step: the control-plane topology (`users`), the UWM state (`uwm`), and whether CMO is scaled down (`uwm`, `plugin-image`).

### D5. Users and RBAC
**Sources, first match wins** (`users` step):

| # | Source | OAuth changed? |
|---|---|---|
| 1 | `E2E_USERS` / `E2E_USERS_FILE`, else a CI user file (D4) | no |
| 2 | `E2E_CREDENTIALS_FILE` whose header records the current API URL | no |
| 3 | create users (`create-users.sh`), if `controlPlaneTopology` is not `External` | yes |
| — | none → **blocked**: "provide `E2E_USERS[_FILE]`, or use the hosted-cluster htpasswd or `idp-htpasswd` CI steps" | — |

- **Parsing.** If a file contains `USERS=`, it is sourced in a subshell and only `USERS` is read. Otherwise each non-empty, non-comment line is one `user:pass`. Entries split on the first `:`. A malformed entry blocks the step and names its position. Passwords never appear in logs or the plan.
- **Roles.** Only the needed roles are assigned, from the pool in the fixed order `ADMIN`, `ALERTING_NS_ADMIN`, `METRICS_TWO_NS`, `DASHBOARDS_VIEWER`. Roles left without a user are listed, and their tests skip. `ADMIN` is always a pool or created user bound to `cluster-admin`, never kubeadmin.
- **Creation, in one code path.** When no credentials file matches the cluster, `create-users.sh`:
  1. generates passwords with `openssl rand -base64 18` for `mp-e2e-admin`, `mp-e2e-alerting-ns-admin`, `mp-e2e-metrics-two-ns`, and `mp-e2e-dashboards-viewer`;
  2. writes `openssl passwd -apr1` hashes to a labeled secret in `openshift-config`, creating or replacing it;
  3. appends the `e2e-htpasswd` IDP to `oauth/cluster` if absent, and records it;
  4. waits until the first user can log in;
  5. writes the credentials file with mode 600 and the API URL header.
  
  The first run and a lost file are the same path.
- **Bindings** (`rbac` step, labeled):

  | Role | Binding |
  |---|---|
  | `ADMIN` | `cluster-admin` ClusterRoleBinding |
  | `ALERTING_NS_ADMIN` | `admin` in `${prefix}-alerting-user` |
  | `METRICS_TWO_NS` | `view` in `${prefix}-metrics-a` and `-metrics-b`, nothing in `-metrics-c` |
  | `DASHBOARDS_VIEWER` | `view` in `${prefix}-dashboards` |

  If verification shows that `view` or `admin` doesn't grant tenancy access to alerts, add `monitoring-rules-view`.
- **Pool size.** The hosted-cluster workflow creates 5 users, enough for 4 roles. The later Perses RBAC suite needs more, either through an upstream user-count parameter or by using `idp-htpasswd`.

*Alternative considered:* impersonation or ServiceAccount tokens. Rejected because RBAC must be exercised through real console login.

### D6. Fixtures
- **`namespaces`:** `${prefix}-{alerting-admin,alerting-user,alerting-private,metrics-a,metrics-b,metrics-c,dashboards}`, labeled, each with a tiny pause Deployment so namespace-labeled series (`kube_pod_info`) exist. It waits until `kube_pod_info` is queryable for each namespace. All namespaces are created in one step because this is cheap (seconds), even for suites that use only some of them.
- **`uwm`:**
  - If `cluster-monitoring-config` is absent, create it with `enableUserWorkload: true`, record it, and wait for `prometheus-user-workload` and `thanos-ruler-user-workload`.
  - If it is already enabled, the step is `satisfied`.
  - If a config exists with it off, the step is blocked and shows the snippet to add. Setup doesn't edit YAML.
  - If CMO has 0 replicas and UWM isn't running, the step is blocked with the remedy: "run `hack/env/setup.sh` before `devspace dev`, or `hack/scale-up-cmo.sh` temporarily".
- **`alert-rules`:** one PrometheusRule (`vector(1)`, `for: 0s`, severity `warning`) in each alerting namespace: `MPE2EAdminAlert`, `MPE2EUserAlert`, and `MPE2EPrivateAlert`. It polls the Alertmanager v2 API (`oc exec` into `alertmanager-main-0`) until all three are active, with a timeout that names the alert.

*Why ahead of time:* UWM and alert propagation take minutes. Paying that once in setup, not per test, is the biggest runtime and flakiness win over Cypress.

### D7. Plugin image
This step is ported from development-tools' `update-monitoring-plugin-image.sh`, without the fixed `sleep 30`:
1. If CMO has 0 replicas (DevSpace or a manual scale-down), the step is blocked with the remedy "unset `E2E_PLUGIN_IMAGE`, or run `devspace purge` first".
2. If the running image already matches, the step is `satisfied`.
3. Otherwise:
   - add the ClusterVersion override marking CMO unmanaged, unless an override already exists; one we didn't add is reused and not recorded;
   - record the previous image;
   - JSON-patch the `-images=monitoring-plugin=` argument;
   - wait on `oc rollout status deployment/monitoring-plugin`.

Teardown restores the previous image and removes only an override this step recorded. The Cypress copy of the script is untouched.

### D8. Authentication
`support/auth.setup.ts` runs as Playwright's `setup` project. For each assigned role it:
1. opens `/`;
2. handles the provider picker if there is one: if exactly one provider besides `kube:admin` is offered, it picks that one; if several are, it uses `E2E_IDP` and fails listing the offered names when that is unset or absent;
3. submits the credentials and waits for the masthead;
4. dismisses the guided tour;
5. saves `.auth/<role>.json`.

When auth is disabled, it only dismisses the tour and produces `ADMIN`.

A spec selects its role with `useRole('ALERTING_NS_ADMIN')`. That sets `storageState` and skips the test with the role's name when the role has no user.

### D9. Test data through the console proxy
`support/alertmanager.ts` uses an `APIRequestContext` built from a role's `storageState` against `/api/alertmanager[-tenancy]/api/v2/silences`, adding `X-CSRFToken` for mutations. The flows need only silences: `private-silence` setup, cleanup, and state cross-checks. Prometheus results are asserted through the UI.

*Alternative considered:* shelling out to `oc` from tests. Rejected because it needs a kubeconfig and is slow.

### D10. Flow structure, isolation, and synchronization
- **One test per flow.** Each flow is one `test()` with a `test.step()` per doc step. Flows use disjoint namespaces and alerts, so files run in parallel (`workers: 2` in CI) while staying serial within a file.
- **Markers and cleanup.** A worker `runId` is used. Silences get `createdBy: monitoring-plugin-e2e` and `comment: e2e <runId> <flow>`. A test-scoped `cleanup` stack runs in fixture teardown, even after a failure.
- **Navigation.** Tests open plugin routes directly with `page.goto('/monitoring/...')`. Only the navigation spec clicks through the console.
- **Config:**
  - `testDir: 'e2e'`, `testMatch: '**/*.e2e.spec.{ts,tsx}'`, `testIdAttribute: 'data-test'`
  - timeouts: expect and action 15s, navigation 60s, test 180s
  - retries `CI ? 1 : 0`; `trace: 'retain-on-failure'`, `screenshot: 'only-on-failure'`, no video
  - reporters: `list`, `html`, and `junit`, all to `E2E_ARTIFACTS_DIR`
  - projects: `setup`, then `chromium`
  - a fixture attaches the browser console log on failure
- **Lint** (`eslint-plugin-playwright`, scoped to `web/e2e/**`):
  - `no-wait-for-timeout`, `no-networkidle`, `no-force-option`, `prefer-web-first-assertions`, `no-conditional-in-test`
  - `no-skipped-test`, allowing conditional skips
  - `no-restricted-imports` for `**/cypress/**`
  - `no-restricted-syntax` for `.pf-` class selectors
  - `valid-test-tags` restricted to `@smoke`, only if the plugin version has it. Otherwise the README documents the rule, with no custom rule.

### D11. Selection
| Concern | Mechanism |
|---|---|
| Feature | the suite folder: `E2E_SUITE=alerting,metrics` |
| Changed code | `E2E_SUITE=affected` (with `E2E_BASE_REF`, default `origin/main`) |
| Critical path | `E2E_GREP=@smoke`, the only tag |
| Environment | `requires.env` compared with `E2E_PREPARED` (below) |
| Known failure | `test.fail()` / `test.fixme()` plus an `issue` annotation |

- **`suites.sh resolve`** validates names against folders that have a `requires.env`, and fails listing the available suites. It is used by setup, `npm run test:e2e`, and `run-ci.sh`.
- **`suites.sh affected [base]`** diffs `base...HEAD` through a `case` table:

  | Changed paths | Suites |
  |---|---|
  | `web/src/features/alerts/**` | `alerting` |
  | `web/src/features/metrics/**` | `metrics` |
  | `web/src/features/legacy-dashboards/**` | `legacy-dashboards` |
  | shared and global paths (listed below) | all |
  | `*.md`, `docs/**`, Cypress-only paths | none |

  Shared and global paths are `web/src/shared/**`, `web/e2e/support/**`, `web/e2e/playwright.config.ts`, `hack/env/**`, `config/*.patch.json`, `web/package*.json`, `pkg/**`, `cmd/**`, `go.*`, `Dockerfile*`, `web/webpack.config.ts`, and any unknown path. Any UI change also adds `navigation`. If nothing is selected, the command exits 0 with "nothing to run".
- **Suite-level skip.** An automatic fixture in `support/test.ts` reads the current spec's `<suite>/requires.env` and skips the test, naming the missing step, when a declared step isn't in `E2E_PREPARED`. Roles are handled by `useRole()` (D8). No per-test calls or lint script are needed.
- **Prow.** Each job (`/test e2e-<suite>`) is the same image with a different `E2E_SUITE`.

*Alternatives considered:*
- *Feature tags alongside folders.* Rejected because a second axis drifts from the folders, which is how the Cypress script matrix grew.
- *`playwright --only-changed`.* Rejected because specs don't import feature source, so it can't see which features changed.

### D12. Page Objects and utilities
- **Page Objects** exist only for pages used by two or more specs: `alerts`, `silences`, `alert-details`, `rule-details`, `metrics`, and `dashboards`. Each is a small class with locator getters and the doc's repeated steps (`filterByName`, `silence`, `expectState`, `expire`). Flow assertions stay in the specs.
- **One-off console pages** (overview, project, topology panel, pod Metrics tab, notification drawer) keep their locators at the top of `console-entry-points.e2e.spec.ts`. They are located by role, name, or console `data-test` IDs.
- **`navigation.ts`** provides `navigateTo(page, ['Observe','Alerting'], 'Silences')` and `selectNamespace(page, ns)`, which waits for the route to reflect the namespace.

### D13. Entry points and CI image
- **npm:** `test:e2e` (resolves `E2E_SUITE` through `suites.sh`), `test:e2e:ui` (UI mode), `e2e:browsers` (`playwright install chromium`), and `lint:tsc:e2e`. The `lint` script includes `./e2e`.
- **Make:**
  - `make test-e2e-ui [E2E_SUITE=…] [E2E_GREP=…]`: `npm run e2e:browsers`, `hack/env/setup.sh`, then `npm run test:e2e`.
  - `make env-plan`: `hack/env/setup.sh --plan`.
  - `make test-shell`: the bash `*.test.sh` files.
  - `make lint-frontend` adds `lint:tsc:e2e`.
  
  Setup and teardown otherwise run as plain scripts.
- **`web/e2e/Dockerfile`:**
  - base image `mcr.microsoft.com/playwright:v1.59.1-noble`, whose browsers match the pinned version;
  - `oc` downloaded once with a pinned `ARG OC_VERSION`, the only place this version lives, since developers use their own `oc`;
  - copies `hack/env` and `web/`, then runs `npm ci`;
  - entrypoint `run-ci.sh`.
- **`run-ci.sh`:**
  1. runs `hack/env/setup.sh`, which detects the CI user file and uses `E2E_PLUGIN_IMAGE` if set;
  2. runs `npm run test:e2e` with `E2E_SUITE`, which defaults to `affected` when `E2E_BASE_REF` is set and to all suites otherwise;
  3. always copies reports to `$ARTIFACT_DIR`, keeping the test exit code.
- **Prow (follow-up):** the `hypershift-hostedcluster-htpasswd` workflow is recommended, since it covers the cluster, users, and cleanup. The alternative is `cluster_claim` with `idp-htpasswd`.

### D14. Command examples
These are also the README's "Common scenarios". Each is checked by running it (tasks 2.10 and 6.6).

```bash
# 1. Personal cluster, first time (users get created)
export KUBECONFIG=~/clusters/dev/kubeconfig
make env-plan                        # optional: see what setup would do
make test-e2e-ui                     # setup for all suites, then run all tests

# 2. Later runs: every step "satisfied", nothing changes
make test-e2e-ui

# 3. One suite: prepares only what metrics needs (no UWM, no alert rules)
make test-e2e-ui E2E_SUITE=metrics

# 4. Smoke tests, or the suites affected by your branch
make test-e2e-ui E2E_GREP=@smoke
make test-e2e-ui E2E_SUITE=affected E2E_BASE_REF=origin/main

# 5. Test a PR image on your cluster
make test-e2e-ui E2E_PLUGIN_IMAGE=quay.io/<you>/monitoring-plugin:pr-1234 E2E_SUITE=alerting

# 6. Local frontend against a remote cluster (auth disabled → admin-only flows)
make start-backend                   # terminal 1
make start-console                   # terminal 2
make test-e2e-ui E2E_BASE_URL=http://localhost:9000

# 7. Bring your own users (e.g. a personal HyperShift cluster)
E2E_USERS='alice:pw1,bob:pw2,carol:pw3,dave:pw4' make test-e2e-ui

# 8. Re-run tests only, without setup
cd web && E2E_SUITE=alerting npm run test:e2e

# 9. Check, apply, or tear down a single step
hack/env/fixtures/alert-rules.sh --check
hack/env/fixtures/alert-rules.sh
hack/env/fixtures/alert-rules.sh --teardown

# 10. Switch setups on the same cluster, or clean up everything
hack/env/teardown.sh uwm             # also removes alert-rules, which depends on it
hack/env/teardown.sh                 # everything setup created

# 11. DevSpace: your local build runs in a dev pod inside the cluster
hack/env/setup.sh                    # first time, BEFORE devspace (CMO must run to enable UWM)
make start-frontend                  # terminal 1
devspace dev                         # terminal 2: choose "mp"
make test-e2e-ui                     # terminal 3 (don't set E2E_PLUGIN_IMAGE)
devspace purge                       # when done

# 12. CI test step (SHARED_DIR, KUBECONFIG, ARTIFACT_DIR set by ci-operator)
E2E_SUITE=alerting web/e2e/run-ci.sh
```

Plan for example 3 on a fresh personal cluster:

```text
cluster · https://console-openshift-console.apps.dev…
suites   metrics → namespaces, role:METRICS_TWO_NS

STEP         STATUS   DETAIL
users        apply    no pool or credentials file; OAuth mutable → create mp-e2e-metrics-two-ns
namespaces   apply    7 test namespaces + pause pods
rbac         apply    METRICS_TWO_NS: view in mp-e2e-metrics-a, mp-e2e-metrics-b
```

Plan for `E2E_SUITE=alerting` under DevSpace when UWM was never enabled. Nothing is applied:

```text
devspace · CMO scaled down
STEP         STATUS     DETAIL
users        satisfied  credentials file matches this cluster
namespaces   satisfied
rbac         satisfied
uwm          blocked    CMO has 0 replicas, so UWM can't be deployed
                        -- run hack/env/setup.sh before devspace dev, or hack/scale-up-cmo.sh temporarily
alert-rules  apply      (not applied: plan has blocked steps)
```

### D15. Placement policy and what isn't migrated
The README holds the unit/component/E2E decision table and a coverage map that sorts each Cypress core-monitoring spec into one of: covered by a flow, a Vitest candidate, or dropped. These are not recreated in E2E: filter permutations, silence form validation, Metrics action and kebab states, units and refresh dropdowns, CSV export formatting, empty states, stacked and disconnected toggles, hide/show graph persistence, and the Targets list.

## Risks / Trade-offs

- **[Risk] UWM startup and alert propagation are slow.** → Mitigation: wait on actual Alertmanager state with a timeout that names the alert, and pay the cost once per cluster.
- **[Risk] An existing `cluster-monitoring-config` has UWM off, or CMO is scaled down.** → Mitigation: the `uwm` step is blocked with a remedy and nothing is applied. This only matters for suites that need UWM.
- **[Risk] The credentials file is lost, or belongs to a recreated cluster.** → Mitigation: the API URL header; a mismatch means new passwords and one OAuth rollout.
- **[Risk] A CI step changes its user file format or IDP name.** → Mitigation: content-based parsing with bash tests for both formats, IDP auto-selection, override variables, and a preflight login per user.
- **[Risk] The pool is too small, or the CI user step is still rolling out.** → Mitigation: roles without a user skip with a reason, and preflight checks each user's login.
- **[Risk] Built-in `view` or `admin` lacks tenancy access to alerts.** → Mitigation: verified in the RBAC task; add `monitoring-rules-view` if needed.
- **[Risk] Under DevSpace, files sync mid-run, or the dev pod isn't serving yet.** → Mitigation: preflight fetches the manifest, the README says not to edit files during a run, and each context starts with a fresh cache.
- **[Risk] Console-owned markup changes upstream.** → Mitigation: those locators are concentrated in one spec and the Page Objects, preferring role and name.
- **[Risk] Built-in dashboard names change between OCP versions.** → Mitigation: names are constants, and assertions check that data is present.
- **[Risk] The affected selector misses an indirect dependency.** → Mitigation: unknown paths run everything, and a full run happens on merge or nightly (Prow follow-up).
- **[Trade-off] Every suite's setup creates all 7 test namespaces.** → They're empty and take seconds, in exchange for one simple step.
- **[Trade-off] Coverage looks lower than Cypress during the transition.** → Mitigation: Cypress keeps running, and the README coverage map tracks each area.

## Migration Plan

1. Land `add-dev-tooling`.
2. Land this change. Cypress jobs are unchanged.
3. Validate the same commands on a fresh personal classic cluster, a HyperShift cluster with a provided pool, and DevSpace. Reach the runtime budget and 10 consecutive green runs.
4. Follow-up: an optional Prow job, made required once stable.
5. Follow-up changes: operator steps and the COO, Incidents, Logging, Tracing, Troubleshooting, Perses, and Virtualization suites, retiring their Cypress counterparts.

**Rollback:** delete `hack/env/`, `web/e2e/`, and the added scripts and targets.

## Open Questions

- Which cluster profile is the reference for the runtime budget? This doesn't affect the implementation.
