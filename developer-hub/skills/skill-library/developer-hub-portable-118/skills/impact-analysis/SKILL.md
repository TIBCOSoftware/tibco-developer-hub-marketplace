---
name: impact-analysis
description: >
  Produce a detailed change-impact analysis for a TIBCO Developer Hub catalog entity (API,
  Component, Resource, System) by reading its real dependency graph from the running DevHub
  Portable hub via the catalog REST API, classifying every related entity into impact tiers, and
  writing a report plus color-coded topology Mermaid diagrams into impact_analysis/. Trigger when
  the user wants an impact analysis, a blast-radius assessment, "what breaks if I change X", a
  dependency or ripple analysis, who-consumes-this, or a coordination/ownership view before
  modifying a catalog entity.
---

# impact-analysis

Answer "what is affected if I change `<entity>`?" from the **live catalog graph**, not guesses.
Resolve the subject → traverse its relations → classify each neighbour into a tier → write a report
plus color-coded topology diagrams under `impact_analysis/`.

## Key facts

- **Data source**: the catalog REST API of the running portable hub — `http://localhost:7007/api/catalog`.
  Same origin as the UI (portable serves both on one port); if 7007 was busy the launcher moved up,
  so read the port from the startup log or probe 7007–7016.
- **Auth**: guest mode is **not** anonymous. Every `/api/*` call needs a bearer token or it answers
  HTTP 500 `AuthenticationError: Missing credentials` (not 401 — don't misread it as a server bug).
  Mint one, valid for an hour:

  ```sh
  HUB=http://127.0.0.1:7007
  TOKEN=$(curl -s "$HUB/api/auth/guest/refresh" | python3 -c 'import sys,json;print(json.load(sys.stdin)["backstageIdentity"]["token"])')
  ```

  `/.backstage/health/v1/readiness` is the only route that answers without it — use it to check the
  hub is up and find which port it took.
- **No MCP server** in this Backstage version — use the REST endpoints below.
- **Endpoints**:
  - `GET /entities/by-name/{kind}/{namespace}/{name}` — one entity plus its `relations`
  - `POST /entities/by-refs` with `{"entityRefs":[…]}` — batch-fetch neighbours in one call
  - `GET /entities?filter=…` — query when you only have a partial selector
- **The graph lives in `relations`**: `apiProvidedBy`/`apiConsumedBy`, `providesApi`/`consumesApi`,
  `dependsOn`/`dependencyOf`, `partOf`/`hasPart`, `ownedBy`/`ownerOf`,
  `subcomponentOf`/`hasSubcomponent`.
- **API entities are passive** — they only surface `apiProvidedBy`/`apiConsumedBy`/`ownedBy`/`partOf`,
  never `dependsOn`. To find what an API depends on, look at the components/resources pointing at it.
- **Output**: everything under `impact_analysis/` — `<entity>-impact-analysis.md` plus
  `catalog-data-snapshot.md` for provenance.
- **Sandbox**: localhost curls need `dangerouslyDisableSandbox: true`.

```sh
CATALOG="http://127.0.0.1:7007/api/catalog"
AUTH=(-H "Authorization: Bearer $TOKEN")

curl -s "${AUTH[@]}" "$CATALOG/entities/by-name/api/default/<entity-name>" | python3 -m json.tool

curl -s "${AUTH[@]}" -X POST "$CATALOG/entities/by-refs" -H "Content-Type: application/json" \
  -d '{"entityRefs":["component:default/foo","api:default/bar"]}' | python3 -m json.tool

curl -s "${AUTH[@]}" "$CATALOG/entities?filter=kind=component,spec.system=car-order-system" | python3 -m json.tool
```

A missing entity yields 404. `by-refs` returns an `items` array aligned to the refs you sent (nulls
for misses).

## Workflow

### 1. Preflight

```sh
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:7007/.backstage/health/v1/readiness
```

Non-200 → the hub is not running, or took a different port (probe 7008–7016). Tell the user to run
`./scripts/devhub-start.sh`; don't start it yourself. Then mint the guest token as shown above —
`node scripts/catalog-check.mjs <Kind>:<name>` is a quick way to confirm the whole chain works.

A near-empty catalog is normal on a fresh portable install: it only holds what
`devhub-local.yaml` registers. If the subject entity is not there, say so rather than analysing a
stub — the user may need to install a sample system from the Marketplace first.

### 2. Resolve the subject

Get kind + name (ask via `AskUserQuestion` if ambiguous). Fetch with
`GET /entities/by-name/{kind}/default/{name}` and record `kind`, `spec.type`, `spec.lifecycle`,
`spec.owner`, `spec.system` and the full `relations` array. A `production` lifecycle raises the risk
bar for breaking changes.

### 3. Traverse the graph

Breadth-first. Collect neighbour refs from `relations`, dedupe, and fetch them in one
`POST /entities/by-refs` so you get their relations and specs too. Repeat per hop; stop at ~2–3 hops
or the system boundary. Capture kind, type, owner, and which relations connect each back toward the
subject. The raw findings become `catalog-data-snapshot.md`.

### 4. Classify into tiers

- **Subject is an API**
  - `apiProvidedBy` (the implementing component) → 🔴 **Direct**: must implement the contract change.
  - `apiConsumedBy` (every caller) → 🔴 **Direct**: breaking changes break them.
  - The provider's own upstream (`consumesApi`/`dependsOn`) → 🟠 **Conditional**: impacted only if the
    new contract needs data the provider must source.
- **Subject is a message contract** (`spec.type: ems-message` / event / asyncapi)
  - **No `apiProvidedBy`? The producer is external.** Inbound messages often have no internal
    provider — the emitter is an external `Resource` reached via a consumer's `dependsOn`. Treat that
    Resource as a 🔴 **Direct source** that must conform, and note its owner: frequently a different
    team than the contract owner. Classic cross-team ripple.
  - **Trace the FIELD, not just the entity.** For a field-level change, follow only paths that carry
    that field. A consumer that re-publishes a *different* downstream message propagates the change
    only if that schema **contains the field** — inspect the definitions. If it doesn't, mark it 🟠
    ("only if a currently-used value is dropped"), not 🔴.
  - The EMS destination `Resource` is **transport** — 🟢 for a payload-field change.
- **Subject is a Component** — consumers of APIs it provides, plus `dependencyOf` /
  `hasSubcomponent` → 🔴; APIs/resources it consumes → 🟠.
- **Subject is a Resource** — components with `dependsOn` on it → 🔴.
- **No path to the subject** → 🟢, and state *why*.
- **Cross-team ripple** → flag 🟠 when a 🟠 upstream entity is also consumed by a different team's
  component. This is the most valuable non-obvious finding — always look for it.

Record `ownedBy` for every impacted entity; it drives the notify list.

### 5. Write the report

`impact_analysis/<entity>-impact-analysis.md`:

1. **Header** — subject ref, system, domain, date, data source (this hub's URL).
2. **Executive summary** — the blast radius in 2–3 sentences plus a tier-count table. Lead with the
   sharpest insight.
3. **Color legend** (below).
4. **Three diagrams** (section 6).
5. **Detailed impact by entity** — relationship, why it's in that tier, the concrete action.
6. **Risk assessment** — additive vs breaking vs needs-new-data, each with risk + radius.
7. **Recommendations & pre-merge checklist** — versioning, contract tests, keep-upstream-additive,
   update the OpenAPI/catalog YAML, deploy order, and an explicit **notify** list by team.
8. **Provenance** — link to `catalog-data-snapshot.md`.

Every claim must trace to a relation you actually fetched. Do not invent dependencies.

### 6. Color-coded topology diagrams (Mermaid)

Three diagrams sharing this legend and `classDef` block:

| Color | Tier | classDef |
|-------|------|----------|
| 🔴 Red | Change / Direct | `change` (subject), `high` (direct neighbours) |
| 🟠 Amber (dashed) | Conditional / transitive | `cond` |
| 🟢 Green | Not impacted | `safe` |
| 🔵 Blue | Owner / stakeholder | `owner` |

```
classDef change fill:#ff4d4f,stroke:#a8071a,color:#ffffff,stroke-width:3px;
classDef high   fill:#ffccc7,stroke:#cf1322,color:#000000,stroke-width:2px;
classDef cond   fill:#ffe7ba,stroke:#d46b08,color:#000000,stroke-width:1px,stroke-dasharray:5 3;
classDef safe   fill:#d9f7be,stroke:#389e0d,color:#000000;
classDef owner  fill:#bae0ff,stroke:#0958d9,color:#000000,stroke-width:2px;
```

1. **Blast-radius topology** — `flowchart LR`, one node per entity labelled
   `name<br/><i>kind · type</i>`, edges labelled `provides` / `consumes` / `dependsOn`, nodes tinted
   by tier.
2. **Layered change-propagation** — `flowchart TD` with subgraphs `Tier 0 → Tier 1 (direct) → Tier 2
   (conditional) → Not impacted`; dotted edges for conditional hops.
3. **Ownership & coordination** — teams as subgraphs, the entities each owns, dotted edges for
   cross-team co-consumer ripples.

**Mermaid gotchas**: safe node IDs (alphanumeric/underscore), display name in the quoted label,
`<br/>` for multiline.

### 7. Optional exports

Mermaid renders in the IDE preview and on GitHub. If the report is destined for a TechDocs page on
the portable hub, note that TechDocs rendering needs `mkdocs` (Python 3, provisioned on first use) —
keep an ASCII + table fallback for the docs copy and leave the Mermaid original under
`impact_analysis/`. Offer PNG/SVG via `npx @mermaid-js/mermaid-cli` if the user wants standalone
images.
