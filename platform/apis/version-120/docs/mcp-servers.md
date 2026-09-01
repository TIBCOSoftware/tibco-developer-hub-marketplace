# MCP Servers

The TIBCO Platform publishes a set of **Model Context Protocol (MCP)** servers that let an AI
assistant drive the platform directly — provisioning capabilities, inspecting applications,
querying metrics and running scaffolder templates — using the same permissions as the calling
user. This entry publishes their definitions so the tools each server exposes are visible in the
TIBCO Developer Hub catalog alongside the REST APIs.

These definitions are published with the marketplace entry for the first time in **1.20**.

## Servers

| Server | Endpoint | Catalog entity | Tools |
|--------|----------|----------------|-------|
| Control Plane | `/cp/mcp` | `tibco-platform-mcp-120` | 26 |
| Platform Observability | `/o11y/mcp` | `observability-mcp-120` | 17 |
| TIBCO BusinessWorks | `/bw/mcp` | `bw-mcp-120` | 25 |
| TIBCO Flogo | `/flogo/mcp` | `flogo-mcp-120` | 19 |
| TIBCO Developer Hub | `/api/mcp-actions/v1` | `tibco-developer-hub-mcp-120` | 12 |

Endpoints are given relative to the subscription host, which is where both the platform MCP
servers and the Developer Hub are served from.

## Transport

All of these servers speak **JSON-RPC 2.0 over MCP's Streamable HTTP transport**, not REST. There
is no downloadable specification document the way there is for the OpenAPI-based APIs: a client
discovers what a server offers at runtime.

Two consequences are worth knowing before you point a client at one:

- **Everything goes over `POST`.** A `GET` on the endpoint is the optional server-to-client SSE
  stream, and these servers do not offer it, so a `GET` correctly returns `405 Method Not Allowed`.
- **The tool list comes from a handshake**, not a single request: `initialize`, then a
  `notifications/initialized` notification, then `tools/list`, carrying the `Mcp-Session-Id` the
  server returns from `initialize`. The `Accept` header must name both `application/json` and
  `text/event-stream`, and a reply may come back SSE-framed even for a `POST`.

Requests are authenticated with the same bearer token as the Control Plane REST API.

All four platform servers negotiate protocol version `2025-06-18` and advertise the `tools` and
`logging` capabilities only — none currently exposes MCP resources or prompts, so the `resources`
and `prompts` collections in their definitions are empty.

## Control Plane MCP Server

`/cp/mcp` — 26 tools. Data plane, capability and resource lifecycle, plus alerting and
license queries. This is the server that provisions and upgrades capabilities.

| Tool | Purpose |
|------|---------|
| `createActivationServer` | Create an activation server resource at the subscription or dataplane level |
| `createBWAgent` | Create a BW agent resource instance to facilitate communication between control plane and on-prem bw agents |
| `createHawkDomainRV` | Create a hawk domain with RV transport |
| `createStorageClass` | Create a storage class to manage storage resources within a data plane |
| `deleteCapabilityInstanceInsideDataPlane` | Delete a capability instance inside a data plane |
| `deleteDataPlane` | Delete a data plane |
| `deleteResourceInstance` | Delete a specific resource instance with subscription scope or data plane scope |
| `getCapabilitiesMetadata` | Get metadata of supported capabilities and their available versions |
| `linkResourceInstancesToDataPlane` | Link resource instances to a data plane |
| `listApps` | List applications in a subscription with support of optional filters |
| `listCapabilityInstances` | List details of multiple capability instances in a subscription with support of optional filters |
| `listDataplanes` | List data planes in a subscription with support of optional filters |
| `listLicenseDetails` | Fetch license details for a subscription with optional scope filters |
| `listResourceInstances` | List details of multiple resource instances in a subscription with optional filters |
| `listResources` | List all possible resources available in platform and their supported types where applicable |
| `manageAlertRule` | Create or update an alert rule to monitor resources and send notifications |
| `manageCapabilityBW5CE` | Provision or update a bw5ce capability inside the given data plane |
| `manageCapabilityBWCE` | Provision or update a bwce capability inside the given data plane |
| `manageCapabilityFlogo` | Provision or update a flogo capability inside the given data plane |
| `manageCapabilityInstance` | Upgrade or refresh a capability instance inside the given data plane |
| `manageCapabilityTIBCOHUB` | Provision or update a TibcoHub capability inside the given data plane |
| `manageDataPlane` | Update or upgrade an existing data plane |
| `manageEmailReceiver` | Create or update an email receiver to be used for receiving alert notifications via an Email |
| `manageGatewayAPI` | Create or update a Gateway API resource at the data plane level |
| `manageIngressController` | Create or update an ingress controller to manage access to services of capabilities within a data plane |
| `manageWebhookReceiver` | Create or update a webhook receiver used for receiving alert notifications via HTTP webhook |

## Platform Observability MCP Server

`/o11y/mcp` — 17 tools. Metrics querying, threshold violation detection and dashboard
management. Several tools render charts server-side and return them as base64 images, so the
image data never has to be relayed back through the model.

| Tool | Purpose |
|------|---------|
| `analyze_application` | Get app health status and metrics in one call |
| `detect_violations` | Find apps exceeding thresholds for CPU, memory, execution time, or request rates |
| `query_metrics` | Query metrics with custom aggregations and time ranges |
| `detect_performance_degradation` | Detect period-over-period performance degradation using PromQL offset |
| `record_app_violation_event` | Record an application violation event on the Control Plane |
| `get_app_audit_history` | Retrieve audit history for an application |
| `chart_single_metric` | Create interactive chart from metrics data (returned as base64 data, no files) |
| `chart_render` | Render a chart for a metric in a single call — no data relay through the LLM |
| `chart_dashboard` | Create multi-panel dashboard comparing multiple metrics side-by-side |
| `chart_before_after` | Generate before/after comparison chart — fetches data server-side |
| `chart_violations` | Generate visualization for threshold violation detection results |
| `dashboard_create` | Create a new platform observability dashboard |
| `dashboard_get` | Get an existing platform observability dashboard by name |
| `dashboard_add_card` | Add a new card to an existing platform observability dashboard |
| `metrics_list_all` | Get complete list of ALL available metrics across ALL capabilities |
| `dashboard_delete` | Delete a platform observability dashboard |
| `metrics_list_by_capability` | Get metrics filtered by a SPECIFIC capability type for targeted dashboard creation |

## TIBCO BusinessWorks MCP Server

`/bw/mcp` — 25 tools, covering both BW5 CE and BW6 capabilities. Application and build lifecycle,
connector and version provisioning, plus diagnostics: thread dumps, smart engine reports and the
read-only OSGi console that pairs with the new `POST /v1/dp/apps/{appId}/instances/{instanceId}/osgi`
endpoint added to the BW6 capability API in 1.20.

| Tool | Purpose |
|------|---------|
| `bw_capture_app_thread_dump` | Capture thread dump for the specified BW app to analyze performance issues or deadlocks |
| `bw_delete_app` | delete (undeploy) a BW application (deployment) from a dataplane |
| `bw_delete_app_build` | delete one or more BW application builds (deployment artifacts) from a dataplane in a single call |
| `bw_deploy_helm_managed_app` | deploy (create a deployment of) a helm managed BW app on a dataplane for the provided build id |
| `bw_download_smart_engine_report` | download a smart engine report for a given BW6 Container app |
| `bw_enable_smart_engine_for_bw_app` | Enable smart engine for a given BW app |
| `bw_generate_smart_engine_report` | generate smart engine report for a given BW app |
| `bw_get_all_capability_summary` | Get all information about a BW5 or BW6 Container capability such as capability chart versions, ingress or gateway details, all provisioned connecto |
| `bw_get_app_details` | get application (deployment) details for a list of BW applications (deployments) in a single call |
| `bw_get_app_endpoints` | get all the app endpoints for given list of BW applications (deployments) |
| `bw_get_apps_for_build` | get all the BW apps (deployments) for one or more build IDs in a single call |
| `bw_get_bw_versions` | List all BW versions for a given capability instance from a dataplane |
| `bw_get_capability_info` | get information about a BW5 or BW6 Container capability |
| `bw_list_app_builds` | List all the existing BW app builds (deployment artifacts) from a dataplane, Returns basic app build information |
| `bw_list_bw_versions_from_controlplane` | List the catalog of all the BW versions from the control plane, Use these to provision on a specific dataplane via 'bw_provision_bw_version' tool |
| `bw_list_connectors` | List all the provisioned BW plugins / connectors and adapters from a dataplane |
| `bw_list_connectors_from_controlplane` | List the catalog of all the BW plugins / connectors and adapters from the control plane, Use these to provision on a specific dataplane via 'bw_pro |
| `bw_list_instances_for_app` | Get all the app instances (deployment instances / replicas) for one or more BW apps (deployments) in a single call |
| `bw_list_namespaces` | List all the namespaces for BW capabilities from a dataplane |
| `bw_list_smart_engine_reports` | list all smart engine reports for a given BW6 Container app |
| `bw_list_supplements` | List all the provisioned supplements for BW capabilities from a dataplane |
| `bw_provision_bw_version` | Provision a BW version in a dataplane. Use 'bw_list_bw_versions_from_controlplane' tool to list available BW versions |
| `bw_provision_connector` | Provision a BW plugin / connector or adapter in the specified dataplane |
| `bw_run_osgi_command` | Run a read-only TIBCO BusinessWorks OSGi console command against a running instance of the specified BW/BWCE app on a dataplane, to inspect or debu |
| `bw_scale_app` | Scale the BW app (deployment) to desired replicas for a given app (deployment) name or id |

## TIBCO Flogo MCP Server

`/flogo/mcp` — 19 tools. The Flogo counterpart to the BusinessWorks server: application and
build lifecycle, connector and version provisioning, namespace and capability inspection. It has
no equivalent of the BusinessWorks diagnostic tools.

| Tool | Purpose |
|------|---------|
| `flogo_delete_app` | Delete (undeploy) an application (deployment) from a dataplane |
| `flogo_delete_app_build` | Delete one or more application builds (deployment artifacts) from a dataplane in a single call |
| `flogo_deploy_helm_managed_app` | Deploy (create a deployment of) a helm managed app on a dataplane for the provided build id |
| `flogo_get_all_capability_summary` | Get all information about FLOGO capability such as capability chart version, ingress or gateway details, all provisioned connectors, FLOGO versions |
| `flogo_get_app_details` | Get details for a list of flogo applications (deployments) in a single call |
| `flogo_get_app_endpoints` | Get the app endpoints for a list of flogo applications (deployments) in a single call |
| `flogo_get_app_instances` | Get the app instances (deployment instances / replicas) for a list of flogo applications (deployments) in a single call |
| `flogo_get_apps_deployed_for_build` | Get all the flogo apps (deployments) deployed using one or more build IDs in a single call |
| `flogo_get_capability_info` | Get information about a specific capability |
| `flogo_get_provisioned_flogo_versions` | Get all provisioned flogo versions for a given capability instance from a dataplane |
| `flogo_list_app_builds` | List all the existing app builds (deployment artifacts) from a dataplane, Returns basic app build information |
| `flogo_list_connectors` | List all the provisioned flogo connectors from the dataplane |
| `flogo_list_connectors_from_controlplane` | List all the available flogo connectors from the control plane (connectors catalog) |
| `flogo_list_namespaces` | List all the namespaces for flogo capability from a dataplane |
| `flogo_list_supplements` | List all the provisioned supplements from a dataplane |
| `flogo_list_versions_from_control_plane` | List all the available flogo versions from control plane (buildtype catalog) |
| `flogo_provision_connector` | Provision a flogo connector in the specified dataplane |
| `flogo_provision_flogo_version` | Provision a flogo version in the dataplane. Use 'flogo_list_versions_from_control_plane' tool to list available flogo versions from control plane |
| `flogo_scale_app` | Scale the app (deployment) to desired replicas for a given flogo app (deployment) |

## TIBCO Developer Hub MCP Server

`/api/mcp-actions/v1` — 12 tools. Unlike the four platform servers, this one is served by the
Developer Hub itself, so it versions with the Hub rather than with the platform release; the
definition published here is from Hub 0.1.14. It exposes the software catalog
(query, fetch, validate, register and unregister entities) and the scaffolder (list actions, dry-run
and execute templates, follow task logs).

| Tool | Purpose |
|------|---------|
| `auth.who-am-i` | Returns the catalog entity and user info for the currently authenticated user |
| `catalog.get-catalog-model-description` | Returns a markdown formatted description of the current catalog model, including all registered entity kinds, annotations, labels, tags, and relati |
| `catalog.get-catalog-entity` | This allows you to get a single entity from the software catalog |
| `catalog.validate-entity` | This action can be used to validate catalog-info.yaml file contents meant to be used with the software catalog |
| `catalog.register-entity` | Registers one or more entities in the Backstage catalog by creating a Location entity that points to a remote catalog-info.yaml file |
| `catalog.unregister-entity` | Unregisters a Location entity and all entities it owns from the Backstage catalog |
| `catalog.query-catalog-entities` | Query entities from the Backstage Software Catalog using predicate filters |
| `scaffolder.list-scaffolder-tasks` | This allows you to list scaffolder tasks that have been created |
| `scaffolder.dry-run-template` | Dry-runs a scaffolder template to validate it without making changes. Returns success with execution logs, or errors for validation failures |
| `scaffolder.list-scaffolder-actions` | Lists all installed Scaffolder actions |
| `scaffolder.execute-template` | Executes a Scaffolder template with its template ref and input parameter values |
| `scaffolder.get-scaffolder-task-logs` | Retrieve the log events for a scaffolder task |
