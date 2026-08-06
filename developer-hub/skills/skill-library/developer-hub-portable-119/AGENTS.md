# AGENTS.md

> ## Status: forward-looking
>
> The DevHub Portable bundle available today is **Backstage 1.41.x**
> (`@backstage/plugin-catalog-backend` 3.4.0) and ships **no** `@backstage/plugin-mcp-actions-backend`
> — the MCP server does not exist in it. This set is prepared for a portable build on the 1.19 line
> (Backstage 1.51.0), where the MCP server would become available exactly as it is in the
> open-source `developer-hub-119` set.
>
> **Until such a bundle ships, use `developer-hub-portable-118`.** Everything here is identical to it
> apart from the MCP guidance, and every skill keeps its REST path — so if you point this set at a
> 1.41.x bundle it still works, it just never finds the MCP server.
>
> When a 1.19-based portable bundle does ship, verify before relying on this set:
> `ls node_modules/@backstage | grep mcp` and check for `tibco.mcpActions` in
> `app-config.portable.yaml`. The tool names, endpoint and query syntax in `MCP-TOOLS.md` come from
> the open-source 1.19 backend and should carry over unchanged.

Project documentation for AI coding agents (Codex, Copilot, Cursor, Devin, …).
Claude Code users: see `CLAUDE.md`, which imports this file and adds the skill list.

## What this is

An **authoring workspace** for TIBCO® Developer Hub content — software templates, import flows and
self service flows — driven against **DevHub Portable**: a self-contained download that runs the
whole Hub (UI + API) in one process, with no Docker, no Postgres, no Yarn monorepo and no build step.

You write YAML here; the portable hub loads it, renders the form, and runs the steps.

The kit lives **inside the portable bundle folder** — `devhub`, `index.js` and `data/` sit right
next to `templates/`. That is the layout this workspace is set up for and the one it is tested in.

```
dev-hub-template-workflow/                 # git repo root
  README.md                                # repo landing page — points in here
  .gitignore                               # ignores everything the bundle download provides
  devhub-bundled-darwin-arm64/             # <- the bundle AND the authoring workspace
    devhub  index.js  node/  node_modules/  data/  .venv/   # runtime, all gitignored
    app-config.portable.yaml                              # stock config, gitignored
    AGENTS.md  CLAUDE.md                   # this documentation
    .claude/skills/                        # the runbooks
    scripts/                               # start / dry-run / run-task / catalog-check
    devhub-local.yaml                      # --config overlay (gitignored — holds tokens)
    devhub-local.template.yaml             # its committed starting point
    templates/<slug>/<slug>.yaml + skeleton/     # software templates    -> /create
    import-flows/<slug>/<slug>.yaml              # import flows          -> /import-flow
    self-service-flows/<slug>/<slug>.yaml        # self service flows    -> /self-service-flow
    template-workspace/dry-run-<N>/              # dry-run output (gitignored)
```

Only what you author is committed; every file the DevHub Portable download provides is ignored, so
the 77 MB `index.js` and the vendored Node runtime never reach GitHub. A fresh clone therefore needs
the portable bundle unpacked over it before the hub can start.

## The portable runtime — what is different from the open-source repo

If you know the `tibco-developer-hub` monorepo, unlearn these:

| | Open-source repo | DevHub Portable (this workspace) |
|---|---|---|
| Start | `yarn start` | `./scripts/devhub-start.sh` |
| Ports | frontend `:3000`, backend `:7007` | **one** port, `7007` by default |
| App URL | `http://localhost:3000/tibco/hub` | `http://localhost:7007/` (no path prefix) |
| API URL | `http://localhost:7007/api/...` | `http://localhost:7007/api/...` (same origin as the UI) |
| Config | `app-config.local.yaml` | `devhub-local.yaml`, layered via `--config` |
| Registering a template | relative path from `packages/backend/` | **absolute** `type: file` target |
| Reload after config change | Backstage CLI hot-restarts | **Ctrl-C and start again** — no watcher |
| Database | Postgres or in-memory SQLite | SQLite files under `<bundle>/data/`, persistent |
| Auth | configurable providers | guest only — but API calls **do** need a token (see below) |
| Frontend changes (themes, plugins) | possible | **impossible** — the frontend is prebuilt |

Everything else — the scaffolder, the TIBCO custom actions, the catalog, tag-based routing — is
the same product and behaves identically.

## Running the hub

```bash
./scripts/devhub-start.sh              # starts on http://localhost:7007
./scripts/devhub-start.sh --port 8088  # flags pass through to the launcher
```

The script resolves the bundle from `$DEVHUB_HOME`, then `./devhub` next to this folder, then
`.devhub-home` (a gitignored one-line path). It passes `devhub-local.yaml` as `--config`, creating
it from `devhub-local.template.yaml` on first run.

**If port 7007 is busy the launcher silently moves to the next free port and prints it.** Read the
startup line `Listening on 127.0.0.1:<port>` rather than assuming. The helper scripts probe
7007–7016 automatically; override with `DEVHUB_URL`.

Stop with Ctrl-C. Reset everything by stopping and deleting `<bundle>/data/`.

Run `npm install` once (a single dependency, `yaml`). Without it the dry-run helper cannot read your
edits off disk and falls back to whatever version of the template the catalog last ingested.

## Calling the API — guest mode is not anonymous

Every `/api/*` route requires credentials. An unauthenticated call answers **HTTP 500
`AuthenticationError: Missing credentials`** — not 401, which makes it easy to misread as a server
bug. Mint a guest identity token first; it is valid for one hour:

```sh
HUB=http://127.0.0.1:7007
TOKEN=$(curl -s "$HUB/api/auth/guest/refresh" | python3 -c 'import sys,json;print(json.load(sys.stdin)["backstageIdentity"]["token"])')

curl -s -H "Authorization: Bearer $TOKEN" "$HUB/api/catalog/entities?limit=5"
```

The helper scripts in `scripts/` do this for you — only hand-written curls need the dance.

The one route that answers **without** a token is `/.backstage/health/v1/readiness`, which is
therefore the right way to check whether the hub is up (and which port it took).

## Registering what you author

`devhub-local.yaml` is layered on top of the bundle's `app-config.portable.yaml`. To make a template
visible, add it under `catalog.locations` with an **absolute** path:

```yaml
catalog:
  locations:
    - type: file
      target: /Users/you/…/dev-hub-template-workflow/templates/my-template/my-template.yaml
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
- **TIBCO Platform** (self service flows only): `cpLink` and `TIBCOPlatformToken` in
  `devhub-local.yaml`. That file is gitignored — never commit a real token, never print one back.

## Helper scripts

| Script | Purpose |
|---|---|
| `./scripts/devhub-start.sh` | Start the portable hub with this workspace's overlay |
| `node scripts/dry-run.mjs --dir <dir> --values <json>` | Dry-run a template; writes `template-workspace/dry-run-<N>/` |
| `node scripts/run-task.mjs --ref template:default/<slug> --values <json>` | Submit a **real** scaffolder task and poll it, streaming step logs |
| `node scripts/catalog-check.mjs <Kind>:<name> …` | Confirm entities landed in the catalog, retrying through refresh lag |

All of them auto-discover the hub's port (`DEVHUB_URL` overrides) and mint their own guest token.

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
publish/register, then add an absolute `type: file` location to `devhub-local.yaml` and tell the user
to restart. Appears at `http://localhost:7007/create`.

### create-import-flow

Author an import flow under `import-flows/<slug>/` — clone → extract → generate → push → register,
using `tibco:git:clone`, `tibco:extract-parameters`, `tibco:create-yaml` (simple) or
`fetch:template` + `.njk` skeletons (advanced), `tibco:git:push`. Tag `import-flow`. Register in
`devhub-local.yaml`, restart, verify at `/import-flow`.

### create-self-service-flow

Author a self service flow under `self-service-flows/<slug>/` that drives the TIBCO Platform through
`tibco:call-platform-api`. Tag `self-service` (this is what routes it). Follow check →
provision-if-missing → act, guarding every provisioning step with `if:` so re-runs are idempotent.
Control Plane calls omit `baseUrl`; Data Plane calls pass
`baseUrl: ${{ parameters.deploymentTarget.dataplaneUrl }}`. Needs `cpLink` + `TIBCOPlatformToken` in
`devhub-local.yaml`.

### test-template

Dry-run a template with `node scripts/dry-run.mjs` and inspect the rendered tree under
`template-workspace/dry-run-<N>/`. No GitHub repo is created.

### test-import-flow

Phase 1 dry-run for structure (TIBCO actions fail as expected), then — after explicit confirmation —
a real run with `scripts/run-task.mjs` against a real repo, verified with `scripts/catalog-check.mjs`.

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

- **The port may not be 7007.** The portable launcher moves up (7007–7016) if the port is taken, so
  the MCP endpoint is `http://localhost:<actual-port>/api/mcp-actions/v1`. Read the port from the
  startup log or `/.backstage/health/v1/readiness`.
- **Auth is not anonymous.** The REST fallbacks in these skills mint a guest bearer token; an MCP
  client would need the equivalent. If tool calls fail with `AuthenticationError: Missing
  credentials`, that is the cause — the same trap the REST paths document.

Every skill in this set works without MCP. The REST path is not a legacy fallback here; on the
current portable bundle it is the only path.
