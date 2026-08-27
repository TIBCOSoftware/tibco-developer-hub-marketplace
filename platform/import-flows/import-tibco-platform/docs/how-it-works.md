# How It Works

## The steps

| # | Step | What it does |
|---|---|---|
| 1 | Clone The Target Repository | Clones the branch you selected into the task workspace. |
| 2 | Read Data Planes | `GET /cp/api/v1/data-planes` — the data planes on the subscription, plus the Control Plane browser URL used for the dashboard links. |
| 3 | Read All Applications | *Entire control plane only.* `GET /cp/api/v1/apps` — every application on the subscription, as `{ id, name }`. |
| 4 | Read Application Details | *Entire control plane with nesting on.* `GET /cp/api/v1/apps/<app-id>`, once per application. |
| 5 | Read Data Plane Applications | *Select data plane only.* `GET /cp/api/v1/data-planes/<id>/apps` — the applications on the chosen data plane, with full detail. |
| 6 | Generate Catalog Entities | Renders the multi-document `catalog-info.yaml` into the target folder. |
| 7 | Show The Generated Catalog File | Logs the workspace so you can read the generated file in the task output. |
| 8 | Commit And Push The Snapshot | Commits `chore: import TIBCO Platform topology from the Developer Hub` and pushes. |
| 9 | Register The Snapshot In The Catalog | Registers the pushed `catalog-info.yaml` as a catalog location. |

Every platform call is a **read**. The import never writes to your platform.

## Why nesting costs one call per application

The subscription-wide application list (`/cp/api/v1/apps`) returns only `{ id, name }` — it carries
no data plane association, so on its own it cannot tell you where an application runs.

The obvious way to fill that gap would be one `GET /cp/api/v1/data-planes/<id>/apps` call per data
plane: a handful of calls instead of hundreds. It is a trap. That endpoint is not available on every
data plane —

- **Control Tower data planes** return `400 CP-40100`, "App operations are currently not supported
  for Control Tower Data Plane";
- **data planes with no capabilities provisioned** return `403 CP-40100`, "You don't have permission
  to perform this action", no matter how much permission the account actually holds.

A single such data plane in the subscription would fail the whole import.

The per-application endpoint avoids all of it: it carries `data-plane-id`, and it needs only the same
subscription-wide read permission that produced the list in the first place. No per-data-plane
permission is involved, so no individual data plane can break the run. The price is one call per
application — which is exactly what **Nest applications under their data plane** buys, and what
turning it off avoids.

The **Select data plane** scope does use the per-data-plane endpoint, and safely: you picked the data
plane, so a failure is immediate, obvious and limited to that one choice.

## Re-running the import

Re-running with the **same Control Plane Name, repository and folder** is a refresh, not a duplicate:

- Entity names are built from platform IDs, so the same data plane and the same application resolve
  to the same entity on every run.
- The push overwrites `catalog-info.yaml` with the new topology.
- Registration is idempotent. On the first run the location is created; on later runs the catalog
  returns `409 Conflict` because it already exists, and the flow treats that as success — the
  location is already there, and the Hub's refresh loop picks up the updated file and applies the
  changes in place.

What that means in practice:

| On the platform | After a re-import |
|---|---|
| Application renamed | The Component title updates; the entity stays the same. |
| Application deployed to a new version | `tibco.com/app-version` and `tibco.com/app-state` update. |
| New data plane or application | Appears in the tree. |
| Data plane or application deleted | Drops out of the generated file, and the catalog removes the entity on its next refresh. |

Schedule a re-run — or just re-run after significant platform changes — to keep the catalog honest.

## Testing the flow

Running the import as a **dry run** validates the form and the rendering, but not the data: the
`tibco:*` actions that talk to the platform are skipped in a dry run, so the data plane and
application lists come back empty and only the Control Plane Component renders. Use a dry run to
check that the template is well-formed; use a real run to see real topology.

## Limitations

- **One owner for the whole tree.** The platform is not asked who owns each application; every entity
  gets the group you picked on the form. Edit the generated file in Git if you need finer-grained
  ownership.
- **The snapshot is point-in-time.** `app-state` and `app-version` are recorded at import and do not
  track the platform afterwards.
- **One control plane per import.** To reflect several control planes, run the flow once per control
  plane with a different name and a different folder in the repository.
- **GitHub only.** The repository picker is restricted to `github.com`.
- **Applications are Components, not deployments.** The import models what exists and where it runs;
  it does not import logs, metrics or runtime controls. The dashboard links on each entity are the
  route back to the Control Plane for those.
