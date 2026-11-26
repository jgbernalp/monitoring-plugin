# Proposal

## Why

The Playwright E2E migration (`migrate-e2e-to-playwright`) adds bash setup scripts and new READMEs. Those need pinned, reproducible lint tools on every machine and in CI. Today the repo has no shell or Markdown linting, and non-npm tools are either installed by hand or, in the case of `golangci-lint`, through a one-off Makefile target. Landing this tooling first, as a small change with no cluster involved, keeps the E2E change focused and easier to review.

## What Changes

- **`make tools`**: a new `Makefile.tools`, included from the root `Makefile`, modeled on perses-operator's `Makefile.tools`.
  - It is the single place for pinned versions of non-npm binaries.
  - Tools install into the already git-ignored `_output/tools/bin`, under versioned file names (for example `shellcheck-v0.11.0`) with a plain-name symlink, so bumping a version reinstalls automatically.
  - It provides `make tools`, `make clean-tools`, and one target per tool.
  - It starts with `shellcheck` and `shfmt`.
- **Shell linting**: `make lint-shell` runs `shellcheck` and `shfmt -d` over `hack/env/**/*.sh` and `web/e2e/**/*.sh`. Both tools install on first use.
- **Markdown linting**: `make lint-markdown` runs `markdownlint-cli2` (an npm devDependency in `web/`) over `hack/env/**/*.md` and `web/e2e/**/*.md`, using a Prettier-compatible style.
- **CI**: the existing lint image (`Dockerfile.test`) installs the shell tools through `Makefile.tools` and runs the new lint targets with `lint-frontend`.
- **Scope**: only the paths the E2E change creates are linted, so this change reformats nothing that already exists. If those paths don't exist yet, the lint targets pass with nothing to check.

## Capabilities

### New Capabilities

- `dev-tooling/tool-installation`: Pinned, project-local installation of the non-npm binaries that development and CI need, through a single command.
- `dev-tooling/shell-and-markdown-lint`: Static analysis and formatting checks for bash scripts, and structural linting for Markdown docs, in the paths the E2E tooling owns.

### Modified Capabilities

<!-- None: there are no existing specs in openspec/specs/. -->

## Impact

- **New files**: `Makefile.tools`, `.markdownlint-cli2.yaml`, `hack/env/.shellcheckrc`, and `.editorconfig` entries for shell formatting.
- **Modified files**:
  - `Makefile`: `include Makefile.tools`, plus new `lint-shell` and `lint-markdown` targets.
  - `web/package.json`: the `markdownlint-cli2` devDependency and a `lint:md` script.
  - `Dockerfile.test`: install the shell tools and run the new targets.
  - `AGENTS.md`: `make tools` as a setup step, and the new lint commands in the pre-PR checklist.
- **Unchanged**: `golangci-lint` stays in the root `Makefile` (moving it is a follow-up), and so do the existing lint targets and existing scripts and docs.
- **Follow-ups**:
  - Move `golangci-lint` into `Makefile.tools`.
  - Extend the linters to the existing `hack/*.sh`, `scripts/*.sh`, and repository docs.
