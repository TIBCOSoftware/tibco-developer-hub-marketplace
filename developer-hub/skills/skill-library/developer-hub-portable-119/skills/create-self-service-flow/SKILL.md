---
name: create-self-service-flow
description: >
  Author a new TIBCO Developer Hub self service flow for a DevHub Portable workspace — a
  Backstage Template that drives the TIBCO Platform through its APIs instead of scaffolding a
  repository. Trigger when the user wants to create a self service flow, add a flow to the Self
  Service page, build & deploy an app to a Data Plane from the Developer Hub, provision a
  capability or connector, expose an app endpoint, or automate a Control Plane / Data Plane
  sequence from a form. Distinct from create-template (scaffolds a new repo) and
  create-import-flow (ingests an existing repo): self service flows use tibco:call-platform-api,
  tibco:file:write, tibco:fetch-api-file and the platform-aware form fields CapabilitySelector /
  DataplaneSelector. Writes the Template entity YAML (plus an optional catalog skeleton) under
  self-service-flows/<slug>/ and registers it in devhub-local.yaml so it appears at
  http://localhost:7007/self-service-flow after a restart.
---

# create-self-service-flow

Create a self service flow under `self-service-flows/<slug>/` and wire it into the portable hub so it
appears at `http://localhost:7007/self-service-flow` after a restart.

## Portable key facts

- One port, no path prefix — the Self Service page is `http://localhost:7007/self-service-flow`.
- **`cpLink` and `TIBCOPlatformToken` go in `devhub-local.yaml`**, not `app-config.local.yaml`. Without
  both, every `tibco:call-platform-api` step fails on its first call. That file is gitignored — never
  commit a token, never print one back to the user.
- Registration = absolute `type: file` entry in `devhub-local.yaml` + a manual restart.
- All the TIBCO platform actions are bundled in portable; `CapabilitySelector` and `DataplaneSelector`
  ship in the prebuilt frontend.
- Live action schemas: `http://localhost:7007/create/actions`.

## What makes a self service flow different

| Aspect | Regular template | Import flow | Self service flow |
|--------|-----------------|-------------|-------------------|
| Purpose | Create a new project | Analyse an existing repo and register it | Execute a sequence of actions on the TIBCO Platform |
| Acts on | GitHub | GitHub + catalog | Control Plane / Data Plane APIs (+ optionally GitHub and catalog) |
| Custom actions | Standard Backstage only | `tibco:git:clone`, `tibco:extract-parameters`, `tibco:create-yaml`, `tibco:git:push` | `tibco:call-platform-api`, `tibco:file:write`, `tibco:fetch-api-file`, `tibco:extract-parameters` |
| Custom form fields | — | — | `CapabilitySelector`, `DataplaneSelector` |
| `spec.type` | `service` / `website` / … | `integration` / `import-flow` | `self-service` (convention) |
| Required tag | — | `import-flow` | **`self-service`** |
| UI location | `/create` | `/import-flow` | `/self-service-flow` + home page card |

A self service flow is the same entity kind as a template. **The `self-service` tag is the only thing
that routes it.** `spec.type: self-service` is convention — set it (it renders as the type chip), but
routing never reads it. A flow also tagged `devhub-marketplace` is excluded from the page.

## Canonical references — fetch before generating output

```sh
BASE=https://raw.githubusercontent.com/TIBCOSoftware/tibco-developer-hub/main/tibco-examples/developer-hub-marketplace-content/self-service-flows
# the canonical end-to-end flow: check → provision → build → deploy → expose → link → register
curl -s $BASE/build-deploy-flogo-app/self-service-build-deploy-flogo-app.yaml
# the same pattern for a BW5CE .ear upload
curl -s $BASE/build-deploy-bw5ce-app/self-service-build-deploy-bw5ce-app.yaml
# the owner group
curl -s $BASE/tibco-self-service-group.yaml
```

Never guess an action's input schema — read it from the reference or from `/create/actions`.

## The custom actions

### `tibco:call-platform-api`

The heart of every flow. Calls any TIBCO Platform API with auth handled for you. Handles JSON bodies,
multipart uploads and URL-encoded data; MIME types come from the file extension (`.zip`, `.json`,
`.flogo`, `.ear`). Defaults to `GET`.

| Input | Type | Required | Description |
|-------|------|----------|-------------|
| `endpoint` | string | Yes | Path to the API endpoint (no host) |
| `method` | string | No | HTTP verb, default `GET` |
| `body` | object | No | JSON payload |
| `filePath` | string | No | Workspace file to upload |
| `contentType` | string | No | `formData` for multipart uploads |
| `formFieldName` | string | No | Form field name for the uploaded file |
| `headers` | object | No | Extra request headers |
| `baseUrl` | string | No | Base URL override — pass the data plane URL for DP calls |
| `requireAuth` | boolean | No | `false` for unauthenticated endpoints |

| Output | Description |
|--------|-------------|
| `status` | HTTP response code |
| `data` | Parsed JSON response body |
| `baseUrl` | Fully resolved URL used |
| `appBaseUrl` | Developer Hub base URL — use for catalog links |
| `cpBrowserUrl` | Control Plane browser URL — use for output links |

**Key rule — Control Plane vs Data Plane.** Omit `baseUrl` for Control Plane calls (`/tp-cp-ws/…`,
`public/v1/cp/…`). Pass `baseUrl: ${{ parameters.deploymentTarget.dataplaneUrl }}` for Data Plane
calls (`public/v1/dp/…`). Getting this wrong is the single most common failure.

Resolution tiers (so the same flow works locally and in production):

| Priority | Base URL source | Token source |
|----------|-----------------|--------------|
| 1 | `baseUrl` action input | `cpToken` action input |
| 2 | `CP_DOMAIN` env var (internal `http://`) | `cpToken` template secret |
| 3 | `cpLink` app config (`https://`) | `TIBCOPlatformToken` app config |

In portable, tier 3 is what you use — both keys live in `devhub-local.yaml`.

### `tibco:file:write`

Writes a string to a workspace file. Two uses: persist an API response so `tibco:extract-parameters`
can query it, or turn a textarea field into an uploadable file. Inputs: `filePath`, `content`,
`overwrite`. When writing an API response, pipe it through `dump`:
`content: ${{ steps['get-cp-flogo-versions'].output.data | dump }}`.

### `tibco:extract-parameters`

Same action as in import flows, almost always `type: json` against a file written by
`tibco:file:write`. Every extracted value is an **array** — use `[0]`, test presence with `.length == 0`.

### `tibco:fetch-api-file`

Pulls an API definition out of the catalog into the workspace. Inputs: `name` (required), `path`
(required), `kind`, `namespace` (default `default`), `preferredApiType`. Outputs `filePath`,
`sourceEntity`, `apiType`.

## The custom form fields

| `ui:field` | Use when |
|-----------|----------|
| `CapabilitySelector` | The flow deploys to a data plane and needs specific capabilities healthy |
| `DataplaneSelector` | The flow only needs a data plane, no capability requirement |

Prefer `CapabilitySelector` for anything that builds or deploys: it queries all data planes,
health-checks the required capability instances, shows only data planes where **all** are healthy, and
auto-selects the first eligible one.

```yaml
    - title: Deployment Details
      required:
        - deploymentTarget
      properties:
        deploymentTarget:
          title: Select Deployment Target
          type: object
          ui:field: CapabilitySelector
          ui:options:
            requiredCapabilities:
              - FLOGO
```

The value is an **object**: `dataplaneId`, `capabilityId`, `dataplaneUrl`, `dataplaneHost`,
`dataplaneName`. With multiple `requiredCapabilities`, the first *PLATFORM*-type entry determines the
deployment URL; *INFRA* capabilities only filter.

## Workflow

### 1. Gather inputs (single batched AskUserQuestion)

- **Slug** — kebab-case, conventionally `self-service-<name>`.
- **Title** / **Description** — shown on the Self Service card.
- **Goal** — *Build & deploy an app* (canonical) · *Provision a capability / connector* ·
  *Import platform apps into the Hub* · *Custom platform sequence*.
- **Technology** (if build/deploy) — `Flogo` · `BW6` · `BW5CE` · `Other`.
- **Deployment target field** — `CapabilitySelector` (default) or `DataplaneSelector`.
- **Required capabilities** (multi-select) — `FLOGO` · `BWCE` · `BW5CE` · `EMS` · `PULSAR` · other.
- **Publish & register?** — does the flow end by publishing `catalog-info.yaml` to GitHub and
  registering the deployed app (default yes for build & deploy).
- **Extra tags** (`self-service` + `tibco` pre-ticked) — `flogo` · `bw6` · `bw5ce` · `deployment` ·
  `developer-hub` · `recommended`.
- **Owner** — default `group:default/tibco-self-service`.

### 2. Create the folder

```
self-service-flows/<slug>/
  <slug>.yaml
  skeleton-<tech>-app/          # only if publishing / registering a catalog entry
    catalog-info.yaml
```

### 3. Generate `<slug>.yaml`

```yaml
apiVersion: scaffolder.backstage.io/v1beta3
kind: Template
metadata:
  name: <slug>
  title: <Title>
  description: '<Description>'
  tags:
    - self-service        # required — this is what routes it to the Self Service page
    - tibco
spec:
  owner: group:default/tibco-self-service
  type: self-service      # convention, not a router
```

Parameters follow three canonical pages: **app details** (name, filename, textarea for the artifact),
**deployment details** (the selector object), **repository location** (`RepoUrlPicker`, only if the
flow publishes).

```yaml
  parameters:
    - title: Provide <Tech> App Details
      required:
        - app_name
      properties:
        app_name:
          title: The name of the <Tech> App
          type: string
          description: Give your app a unique name
          default: 'my-app'
        filename:
          title: Filename
          type: string
          default: 'app.json'
          pattern: '^[a-zA-Z0-9._-]+$'
        content:
          title: <Tech> App Configuration
          type: string
          ui:widget: textarea
          ui:options:
            rows: 20
```

#### Steps — check → provision-if-missing → act

Guard optional work with `if:` so the flow is idempotent. Never provision what is already there.

```yaml
  steps:
    - id: test_connection
      name: Check DP Flogo Versions
      action: tibco:call-platform-api
      input:
        baseUrl: ${{ parameters.deploymentTarget.dataplaneUrl }}
        endpoint: 'public/v1/dp/flogoversions'

    - id: get-cp-flogo-versions
      name: Get CP Flogo Versions
      if: ${{ steps.test_connection.output.data.totalBuildtypes == 0 }}
      action: tibco:call-platform-api
      input:
        baseUrl: ${{ parameters.deploymentTarget.dataplaneUrl }}
        endpoint: 'public/v1/cp/flogoversions'

    - id: save-cp-flogo-versions
      name: Save CP Flogo Versions
      if: ${{ steps.test_connection.output.data.totalBuildtypes == 0 }}
      action: tibco:file:write
      input:
        filePath: flogo-versions.json
        content: ${{ steps['get-cp-flogo-versions'].output.data | dump }}
        overwrite: true

    - id: extract-latest-flogo-version
      name: Extract Latest Flogo Version
      if: ${{ steps.test_connection.output.data.totalBuildtypes == 0 }}
      action: tibco:extract-parameters
      input:
        extractParameters:
          latest_flogo_version:
            type: json
            filePath: flogo-versions.json
            jsonPath: '$[-1:].version'

    - id: provision-flogo-version
      name: Provision Flogo Version
      if: ${{ steps.test_connection.output.data.totalBuildtypes == 0 }}
      action: tibco:call-platform-api
      input:
        baseUrl: ${{ parameters.deploymentTarget.dataplaneUrl }}
        endpoint: 'public/v1/dp/flogoversions/${{ steps["extract-latest-flogo-version"].output.latest_flogo_version[0] }}'
        method: POST
```

Then write the artifact and build it as multipart form data:

```yaml
    - id: write-custom-file
      name: App Data File
      action: tibco:file:write
      input:
        filePath: ${{ parameters.filename }}
        content: ${{ parameters.content }}
        overwrite: true

    - id: build_app
      name: Building App
      action: tibco:call-platform-api
      input:
        baseUrl: ${{ parameters.deploymentTarget.dataplaneUrl }}
        endpoint: 'public/v1/dp/builds'
        method: POST
        filePath: ${{ steps['write-custom-file'].output.filePath }}
        contentType: 'formData'
        formFieldName: ${{ parameters.filename }}
        body:
          request:
            buildName: ${{ parameters.app_name + "-build" }}
        headers:
          'accept': 'application/json'

    - id: wait_for_build
      name: Wait for Build
      action: debug:wait
      input:
        seconds: 30

    - id: deploy_app
      name: Deploying App
      action: tibco:call-platform-api
      input:
        baseUrl: ${{ parameters.deploymentTarget.dataplaneUrl }}
        endpoint: 'public/v1/dp/builds/${{ steps.build_app.output.data.buildId }}/deploy'
        method: POST
        body:
          appId: ''
          buildId: ${{ steps.build_app.output.data.buildId }}
          eula: true
          appName: ${{ parameters.app_name }}
          replicas: 1
```

Linking the app back to the Hub is a **Control Plane** call — no `baseUrl`:

```yaml
    - id: link-deployed-app
      name: Link App
      action: tibco:call-platform-api
      input:
        endpoint: '/tp-cp-ws/v1/data-planes/${{ parameters.deploymentTarget.dataplaneId }}/capabilities/${{ parameters.deploymentTarget.capabilityId }}/apps'
        method: PUT
        body:
          appId: ${{ steps.deploy_app.output.data.appId }}
          appName: ${{ parameters.app_name }}
          appLinks:
            - linkName: ${{ parameters.app_name }}
              linkType: 'developer_hub'
              href: ${{ steps.test_connection.output.appBaseUrl + "/catalog/default/component/" + parameters.app_name }}
          eula: true
```

Note `appBaseUrl` resolves to the portable hub's own root (`http://localhost:7007`), so the link only
works from the machine running the bundle. Fine for local authoring; call it out if the user expects
the link to be shareable.

Finish with `fetch:template` → `publish:github` → `catalog:register` if the user wants registration.
Insert `debug:log` steps after build and deploy so the IDs are visible in the task log — that is what
makes a failed run diagnosable.

#### Output — text plus links

```yaml
  output:
    text:
      - title: Your app has been built and deployed successfully!
        content: |
          **AppName:** ${{ parameters.app_name }}
          **DataPlane:** ${{ parameters.deploymentTarget.dataplaneName }}
          **AppId:** ${{ steps.deploy_app.output.data.appId }}
    links:
      - title: Repository
        icon: github
        url: ${{ steps.publish.output.remoteUrl }}
      - title: Open in catalog
        icon: catalog
        entityRef: ${{ steps.registerItem.output.entityRef }}
      - title: Dataplane Details
        icon: catalog
        url: ${{ steps.test_connection.output.cpBrowserUrl + "/cp/app/data-plane?dp_id=" + parameters.deploymentTarget.dataplaneId }}
```

### 4. Ensure the owner group exists

`group:default/tibco-self-service` must resolve, or `catalog:register` and ownership display fail.
Check with `node scripts/catalog-check.mjs Group:tibco-self-service`. If missing, fetch the reference
group YAML, save it as `self-service-flows/<slug>/tibco-self-service-group.yaml`, and register it as
its own location too.

### 5. Verify platform config in `devhub-local.yaml`

```yaml
cpLink: 'https://<your-control-plane-host>'
TIBCOPlatformToken: '<bearer token>'
```

Report which is missing. Never invent a value; never echo the token.

### 6. Register in `devhub-local.yaml`

`pwd` for the absolute path, then append under `catalog.locations`, preserving existing entries and
skipping duplicates:

```yaml
    - type: file
      target: /abs/path/to/dev-hub-template-workflow/self-service-flows/<slug>/<slug>.yaml
```

### 7. Restart hint

> Restart the hub: **Ctrl-C**, then `./scripts/devhub-start.sh`.

Then `node scripts/catalog-check.mjs Template:<slug>`.

### 8. Verify (best-effort)

With Playwright MCP tools: navigate to `http://localhost:7007/self-service-flow`, confirm the card,
click **Choose**, and confirm the `CapabilitySelector` populates with real data planes instead of
erroring — that is the fastest check that `cpLink` and the token are right. Then suggest
`/test-self-service-flow`.

## Don't

- Don't omit the `self-service` tag — `spec.type` alone does nothing.
- Don't add `devhub-marketplace` to a flow you want on the Self Service page.
- Don't pass `baseUrl` on Control Plane calls, and don't omit it on Data Plane calls.
- Don't treat `deploymentTarget` as a string — it is an object.
- Don't read `tibco:extract-parameters` output directly — always `[0]`.
- Don't feed an API response into `tibco:file:write` without `| dump`.
- Don't provision unconditionally — every provisioning step needs an `if:` guard.
- Don't chain build → deploy with no wait; the build is asynchronous.
- Don't use hyphenated step IDs in dot notation — `steps.get-cp-flogo-versions` parses as
  subtraction. Use `steps['get-cp-flogo-versions']`.
- Don't add a `debug` boolean as in `create-template`; platform actions are not dry-run aware, so it
  gives false confidence. Use `/test-self-service-flow`.
- Don't hardcode a data plane URL, app ID or Control Plane host.
- Don't write `TIBCOPlatformToken` into anything but the gitignored `devhub-local.yaml`.
- Don't restart the hub for the user.
