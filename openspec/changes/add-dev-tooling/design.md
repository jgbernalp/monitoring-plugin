# Design

## Context

See proposal.md. These current-state facts shape the design:
- The root `Makefile` already installs `golangci-lint` into `_output/tools/bin`, and `_output/` is git-ignored.
- `Dockerfile.test` (`ubi9/nodejs-22`, with no Go toolchain) is the CI lint image. Its entrypoint is `make lint-frontend`, and it copies only `web/` and `scripts/`.
- The repo uses Prettier 3 (`web/.prettierrc.yml`). Existing bash scripts in `hack/` and `scripts/` use 4-space indentation.
- Project rule: Node tools run through npm scripts, never through `npx`.

## Goals / Non-Goals

**Goals:** pinned and reproducible lint tools that install on first use, the same versions locally and in CI, and minimal Make code.

**Non-Goals:** moving `golangci-lint`, linting existing files, and tools that npm already manages.

## Decisions

### D1. `Makefile.tools` with versioned binaries
`Makefile.tools` is included from the root `Makefile`.

```make
LOCALBIN ?= $(CURDIR)/_output/tools/bin
SHELLCHECK_VERSION ?= v0.11.0      # final versions chosen at implementation, each released ≥2 weeks earlier
SHFMT_VERSION      ?= v3.12.0
SHELLCHECK = $(LOCALBIN)/shellcheck-$(SHELLCHECK_VERSION)
SHFMT      = $(LOCALBIN)/shfmt-$(SHFMT_VERSION)

$(SHELLCHECK): ; <download release tarball for $(OS)/$(ARCH)>, extract, then ln -sf to $(LOCALBIN)/shellcheck
$(SHFMT):      ; <download release binary>, chmod +x, then ln -sf to $(LOCALBIN)/shfmt
tools: $(SHELLCHECK) $(SHFMT)
clean-tools: ; rm -rf $(LOCALBIN)
```

- **Versions in file names.** Because the file name includes the version, Make's own file-based rules reinstall on a bump. No version parsing or `validate-tools` target is needed.
- **Platform.** OS and architecture come from `uname -s` and `uname -m`, not `go env`, because the CI image has no Go. Architecture names are mapped per tool: shellcheck uses `x86_64` and `aarch64`, while shfmt uses `amd64` and `arm64`.
- **Downloads.** `curl -fsSL --retry 5` into a temp file, then move into place, so an interrupted download never leaves a partial binary.

*Alternatives considered:*
- *perses-style "skip if the file exists".* Rejected because version bumps would be silently ignored.
- *A container-image fallback.* Rejected because it needs a container runtime and keeps versions in a second place.
- *The npm `shellcheck` wrapper.* Rejected because it downloads at install time, which breaks hermetic Konflux builds.

### D2. Shell lint
- `hack/env/.shellcheckrc` sets `shell=bash` and `external-sources=true`, so sourced libraries are followed.
- `.editorconfig` holds the `shfmt` settings for `[hack/env/**.sh]` and `[web/e2e/**.sh]`: 4-space indent, switch-case indent, and binary operators at line start.
- `lint-shell: $(SHELLCHECK) $(SHFMT)` collects files with `find hack/env web/e2e -name '*.sh' 2>/dev/null`. If there are none, it prints "nothing to check" and exits 0. Otherwise it runs `$(SHELLCHECK) --severity=style` and `$(SHFMT) -d`.

### D3. Markdown lint
- `markdownlint-cli2` is pinned as a `web/` devDependency, with `"lint:md": "markdownlint-cli2"`.
- The root `.markdownlint-cli2.yaml` sets `globs: ["hack/env/**/*.md", "web/e2e/**/*.md"]`, extends `markdownlint/style/prettier`, and disables `MD013` (line length).
- `lint-markdown` runs `cd web && npm run lint:md -- --config ../.markdownlint-cli2.yaml`. If no files match, it exits 0.

*Alternative considered:* Prettier alone for Markdown. Rejected because it formats but misses structural issues.

### D4. CI wiring
`Dockerfile.test` additionally copies `Makefile.tools`, runs `make tools`, copies `hack/` and `.markdownlint-cli2.yaml`, and sets its entrypoint to `make lint-frontend lint-shell lint-markdown`. If the Prow job overrides the entrypoint instead of using it, `lint-frontend` calls the two new targets, so enforcement doesn't depend on a release-repo change.

## Risks / Trade-offs

- **[Risk] Release asset names change.** → Mitigation: one explicit download rule per tool. The CI image build fails loudly.
- **[Risk] The Prow job overrides the image entrypoint.** → Mitigation: the D4 fallback through `lint-frontend`.
- **[Trade-off] Existing scripts stay unlinted.** → This is a deliberate follow-up, to avoid unrelated churn.

## Migration Plan

Land before `migrate-e2e-to-playwright`. Rollback is to delete `Makefile.tools`, the config files, and the targets, and to revert `Dockerfile.test`.
