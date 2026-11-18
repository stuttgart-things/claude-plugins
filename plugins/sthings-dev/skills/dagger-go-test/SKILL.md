---
name: dagger-go-test
description: Run Go tests of a stuttgart-things repo through the Dagger Go module (go test is broken in the dev sandbox), including finding a live SSH agent socket. Use when running, debugging or waiting on go test, task test, or dagger call test, or when git push/fetch or dagger fails with "Permission denied (publickey)" or "agent: client error".
---

# Go tests via Dagger

In the dev sandbox `go build` works, but `go test` breaks at the linker.
The tests therefore run in a container through the stuttgart-things Dagger
Go module.

## 1. Find a live SSH agent socket

Dagger and git over SSH need `SSH_AUTH_SOCK`. The sockets in `~/.ssh/agent/`
rotate, and **the newest one is not necessarily alive**. With a dead socket,
Dagger hangs for an hour without printing anything. Test each socket:

```bash
for s in $(ls -t ~/.ssh/agent/s.* 2>/dev/null); do
  SSH_AUTH_SOCK=$s ssh-add -l >/dev/null 2>&1 && { export SSH_AUTH_SOCK=$s; break; }
done
ssh-add -l >/dev/null 2>&1 || echo "no live SSH agent"
```

The shell state does not persist between tool calls, so put the export in
the same command as dagger/git, or pass the path you found explicitly.

## 2. Run the tests

```bash
dagger -m github.com/stuttgart-things/dagger/go call test --src . --test-arg ./...
# with a local checkout of the module:
dagger -m ~/projects/dagger/go call test --src . --test-arg ./...
```

- Give the run a hard upper bound: `timeout 580 dagger …`, or start it in
  the background and wait for the notification. Don't run it in the
  foreground with the default timeout.
- Ignore the `[dagger] Xm elapsed · 1 steps running` lines in logs:
  `grep -v 'elapsed ·'`.
- If there is no output for more than ~10 minutes: stop the run and check
  the socket (step 1). An error with `failed to list SSH agent identities:
  agent: client error: EOF` means the socket is dead.
- The first run pulls images and the Go cache and takes a few minutes.
  After that a run takes about 1–2 minutes.

## 3. Without an SSH agent

If no socket is alive, the user has to start a new session or agent. In
the meantime:

- git push over HTTPS with the gh token, without changing the global config:
  ```bash
  git -c credential.helper= -c credential.helper='!gh auth git-credential' \
    push https://github.com/<owner>/<repo>.git <branch>
  ```
- Create a tag through the API instead of `git push origin <tag>`:
  ```bash
  gh api repos/<owner>/<repo>/git/refs -f ref=refs/tags/<tag> -f sha=$(git rev-parse <tag>)
  ```
- Dagger needs the agent even to load the Go module, so
  wait for it. CI (`Go Lint, Test & Build`) runs the same tests on the PR.

## Lint

`golangci-lint` run locally fails on newer Go versions with
`export data version … is greater than maximum supported version`. In that
case the CI job `validate / Lint-Go` is the check. Common findings in new
code: errcheck on `fmt.Fprint*` (write `_, _ = fmt.Fprintf(...)`, as in
claims), and staticcheck QF1001/QF1008/SA4031.
