# Import the TIBCO Platform

This import flow reflects a **live TIBCO Platform** in the TIBCO Developer Hub catalog. It reads the
topology of your Control Plane through the Platform API, renders it as catalog entities, commits that
snapshot to a GitHub repository and registers it in the catalog.

The result is a three-level component tree you can browse, search, filter and diagram like any other
software in the Hub:

```
Control Plane                     (Component, type: control-plane)
├── Data Plane "eu-prod"          (Component, type: data-plane)
│   ├── order-api                 (Component, type: application)
│   └── invoice-processor         (Component, type: application)
├── Data Plane "dev-cluster"      (Component, type: data-plane)
│   └── customer-sync             (Component, type: application)
└── Unassigned applications       (Component, type: data-plane) — only when needed
    └── legacy-batch              (Component, type: application)
```

Nothing about your platform is modified. Every call the flow makes is a read.

## Why import the platform

- **One catalog for everything.** Your platform runtime shows up next to the APIs, BW6 projects and
  Flogo apps already in the Hub, owned by the same groups and searchable in the same place.
- **Ownership and discovery.** Every imported Component carries an owner, so `/catalog` filters such
  as "owned by my group" cover platform workloads too.
- **A versioned snapshot.** The generated `catalog-info.yaml` is committed to Git, so the topology at
  any point in time is in your repository history, not only in the Hub's database.
- **Dependency views.** The Hub's system diagram renders the Control Plane → Data Plane →
  Application hierarchy, which is a faster answer to "what runs where" than clicking through the
  Control Plane UI.

## Before you start

| Requirement | Detail |
|---|---|
| Platform connection | The Developer Hub must be configured to reach your Control Plane. The flow uses the Hub's own platform connection — it never asks you for a token. |
| Read permission | The account behind that connection needs to read data planes and applications on the subscription. |
| A GitHub repository | An existing repository the generated `catalog-info.yaml` is committed into, with the Hub's GitHub integration allowed to push to it. |

## Running it

1. Open **Import Flows** in the Developer Hub and pick **Import the TIBCO Platform**.
2. Name the Control Plane and choose the owning group.
3. Choose **what to import** — the entire control plane or a single data plane. See
   [Import Options](import-options.md).
4. Point the flow at the repository, branch and folder that will hold the snapshot.
5. Run it. When the task finishes, follow **Open Control Plane component in catalog** to the imported
   tree.

Re-running the flow with the same inputs updates the existing entities in place instead of creating
duplicates — see [How It Works](how-it-works.md#re-running-the-import).

## Where to go next

- **[Import Options](import-options.md)** — what every field on the form does, and which combination
  to pick.
- **[What Gets Created](catalog-result.md)** — the exact entities, names, tags and annotations, and
  how they appear in the Hub.
- **[How It Works](how-it-works.md)** — the API calls behind each option, the cost of nesting,
  re-import behaviour and known limitations.
