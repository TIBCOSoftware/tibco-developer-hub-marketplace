# Control Plane API Changelog

Complete changelog for the TIBCO Platform Control Plane API from version 1.7 through 1.20.

---

## Version 1.19 → 1.20

**API Version:** 1.20.0

This release adds **OAuth2 client registration and revocation** to the Identity Management endpoints, a **Redis resource type** for data plane resource instances, and a fuller **team update** operation. It also drops the retired `SB` capability from the provisioning surface.

### New Endpoints

| Method | Path | Operation ID | Description |
|--------|------|--------------|-------------|
| POST | `/idm/v1/oauth2/clients` |  | Register an OAuth2 client and return the generated client credentials |
| DELETE | `/idm/v1/oauth2/clients/{clientId}` |  | Revoke and delete a registered OAuth2 client |

Registration takes `client_name`, `token_endpoint_auth_method` (`client_secret_post` or `client_secret_basic`) and `scope` as `application/x-www-form-urlencoded`, and returns an `OAuth2Client`. The `client_id` and `client_secret` it returns are exactly what the existing `/idm/v1/oauth2/token` client-credentials flow consumes, so a client can now be provisioned end to end through the API. Revocation is immediate and invalidates every access token already issued to the client; it returns `204` with no content.

### Removed Endpoints

None.

### New Schemas

- **resourceRedisConfigSchema** — Redis connection details for a data plane resource instance: `name`, `host` and `password` are required, plus optional `port`, `db` and a `tls` string enum (`"true"` / `"false"`). Marked `x-since: 1.19.0`.

### Removed Schemas

- **sbCapabilityProvisionSchema** — removed together with the `SB` capability enum value and its provisioning examples (see Modified Endpoints).

### Modified Endpoints

- **POST `/cp/api/v1/data-planes/{dataPlaneId}/resources/instances/{type}`** — new `REDIS_CONFIG` resource type added to the `type` enum, the capability version map (`1.19.0`), the request-body discriminator mapping and the `oneOf` list, with a `REDIS_CONFIG_EXAMPLE` request example.
- **POST `/cp/api/v1/data-planes/{dataPlaneId}/capabilities/{capabilityId}`** and **PUT `.../capabilities/{capabilityInstanceId}/update`** — the `SB` capability value, its version-map entry and its `SB_EXAMPLE` provisioning example are gone, and the discriminator mapping and `oneOf` list no longer reference `sbCapabilityProvisionSchema`.
- **PUT `/cp/api/v1/teams/{teamId}`** — now also updates the team's member `users` (email addresses) and sub-`teams` (team IDs), not just name and description, and returns the full `teamResponse` (`teamId`, `name`, `description`) instead of the previous bare `{ "message": "..." }` object. **This is a response-shape change for existing clients.**

### Notes

- Endpoint count rises from 63 to 65; schema count stays at 86 — one schema added, one removed.

---

## Version 1.18 → 1.19

**API Version:** 1.19.0

This release introduces **activation license management** at both the subscription and data plane level, **hierarchical teams** (sub-teams and members managed directly on the team), **namespace- and BW5/BW6-scoped permissions**, a capability **refresh** operation, and provisioning support for additional capability types with an optional `provisioner-recipe` passthrough.

### New Endpoints

| Method | Path | Operation ID | Description |
|--------|------|--------------|-------------|
| GET | `/cp/api/v1/groups` |  | List IDP group names |
| GET | `/cp/api/v1/licenses/all` |  | Get all licenses |
| GET | `/cp/api/v1/subscription/license` |  | Get subscription license |
| PUT | `/cp/api/v1/subscription/license` |  | Upload subscription license |
| DELETE | `/cp/api/v1/subscription/license` |  | Delete subscription license |
| GET | `/cp/api/v1/data-planes/{dataPlaneId}/license` |  | Get data plane license |
| PUT | `/cp/api/v1/data-planes/{dataPlaneId}/license` |  | Upload data plane license |
| DELETE | `/cp/api/v1/data-planes/{dataPlaneId}/license` |  | Delete data plane license |
| GET | `/cp/api/v1/data-planes/{dataPlaneId}/license-metadata` |  | Get license metadata for a data plane |
| PUT | `/cp/api/v1/data-planes/{dataPlaneId}/capabilities/{capabilityInstanceId}/refresh` |  | Refresh Capability in a Data Plane |

### Removed Endpoints

None.

### New Schemas

- asCapabilityProvisionSchema
- emsCapabilityProvisionSchema
- sbCapabilityProvisionSchema
- licenseDetails
- teamDetailResponse

### Modified Schemas

- **Error** — reworked error format: `code`, `message`, `details`, `status` and `context` replaced by `errorCode`, `errorMsg` and `contextAttributes`, with common error codes now documented (`ATMOSPHERE-11001` bad request, `ATMOSPHERE-11002` internal error, `ATMOSPHERE-11006` forbidden, `PLATFORM-PE-001` pengine validation error).
- **bw5ceCapabilityProvisionSchema / bwceCapabilityProvisionSchema / flogoCapabilityProvisionSchema** — new optional `provisioner-recipe` object; when provided (non-empty) it is forwarded as-is to the concrete-recipe API.
- **Permission payloads** (`inviteUserRequest`, `updateGroupsPermissionsRequest`, `updateTeamsPermissionsRequest`, `updateUserPermissionsRequest`, `groupsPermissionsResponse`, `teamPermissionDetail`, `userPermissionDetail`) — two new scoping fields: `namespaceId` (namespace-level permissions within a data plane) and `resourceInstanceId` (specific BW5 domain ID or BW6 agent ID when `instanceId` is `bw5` or `bw6`). `roleId` is now a documented enum: `OWNER`, `IDP_MANAGER`, `TEAM_ADMIN`, `BROWSE_ASSIGNMENTS` (CP-level) and `PLATFORM_OPS`, `DEV_OPS`, `CAPABILITY_ADMIN`, `CAPABILITY_USER`, `FIN_OPS` (DP-level).
- **teamRequest** — new `teams` (sub-team IDs) and `users` (member email addresses) arrays: teams can now be created and updated with nested sub-teams and members in one call.
- **listUsersResponse** — `roleIds` removed; new `status` field per user (`invited` | `active`), plus `firstName`, `lastName` and a `numberOfRecords` counter in the response example.
- **userPermissionDetail** — new `dataplaneName` field (resolved display name of the data plane).

### Modified Endpoints

- **GET /cp/api/v1/teams/{teamId}** — now returns the new `teamDetailResponse` (full team detail including `users`, sub-`teams`, `immediatelyContainedIn` and `indirectlyContainedIn`).
- **PUT /cp/api/v1/members** — richer request examples covering CP roles, DP roles, BW5/BW6 product permissions and namespace scoping.
- **POST /cp/api/v1/data-planes/{dataPlaneId}/capabilities/{capabilityId}**, **PUT .../capabilities/{capabilityInstanceId}/update** and **PUT .../capabilities/{capabilityInstanceId}/upgrade** — accept the new optional `provisioner-recipe` field; provision/update examples added for the new capability provision schemas.
- **POST /cp/api/v1/users-with-temp-passwords** — behavior clarified: users are created as active immediately with `TEAM_ADMIN` permission, no invitation email is sent, and the user must change the temporary password on first login.
- **GET /cp/api/v1/users** — response example updated with `numberOfRecords` and per-user `firstName`, `lastName` and `status`.

### Notes

- All 1.18 endpoints remain unchanged in shape — this release is purely additive at the endpoint level.
- Endpoint count grew from 53 to 63; schema count grew from 81 to 86.

---

## Version 1.17 → 1.18

**API Version:** 1.18.0

This release introduces full **team and user management** and **fine-grained permissions** to the Control Plane API, alongside new data plane namespace and route-resource operations.

### New Endpoints

| Method | Path | Operation ID | Description |
|--------|------|--------------|-------------|
| POST | `/cp/api/v1/data-planes/{dataPlaneId}/namespace-commands` |  | Generate commands to create and register a namespace |
| PUT | `/cp/api/v1/data-planes/{dataPlaneId}/route-resource` |  | Switch Route resource for Control Tower Data Plane |
| GET | `/cp/api/v1/groups/permissions` |  | Get groups permissions |
| POST | `/cp/api/v1/groups/permissions` |  | Update groups permissions |
| PUT | `/cp/api/v1/members` |  | Invite users to CP subscription |
| GET | `/cp/api/v1/teams` |  | List teams |
| POST | `/cp/api/v1/teams` |  | Create team |
| POST | `/cp/api/v1/teams/permissions` |  | Update teams permissions |
| GET | `/cp/api/v1/teams/{teamId}` |  | Get team |
| PUT | `/cp/api/v1/teams/{teamId}` |  | Update team |
| DELETE | `/cp/api/v1/teams/{teamId}` |  | Delete team |
| GET | `/cp/api/v1/teams/{teamId}/permissions` |  | Get team permissions |
| GET | `/cp/api/v1/users` |  | List users in CP subscription |
| POST | `/cp/api/v1/users/permissions` |  | Update user permissions (full-replace) |
| DELETE | `/cp/api/v1/users/{userEntityId}` |  | Remove user from CP subscription |
| GET | `/cp/api/v1/users/{userEntityId}/permissions` |  | Get user permissions |

### Removed Endpoints

| Method | Path | Description |
|--------|------|-------------|
| POST | `/cp/api/v1/data-planes/{dataPlaneId}/namespace` | Generate commands to create and register a namespace (replaced by `.../namespace-commands`) |

### New Schemas

- apiResponseNamespaceCommands
- groupsPermissionsResponse
- inviteUserRequest
- listTeamsResponse
- listUsersResponse
- memberOperationResponse
- resourceGatewayControllerSchema
- resourceNamespaceSchema
- routeResourceSwitchPayload
- teamPermissionDetail
- teamRequest
- teamResponse
- updateGroupsPermissionsRequest
- updateGroupsPermissionsResponse
- updateTeamsPermissionsRequest
- updateUserPermissionsRequest
- userPermissionDetail

### Notes

- The data plane namespace endpoint was renamed from `POST /namespace` to `POST /namespace-commands`; the operation (generate commands to create and register a namespace) is unchanged.
- Endpoint count grew from 38 to 53; schema count grew from 64 to 81.

---

## Version 1.16 → 1.17

**API Version:** 1.17.0

### New Endpoints

| Method | Path | Operation ID | Description |
|--------|------|--------------|-------------|
| PUT | `/cp/api/v1/data-planes/{dataPlaneId}/upgrade` |  | Upgrade a Data Plane |
| PUT | `/cp/api/v1/data-planes/{dataPlaneId}/capabilities/{capabilityInstanceId}/upgrade` |  | Upgrade Capability in a Data Plane |
| PUT | `/cp/api/v1/data-planes/{dataPlaneId}/capabilities/{capabilityInstanceId}/update` |  | Update Capability in a Data Plane |

### Modified Endpoints

The following endpoints have been modified:

- **POST /cp/api/v1/data-planes/k8s**
- **POST /cp/api/v1/data-planes/control-tower**
- **POST /cp/api/v1/data-planes/{dataPlaneId}/namespace**
- **GET /cp/api/v1/data-planes**
- **GET /cp/api/v1/data-planes/status**
- **GET /cp/api/v1/data-planes/{dataPlaneId}/status**
- **GET /cp/api/v1/data-planes/{dataPlaneId}/commands/{operation}**
- **GET /cp/api/v1/data-planes/{dataPlaneId}**
- **PUT /cp/api/v1/data-planes/{dataPlaneId}**
- **DELETE /cp/api/v1/data-planes/{dataPlaneId}**
- **PUT /cp/api/v1/data-planes/{dataPlaneId}/resource-association**
- **GET /cp/api/v1/data-planes/{dataPlaneId}/apps**
- **GET /cp/api/v1/apps**
- **GET /cp/api/v1/apps/{appId}**
- **GET /cp/api/v1/apps/{appId}/status**
- **POST /cp/api/v1/resources/instances/{type}**
- **POST /cp/api/v1/data-planes/{dataPlaneId}/resources/instances/{type}**
- **GET /cp/api/v1/resources**
- **GET /cp/api/v1/resources/instances**
- **GET /cp/api/v1/data-planes/{dataPlaneId}/resources/instances**
- **GET /cp/api/v1/resources/instances/{resourceInstanceId}**
- **DELETE /cp/api/v1/resources/instances/{resourceInstanceId}**
- **PUT /cp/api/v1/resources/instances/{resourceInstanceId}**
- **DELETE /cp/api/v1/data-planes/{dataPlaneId}/resources/instances/{resourceInstanceId}**
- **POST /cp/api/v1/data-planes/{dataPlaneId}/capabilities/{capabilityId}**
- **GET /cp/api/v1/capabilities**
- **GET /cp/api/v1/capabilities/instances**
- **GET /cp/api/v1/data-planes/{dataPlaneId}/capabilities/instances**
- **GET /cp/api/v1/data-planes/{dataPlaneId}/capabilities-instances/status**
- **GET /cp/api/v1/data-planes/{dataPlaneId}/capabilities/instances/{capabilityInstanceId}/status**
- **GET /cp/api/v1/data-planes/{dataPlaneId}/capabilities/instances/{capabilityInstanceId}**
- **DELETE /cp/api/v1/data-planes/{dataPlaneId}/capabilities/instances/{capabilityInstanceId}**
- **POST /cp/api/v1/users-with-temp-passwords**
- **POST /idm/v1/oauth2/tokens/operations/revoke**

### New Schemas

- dataPlaneUpgradeResponse
- webhookReceiverSchema

---

## Version 1.7 to 1.8

This was the most significant transition, essentially a complete API redesign with a new URL structure standardized under `/cp/api/v1/`.

**Title changed:** `TIBCO Platform API - Direct` to `TIBCO Platform APIs`

**New Tags:** Apps, Capabilities, Data Planes, Resources

### New Endpoints (25)

| Method | Path | Description |
|---|---|---|
| `GET` | `/cp/api/v1/apps/{appId}` | Get app details by ID on a subscription |
| `GET` | `/cp/api/v1/capabilities/instances` | Get all capability instances details in a subscription |
| `POST` | `/cp/api/v1/data-planes/control-tower` | Register a control-tower data plane |
| `POST` | `/cp/api/v1/data-planes/k8s` | Register a k8s data plane |
| `GET` | `/cp/api/v1/data-planes/status` | Get Data Planes status |
| `DELETE` | `/cp/api/v1/data-planes/{dataPlaneId}` | Unregister a data plane |
| `GET` | `/cp/api/v1/data-planes/{dataPlaneId}` | Get data plane details |
| `PUT` | `/cp/api/v1/data-planes/{dataPlaneId}` | Update data plane details (name, description, tags) |
| `GET` | `/cp/api/v1/data-planes/{dataPlaneId}/apps` | Get app details deployed on a data plane |
| `GET` | `/cp/api/v1/data-planes/{dataPlaneId}/capabilities-instances/status` | Get capabilities instances status |
| `GET` | `/cp/api/v1/data-planes/{dataPlaneId}/capabilities/instances` | Get all capability instances details in a data plane |
| `DELETE` | `/cp/api/v1/data-planes/{dataPlaneId}/capabilities/instances/{capabilityInstanceId}` | De-provision a capability instance |
| `GET` | `/cp/api/v1/data-planes/{dataPlaneId}/capabilities/instances/{capabilityInstanceId}` | Get capability instance details |
| `GET` | `/cp/api/v1/data-planes/{dataPlaneId}/capabilities/instances/{capabilityInstanceId}/status` | Get capability instance status |
| `POST` | `/cp/api/v1/data-planes/{dataPlaneId}/capabilities/{capabilityId}` | Provision a Capability |
| `GET` | `/cp/api/v1/data-planes/{dataPlaneId}/commands/{operation}` | Get Data Plane commands |
| `GET` | `/cp/api/v1/data-planes/{dataPlaneId}/resources/instances` | Get Data Plane Resource Instances |
| `DELETE` | `/cp/api/v1/data-planes/{dataPlaneId}/resources/instances/{resourceInstanceId}` | Delete Data Plane Resource Instance |
| `POST` | `/cp/api/v1/data-planes/{dataPlaneId}/resources/instances/{type}` | Create Data plane level resource instance |
| `GET` | `/cp/api/v1/data-planes/{dataPlaneId}/status` | Get Data Plane status |
| `GET` | `/cp/api/v1/resources/instances` | Get All Resource Instances |
| `DELETE` | `/cp/api/v1/resources/instances/{resourceInstanceId}` | Delete Resource Instance at subscription level |
| `GET` | `/cp/api/v1/resources/instances/{resourceInstanceId}` | Get Resource Instance details |
| `PUT` | `/cp/api/v1/resources/instances/{resourceInstanceId}` | Update resource instance |
| `POST` | `/cp/api/v1/resources/instances/{type}` | Create a subscription level resource instance |

### Removed Endpoints (16)

| Method | Path |
|---|---|
| `GET` | `/cp/api/v1/capability-instances` |
| `GET` | `/cp/api/v1/data-planes/{dp_id}` |
| `GET` | `/cp/api/v1/resource-instances` |
| `GET` | `/cp/v1/account/users` |
| `GET` | `/cp/v1/accounts/user/flat-permissions/{userEmail}` |
| `GET` | `/cp/v1/capabilities-metadata` |
| `POST` | `/cp/v1/data-planes` |
| `GET` | `/cp/v1/data-planes-status` |
| `DELETE` | `/cp/v1/data-planes/{DPId}` |
| `GET` | `/cp/v1/data-planes/{DPNamespace}/commands/delete` |
| `DELETE` | `/cp/v1/data-planes/{dpId}/capabilities/{capabilityInstanceId}` |
| `POST` | `/cp/v1/resource-instances` |
| `GET` | `/cp/v1/whoami` |
| `GET` | `/public/v1/oauth2/userinfo` |
| `GET` | `/tp-cp-ws/v1/data-planes/{dp_id}/app-details` |
| `GET` | `/tp-cp-ws/v1/resource-instances-details` |

### New Schemas (49)

Major new schemas introduced:

- **`AppDetails`** - 26 properties covering app metadata, capability info, and deployment details
- **`apiResponseDataPlaneDetails`** - 13 properties for data plane configuration
- **`registerControlTowerDataPlanePayload`** - 11 properties for registering control-tower data planes
- **`registerK8sDataPlanePayload`** - 11 properties for registering Kubernetes data planes
- **`tibcoHubCapabilityProvisionSchema`** - 6 properties for Developer Hub capability provisioning
- **`bwceCapabilityProvisionSchema`** - 6 properties for BWCE capability provisioning
- **`flogoCapabilityProvisionSchema`** - 4 properties for Flogo capability provisioning

Resource schemas: `resourceBWAgentSchema`, `resourceDatabaseConfigSchema`, `resourceHawkDomainSchema`, `resourceIngressControllerSchema`, `resourceMetricsServerExporterPrometheusSchema`, `resourceO11YV3Schema`, `resourceStorageClassSchema`

Observability schemas: `alertRuleSchema`, `elasticSearchSchema`, `kafkaSchema`, `otlpSchema`, `emailReceiverSchema`

### Removed Schemas (1)

- `Object`

### Modified Endpoints (4)

- `GET /cp/api/v1/apps` - summary, responses modified
- `GET /cp/api/v1/capabilities` - summary, responses, description modified
- `GET /cp/api/v1/data-planes` - summary, parameters, responses, description modified
- `GET /cp/api/v1/resources` - summary, responses, description modified

---

## Version 1.8 to 1.9

### Schema Changes

- **`bwceCapabilityProvisionSchema`**:
    - Added `auto-discovery-config` (type: `object`)
    - Added `auto-discovery-enabled` (type: `boolean`)

### Modified Endpoints (2)

- `POST /cp/api/v1/data-planes/{dataPlaneId}/capabilities/{capabilityId}` - request body modified
- `POST /cp/api/v1/data-planes/{dataPlaneId}/resources/instances/{type}` - request body modified

---

## Version 1.9 to 1.10

**Title changed:** `TIBCO Platform APIs` to `Control Plane APIs`

### New Endpoints (3)

| Method | Path | Description |
|---|---|---|
| `GET` | `/cp/api/v1/apps/{appId}/status` | Get app status from Monitoring |
| `POST` | `/cp/api/v1/data-planes/{dataPlaneId}/namespace` | Create and register a namespace |
| `PUT` | `/cp/api/v1/data-planes/{dataPlaneId}/resource-association` | Link resource instances to a Data Plane |

### New Schemas (6)

- **`activationServerSchema`** - 3 properties: description, name, url
- **`apiResponseAppStatus`** - 4 properties: id, instance, name, status
- **`apiResponseRegisterDataPlaneNamespace`** - 3 properties: context, response, status
- **`bw5ceCapabilityProvisionSchema`** - 4 properties for BW5 CE capability provisioning
- **`dataPlaneLinkResourcePayload`** - 3 properties: operation, resource-instance-id, resource-type
- **`registerDataPlaneNamespacePayload`** - 1 property: namespace

### Schema Changes

- **`alertRuleSchema`**: Added `severity` (type: `string`)
- **`apiResponseAppDetails`**: Added `status` (type: `string`)

### Modified Endpoints (7)

- `GET /cp/api/v1/apps` - parameters modified
- `GET /cp/api/v1/capabilities` - responses modified
- `GET /cp/api/v1/data-planes/{dataPlaneId}/apps` - parameters modified
- `POST /cp/api/v1/data-planes/{dataPlaneId}/capabilities/{capabilityId}` - parameters and request body modified
- `POST /cp/api/v1/data-planes/{dataPlaneId}/resources/instances/{type}` - parameters and request body modified
- `PUT /cp/api/v1/resources/instances/{resourceInstanceId}` - parameters and request body modified
- `POST /cp/api/v1/resources/instances/{type}` - parameters and request body modified

---

## Version 1.10 to 1.11

### New Endpoints (1)

| Method | Path | Description |
|---|---|---|
| `POST` | `/cp/api/v1/users-with-temp-passwords` | Add users to CP subscription |

**New Tags:** Users

### New Schemas (2)

- **`apiResponseAddUsersToSubscription`** - 3 properties: context, response, status
- **`userWithTempPasswordRequest`**

### Modified Endpoints (32)

All 32 existing endpoints received summary and description updates. This was a major documentation refresh across the entire API surface. Key changes:

- Summary text standardized across all endpoints
- Description text expanded with more detail
- Several endpoints received parameter modifications
- Data plane registration endpoints (`control-tower`, `k8s`) received response and request body updates
- Capability instance endpoints received parameter and response updates

---

## Version 1.11 to 1.12

### Schema Changes

- **`alertRuleSchema`**: Removed `enabled` field (was type: `boolean`)

### Modified Endpoints (1)

- `POST /cp/api/v1/data-planes/{dataPlaneId}/capabilities/{capabilityId}` - request body modified

---

## Version 1.12 to 1.13

### Modified Endpoints (2)

- `DELETE /cp/api/v1/data-planes/{dataPlaneId}/capabilities/instances/{capabilityInstanceId}` - parameters modified
- `GET /cp/api/v1/data-planes/{dataPlaneId}/commands/{operation}` - parameters modified

---

## Version 1.13 to 1.14

### Schema Changes

- **`bwceCapabilityProvisionSchema`**: Added `non-tokenized-properties-enabled` (type: `boolean`)

### Modified Endpoints (3)

- `POST /cp/api/v1/data-planes/{dataPlaneId}/capabilities/{capabilityId}` - request body modified
- `POST /cp/api/v1/data-planes/{dataPlaneId}/namespace` - summary changed, request body modified
- `POST /cp/api/v1/data-planes/{dataPlaneId}/resources/instances/{type}` - request body modified

---

## Version 1.14 to 1.15

### New Endpoints (2)

| Method | Path | Description |
|---|---|---|
| `POST` | `/idm/v1/oauth2/token` | OAuth2 Token endpoint |
| `POST` | `/idm/v1/oauth2/tokens/operations/revoke` | Revokes OAuth2 access-token |

**New Tags:** OAuth2

### New Schemas (5)

- **`AccessTokenResponse`** - 4 properties: access_token, expires_in, scope, token_type
- **`IdmError`** - 3 properties: contextAttributes, errorCode, errorMsg
- **`IdmMessage`** - 1 property: message
- **`OAuth2Client`** - 4 properties: client_id, client_secret, scope, token_endpoint_auth_method
- **`RevokeInitialAccessTokenRequest`** - 2 properties: comment, ids

### Schema Changes (6)

Gateway resource support added across all capability provisioning schemas:

- **`bw5ceCapabilityProvisionSchema`**: Added `gateway-resource-instance-id` (type: `string`)
- **`bwceCapabilityProvisionSchema`**: Added `gateway-resource-instance-id` (type: `string`)
- **`flogoCapabilityProvisionSchema`**: Added `gateway-resource-instance-id` (type: `string`)
- **`tibcoHubCapabilityProvisionSchema`**: Added `gateway-resource-instance-id` (type: `string`)

Data plane registration schemas updated:

- **`registerControlTowerDataPlanePayload`**: Added `connection-details` (type: `object`)
- **`registerK8sDataPlanePayload`**: Added `connection-details` (type: `object`)

### Modified Endpoints (1)

- `POST /cp/api/v1/data-planes/{dataPlaneId}/capabilities/{capabilityId}` - request body modified

---

## Version 1.15 to 1.16

No changes. Version bump only (1.15.0 to 1.16.0).
