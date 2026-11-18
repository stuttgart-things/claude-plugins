# claude-plugins

Claude Code plugin marketplace of stuttgart-things: skills from the
everyday platform work, versioned and installed with `/plugin`.

## Install

```text
/plugin marketplace add stuttgart-things/claude-plugins
/plugin install sthings-dev@stuttgart-things
/plugin install labul@stuttgart-things
```

Or from the shell: `claude plugin marketplace add stuttgart-things/claude-plugins`,
then `claude plugin install sthings-dev@stuttgart-things`.

Updates: `/plugin marketplace update stuttgart-things`, then `/plugin update <plugin>@stuttgart-things`.

### For a whole repo

Put this in the repo's `.claude/settings.json`. Anyone who opens the repo in
Claude Code is offered the marketplace and the plugins:

```json
{
  "extraKnownMarketplaces": {
    "stuttgart-things": {
      "source": { "source": "github", "repo": "stuttgart-things/claude-plugins" }
    }
  },
  "enabledPlugins": {
    "sthings-dev@stuttgart-things": true
  }
}
```

## Plugins

| Plugin | Skills | For |
| --- | --- | --- |
| [`sthings-dev`](plugins/sthings-dev) | `go-service-bootstrap`, `semver-start-v0`, `dagger-go-test` | building and releasing Go services like claims/pvectl |
| [`labul`](plugins/labul) | `labul-template-round`, `pvectl` | LabUL Proxmox: template round, pvectl for agents |

In Claude Code the skills are named `<plugin>:<skill>`, e.g.
`sthings-dev:semver-start-v0`. Claude loads them by their description when
a task fits, or you call them with `/sthings-dev:semver-start-v0`.

## Public repo

This repo is public. Skills describe procedures, not secrets: no
passwords, tokens, lab endpoints/IPs or Vault paths. They name credential
**files** (`~/.secrets.d/…`) that are used through `sops exec-env`.

## Development

```text
plugins/<plugin>/
  .claude-plugin/plugin.json   # name, version, description
  skills/<skill>/SKILL.md      # frontmatter name + description (when to load), then the procedure
  README.md
.claude-plugin/marketplace.json  # lists the plugins
```

- Test locally without installing: `claude --plugin-dir plugins/<plugin>`.
- Validate: `claude plugin validate .` and `claude plugin validate plugins/<plugin>` (also in CI).
- A new skill is a new folder under `skills/`; a new plugin also needs an entry in `marketplace.json`.
- **Release:** bump `version` in `plugin.json` (a new plugin starts at `0.1.0`).
  After the merge, `claude plugin tag plugins/<plugin> --push` tags it as `<plugin>--v<version>`.
  Bump the version with every change to a plugin: installed copies are updated by version.
- PR titles are Conventional Commits.
