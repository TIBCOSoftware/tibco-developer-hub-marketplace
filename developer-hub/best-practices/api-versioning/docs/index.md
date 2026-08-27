# Best Practices: API Versioning in TIBCO Developer Hub

**Audience:** platform teams, API owners and architects modelling APIs in the Developer Hub catalog.
**Worked example:** the TIBCO Platform APIs, as published in the Developer Hub Marketplace.

---

## 1. Why this needs a convention

TIBCO Developer Hub — and the Backstage catalog underneath it — has no built-in concept of API
versions. An `API` entity has exactly five spec fields:

```yaml
spec:
  type:        # openapi | asyncapi | graphql | grpc
  lifecycle:   # free-form string
  owner:
  system:      # optional
  definition:  # the specification document
```

There is no `version` field, no version grouping, and no link between an entity and the git history
of the repository its specification came from. The community proposal to add this (backstage/backstage
issue #11027, *"Versioned API support"*) was **closed as not planned**.

The practical consequences:

- An API entry points at **one** specification, resolved from wherever it was registered. Register it
  against `main` and it silently re-reads the latest spec on every catalog refresh — previous versions
  vanish.
- Git tags in the source repository are **not** surfaced anywhere in the catalog.
- Therefore: **version must be modelled explicitly, as part of entity identity.**

That sounds like a limitation, but the convention that follows from it is clean, and Developer Hub
already ships a full-scale implementation of it. The rest of this guide uses that implementation
as the reference.

!!! tip "The documentation half of this is automatable"
    Every practice below asks you to publish, per version, an accurate account of what changed. The
    **`api-version-diff`** skill in the
    [Developer Hub Skills Library](https://github.com/TIBCOSoftware/tibco-developer-hub-marketplace/tree/main/developer-hub/skills)
    does exactly that: point it at two specification files and it produces the TechDocs changelog,
    classifying every change as breaking or additive. See
    [§3.9](practices.md#39-version-the-documentation-with-the-api).

---

## 2. The worked example: how the TIBCO Platform APIs are versioned

The Platform API content lives under
`platform/apis/`. It publishes **thirteen
releases** of the platform API set, 1.7 through 1.19, all installable side by side — 59 `API`
entities in total.

### Directory anatomy

```
tibco-platform-apis/
├── tibco-platform-domain-group.yaml        # shared Group + Domain, referenced by every version
├── version-118/
│   ├── catalog-info-apis-118.yaml          # Location — the unit of registration
│   ├── tibco-platform-api-standalone-platform.yaml        → tibco-platform-api-118
│   ├── tibco-platform-api-standalone-bwce.yaml            → bwce-capability-api-118
│   ├── tibco-platform-api-standalone-flogo.yaml           → flogo-capability-api-118
│   ├── tibco-platform-api-standalone-bw5ce.yaml           → bw5ce-capability-api-118
│   ├── tibco-platform-api-standalone-developer-hub.yaml   → developer-hub-api-118
│   ├── control-plane-api-118.json          # the frozen specification documents
│   ├── bwce-capability-api-118.json
│   ├── flogo-capability-api-118.json
│   ├── bw5ce-capability-api-118.json
│   ├── backstage-api-1.41.1.yaml
│   ├── mkdocs.yaml + docs/                 # TechDocs for this version
├── version-117/  …
└── version-17/   …
```

### A single entity

```yaml
apiVersion: backstage.io/v1alpha1
kind: API
metadata:
  name: 'tibco-platform-api-118'
  description: 'API of the TIBCO Platform (1.18)'
  annotations:
    backstage.io/techdocs-ref: dir:.
  tags:
    - tibco-platform
    - platform
    - core
spec:
  type: openapi
  lifecycle: production
  owner: TIBCO
  definition:
    $text: ./control-plane-api-118.json
```

### The Location that binds a release together

```yaml
apiVersion: backstage.io/v1alpha1
kind: Location
metadata:
  name: tibco-platform-apis-118
  description: A collection of all TIBCO Platform API's
spec:
  targets:
    - ./../tibco-platform-domain-group.yaml
    - ./tibco-platform-api-standalone-platform.yaml
    - ./tibco-platform-api-standalone-bwce.yaml
    - ./tibco-platform-api-standalone-flogo.yaml
    - ./tibco-platform-api-standalone-developer-hub.yaml
    - ./tibco-platform-api-standalone-bw5ce.yaml
```

Everything in this guide is drawn from this structure.

**Next:** [the ten practices](practices.md).
