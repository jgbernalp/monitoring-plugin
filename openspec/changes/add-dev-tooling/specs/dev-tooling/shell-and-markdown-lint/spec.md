# Spec Delta

## Purpose

Catches defects and formatting drift in the bash scripts and Markdown documentation of the E2E environment and test tooling before they merge, without touching existing files.

## ADDED Requirements

### Requirement: Shell scripts are statically analyzed and format-checked
Every bash script under the environment-setup and E2E test directories SHALL pass a static shell analyzer and a shell formatting check, run through one Make target.

#### Scenario: Shell defect
- **WHEN** a script in those directories expands a variable without quotes where the analyzer flags it
- **THEN** the shell lint target exits non-zero and names the file, line, and rule

#### Scenario: Unformatted script
- **WHEN** a script in those directories does not match the configured formatting
- **THEN** the shell lint target exits non-zero and prints the diff to apply

#### Scenario: Sourced libraries
- **WHEN** a script sources a shared library from the same directory tree
- **THEN** the analyzer follows the sourced file instead of reporting it as missing

### Requirement: Markdown documents are linted
Every Markdown document under the environment-setup and E2E test directories SHALL pass a Markdown linter whose style is compatible with the repository's existing formatter, run through one Make target.

#### Scenario: Markdown defect
- **WHEN** a README in those directories skips a heading level or has a fenced code block without a language
- **THEN** the Markdown lint target exits non-zero and names the file, line, and rule

### Requirement: Lint scope is limited to the new tooling paths
The shell and Markdown linters SHALL check only the environment-setup and E2E test directories. Files elsewhere SHALL NOT be checked.

#### Scenario: Existing files untouched
- **WHEN** the lint targets run on the repository
- **THEN** existing scripts such as `hack/install-coo.sh` and existing docs such as `README.md` are not checked

#### Scenario: Paths not created yet
- **WHEN** the lint targets run before any file exists in those directories
- **THEN** they exit zero, reporting that there was nothing to check

### Requirement: Lint is enforced in CI
The CI lint job SHALL run the shell and Markdown lint targets alongside the existing frontend lint.

#### Scenario: Failing file in a pull request
- **WHEN** a pull request adds a script or README in those directories that fails either linter
- **THEN** the CI lint job fails
