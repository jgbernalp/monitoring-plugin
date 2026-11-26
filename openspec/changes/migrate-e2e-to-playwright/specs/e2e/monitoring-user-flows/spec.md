# Spec Delta

## Purpose

Defines the end-to-end user flows for the core `monitoring-plugin` that depend on a live console and cluster: admin and non-admin alerting, namespace-scoped metrics, legacy dashboard permissions, and console navigation entry points. Regressions in RBAC, backend round-trips, and console integration are caught without re-testing component-level behavior.

## ADDED Requirements

### Requirement: Alerting flow for admins
The suite SHALL verify that an admin can find a fixture alert that fires immediately, silence it, confirm it is silenced, expire the silence, and confirm the alert is active again. All of this runs against the cluster's Alertmanager.

#### Scenario: Silence and expire as admin
- **GIVEN** a fixture alert rule that fires immediately in the admin alerting test namespace
- **WHEN** the admin filters the alerts list by the fixture alert's name and silences it with a test-marked comment
- **THEN** filtering for silenced alerts shows the fixture alert as Silenced
- **AND** after the admin expires that silence, the fixture alert is shown as Firing in the alerts list

### Requirement: Alerting flow for non-admins
The suite SHALL verify that a namespace-admin user sees and can silence only alerts in their namespace, and cannot see alerts or silences belonging to namespaces they have no access to.

#### Scenario: Namespace admin manages own alert
- **GIVEN** a user who is admin of one test namespace, a fixture alert firing in that namespace, a fixture alert firing in a namespace the user cannot access, and a silence named `private-silence` that targets the inaccessible alert
- **WHEN** the user selects their namespace and filters for the inaccessible alert
- **THEN** that alert is not shown
- **AND** the user can silence their fixture alert, see it as Silenced, expire the silence, and see it as Firing again
- **AND** searching the silences list for `private-silence` shows no result

#### Scenario: Group-based namespace admin
- **WHEN** a user's admin access to the namespace comes from group membership instead of a direct binding
- **THEN** this case is out of scope for this change and is recorded as a follow-up

### Requirement: Namespace-scoped metrics access
The suite SHALL verify that a user with metrics access to exactly two namespaces can query metrics in each of them through the Metrics page, and cannot read metrics from a namespace they have no access to.

#### Scenario: Read metrics in a permitted namespace
- **GIVEN** a user allowed to read metrics in two test namespaces
- **WHEN** the user selects the first namespace and runs a query
- **THEN** results are shown, and every series belongs to that namespace

#### Scenario: Switch to the other permitted namespace
- **WHEN** the user switches to the second permitted namespace
- **THEN** the query re-runs and the results change to series from the second namespace

#### Scenario: Forbidden namespace
- **WHEN** the user manually builds a query or URL targeting a namespace they cannot access
- **THEN** no series from that namespace are shown, and the page shows the access error or empty result returned by the backend

### Requirement: Legacy dashboard permissions for non-admins
The suite SHALL verify that a user with monitoring view in one namespace can open namespace-scoped legacy dashboards with data for that namespace, and cannot view dashboard data for namespaces they lack access to.

#### Scenario: Permitted namespace dashboard
- **GIVEN** a user with monitoring view in one test namespace
- **WHEN** the user opens the Dashboards page with that namespace selected and picks a namespace-scoped dashboard
- **THEN** its panels render data for that namespace

#### Scenario: Cluster-only dashboards and forbidden namespaces
- **WHEN** the same user looks for a cluster-scoped dashboard, or opens a dashboard URL for a namespace they cannot access
- **THEN** cluster-only dashboards are not offered, and no data from the forbidden namespace is rendered

### Requirement: Console navigation entry points
The suite SHALL verify, as the admin, that each console entry point into the plugin opens the correct plugin page with the correct context. These entry points are tagged `@smoke`.

#### Scenario: Overview to Alerting
- **WHEN** the admin follows the alerts link on the cluster Overview page
- **THEN** the Alerting page opens

#### Scenario: Project to Alerting
- **WHEN** the admin follows the alerts link from a project's details page
- **THEN** the Alerting page opens scoped to that project

#### Scenario: Topology Observe panel to dashboard
- **WHEN** the admin opens Topology for `openshift-monitoring`, selects a workload, and follows the dashboard link in the side panel's Observe tab
- **THEN** the Dashboards page opens with `openshift-monitoring` selected and the dashboard that the link names

#### Scenario: Topology Observe panel to Metrics
- **WHEN** the admin follows an Inspect link in the side panel's Observe tab
- **THEN** the Metrics page opens with that panel's query and rendered results

#### Scenario: Pod Metrics tab to Metrics page
- **WHEN** the admin opens a pod's Metrics tab and follows a chart's link to the Metrics page
- **THEN** the Metrics page opens with the chart's query, scoped to the pod's namespace

#### Scenario: Notification drawer to alert
- **WHEN** the admin opens the notification drawer and selects a firing alert
- **THEN** that alert's details page opens

#### Scenario: Alert details to Metrics
- **WHEN** the admin uses Inspect on an alert's details page
- **THEN** the Metrics page opens with the alert expression and rendered results

#### Scenario: Alerting rule details to Metrics
- **WHEN** the admin uses Inspect on an alerting rule's details page
- **THEN** the Metrics page opens with the rule expression and rendered results

### Requirement: Suite runtime budget
On a prepared cluster with two parallel workers, the `@smoke` subset SHALL complete in under 5 minutes and the full monitoring suite in under 15 minutes. Environment setup is excluded from both budgets.

#### Scenario: Full run duration
- **WHEN** the full monitoring suite runs against a prepared `remote` or `ci` environment
- **THEN** the reported wall-clock duration is under 15 minutes
