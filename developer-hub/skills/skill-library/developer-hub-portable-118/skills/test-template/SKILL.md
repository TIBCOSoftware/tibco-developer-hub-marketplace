---
name: test-template
description: >
  Dry-run a Backstage scaffolder template against the running DevHub Portable hub and write the
  rendered files into template-workspace/dry-run-<N>/ for inspection. Trigger when the user wants
  to test a template, preview template output, dry-run a template, see what a template generates,
  or debug a ${{ values.* }} substitution — without creating a GitHub repo. Picks the template
  from templates/, prompts for parameter values via AskUserQuestion, and runs
  scripts/dry-run.mjs, which uploads the whole template directory to /api/scaffolder/v2/dry-run
  and unpacks the response.
---

# test-template

Render a template through the scaffolder dry-run API and persist the result under
`template-workspace/dry-run-<N>/`. Useful for inspecting `${{ values.* }}` substitutions, validating
skeleton output, and iterating without publishing to GitHub each time.

## Key facts

- **The helper does the work**: `node scripts/dry-run.mjs --dir <template dir> --values <values.json>`.
  It discovers the hub's port, loads the Template entity, base64s the whole template directory, gzips
  the payload, POSTs to `/api/scaffolder/v2/dry-run`, and writes the rendered tree plus
  `_dry-run-log.json` / `_dry-run-output.json` into the next free `dry-run-<N>` folder.
- **The template does not need to be registered.** Dry-run uploads the entity inline. That is what
  makes it the fast inner loop on portable, where registering means a restart.
- **YAML parsing**: the script uses the `yaml` package if `npm install` has been run; otherwise it
  falls back to fetching the registered entity from the catalog. If it reports
  `loaded from catalog`, your latest on-disk edits may not be what was tested — run `npm install`.
- **Body limit is 10 MB and baked into the bundle.** An HTTP 413 means the skeleton is too big; it
  cannot be raised from config the way it can in the open-source repo.
- **Sandbox**: the script makes localhost calls, so run it with `dangerouslyDisableSandbox: true`.

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

Non-200 → the hub is not running, or took a different port (the script scans 7007–7016). Tell the user
to run `./scripts/devhub-start.sh`. **Don't start it yourself** — it runs in their terminal.

Use the readiness route, not a catalog route: `/api/*` without a bearer token answers HTTP 500
`Missing credentials`, which looks like a broken backend but only means "no token". The scripts mint
a guest token themselves.

### 4. Run the dry-run

Write the confirmed values to a scratch JSON file, then:

```sh
node scripts/dry-run.mjs --dir templates/<slug> --values ${TMPDIR:-/tmp}/devhub-skills/test-template/values.json
```

The output directory auto-increments, so prior runs stay available for `diff -r`.

### 5. Surface the result

Keep it tight:

- Path, file count, and "values applied" evidence from one rendered file — usually the rendered
  `catalog-info.yaml`.
- Anything in `_dry-run-log.json` worth reading. Entries have shape `{ body: { message, stepId?,
  status? } }` — no top-level `level`. `status: 'skipped'` confirms the `debug` guards worked.
- `_dry-run-output.json` if non-empty (template links — the `Repository` URL is mocked; no repo exists).

Don't dump whole file contents unless asked.

## Common failure modes

| Symptom | Cause | Fix |
|---|---|---|
| `No DevHub Portable backend reachable` | Hub not running, or on an unexpected port | `./scripts/devhub-start.sh`; or `DEVHUB_URL=http://127.0.0.1:<port>` |
| HTTP 500 `Missing credentials` on a hand-written curl | Portable's guest mode still requires a bearer token | Mint one from `/api/auth/guest/refresh`, or just use the scripts |
| HTTP 413 | Skeleton exceeds the bundle's 10 MB limit | Trim the skeleton — the limit is compiled into portable |
| HTTP 400 `Input template is not a template` | YAML parse failed or a required field is missing | Check `apiVersion`, `kind`, `metadata.name`, `spec.steps` |
| HTTP 400 with jsonschema errors | `values` don't satisfy `spec.parameters` | Fix the value — `repoUrl` must be `github.com?owner=X&repo=Y` |
| HTTP 500 `ENOENT: …/skeleton` | Uploaded from inside `skeleton/` | The script walks from the template root; check `--dir` |
| `loaded from catalog` warning + stale output | No `yaml` package installed | `npm install` in the workspace |
| Step failed on a `tibco:*` action | Those actions are not dry-run aware | Expected — validate structure here, run for real with the matching test skill |

## Don't

- Don't start or restart the hub yourself.
- Don't send a `templateRef` to the dry-run endpoint — it validates a full entity and rejects refs
  with HTTP 400. (The **tasks** endpoint is the opposite: it only takes a ref.)
- Don't upload only `skeleton/` — `fetch:template` resolves `./skeleton` relative to the upload root.
- Don't overwrite previous `dry-run-N` directories; auto-increment so runs stay diffable.
- Don't chase the `Repository` URL in the output — it is a placeholder.
- Don't use `dangerouslyDisableSandbox: true` for anything but the localhost calls.
