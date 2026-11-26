# Spec Delta

## Purpose

Defines the conventions and shared infrastructure every Playwright E2E test relies on (placement, layout, Page Objects, authentication, isolation, synchronization, selection, and reporting) so the suite stays fast, deterministic, and free of duplication as more areas migrate.

## ADDED Requirements

### Requirement: Test placement policy
An E2E test SHALL exist only for behavior that needs a real console and cluster:
- plugin integration with console pages and navigation;
- real round-trips with Prometheus, Alertmanager, and the Kubernetes API;
- RBAC.

Rendering, validation, and formatting logic SHALL be covered by unit or component tests.

#### Scenario: Reviewing a proposed E2E test
- **WHEN** a new E2E test only asserts form validation messages, dropdown options, or empty-state rendering driven by props
- **THEN** it is rejected under the documented placement policy and redirected to unit or component tests

#### Scenario: Placement policy is documented
- **WHEN** a contributor opens the E2E README
- **THEN** it contains a decision table mapping kinds of behavior to unit, component, or E2E tests

### Requirement: Test folder and naming conventions
Each test type SHALL follow its own location and naming rule:
- Unit tests sit next to the code they test, named `*.spec.ts(x)`.
- Component tests sit next to their component, named `*.browser.spec.tsx`.
- E2E tests live in `web/e2e/<suite>/`, named `<user-flow>.e2e.spec.ts(x)`.

Each runner SHALL collect only its own files.

#### Scenario: E2E file discovery
- **WHEN** the E2E runner lists tests
- **THEN** it includes only `*.e2e.spec.ts(x)` files under `web/e2e/`, and no unit, component, or Cypress files

#### Scenario: Unit runner ignores E2E specs
- **WHEN** the unit test command runs
- **THEN** no `*.e2e.spec.*` file is executed

### Requirement: One test per user flow
Each documented user flow SHALL be a single test whose steps are reported individually. A flow SHALL be written once and parameterized when a dimension such as namespace changes behavior, instead of being copied.

#### Scenario: Step-level reporting
- **WHEN** a step in a user flow fails
- **THEN** the report shows the flow's name and the failing step's name

### Requirement: Page Objects for shared pages, utilities for shared actions
A console page or view used by two or more spec files SHALL have one Page Object holding its locators and repeated steps. Pages used by a single spec SHALL keep their locators in that spec. Shared utilities SHALL provide logging in as a test role, menu › submenu › tab navigation, and switching the active namespace.

#### Scenario: Navigating by menu path
- **WHEN** a test needs the Silences tab under Observe › Alerting
- **THEN** it calls the shared navigation utility with that path

#### Scenario: Switching namespace
- **WHEN** a test needs to scope a page to a namespace
- **THEN** it uses the shared namespace utility, which waits until the page reflects the selection

#### Scenario: Page used by one spec
- **WHEN** only the navigation spec interacts with the Topology side panel
- **THEN** its locators live in that spec, and no Page Object is created for it

### Requirement: Authenticate once per role per run
The harness SHALL log each assigned role in through the console once per run, persist the session, and reuse it for every test that runs as that role. Tests SHALL NOT perform UI login themselves. Selecting a role whose user is not assigned SHALL skip the test with a reason.

#### Scenario: Multiple tests for one role
- **WHEN** several tests run as the admin role
- **THEN** the login page is visited once for that role during the run

#### Scenario: Role not assigned
- **WHEN** a test selects a role that has no user assigned
- **THEN** the test is reported as skipped with a message naming the role

#### Scenario: Auth-disabled console
- **WHEN** the console runs with user auth disabled
- **THEN** login is skipped, and tests that select a non-admin role are skipped with a reason

#### Scenario: Login fails
- **WHEN** an assigned user's credentials are rejected
- **THEN** tests for that role do not run, and the error names the user and identity provider

### Requirement: Identity provider selected without configuration
When the login page offers no provider choice, the harness SHALL submit credentials directly. When it offers exactly one provider besides `kube:admin`, the harness SHALL select that provider. A provider name SHALL be required only when several are offered.

#### Scenario: Single test provider
- **WHEN** the login page offers `kube:admin` and `htpasswd`, and no provider is configured
- **THEN** the harness selects `htpasswd`

#### Scenario: Several providers
- **WHEN** the login page offers several providers besides `kube:admin`, and the configured one is missing or absent from the page
- **THEN** login fails and lists the offered providers

### Requirement: Locators use stable test IDs
Tests SHALL locate plugin-owned elements by `data-test` attributes from the shared test-ID constants, or by accessible role and name. CSS class names and DOM structure SHALL NOT be used for plugin-owned elements.

#### Scenario: New locator needed
- **WHEN** a test must interact with a plugin element that has no test ID
- **THEN** a test ID is added to the shared constants and the component, and the test imports the constant

### Requirement: Deterministic synchronization
Tests SHALL wait only through auto-retrying assertions, or through explicit waits on specific network responses or element states. Fixed-duration sleeps SHALL NOT be used.

#### Scenario: Waiting for query results
- **WHEN** a test runs a PromQL query
- **THEN** it waits for the query response and a rendered result, not for a fixed delay

#### Scenario: Linting enforces the rule
- **WHEN** an E2E file contains a fixed-duration wait
- **THEN** the frontend lint step reports an error

### Requirement: Test isolation and parallel safety
Each user flow SHALL be independent of execution order. Different flows SHALL use disjoint fixture namespaces and alerts so that they can run in parallel. State a test changes SHALL carry a run-scoped marker and SHALL be cleaned up even when the test fails.

#### Scenario: Parallel flows
- **WHEN** the admin and non-admin alerting flows run at the same time
- **THEN** each silences only its own fixture alert, and neither affects the other's assertions

#### Scenario: Test aborted mid-flow
- **WHEN** a flow fails after creating a silence
- **THEN** cleanup still expires that silence, and teardown removes any leftovers carrying the test marker

### Requirement: Test data comes from fixtures, not UI
Preconditions SHALL come from environment setup or from monitoring APIs. The UI SHALL be used only for the behavior under test.

#### Scenario: Silence that must be hidden
- **WHEN** a flow needs a silence that the tested user must not see
- **THEN** that silence is created through the API as admin before the flow starts, and removed afterward

### Requirement: Suites declare their needs and skip when unmet
Each suite folder SHALL declare the setup steps and roles it needs in a file readable by both bash and the test runner. When the environment file does not report a declared step as prepared, the suite's tests SHALL be skipped with a reason naming the step. Callers SHALL NOT need to exclude tests for environments that lack something.

#### Scenario: Step not prepared
- **WHEN** the `alerting` suite runs after setup was scoped to `metrics` only
- **THEN** the alerting tests are reported as skipped, naming the missing alert-rules step

#### Scenario: Missing declaration file
- **WHEN** a suite folder has no declaration file
- **THEN** suite resolution fails and names the folder

### Requirement: Suite selection by folder
Suite folders SHALL be the only way to select tests by feature. The E2E entry points SHALL accept a comma-separated list of suites, the value `affected`, or nothing (all suites), and SHALL behave the same locally and in CI.

#### Scenario: Run one suite
- **WHEN** a caller selects suite `alerting`
- **THEN** only tests under `web/e2e/alerting/` run

#### Scenario: Unknown suite
- **WHEN** a caller selects a suite that has no folder
- **THEN** the entry point exits non-zero before running tests and lists the available suites

### Requirement: Affected-suite selection
Selecting `affected` SHALL map the files changed against a base revision to suites, without a TypeScript build:
- A change confined to one feature runs that feature's suite.
- A change to shared code, the harness, setup scripts, plugin configuration, dependencies, or an unrecognized path runs every suite.
- A change that cannot affect the UI runs none.

#### Scenario: Feature-only change
- **WHEN** a diff touches only alerting feature source
- **THEN** only the `alerting` suite runs, plus `navigation`, which covers every UI change

#### Scenario: Shared change
- **WHEN** a diff touches shared frontend code or `web/package.json`
- **THEN** every suite runs

#### Scenario: Documentation-only change
- **WHEN** a diff touches only Markdown files outside `web/e2e`
- **THEN** no suite runs, and the command exits zero

### Requirement: Tags only for cross-cutting concerns
Tags SHALL be used only for concerns that span suites, and SHALL NOT name a feature, an environment, or a test status. The only tag SHALL be `@smoke`, which marks the fast critical path.

#### Scenario: Smoke run across suites
- **WHEN** a caller filters to `@smoke`
- **THEN** only smoke-tagged tests run, from every selected suite

### Requirement: Known failures are tracked, not hidden
A test known to be broken SHALL be marked with the runner's fixme or expected-failure annotation, together with an issue link. It SHALL stay visible in reports, and SHALL NOT be excluded by tags.

#### Scenario: Known bug
- **WHEN** a flow fails because of a tracked product bug
- **THEN** the test is annotated as expected-to-fail with the issue link, and the report shows it as a known failure

#### Scenario: Known failure fixed
- **WHEN** a test annotated as expected-to-fail starts passing
- **THEN** the run reports it so the annotation can be removed

### Requirement: Failure diagnostics
For each failed test the harness SHALL keep a trace, a screenshot, and the browser console log. Every run SHALL produce an HTML report and a JUnit XML report.

#### Scenario: Failure in CI
- **WHEN** a test fails in CI
- **THEN** its trace, screenshot, and console log are written to the artifacts directory alongside `junit.xml`

#### Scenario: Passing tests produce no heavy artifacts
- **WHEN** a test passes on its first attempt
- **THEN** no trace or video is kept for it

### Requirement: Bounded retries with flake visibility
Retries SHALL be off locally and at most one in CI. A test that passes only on retry SHALL be reported as flaky.

#### Scenario: Flaky test in CI
- **WHEN** a test fails once and passes on retry in CI
- **THEN** the run passes and the report lists the test as flaky

### Requirement: Coexistence with the Cypress suite
The Playwright suite SHALL NOT modify, import from, or depend on files under `web/cypress/`. The existing Cypress commands and CI jobs SHALL keep working unchanged.

#### Scenario: Cypress still runs
- **WHEN** the Playwright suite is added
- **THEN** the existing Cypress run, lint, and type-check commands behave as before

### Requirement: Single entry points
The suite SHALL run through one npm script and one Make target that also runs setup. The standard frontend lint commands SHALL lint and type-check it.

#### Scenario: Run through Make
- **WHEN** a user runs the Make target with suite `metrics`
- **THEN** setup prepares what `metrics` needs, the `metrics` suite runs, and the command exits non-zero on any failure

#### Scenario: Lint covers E2E code
- **WHEN** the frontend lint runs
- **THEN** E2E specs and support files are linted and type-checked

### Requirement: CI container image
The repository SHALL provide a container image with the browsers, Node, `oc`, the setup scripts, and the E2E sources. It SHALL run setup and the selected suites in CI from a kubeconfig and a CI user file alone.

#### Scenario: One CI job per suite
- **WHEN** a CI job runs the image with one suite selected
- **THEN** it runs setup, preflight, and that suite, and writes reports to the artifacts directory
