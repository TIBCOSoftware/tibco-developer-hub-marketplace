# What Gets Created

Everything the flow imports is a **Component**. The three levels are told apart by `spec.type`
(`control-plane`, `data-plane`, `application`) and tied together with `spec.subcomponentOf`, which is
what the Developer Hub renders as a hierarchy.

Using Components throughout — rather than a System with Resources, say — means the whole tree shows
up under one **Type** facet in the catalog, and the standard "sub-components" table on an entity page
does the navigation for free.

## The entities

| Level | Entity name | `spec.type` | Parent |
|---|---|---|---|
| Control Plane | `cp-<slug of the name you entered>` | `control-plane` | — |
| Data Plane | `dp-<data plane id>` | `data-plane` | the Control Plane |
| Application | `app-<application id>` | `application` | its Data Plane (or the Control Plane) |
| Unassigned bucket | `cp-<slug>-unassigned` | `data-plane` | the Control Plane |

Names are built from **platform IDs**, not from display names. That is what makes a re-import update
the same entities instead of creating a second copy when an application or data plane is renamed on
the platform.

### Control Plane

```yaml
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: cp-tibco-control-plane
  title: "TIBCO Control Plane"
  description: TIBCO Platform Control Plane, imported into the Developer Hub
  tags: [tibco, control-plane]
  annotations:
    tibco.com/imported-by: import-tibco-platform
  links:
    - url: https://<your-control-plane>/
      title: Control Plane Dashboard
      icon: dashboard
spec:
  type: control-plane
  lifecycle: production
  owner: group:default/tibco-self-service
```

The root of the tree. Its entity page carries a **Control Plane Dashboard** link straight back to the
platform UI, and lists the data planes as sub-components.

### Data Plane

```yaml
metadata:
  name: dp-<data-plane-id>
  title: "eu-prod"
  description: "Data plane eu-prod on TIBCO Control Plane"
  tags: [tibco, data-plane]
  annotations:
    tibco.com/data-plane-id: <data-plane-id>
    tibco.com/imported-by: import-tibco-platform
  links:
    - url: https://<your-control-plane>/cp/app/data-plane?dp_id=<data-plane-id>
      title: Data Plane Details
      icon: dashboard
spec:
  type: data-plane
  lifecycle: production
  owner: group:default/tibco-self-service
  subcomponentOf: cp-tibco-control-plane
```

The **Data Plane Details** link is deep-linked to that specific data plane in the Control Plane UI,
so the Hub entity page is a usable jumping-off point rather than just a record.

### Application

```yaml
metadata:
  name: app-<application-id>
  title: "order-api"
  description: "Handles inbound orders"        # only when the platform reports one
  tags: [tibco, application]
  annotations:
    tibco.com/app-id: <application-id>
    tibco.com/data-plane-id: <data-plane-id>
    tibco.com/capability: <capability id>       # only when reported
    tibco.com/app-state: RUNNING                # only when reported
    tibco.com/app-version: "1.4.0"              # only when reported
    tibco.com/imported-by: import-tibco-platform
spec:
  type: application
  lifecycle: production
  owner: group:default/tibco-self-service
  subcomponentOf: dp-<data-plane-id>
```

### Unassigned applications

Only rendered when it would not be empty. The platform's subscription-wide application list is the
safety net for the import: any application in that list that no data plane claimed is placed in a
bucket Component rather than dropped, so an import never silently loses an application.

```yaml
metadata:
  name: cp-tibco-control-plane-unassigned
  title: Unassigned applications
  description: "Applications on TIBCO Control Plane that no data plane reported"
  tags: [tibco, data-plane]
spec:
  type: data-plane
  subcomponentOf: cp-tibco-control-plane
```

Applications land here when the platform reports them for the subscription but does not associate
them with a data plane the import rendered — typically an application whose data plane was
deleted, or one on a data plane the account cannot see. A bucket that keeps growing between imports
is worth investigating on the platform side.

## Annotations

| Annotation | On | Meaning |
|---|---|---|
| `tibco.com/imported-by` | every entity | Always `import-tibco-platform`. Marks the entity as machine-generated — useful to filter on before bulk-editing or cleaning up. |
| `tibco.com/data-plane-id` | data planes, nested applications | The platform's data plane id. |
| `tibco.com/app-id` | applications | The platform's application id. |
| `tibco.com/capability` | applications | The capability instance the application belongs to (BW6, Flogo, …). |
| `tibco.com/app-state` | applications | The application state at import time, e.g. `RUNNING`. |
| `tibco.com/app-version` | applications | The deployed application version at import time. |

!!! note "State and version are a snapshot, not a live feed"
    `app-state` and `app-version` record what the platform reported **when the import ran**. They do
    not update on their own — re-run the flow to refresh them. Treat the Control Plane UI as the
    source of truth for live status.

Which annotations an application gets depends on how it was imported:

| Import | `app-id` | `data-plane-id` | `capability` | `app-state` | `app-version` |
|---|---|---|---|---|---|
| Select data plane | yes | yes | yes | yes | yes |
| Entire control plane, nesting **on** | yes | yes | yes | yes | yes |
| Entire control plane, nesting **off** | yes | — | — | — | — |
| Unassigned bucket | yes | — | — | — | — |

## How it appears in the Developer Hub

**Catalog list** — all imported entities are Components. Filter by **Type** for `control-plane`,
`data-plane` or `application`, or by the `tibco` tag for everything the import produced.

**Entity page** — each Component shows its parent under *Relations* and its children in the
sub-components table, so you can walk Control Plane → Data Plane → Application without leaving the
Hub. Annotations appear in the entity metadata, and the dashboard links jump back to the platform.

**Diagram** — the Control Plane's *Diagram* tab renders the whole tree as a graph, which is the
quickest way to see how applications are spread across data planes.

**Search** — applications are searchable by name, so an operator who only knows an application's name
can find which data plane and control plane it belongs to.

**Ownership** — every entity is owned by the group you picked, so it appears in that group's page and
in "owned by me" filters alongside the rest of their software.
