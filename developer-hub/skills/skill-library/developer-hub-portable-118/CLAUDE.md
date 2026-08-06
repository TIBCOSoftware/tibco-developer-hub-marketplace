# CLAUDE.md

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
