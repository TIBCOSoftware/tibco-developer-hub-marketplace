# Developer Hub Skills Library

A library of **skills for AI coding agents** that automate the most common TIBCO Developer Hub
tasks — bootstrapping a local environment, authoring Software Templates, Import Flows, and Self
Service Flows, rebranding the portal with a custom theme, and testing it all end-to-end. Drop
these skills into your Developer Hub checkout and your coding agent follows the same proven
playbook every time.

Each skill set bundles three things:

- **The skills** (`SKILL.md` runbooks) covering the full Developer Hub lifecycle — twelve for the
  open-source repo, ten for the portable distribution.
- **`AGENTS.md`** — project documentation in the open AGENTS.md standard, auto-discovered by all
  major coding agents.
- **`CLAUDE.md`** — a thin Claude Code entry point that imports `AGENTS.md` and lists the skills.

## Pick your variant

The library ships **four skill sets**, one per Developer Hub flavour. They differ in how the agent
reaches the Hub, not in what the skills achieve — pick the folder that matches what you run, copy
that one, and ignore the rest.

| Folder | Developer Hub | Backstage | Catalog & scaffolder access | Skills |
|---|---|---|---|---|
| **`developer-hub-118/`** | 1.18, open-source repo | 1.41.1 | REST API | 12 |
| **`developer-hub-119/`** | 1.19, open-source repo | 1.51.0 | **MCP server**, REST fallback | 12 |
| **`developer-hub-portable-118/`** | DevHub Portable, `portable-v1.18.x` | 1.41.x | REST API, token required | 10 |
| **`developer-hub-portable-119/`** | DevHub Portable, `portable-v1.19.0`+ | 1.51.0 | **MCP server**, REST fallback | 10 |

**Not sure which?** Run `node -e "console.log(require('@backstage/plugin-catalog-backend/package.json').version)"`
in your checkout: `3.4.x` → the 118 sets, `3.8.x` → the 119 sets. If you unzipped a bundle and run it
with `./devhub`, you are on portable; if you run `yarn start` in a monorepo, you are on the
open-source repo.

### What changes between 118 and 119

Developer Hub 1.19 ships `@backstage/plugin-mcp-actions-backend`, exposing the catalog and scaffolder
as **eleven MCP tools** (`catalog.query-catalog-entities`, `scaffolder.dry-run-template`,
`scaffolder.execute-template`, …) at `/api/mcp-actions/v1`. The 119 sets use those tools as the
primary path — typed calls and predicate queries instead of hand-assembled REST — and add a shared
`MCP-TOOLS.md` reference.

The MCP server ships **disabled** (`tibco.mcpActions.enabled: false`), so every 119 skill keeps its
REST path. The set works either way; you just get the terser route once you switch it on.

One honest limit: MCP's `scaffolder.dry-run-template` returns validation only, **not** the rendered
file tree. `test-template` therefore uses it as a fast structural gate and still calls the REST
dry-run for the actual output.

### What changes for portable

DevHub Portable is a single-process download — no Docker, no Postgres, no monorepo, no build step —
so two skills are **deliberately absent**:

- **`setup-dev-hub`** — there is nothing to install or build. You unzip and run `./devhub`.
- **`create-theme`** — the frontend is a prebuilt `index.js`. With no `packages/app` source and no
  build step, a theme cannot be applied at all. Rebranding needs the open-source repo.

The remaining ten are adapted rather than copied: the hub may not be on port 7007 (the launcher moves
up to 7016 if it is taken), and **guest mode is not anonymous** — every `/api/*` call needs a bearer
token minted from `/api/auth/guest/refresh`, or it fails with HTTP 500
`AuthenticationError: Missing credentials`.

**Pick the portable set by release tag.** `portable-v1.19.0` (7 Aug 2026) and newer are on the 1.19
line and carry the MCP server — use `developer-hub-portable-119/`. Older `portable-v1.18.x` bundles
are Backstage 1.41.x with no MCP server — use `developer-hub-portable-118/`. `cat <bundle>/.devhub-release`
tells you which you unpacked.

Note that the MCP plugin is **compiled into the portable `index.js`**, not installed under
`node_modules/`, so the `plugin-catalog-backend` version check above is the reliable discriminator —
`ls node_modules/@backstage | grep mcp` finds nothing on portable even when MCP is running.


## What is the TIBCO Developer Hub?

A self-service developer portal built on Backstage.io that centralises your TIBCO services,
templates, and documentation in one place. It scaffolds new BWCE, Flogo, and EMS applications in
minutes via **Software Templates**, automates ingestion of existing repositories via **Import
Flows**, drives the TIBCO Platform directly via **Self Service Flows**, and offers a
**Marketplace** of reusable integration patterns. It runs as a Yarn 4
monorepo (`packages/app` frontend on `:3000`, `packages/backend` on `:7007`) with in-repo
Backstage plugins under the `@internal/*` scope.

Setting it up, extending it, and keeping every contribution consistent normally requires a lot of
tribal knowledge. That is exactly what these skills capture.

## The AI Skills approach

The idea is simple: **teach AI agents your team's exact workflows — once.** Instead of explaining
the same template conventions, config wiring, and test procedure to every developer (and every
agent), you encode each workflow as a skill. Any team member then invokes the same skill and the
agent follows the identical, reviewed playbook — no drift, no re-explaining.

### What is a skill?

A **skill** is a plain-Markdown runbook stored at `.claude/skills/<name>/SKILL.md`. It describes,
for one repeatable task: the **trigger** (when the agent should use it), the **key facts**, the
**step-by-step workflow**, and the **failure modes**. The agent reads the skill on demand, asks
the developer for any missing inputs, then executes the whole workflow — file creation, config
wiring, API calls, and browser verification.

```markdown
---
name: <skill-name>
description: <when the agent should use this skill>
---

# Step-by-step instructions, key facts, command references, and failure modes...
```

### How it works

1. **Developer invokes** the skill — e.g. types `/setup-dev-hub` or `/create-template` in the
   agent chat.
2. **Agent reads** the `SKILL.md` runbook — trigger, key facts, step-by-step workflow.
3. **Agent asks** targeted questions (slug, title, tags, parameters) via UI prompts.
4. **Agent executes** all file creation, YAML generation, config wiring, and browser verification.
5. **Developer gets** a fully wired, tested artefact in minutes — not hours.

## The twelve skills

The library covers the full Developer Hub lifecycle — from a clean checkout, through authoring
and testing templates, import flows, and self service flows, to rebranding the portal, to
deciding whether to reuse or build a service, assessing the blast radius of a change before you
make it, tracing where a data field comes from and where it ends up, and documenting what changed
between two versions of an API.

| Skill | When to use | What it produces |
|-------|-------------|------------------|
| **`setup-dev-hub`** | Setting up for the first time, onboarding a new machine, or getting the app running locally. | A working local environment: config files created from templates, dependencies installed, backend on `:7007` and frontend on `:3000` (`http://localhost:3000/tibco/hub`). |
| **`create-template`** | Adding a new BWCE, Flogo, or any service template to the `/create` page. | `templates/<slug>/<slug>.yaml` (the `Template` entity) plus a `skeleton/` with `catalog-info.yaml`, a `debug` parameter for safe dry-runs, and the catalog registration. |
| **`create-import-flow`** | Ingesting an existing BWCE, BW6, BW5, Flogo, or EMS repository into the catalog. | An import-flow `Template` using the TIBCO custom actions (clone → extract-parameters → create-yaml → push), optionally with Nunjucks entity skeletons, visible at `/import-flow`. |
| **`create-self-service-flow`** | Automating an action on the TIBCO Platform from a form — build & deploy an app to a Data Plane, provision a capability or connector, expose an endpoint. | A `self-service` `Template` built on `tibco:call-platform-api` and the platform-aware `CapabilitySelector` field, with idempotent check → provision → act steps, visible at `/self-service-flow`. |
| **`create-theme`** | Adding a brand theme, rebranding for a customer, or changing the sidebar logo. | `packages/app/src/themes/<slug>ThemeLight.ts` (and a dark variant if requested), registered in `App.tsx`, with the logo swap wired in `Root.tsx` and type-checked. |
| **`test-template`** | Testing a template, previewing output, or debugging `${{ values.* }}` substitutions. | A full rendered file tree under `template-workspace/dry-run-<N>/` via the scaffolder dry-run API — no GitHub repo created. |
| **`test-import-flow`** | End-to-end validation of an import flow: structure check plus real catalog registration. | Phase 1 dry-run validation, then a real scaffolder task run and a catalog-API check confirming the imported entities were registered. |
| **`test-self-service-flow`** | End-to-end validation of a self service flow: structure check, then a real run against your Control Plane. | Phase 1 dry-run validation, then a live scaffolder task, verified against the **platform** APIs (build exists, app running, endpoint public) and the catalog — plus a cleanup offer. |
| **`reuse-or-build`** | Deciding whether an existing service already carries the data you need, or a new one must be built — "where can I get X from?", before scaffolding a new component. | A decision report (✅ Reuse / 🟡 Extend / 🔴 Build) with a field-level coverage matrix and color-coded topology diagrams under `reports/`, grounded in the live catalog read via the catalog REST API. |
| **`impact-analysis`** | Assessing the blast radius before changing a catalog entity — "what breaks if I change this API/Component/Resource?". | A report plus color-coded integration-topology diagrams under `impact_analysis/`, grounded in the live catalog graph read via the catalog REST API. |
| **`data-lineage`** | Tracing where a data field or message comes from and where it ends up — provenance, audit, and governance questions across the whole integration landscape. | A lineage report with a per-hop field table (🟢 carried · 🔵 renamed · 🟡 derived · ⚪ dropped), flow and field-propagation SVG diagrams under `reports/`, and the governance findings — team hand-offs, convention flips, and transformations the catalog cannot verify. |
| **`api-version-diff`** | Publishing a new version of an API — "what changed between 1.18 and 1.19?", generating a changelog or migration guide, or gating CI on breaking changes. | A TechDocs changelog page in the version's folder, wired into its `mkdocs.yaml` nav, with every change classified 🔴 breaking / 🟢 additive / 🔵 note — plus a reusable `apidiff.mjs` that exits `1` on a breaking change so the same command works as a CI gate. |

## Business value

Encoding these workflows as skills turns hours of expert work into minutes of guided automation.

- **Speed** — new template: ~2 hrs → ~10 min; import flow: ~4 hrs → ~15 min; self service flow:
  ~1 day → ~30 min; onboard a developer: days → ~1 hr; theme rebrand: ~4 hrs → ~20 min.
- **Consistency** — every template follows the TIBCO tag and folder conventions, the `debug`
  parameter is always added for safe dry-runs, self service flows always guard their provisioning
  steps so re-runs are idempotent, and catalog entries are always registered correctly.
- **Knowledge retention** — tribal knowledge becomes executable, version-controlled runbooks that
  keep working after the original author has moved on; new team members are productive on day one.
- **Governance** — TIBCO tag conventions are enforced automatically, catalog metadata stays
  consistent, and every workflow is reviewable and improvable in version control.

## Cross-agent compatibility

The skills under `.claude/skills/` are Claude Code-native, and `AGENTS.md` (an open standard
adopted by 20k+ projects) is auto-discovered by all major agents — so the same project knowledge
is available everywhere.

| Agent | Config file discovered | Reads project docs | Runs skills natively |
|-------|------------------------|--------------------|----------------------|
| Claude Code | `CLAUDE.md` → `@AGENTS.md` | yes | yes |
| OpenAI Codex | `AGENTS.md` | yes | yes |
| GitHub Copilot | `AGENTS.md` + `copilot-instructions.md` | yes | — |
| Cursor | `AGENTS.md` + `.cursor/rules/` | yes | — |
| Devin | `AGENTS.md` + `SKILL.md` runbooks | yes | yes |
| Gemini CLI | `AGENTS.md` | yes | — |

## Getting started

The [skill library assets](https://github.com/TIBCOSoftware/tibco-developer-hub-marketplace/tree/main/developer-hub/skills/skill-library)
are in this entry's repository folder. To use them in your Developer Hub checkout:

1. **Pick your variant folder** — see the table above.
2. Copy that folder's `skills/` into your project's **`.claude/skills/`** directory (so each skill
   lives at `.claude/skills/<name>/SKILL.md`). Do not mix skills from two variants.
3. Copy the same folder's **`AGENTS.md`** and **`CLAUDE.md`** to your repository root. If you already
   have these files, merge the **Workflows** / **Skills** sections in rather than overwriting.
   For the 119 variants, copy **`MCP-TOOLS.md`** alongside them — the skills reference it.
4. Open the project in a coding agent (Claude Code, or any agent that reads `AGENTS.md`).
5. Invoke a skill — e.g. `/setup-dev-hub` to bootstrap the environment (open-source variants), or
   `/create-template` to author your first template.

That's it — the agent reads the runbook and drives the task to completion, asking you for any
inputs it needs along the way.
