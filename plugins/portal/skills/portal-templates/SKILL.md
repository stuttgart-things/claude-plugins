---
name: portal-templates
description: Find the right developer-portal (Backstage) Scaffolder template or offering for what the user wants to order or create - a VM, a cluster, a Crossplane claim, a repo, a change or deletion request - and hand them the link to the form. Use when the user asks how to order, request or create something on the stuttgart-things platform, or which template to use, and an sthings-portal-* MCP server is connected.
---

# Find a template in the portal

Templates are catalog entities of kind `Template`; the portal cannot run them
for you (its MCP server is read-only), so the result is **a link to the form**
plus what the user will need to fill in.

## Find candidates

```json
{"query": {"kind": "Template"}, "fields": ["metadata.name", "metadata.title", "metadata.description", "metadata.tags", "spec.type", "metadata.annotations.backstage.stuttgart-things.com/audience"]}
```

There are a few dozen templates: fetch them all with name, title and
description and choose from those. Narrow with `metadata.tags`
(`{"$contains": "vm"}`) or `spec.type` if needed. Do not rely on
`fullTextFilter` (it misses obvious matches). For the whole template (its
parameters), `catalog.get-catalog-entity` with `kind: "Template"`.

Curated entry points are the offerings: `Resource`s of type `offering`
(`/offerings` in the portal). Their `metadata.links` point at the templates,
written as `portal:/create/templates/default/<name>`: `portal:` stands for the
portal's own host.

## Who can use it

Templates are platform-only unless annotated
`backstage.stuttgart-things.com/audience: developers`. Developers and guests do
not see the others at all. The MCP token reads like platform-team, so check
the annotation and tell a non-platform user when a template is not for them,
instead of sending a link that ends in "not found".

## Answer

- The form: `/create/templates/default/<template-name>` on the user's portal
  (host from their MCP config). Offerings link the same way.
- Summarize the required parameters from `spec.parameters` (the `required`
  lists), so the user knows what to have ready.
- If nothing fits, say so; requests outside the templates go to platform-team.
