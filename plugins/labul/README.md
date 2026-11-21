# labul

LabUL Proxmox operations with [pvectl](https://github.com/stuttgart-things/pvectl).

| Skill | Use |
| --- | --- |
| `labul-template-round` | template round: Packer build, `pvectl template promote` (ul-pve11 + FS9100 copy), repin PRs (proxmoxvm chart, flux, flux-infra tag, backstage `vm_clone_id`) |
| `pvectl` | pvectl in programmatic mode (`-o json`, exit codes), safety rules: never print descriptions, writes only after an OK |

Requires `pvectl` v0.4.0 or later, `sops` and the credential file
`~/.secrets.d/labul/proxmox.env`.

Once per machine, trust the Proxmox certificate yourself; the skills use no
`--insecure` and never pin a certificate on their own:

```bash
sops exec-env ~/.secrets.d/labul/proxmox.env 'pvectl tls trust'
```

Compare the SHA-256 it shows with the web UI (node > System > Certificates)
before you confirm. Alternatively set `PVE_CA_CERT` to the cluster CA
(`/etc/pve/pve-root-ca.pem`). If the certificate is `untrusted`, the skills ask
you to run the command. If it `CHANGED`, they stop and report it. See
[pvectl: TLS without `--insecure`](https://github.com/stuttgart-things/pvectl#tls-without---insecure).

```text
/plugin install labul@stuttgart-things
```
