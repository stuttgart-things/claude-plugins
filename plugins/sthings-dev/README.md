# sthings-dev

Building and releasing stuttgart-things Go services.

| Skill | Use |
| --- | --- |
| `go-service-bootstrap` | new Go service/CLI like claims/pvectl: files from the reference repo, workflows, release gate, first release v0.1.0 |
| `semver-start-v0` | semantic-release starts at v0.1.0 instead of v1.0.0 (`v0.0.0` tag on the first commit); also covers a repo released as v1.0.0 by mistake |
| `dagger-go-test` | Go tests through the Dagger Go module, finding a live SSH agent socket, HTTPS/API fallback without an agent |

```text
/plugin install sthings-dev@stuttgart-things
```
