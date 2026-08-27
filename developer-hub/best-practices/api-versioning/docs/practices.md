# 3. The practices

Ten conventions, each with the Platform example that implements it and the rule to take away.

## 3.1 Make the version part of the entity name

| | |
| --- | --- |
| **Platform example** | `tibco-platform-api-118`, `bwce-capability-api-117`, `flogo-capability-api-115` |
| **Rule** | `<api-name>-<version>` — the version is a suffix on `metadata.name` |

Entity names must be unique within kind + namespace, so two concurrently visible versions *require*
two names. Encoding the version in the name is the only approach that works with the catalog's
identity model, and it has a useful side effect: the entity reference (`api:default/tibco-platform-api-118`)
is self-describing wherever it appears — in relations, in URLs, in impact reports.

**On format:** entity names permit `[a-zA-Z0-9]` separated by `[-_.]`, up to 63 characters — so
`tibco-platform-api-1.18` would be legal. The platform content deliberately drops the dot (`1.18` →
`118`) to keep URLs and entity refs free of characters that need escaping or invite typos. Either is
defensible; **pick one and never mix**. Note that catalog *tags* are stricter than names — lowercase
alphanumeric separated by hyphens only, no dots — so a tag-based grouping scheme must use
`tibco-platform`, not `tibco.platform`.

## 3.2 Version by major release, not by every change

| | |
| --- | --- |
| **Platform example** | One entry set per platform minor release (1.17, 1.18, 1.19) — not per patch |
| **Rule** | Create a new entry only when consumers must take action |

Minor and patch releases are backwards-compatible by definition, so they should flow into the
existing entry — that is what "backwards compatible" means. Create a new entry when a consumer would
have to change code, which for a public API means a **major** version.

The platform example versions per *release train* because that is the unit its consumers actually
care about ("which platform release am I on?"). Choose the axis your consumers version against — for
a product API that is usually the product release; for a service API it is the API's own major.

## 3.3 Freeze the specification alongside the entity

| | |
| --- | --- |
| **Platform example** | `version-118/control-plane-api-118.json` — the spec document lives inside the version folder |
| **Rule** | A superseded version's specification must never change again |

This is the single most important practice, and the platform content implements the strongest form
of it: each version folder physically contains its own copy of the specification document, referenced
relatively (`$text: ./control-plane-api-118.json`). Publishing 1.19 cannot alter what 1.18 shows,
because 1.19 is a different file in a different folder.

If you don't want copies in the repository, the equivalent is to **register each superseded version's
catalog file from an immutable git tag** rather than a branch:

```
https://github.com/your-org/orders/blob/v1.9.4/catalog/orders-api-v1.yaml   ← tag, frozen
https://github.com/your-org/orders/blob/main/catalog/orders-api-v2.yaml     ← branch, current
```

Only the current major should track a moving branch. Every superseded version points at a tag.

## 3.4 Register a version as a set, atomically

| | |
| --- | --- |
| **Platform example** | `catalog-info-apis-118.yaml` — one `Location` listing all five APIs of that release |
| **Rule** | One `Location` per release; install or remove a version in one action |

The Platform release isn't one API, it's five (platform core, BWCE, Flogo, BW5CE, Developer Hub).
Wrapping them in a single `Location` means a user installs "Platform APIs 1.18" as a unit, and the
Hub can later drop the whole set just as cleanly.

This also handles **APIs entering and leaving the set over time** with no special mechanism: releases
1.7–1.9 contain four APIs, while 1.10 onwards contain five — BW5CE Capability was added at 1.10, and
it simply appears in the newer Location files. Each release describes its own shape.

## 3.5 Separate the release version from the contract version

| | |
| --- | --- |
| **Platform example** | `developer-hub-api-118` serves the specification `backstage-api-1.41.1.yaml` |
| **Rule** | Don't force one version number onto two independent things |

The Developer Hub API entry is named for the platform release, but the contract it exposes is the
Backstage catalog API — two different version lines that move at different speeds. Platform 1.10
through 1.18 all serve Backstage 1.41.1; 1.19 moves to 1.51.0. The entity name tracks the version
consumers select ("I'm on platform 1.18"); the specification carries its own version internally.

Where they differ, keep both visible: release version in the name, exact contract version in
`metadata.title` and an annotation.

## 3.6 Use `lifecycle` to signal supported status

| | |
| --- | --- |
| **Platform example** | Every entry is `lifecycle: production` — see the gap noted in [§4](your-apis.md#4-what-the-platform-example-does-not-yet-demonstrate) |
| **Rule** | `experimental` → `production` → `deprecated`, and keep it current |

`spec.lifecycle` is a free-form string, but these three values are the de-facto standard, and
Developer Hub renders lifecycle as a status on the entry and offers it as a filter in the API
Explorer. That makes *"show me every deprecated API"* a single click — but only if entries are
maintained as they age. Moving a version to `deprecated` should be a step in your release checklist,
not an afterthought.

## 3.7 Group versions so the API Explorer stays navigable

| | |
| --- | --- |
| **Platform example** | Shared `tibco-platform` tag on all 59 entries; shared `TIBCO` Domain and Group |
| **Rule** | Give every version of an API a shared grouping handle |

With thirteen releases × five APIs, an ungrouped API Explorer would be unusable. Two mechanisms, both
free:

- **Tags** — every platform entry carries `tibco-platform`, plus a role tag (`platform`, `core`,
  `capability`). Filter to one family in a click.
- **System / Domain** — putting all versions of an API in the same `spec.system` gives you a version
  family view via the System page's *has part* relations, with no custom development.

For your own APIs, the shared `system` is usually the better handle: it groups the versions *and*
ties them to the service that implements them.

## 3.8 Make version selection a first-class user experience

| | |
| --- | --- |
| **Platform example** | Two Marketplace entries — one pinned to the current release, one with a version picker |
| **Rule** | Current version gets a prominent entry; historical versions sit behind one picker |

This is the pattern that solves the "thirteen versions clutter the catalog" problem without writing a
line of frontend code:

- **`mp-entry-doc-platform-apis-118`** — *"Platform APIs 1.18 (Most Recent)"*, flagged `isNew: true`,
  `isMultiInstall: false`. One-click install of the current release.
- **`doc-platform-previous-apis`** — *"Platform APIs (version 1.7 – 1.18)"*, flagged
  `isMultiInstall: true`, with a `platformVersion` dropdown listing every release and defaulting to
  the newest.

The second entry constructs the registration URL from the chosen version:

```yaml
steps:
  - id: registerItem
    action: catalog:register
    input:
      catalogInfoUrl: ${{ "https://github.com/TIBCOSoftware/tibco-developer-hub-marketplace/tree/main/platform/apis/version-"
                          + parameters.platformVersion + "/catalog-info-apis-" + parameters.platformVersion + ".yaml" }}
```

One template covers every historical version, and adding a release means adding one line to the
`enum`. The same construction works for the output links, so the user lands directly on the rendered
specification for the version they just installed.

## 3.9 Version the documentation with the API

| | |
| --- | --- |
| **Platform example** | `backstage.io/techdocs-ref: dir:.` with `mkdocs.yaml` + `docs/` inside each version folder |
| **Rule** | Each version's entry links to that version's documentation |

Because the TechDocs reference is relative to the entity's own directory, each release ships and
serves its own documentation set. A user reading the 1.15 API gets the 1.15 guidance, not the current
one. This is where migration guides belong.

### Generating the changelog with the `api-version-diff` skill

The reason per-version documentation goes stale is that writing it by hand is tedious and easy to
skip. The **`api-version-diff`** skill in the
[Developer Hub Skills Library](https://github.com/TIBCOSoftware/tibco-developer-hub-marketplace/tree/main/developer-hub/skills)
removes that excuse. Point it at two specification documents and it:

1. runs a structural comparison — operations, parameters, request bodies, response codes, shared
   components, schemas, schema properties, enums and security schemes;
2. classifies every finding **🔴 breaking / 🟢 additive / 🔵 note**, handling the OpenAPI 3.0 → 3.1
   `nullable` re-spelling that otherwise reports a whole schema as deleted;
3. writes the changelog page into the version folder in the structure described above, wires it into
   `mkdocs.yaml`, and pairs removals with additions so a rename reads as a rename;
4. cross-references `/impact-analysis` on the API entity, so a breaking change is reported as
   *"breaks these three components, owned by these two teams"* rather than just *"breaking"*.

```sh
# In your Developer Hub checkout
node .claude/skills/api-version-diff/apidiff.mjs \
     version-118/control-plane-api-118.json \
     version-119/control-plane-api-119.json --from 1.18 --to 1.19
```

The helper exits **1** when it finds a breaking change and **0** when it does not, which makes the
same command the breaking-change gate described in
[§5.3](your-apis.md#53-enforce-the-policy-in-ci).

The Platform API changelogs under `version-<NNN>/docs/` are the reference output — one page per API,
each with a section per version transition.

## 3.10 Automate registration from the release pipeline

| | |
| --- | --- |
| **Platform example** | `catalog:register` action with a version-parameterised URL |
| **Rule** | Publishing a version and cataloguing it are one action, not two |

The manual step — "remember to add the new version to the Hub" — is the step that gets skipped, and
a catalog that lags reality stops being trusted. Wire it into the release:

1. On a release tag, generate the versioned entity file(s) and the `Location`, pinned to that tag.
2. Register them via the catalog registration action (or the Developer Hub catalog API) from CI.
3. Generate the changelog for the transition (§3.9) and commit it into the version folder.
4. In the same job, flip the *previous* major to `lifecycle: deprecated` and set its sunset date.

Developer Hub's self-service flows can host step 2 directly — the Marketplace template above is a
working example to copy.

**Next:** [applying it to your own APIs](your-apis.md).
