---
name: labul-template-round
description: Run a LabUL Proxmox template round end to end - Packer build on ul-pve01, pvectl template promote (migrate to ul-pve11, FS9100 copy), then the repin PRs (proxmoxvm chart -> flux release -> flux-infra tag; backstage create-terraform-vm vm_clone_id). Use when a new LabUL template (ubuntu26, rocky10, ...) was built or should be rolled out, or when the user mentions templateVmId, vm_clone_id, FS9100 copy or a template round.
---

# LabUL template round

```text
Packer (ul-pve01) -> pvectl promote: checks -> migrate ul-pve11 -> FS9100 copy -> repin PRs -> chart -> flux -> flux-infra tag
```

Also load the `pvectl` skill. Its rules apply: **never print a description,
writes only after an OK.**

## Hard rules

- Proxmox writes (promote, migrate, clone, template, delete) **only after
  the user's explicit OK** for that exact action. Show the dry-run first.
- Never touch the old, still-pinned templates (as of 2026-11: **211, 266**).
  Promote only touches its source and creates the copy.
- Work in your own worktree; **never `git switch`** in
  `~/projects/stuttgart-things`. The user merges every PR themselves: give
  them the `gh pr merge` commands, in order.

## 1. Build

The Packer build runs through Crossplane on LabDA machinery (a `PackerBuild`
XR with `gitWorkspaceSubdirectory: packer/builds/<os>-labul-proxmox-base-os`).
See the README of that build. The template ends up on **ul-pve01**
(`DD-sthings`, pool `stuttgart-things`). Note its VMID from the build log or
from `pvectl vms <os> --templates -o json`.

## 2. Check and promote

```bash
E=~/.secrets.d/labul/proxmox.env
sops exec-env $E 'pvectl --insecure config <vmid> -o json'          # cpu host, agent; a missing cloud-init drive is only a warning on templates
sops exec-env $E 'pvectl --insecure template promote <vmid> --to ul-pve11 --fs9100-copy --repo <worktree> --dry-run -o json'
```

Show the user the plan (steps, the copy's new VMID and name `<name>-fs9100`,
the repins). **After the OK**:

```bash
sops exec-env $E 'pvectl --insecure template promote <vmid> --to ul-pve11 --fs9100-copy --repo <worktree> --yes -o json'
```

- What it does: migrate to ul-pve11, then a full clone from ul-pve11 with
  `storage=FS9100-01-1 target=ul-pve01 pool=stuttgart-things`, then convert
  to a template. Why the copy: Telmate Terraform sends no `storage` on clone
  and gets 403 on `DD-sthings`.
- Idempotent. After an error (exit 5), look at the log with `tasks --follow
  <UPID>` and run the **same** command again. Steps that are already done
  are skipped.
- Exit 4 (checks): report to the user. Use `--skip-checks` only on their
  instruction. If cpu is not `host`, fix the Packer build (#4650).

## 3. Repin PRs

`plan.repins` in the JSON (or `pvectl repin --os <os> --template <vmid>
--copy <copy> --repo <worktree>`) lists the exact keys. Order:

1. **stuttgart-things PR, chart:**
   `crossplane/platform/capabilities/proxmoxvm/values.yaml`
   - ubuntu26: `environments.labul.templateVmId` **and**
     `templates.ubuntu26.templateVmId`
   - other OS: only `templates.<os>.templateVmId`
   - keep the comment above it up to date (date, VMID, "migrated to
     ul-pve11, templateNode stays unset")
   - bump `version` in `Chart.yaml` (minor) and add a line to the version
     comment there. After the merge, `push-platform-charts.yaml` pushes the
     chart to `ghcr.io/stuttgart-things/charts/proxmoxvm`.
2. **stuttgart-things PR, Terraform (FS9100 copy):**
   `backstage/templates/create-terraform-vm/template.yaml` **and**
   `machinery-cluster-labul.yaml`: the `vm_clone_id` default set to the copy
   VMID, **in both files in the same PR**.
   Can go in the same PR as 1 or separately; it does not depend on the
   chart.
3. **flux PR** (stuttgart-things/flux) once the chart exists:
   `cicd/crossplane/capabilities/components/proxmoxvm/release.yaml`, the
   default of `CROSSPLANE_CAPABILITY_PROXMOXVM_CHART_VERSION` set to the
   new chart version (Renovate may open this PR itself). After the merge,
   the Release workflow cuts a flux tag `v1.x.0`.
4. **flux-infra tag** in stuttgart-things:
   `clusters/labda/vsphere/machinery/sources/flux-infra.yaml`, `tag:` set
   to the new flux tag (Renovate opens `update dependency
   stuttgart-things/flux` PRs for this as well). Since flux v1.218.0 the
   capability chart versions come from this tag. To run ahead of the tag,
   set `CROSSPLANE_CAPABILITY_PROXMOXVM_CHART_VERSION` temporarily in
   `clusters/labda/vsphere/machinery/cicd/platform.yaml`, and remove it
   again with the next tag.

Give the user the merge commands in this order: chart PR, Terraform PR,
flux PR, flux-infra PR. Only make the next PR once the release it needs
exists.

## 4. Done when

- `pvectl vms <os> --templates -o json`: the template is on ul-pve11, and
  the copy on ul-pve01 with `-fs9100` and the template flag.
- The repins are merged, the flux-infra tag is deployed on machinery.
- A test clone through both paths (Crossplane proxmoxvm claim and
  create-terraform-vm) gets an IP and the right hostname. That is a write:
  only after the user's OK.
- Old templates stay as they are until the user retires them.
