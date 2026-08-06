# Developer Hub API Changelog

Complete changelog for the TIBCO Developer Hub catalog API through platform version 1.19.

The Developer Hub catalog API is the OpenAPI document published by the Backstage catalog backend
plugin the Hub is built on, so it changes when the Hub's Backstage version changes rather than with
every platform release:

| Platform Version | Spec published with the entry | Operations | Schemas |
|---|---|---|---|
| 1.7 – 1.9 | `backstage-api.yaml` — abbreviated catalog spec | 16 | 23 |
| 1.10 – 1.18 | `backstage-api-1.41.1.yaml` — Backstage 1.41.1 | 16 | 23 |
| 1.19 | `backstage-api-1.51.0.yaml` — **Backstage 1.51.0** | **20** | **24** |

The only customisation applied to the published spec is the `servers` block, which is replaced with
the two TIBCO Developer Hub base URLs (`/tibco/hub/api/catalog/` and
`http://localhost:7007/api/catalog/`) instead of upstream's `servers: [{ url: "/" }]`.

---

## Version 1.18 → 1.19

**Backstage version:** 1.41.1 → 1.51.0
(`@backstage/plugin-catalog-backend` 3.8.0, `@backstage/catalog-client` 1.16.0)

No operations, parameters or schemas were **removed**. Every change is additive except the OpenAPI
dialect bump, which changes how nullable types are expressed.

| Area | Change |
|---|---|
| Spec dialect | `3.0.3` → `3.1.0` |
| Operations | 4 added (16 → 20) |
| Query parameters | 2 added |
| Schemas | 1 added, 2 modified |

### New Endpoints

| Method | Path | Operation ID | Description |
|--------|------|--------------|-------------|
| POST | `/entities/by-query` | `QueryEntitiesByPredicate` | Request-body twin of `GET /entities/by-query`, for filters that exceed practical URL length limits |
| POST | `/entity-facets` | `QueryEntityFacetsByPredicate` | Request-body twin of `GET /entity-facets` |
| POST | `/locations/by-query` | `GetLocationsByQuery` | Paginated, filterable listing of locations alongside the unpaginated `GET /locations` |
| PUT | `/locations/{id}` | `UpdateLocation` | Update a location in place — locations were previously create/read/delete only |

**`POST /entities/by-query`** takes an optional JSON body and returns the same
`EntitiesQueryResponse` as the GET form:

| Field | Type | Notes |
|---|---|---|
| `cursor` | string | cursor pagination |
| `limit` | number | |
| `offset` | number | |
| `orderBy` | array of `{ field, order: asc\|desc }` | multi-field sorting |
| `fullTextFilter` | `{ term, fields[] }` | free-text search |
| `fields` | string[] | response field projection |
| `totalItems` | `include` \| `exclude` | skip the (expensive) count; defaults to `include` |
| `query` | `JsonObject` | structured filter predicate |

**`POST /entity-facets`** requires `facets: [string]` and accepts an optional `query` predicate,
returning `EntityFacetsResponse`.

**`POST /locations/by-query`** accepts an optional `cursor`, `limit` and `query`, and returns the new
`LocationsQueryResponse` schema.

**`PUT /locations/{id}`** takes a required `LocationInput` body and returns `200` with the updated
`Location`.

### Removed Endpoints

None. All 16 pre-existing operations keep the same `operationId`s and response codes.

### New Query Parameters

- **`totalItems` on `GET /entities/by-query`** — new shared component
  `#/components/parameters/totalItems`, `enum: [include, exclude]`. Controls whether `totalItems` is
  computed in the response. Computing the total is expensive on large catalogs; pass `exclude` when
  the caller only needs the page (for example a cursor-paginated UI that shows the count
  cosmetically). Defaults to `include`. Upstream notes that further values, such as an approximate
  mode, may be added later.
- **`onConflict` on `POST /locations`** — `enum: [refresh, reject]`. Behaviour when the location
  already exists. `reject` is the default and matches the previous hardcoded behaviour, returning
  `409`. `refresh` triggers a refresh of the existing location entity and returns `201`, making
  location registration idempotent without a read-then-write round trip.

### New Schemas

- **`LocationsQueryResponse`** — response envelope for `POST /locations/by-query`:

    ```yaml
    LocationsQueryResponse:
      type: object
      required: [items, totalItems, pageInfo]
      additionalProperties: false
      properties:
        items:      { type: array, items: { $ref: '#/components/schemas/Location' } }
        totalItems: { type: number }
        pageInfo:
          type: object
          properties:
            nextCursor: { type: string }
    ```

### Modified Schemas

- **`Location`** — new **required** property `entityRef` (the entity ref of the corresponding
  `Location` kind entity, e.g. `location:default/generated-<sha1hex>`). `required` is now
  `[target, type, id, entityRef]`. **This is the one change worth watching**: any client that
  constructs a `Location` object for validation against this schema — as opposed to only consuming
  responses from the server — fails validation until it supplies the field.
- **`NullableEntity`** — restructured for the OpenAPI 3.1 dialect only; semantics unchanged. Was a
  plain object with `nullable` semantics, now an `anyOf` of the object and `type: "null"`.
- **`AnalyzeLocationEntityField.value`** — same dialect change via `oneOf` (`type: string` +
  `type: "null"` instead of `nullable: true`).

The schema count grew from 23 to 24. Every other schema, and `components.securitySchemes.JWT`
(HTTP bearer), is unchanged.

### Notes

- **The OpenAPI dialect bump is the only change that can break a consumer.** OpenAPI 3.1 drops the
  3.0 `nullable: true` keyword in favour of a real `"null"` type. The Developer Hub API docs viewer
  (swagger-ui) renders 3.1 fine; any older validator or code generator in your pipeline that only
  speaks 3.0 needs checking.
- `GET /entities` parameters (`fields`, `limit`, `filter`, `offset`, `after`, `order`) are unchanged,
  as are the `GET /entities/by-query` parameters other than the added `totalItems` — including
  `fullTextFilterTerm` and `fullTextFilterFields`, which were already present in 1.41.1.
- `info.description` changed cosmetically upstream: the work-in-progress note is now an admonition
  (`:::note ... :::`) rather than a Markdown blockquote, and the `identityApiRef` link moved to
  `backstage.io/api/stable/...`.

---

## Version 1.9 → 1.10

**Spec:** `backstage-api.yaml` → `backstage-api-1.41.1.yaml`

The abbreviated catalog spec published with platform 1.7 – 1.9 was replaced by the spec exported from
the Backstage catalog backend plugin itself — the exact document the running backend routes against.
The operation set is the same 16 operations; the descriptions, examples and schema detail are
substantially richer.

---

## Regenerating the spec

The catalog OpenAPI document is not hand-maintained. It is exported from the installed backend
plugin package:

```
node_modules/@backstage/plugin-catalog-backend/dist/schema/openapi/generated/router.cjs.js  →  exports.spec
```

After a future Backstage bump, regenerate it from a Developer Hub checkout with:

```bash
node -e "
const { spec } = require('./node_modules/@backstage/plugin-catalog-backend/dist/schema/openapi/generated/router.cjs.js');
const yaml = require('yaml');
spec.servers = [{ url: '/tibco/hub/api/catalog/' }, { url: 'http://localhost:7007/api/catalog/' }];
require('fs').writeFileSync('backstage-1XX-api.yaml', yaml.stringify(spec, { lineWidth: 0 }));
"
```
