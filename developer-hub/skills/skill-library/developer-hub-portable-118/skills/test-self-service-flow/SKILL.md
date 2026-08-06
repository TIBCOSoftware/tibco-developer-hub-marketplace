---
name: test-self-service-flow
description: >
  Test a TIBCO Developer Hub self service flow against DevHub Portable in two phases: (1) dry-run
  structure validation via the scaffolder dry-run API — verifies YAML, parameter schema and
  skeleton rendering without touching the TIBCO Platform; (2) a live scaffolder task run against a
  real Control Plane / Data Plane, polled to completion, then verified against the platform APIs
  (build, app, endpoint) and the catalog. Trigger when the user wants to test a self service flow,
  run a build & deploy flow end-to-end, check that an app was actually deployed to a data plane,
  debug a tibco:call-platform-api step, or confirm a newly created self service flow works.
---

# test-self-service-flow

Two-phase end-to-end test for self service flows on DevHub Portable.

**Phase 1 (dry-run)** validates template YAML, the parameter schema and any skeleton rendering. No
platform call succeeds — expected.

**Phase 2 (live run)** runs the flow for real. It consumes Data Plane resources. Treat it as a
deployment, not a rehearsal.

## Key facts

- **Helpers**: `scripts/dry-run.mjs`, `scripts/run-task.mjs`, `scripts/catalog-check.mjs`. They
  auto-discover the port and mint their own guest token. Localhost calls need
  `dangerouslyDisableSandbox: true`.
- **Guest mode still needs a token.** A hand-written curl to `/api/*` without
  `Authorization: Bearer <token from /api/auth/guest/refresh>` answers HTTP 500 `Missing
  credentials`. `/.backstage/health/v1/readiness` is the unauthenticated liveness check. (This is
  the *hub's* auth — unrelated to `TIBCOPlatformToken`, which authenticates the platform calls.)
- **Not dry-run aware**: `tibco:call-platform-api`, `tibco:file:write`, `tibco:fetch-api-file`,
  `tibco:extract-parameters`. They all attempt real execution during a dry-run and fail.
- **`CapabilitySelector` is a frontend field.** The dry-run and tasks APIs take whatever object you
  send — no health filtering runs. You must assemble a complete, correct `deploymentTarget` by hand
  for Phase 2.
- **Phase 2 needs the flow registered** (`devhub-local.yaml` + restart) — the tasks API resolves it by
  ref. Dry-run does not.
- **Portable config**: `cpLink` and `TIBCOPlatformToken` live in `devhub-local.yaml`. Check both
  before Phase 2; report which is missing, never print the value.
- **Timeouts**: build + deploy takes minutes. Poll for at least 600s (`--timeout 600`, the default).

- **Scratch files**: helper scripts, dumps and intermediate JSON go under
  `${TMPDIR:-/tmp}/devhub-skills/test-self-service-flow/` — `mkdir -p` it before the first write and remove it when the run
  finishes. Don't write straight into `/tmp`: concurrent runs collide there, and a
  published skill should not leave loose `.mjs` files in a shared directory.

## Workflow

### 1. Pick the flow

```sh
ls self-service-flows/
```

Select those tagged **`self-service`** — that is what the Self Service page filters on, so it is the
right selector here too. Don't filter on `spec.type`; it is conventional, not required.

### 2. Read the flow and map what it will do

Before running anything, read `self-service-flows/<slug>/<slug>.yaml` and summarise for the user:

- every `tibco:call-platform-api` step: `method`, `endpoint`, and whether it targets the **Control
  Plane** (no `baseUrl`) or the **Data Plane** (`baseUrl` set)
- which steps are `if:`-guarded and under what condition they run
- whether it ends with `publish:github` + `catalog:register`
- the resources it creates: build, app, public endpoint, GitHub repo, catalog entity

This is the blast-radius summary the user needs before approving Phase 2.

### 3. Preflight

```sh
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:7007/.backstage/health/v1/readiness
```

Not running (or on another port — the scripts scan 7007–7016) → tell the user to run
`./scripts/devhub-start.sh`. Don't start it yourself.

Read `devhub-local.yaml` for `cpLink` and `TIBCOPlatformToken`. Missing either → every platform call
in Phase 2 fails with an auth or connection error. Say which is missing; do not invent a token.

### 4. Phase 1 — dry-run

Propose values in one `AskUserQuestion`:

- `app_name`: `dry-run-app`
- `filename`: keep the schema default (`app.json`, `app.ear`)
- `content`: a minimal placeholder — real content is not needed for structure validation
- `deploymentTarget`: a stub object, since the selector does not run:

```json
{
  "dataplaneId": "dp-test",
  "capabilityId": "cap-test",
  "dataplaneUrl": "https://dataplane.invalid",
  "dataplaneHost": "dataplane.invalid",
  "dataplaneName": "Test Data Plane"
}
```

- `repoUrl` (if present): `github.com?owner=test&repo=test-app`

```sh
node scripts/dry-run.mjs --dir self-service-flows/<slug> --values ${TMPDIR:-/tmp}/devhub-skills/test-self-service-flow/values.json
```

Report:

- HTTP 400 with jsonschema errors → the parameter schema rejected your values. This is the main thing
  Phase 1 catches, and it catches it in seconds instead of minutes. Most often `deploymentTarget` was
  sent as a string instead of an object.
- HTTP 400 `Input template is not a template` → YAML or missing `apiVersion`/`kind`/`metadata.name`/
  `spec.steps`.
- Rendered skeleton files, if the flow has one.
- Failures on the four TIBCO actions → **expected**; list them as such.
- Failures on any *other* step → fix before Phase 2.

> The TIBCO platform actions are not dry-run aware and failed as expected. What is validated: the
> template structure, the parameter schema and skeleton rendering. Phase 2 runs the flow for real
> against your Control Plane.

### 5. Phase 2 — live run

Ask explicitly, naming the concrete resources from step 2:

> Phase 2 runs this flow for real against your TIBCO Platform: it will build and deploy an
> application on data plane `<name>`, expose a public endpoint, create a GitHub repository, and
> register a catalog entity. These consume real resources and are not cleaned up automatically.
> Continue?

If the user declines, stop.

#### 5a. Assemble `deploymentTarget` from the Control Plane

The selector does not run over the API, so build the object from live data instead of guessing:

```sh
curl -s -H "Authorization: Bearer $TIBCO_PLATFORM_TOKEN" \
  "$CP_LINK/tp-cp-ws/v1/data-planes" | head -c 2000
```

Take `dataplaneId`, the matching capability instance id (`capabilityId`), the data plane URL
(`dataplaneUrl`) and its host (`dataplaneHost`). Confirm the assembled object with the user.

Also collect the real app name, the real artifact `content` (a working Flogo app JSON or BW5CE `.ear`
— ask for a path and read it), and a `repoUrl` the hub's GitHub token can create.

#### 5b. Run it

```sh
node scripts/run-task.mjs --ref template:default/<slug> --values ${TMPDIR:-/tmp}/devhub-skills/test-self-service-flow/live-values.json --timeout 600
```

The script streams step log lines. The flow's `debug:log` steps print `buildId` and `appId` there —
capture both; they are the inputs to verification.

### 6. Verify against the platform

`completed` only means every step returned 2xx. Confirm the platform actually holds the resources:

```sh
curl -s -H "Authorization: Bearer $TIBCO_PLATFORM_TOKEN" "$DATAPLANE_URL/public/v1/dp/builds/$BUILD_ID"
curl -s -H "Authorization: Bearer $TIBCO_PLATFORM_TOKEN" "$DATAPLANE_URL/public/v1/dp/apps/$APP_ID"
curl -s -H "Authorization: Bearer $TIBCO_PLATFORM_TOKEN" "$DATAPLANE_URL/public/v1/dp/apps/$APP_ID/endpoints"
```

Report app status, replica count and the public endpoint URL. An app in `Pending` /
`CrashLoopBackOff` / `Failed` after a `completed` task is the classic false positive: the deploy API
accepted the request, the container never came up.

### 7. Verify in the catalog

Only if the flow ends with `catalog:register`:

```sh
node scripts/catalog-check.mjs Component:<app_name>
```

### 8. Report and offer cleanup

Summarise: task status · buildId · appId · app state · public endpoint · repository URL · catalog
entity ref.

Then list what the run left behind and **offer** to remove it — never delete without an explicit yes:

```sh
curl -s -X DELETE -H "Authorization: Bearer $TIBCO_PLATFORM_TOKEN" \
  "$DATAPLANE_URL/public/v1/dp/apps/$APP_ID"
```

The GitHub repo and the catalog entity also remain. Mention both; deleting the repo is the user's call.

## Common failure modes

| Symptom | Cause | Fix |
|---------|-------|-----|
| `No DevHub Portable backend reachable` | Hub not running | `./scripts/devhub-start.sh` |
| Phase 1 HTTP 400 jsonschema errors | Values don't satisfy `spec.parameters` | Usually `deploymentTarget` sent as a string |
| Phase 1 platform-action errors | Expected — not dry-run aware | Normal |
| Phase 2 HTTP 400 `no such template` | Flow not registered / hub not restarted | Add the absolute location to `devhub-local.yaml`, restart |
| Phase 2 401/403 on the first call | `TIBCOPlatformToken` missing or expired | Refresh it in `devhub-local.yaml` and restart |
| Phase 2 `ENOTFOUND` / connection refused | `cpLink` wrong, or a DP call missing `baseUrl` | CP calls omit `baseUrl`; DP calls need `deploymentTarget.dataplaneUrl` |
| Phase 2 404 on `public/v1/dp/…` | Called against the Control Plane | Add the `baseUrl` |
| Extraction returns nothing | Response written without `\| dump`, or wrong JSONPath | Pipe `output.data \| dump`; test the JSONPath against the saved file |
| Deploy 404s on `buildId` | Build unfinished when deploy ran | Increase `debug:wait`, or poll build status |
| Task completes but the app never runs | Deploy accepted, container failed | Check app status (step 6) and the app logs in the Control Plane |

## Don't

- Don't run Phase 2 without an explicit confirmation that names the resources it will create.
- Don't run Phase 2 against a production data plane unless the user says so in as many words.
- Don't skip Phase 1 — it catches schema errors in seconds that would surface minutes into a live run.
- Don't treat `status: completed` as proof of success.
- Don't print, log or write `TIBCOPlatformToken` into any file, script or summary.
- Don't start or restart the hub yourself.
- Don't delete deployed apps, repos or catalog entities without asking.
- Don't retry a failed live run blindly — repeated runs leave orphaned builds and apps on the data
  plane.