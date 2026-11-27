# Spec Delta

## Purpose

Gives developers and CI one central place and one command to prepare a cluster for E2E runs. The command adapts to the environment it finds, prepares only what the selected tests need, can undo what it prepared, and hands the result to the test runner in a single inspectable file, without any TypeScript build.

## ADDED Requirements

### Requirement: Setup runs without a TypeScript toolchain
Environment preparation, validation, and teardown SHALL be bash scripts that depend only on bash, `oc`, `curl`, `openssl`, and standard POSIX utilities. They SHALL run before `npm install` or any compilation step.

#### Scenario: Fresh checkout
- **WHEN** a user on a fresh checkout without `node_modules` runs setup with a valid kubeconfig
- **THEN** setup completes and writes the environment file without invoking node, npm, or tsc

#### Scenario: Missing required tool
- **WHEN** setup runs and `oc` is not on `PATH`
- **THEN** it exits non-zero before touching the cluster and names the missing tool

### Requirement: Steps are small, independent units
Each preparation step SHALL live in its own file in a single steps folder. Each step SHALL provide a read-only check, an idempotent apply, and a teardown. Each step SHALL be runnable on its own for local development. The order of steps SHALL be declared in one place.

#### Scenario: Run one step directly
- **WHEN** a developer runs a single step file
- **THEN** it verifies that the steps it depends on are satisfied, failing with their names if they are not, and then checks and applies only that step

#### Scenario: Check one step
- **WHEN** a developer runs a single step file in check mode
- **THEN** it reports whether the step is satisfied, would apply, or is blocked, without changing the cluster

### Requirement: Plan before apply
The setup command SHALL check every needed step before changing anything, and print a plan marking each step as satisfied, to apply, or blocked with a reason and remedy. If any step is blocked, setup SHALL exit non-zero without applying anything. A plan-only mode SHALL stop after printing. Each applied step SHALL be checked again and report satisfied, so a re-run with the same selection changes nothing.

#### Scenario: Plan only
- **WHEN** setup runs in plan-only mode
- **THEN** it prints the plan and exits without changing the cluster or writing credentials

#### Scenario: Blocked step
- **WHEN** any needed step is blocked
- **THEN** setup prints the full plan with the reason and remedy, and exits non-zero before applying any step

#### Scenario: Second run
- **WHEN** setup runs twice in a row with the same suite selection
- **THEN** the second run marks every step as satisfied and does not restart the plugin, change OAuth, or recreate fixtures

### Requirement: Behavior follows the detected environment
Setup SHALL NOT require the caller to name the environment. It SHALL derive the console URL, whether console auth is disabled, how the plugin is served, whether the cluster monitoring operator is scaled down, and where test users come from, from what it finds. Each of these SHALL be overridable through an environment variable. Setup SHALL be fully drivable through environment variables, and SHALL never wait for input.

#### Scenario: Deployed console
- **WHEN** no console URL is given
- **THEN** setup uses the cluster's console route, and checks the plugin through the console

#### Scenario: Local console
- **WHEN** the console URL is a localhost URL
- **THEN** setup checks the locally served plugin manifest, and detects whether the local console runs with auth disabled

#### Scenario: Local console with auth disabled
- **WHEN** the local console runs with auth disabled
- **THEN** setup prepares no test users or role bindings, and the admin role counts as assigned without a user

#### Scenario: CI job
- **WHEN** setup runs inside a CI job that provides a user file
- **THEN** it takes test users from that file without prompting, and the plan names the file

#### Scenario: Headless invocation
- **WHEN** setup runs with no TTY attached
- **THEN** it completes, or fails with a named reason, without blocking

### Requirement: Preparation is scoped to the selected suites
Each E2E suite SHALL declare which preparation steps and test roles it needs, omitting steps already implied by those. Setup SHALL resolve the selected suites the same way the test runner does, add the steps those depend on, and prepare only those. With no suite selected, setup SHALL prepare what every suite needs. Title or tag filters SHALL NOT affect what setup prepares.

#### Scenario: Suite without custom alerts
- **WHEN** setup runs for only the `metrics` suite
- **THEN** it does not inspect or change user workload monitoring and creates no alert rules

#### Scenario: Dependent step
- **WHEN** a selected suite needs the alert rules
- **THEN** setup also prepares user workload monitoring, which the alert rules depend on

### Requirement: Test users are resolved by source precedence
Setup SHALL take test users from the first available source, in this order:
1. A provided pool, either set explicitly or found in the user files that known CI steps write.
2. A local credentials file from an earlier run against the same cluster.
3. Users created by setup, on clusters whose OAuth configuration can be changed from the cluster itself.

#### Scenario: CI user file
- **WHEN** a CI step has written a user file, either the hosted-cluster htpasswd format or the `idp-htpasswd` runtime format
- **THEN** setup assigns the needed roles from it and does not change OAuth

#### Scenario: Both CI files present
- **WHEN** both known CI user files exist and no pool is set explicitly
- **THEN** setup uses the hosted-cluster htpasswd file, and the plan names it

#### Scenario: Pool too small
- **WHEN** the pool has fewer users than the needed roles
- **THEN** setup assigns the users it has and lists the roles left without a user (their tests are skipped, see the test-harness "Role not assigned" scenario)

#### Scenario: HyperShift without a pool
- **WHEN** no pool is provided, no credentials file matches, and the cluster's control plane is hosted externally
- **THEN** the users step is blocked, and the remedy points to providing a pool or to the CI user steps

### Requirement: Stable role assignment
Setup SHALL give every role a distinct user, assigned in a fixed order regardless of which suites are selected. Only the role bindings SHALL follow the selected suites. The admin role's user SHALL be granted cluster-admin.

#### Scenario: Different selections on the same cluster
- **WHEN** setup runs first for `metrics` only and later for every suite, with the same pool
- **THEN** each user keeps the same role in both runs, and the second run only adds the missing bindings

### Requirement: User pool formats
Setup SHALL accept a pool as `username:password` pairs in a comma-separated variable, in a sourceable file that assigns `USERS`, or in a file with one pair per line. It SHALL detect a file's format from its content.

#### Scenario: One pair per line
- **WHEN** the pool file holds `user1:abc/+=` and `user2:def` on separate lines
- **THEN** setup reads two users and keeps each password intact

#### Scenario: Sourceable file with other exports
- **WHEN** the pool file assigns `USERS` alongside unrelated exports
- **THEN** setup reads only `USERS`

#### Scenario: Malformed entry
- **WHEN** an entry has no `:` or an empty username
- **THEN** the users step is blocked, naming the entry's position without printing any password

### Requirement: Created users stay private to the runner
When setup creates users, it SHALL create one for every role, with predictable usernames per role and random passwords. The passwords SHALL be written only to a git-ignored, owner-only file on the machine running setup, keyed to the cluster. The cluster SHALL hold only password hashes. Whenever no credentials file matches the cluster, setup SHALL generate new passwords and replace the stored hashes.

#### Scenario: First run
- **WHEN** setup creates users on a cluster with mutable OAuth
- **THEN** it adds a test-owned identity provider without removing existing ones, waits until the users can log in, and writes the credentials file

#### Scenario: Re-run
- **WHEN** setup runs again from the same machine against the same cluster
- **THEN** it reuses the credentials file and does not change OAuth

#### Scenario: Credentials file lost
- **WHEN** the test-owned identity provider exists but no credentials file matches the cluster
- **THEN** setup generates new passwords, replaces the hashes, waits until the users can log in, and writes a new credentials file

### Requirement: RBAC bindings for test roles
Setup SHALL bind each needed role to exactly the permissions it defines, and to nothing beyond them. The roles are:
- cluster admin;
- admin of one test namespace;
- monitoring view in two test namespaces, shared by the metrics and dashboards suites.

#### Scenario: Namespace admin role
- **WHEN** RBAC is prepared for the non-admin alerting role
- **THEN** that user is admin of its alerting test namespace and has no cluster-wide monitoring permissions

#### Scenario: Namespace viewer role
- **WHEN** RBAC is prepared for the namespace viewer role
- **THEN** that user can read metrics in exactly two test namespaces and has no access to a third

### Requirement: Fixture alerts are firing before tests start
Setup SHALL create the test namespaces and the alert rules that fire immediately, and SHALL finish only when every fixture alert is firing.

#### Scenario: Alerts ready
- **WHEN** the alert rules step completes
- **THEN** each fixture alert is reported as firing by Alertmanager

#### Scenario: Alerts do not fire in time
- **WHEN** a fixture alert is not firing within the timeout
- **THEN** setup exits non-zero, naming the alert and its namespace

### Requirement: User workload monitoring enablement
When needed, setup SHALL turn on user workload monitoring by creating the cluster monitoring configuration if it is absent, and SHALL do nothing if it is already on. It SHALL block, without changes, when an existing configuration has it off, or when the cluster monitoring operator is scaled to zero and it is not already running.

#### Scenario: No configuration
- **WHEN** no cluster monitoring configuration exists
- **THEN** setup creates a test-owned one that turns user workload monitoring on, and waits until it is running

#### Scenario: Configuration with it off
- **WHEN** an existing configuration does not turn it on
- **THEN** the step is blocked, and the remedy shows the setting to add

#### Scenario: Operator scaled down
- **WHEN** it is not running and the cluster monitoring operator is scaled to zero, as during a DevSpace session
- **THEN** the step is blocked, and the remedy is to end the DevSpace session, run setup, and start the session again. The remedy SHALL NOT suggest scaling the operator up, which would replace the development pod

### Requirement: Custom plugin image
When a plugin image is given, setup SHALL stop the cluster monitoring operator from managing the plugin, deploy that image, and wait until it is ready. It SHALL do nothing if that image is already running. It SHALL block if the operator is scaled to zero, because the plugin is then owned by a development session.

#### Scenario: Deploy an image
- **WHEN** a plugin image is given and differs from the running one
- **THEN** the plugin runs that image when setup finishes

#### Scenario: DevSpace session
- **WHEN** a plugin image is given while the cluster monitoring operator is scaled to zero
- **THEN** the step is blocked, and the remedy is to unset the image or end the session

#### Scenario: Existing unmanaged override
- **WHEN** the operator is already marked unmanaged by an override that setup did not create
- **THEN** setup reuses it, and teardown leaves it in place

### Requirement: Preflight validation
Before tests run, setup SHALL verify what no preparation step already verifies, reporting every failed check:
- the cluster API and console respond;
- the plugin is enabled and serves its manifest.

Fixture readiness is verified by each step's post-apply check, and user logins by the test harness's authentication.

#### Scenario: Plugin enabled but not serving
- **WHEN** the plugin is enabled but its manifest cannot be fetched, for example because a DevSpace pod is still waiting for the frontend build
- **THEN** preflight fails, saying the plugin is enabled but not serving

#### Scenario: Local plugin not running
- **WHEN** the console is local and the local plugin manifest is not served
- **THEN** preflight fails, naming the local process to start

### Requirement: Reversible teardown, full or partial
Teardown SHALL revert only what setup created, using recorded ownership: test namespaces, role bindings, alert rules, identity providers, cluster monitoring configuration, cluster-version overrides, and test silences. It SHALL support removing everything or only selected steps, tearing down any prepared steps that depend on a selected step first. Resources that setup did not create SHALL stay unchanged.

#### Scenario: Full teardown
- **WHEN** teardown runs with no step selected
- **THEN** every prepared step is reverted in reverse order, the local credentials file for that cluster is deleted, and the cluster returns to its pre-setup state

#### Scenario: Partial teardown to switch setups
- **WHEN** teardown runs for user workload monitoring only, while the alert rules that depend on it are prepared
- **THEN** the alert rules are removed first, then user workload monitoring, and users, RBAC, and the test namespaces stay in place

#### Scenario: Pre-existing configuration
- **WHEN** user workload monitoring or the unmanaged override existed before setup
- **THEN** teardown leaves them unchanged

### Requirement: Single generated environment file
Setup SHALL write every value the test runner needs to one sourceable `KEY=value` file, git-ignored and owner-only. This includes which steps are prepared and which roles have users. The test runner SHALL read that file, and exported environment variables SHALL take precedence over it.

#### Scenario: Run tests after setup
- **WHEN** setup has written the file and the user runs the E2E test command
- **THEN** the tests use its console URL, role assignments, and prepared steps without further flags

#### Scenario: Override a value
- **WHEN** the user exports a different console URL than the file holds
- **THEN** the tests use the exported value
