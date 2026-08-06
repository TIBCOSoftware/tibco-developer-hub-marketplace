---
name: test-import-flow
description: >
  Test an import flow against DevHub Portable in two phases: (1) dry-run structure validation to
  verify YAML syntax and Nunjucks skeleton rendering without any real GitHub activity; (2) a live
  scaffolder task run against a real GitHub repository, polled to completion, then verified via the
  catalog API that the imported entities were actually registered in the Developer Hub. Trigger
  when the user wants to test an import flow, verify a catalog import, run an import flow
  end-to-end, check that components were registered after an import, or confirm a newly created
  import flow works.
---

# test-import-flow

Two-phase end-to-end test for import flows on DevHub Portable.

**Phase 1 (dry-run)** validates template YAML and Nunjucks skeleton rendering. No GitHub activity.
The TIBCO actions fail here — they are not dry-run aware. That is expected, not a broken template.

**Phase 2 (live run + catalog verification)** submits a real task against a real repo, polls it, then
confirms the imported entities landed in the catalog.

## Key facts

- **Helpers**: `scripts/dry-run.mjs` (Phase 1), `scripts/run-task.mjs` (Phase 2),
  `scripts/catalog-check.mjs` (verification). All auto-discover the hub's port, mint their own guest
  token, and need `dangerouslyDisableSandbox: true` because they call localhost.
- **Guest mode still needs a token.** A hand-written curl to `/api/*` without
  `Authorization: Bearer <token from /api/auth/guest/refresh>` answers HTTP 500 `Missing
  credentials`. Check liveness with `/.backstage/health/v1/readiness`, which needs no token.
- **Registration matters for Phase 2 only.** Dry-run uploads the entity inline; the tasks API
  resolves `templateRef: template:default/<slug>` from the catalog, so the flow must be registered in
  `devhub-local.yaml` and the hub restarted before a live run.
- **GitHub token**: `tibco:git:clone` on a private repo and `tibco:git:push` both need
  `GITHUB_TOKEN` exported before the hub was started. Anonymous access is read-only and rate-limited.
- **Not dry-run aware**: `tibco:git:clone`, `tibco:extract-parameters`, `tibco:create-yaml`,
  `tibco:git:push`.

- **Scratch files**: helper scripts, dumps and intermediate JSON go under
  `${TMPDIR:-/tmp}/devhub-skills/test-import-flow/` — `mkdir -p` it before the first write and remove it when the run
  finishes. Don't write straight into `/tmp`: concurrent runs collide there, and a
  published skill should not leave loose `.mjs` files in a shared directory.

## Workflow

### 1. Pick the import flow

```sh
ls import-flows/
```

Read each `<slug>.yaml` and select those tagged `import-flow`. One match → use it; several → ask.
None → point the user at `/create-import-flow`.

### 2. Phase 1 — dry-run

Read the flow's `spec.parameters` and propose safe values in one `AskUserQuestion`:

- `repoUrl`: `github.com?owner=test&repo=test-app` (fake — the clone will fail, as expected)
- `application`: `test-application`
- `application_folder`: `test-folder` (if present)
- `system`: `test-system` (if present)
- `owner`: `group:default/test`

Preflight the hub, write the values to a scratch JSON, then:

```sh
node scripts/dry-run.mjs --dir import-flows/<slug> --values ${TMPDIR:-/tmp}/devhub-skills/test-import-flow/values.json
```

Report:

- rendered `.njk` output (advanced flows) — proof that Nunjucks substitution works
- `clone` / `extract` / `push` errors — list them explicitly as **expected**
- any *other* failing step — a real problem
- HTTP 400 `Input template is not a template` → YAML structure problem

> Steps `clone`, `extract` and `push` are not dry-run aware and failed as expected. The template
> structure and the Nunjucks skeleton rendering are validated. Phase 2 runs it for real.

### 3. Phase 2 — live run

Confirm first, explicitly:

> Phase 2 submits a real scaffolder task. It will clone the GitHub repo you name, extract metadata
> from its source files, generate catalog YAML, commit it back (creating a PR), and register entities
> in the catalog. Continue?

If the user declines, stop.

#### 3a. Preconditions

- The flow is registered in `devhub-local.yaml` and the hub has been restarted since
  (`node scripts/catalog-check.mjs Template:<slug>` — a miss means it is not registered).
- `GITHUB_TOKEN` was exported before the hub started, if the flow pushes.

#### 3b. Gather real values

Ask via `AskUserQuestion`:

- `repoUrl`: a **real** repo (`github.com?owner=<org>&repo=<name>`) the token can read and push to
- `application`: the real project folder name as it exists in the repo
- `application_folder`, `system`: if the flow has them
- `owner`: e.g. `group:default/my-team`

Explain what the flow will look for in the repo (from its `extractParameters` file paths) so the user
can pick a repo that matches.

#### 3c. Run it

```sh
node scripts/run-task.mjs --ref template:default/<slug> --values ${TMPDIR:-/tmp}/devhub-skills/test-import-flow/live-values.json --timeout 300
```

The script streams step log lines as they arrive and exits non-zero on failure.

| Failure | Likely cause | Fix |
|---------|-------------|-----|
| `clone` fails | Repo missing, or no token for a private repo | Export `GITHUB_TOKEN` and restart the hub; check the URL |
| `extract` fails | File path expression resolves to nothing | Verify `application` / `application_folder` against the real repo layout |
| `push` fails | Token lacks write access | Use a token with `repo` scope |
| `register` fails | Generated file URL points at the wrong branch or path | Check the push landed on `main` and the `catalogInfoUrl` matches |

### 4. Catalog verification

The entity name comes from extraction, so read it out of the task log rather than assuming:

```sh
node scripts/catalog-check.mjs Component:<extracted-name>
# advanced flows register more than one kind:
node scripts/catalog-check.mjs System:<system> Component:<name> API:<api>
```

The script retries three times with a 5s gap to ride out catalog refresh lag, and prints kind, name,
type, owner, tags and the entity's URL on this hub.

Report: one line per registered entity, plus "✓ Import flow completed — N entities registered". If one
is missing, name the `catalog:register` step to check and the exact URL it used.

## Common failure modes

| Symptom | Cause | Fix |
|---------|-------|-----|
| `No DevHub Portable backend reachable` | Hub not running | `./scripts/devhub-start.sh` |
| Phase 1 HTTP 413 | Skeleton over the bundle's 10 MB limit | Trim it — not configurable in portable |
| Phase 1 clone/extract/push errors | Expected — not dry-run aware | Normal |
| Phase 2 HTTP 400 `no such template` | Flow not registered, or the hub was not restarted | Add the absolute location to `devhub-local.yaml`, restart |
| Catalog empty after the task completes | Refresh lag | The check script already retries; give it a minute |
| Catalog still empty after retries | `catalog:register` URL wrong | Read the register step's URL in the task log |

## Don't

- Don't start or restart the hub yourself.
- Don't skip the confirmation before Phase 2 — it creates real GitHub content.
- Don't verify by looking for the GitHub PR; catalog entity presence is the success criterion.
- Don't overwrite prior `dry-run-N` directories.
- Don't hardcode the entity name for verification — it comes from source-file extraction.
- Don't retry a failed live run blindly; read the step log, fix the flow, then re-run.
