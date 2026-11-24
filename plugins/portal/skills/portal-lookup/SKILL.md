---
name: portal-lookup
description: Answer questions about stuttgart-things software and infrastructure from the developer portal (Backstage) through its MCP server - who owns a component, API, VM, cluster or template, which system it belongs to, where its repo and docs are, what is registered for a team. Use whenever the user asks "who owns", "where is", "what belongs to", "is there a … for", or names a stuttgart-things service, system or team, and an sthings-portal-* MCP server is connected.
---

# Look things up in the portal

The portal's MCP servers are named `sthings-portal-<lab>` (e.g.
`sthings-portal-labul`, `sthings-portal-labda`). Both instances read the same
catalog files from Git, so for catalog questions either one answers; prefer
the lab the user names. If no such server is connected, say so and point to
the portal docs (`docs/mcp.md` in stuttgart-things/sthings-backstage) instead
of guessing.

Everything here is **read-only**. The portal cannot register, change or delete
anything for you, and the tools do not show secrets.

## Tools

| Tool | Use |
| --- | --- |
| `catalog.query-catalog-entities` | find entities: input `query` (predicates), `fields`, `limit` |
| `catalog.get-catalog-entity` | one entity by `name` (`kind`, `namespace` optional) |
| `catalog.get-catalog-model-description` | kinds and fields, if a query needs a field you do not know |
| `catalog.validate-entity` | check a `catalog-info.yaml` before it is committed |
| `search.query` | full text over catalog, TechDocs and announcements (`term`) |
| `techdocs.get-techdocs-metadata` | docs site of an entity (`name`, `kind`) |

## Queries that work

The parameter is **`query`** (not `filter`): an unknown parameter is ignored
and you get unrelated entities back. Always pass `fields` to keep answers
small.

```json
{"query": {"kind": "Component", "spec.type": "service"}, "fields": ["metadata.name", "spec.owner", "spec.system"]}
```

- Owned by a team: `{"relations.ownedby": "group:default/platform-team"}`.
  Use the relation, not `spec.owner`: owners are written short
  (`platform-team`) or as full refs.
- Part of a system: `{"relations.partof": "system:default/homerun2"}`.
- With docs: `{"metadata.annotations.backstage.io/techdocs-ref": {"$exists": true}}`.
- Combine with `$all`, `$any`, `$not`; prefix match with `$hasPrefix`.

Entity refs look like `component:default/homerun2-core-catcher`,
`resource:default/<vm>-<cloud>-vm`, `template:default/create-vm`.

## stuttgart-things conventions

- `spec.type` values with a meaning here: `service`, `website`, `library`,
  `documentation`, `database`, `vm`, `vm-template`, `kubernetes-cluster`,
  `machinery-order`, `offering`.
- VMs are `Resource`s of type `vm`, written by the VM provider; their
  `vm.stuttgart-things.com/*` annotations are the **last inventory run**, not
  the current state. For the current state use the `portal-vm-status` skill.
- Clusters are `Resource`s of type `kubernetes-cluster` with
  `cluster.stuttgart-things.com/*` annotations (status, deployments).
- The repo is `github.com/project-slug`, the source
  `backstage.io/source-location`.

## Answering

- Give the entity ref and the portal path, e.g. `/catalog/default/component/<name>`
  and `/docs/default/component/<name>` for docs. The user's portal host is in
  their MCP config, not in this skill.
- `techdocs.get-techdocs-metadata` only knows docs that were built once (the
  portal builds them when someone opens the Docs tab). "No TechDocs metadata
  found" with a `backstage.io/techdocs-ref` annotation means: not built yet,
  link `/docs/...` anyway.
- Say when something is **not** in the catalog instead of inventing an owner.
