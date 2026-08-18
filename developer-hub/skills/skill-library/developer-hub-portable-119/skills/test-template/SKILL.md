---
name: test-template
description: >
  Dry-run a Backstage scaffolder template against the running DevHub Portable hub and write the
  rendered files to a scratch folder for inspection. Trigger when the user wants
  to test a template, preview template output, dry-run a template, see what a template generates,
  or debug a ${{ values.* }} substitution — without creating a GitHub repo. Picks the template
  from templates/, prompts for parameter values via AskUserQuestion, then uploads the whole
  template directory to /api/scaffolder/v2/dry-run and unpacks the response.
---

# test-template

Render a template through the scaffolder dry-run API and persist the result under a scratch
folder. Useful for inspecting `${{ values.* }}` substitutions, validating
skeleton output, and iterating without publishing to GitHub each time.

## Key facts

- **MCP first, if it is enabled.** On a 1.19-based portable build the scaffolder is exposed over
  MCP: `scaffolder.dry-run-template` (`{ templateYaml, values, files }` → `{ valid, errors, log,
  steps }` — validation only, **no rendered files**), `scaffolder.execute-template`
  (`{ templateRef, values, secrets }` → `{ taskId }`, side-effecting — confirm with the user first),
  `scaffolder.get-scaffolder-task-logs` (`{ taskId, after }` to tail) and
  `scaffolder.list-scaffolder-tasks`. See `MCP-TOOLS.md`.
  **Check it is actually reachable first** — `curl -s -o /dev/null -w '%{http_code}' -X POST
  http://localhost:<port>/api/mcp-actions/v1`. A `404` means MCP is switched off; turn it on with
  `tibco.mcpActions.enabled: true` in `devhub-local.yaml` and restart. Do **not** probe
  `node_modules` — the portable build compiles the plugin into `index.js`, so
  `ls node_modules/@backstage | grep mcp` finds nothing even when MCP is running. If MCP is off, the
  REST steps below are unchanged and remain the path.

- **No helper script ships with the bundle** — you drive the dry-run over REST.
  `POST /api/scaffolder/v2/dry-run` takes
  `{ template: <the parsed Template entity>, values: {…}, directoryContents: [ { path, base64Content } ] }`
  and returns `{ log, directoryContents, output, steps }`, where `directoryContents` is the rendered
  tree, base64 again. Step 4 has a ready-to-run script.
- **The template does not need to be registered.** Dry-run uploads the entity inline. That is what
  makes it the fast inner loop on portable, where registering means a restart.
- **YAML parsing**: use the bundle's own Python — `<bundle>/.venv/bin/python3` ships PyYAML (it is
  provisioned on first start for TechDocs). Always read the template **off disk**, never from the
  catalog: testing your latest edits is the entire point.
- **Body limit is 10 MB and baked into the bundle.** An HTTP 413 means the skeleton is too big; it
  cannot be raised from config the way it can in the open-source repo.
- **Sandbox**: these are localhost calls, so run them with `dangerouslyDisableSandbox: true`.

- **Scratch files**: helper scripts, dumps and intermediate JSON go under
  `${TMPDIR:-/tmp}/devhub-skills/test-template/` — `mkdir -p` it before the first write and remove it when the run
  finishes. Don't write straight into `/tmp`: concurrent runs collide there, and a
  published skill should not leave loose `.mjs` files in a shared directory.

## Workflow

### 1. Pick the template

```sh
ls templates/
```

One folder → use it. Several → `AskUserQuestion` (single-select, one option per folder). Each folder
holds `<slug>.yaml` and usually a `skeleton/`.

### 2. Introspect parameters and gather values

Read `templates/<slug>/<slug>.yaml`, walk `spec.parameters[*].properties` and `required`, and propose
dry-run defaults in one batched `AskUserQuestion`:

- `name`: `dry-run-<timestamp>`
- `description`: `Dry-run test`
- `owner`: `group:default/test`
- `repoUrl`: `github.com?owner=test&repo=<name>`
- `debug`: **`true`** whenever the template has it — this skips publish/register
- enum / numeric: take the schema `default:`

Show the assembled values and confirm before running.

### 3. Preflight

```sh
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:7007/.backstage/health/v1/readiness
```

Non-200 → the hub is not running, or took a different port (probe 7007–7057). Tell the user
to run `./devhub --config devhub-local.yaml`. **Don't start it yourself** — it runs in their terminal.

Use the readiness route, not a catalog route: `/api/*` without a bearer token answers HTTP 500
`Missing credentials`, which looks like a broken backend but only means "no token". The script below
mints its own guest token.

### 4. Run the dry-run

Write the confirmed values to a scratch JSON file, then write this script beside it and run it:

```sh
S="${TMPDIR:-/tmp}/devhub-skills/test-template"; mkdir -p "$S"

cat > "$S/dry-run.py" <<'PY'
import base64, json, pathlib, sys, urllib.request, yaml

hub, tdir, values_file, outdir = sys.argv[1], pathlib.Path(sys.argv[2]), sys.argv[3], pathlib.Path(sys.argv[4])
body = {
    "template": yaml.safe_load(sorted(tdir.glob("*.yaml"))[0].read_text()),
    "values": json.load(open(values_file)),
    "directoryContents": [
        {"path": str(p.relative_to(tdir)), "base64Content": base64.b64encode(p.read_bytes()).decode()}
        for p in sorted(tdir.rglob("*")) if p.is_file()
    ],
}
tok = json.load(urllib.request.urlopen(f"{hub}/api/auth/guest/refresh"))["backstageIdentity"]["token"]
req = urllib.request.Request(f"{hub}/api/scaffolder/v2/dry-run", data=json.dumps(body).encode(),
    headers={"Authorization": f"Bearer {tok}", "Content-Type": "application/json"})
res = json.load(urllib.request.urlopen(req))
outdir.mkdir(parents=True, exist_ok=True)
for f in res.get("directoryContents", []):
    d = outdir / f["path"]; d.parent.mkdir(parents=True, exist_ok=True)
    d.write_bytes(base64.b64decode(f["base64Content"]))
(outdir / "_log.json").write_text(json.dumps(res.get("log", []), indent=2))
(outdir / "_output.json").write_text(json.dumps(res.get("output", {}), indent=2))
print(f"{len(res.get('directoryContents', []))} files -> {outdir}")
PY

BUNDLE=/abs/path/to/devhub-bundled-<platform>
"$BUNDLE/.venv/bin/python3" "$S/dry-run.py" \
    http://127.0.0.1:7007 templates/<slug> "$S/values.json" "$S/rendered-1"
```

Give each run its own output folder (`rendered-1`, `rendered-2`, …) so prior runs stay available
for `diff -r`.

### 5. Surface the result

Keep it tight:

- Path, file count, and "values applied" evidence from one rendered file — usually the rendered
  `catalog-info.yaml`.
- Anything in `_log.json` worth reading. Entries have shape `{ body: { message, stepId?,
  status? } }` — no top-level `level`. `status: 'skipped'` confirms the `debug` guards worked.
- `_output.json` if non-empty (template links — the `Repository` URL is mocked; no repo exists).

Don't dump whole file contents unless asked.

## Common failure modes

| Symptom | Cause | Fix |
|---|---|---|
| Connection refused on 7007 | Hub not running, or on an unexpected port | `./devhub --config devhub-local.yaml`; or probe 7007–7057 for the real port |
| HTTP 500 `Missing credentials` | Portable's guest mode still requires a bearer token | Mint one from `/api/auth/guest/refresh` and send it as `Authorization: Bearer` |
| HTTP 413 | Skeleton exceeds the bundle's 10 MB limit | Trim the skeleton — the limit is compiled into portable |
| HTTP 400 `Input template is not a template` | YAML parse failed or a required field is missing | Check `apiVersion`, `kind`, `metadata.name`, `spec.steps` |
| HTTP 400 with jsonschema errors | `values` don't satisfy `spec.parameters` | Fix the value — `repoUrl` must be `github.com?owner=X&repo=Y` |
| HTTP 500 `ENOENT: …/skeleton` | Uploaded from inside `skeleton/` | Point the script at the template root, not at `skeleton/` |
| `ModuleNotFoundError: yaml` | Used system Python instead of the bundle's | Run it with `<bundle>/.venv/bin/python3` |
| Step failed on a `tibco:*` action | Those actions are not dry-run aware | Expected — validate structure here, run for real with the matching test skill |

## Don't

- Don't start or restart the hub yourself.
- Don't send a `templateRef` to the dry-run endpoint — it validates a full entity and rejects refs
  with HTTP 400. (The **tasks** endpoint is the opposite: it only takes a ref.)
- Don't upload only `skeleton/` — `fetch:template` resolves `./skeleton` relative to the upload root.
- Don't overwrite previous `dry-run-N` directories; auto-increment so runs stay diffable.
- Don't chase the `Repository` URL in the output — it is a placeholder.
- Don't use `dangerouslyDisableSandbox: true` for anything but the localhost calls.
