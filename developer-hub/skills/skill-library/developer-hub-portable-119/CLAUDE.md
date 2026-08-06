# CLAUDE.md

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

The last three in the table are not shipped inside the bundle download; they are added by this
marketplace entry. They work unmodified against the portable hub — `lineage.py` needs only the
standard library, and `apidiff.mjs` runs on the bundled `./node/bin/node` (JSON specs directly;
YAML needs one PyYAML conversion step, which the skill documents).

### House rules

- Localhost calls from Bash need `dangerouslyDisableSandbox: true`. Nothing else does.
- Don't start, restart or kill the hub yourself — print the command and let the user run it.
- Before a live run (`scripts/run-task.mjs`), state exactly what it will create and get a yes.
- Prefer the helper scripts in `scripts/` over pasting ad-hoc fetch code into `/tmp`.

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
