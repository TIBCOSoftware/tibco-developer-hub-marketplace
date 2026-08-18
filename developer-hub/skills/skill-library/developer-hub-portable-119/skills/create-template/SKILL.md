---
name: create-template
description: >
  Author a new Backstage scaffolder template for a TIBCO Developer Hub Portable workspace.
  Trigger when the user wants to create a new software template, add a scaffolder template,
  write a BW6/Flogo/service template, or scaffold a new entry under templates/. Gathers
  metadata, parameters and steps via AskUserQuestion, writes the Template entity YAML plus a
  starter skeleton/ (with catalog-info.yaml) under templates/<slug>/, and registers it with an
  absolute file location in devhub-local.yaml so it appears at http://localhost:7007/create
  after the portable hub is restarted.
---

# create-template

Create a software template under `templates/<slug>/` and wire it into the portable hub's catalog so
it appears at `http://localhost:7007/create` after a restart.

## Portable key facts

- **One port, no path prefix.** UI and API are both on `http://localhost:7007` (or whatever the
  launcher printed). There is no `:3000` and no `/tibco/hub`.
- **Registration is an absolute `type: file` target in `devhub-local.yaml`.** No
  `app-config.local.yaml`, no `../../` relative paths — the backend's cwd is wherever the user
  launched `devhub`.
- **`catalog.locations` replaces, not merges.** Your overlay's array wins over the bundle's, so
  never drop existing entries when appending.
- **No hot reload.** A new location needs Ctrl-C and a restart. Say so; don't restart it yourself.

## Canonical reference — read before generating output

Fetch the reference template first; do not write a template from memory:

```sh
curl -s https://raw.githubusercontent.com/TIBCOSoftware/tibco-developer-hub/main/tibco-examples/bwce-bookstore-template/bwce-bookstore-template.yaml
```

More examples via the tree API
(`https://api.github.com/repos/TIBCOSoftware/tibco-developer-hub/git/trees/main?recursive=1`,
filter for `tibco-examples/`). The live schema for every installed action is at
`http://localhost:7007/create/actions`.

## Workflow

### 1. Gather inputs (AskUserQuestion, batched)

Ask in a single tool call where possible; multi-select for tags.

- **Slug** — kebab-case, e.g. `bw6-orders`, `flogo-webhook`. Folder name, file name, `metadata.name`.
- **Title** — shown on the card.
- **Description** — one sentence under the title.
- **Type** — `service` / `bw6` / `flogo` / `website` / `library` / Other → `spec.type`.
- **Owner** — Backstage group ref, default `group:default/tibco-templates`.
- **Tags** (multi-select) — `tibco`, `bw6`, `flogo`, `recommended`, plus Other. Tags decide the page:
  `import-flow` → `/import-flow` (use the `create-import-flow` skill instead), `self-service` →
  `/self-service-flow` (use `create-self-service-flow`), `devhub-marketplace` → `/marketplace`,
  anything else → `/create`.
- **Parameters** — default set: `name` (EntityNamePicker, required), `description`, `owner`
  (OwnerPicker, Group), `repoUrl` (RepoUrlPicker, github.com, required). Ask for extras
  (string/number/enum) with title, type, default, required.
- **Publish target** — `publish:github` (default) / none (catalog-only).
- **Skeleton content** — copy an existing directory the user names / generate a minimal stub.

Always add a `debug` boolean (default `false`) and guard every side-effecting step with
`if: ${{ not parameters.debug }}`. That is what makes `test-template` safe.

### 2. Create the folder

```
templates/<slug>/
  <slug>.yaml
  skeleton/
    catalog-info.yaml
    README.md
```

Generate fresh content, or `cp -R` a source directory the user explicitly named. Don't copy a
reference example wholesale and rename it.

### 3. Generate the Template entity

```yaml
apiVersion: scaffolder.backstage.io/v1beta3
kind: Template
metadata:
  name: <slug>
  title: <Title>
  description: <Description>
  tags:
    - <tag-1>
spec:
  owner: <owner-group>
  type: <type>

  parameters:
    - title: Fill in some steps
      required:
        - name
      properties:
        name:
          title: Name
          type: string
          description: Unique name of the component
          ui:field: EntityNamePicker
          ui:autofocus: true
        description:
          title: Description
          type: string
          description: A description for the component
        owner:
          title: Owner
          type: string
          description: Owner of the component
          ui:field: OwnerPicker
          ui:options:
            allowedKinds:
              - Group
    - title: Choose a location
      required:
        - repoUrl
      properties:
        repoUrl:
          title: Repository Location
          type: string
          ui:field: RepoUrlPicker
          ui:options:
            allowedHosts:
              - github.com
        debug:
          title: Debug mode
          type: boolean
          description: Skip publish and register steps — use for dry-run testing
          default: false

  steps:
    - id: fetch
      name: Fetch Skeleton
      action: fetch:template
      input:
        url: ./skeleton
        values:
          name: ${{ parameters.name }}
          description: ${{ parameters.description }}
          destination: ${{ parameters.repoUrl | parseRepoUrl }}
          owner: ${{ parameters.owner }}

    - id: publish
      name: Publish
      if: ${{ not parameters.debug }}
      action: publish:github
      input:
        description: This is ${{ parameters.name }}
        repoUrl: ${{ parameters.repoUrl }}

    - id: register
      name: Register
      if: ${{ not parameters.debug }}
      action: catalog:register
      input:
        repoContentsUrl: ${{ steps.publish.output.repoContentsUrl }}
        catalogInfoPath: '/catalog-info.yaml'

  output:
    links:
      - title: Repository
        url: ${{ steps.publish.output.remoteUrl }}
      - title: Open in catalog
        icon: catalog
        entityRef: ${{ steps.register.output.entityRef }}
```

Forward every extra parameter into `steps.fetch.input.values.<name>`. If publish is `none`, drop the
`publish`/`register` steps and the `output` block.

### 4. Generate the skeleton

`skeleton/catalog-info.yaml` (always):

```yaml
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: ${{values.name | dump}}
  description: ${{values.description | dump}}
  annotations:
    github.com/project-slug: ${{values.destination.owner + "/" + values.destination.repo}}
    backstage.io/techdocs-ref: dir:.
spec:
  type: <type>
  lifecycle: development
  owner: ${{values.owner | dump}}
```

Drop the `project-slug` annotation when there is no publish step. `skeleton/README.md` gets
`# ${{values.name}}` plus the description.

### 5. Register in `devhub-local.yaml`

1. `pwd` to get the workspace's absolute path.
2. Read `devhub-local.yaml` (create it if missing — the bundle does not ship one).
3. Append under `catalog.locations`, preserving every existing entry **including commented-out ones**:

```yaml
catalog:
  locations:
    # …existing entries kept verbatim…
    - type: file
      target: /abs/path/to/my-devhub-content/templates/<slug>/<slug>.yaml
```

4. Idempotent: skip if that exact `target:` is already present.

Use `Edit` with a unique anchor (the last existing entry), not a whole-file rewrite.

### 6. Restart hint

The hub reads `--config` once at startup:

> Restart the hub to pick up the new location: press **Ctrl-C** in the terminal running it, then
> `./devhub --config devhub-local.yaml`.

Don't kill or start the process yourself.

### 7. Verify (best-effort)

After the user confirms the restart:

```sh
curl -s -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' \
     -X POST "$HUB/api/catalog/entities/by-refs" \
     -d '{"entityRefs":["template:default/<slug>"]}'
```

Then, if Playwright MCP tools are available, navigate to `http://localhost:7007/create`, confirm the
card shows the right title and description, and click into it to confirm the form renders. Do **not**
run it through to publish — that creates a real GitHub repo. Suggest `/test-template` instead.

If the entity does not appear, read the hub's terminal output for catalog ingestion errors — usual
causes are malformed YAML, a missing `spec.owner`, a relative `target:` path, or a `catalog.locations`
overlay that replaced the list you thought you were appending to.

## Don't

- Don't write template files outside `templates/<slug>/`.
- Don't use a relative `target:` in `devhub-local.yaml` — the backend cwd is not this folder.
- Don't edit `app-config.portable.yaml` inside the bundle; that file is replaced on every reinstall.
  All local config belongs in your untracked `devhub-local.yaml` overlay.
- Don't drop the stock example-catalog location when appending, unless the user wants an empty hub.
- Don't tag the template `devhub-internal`; it hides the entry from listings.
- Don't forget `fetch:template` runs the skeleton through Nunjucks — literal `${{ }}` in pasted code
  samples needs `{% raw %}…{% endraw %}` fences. Flogo `.flogo` JSON does **not** need them; its
  `$activity[...]` syntax does not collide with Nunjucks.
- Don't restart the hub for the user.
