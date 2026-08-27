# 6. Anti-patterns

| Don't | Why | Instead |
| --- | --- | --- |
| Point a superseded version at `main` | The "old" spec silently becomes the new one | Pin to a tag, or freeze a copy ([§3.3](practices.md#33-freeze-the-specification-alongside-the-entity)) |
| Create an entry per patch release | Unnavigable within a quarter | One entry per major ([§3.2](practices.md#32-version-by-major-release-not-by-every-change)) |
| Use a namespace per version | Fragments ownership and search; awkward refs | Version suffix on the name ([§3.1](practices.md#31-make-the-version-part-of-the-entity-name)) |
| Leave old versions as `production` | The lifecycle filter becomes meaningless | Deprecate on supersession ([§3.6](practices.md#36-use-lifecycle-to-signal-supported-status)) |
| Delete an old version's entry on release | Breaks links and hides who's still on it | Deprecate, sunset, then remove ([§3.6](practices.md#36-use-lifecycle-to-signal-supported-status)) |
| Register versions by hand | Catalog drifts from reality, then loses trust | Automate from the release pipeline ([§3.10](practices.md#310-automate-registration-from-the-release-pipeline)) |
| Ship a version with no changelog | Consumers can't tell whether upgrading is safe | Generate it with `api-version-diff` ([§3.9](practices.md#generating-the-changelog-with-the-api-version-diff-skill)) |
| Mix naming conventions | Broken links and duplicate content ([§4](your-apis.md#4-what-the-platform-example-does-not-yet-demonstrate)) | Pick one format and enforce it |

---

# 7. Quick reference

| Concern | Practice | Platform example |
| --- | --- | --- |
| Multiple live versions | One entry per major | `tibco-platform-api-117/118/119` |
| Version in the model | Suffix on `metadata.name` | `-118` |
| Exact contract version | `metadata.title` + annotation | `backstage-api-1.41.1.yaml` |
| Historic versions | Frozen spec copy, or tag-pinned location | `version-118/control-plane-api-118.json` |
| Release as a unit | One `Location` per version | `catalog-info-apis-118.yaml` |
| Supported status | `spec.lifecycle` | *(gap — all `production`)* |
| Grouping versions | Shared tag + shared `system` | `tibco-platform` tag, `TIBCO` Domain |
| Version selection UX | Pinned current + multi-install picker | Two Marketplace entries |
| Documentation | Per-version TechDocs | `techdocs-ref: dir:.` per folder |
| Changelog per version | Generated from the two specs | `api-version-diff` skill → `version-<NNN>/docs/` |
| Who's on which version | `providesApis` / `consumesApis` | *(gap — not modelled)* |
| Policy enforcement | Spec diff + lint in CI | `apidiff.mjs` exit code as the gate |
| Design → runtime | Gateway as `Resource`, API `dependsOn` | *(external to the catalog)* |

---

# Related skills

Three skills in the Developer Hub Skills Library implement parts of this guide:

| Skill | Use it for |
| --- | --- |
| **`api-version-diff`** | Comparing two versions of a specification and publishing the difference as per-version TechDocs — [§3.9](practices.md#generating-the-changelog-with-the-api-version-diff-skill). Doubles as the CI breaking-change gate — [§5.3](your-apis.md#53-enforce-the-policy-in-ci). |
| **`impact-analysis`** | Finding out who actually consumes the version you want to deprecate, before you announce a sunset date — [§5.2](your-apis.md#52-declare-who-consumes-which-version). |
| **`reuse-or-build`** | Checking whether an existing API version already carries the data a new consumer needs, before adding another one. |

The library is a Marketplace entry: **Developer Hub Skills Library**.

---

# References

- Platform API content — `platform/apis/`
- Marketplace entries — *Platform APIs 1.18 (Most Recent)*, *Platform APIs (version 1.7 – 1.18)*
- Backstage API entity model — <https://backstage.io/docs/features/software-catalog/descriptor-format#kind-api>
- Upstream versioning proposal (closed, not planned) — <https://github.com/backstage/backstage/issues/11027>
