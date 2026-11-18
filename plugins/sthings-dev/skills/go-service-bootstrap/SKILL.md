---
name: go-service-bootstrap
description: Create a new stuttgart-things Go service or CLI repo "like claims" - layout, Taskfile, workflows from github-workflow-templates, release gate, goreleaser/ko, renovate, catalog-info - and get it to a first release v0.1.0. Use when the user wants a new Go repo/service/CLI in stuttgart-things, or asks to set one up like claims or pvectl.
---

# New Go service like claims/pvectl

The reference is the **newest** of these repos (currently
`stuttgart-things/pvectl`, before that `claims`). Copy its files and
replace the name. Don't write them from scratch: the workflows depend on
details (the release gate, the module name, the ldflags paths).

## 0. Decide with the user

- Repo name, public/private (stuttgart-things repos are mostly public:
  **no secrets, no lab endpoints, no Vault paths** in code or docs).
- CLI only, or also a library (`pkg/`) and a container image (ko).
- Credentials only from env vars, e.g. through
  `sops exec-env ~/.secrets.d/<lab>/<x>.env '<tool> …'`.

## 1. Create the repo and the first commit

```bash
gh repo create stuttgart-things/<name> --public --description "<one sentence>"
```

The first commit goes straight to `main` (a skeleton: LICENSE, README stub,
.gitignore). Everything else goes through a PR, which the user merges.

**Right after that, seed the version tag** (skill `semver-start-v0`):
`v0.0.0` on the first commit, so that the first release is v0.1.0 and not
v1.0.0.

## 2. Files from the reference repo

| File | Replace |
| --- | --- |
| `main.go`, `cmd/root.go`, `cmd/version.go`, `internal/buildinfo/` | module path |
| `go.mod` | `module github.com/stuttgart-things/<name>`, current Go version |
| `Taskfile.yaml` | `PROJECT`, `MODULE`; the `test` task runs Dagger (skill `dagger-go-test`) |
| `.goreleaser.yaml` | `id`, `binary`, the three `-X …/cmd.{version,commit,buildDate}`, `name` |
| `.ko.yaml` | `id` |
| `.releaserc.json` | unchanged (`tagFormat: v${version}`) |
| `renovate.json`, `.pre-commit-config.yaml` | unchanged |
| `.gitignore` | `/<name>` (the binary), `/bin/` |
| `catalog-info.yaml` | `name`, `description`, `project-slug`, `tags`; `type: tool` or `service`, `owner: group:default/platform-team` |
| `.github/workflows/golang-validation.yaml` | `module-name: <name>` |
| `.github/workflows/lint-repo.yaml` | `output-file: "/tmp/<name>.txt"` |
| `.github/workflows/build-scan-image.yaml` | unchanged (`call-ko-build.yaml`) |
| `.github/workflows/pr-title.yaml` | unchanged: the PR title must be a Conventional Commit, because the squash merge uses it as the release commit |
| `.github/workflows/release.yaml` | unchanged: gate (both workflows green for main's head), then `call-go-release.yaml` |

Afterwards, `grep -rn '<old name>' .` must find nothing.

## 3. Code conventions

- `cmd/` (cobra) as a thin surface over `pkg/` (the library) or `internal/`.
- Two modes over the same logic: interactive (huh/bubbletea/lipgloss,
  styles as in claims) **only** with a TTY, no `-o json` and no
  `--non-interactive`. Otherwise programmatic: every input has a flag, a
  missing input is an error naming the flag, `-o json` puts exactly one
  JSON document on stdout and logs on stderr, and errors are JSON as well.
- Defined exit codes, documented in the README. Writes need `--dry-run`
  and `--yes`.
- Lint-proof from the start: `_, _ = fmt.Fprintf(w, …)` for writes whose
  error you ignore; no `if x == nil` on slices from `make`.
- Tests: a fake API in `pkg/<x>/<x>test`, e2e tests of the CLI in
  `cmd/cmd_test.go`.

## 4. First PR and release

1. Branch `feat/<name>-v1`, PR title `feat: <name> …` (not `feat!:`).
2. Tests via Dagger (skill `dagger-go-test`). In CI, `validate / Lint-Go`
   is the binding check.
3. Wait for the checks: `gh pr checks <n> -R stuttgart-things/<name> --watch`.
4. Give the user the merge command:
   `gh pr merge <n> -R stuttgart-things/<name> --squash --delete-branch`
5. After the merge, the Release workflow runs twice (once per workflow it
   waits on). The first run is gated and does nothing; the second one
   releases. Check the result:
   `gh release list -R stuttgart-things/<name> -L 1` should show **v0.1.0**,
   with binaries for linux/darwin/windows × amd64/arm64.
6. Smoke test with the release binary (`gh release download … -p
   '<name>_*_linux_amd64.tar.gz'`, then `<name> version`).
