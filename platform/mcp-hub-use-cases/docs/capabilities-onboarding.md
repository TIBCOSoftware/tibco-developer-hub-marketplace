# Capabilities — Onboarding & tool lifecycle

TIBCO MCP Hub is the on-ramp and control point for tools that agents use through MCP. This page covers five non-plugin capabilities: bulk import from API specifications, BW5 WSDL import, sidecar discovery, TIBCO Flogo federation, and runtime removal of tools from a Virtual Server. For enterprise identity, external IdP setup, and scoped access tokens, see [external identity and governance](capabilities-identity-governance.md).

| Capability | Scenario | Where to find it | What it proves |
|---|---|---|---|
| API-spec discovery & bulk import | An integration team imports existing API specifications and publishes the operations it picks, without building a flow per service | Gateway → Virtual Tools → Discover API specs; bulk tool import API | One OpenAPI or WSDL app can expand into many tools, then a Virtual Server push exposes them through MCP |
| BW5 WSDL integration | Keep an existing BW5 SOAP service and call it from an agent | Gateway → Virtual Tools → Import WSDL | The Gateway bridges MCP tool calls to an unchanged SOAP operation; SOAP-specific governance is covered in [response shaping](policies-response-shaping.md) |
| Sidecar registry auto-discovery | Applications appear in the Hub when their Kubernetes workload satisfies the MCP discovery contract | Gateway server discovery drawer; Data Plane discovery sidecar | A labelled, MCP-speaking workload is listed as Verified; unlabelled or non-MCP workloads are not listed |
| TIBCO Flogo federation | Govern native Hub tools and federated Flogo tools through one Virtual Server | MCP server registration by URL; Virtual Server composition; optional policy binding | A federated MCP tool and a native Hub tool can sit in one catalogue and receive the same Virtual Server governance |
| Runtime tool lifecycle | Remove one risky tool from one endpoint while the rest of the server keeps running | Gateway → Virtual Servers → row actions → Edit server / tools; Push Changes to Gateway | A stale agent cannot call the removed tool because enforcement happens at the Gateway |

!!! note "Reference measurements"
    Figures on this page come from an internal reference test environment (one gateway, one caller,
    test fixtures). They illustrate relative behaviour; they are not performance commitments. Measure
    in your own environment.

## API-spec discovery & bulk import — No flow per service

> We already have services that agents can call. Can we onboard the estate without building one flow per service?

**Example scenario.** A freight-logistics integration team wants to import existing API specifications, pick the operations to expose, and publish them through one governed Virtual Server without building a flow per service. The underlying problem is common in long-lived estates: a few services have been wrapped successfully, but the rest cannot be processed one endpoint at a time.

In the reference environment, the Discover API specs drawer scanned the Data Plane namespace and found two OpenAPI specifications. Importing one customer-data API produced four REST tools: `get-customers`, `get-order-items`, `get-orders`, and `get-products`. After those tools were placed on a Virtual Server and pushed to the Gateway, an MCP client called `get-customers` and received 12 customer records, then called `get-orders` and received 67 order records. No separate flow was authored for each endpoint.

**What it proves.** MCP Hub can use a served OpenAPI or WSDL specification as the source for multiple Virtual Tools. The import action creates the tool definitions in the Hub; the push publishes the resulting Virtual Server to the Gateway.

**When to use it.** Use discovery import when services already run in your Data Plane, already serve OpenAPI or WSDL, and you want a whole application onboarded as tools. Use a hand-built REST or SOAP Virtual Tool when you need one bespoke endpoint, custom request shaping, or a URL the discovery sidecar cannot see.

### Where to find it

| Setting | Value |
|---|---|
| Capability | API-spec discovery and bulk import |
| Console path | MCP Hub → Gateway → Virtual Tools → Discover API specs |
| API shape | Bulk tool creation for imported specifications |
| Input | OpenAPI or WSDL served by services in the Data Plane namespace |
| Output | Virtual Tools that can be selected on a Virtual Server |
| Applies live after | Push Changes to Gateway and a `synced` result |

### How to do it

1. Open the Gateway in MCP Hub and go to **Virtual Tools**.
2. Select **Discover API specs**. The discovery sidecar scans the namespace and lists discovered specs with a checkbox per spec and a select-all control.
3. Select the application specs you want to import. In the reference run, one customer-data API was selected and a second spec left unselected.
4. Select **Import N specs**. MCP Hub expands the selected spec into tools. For OpenAPI, each endpoint is parsed, named, and rebased to the in-cluster service origin. For WSDL, operations are registered from the service definition.
5. Create or edit a Virtual Server and select the imported tools.
6. Select **Push Changes to Gateway** and wait for the result to report `synced`.
7. Verify from an MCP client with `tools/list` and `tools/call`, not only from the import toast.

Typical onboarding questions map to the following product behavior:

| Requirement | Product answer | Considerations |
|---|---|---|
| Find the API specifications already served in the Data Plane and turn them into MCP tools | **Partly.** The discovery sidecar can find multiple OpenAPI or WSDL specs in the Data Plane namespace, and each spec can expand to many tools. | Discovery is namespace-scoped. Exposure is a second step: import first, then push. |
| Onboard every operation of one API without writing a flow per operation | **Yes, per specification.** One OpenAPI or WSDL application can expand into its operations or endpoints without a flow per service. | MCP Hub does not import an entire external service catalogue as one object. Multi-select is per-spec fan-out, not whole-catalogue import. |
| Create the tools for several specifications in one step | **Yes.** The drawer drives a transactional bulk create for the tools produced from selected specs. | Treat generated names as part of the release gate: tool names must be lower-case and DNS-safe, and a non-conforming name can cause the push to the Gateway to fail. |
| Choose what to import from a list rather than configuring each tool | **Yes.** The workflow is checkbox, select-all if desired, and **Import N specs**. | Import writes Hub rows; push applies them at the Gateway. |

### What happens

The imported tools appear in the Hub inventory, then the Virtual Server push reconciles them to the Gateway. In the reference run, the push result reported `Synced · All entities pushed successfully`, and the agent-facing endpoint returned real sample data through the imported tools.

An MCP client sees normal tool names and normal tool results. For example, `tools/list` returned the four imported tools, and `tools/call get-customers {}` returned `isError:false` with 12 customer records. The important proof is that the agent called tools generated from the API specification, not individually authored flows.

### Setup tips

- Keep generated tool names lower-case and DNS-safe before the push; a non-conforming name can cause the push to the Gateway to fail. OpenAPI discovery uses the Hub parser to lower-case generated names; apply the same discipline to other bulk sources.
- Read the push result, not just the import toast. Import creates Hub records; only the push updates the runtime Gateway.
- If a spec is already imported, expect the drawer to treat it differently from a new spec. Review the selected set before importing.
- If the service is outside the Data Plane namespace, use URL or paste import instead of namespace discovery.
- Test at least one imported tool with a plain MCP client before connecting an agent team.

### Considerations

This capability covers the core requirement, but it is scoped. It imports per application specification, not an arbitrary enterprise catalogue. It discovers within the Data Plane namespace, not across every registry you own. It also separates selection, import, Virtual Server composition, and push; plan operational checks around those separate states.

## BW5 WSDL integration — The SOAP you kept

> The BW5 SOAP service still runs and still matters. Can an agent call it without rewriting it?

**Example scenario.** A bank or integration team has a BW5 SOAP service that should remain in place. The WSDL is available, but the addresses inside it are not the addresses the Gateway should call from the Data Plane. MCP Hub imports the WSDL, accepts an endpoint override, and lets the Gateway adapt MCP tool calls into SOAP requests.

The reference service is the ZipInfo BW5 sample. Its WSDL defines service `ZipInfo-service1.serviceagent`, two SOAP ports, and two operations: `getCityInfo` and `getCityDistance`. Registering both ports and both operations produced four SOAP tool entries. The test selected the city-distance operation on one endpoint and called it from an agent.

**What it proves.** MCP Hub can configure a Gateway bridge from an existing BW5 SOAP service definition. BW5 continues to serve SOAP; the Gateway exposes an MCP tool façade. For the fuller BW5 integration walkthrough, see [integration.md](integration.md). For SOAP-specific policy behavior, see [policies-response-shaping.md](policies-response-shaping.md).

**When to use it.** Use WSDL import when the service contract is already SOAP and the right integration choice is to preserve it. Use OpenAPI discovery or REST Virtual Tools when the service already publishes REST. Use Flogo federation when the upstream already speaks MCP.

### Where to find it

| Setting | Value |
|---|---|
| Capability | BW5 WSDL integration |
| Console path | MCP Hub → Gateway → Virtual Tools → Import WSDL |
| Input | WSDL XML, pasted or loaded from a reachable service definition |
| Required in this case | Endpoint override for the SOAP address the Gateway can reach |
| Output | SOAP-backed Virtual Tools |
| Applies live after | Virtual Server selection and Push Changes to Gateway |

### How to do it

1. Open **Virtual Tools** and choose **Import WSDL**.
2. Paste the WSDL XML or provide a reachable WSDL URL.
3. Select **Scan addresses**, then parse and preview the operations.
4. Provide an endpoint override when the WSDL contains development, workstation, or otherwise unreachable addresses. A typical in-cluster override has this shape:

```text
http://<bw5-service>.<namespace>.svc.cluster.local:<port>/Service/ZipInfo-service1.serviceagent/ZipPortEndpoint1
```

5. Register the desired operations. For the ZipInfo sample, the important operation schema was:

| Operation | Request shape | Response shape |
|---|---|---|
| `getCityInfo` | `zip` as a string | city information |
| `getCityDistance` | object with `fromZip` and `toZip` | from-city, to-city, and distance |

6. Add the SOAP tool to a Virtual Server and push the changes to the Gateway.
7. If the Virtual Server requires token authentication, issue a scoped token and configure the agent connection with that token.

### What happens

Before the SOAP tool is available, the agent cannot answer a request that requires the BW5 service. After the tool is selected and pushed, the same agent can call the SOAP-backed tool through MCP.

In the reference run, the prompt asked for the BW5 ZipInfo response for `fromZip` `90210` and `toZip` `10001`. The SOAP sample returned its canned response: Urbana (`61801`), Champaign (`61820`), and distance `4`. The value proves connectivity to the SOAP service and correct request bridging; it does not prove a geographic calculation.

This scenario proves onboarding. Governing the SOAP-backed tool — before-tool policies, rate limits and argument masking — is covered in [response shaping](policies-response-shaping.md); test the policies you depend on against the SOAP tool itself.

### Setup tips

- Check the WSDL addresses before import. Legacy WSDLs often contain addresses that are valid for the original runtime but not for the Gateway.
- Prefer an endpoint the Gateway can reach inside the Data Plane. Do not assume public ingress or self-signed URL retrieval will be available.
- Inspect generated schemas before exposing the tool to agents. Scalar SOAP parts and object-shaped SOAP parts surface differently.
- Read the expanded tool result. Agent prose can summarize or misunderstand a canned sample.
- Issue short-lived, scoped access tokens for Virtual Servers that are not open access; identity patterns live in [external identity and governance](capabilities-identity-governance.md).

### Considerations

This case proves an inline WSDL import with endpoint override and an agent call to one unchanged BW5 SOAP sample. It does not prove every WSDL shape, every authentication pattern, or public WSDL retrieval. For plugin behavior on SOAP-backed tools, see [response shaping](policies-response-shaping.md); keep the integration proof and the governance proof separate.

## Sidecar registry auto-discovery — By contract

> Ship an MCP-speaking application with the right workload contract, and the Hub lists it.

**Example scenario.** A platform team ships integration applications continuously. Each application that should be callable by an agent used to require a ticket: find its service URL, register it by hand, test it, and then wire it into a Virtual Server. Sidecar registry auto-discovery turns that into a workload contract.

The discovery sidecar watches the Data Plane namespace. It asks for workloads with the platform user-application label, then verifies whether the workload is MCP-capable by connector metadata or by a live MCP handshake. In the reference environment, a newly deployed application appeared in the Hub on the next discovery scan — about 18 seconds after deployment in that run — and was already marked **Verified**. One registration action and one push put its tools behind the Gateway.

**What it proves.** Discovery is not a loose URL scrape. An unlabelled workload is not enumerated at all. A labelled workload that speaks HTTP but does not complete an MCP handshake is still absent. The gate is the contract plus protocol, not a successful TCP connection or a generic HTTP 200.

**When to use it.** Use sidecar discovery when your applications already run in the Data Plane and can carry the platform label and MCP contract in their deployment metadata or protocol surface. Use API-spec import when the app serves OpenAPI or WSDL but does not speak MCP. Use manual server registration when a known MCP endpoint is reachable but outside the sidecar's discovery scope.

### Where to find it

| Setting | Value |
|---|---|
| Capability | Sidecar registry auto-discovery |
| Console path | MCP Hub → Gateway → server discovery drawer |
| Runtime component | Discovery sidecar beside the Gateway |
| Hard gate | `platform.tibco.com/workload-type=user-app` |
| MCP gate | Connector segment `Model Context Protocol (MCP)` or successful handshake on a reserved MCP path |
| Cadence | 45 seconds; observed wait depends on where deployment lands in the cycle |

### How to do it

1. Add the user-application label to the workload metadata:

```yaml
metadata:
  labels:
    platform.tibco.com/workload-type: user-app
```

2. Satisfy one MCP arm. Either declare the connector segment exactly:

```yaml
metadata:
  annotations:
    platform.tibco.com/connectors: "Model Context Protocol (MCP)"
```

or serve MCP so the sidecar can complete a handshake on one of the reserved paths:

```text
/mcp
/rest/mcp
/soap/mcp
```

3. Deploy the application in the Data Plane namespace.
4. Open the Gateway discovery drawer and wait for the next scan interval.
5. Register the **Verified** row.
6. Push to the Gateway, then test with an MCP client or agent.

### What happens

A matching application is listed with a **Verified** state because the sidecar has completed the MCP check. Registration records the server in the Hub. The push then sends the server to the Gateway so its tools can federate and appear to MCP clients.

The negative cases are just as important. A workload deployed without the platform label is not listed. A labelled BW service that returns ordinary HTTP responses but does not complete MCP handshake is not listed. A different application with no explicit connector annotation can still be listed if it serves MCP successfully; if more than one port or reserved path answers, one deployment may produce more than one discovered row.

### Setup tips

- Treat 45 seconds as a scan cadence, not a service-level latency. A workload may appear quickly or near the end of the current interval.
- A `200` response is not proof of MCP. Confirm the handshake, not only the status code.
- If a row is present but you need to know which discovery arm admitted it, inspect the workload metadata and protocol behavior. The drawer displays the result, not full provenance.
- Registering is separate from discovery, and pushing is separate from registration.
- Use stable workload labels and annotations in deployment manifests so discovery follows the application release process.

### Considerations

Sidecar discovery is deliberately bounded. It only considers workloads in scope for the Data Plane and contract. The Hub does not override a missing label from the console. It also does not auto-publish discovered services: an operator still registers the row and pushes the change. This keeps discovery useful for onboarding without turning every matching workload into a live agent-facing endpoint by default.

## TIBCO Flogo federation — One governed catalogue

> You can see the seam between a native tool and a federated Flogo tool. The Virtual Server policy cannot.

**Example scenario.** A specialty insurer runs many TIBCO Flogo applications. Each already exposes a small MCP server. Without federation, agents would need to talk to many unrelated endpoints, each with separate governance. With MCP Hub federation, the upstream MCP servers stay in their own pods and lifecycle, while MCP Hub presents a curated Virtual Server endpoint.

In the reference run, the Virtual Server contained five tools: one native Hub-authored REST tool and four federated Flogo tools. The governance check used one native tool, `hr-directory-getusers`, and one federated tool, `customer-mcp-getcustomers`. A single RateLimiter binding on the Virtual Server refused the second call to each tool with the same message.

**What it proves.** Federation by URL registers an upstream MCP server by URL and federates its tools into the Hub catalogue. The Hub stores the upstream URL; it does not take over the Flogo application's lifecycle. Governance attached to the Virtual Server can apply across the native and federated tools exposed by that Virtual Server.

**When to use it.** Use Flogo federation when the upstream applications already speak MCP and you need one governed catalogue. If they do not speak MCP yet, use API-spec discovery and bulk import, WSDL import, or a hand-built Virtual Tool depending on the available contract.

### Where to find it

| Setting | Value |
|---|---|
| Capability | TIBCO Flogo MCP federation |
| Console path | MCP Hub → Gateway → MCP server registration; Virtual Servers for composition |
| Upstream input | MCP server URL exposed by the Flogo application |
| Hub output | Federated tool names following the `<server>-<tool>` pattern |
| Governance point | Virtual Server, not the upstream pod |
| Validation policy used | `RateLimiterPlugin` on Before Tool in Block (`sequential`) mode |

### How to do it

1. Confirm the Flogo application exposes an MCP server endpoint reachable by the Gateway.
2. Register the upstream MCP server URL in MCP Hub.
3. Push to the Gateway so tools federate into the catalogue.
4. Create or edit a Virtual Server and select both native Hub tools and federated tools as needed.
5. Attach governance to the Virtual Server, not to the upstream application. The reference validation used this policy configuration:

```json
{
  "by_tool": {
    "hr-directory-getusers": "1/m",
    "customer-mcp-getcustomers": "1/m"
  },
  "algorithm": "sliding_window"
}
```

| Policy setting | Value |
|---|---|
| Plugin | `RateLimiterPlugin` |
| Hook | Before Tool (`tool_pre_invoke`); also declares `prompt_pre_fetch` |
| Mode | Block (`sequential`) |
| Priority | `20` |
| Scope | Virtual Server |

6. Push the binding, then test one native tool and one federated tool twice.

### What happens

The first call to the native tool completed and returned records. The second call to the same native tool was refused. The first call to the federated Flogo tool completed and returned records. The second call to the federated tool was refused with the same policy message:

```text
tool_pre_invoke blocked by plugin RateLimiterPlugin: RATE_LIMIT - Rate limit exceeded
```

The key detail is the `by_tool` scope. It proves the federated refusal is evaluated for the federated tool itself, not merely caused by a shared budget already consumed by the native call.

### Setup tips

- Register by URL carefully. If the same upstream URL already exists, the Hub may reuse the existing server row rather than creating a duplicate. Verify the result after registration.
- Keep Virtual Server governance at the Virtual Server layer so it applies to every exposed tool, regardless of where that tool originated.
- If the policy editor opens in a form view and you need to paste JSON, switch to JSON before editing.
- Use paired tests: native success then native refusal, federated success then federated refusal.
- If the upstream service does not already speak MCP, federation is the wrong on-ramp.

### Considerations

This case is bounded to federated MCP tools behind a Virtual Server. It does not prove that every integration mechanism inherits every policy in the same way. The BW5 WSDL case above shows why that distinction matters, and SOAP-specific governance belongs in [policies-response-shaping.md](policies-response-shaping.md). Recovery time depends on the algorithm and where in the window the first call landed; test timing-sensitive limits under your own configuration.

## Runtime tool lifecycle — One click, gone

> When a tool becomes risky, remove it from the endpoint now; do not wait for redeploys or agent reconnects.

**Example scenario.** A vendor discloses a breach, and the incident playbook says every tool path reachable through that vendor must stop within minutes. Operators do not want to delete the upstream service, because other endpoints may still need unaffected tools. They need to remove one tool from one Virtual Server and have the Gateway enforce that decision immediately after push.

In the reference run, a Virtual Server initially exposed four federated customer-data tools. The operator removed `customer-mcp-getcustomers` from that Virtual Server, saved, and pushed. The next agent call to the removed tool failed even though the agent's own catalogue still listed all four tools.

**What it proves.** Runtime enforcement lives at the Gateway, not in the agent's cached catalogue. A stale client can still attempt the old tool call, but the Gateway refuses it. The same endpoint continued to serve a survivor tool.

**When to use it.** Use runtime tool lifecycle controls when one tool should stop being callable on one Virtual Server. Use source-server deletion or upstream remediation only when the tool should disappear from the estate, including every Virtual Server that references it.

### Where to find it

| Setting | Value |
|---|---|
| Capability | Runtime tool lifecycle |
| Console path | MCP Hub → Gateway → Virtual Servers → row actions → Edit server / tools |
| Runtime object changed | Virtual Server tool selection |
| Source object changed | None; the upstream tool remains available to other Virtual Servers |
| Applies live after | Save changes, then Push Changes to Gateway |
| Enforcement point | Gateway |

### How to do it

1. Open the Gateway and select the **Virtual Servers** tab.
2. Use the row action menu for the target Virtual Server and select **Edit server / tools**. Selecting the server name opens its plugin studio; use the row action for tool membership.
3. In the tool picker, switch to **Selected only** if helpful, search for the target tool, and select the already-picked row to remove it.
4. Confirm the drawer shows one tool removed after saving, then save the change.
5. Select **Push Changes to Gateway**. Review the diff and confirm it updates the Virtual Server you intended.
6. Verify in the Hub that the target tool is no longer listed on the Virtual Server.
7. Test two calls: the removed tool from a stale client, and a survivor tool on the same endpoint.

### What happens

The removed tool is no longer present in the Virtual Server after push. In the reference run, the operator's push took 5,292 ms from confirm click to result surface. The agent's next call rendered the refusal 2,787 ms after send, and the failed tool call itself took 67 ms.

The client-visible error was:

```text
Tool not found: customer-mcp-getcustomers
```

The stale catalogue is the important proof. The agent connection still listed the removed tool; no rediscovery had happened. The Gateway still refused the call. A control call to `customer-mcp-getorderitems` on the same endpoint succeeded and returned its order-item rows.

### Setup tips

- Use the tool picker's selected count as the authority. Large lists may virtualize rows, so visible row counts can mislead.
- Review the push diff for the Virtual Server update and read the push result details. A green summary is not a substitute for checking warnings and entity counts.
- Test with a stale client if you need to prove Gateway enforcement. A fresh `tools/list` only proves the catalogue view has converged.
- Keep a survivor-tool call in the test plan so the result proves one tool was removed, not the whole endpoint.
- Record whether the endpoint URL is preserved. It is when the Virtual Server is updated in place; if the Virtual Server is recreated, its endpoint URL changes.

### Considerations

Tool removal is Virtual Server scoped. The upstream tool still exists, and any other Virtual Server that includes it can still serve it. Endpoint preservation is conditional: in the reference run the Virtual Server was updated in place, so the endpoint remained stable; if the Virtual Server is recreated instead, its endpoint URL changes and connected clients must reconnect. The push is fast in the reference environment, but it is still a push, not an instantaneous local edit.
