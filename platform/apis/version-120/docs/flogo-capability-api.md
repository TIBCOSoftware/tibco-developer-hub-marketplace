# Flogo Capability API Changelog

Complete changelog for the TIBCO Flogo Capability API from version 1.7 through 1.20.

---

## Version 1.19 → 1.20

**API Version:** 1.20.0

This release adds **build log streaming** and much richer **build timing**, alongside the per-container instance status shared with the BW6 and BW5 CE capability APIs.

### New Endpoints

| Method | Path | Operation ID | Description |
|--------|------|--------------|-------------|
| GET | `/v1/dp/builds/{buildId}/logs` | `streamBuildLogs` | Stream the build logs for the given build ID |

While the build job pod is running the logs stream live; once the pod is gone the durable stored log is returned instead. `follow=true` keeps the connection open and tails new lines as they are produced, and `tail=N` returns only the last N lines (minimum 1). The response is `text/plain`. Neither the BW6 nor the BW5 CE capability API has this operation.

### Removed Endpoints

None.

### New Schemas

- **ContainerStatusInfo**: per-container status — `name` (string), `ready` (boolean), `state` (string: `Running`, `Waiting` or `Terminated`), `reason` (string, e.g. `ContainerCreating`, `CrashLoopBackOff`), `restartCount` (int32) and `isInit` (boolean, present only for init containers).

### Schema Changes

- **BuildStatusResponse**: added build timing. `queued` (object) carries `queuedForSeconds` (int64 — time spent waiting, or until the build started) and `queueTimeoutSeconds`, and is present once the build has been queued. `building` (object) carries `buildingForSeconds` (int64 — time spent building, or the total duration for terminal builds) and the per-command `buildTimeoutSeconds`; it is always present once the build is queued, with `buildingForSeconds` at 0 until the build actually starts. Also added `elapsedTimeSeconds` (int64, queued until now, or until the build finished) and `failureReason` (string, from `build.json` `status.reason`, present only on failed builds).
- **AppInstancesResponse**: added `containerStatuses`, an array of `ContainerStatusInfo` holding the init containers first (flagged with `isInit`) and then the regular containers, and `ready` (boolean), true only when every regular container is ready — init containers are not counted. The existing `status` field is now documented as the raw Kubernetes pod phase (`Pending`/`Running`/`Succeeded`/`Failed`/`Unknown`), which may not reflect container readiness; use `ready` for that.

Endpoint count rises from 53 to 54; schema count from 46 to 47.

---

## Version 1.18 → 1.19

**API Version:** 1.19.0

### New Endpoints

| Method | Path | Operation ID | Description |
|--------|------|--------------|-------------|
| DELETE | `/v1/dp/builds` | `TagAndCleanupAppBuilds` | Tag, untag and clean up unused application builds older than a given age |

`ageInDays` and `mode` are both required; `mode` accepts `tag` (default), `untag` or `cleanup`.
`tag` marks unused builds older than `ageInDays` with `olderthan**age**days`, `untag` removes that
tag again, and `cleanup` deletes every build carrying it. Builds tagged `DoNotDelete` are skipped
during cleanup. Removed builds cannot be recovered.

Unlike the BW6 and BW5 CE variants of this operation, the Flogo version has no `bulk` mode.

### Removed Endpoints

None.

### New Schemas

- **TagAndCleanupResponse**: per-build result of the build cleanup operation — `buildId` (string), `status` (string), `message` (string).

### Modified Endpoints

- **DELETE `/v1/dp/builds/{buildId}`**: description updated — builds carrying the `DoNotDelete` tag cannot be deleted.

Endpoint count rises from 52 to 53; schema count from 45 to 46.

---

## Version 1.17 → 1.18

**API Version:** 1.18.0

### Schema Changes

- **NamespaceInfo**: added new optional `viewOnly` field (boolean).

### Modified Endpoints

- **GET `/v1/dp/supplements/{connector}`**: the `connector` path parameter enum gained a new value `SAPSolutions` (now `OracleDatabase`, `IBM-MQ`, `SAPSolutions`).

No endpoints were added or removed. Endpoint count remains 52; schema count remains 45.

---

## Version 1.16 → 1.17

**API Version:** 1.17.0

### New Endpoints

No new endpoints added.

### New Schemas

No new schemas added.

---

## Version 1.7 to 1.8

### New Endpoints (1)

| Method | Path | Description |
|---|---|---|
| `GET` | `/v2/dp/apps/{appId}/release/history` | Release history of an Application Chart Deployment |

### Schema Changes

- **`AppResponse`**:
    - Added `deployedAppName` (type: `string`)
    - Added `userAppName` (type: `string`)

---

## Version 1.8 to 1.9

### Schema Changes

- **`AppDetailsResponse`**:
    - Added `buildAuthor` (type: `string`)
    - Added `buildName` (type: `string`)

---

## Version 1.9 to 1.10

### New Endpoints (2)

| Method | Path | Description |
|---|---|---|
| `GET` | `/v1/dp/apps/{appId}/instances` | Get app instances |
| `GET` | `/v1/dp/builds/{buildId}/info` | Get detailed info about the build |

### New Schemas (2)

- **`AppInstancesResponse`** - 4 properties: `instanceName`, `namespace`, `podIP`, `status`
- **`BuildInfoForListBuilds`** - 8 properties: `buildId`, `connectorsUsed`, `createdBy`, `createdDate`, `flogoBaseVersion`, `name`, `nonDPBuild`, `tags`

### Schema Changes

Major restructuring of the `BuildInfo` schema:

- **`BuildInfo`**:
    - Added: `appDescription`, `appName`, `appType`, `appVersion`, `author`, `baseDeployment` (object), `buildName`, `contribMap`, `contribs` (array), `created` (integer), `dataPlaneId`, `enableServiceMesh` (boolean), `flogoApp` (object), `instanceId`, `portMappings` (array), `status` (object), `systemProperties` (array), `userContribs` (array)
    - Removed: `connectorsUsed` (array), `createdBy` (string), `createdDate` (integer), `flogoBaseVersion` (string), `name` (string)

The removed fields from `BuildInfo` were moved to the new `BuildInfoForListBuilds` schema, indicating a separation of concerns between detailed build info and list views.

---

## Version 1.10 to 1.11

No significant changes. Minor cosmetic/formatting updates only.

---

## Version 1.11 to 1.12

### New Endpoints (1)

| Method | Path | Description |
|---|---|---|
| `GET` | `/v1/dp/namespaces` | List Namespaces from the dataplane |

### New Schemas (1)

- **`NamespaceInfo`** - 1 property: `namespace`

---

## Version 1.12 to 1.13

No significant changes. Minor cosmetic/formatting updates only.

---

## Version 1.13 to 1.14

### New Endpoints (1)

| Method | Path | Description |
|---|---|---|
| `GET` | `/v1/dp/builds/{buildId}/status` | Get build status |

### Modified Endpoints (1)

- `POST /v1/dp/builds` - responses modified

---

## Version 1.14 to 1.15

### Schema Changes (3)

Gateway support and network policy updates:

- **`AppEndpointsResponse`**:
    - Added `gatewayControllerName` (type: `string`)
    - Added `gatewayName` (type: `string`)
    - Added `gatewayNamespace` (type: `string`)
    - Added `gatewaySectionName` (type: `string`)
    - Added `resourceInstanceName` (type: `string`)
- **`EgressNetworkPolicies`**:
    - Added `clusterEgress` (type: `string`)
    - Added `databaseEgress` (type: `string`)
    - Added `msgInfra` (type: `string`)
    - Added `proxyEgress` (type: `string`)
    - Added `userApps` (type: `string`)
- **`MakePublicEndpointRequest`**:
    - Added `gatewayControllerName` (type: `string`)
    - Added `gatewayHostName` (type: `string`)
    - Added `gatewayName` (type: `string`)
    - Added `gatewayNamespace` (type: `string`)
    - Added `gatewaySectionName` (type: `string`)
    - Added `resourceInstanceName` (type: `string`)

### Modified Endpoints (7)

- `GET /v1/dp/apps` - parameters modified
- `GET /v1/dp/apps/{appId}/endpoints` - summary and description changed
- `DELETE /v1/dp/apps/{appId}/endpoints/public` - summary and description changed
- `POST /v1/dp/apps/{appId}/endpoints/public` - summary, request body, and description modified
- `POST /v1/dp/builds` - parameters modified
- `DELETE /v2/dp/apps/{appId}/endpoints/public` - summary and description changed
- `POST /v2/dp/apps/{appId}/endpoints/public` - summary, request body, and description modified

---

## Version 1.15 to 1.16

### Modified Endpoints (1)

- `POST /v1/dp/builds` - request body modified: added optional `request` payload with the following structure:
    - `buildName` (type: `string`) - e.g., `"my-app-build"`
    - `dependencies` (type: `array`) - list of objects with `id`, `name`, and `version` fields
    - `tags` (type: `array`) - list of string tags
