# AGENTS.md

> ## Which bundle this set is for
>
> This set targets **DevHub Portable on the 1.19 line** — `portable-v1.19.0` (released 7 Aug 2026)
> or newer. That build is Backstage 1.51.0 (`@backstage/plugin-catalog-backend` 3.8.0) and includes
> `@backstage/plugin-mcp-actions-backend`, so the MCP server is available exactly as in the
> open-source `developer-hub-119` set.
>
> On an older 1.18-line bundle (Backstage 1.41.x) use `developer-hub-portable-118` instead. This set
> still works there — every skill keeps its REST path — it just never finds the MCP server.
>
> Check which line you are on with `cat <bundle>/.devhub-release`, or:
> `node -e "console.log(require('./node_modules/@backstage/plugin-catalog-backend/package.json').version)"`
> — `3.8.x` is the 1.19 line, `3.4.x` the 1.18 line. **Do not look for the MCP plugin in
> `node_modules`:** the portable build compiles it into `index.js`, so `ls node_modules/@backstage |
> grep mcp` finds nothing even when MCP is running. Probe the endpoint instead (see "MCP on
> portable").

Project documentation for AI coding agents (Codex, Copilot, Cursor, Devin, …).
Claude Code users: see `CLAUDE.md`, which imports this file and adds the skill list.

## What this is

**DevHub Portable** is a self-contained download that runs the whole TIBCO® Developer Hub (UI + API)
in one process — no Docker, no Postgres, no Yarn monorepo and no build step. This documentation and
the skills beside it turn that download into an **authoring workspace** for Developer Hub content:
software templates, import flows and self service flows.

You write YAML; the portable hub loads it, renders the form, and runs the steps.

The unzipped bundle folder **is** the install — there is no outer git repo and no scaffolding around
it. This is everything it contains:

```
devhub-bundled-<platform>/         # the unzip target; the whole install
  devhub                           # the launcher — start the hub with ./devhub
  index.js                         # the entire backend, one esbuild bundle
  node/  node_modules/             # embedded Node runtime + native modules
  data/                            # SQLite state, persists across restarts
  .venv/                           # mkdocs for TechDocs, provisioned on first run
  app-config.portable.yaml         # the stock config, always loaded
  .devhub-release                  # which release this bundle came from
  AGENTS.md  CLAUDE.md  MCP-TOOLS.md    # this documentation
  .claude/skills/                  # the runbooks
```

**Nothing in the bundle needs to be edited, and nothing you author has to live inside it.** The hub
reads your content by absolute path, so keep it in a folder of your own — that way re-downloading or
upgrading the bundle never touches your work:

```
my-devhub-content/                          # yours; name and location are free
  devhub.yaml                               # config overlay -> ./devhub --config <abs path>
  templates/<slug>/<slug>.yaml + skeleton/  # software templates -> /create
  import-flows/<slug>/<slug>.yaml           # import flows       -> /import-flow
  self-service-flows/<slug>/<slug>.yaml     # self service flows -> /self-service-flow
```

That layout is a convention these skills follow, not something the bundle creates or requires.

## The portable runtime — what is different from the open-source repo

If you know the `tibco-developer-hub` monorepo, unlearn these:

| | Open-source repo | DevHub Portable (this workspace) |
|---|---|---|
| Start | `yarn start` | `./devhub` |
| Ports | frontend `:3000`, backend `:7007` | **one** port, `7007` by default |
| App URL | `http://localhost:3000/tibco/hub` | `http://localhost:7007/` (no path prefix) |
| API URL | `http://localhost:7007/api/...` | `http://localhost:7007/api/...` (same origin as the UI) |
| Config | `app-config.local.yaml` | a file of your own, layered via `--config` |
| Registering a template | relative path from `packages/backend/` | **absolute** `type: file` target |
| Reload after config change | Backstage CLI hot-restarts | **Ctrl-C and start again** — no watcher |
| Database | Postgres or in-memory SQLite | SQLite files under `<bundle>/data/`, persistent |
| Auth | configurable providers | guest only — but API calls **do** need a token (see below) |
| Frontend changes (themes, plugins) | possible | **impossible** — the frontend is prebuilt |

Everything else — the scaffolder, the TIBCO custom actions, the catalog, tag-based routing — is
the same product and behaves identically.

## Running the hub

Run the launcher from the bundle root:

```bash
./devhub                                  # start on http://localhost:7007
./devhub --port 8088                      # pick a port
./devhub --config /abs/path/devhub.yaml   # layer your own config (repeatable)
./devhub --help
```

**The default port falls back; an explicit one does not.** With no `--port`, a busy 7007 makes the
launcher search upward (up to +50, i.e. 7007–7057) and print
`Port 7007 is in use — starting on 7008 instead.` If you pass `--port` or set `DEVHUB_PORT`, the
launcher refuses to move and exits with an error instead. Either way, read the port off the startup
banner rather than assuming:

```
Starting TIBCO Developer Hub (portable)
  Version: portable-v1.19.0
  URL:     http://localhost:7007
  Data:    /…/devhub-bundled-darwin-arm64/data
```

To find a hub you did not start, probe `/.backstage/health/v1/readiness` across 7007–7057 — it is
the one route that needs no token.

`export GITHUB_TOKEN=ghp_…` before starting and the launcher feeds it into the GitHub integration.
Stop with Ctrl-C. Reset everything by stopping and deleting `<bundle>/data/`.

## Calling the API — guest mode is not anonymous

Every `/api/*` route requires credentials. An unauthenticated call answers **HTTP 500
`AuthenticationError: Missing credentials`** — not 401, which makes it easy to misread as a server
bug. Mint a guest identity token first; it is valid for one hour:

```sh
HUB=http://127.0.0.1:7007
TOKEN=$(curl -s "$HUB/api/auth/guest/refresh" | python3 -c 'import sys,json;print(json.load(sys.stdin)["backstageIdentity"]["token"])')

curl -s -H "Authorization: Bearer $TOKEN" "$HUB/api/catalog/entities?limit=5"
```

Every `/api/*` call in these skills needs that header. Mint the token once per session and reuse it.

The one route that answers **without** a token is `/.backstage/health/v1/readiness`, which is
therefore the right way to check whether the hub is up (and which port it took).

## Registering what you author

The bundle's `app-config.portable.yaml` is always loaded; a file you pass with `--config` is layered
on top of it. **The bundle does not ship such a file — you create it.** These skills call it
`devhub-local.yaml` by convention and assume you start the hub with it:

```bash
./devhub --config /abs/path/devhub-local.yaml
```

Point `catalog.locations` at your content with an **absolute** path:

```yaml
# devhub-local.yaml — yours; keep it out of version control, it holds tokens
catalog:
  locations:
    - type: file
      target: /abs/path/my-devhub-content/templates/my-template/my-template.yaml
```

Then **restart the hub**. Two rules that bite:

- `catalog.locations` **replaces** the array in the portable config; it does not merge. Keep the
  stock example-catalog entry in your overlay if you still want it.
- Relative targets resolve against the backend's cwd (wherever you launched `devhub`), so always
  write absolute paths.

Editing the *content* of an already-registered file does not need a restart — the catalog refresh
loop re-reads it within a couple of minutes, or you can force it from the entity page's Refresh
button. Adding a *new* location always needs a restart.

## Where things appear — tags drive routing

A template, an import flow and a self service flow are all the same entity kind
(`scaffolder.backstage.io/v1beta3`, `kind: Template`). Only `metadata.tags` decides where it shows up:

| Tag | Page | Notes |
|---|---|---|
| `import-flow` | `/import-flow` | suppressed by `devhub-internal` |
| `self-service` | `/self-service-flow` + home page card | excluded from `/create`; suppressed by `devhub-marketplace` |
| `devhub-marketplace` | `/marketplace` | marketplace entries only |
| anything else | `/create` | the default |

`spec.type` is a display chip, never a router.

## The TIBCO custom scaffolder actions

All bundled in portable (verified in the runtime): `tibco:git:clone`, `tibco:extract-parameters`,
`tibco:create-yaml`, `tibco:git:push`, `tibco:call-platform-api`, `tibco:file:write`,
`tibco:fetch-api-file`. The platform-aware form fields `CapabilitySelector` and `DataplaneSelector`
ship in the prebuilt frontend.

The live schema of every installed action is at `http://localhost:7007/create/actions` — read it
there instead of guessing.

**None of the TIBCO actions are dry-run aware.** They attempt real work and fail during a dry-run.
That is expected and is not a broken template.

## Secrets

- **GitHub**: `export GITHUB_TOKEN=ghp_…` before starting the hub — the launcher injects it into the
  GitHub integration. Needed for `publish:github`, `tibco:git:push`, and registering github.com URLs.
  Without it GitHub reads are anonymous and rate-limited to 60/hour.
- **TIBCO Platform** (self service flows only): `cpLink` and `TIBCOPlatformToken` in your `--config`
  file. Keep that file out of version control — never commit a real token, never print one back.

## The scaffolder API — dry-run, run, verify

The bundle ships **no helper scripts**. Drive the scaffolder over REST; every call needs the guest
bearer token from the section above (`$HUB` and `$TOKEN` are assumed set).

**Dry-run** — renders a template without creating anything. The template is uploaded inline, so it
does **not** need to be registered first; that makes this the fast inner loop on portable, where
registering costs a restart.

```sh
# body: { template: <the Template entity>, values: {…}, secrets?: {…},
#         directoryContents: [ { path, base64Content }, … ] }   # skeleton files, base64
curl -s -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' \
     -X POST "$HUB/api/scaffolder/v2/dry-run" -d @dry-run-body.json
```

Returns `{ log, directoryContents, output, steps }` — `directoryContents` is the rendered tree,
base64 again, which you decode to inspect the result.

**Real run** — creates things. Confirm with the user first.

```sh
# 201 -> { id: "<taskId>" }
curl -s -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' \
     -X POST "$HUB/api/scaffolder/v2/tasks" \
     -d '{"templateRef":"template:default/<slug>","values":{…}}'

curl -s -H "Authorization: Bearer $TOKEN" "$HUB/api/scaffolder/v2/tasks/<taskId>"          # status
curl -s -H "Authorization: Bearer $TOKEN" "$HUB/api/scaffolder/v2/tasks/<taskId>/events"   # step logs
```

`templateRef` is resolved from the catalog, so a live run **does** need the template registered.
Task status is one of `open` / `processing` / `completed` / `failed` / `cancelled`; `/cancel` and
`/retry` exist alongside `/events`.

**Verify** what landed in the catalog:

```sh
curl -s -H "Authorization: Bearer $TOKEN" "$HUB/api/catalog/entities/by-name/component/default/<name>"
curl -s -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' \
     -X POST "$HUB/api/catalog/entities/by-refs" -d '{"entityRefs":["component:default/<name>"]}'
```

The catalog refresh loop is not instantaneous — retry for a minute or two before concluding an
entity is missing.

Two portable specifics worth knowing:

- **The request body limit is 10 MB**, baked into the bundle. An HTTP 413 on dry-run means the
  skeleton is too big; it cannot be raised from config the way it can in the open-source repo.
- **`cpToken` is injected for you.** A middleware adds the Control Plane token to `secrets.cpToken`
  on every `POST /v2/tasks`, so self service flows do not need it passed by hand.

## Reference content

There is no `tibco-examples/` folder here. Canonical examples are fetched from GitHub when a skill
needs one — always read a reference before generating output:

| What | URL |
|---|---|
| Software template | `https://raw.githubusercontent.com/TIBCOSoftware/tibco-developer-hub/main/tibco-examples/bwce-bookstore-template/bwce-bookstore-template.yaml` |
| Import flow (simple) | `…/main/tibco-examples/import-flow-v2/import-flow-bwce-v2.yaml` |
| Import flow (advanced) | `…/main/tibco-examples/advanced-import-flows/import-flow-bw6.yaml` |
| Self service flow | `…/main/tibco-examples/developer-hub-marketplace-content/self-service-flows/build-deploy-flogo-app/self-service-build-deploy-flogo-app.yaml` |
| Self service owner group | `…/main/tibco-examples/developer-hub-marketplace-content/self-service-flows/tibco-self-service-group.yaml` |

Browse more with the GitHub tree API:
`https://api.github.com/repos/TIBCOSoftware/tibco-developer-hub/git/trees/main?recursive=1`.

## Agent working rules

- **Never start or stop the hub on the user's behalf** unless asked — it runs in their terminal.
- Bash localhost calls are blocked by the default sandbox; retry those with
  `dangerouslyDisableSandbox: true`. File reads/writes inside the workspace are fine in-sandbox.
- The request body limit is 10 MB and is baked into the bundle — it cannot be raised from config.
  An HTTP 413 on dry-run means the skeleton is too big, not that config is wrong.
- Catalog entity names are global. Prefix generic-sounding names so two samples don't collide.
- Say "TIBCO Developer Hub", not "Backstage", in anything a user reads. Say **BW6**, never "BWCE"
  in new prose (the product was renamed); existing tags and example filenames keep their spelling.

## Workflows

Full runbooks live in `.claude/skills/<name>/SKILL.md`. Claude Code executes them automatically;
other agents can read them as reference.

### create-template

Author a software template under `templates/<slug>/`. Gather slug/title/description/type/owner/tags
/parameters/publish target, read the reference template from GitHub first, write `<slug>.yaml` plus a
`skeleton/` (always including `catalog-info.yaml`), always add a `debug` boolean guarding
publish/register, then add an absolute `type: file` location to your `--config` file and tell the user
to restart. Appears at `http://localhost:7007/create`.

### create-import-flow

Author an import flow under `import-flows/<slug>/` — clone → extract → generate → push → register,
using `tibco:git:clone`, `tibco:extract-parameters`, `tibco:create-yaml` (simple) or
`fetch:template` + `.njk` skeletons (advanced), `tibco:git:push`. Tag `import-flow`. Register in
your `--config` file, restart, verify at `/import-flow`.

### create-self-service-flow

Author a self service flow under `self-service-flows/<slug>/` that drives the TIBCO Platform through
`tibco:call-platform-api`. Tag `self-service` (this is what routes it). Follow check →
provision-if-missing → act, guarding every provisioning step with `if:` so re-runs are idempotent.
Control Plane calls omit `baseUrl`; Data Plane calls pass
`baseUrl: ${{ parameters.deploymentTarget.dataplaneUrl }}`. Needs `cpLink` + `TIBCOPlatformToken` in
your `--config` file.

### test-template

Dry-run a template with `POST /api/scaffolder/v2/dry-run` and decode the returned
`directoryContents` to inspect the rendered tree. No GitHub repo is created.

### test-import-flow

Phase 1 dry-run for structure (TIBCO actions fail as expected), then — after explicit confirmation —
a real run via `POST /api/scaffolder/v2/tasks` against a real repo, verified against the catalog API.

### test-self-service-flow

Phase 1 dry-run, then — after an explicit blast-radius confirmation — a live run that consumes real
Data Plane resources. `CapabilitySelector` never runs over the API, so `deploymentTarget` must be
assembled by hand from `GET /tp-cp-ws/v1/data-planes`. Verify against the platform APIs, not just the
task status; then offer cleanup.

### impact-analysis

Answer "what breaks if I change `<entity>`?" from the live catalog graph at
`http://localhost:7007/api/catalog`, classify neighbours into 🔴/🟠/🟢 tiers, and write a report plus
three color-coded Mermaid topology diagrams under `impact_analysis/`.

## MCP on portable

`MCP-TOOLS.md` in this folder is the tool reference — endpoint, the eleven `catalog.*` /
`scaffolder.*` tools, the predicate query syntax and the gotchas. Two portable-specific differences
from the open-source set:

- **The port may not be 7007.** The portable launcher moves up (7007–7057) if the port is taken, so
  the MCP endpoint is `http://localhost:<actual-port>/api/mcp-actions/v1`. Read the port from the
  startup log or `/.backstage/health/v1/readiness`.
- **Auth is not anonymous.** The REST fallbacks in these skills mint a guest bearer token; an MCP
  client would need the equivalent. If tool calls fail with `AuthenticationError: Missing
  credentials`, that is the cause — the same trap the REST paths document.
- **The on/off switch is `tibco.mcpActions.enabled`.** Set it to `true` in your `--config` file and
  restart to switch the server on; `false` makes `/api/mcp-actions/*` answer
  `404 {"error":"MCP actions is disabled"}`. The *top-level* `mcpActions` key is a different thing —
  it carries the server's `name` and `description` for MCP clients, and has no `enabled` flag.

  ⚠️ In `portable-v1.19.0` as published, the bundled `app-config.portable.yaml` puts `enabled: false`
  under that top-level `mcpActions` key, where nothing reads it, and never sets
  `tibco.mcpActions.enabled`. The effect is that MCP is **reachable by default** on that build,
  despite the config appearing to disable it. (`/api/*` still requires a guest token, so this is not
  an unauthenticated hole.) Probe the endpoint to find out which state you are in rather than
  trusting the config file.

Every skill in this set works without MCP. Each documents a REST path that is fully supported, not a
legacy fallback — so a bundle with MCP switched off costs you terser calls, nothing else.
