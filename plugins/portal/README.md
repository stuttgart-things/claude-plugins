# portal

Skills for the stuttgart-things developer portal (Backstage) through its MCP
server: who owns what, where the repo and docs are, which template to use,
and the live state of VMs.

| Skill | Use |
| --- | --- |
| `portal-lookup` | owners, systems, repos, docs of components, APIs, resources, templates; catalog query patterns that work |
| `portal-vm-status` | VMs from the catalog, their current state live from Proxmox/vSphere/Harvester (`vm-info.get-live-vm-info`) |
| `portal-templates` | the right Scaffolder template or offering for an order, with the form link and the required inputs |

The portal is **read-only** for AI clients: the skills look things up and
link to forms; changing VMs goes through other plugins (e.g. `labul`'s
`pvectl`).

## The MCP server

The plugin brings the skills, not the connection: portal URLs and tokens do
not belong in this public repo. Connect each portal instance once, for all
projects, with the token from your sops file (platform-team only):

```bash
sops exec-env ~/.secrets.d/<lab>/backstage.env \
  'claude mcp add -s user --transport http sthings-portal-<lab> "$BACKSTAGE_MCP_URL" --header "Authorization: Bearer $BACKSTAGE_MCP_TOKEN"'
claude mcp list | grep sthings-portal
```

The skills expect servers named `sthings-portal-<lab>`. How the tokens are
created and what they can do:
[docs/mcp.md in sthings-backstage](https://github.com/stuttgart-things/sthings-backstage/blob/main/docs/mcp.md).

```text
/plugin install portal@stuttgart-things
```
