# BWCE Capability API Changelog

Complete changelog for the TIBCO BusinessWorks 6 (Containers) Capability API from version 1.7 through 1.19.

---

## Version 1.18 → 1.19

**API Version:** 1.19.0

This is the largest capability API release in the BW6 line so far. It completes the **autodiscovery
app lifecycle** — apps discovered on a data plane can now be deleted, scaled and reconfigured through
the API rather than only inspected — and adds **build cleanup** in line with the other capability APIs.

The v1 endpoints apply to **non-Helm managed** apps; the v2 endpoints apply to **Helm managed** apps.

### New Endpoints

| Method | Path | Operation ID | Description |
|--------|------|--------------|-------------|
| DELETE | `/v1/bwce/apps/{appName}` | `DeleteBWCEApp` | Delete an autodiscovered app, removing its deployment, config maps, secrets, service, ingress and HTTP routes (non-Helm only) |
| PUT | `/v1/bwce/apps/{appName}/scale` | `AppBWCEScale` | Scale an autodiscovered app to a given `replicas` count (non-Helm only) |
| GET | `/v1/bwce/apps/{appName}/yaml` | `GetBWCEAppYaml` | Get the app YAML of an autodiscovered app deployment (non-Helm only) |
| PUT | `/v1/bwce/apps/{appName}/yaml` | `PutAppYAMLForBWCEApp` | Apply an `app.yaml` configuration to an autodiscovered app (non-Helm only) |
| DELETE | `/v2/bwce/apps/{appName}` | `DeleteBWCEAppRelease` | Delete an autodiscovered Helm managed app release (Helm only) |
| PUT | `/v2/bwce/apps/{appName}/release/values` | `UpdateBWCEAppChartValues` | Update the Helm chart values (`values.yaml`) of an autodiscovered Helm app |
| DELETE | `/v1/dp/builds` | `TagAndCleanupAppBuilds` | Tag, untag, clean up or bulk delete application builds |

`DELETE /v1/dp/builds` takes a required `mode` of `tag`, `untag`, `cleanup` or `bulk`. The first three
are age-based and require `ageInDays`: `tag` marks unused builds older than that age with
`olderthan**age**days`, `untag` removes that tag again, and `cleanup` deletes everything carrying it.
`bulk` instead deletes the specific builds listed in the repeatable `buildIds` query parameter and
skips any build still in use by an app. Removed builds cannot be recovered.

### Removed Endpoints

None.

### New Schemas

- **TagAndCleanupResponse**: per-build result of the build cleanup operation — `buildId` (string), `status` (string), `message` (string).

### Modified Endpoints

- **GET `/v1/dp/apps`**: new optional `excludeAutodiscoveredApps` query parameter (boolean, default `false`) to leave autodiscovered apps out of the result.
- **GET `/v2/bwce/apps/{appName}/release/status`**: description clarified — applicable only to Helm managed apps.
- **GET `/v2/bwce/apps/{appName}/release/values`**: description clarified — applicable only to Helm managed apps.

Endpoint count rises from 67 to 74; schema count from 64 to 65.

---

## Version 1.17 → 1.18

**API Version:** 1.18.0

### Schema Changes

- **NamespaceInfo**: added new optional `viewOnly` field (boolean).

No endpoints were added or removed. Endpoint count remains 67; schema count remains 64.

---

## Version 1.16 → 1.17

**API Version:** 1.17.0

### New Endpoints

No new endpoints added.

### New Schemas

- WsdlInfo

---

## Version 1.7 to 1.8

### New Endpoints (3)

| Method | Path | Description |
|---|---|---|
| `GET` | `/v2/bwce/apps/{appName}/instances` | Get Autodiscovered BWCE Application instances |
| `GET` | `/v2/bwce/apps/{appName}/release/status` | Status of Autodiscovered BWCE Application Chart Deployment |
| `GET` | `/v2/bwce/apps/{appName}/release/values` | Get helm chart values from Autodiscovered BWCE app deployment |

---

## Version 1.8 to 1.9

### New Endpoints (1)

| Method | Path | Description |
|---|---|---|
| `GET` | `/v1/dp/supplements/{connector}` | Export the Supplement |

---

## Version 1.9 to 1.10

**Title changed:** `TIBCO BusinessWorks(TM) Container Edition Capability APIs` to `TIBCO BusinessWorks(TM) 6 (Containers) Capability APIs`

**Description changed:** Updated from "TIBCO BusinessWorks(TM) Container Edition" to "TIBCO BusinessWorks(TM) 6 (Containers)" terminology throughout.

### Modified Endpoints (17)

Major documentation refresh across the API, with summary and description updates reflecting the product rename:

- `GET /v1/cp/bwceversions` - summary and description changed
- `GET /v1/dp/apps` - description changed
- `POST /v1/dp/builds` - parameters and description modified
- `PUT /v1/dp/builds` - parameters modified
- `POST /v1/dp/builds/{buildId}/rebuild` - parameters and description modified
- `GET /v1/dp/bwceversions` - summary and description changed
- `DELETE /v1/dp/bwceversions/{version}` - summary, parameters, description modified
- `POST /v1/dp/bwceversions/{version}` - summary, parameters, description modified
- `DELETE /v1/dp/bwceversions/{version}/baseimages/{baseimagetag}` - summary, parameters, description modified
- `PUT /v1/dp/bwceversions/{version}/custombaseimage` - summary, parameters, request body, description modified
- `PUT /v1/dp/bwceversions/{version}/tags` - summary, parameters, request body, description modified
- `GET /v2/bwce/apps/{appName}/instances` - summary and description changed
- `GET /v2/bwce/apps/{appName}/release/status` - summary and description changed
- `GET /v2/bwce/apps/{appName}/release/values` - summary and description changed
- `GET /v2/dp/apps/{appId}/release/values` - summary and description changed
- `PUT /v2/dp/apps/{appId}/release/values` - summary and description changed
- `POST /v2/dp/deploy/release` - parameters modified

---

## Version 1.10 to 1.11

No significant changes. Minor cosmetic/formatting updates only.

---

## Version 1.11 to 1.12

### Schema Changes

- **`EgressNetworkPolicies`**: Added `clusterEgress` (type: `string`)

---

## Version 1.12 to 1.13

No significant changes. Minor cosmetic/formatting updates only.

---

## Version 1.13 to 1.14

No significant changes. Minor cosmetic/formatting updates only.

---

## Version 1.14 to 1.15

### Schema Changes (2)

Gateway support added to endpoint and public endpoint schemas:

- **`AppEndpointsResponse`**:
    - Added `gatewayControllerName` (type: `string`)
    - Added `gatewayName` (type: `string`)
    - Added `gatewayNamespace` (type: `string`)
    - Added `gatewaySectionName` (type: `string`)
    - Added `resourceInstanceName` (type: `string`)
- **`MakePublicEndpointRequest`**:
    - Added `gatewayControllerName` (type: `string`)
    - Added `gatewayHostName` (type: `string`)
    - Added `gatewayName` (type: `string`)
    - Added `gatewayNamespace` (type: `string`)
    - Added `gatewaySectionName` (type: `string`)
    - Added `resourceInstanceName` (type: `string`)

### Modified Endpoints (5)

- `GET /v1/dp/apps/{appId}/endpoints` - description changed
- `DELETE /v1/dp/apps/{appId}/endpoints/public` - description changed
- `POST /v1/dp/apps/{appId}/endpoints/public` - request body and description modified
- `DELETE /v2/dp/apps/{appId}/endpoints/public` - description changed
- `POST /v2/dp/apps/{appId}/endpoints/public` - summary, request body, and description modified

---

## Version 1.15 to 1.16

### Schema Changes (3)

Connector catalog metadata enrichment:

- **`CPPlugin`**:
    - Added `description` (type: `string`)
    - Added `displayVersion` (type: `string`)
    - Added `docLink` (type: `string`)
    - Added `endOfSupportDate` (type: `string`)
    - Added `releaseDate` (type: `string`)
    - Added `supportedBWCEVersions` (type: `string`)
    - Added `supportedBWVersions` (type: `string`)
- **`ConnectorCatalog`**:
    - Added `displayVersion` (type: `string`)
    - Added `endOfSupportDate` (type: `string`)
    - Added `releaseDate` (type: `string`)
    - Added `supportedBWCEVersions` (type: `string`)
    - Added `supportedBWVersions` (type: `string`)
    - Added `version` (type: `string`)
- **`EndpointInfo`**:
    - Added `publicSwaggerJsonURL` (type: `string`)

### Modified Endpoints (2)

- `GET /v1/dp/apps/{appId}/details` - responses modified (array to single object for AppDetailsResponse)
- `GET /v1/dp/apps/{appId}/endpoints` - responses modified (array to single object for AppEndpointsResponse)
