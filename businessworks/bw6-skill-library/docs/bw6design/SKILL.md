---
name: bw6design
description: Drive the TIBCO BusinessWorks 6.x design-time CLI (bwdesign) to author and manage BW projects from the command line — create/import/export applications and modules, add processes/activities/groups/links, configure module & process properties, shared resources (HTTP/JMS/JNDI), WSDL/SOAP/REST bindings, Swagger operations, tests & assertions, validate, generate pom.xml/diagrams/EARs, and configure MCP. Use whenever the user wants to script or automate BW6 Studio design tasks, or asks how to run bwdesign / a specific bwdesign command.
---

# bwdesign — BusinessWorks 6.x Design-Time CLI

`bwdesign` is the headless command-line front end to TIBCO BusinessWorks Studio (Eclipse-based). It opens (or creates) an Eclipse workspace and runs design-time operations against the BW projects in it.

- **Tool path (typical Windows install):** `<TIBCO_HOME>\bw\<version>\bin\bwdesign.exe` (e.g. `C:\TIBCO\BW6_V100\bw\6.12\bin\bwdesign.exe`)
- **Documented against:** BW 6.12.x (run `edition` to print the edition).
- **Full command catalog:** see [reference.md](reference.md) for syntax/arguments/options of every command.

## Running it (read this first — there are real gotchas)

1. **Run from the `bin` directory.** The wrapper needs `bwdesign.tra` in the current working directory. Running it by absolute path from elsewhere fails with `Failed to open properties file : bwdesign.tra`.
   ```cmd
   cd /d C:\TIBCO\BW6_V100\bw\6.12\bin
   ```

2. **Always pass an isolated `-data <workspace>`** unless you specifically need the user's Studio workspace. Without it bwdesign may use a default Eclipse workspace, and if that workspace is open in the IDE the launch fails with an Eclipse `Workspace is already closed` / OSGi bundle error. A scratch dir avoids the lock:
   ```cmd
   bwdesign.exe -data C:\temp\bwdesign_ws -silent <command>
   ```
   To operate on real projects, point `-data` at the workspace that contains them. If Business Studio is already holding that workspace open, **do not close Studio** — pivot to the MCP fallback below.

## Workspace locked? Use Business Studio's MCP server instead

If Business Studio is running with the target workspace open, `bwdesign` cannot operate on it. Symptoms:

- `Workspace [<path>] is currently in use by another application.` in the batch output
- Eclipse `Workspace is already closed` / OSGi bundle errors on launch
- Confirm with `tasklist | findstr /i TIBCOBusinessStudio` (Windows) or by looking for `<workspace>/.metadata/.lock`.

**Whenever this happens (or whenever Studio is running at all), STOP shelling out to `bwdesign` and use the `mcp__bw__*` MCP tools instead.** Business Studio exposes an HTTP MCP server that runs *inside* the live Studio process, so it operates through the same Eclipse instance holding the lock rather than fighting it — no workspace juggling, no scratch dirs, no "close Studio" ceremony.

Nearly every `bwdesign` command has an MCP equivalent — a partial map:

| bwdesign command | MCP tool |
|---|---|
| `system:createBWApplicationModule` | `mcp__bw__createBWApplicationProject` |
| `system:createProcess` | `mcp__bw__createBWProcess` |
| `system:createActivity` | `mcp__bw__createActivity` |
| `system:linkActivity` | `mcp__bw__createLink` |
| `system:configureActivity` | `mcp__bw__configureActivity` / `mcp__bw__configureActivityWithValue` |
| `system:listActivityTypes` | `mcp__bw__listActivity` |
| `system:validate` | `mcp__bw__getCompilationErrors` |
| `system:moduleProperty` | `mcp__bw__createModuleProperty` / `mcp__bw__updateModulePropertyValue` |
| `system:export -ear` | `mcp__bw__exportToEar` |

Prerequisites: an `.mcp.json` in the workspace root (or in `~/.claude.json`) that registers the server, e.g.:

```json
{ "mcpServers": { "bw": { "type": "http", "url": "http://localhost:8080/mcp" } } }
```

If MCP tools aren't visible in the session, the server isn't registered / Studio isn't running / the port is wrong — fix the config and restart Claude Code so tools re-load.

**Rule of thumb:** default to MCP tools whenever Studio is up; only fall back to `bwdesign` CLI for headless CI / batch scripting where no Studio is running.

3. **Usage form:** `bwdesign [options] command <args>`. With no command it drops into an interactive `bwdesign>` shell.

4. **Batch / non-interactive:** pipe commands on stdin, one per line, ending with `exit`:
   ```cmd
   (echo system:createProcess -p MyModule MyProc com.acme.pkg& echo exit) | bwdesign.exe -data C:\temp\bwdesign_ws -silent
   ```

5. **Strip ANSI color codes** from captured output (the shell emits them even when piped):
   ```powershell
   # Optional in PowerShell when capturing output:
   $output -replace "`e\[[0-9;]*m", ""
   ```

6. **One hung command blocks the whole batch.** When scripting many commands, prefer one invocation per logical step (or small chunks) and wrap with `timeout` so a stall doesn't hang everything:
   ```cmd
   (echo <cmd>& echo exit) | bwdesign.exe -data C:\temp\bwdesign_ws -silent
   ```

7. **`help` syntax gotcha:** use the **bare** command name — `help createProcess`, **not** `help system:createProcess`. In batch mode the `area:command` form returns empty output (and `help area:command` for the `name:name` commands errors with `Area not found`). `help` / `help -all` lists all commands.

## Global options

| Option | Meaning |
|---|---|
| `-data <workspace>` | Eclipse workspace folder (defaults to Studio's current workspace — override it, see above) |
| `-clean` | Clean the Eclipse workspace on launch |
| `-dir <directory>` | Set the current working directory |
| `-log [-a] <file>` | Send (or append, with `-a`) output to a file |
| `-time` | Prefix log messages with timestamps |
| `-debug` | Extra logging |
| `-silent` | Suppress the intro/help banner |

## Command groups (full details in reference.md)

- **Shell/workspace:** `cd`, `pwd`, `ls`, `clear`, `clean`, `edition`, `setedition`, `execute` (run a batch script file), `exit`.
- **Projects & build:** `system:create`, `system:createBWApplicationModule`, `system:delete`, `system:import`, `system:export`, `system:importpreferences`, `system:validate`, `generatepom`, `addDependency`, `copyArtifacts`, `diagram:gen_diagrams`, `generate_manifest_json`.
- **Processes & activities:** `system:createProcess`, `system:createActivity`, `system:renameActivity`, `system:linkActivity`, `system:configureActivity`, `system:configureInputVariable`, `system:configureActivityEditorSchema`, `system:createGroup`, `system:createCatch`, `system:setCallProcessName`, `system:listActivityTypes`, `system:addtag`, `system:configuretag`, `system:bwPackage`, `system:addjdbcparam`.
- **Services/WSDL/bindings:** `system:createWSDL`, `system:setServicePortType`, `system:createBinding`, `system:setSOAPBindingTransportType`, `createSwaggerOperation`, `system:createConversation`, `system:configureConversationKey`.
- **Properties, variables & resources:** `system:moduleProperty`, `system:processProperty`, `system:processVariable`, `system:sharedVariable`, `system:display`, `system:copyModuleProp`, `system:connectResource`, `createHttpConnector`, `createJMSConnection`, `createJNDIConnection`, `system:createApplicationProfile`, `system:renameApplicationProfile`, `system:deleteApplicationProfile`, `export_to_consul`.
- **Testing:** `createTestFolder`, `createTestSuite`, `createTestFile`, `addAssertion`, `deleteAssertion`, `listAllTests`, `mockOutputFile`.
- **MCP (AI):** `generate_mcp_skeleton`, `generate_mcp_config`, `generate_mcp_ear`, `system:setHttpMcpServerEnabled`.

## Typical workflow

```bash
cd C:\TIBCO\BW6_V100\bw\6.12\bin
WS=/path/to/workspace        # contains/will contain the BW projects

# Create an application module with a main process
printf 'system:createBWApplicationModule MyApp MyMainProcess 1.0.0\nexit\n' \
  | ./bwdesign -data "$WS" -silent 2>&1 | sed 's/\x1b\[[0-9;]*m//g'

# Add a process, then validate, then export an EAR
printf 'system:createProcess -p MyApp.module Sub com.acme.pkg\nexit\n' \
  | ./bwdesign -data "$WS" -silent 2>&1 | sed 's/\x1b\[[0-9;]*m//g'
printf 'system:validate MyApp.module\nexit\n' \
  | ./bwdesign -data "$WS" -silent 2>&1 | sed 's/\x1b\[[0-9;]*m//g'
printf 'system:export -ear MyApp -path /tmp/out\nexit\n' \
  | ./bwdesign -data "$WS" -silent 2>&1 | sed 's/\x1b\[[0-9;]*m//g'
```

When unsure of a command's exact arguments, run `help <bareName>` against a scratch workspace, or consult [reference.md](reference.md).
