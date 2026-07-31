# BW6 Skill Library

A library of **skills for AI coding agents** that automate common TIBCO
BusinessWorks 6.x development tasks. Drop these skills into your project and your coding agent
(Claude Code, Codex, Cursor, and others) follows the same proven playbook every time — no drift,
no re-explaining.

The library currently ships **19 skills** across four groups: core skills that teach the agent how
to drive BW6, a catalogued set of runnable sample applications, and eight canonical integration
patterns. It keeps growing.

## The AI Skills approach

The idea is simple: **teach AI agents your team's exact BusinessWorks workflows — once.** Instead
of explaining the same CLI syntax, project conventions, and gotchas to every developer (and every
agent), you encode each workflow as a skill. Any team member then invokes the same skill and the
agent follows the identical, reviewed playbook.

### What is a skill?

A **skill** is a plain-Markdown runbook (`SKILL.md`) that describes, for one repeatable task: the
**trigger** (when the agent should use it), the **key facts**, the **step-by-step workflow**, and
the **failure modes**. The agent reads the skill on demand, asks for any missing inputs, then
executes the whole workflow.

## Skills in this library

### Core skills

The foundation — how the agent talks to BW6, what the words mean, and what "good" looks like.

| Skill | What it does |
|---|---|
| [`bwdesign`](bw6design/SKILL.md) | Drives the headless BusinessWorks Studio CLI to author and manage BW projects without opening the IDE — applications, modules, processes, activities, links, properties, shared resources, bindings, tests, validation, EARs. Includes a [full command reference](bw6design/reference.md) and the real-world gotchas of running the CLI. |
| [`glossary`](glossary/SKILL.md) | Translates plain-English or rival-platform vocabulary ("workflow", "queue listener", "Mule flow", "Camel route", "Lambda") into canonical BW6 nouns and tool names *before* any building starts. Run it first on any vague request. |
| [`bw6-rules`](bw6-rules/SKILL.md) | 62 quality rules and 25 metrics for reviewing a BW6 project — security posture (TLS, hardcoded credentials, binding policies), checkpoint and transaction placement, JDBC/JMS/HTTP misuse, namespace hygiene, naming conventions. See the [rules index](bw6-rules/RULES.md) and [metrics](bw6-rules/METRICS.md). |
| [`bw6-prompt-library`](bw6-prompt-library/SKILL.md) | Browse-only catalog of the sample-application skills below. Ask "what BW6 samples do we have?" and the agent lists them without building anything. |

### Sample application skills

Each one builds a complete, validating BW6 application end to end. They are the fastest way to see
a palette in action — and a good starting point to fork into something of your own.

| Skill | Category | Builds |
|---|---|---|
| [`basicRetailOrderLogger`](basicRetailOrderLogger/SKILL.md) | Basic | Timer → Log → Mapper → subprocess call, with module properties, a property group, and an XSD. The BW6 "hello world". |
| [`fileRetailSalesFile`](fileRetailSalesFile/SKILL.md) | File | Timer-driven sales summary that maps against an XSD and appends a line to an output file, using process properties that shadow module properties. |
| [`jdbcRetailInventoryDb`](jdbcRetailInventoryDb/SKILL.md) | JDBC | Scheduled JDBC Query + JDBC Update against a Postgres retail DB via a shared JDBC Connection resource. |
| [`jmsRetailOrderQueue`](jmsRetailOrderQueue/SKILL.md) | JMS | JMS Receive Message starter with JNDI + JMS Connection shared resources and a Reply to JMS Message — the EMS queue request/reply pattern. |
| [`restRetailProductService`](restRetailProductService/SKILL.md) | REST | REST service with Swagger enabled, exposing `GET /products/{productId}` and `POST /products` backed by an XSD. |
| [`soapRetailLoyaltyService`](soapRetailLoyaltyService/SKILL.md) | SOAP | SOAP service generated *from* a WSDL, auto-creating SOAP Receive and SOAP Reply, with Mapper and Log wired in between. |
| [`subprocessRetailReturns`](subprocessRetailReturns/SKILL.md) | SubProcess | Main process → subprocess handoff via CallProcess, with a validation subprocess that maps and logs return status. |

### Integration pattern skills

Canonical shapes for the integrations teams actually build. Each skill covers the intent, the
typical flow, the design decisions, and the failure modes — then scaffolds it.

| Skill | Pattern |
|---|---|
| [`pattern-rest-to-db-query`](pattern-rest-to-db-query/SKILL.md) | REST GET → parameterized JDBC SELECT → JSON. Read-only, stateless, cacheable. |
| [`pattern-rest-to-db-command`](pattern-rest-to-db-command/SKILL.md) | REST POST/PUT/PATCH/DELETE → transactional JDBC INSERT/UPDATE/DELETE/MERGE. |
| [`pattern-rest-to-ems`](pattern-rest-to-ems/SKILL.md) | REST → canonical schema → EMS publish → HTTP 202 Accepted. Async intake / command bus. |
| [`pattern-source-to-ems-publish`](pattern-source-to-ems-publish/SKILL.md) | DB poll, file watch, or schedule → canonical event → EMS publish. The producer side of pub-sub / CDC. |
| [`pattern-ems-subscribe-to-target`](pattern-ems-subscribe-to-target/SKILL.md) | EMS subscribe → transform → write into a target system (DB, REST, SOAP, file, ERP). The consumer side. |
| [`pattern-rest-to-soap-mediation`](pattern-rest-to-soap-mediation/SKILL.md) | REST/JSON front over a legacy SOAP backend, including SOAP fault translation. |
| [`pattern-soap-to-rest-mediation`](pattern-soap-to-rest-mediation/SKILL.md) | Preserve an existing SOAP contract while the backend moves to REST — the strangler direction. |
| [`pattern-rest-sync-backend-proxy`](pattern-rest-sync-backend-proxy/SKILL.md) | API facade / backend-for-frontend that synchronously proxies to an existing REST or SOAP service. |

## How to use it

Install this entry to add the documentation to your Developer Hub, then copy the skills and the two
agent instruction files from the [BW6 Skill Library source](#source-code) into your own project:

```text
your-bw-project/
├── AGENTS.md            ← the instructions every agent reads
├── CLAUDE.md            ← thin pointer to AGENTS.md, for Claude Code
├── .mcp.json            ← BW Studio MCP server (optional fallback path)
└── .claude/
    └── skills/
        ├── bwdesign/
        ├── glossary/
        ├── bw6-rules/
        ├── bw6-prompt-library/
        └── pattern-*/
```

Skills are discovered from `.claude/skills/` in Claude Code. Other agents look elsewhere — Codex
and Cursor read `AGENTS.md` directly, so the tables in that file are what point them at the right
runbook.

### `AGENTS.md` — the authority

`AGENTS.md` is the single source of truth for how an agent must work with BW6 in your repository.
It is the cross-agent convention: Claude Code, Codex, Cursor and others all read a file by that
name, so writing the rules once covers every tool your team uses.

The shipped `AGENTS.md` already carries the rules that matter most:

- **The hard rule — never directly edit BW source files.** No `Write`, `Edit`, `sed`, or `python`
  against `.bwp`, `module.bwm`, `MANIFEST.MF`, `.substvar`, WSDLs, XSDs, shared resources, or
  tests. A `.bwp` holds four coupled representations of the same process (BPEL model, diagram
  notation, variable descriptors, per-activity XSLT bindings); hand-editing desynchronises them and
  produces invisible activities or files Studio refuses to open.
- **Which interface to use.** `bwdesign` CLI by default; fall back to the BW Studio MCP server only
  when the workspace is locked; if the MCP server is unreachable, stop and ask the user to enable it
  rather than working around it.
- **Environment resolution.** `BW_HOME`, `BW_VERSION`, and the Eclipse workspace path, plus how to
  discover them.
- **The standard build lifecycle** and the **verification rule** — never report a BW app as done
  without a clean `system:validate` run, and quote the output rather than asserting it passed.
- **Skill tables** telling the agent which skill to read for which task.

**What to customise after copying:**

1. Pin your environment in the *Environment* section — the real `BW_HOME`, BW version, and
   workspace path — so the agent stops asking at the start of every session.
2. Set `<REFERENCE_WS>` to a workspace of existing BW projects, if you have one. Several
   workarounds harvest activity type IDs from it.
3. Add your own house rules — naming conventions, mandatory module properties, which config
   profiles exist, review requirements.
4. Trim the skill tables if you only installed some of the skills.

### `CLAUDE.md` — the Claude Code entry point

Claude Code reads `CLAUDE.md` automatically at the start of every session. Rather than duplicating
guidance (two files that drift apart is worse than one), the shipped `CLAUDE.md` is deliberately
thin: it points at `AGENTS.md` as the authority and repeats only the two load-bearing rules —
never edit BW source files directly, and `bwdesign` is the default interface.

Keep it that way. When you add a rule, add it to `AGENTS.md`; promote something into `CLAUDE.md`
only when it is important enough that an agent must see it before reading anything else.

If your project already has a `CLAUDE.md`, merge the pointer in rather than overwriting:

```markdown
All guidance for working with TIBCO BusinessWorks 6 projects lives in **[AGENTS.md](AGENTS.md)**.
Read it before creating or modifying any BW project.
```

### `.mcp.json` — the fallback path

`.mcp.json` registers the BW Studio MCP server as `bw`, so its tools appear to the agent as
`mcp__bw__*`. That name is load-bearing — the `glossary` skill instructs downstream steps to use
`mcp__bw__*` tool names.

The port is a Studio preference and is **not** fixed (8080, 8181 and 8686 have all been observed),
so update the `url` to match yours. Enable the server in Studio under **Window ▸ Preferences ▸
BusinessWorks AI ▸ HTTP MCP Server** and use the unified `/mcp` endpoint. This is only needed for
the fallback path — the `bwdesign` CLI works without Studio running at all.

## Source code

Every skill in this library, along with `AGENTS.md`, `CLAUDE.md`, and `.mcp.json`, lives on GitHub:

**[TIBCOSoftware/tibco-developer-hub-marketplace › businessworks/bw6-skill-library](https://github.com/TIBCOSoftware/tibco-developer-hub-marketplace/tree/main/businessworks/bw6-skill-library)**

To pull just the skills into an existing project:

```bash
git clone --depth 1 https://github.com/TIBCOSoftware/tibco-developer-hub-marketplace.git /tmp/dh-marketplace
cp -R /tmp/dh-marketplace/businessworks/bw6-skill-library/skills/* your-bw-project/.claude/skills/
cp /tmp/dh-marketplace/businessworks/bw6-skill-library/{AGENTS.md,CLAUDE.md,.mcp.json} your-bw-project/
```

Then customise `AGENTS.md` for your environment as described above.
