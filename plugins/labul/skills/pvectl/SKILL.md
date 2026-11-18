---
name: pvectl
description: Use pvectl (stuttgart-things/pvectl) to query and change LabUL Proxmox - VMs, templates, configs, tasks, migrate, HA, template promote - safely from an agent. Use whenever a task touches LabUL Proxmox VMs or templates, Proxmox tasks/UPIDs, or mentions pvectl, pve.py or qm-style operations on LabUL.
---

# pvectl for agents

pvectl replaces `hack/labul/pve.py`. As an agent, **always run it in
programmatic mode**: `-o json` (or `--non-interactive`), so it never waits
at a prompt.

## Rules (not negotiable)

- **Never print a VM description (notes).** In LabUL templates they contain
  plaintext passwords. pvectl drops them itself. Don't get them some other
  way either (curl against the API, `qm config`, the go-proxmox debug log).
- **Writes** (`migrate`, `template promote`, `start`, `ha-recover`) against
  the real environment **only after the user's explicit OK** for that exact
  action. First `--dry-run`, show the plan, then `--yes`.
- **Reads** (`vms`, `config`, `tasks`, `ha`, `repin`) may run without asking.
- Never touch templates the user marks as pinned (as of 2026-11: 211, 266).

## Call

Credentials only through sops, never in the command or the environment of
other tools:

```bash
E=~/.secrets.d/labul/proxmox.env
sops exec-env $E 'pvectl --insecure vms ubuntu26 -o json'
```

`--insecure` because the lab CA is not in the system store (alternative:
`PVE_CA_CERT`). Install: `gh release download -R stuttgart-things/pvectl -p
'pvectl_*_linux_amd64.tar.gz' -O - | tar xz pvectl`.

## Commands

| Command | Writes | JSON |
| --- | --- | --- |
| `vms [filter] [--templates\|--all]` | no | array; with a filter also IPs and recent tasks |
| `config <vmid>` | no | config (without description), disks, checks (cpu host, agent, cloud-init) |
| `tasks [--node N --vmid ID --type T --errors --since 96h]` | no | array |
| `tasks --follow <UPID> [--timeout 2h]` | no | final status + log; log lines on stderr |
| `ha [<node>] <vmid>` | no | HA state, tasks, log |
| `repin --os <os> --template N [--copy M] [--repo <checkout>]` | no | the changes in stuttgart-things |
| `migrate <node> <vmid> <target> [--targetstorage S]` | yes | |
| `template promote <vmid> --to ul-pve11 --fs9100-copy [--repo <checkout>] [--target-storage S]` | yes | `plan` with steps, new VMID, repins |
| `start`, `ha-recover` | yes | |

The details are in `pvectl <cmd> --help` (examples for both modes).

## Exit codes

| Code | Meaning | Agent reaction |
| --- | --- | --- |
| 0 | ok, including "nothing to do" | |
| 1 | API/network/TLS | check `--insecure` and the VPN, then report |
| 2 | usage: missing flag/env, write without `--yes` | read the `flag` field of the JSON error and add the flag. Don't add `--yes` without an OK |
| 3 | not found (also `vms` without a match: `[]`) | check the filter/VMID |
| 4 | config check failed / promote source is not a template | report the checks; `--skip-checks` only on the user's instruction |
| 5 | Proxmox task failed | `tasks --follow <UPID>` for the log, report |
| 130 | aborted | the task may keep running: `tasks --follow` |

Errors in JSON: `{"error": "...", "code": 2, "kind": "usage", "flag": "--vmid"}`.
