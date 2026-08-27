# 4. What the platform example does *not* yet demonstrate

Held to its own standard, the shipped content has three gaps worth knowing about, because they are
exactly the things you should add for your own APIs:

**All 59 entries are `lifecycle: production`.** Platform 1.7 is presented with the same status as
1.19. There is nothing in the catalog telling a user that 1.7 is years old and unsupported. Older
versions should be `deprecated`, and the Hub's lifecycle filter would then do real work.

**No succession or sunset metadata.** Nothing on the 1.15 entry points to 1.16, and nothing states
when 1.15 support ends. A user landing on an old entry from a search result has no signal to move on.

**No consumer relations.** The platform API entries declare no `providesApis` / `consumesApis` links,
so the catalog can't answer *"which of our components call platform API 1.14?"* For product
documentation that's acceptable; for your own internal APIs it is the highest-value thing the catalog
can give you (see [§5.2](#52-declare-who-consumes-which-version)).

Two minor housekeeping notes: versions 1.7–1.10 contain duplicate Location files under both naming
conventions (`catalog-info-apis-1.9.yaml` and `catalog-info-apis-19.yaml`, byte-identical bar a
trailing newline), and the multi-version Marketplace template builds a `/tree/` URL where the pinned
one uses `/blob/`. Both are harmless, and both illustrate why
[§3.1](practices.md#31-make-the-version-part-of-the-entity-name)'s "pick one convention and never
mix" is worth enforcing early.

---

# 5. Applying this to your own APIs

## 5.1 Entity template

```yaml
apiVersion: backstage.io/v1alpha1
kind: API
metadata:
  name: orders-api-v1
  title: Orders API v1
  description: 'Orders API — v1 (deprecated, sunset 2026-12-31)'
  tags: [orders, rest, deprecated]
  annotations:
    backstage.io/techdocs-ref: dir:.
    api.your-org.com/version: '1.9.4'                     # exact contract version
    api.your-org.com/superseded-by: 'api:default/orders-api-v2'
    api.your-org.com/sunset-date: '2026-12-31'
  links:
    - title: Migration guide — v1 to v2
      url: https://your-docs/orders/migration-v1-to-v2
spec:
  type: openapi
  lifecycle: deprecated          # experimental → production → deprecated
  owner: orders-team
  system: order-management       # shared across every version — groups the family
  definition:
    $text: ./openapi-v1.yaml     # frozen copy, or a tag-pinned location
```

## 5.2 Declare who consumes which version

This is the step that turns the catalog from documentation into an operational tool, and it is worth
more than everything else in this guide combined. The implementing service declares every major it
still serves; each consumer declares the exact one it calls:

```yaml
# the service implementing the API
spec:
  providesApis: [orders-api-v1, orders-api-v2]

# each consuming service
spec:
  consumesApis: [orders-api-v1]
```

The v1 entry's page now answers *"who is still on v1, and which team owns them?"* — the question that
actually blocks a deprecation. Developer Hub renders this as a topology diagram, and the
`impact-analysis` skill turns it into a written impact report for any entry: every dependent,
classified by severity, plus the teams to notify.

!!! note "Use the standard relations"
    Use `providesApis` / `consumesApis`. A non-standard `apiConsumedBy` field appears in some older
    example content; nothing in Developer Hub reads it.

## 5.3 Enforce the policy in CI

The catalog records versioning decisions; it doesn't police them. Three checks in the pipeline:

| Check | Tool | Fails the build when |
| --- | --- | --- |
| Breaking-change detection | `api-version-diff`'s `apidiff.mjs`, `oasdiff`, `openapi-diff` | A breaking change lands without a major bump |
| Specification linting | Spectral + house ruleset | Naming, errors, pagination or auth deviate from standard |
| Catalog validation | Developer Hub entity validation | The generated entity wouldn't register cleanly |

Breaking-change detection is the substantive one. It converts "we version properly" from a stated
intention into something mechanically true.

The helper shipped with the `api-version-diff` skill doubles as this gate — it exits `1` when it
finds a breaking change, `0` when it does not:

```sh
# fails the build if the new spec breaks the published one
node .claude/skills/api-version-diff/apidiff.mjs published/openapi.yaml openapi.yaml \
  || { echo "Breaking change detected — bump the major version"; exit 1; }
```

Run the skill itself on the same pair to turn that failure into the changelog and migration guide the
major bump needs — see [§3.9](practices.md#generating-the-changelog-with-the-api-version-diff-skill).

## 5.4 Link design-time to runtime

If an API gateway fronts these APIs at runtime, model the gateway as a `Resource` entity and have the
API entries `dependsOn` it. The specification, its versions, its consumers and its enforcement point
then sit in one graph — which is precisely the seam between design-time governance and runtime
management.

**Next:** [anti-patterns and the quick reference](reference.md).
