# Tasks

## 1. Pinned tools

- [ ] 1.1 Add `Makefile.tools` (D1) with `shellcheck` and `shfmt` installed as versioned binaries plus symlinks, and the `tools` and `clean-tools` targets, and include it from the root `Makefile`. Verify:
  - `make clean-tools && make tools` installs both tools, and `_output/tools/bin/shellcheck --version` shows the pinned version.
  - Changing `SHFMT_VERSION` and running `make tools` installs the new file.
  - `make tools` succeeds in a `ubi9/nodejs-22` container with no Go.
  - Nothing is written outside `_output/`.

## 2. Linters

- [ ] 2.1 Add shell lint (D2): `hack/env/.shellcheckrc`, the `.editorconfig` entries, and `make lint-shell`. Verify:
  - With no scripts present, it exits 0 with "nothing to check".
  - A stub script passes.
  - Scripts with an unquoted `$var` or wrong indentation fail, naming the file and rule.
  - `hack/install-coo.sh` is not checked.
  - It works from a clean checkout, installing the tools first.
- [ ] 2.2 Add Markdown lint (D3): the pinned `markdownlint-cli2` devDependency, the `lint:md` script, `.markdownlint-cli2.yaml`, and `make lint-markdown`. Verify:
  - With no files present, it exits 0.
  - A stub README passes.
  - A heading-level jump and a code fence without a language each fail, naming the file and rule.
  - The root `README.md` is not checked.

## 3. CI and docs

- [ ] 3.1 Update `Dockerfile.test` (D4). Verify: `podman build -f Dockerfile.test .` succeeds, and running it fails when a broken script is added under `hack/env/`. Check how the Prow lint job invokes the image, and apply the `lint-frontend` fallback if it overrides the entrypoint.
- [ ] 3.2 Update `AGENTS.md`: add `make tools` to the setup steps, and add `make lint-shell` and `make lint-markdown` to the pre-PR checks. Verify: `make lint-frontend` and `make test-translations` still pass.

## Workflow follow-up

- Move `golangci-lint` into `Makefile.tools`.
- Extend the linters to the existing `hack/*.sh`, `scripts/*.sh`, and docs.
