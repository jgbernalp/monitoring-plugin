# Spec Delta

## Purpose

Gives contributors and CI one command that installs pinned versions of the non-npm binaries the repository needs, into a project-local directory, so versions don't drift between machines and CI.

## ADDED Requirements

### Requirement: Single source of pinned tool versions
Every non-npm binary the repository installs for its lint and tooling workflows SHALL have its version declared in exactly one place. Binaries that developers bring themselves, such as `oc`, are out of scope. CI images SHALL install tools from those declarations instead of repeating versions.

#### Scenario: CI image uses the same versions
- **WHEN** the CI lint image is built
- **THEN** it installs the shell tools through the shared declarations, and no tool version appears in the Dockerfile

### Requirement: Project-local installation with one command
One command SHALL install every declared tool into a git-ignored directory inside the repository, and another command SHALL remove them. Installing SHALL NOT modify anything outside the repository.

#### Scenario: Fresh machine
- **WHEN** a contributor with only make, bash, curl, and tar runs the install command
- **THEN** every declared tool is installed at its pinned version in the project tool directory

#### Scenario: Clean
- **WHEN** a contributor runs the clean command
- **THEN** the project tool directory is removed

### Requirement: On-demand installation and version bumps
Make targets that use a tool SHALL install it on first use. Changing a tool's pinned version SHALL cause the next use to install the new version, without a manual clean.

#### Scenario: First use
- **WHEN** a contributor runs a lint target before installing any tools
- **THEN** the tools it needs are installed first, and the lint runs with the pinned versions

#### Scenario: Version bump
- **WHEN** a tool's pinned version changes and a target that uses it runs
- **THEN** the new version is installed and used

### Requirement: Platform support
Tool installation SHALL work on Linux and macOS, on both amd64 and arm64, without requiring a Go toolchain.

#### Scenario: Node-based CI image without Go
- **WHEN** the install command runs in the Node-based CI lint image
- **THEN** it succeeds without Go installed
