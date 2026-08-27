# Import Options

The form has three sections: **Control Plane**, **What To Import** and **Repository Location**.

---

## Control Plane

### Control Plane Name

*Required. Text, max 60 characters. Default: `TIBCO Control Plane`.*

The display name of the Control Plane. It is used twice:

- as the **title** of the Control Plane Component, shown in the catalog and on the entity page;
- lower-cased and with spaces replaced by hyphens, as the **catalog name** of that Component.

`TIBCO Control Plane` becomes the entity `component:default/cp-tibco-control-plane`.

!!! note "This name is the identity of the import"
    The catalog name is derived from this field, so changing it on a later run creates a *second*
    Control Plane tree rather than renaming the first. Pick a name you can live with — for example
    `EMEA Production` if you import more than one control plane into the same Hub — and keep it
    stable. Keep it to letters, digits, spaces and hyphens; other punctuation survives the slug and
    can produce an invalid entity name.

### Owner

*Required. Group picker. Default: `group:default/tibco-self-service`.*

The group that will own **every** entity this flow creates — the Control Plane, the Data Planes and
all applications. Ownership drives the "owned by" filters in the catalog and the entity's Owner tab,
so pick the team that is accountable for the platform rather than the person running the import.

The flow does not derive per-application ownership from the platform; if different teams own
different applications, adjust the generated `catalog-info.yaml` in the repository after the import,
or run the flow once per data plane with a different owner each time.

---

## What To Import

### Scope

*Required. Choice of `Entire control plane` or `Select data plane`. Default: `Entire control plane`.*

This is the main decision on the form, and it changes both what is imported and which platform
endpoints are used.

| | Entire control plane | Select data plane |
|---|---|---|
| Data Planes imported | all of them | the one you pick |
| Applications imported | all applications in the subscription | the applications on that data plane |
| Application detail | id, name, description, state, version, capability *(with nesting on)* | id, name, description, state, version, capability |
| Platform calls | 1 + 1 + one per application *(with nesting on)* | 1 + 1 |
| Best for | the full picture; a first import | one team's environment; large subscriptions |

#### Entire control plane

Imports the whole topology. This is the option that answers "what does our platform look like",
and the one to use for a first import.

It reveals an extra option:

##### Nest applications under their data plane

*Boolean. Default: on. Only shown for `Entire control plane`.*

Controls the shape of the imported tree.

**On** — the flow looks up each application individually to find out which data plane it runs on, so
you get the full three-level hierarchy:

```
Control Plane
└── Data Plane
    └── Application
```

**Off** — the flow skips those lookups and lists every application directly under the Control Plane.
Data Planes are still imported, but applications are not attached to them:

```
Control Plane
├── Data Plane          (no applications underneath)
└── Application         (flat, alongside the data planes)
```

The trade-off is time. Nesting costs **one extra platform API call per application**, so a
subscription with 200 applications makes 200 extra calls and the task takes correspondingly longer.
Turn nesting off when you want a fast inventory of *what exists* and do not need *where it runs*, or
when a first nested import is taking longer than you are willing to wait.

Applications imported with nesting on also carry richer metadata — state, version and capability —
because the per-application lookup returns it. See
[What Gets Created](catalog-result.md#annotations).

#### Select data plane

Imports a single data plane and the applications on it. The **Data Plane** field appears and lets you
pick one from your platform; it is required.

The Control Plane Component is still created, with just this one Data Plane underneath it. That means
you can import data planes one at a time — run the flow again with a different data plane and the
same Control Plane Name, and the new branch is added to the same tree.

Applications always carry the full metadata in this mode: the per-data-plane endpoint returns state,
version and capability in one call, so there is no nesting option to trade off.

!!! warning "Import one data plane at a time, but into the same repository folder"
    Each run overwrites `catalog-info.yaml` in the target folder. To keep several data planes in one
    tree, give each run a **different folder** in the repository — otherwise the second run replaces
    the first run's snapshot and the entities from the first data plane are removed from the catalog
    on the next refresh.

---

## Repository Location

The generated `catalog-info.yaml` is committed to Git and registered in the catalog from its GitHub
URL. That keeps a versioned snapshot of the topology in your repository, and lets the Hub's refresh
loop keep the entities current.

### Target GitHub Repository

*Required. Repository picker.*

An **existing** repository the snapshot is committed into. The flow does not create repositories. The
Developer Hub's GitHub integration must be allowed to push to it.

### Branch

*Required. Text. Default: `main`.*

The branch that is cloned and pushed to. It must already exist.

### Folder In The Repository

*Required. Text, `[a-zA-Z0-9._/-]` only. Default: `tibco-platform`.*

The folder that will hold the generated `catalog-info.yaml`; nested paths such as
`platform/emea-prod` are allowed. The file is always named `catalog-info.yaml`, so the folder is what
separates one import from another:

- **Same folder, same Control Plane Name** — a refresh. The file is overwritten and the existing
  entities are updated in place.
- **Different folder** — a separate snapshot, registered as its own catalog location. Use this for a
  second control plane, or to build a tree one data plane at a time.
