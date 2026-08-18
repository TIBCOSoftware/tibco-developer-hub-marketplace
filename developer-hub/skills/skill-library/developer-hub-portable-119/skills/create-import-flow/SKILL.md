---
name: create-import-flow
description: >
  Author a new TIBCO Developer Hub import flow for a DevHub Portable workspace. Trigger when
  the user wants to create an import flow, add an import-flow template, build a BW6/BW5/Flogo/EMS
  importer, or scaffold an entry that ingests existing source repositories into the catalog.
  Distinct from create-template: import flows follow the clone → extract → generate → push →
  register pattern using the TIBCO custom actions (tibco:git:clone, tibco:extract-parameters,
  tibco:create-yaml, tibco:git:push). Writes the Template entity YAML (and optional Nunjucks
  entity skeletons) under import-flows/<slug>/ and registers it with an absolute file location
  in devhub-local.yaml so it appears at http://localhost:7007/import-flow after a restart.
---

# create-import-flow

Create an import flow under `import-flows/<slug>/` and wire it into the portable hub so it appears
at `http://localhost:7007/import-flow` after a restart.

## Portable key facts

- One port, no path prefix: UI and API both on `http://localhost:7007`.
- Registration = an **absolute** `type: file` entry under `catalog.locations` in `devhub-local.yaml`,
  then a manual restart (Ctrl-C, `./devhub --config devhub-local.yaml`). No hot reload.
- `catalog.locations` in the overlay **replaces** the bundle's list — append, never rewrite.
- `tibco:git:clone` and `tibco:git:push` need a GitHub token: the user exports `GITHUB_TOKEN` before
  starting the hub. Without it, clone works anonymously (60 req/hour) and push fails.
- All four TIBCO actions used here are bundled in portable, and none of them is dry-run aware.

## What makes an import flow different from a regular template

| Aspect | Regular template | Import flow |
|--------|-----------------|-------------|
| Purpose | Create a new project from scratch | Analyse an existing repo and register it |
| Input | User-provided metadata | GitHub repo URL + app/project name |
| Metadata source | User fills in the form | Extracted from source files (XML, JSON, regex) |
| Skeleton | `skeleton/` via `fetch:template` | `entity-skeletons-<tech>/` (advanced) or inline YAML (simple) |
| Steps | fetch → publish → register | clone → extract → generate → push → register |
| Custom actions | Standard Backstage only | `tibco:git:clone`, `tibco:extract-parameters`, `tibco:create-yaml`, `tibco:git:push` |
| `debug` parameter | Always added | Not used |
| UI location | `/create` | `/import-flow` |

## Canonical references — fetch before generating output

```sh
BASE=https://raw.githubusercontent.com/TIBCOSoftware/tibco-developer-hub/main/tibco-examples
# simple: one Component, inline YAML
curl -s $BASE/import-flow-v2/import-flow-bwce-v2.yaml
curl -s $BASE/import-flow-v2/import-flow-flogo-v2.yaml
# advanced: multiple entity types, Nunjucks skeletons
curl -s $BASE/advanced-import-flows/import-flow-bw6.yaml
curl -s $BASE/advanced-import-flows/entity-skeletons-bw6/component.yaml.njk
```

Action schemas as installed in *this* hub: `http://localhost:7007/create/actions`.

## Workflow

### 1. Gather inputs (single batched AskUserQuestion)

- **Slug** — kebab-case, e.g. `import-bw6-orders`. Folder, file name, `metadata.name`.
- **Title** / **Description** — shown at `/import-flow`.
- **Technology** — `BW6` · `BW5` · `Flogo` · `EMS Server` · `Custom`.
- **Complexity**:
  - *Simple* — one `Component` via `tibco:create-yaml`. Only when a single entity type is needed
    **and** no field needs a `$text`/`$yaml`/`$json` reference.
  - *Advanced* — Nunjucks `.njk` skeletons + `fetch:template`. **Required** when multiple entity
    types are needed, or any field needs `$text`/`$yaml`/`$json` (e.g. `API.spec.definition.$text`).
- **Entity types** (advanced, multi-select): `Component` · `System` · `API` · `Resource`.
- **Extraction definitions** — for each: parameter name (snake_case), type (`xml`/`json`/`file`/
  `workspace`), file path expression, query (xPath / jsonPath / regex). Pre-fill by technology:
  - BW6: `bw6_project_name`, xml, xPath `string(/projectDescription/name)` from `.project`
  - Flogo: `flogo_project_name`, json, jsonPath `$.name` from `<app>.flogo`
  - BW5: `bw5_project_name` via workspace scan
  - EMS: `ems_server_name` via file regex
- **Extra tags** (multi-select; `import-flow` + `tibco` pre-ticked): `bw6` · `flogo` · `bw5` · `ems` ·
  `developer-hub` · `recommended`.
- **Owner** — default `group:default/tibco-imported`.

### 2. Resolve `spec.type`

| Technology | `spec.type` |
|-----------|-------------|
| BW6 / BW5 / Flogo | `integration` |
| EMS Server | `messaging` |
| Simple (any) | `import-flow` |
| Custom | ask, else `import-flow` |

### 3. Create the folder

Simple:
```
import-flows/<slug>/<slug>.yaml
```

Advanced:
```
import-flows/<slug>/
  <slug>.yaml
  entity-skeletons-<tech>/
    component.yaml.njk        # always
    system.yaml.njk           # if System selected
    apis.yaml.njk             # if API selected
    resources.yaml.njk        # if Resource selected
```

No `skeleton/` directory — import flows do not use the standard skeleton pattern.

### 4. Generate `<slug>.yaml`

#### Parameters — always these two pages

```yaml
parameters:
  - title: Repository Location
    required:
      - repoUrl
    properties:
      repoUrl:
        title: GitHub repository with existing <Technology> project
        type: string
        ui:field: RepoUrlPicker
        ui:options:
          allowedHosts:
            - github.com

  - title: Fill in some steps
    required:
      - application
      - owner
    properties:
      application:
        title: <Technology> Application
        type: string
        description: Name of the <Technology> application to import
      owner:
        title: Owner
        type: string
        description: Owner of the component in the catalog
        ui:field: OwnerPicker
        ui:options:
          allowedKinds:
            - Group
```

Advanced flows add `application_folder` (folder containing the app) and `system` (catalog system name).

#### Steps — simple pattern

```yaml
steps:
  - id: clone
    name: Clone the Project
    action: tibco:git:clone
    input:
      failOnError: true
      repoUrl: ${{ "https://" + (parameters.repoUrl | parseRepoUrl).host + "/" + (parameters.repoUrl | parseRepoUrl).owner + "/" + (parameters.repoUrl | parseRepoUrl).repo }}

  - id: extract
    name: Extract Parameters
    action: tibco:extract-parameters
    input:
      failOnError: true
      extractParameters:
        <param_name>:
          type: <xml|json|file|workspace>
          filePath: ${{ parameters.application + "/..." }}
          xPath: string(/projectDescription/name)   # xml
          # jsonPath: $.name                        # json (or xml alternative)
          # regex: (pattern)                        # file / workspace

  - id: createYaml
    name: Create YAML
    action: tibco:create-yaml
    input:
      outputFile: ${{ parameters.application + "/" + parameters.application + "-<tech>-catalog-info.yaml" }}
      outputStructure:
        apiVersion: backstage.io/v1alpha1
        kind: Component
        metadata:
          name: ${{ steps.extract.output.<param_name>[0] }}
          description: ${{ steps.extract.output.<desc_param>[0] }}
          tags:
            - <tech-tag>
          annotations:
            github.com/project-slug: ${{ "https://" + (parameters.repoUrl | parseRepoUrl).host + "/" + (parameters.repoUrl | parseRepoUrl).owner + "/" + (parameters.repoUrl | parseRepoUrl).repo }}
        spec:
          type: <spec.type>
          lifecycle: production
          owner: ${{ parameters.owner }}

  - id: push
    name: Push Current Repo
    action: tibco:git:push
    input:
      failOnError: true

  - id: register
    name: Register
    action: catalog:register
    input:
      catalogInfoUrl: ${{ "https://" + (parameters.repoUrl | parseRepoUrl).host + "/" + (parameters.repoUrl | parseRepoUrl).owner + "/" + (parameters.repoUrl | parseRepoUrl).repo + "/blob/main/" + parameters.application + "/" + parameters.application + "-<tech>-catalog-info.yaml" }}
```

#### Steps — advanced pattern

Replace `createYaml` with `fetchRS` + `rename`:

```yaml
  - id: fetchRS
    name: Resource Skeleton
    action: fetch:template
    input:
      url: ./entity-skeletons-<tech>/
      targetPath: ${{ parameters.application_folder }}
      templateFileExtension: true
      values:
        repoUrl: ${{ parameters.repoUrl }}
        owner: ${{ parameters.owner }}
        application_folder: ${{ parameters.application_folder }}
        application: ${{ parameters.application }}
        <tech>_system: ${{ parameters.system }}
        <tech>_project_name: ${{ steps.extract.output.<param_name>[0] }}

  - id: rename
    name: Rename Descriptor Files
    action: fs:rename
    input:
      files:
        - from: ${{ parameters.application_folder + "/system.yaml" }}
          to: ${{ parameters.application_folder + "/system-" + parameters.application + ".yaml" }}
          overwrite: true
        - from: ${{ parameters.application_folder + "/component.yaml" }}
          to: ${{ parameters.application_folder + "/components-" + parameters.application + ".yaml" }}
          overwrite: true
```

and use one `catalog:register` step per entity type (`registerSystem`, `registerComponents`, …), each
with the matching `catalogInfoUrl`.

#### Output

```yaml
output:
  links:
    - title: Open in catalog
      icon: catalog
      entityRef: ${{ steps.register.output.entityRef }}
    - title: Repository (Pull Request)
      url: ${{ steps.push.output.remoteUrl }}
```

Advanced flows list one catalog link per registered entity type.

### 5. Nunjucks skeletons (advanced only)

Model on `entity-skeletons-bw6/*.njk` fetched in the references step.

- `${{ values.<param> }}` for substitution
- `{%- if(values.field) %}…{%- endif %}` for optional sections
- `{%- for item in values.list %}…{%- endfor %}` for repeats
- Generic file names (`component.yaml.njk`) — `fs:rename` produces the final names
- `apiVersion: backstage.io/v1alpha1` always

```yaml
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: ${{ values.<tech>_project_name }}
{%- if(values.<tech>_project_description) %}
  description: ${{ values.<tech>_project_description }}
{%- endif %}
  tags:
    - <tech-tag>
spec:
  type: <spec.type>
  lifecycle: production
  owner: ${{ values.owner }}
{%- if(values.<tech>_system) %}
  system: ${{ values.<tech>_system }}
{%- endif %}
```

### 6. Register in `devhub-local.yaml`

`pwd` for the absolute path, then append under `catalog.locations`, preserving every existing entry
(including commented-out ones) and skipping duplicates:

```yaml
    - type: file
      target: /abs/path/to/my-devhub-content/import-flows/<slug>/<slug>.yaml
```

### 7. Restart hint

> Restart the hub: **Ctrl-C** in its terminal, then `./devhub --config devhub-local.yaml`.

Then confirm ingestion:

```sh
curl -s -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' \
     -X POST "$HUB/api/catalog/entities/by-refs" \
     -d '{"entityRefs":["template:default/<slug>"]}'
```

A `null` in `items` means it is not registered — check the path and that the hub was restarted.

### 8. Verify (best-effort)

With Playwright MCP tools: navigate to `http://localhost:7007/import-flow`, confirm the card appears
with the right title and description, click into it and confirm the repo picker and application
fields render. Then suggest `/test-import-flow` for an actual run.

## Don't

- Don't add a `debug` parameter unless asked. If added, guard only `push` and `register` —
  `clone` and `extract` must always run because everything downstream depends on their output.
- Don't create a `skeleton/` directory.
- Don't use `publish:github` — import flows write back with `tibco:git:push`.
- Don't use `fetch:plain` instead of `tibco:git:clone`; the TIBCO action handles auth and
  `failOnError` semantics.
- Don't omit `failOnError: true` on clone and extract.
- Don't read `steps.extract.output.<param>` directly — it is always an array; use `[0]`.
- Don't put `$text`, `$yaml` or `$json` keys inside `tibco:create-yaml`'s `outputStructure`. When the
  template is registered, Backstage's PlaceholderProcessor tries to resolve them and fails. Use the
  advanced `.njk` pattern instead — `.njk` files are never processed as catalog entities.
- Don't use a relative `target:` in `devhub-local.yaml`, and don't edit the bundle's
  `app-config.portable.yaml`.
- Don't tag the flow `devhub-internal` — it suppresses the entry from `/import-flow`.
- Don't restart the hub for the user.
