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
- **Live** from the hypervisor (cached ~30 s): `vm-info.list-live-vms` for
  all VMs of one provider in **one call**, `vm-info.get-live-vm-info` for a
  single VM. Use them when the user wants the current state, and say that you
  did.

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

Use the catalog to list and filter without asking the hypervisor. Leave out
the provider line for all VMs, the `$not` line for all states. List the
providers first if the user names a lab you cannot map:
`{"query": {"kind": "Resource", "spec.type": "vm"}, "fields": ["metadata.annotations.backstage.io/managed-by-location"]}`.

## Live state

**Several VMs or an overview:** `vm-info.list-live-vms` with `provider` (e.g.
`proxmox-labul`) and optionally `status` (`stopped`, `running`, ...). One
call, the hypervisor is asked for every VM of the provider; the tool's
description lists the providers this portal instance can ask. Output:
`total` (VMs of the provider in the catalog) and `vms[]` with `entityRef`,
`found` and `info` or `message`. "Which VMs on LabUL are stopped" is
`{"provider": "proxmox-labul", "status": "stopped"}`.

**One VM:** `vm-info.get-live-vm-info` with `entityRef`
(`resource:default/<name>` or just the name).

`info` has `status`, `uptimeSeconds`, `ipAddresses`, `host`, `cpu.count`,
`cpu.usagePercent`, `memory.totalMiB`, `memory.usedMiB`, `fetchedAt`.

- `found: false` always comes with a `message`, also when filtering by
  status: a VM that could not be asked is listed, so an empty result really
  means "none". Report those VMs separately instead of dropping them.
  - "not managed by a live VM source": a hand-written VM, there is no live
    state.
  - "is not configured here" (or the tool rejects the provider): this portal
    instance has no credentials for it, ask the other `sthings-portal-<lab>`.
  - Anything else is the hypervisor refusing or failing (e.g. "vSphere Login
    failed: Permission to perform this operation was denied"); report it,
    fall back to the catalog status and say it may be stale.
- Older portal versions only have `get-live-vm-info` and answer hypervisor
  errors with "Request failed with 500"; then narrow with the catalog first
  and ask per VM.
- Memory "used" is host-side on Proxmox (close to total for a busy VM), guest
  active memory on vSphere. Do not call a VM out of memory on that basis.

## Changing a VM

The portal is read-only: no start, stop or migrate. For LabUL Proxmox use the
`labul` plugin's `pvectl` skill (writes only after the user's explicit OK).
Do not try other ways to the hypervisor.
