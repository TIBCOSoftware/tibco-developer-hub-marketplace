# CLAUDE.md

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

This file provides guidance to Claude Code (claude.ai/code) when working in this repository.
Project documentation shared with all AI agents lives in `AGENTS.md` and is imported below.

@AGENTS.md

## Claude Code-specific

### Skills

Ten skills live under `.claude/skills/` — invoke them with `/skill-name`:

| Skill | Purpose |
|-------|---------|
| `create-template` | Author a software template (`/create`) |
| `create-import-flow` | Author an import flow (`/import-flow`) |
| `create-self-service-flow` | Author a self service flow driving the TIBCO Platform APIs (`/self-service-flow`) |
| `test-template` | Dry-run a template and inspect the rendered output |
| `test-import-flow` | Validate an import flow (dry-run + live run + catalog verification) |
| `test-self-service-flow` | Validate a self service flow (dry-run + live platform run + verification) |
| `impact-analysis` | Assess the blast radius of a catalog entity via the catalog REST API |
| `reuse-or-build` | Decide whether an existing service already carries the data you need |
| `data-lineage` | Trace where a field or message comes from and where it ends up |
| `api-version-diff` | Diff two versions of an API specification and publish the differences as TechDocs |

Two skills from the open-source set are deliberately **absent**, because they cannot work here:

- **`setup-dev-hub`** — portable needs no install or build. You unzip and run `./devhub`; see the
  "Running the hub" section.
- **`create-theme`** — the portable frontend is a prebuilt `index.js`. There is no `packages/app`
  source to edit and no build step to run, so a theme cannot be applied. Use the open-source repo
  (the `developer-hub-118` / `developer-hub-119` skill sets) for rebranding.

All ten ship inside the bundle download, under `.claude/skills/`, together with their helper
files — `lineage.py` needs only the standard library, and `apidiff.mjs` runs on the bundled
`./node/bin/node` (JSON specs directly; YAML needs one PyYAML conversion step, which the skill
documents).

### House rules

- Localhost calls from Bash need `dangerouslyDisableSandbox: true`. Nothing else does.
- Don't start, restart or kill the hub yourself — print the command and let the user run it.
- Before a live run (`POST /api/scaffolder/v2/tasks`), state exactly what it will create and get a yes.
- The bundle ships no helper scripts. Drive the hub over its REST API, and keep any scratch script
  you need under `${TMPDIR:-/tmp}/devhub-skills/<skill>/` rather than loose in `/tmp`.

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
