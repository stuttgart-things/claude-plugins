---
name: portal-vm-status
description: Get the current state of stuttgart-things VMs (Proxmox, vSphere, Harvester) live from the hypervisor through the developer portal's MCP server - running or stopped, uptime, IP addresses, host/node, CPU and memory. Use when the user asks whether a VM is running, which VMs are stopped, where a VM runs, its IP, or for an overview of a lab's VMs, and an sthings-portal-* MCP server is connected.
---

# VM status through the portal

Two sources, do not mix them up:

- The **catalog** (`catalog.query-catalog-entities`): every VM as a
  `Resource` of type `vm`, with `vm.stuttgart-things.com/status`, `cpus`,
  `memory-mib`, `os`. A snapshot of the provider's last run (every few
  minutes), good for listing and filtering.
- **`vm-info.get-live-vm-info`**: asks the hypervisor now (cached ~30 s).
  Use it when the user wants the current state, and say that you did.

## Find the VMs

The provider of a VM is its `backstage.io/managed-by-location`, e.g.
`sthings-vms:proxmox-labul` or `sthings-vms:vsphere-labda`.

```json
{"query": {"$all": [
  {"kind": "Resource", "spec.type": "vm"},
  {"metadata.annotations.backstage.io/managed-by-location": "sthings-vms:proxmox-labul"},
  {"$not": {"metadata.annotations.vm.stuttgart-things.com/status": "running"}}
]}, "fields": ["metadata.name", "metadata.annotations"]}
```

Leave out the provider line for all VMs, the `$not` line for all states. List
the providers first if the user names a lab you cannot map:
`{"query": {"kind": "Resource", "spec.type": "vm"}, "fields": ["metadata.annotations.backstage.io/managed-by-location"]}`.

## Live state

`vm-info.get-live-vm-info` with `entityRef` (`resource:default/<name>` or just
the name). Output: `found`, and in `info`: `status`, `uptimeSeconds`,
`ipAddresses`, `host`, `cpu.count`, `cpu.usagePercent`, `memory.totalMiB`,
`memory.usedMiB`, `fetchedAt`.

- One call per VM. For "which VMs are stopped" first narrow with the catalog,
  then confirm the candidates live; for a full overview, ask live for all VMs
  of the provider and mention the number of calls if it is large.
- `found: false` with "not managed by a live VM source": a hand-written VM,
  there is no live state. "is not configured here": this portal instance has
  no credentials for that provider, ask the other `sthings-portal-<lab>`.
- An error (e.g. "Request failed with 500") is the hypervisor refusing or
  failing; report it, fall back to the catalog status and say it may be stale.
- Memory "used" is host-side on Proxmox (close to total for a busy VM), guest
  active memory on vSphere. Do not call a VM out of memory on that basis.

## Changing a VM

The portal is read-only: no start, stop or migrate. For LabUL Proxmox use the
`labul` plugin's `pvectl` skill (writes only after the user's explicit OK).
Do not try other ways to the hypervisor.
