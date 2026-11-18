# labul

LabUL Proxmox operations with [pvectl](https://github.com/stuttgart-things/pvectl).

| Skill | Use |
| --- | --- |
| `labul-template-round` | template round: Packer build, `pvectl template promote` (ul-pve11 + FS9100 copy), repin PRs (proxmoxvm chart, flux, flux-infra tag, backstage `vm_clone_id`) |
| `pvectl` | pvectl in programmatic mode (`-o json`, exit codes), safety rules: never print descriptions, writes only after an OK |

Requires `pvectl`, `sops` and the credential file `~/.secrets.d/labul/proxmox.env`.

```text
/plugin install labul@stuttgart-things
```
