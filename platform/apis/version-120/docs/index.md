# TIBCO Platform APIs 1.20 - Changelog

This document provides a comprehensive changelog of the TIBCO Platform APIs across all versions, from 1.7 through the current release 1.20. Use the navigation to view detailed changes for each individual API, and [MCP Servers](mcp-servers.md) for the Model Context Protocol servers published with this release.

## API Overview

The TIBCO Platform exposes five main API categories:

- **Control Plane API (Platform)** - Core platform management: data planes, capabilities, resources, apps, and users
- **BWCE Capability API** - TIBCO BusinessWorks 6 (Containers) capability management on data planes
- **BW5 CE Capability API** - TIBCO BusinessWorks 5 (Containers) capability management (introduced in 1.10)
- **Flogo Capability API** - TIBCO Flogo capability management on data planes
- **Developer Hub API** - The catalog API of the TIBCO Developer Hub: entities, locations, facets and validation

Alongside the REST APIs, the platform publishes five **MCP servers** whose tool definitions ship with this entry for the first time in 1.20. See [MCP Servers](mcp-servers.md).

## API Version Summary

| Platform Version | Control Plane | BWCE Capability | BW5 CE Capability | Flogo Capability |
|---|---|---|---|---|
| 1.7 | 1.0 | 1.7.0 | N/A | 1.7.0 |
| 1.8 | 1.0 | 1.8.0 | N/A | 1.8.0 |
| 1.9 | 1.0 | 1.9.0 | N/A | 1.9.0 |
| 1.10 | 1.10.0 | 1.10.0 | 1.10.0 | 1.10.0 |
| 1.11 | 1.11.0 | 1.11.0 | 1.11.0 | 1.11.0 |
| 1.12 | 1.12.0 | 1.12.0 | 1.12.0 | 1.12.0 |
| 1.13 | 1.13.0 | 1.13.0 | 1.13.0 | 1.13.0 |
| 1.14 | 1.14.0 | 1.14.0 | 1.14.0 | 1.14.0 |
| 1.15 | 1.15.0 | 1.15.0 | 1.15.0 | 1.15.0 |
| 1.16 | 1.16.0 | 1.16.0 | 1.16.0 | 1.16.0 |
| 1.17 | 1.17.0 | 1.17.0 | 1.17.0 | 1.17.0 |
| 1.18 | 1.18.0 | 1.18.0 | 1.18.0 | 1.18.0 |
| 1.19 | 1.19.0 | 1.19.0 | 1.19.0 | 1.19.0 |
| 1.20 | 1.20.0 | 1.20.0 | 1.20.0 | 1.20.0 |

## Endpoint Count per Version

| Platform Version | Control Plane | BWCE | BW5 CE | Flogo |
|---|---|---|---|---|
| 1.7 | 20 | 63 | N/A | 47 |
| 1.8 | 29 | 66 | N/A | 48 |
| 1.9 | 29 | 67 | N/A | 48 |
| 1.10 | 32 | 67 | 51 | 50 |
| 1.11 | 33 | 67 | 51 | 50 |
| 1.12 | 33 | 67 | 51 | 51 |
| 1.13 | 33 | 67 | 51 | 51 |
| 1.14 | 33 | 67 | 51 | 52 |
| 1.15 | 35 | 67 | 51 | 52 |
| 1.16 | 35 | 67 | 52 | 52 |
| 1.17 | 38 | 67 | 52 | 52 |
| 1.18 | 53 | 67 | 52 | 52 |
| 1.19 | 63 | 74 | 53 | 53 |
| 1.20 | 65 | 75 | 53 | 54 |

## Schema Count per Version

| Platform Version | Control Plane | BWCE | BW5 CE | Flogo |
|---|---|---|---|---|
| 1.7 | 1 | 63 | N/A | 42 |
| 1.8 | 49 | 63 | N/A | 42 |
| 1.9 | 49 | 63 | N/A | 42 |
| 1.10 | 55 | 63 | 63 | 44 |
| 1.11 | 57 | 63 | 63 | 44 |
| 1.12 | 57 | 63 | 63 | 45 |
| 1.13 | 57 | 63 | 64 | 45 |
| 1.14 | 57 | 63 | 64 | 45 |
| 1.15 | 62 | 63 | 64 | 45 |
| 1.16 | 62 | 63 | 64 | 45 |
| 1.17 | 64 | 64 | 64 | 45 |
| 1.18 | 81 | 64 | 64 | 45 |
| 1.19 | 86 | 65 | 65 | 46 |
| 1.20 | 86 | 66 | 66 | 47 |

## Developer Hub API

The Developer Hub catalog API is the OpenAPI document published by the Backstage catalog backend the Hub is built on, so it changes when the Hub's Backstage version changes rather than with every platform release.

| Platform Version | Spec published with the entry | Endpoints | Schemas |
|---|---|---|---|
| 1.7 - 1.9 | `backstage-api.yaml` - abbreviated catalog spec | 16 | 23 |
| 1.10 - 1.18 | `backstage-api-1.41.1.yaml` - Backstage 1.41.1 | 16 | 23 |
| 1.19 - 1.20 | `backstage-api-1.51.0.yaml` - **Backstage 1.51.0** | **20** | **24** |

## MCP Servers

The platform's Model Context Protocol servers are published with the entry from 1.20 onwards. They
are not OpenAPI documents: each definition is the server's advertised tool list, captured at
release time. See [MCP Servers](mcp-servers.md) for the full tool inventory.

| Server | Endpoint | Catalog entity | Tools |
|---|---|---|---|
| Control Plane | `/cp/mcp` | `tibco-platform-mcp-120` | 26 |
| Platform Observability | `/o11y/mcp` | `observability-mcp-120` | 17 |
| TIBCO BusinessWorks | `/bw/mcp` | `bw-mcp-120` | 25 |
| TIBCO Flogo | `/flogo/mcp` | `flogo-mcp-120` | 19 |
| TIBCO Developer Hub | `/api/mcp-actions/v1` | `tibco-developer-hub-mcp-120` | 12 |

## API First Appearance

| API | First Appeared In |
|---|---|
| Control Plane API | 1.7 (earliest version tracked) |
| BWCE Capability API | 1.7 (earliest version tracked) |
| BW5 CE Capability API | **1.10** |
| Flogo Capability API | 1.7 (earliest version tracked) |
| Developer Hub API | 1.7 (earliest version tracked) |

## Key Highlights

- **Version 1.7 to 1.8** was the biggest transition for the Control Plane API, essentially a complete API redesign: 16 old endpoints removed, 25 new ones added, 49 new schemas introduced, and the URL structure was standardized under `/cp/api/v1/`.
- **Version 1.10** introduced the **BW5 CE Capability API** with 51 endpoints and 63 schemas, closely mirroring the BWCE API structure.
- **Version 1.15** was the second most significant release, introducing OAuth2 endpoints (`/idm/v1/oauth2/token` and revoke), gateway resource support across all capability provisioning schemas, and gateway-related schema fields across all three capability APIs.
- **Version 1.16** focused on connector catalog metadata fields (`displayVersion`, `endOfSupportDate`, `releaseDate`, etc.) in BWCE and BW5 CE APIs, and added a new custom base image endpoint for BW5 CE.
- **Version 1.18** is the largest Control Plane release since 1.8: full team and user management plus fine-grained permissions (16 new endpoints, 17 new schemas), the data plane `namespace` endpoint renamed to `namespace-commands`, and a new Control Tower `route-resource` switch. The capability APIs saw only minor changes — a new `viewOnly` field on `NamespaceInfo` across BWCE, BW5 CE, and Flogo, plus a new `SAPSolutions` connector value in Flogo.
- **Version 1.19** brings **activation license management** to the Control Plane API — upload, retrieve and delete licenses at the subscription and data plane level (10 new endpoints) — plus hierarchical teams (sub-teams and members on the team itself), namespace- and BW5/BW6-scoped permissions with a documented role enum, a capability refresh operation, and an optional `provisioner-recipe` passthrough on capability provision, update and upgrade.
- **Version 1.19** is also the first release in which all three capability APIs gain **build lifecycle management**: a shared `DELETE /v1/dp/builds` operation that tags, untags, cleans up or (BW6 and BW5 CE) bulk-deletes application builds, backed by a new `TagAndCleanupResponse` schema. BW6 additionally gets a full **autodiscovery app lifecycle** (delete, scale, get/apply app YAML for non-Helm apps; delete release and update chart values for Helm apps — 7 new endpoints in total), and BW5 CE exposes the public Swagger URLs of a deployed app on `EndpointInfo`.
- **Version 1.19** also upgrades the **Developer Hub** from Backstage 1.41.1 to **1.51.0**: four new catalog operations (POST twins of the entity-query and entity-facets endpoints for filters too large for a URL, a paginated location query, and location update), an idempotent `onConflict=refresh` on location registration, a `totalItems=exclude` opt-out of the expensive count on large catalogs, and a required `entityRef` on `Location`. The spec also moves to the OpenAPI 3.1 dialect, which is the only change that can break an existing consumer.
- **Version 1.20** adds **OAuth2 client lifecycle** to the Control Plane API — a client can now be registered and revoked through `/idm/v1/oauth2/clients` and feed straight into the existing client-credentials token flow, no Control Plane UI needed — plus a `REDIS_CONFIG` data plane resource type, and a `PUT /cp/api/v1/teams/{teamId}` that updates members and sub-teams and returns the full team object instead of a bare message (the one response-shape change in this release). The retired `SB` capability and its provisioning schema are gone from the API surface.
- **Version 1.20** also gives all three capability APIs **per-container instance status**: `AppInstancesResponse` gains a `containerStatuses` array (a new `ContainerStatusInfo` schema covering init and regular containers) and a `ready` flag, with the old `status` field now explicitly documented as the raw Kubernetes pod phase that may not reflect readiness. On top of that shared change, BW6 gets an **OSGi console** endpoint for running commands against a live instance, and Flogo gets **build log streaming** (`follow`/`tail`) with queue and build timing on `BuildStatusResponse`. BW5 CE gains no new endpoints this release, and the Developer Hub stays on Backstage 1.51.0.
- **Version 1.20** is the first release to publish the platform's **MCP server** definitions with this entry: five servers exposing 99 tools in total, covering the Control Plane, observability, BusinessWorks, Flogo and the Developer Hub itself. They are JSON-RPC over MCP's Streamable HTTP transport rather than REST, so they are catalogued as `mcp-server` API entities carrying a captured tool list instead of an OpenAPI document.
