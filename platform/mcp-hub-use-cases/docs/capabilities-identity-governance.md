# Capabilities — Identity, external policy & deployment

TIBCO MCP Hub can accept identities from the provider you already operate, call an external policy decision point before a tool runs, and leave enough evidence for security, operations and audit teams to reconstruct a governed call. The same model applies whether the runtime is hosted or deployed on infrastructure you operate. This page focuses on capabilities rather than individual plugin internals; for the UnifiedPDP and Permission details, see [access policies](policies-access.md), and for ordering several plugins on one Virtual Server, see [the plugin pipeline](plugin-pipeline.md).

| Capability | Scenario | Where it lives | What it proves |
|---|---|---|---|
| External identity plus external policy | Two gates on one call path | Virtual Server authentication, Permission stage, Before Tool policy check | The caller's token and the enterprise policy engine can both decide the same tool call, with distinct refusals. |
| External policy availability | Policy engine outage | Before Tool policy check on the MCP Gateway | Fail-closed behaviour is visible to the caller and becomes part of the availability design. |
| Bring your own identity provider | Your IdP, not ours | Virtual Server external OAuth/OIDC settings | A standards-compliant provider can issue the token; no vendor-specific plugin is required. |
| IdP connection checklist | Connect your identity provider | IdP application settings plus Virtual Server authentication | Issuer, JWT format, audience, claims and JWKS trust are the practical settings to get right. |
| On-premises deployment | Both planes on customer-managed infrastructure | TIBCO Control Plane, Data Plane and MCP Gateway installation | MCP Hub is a platform capability that can run beside BW6 and TIBCO Flogo on infrastructure you operate. |
| Call accountability | Six settings, one governed call | Virtual Server governance pipeline | Identity, decision and record can be composed into one auditable call path. |

!!! note "Reference measurements"
    Figures on this page come from an internal reference test environment (one gateway, one caller,
    test fixtures). They illustrate relative behaviour; they are not performance commitments. Measure
    in your own environment.

## External identity plus external policy — Two gates, one call path

> “Our agents use our identity provider, and our policy engine decides entitlement. The MCP layer must respect both.”

**Example scenario.** An online retailer exposes one MCP catalogue to support and billing agents. The platform team does not want agent accounts or entitlements copied into the gateway. The agent signs in with the enterprise identity provider; the access token carries `email` and `groups`; then the gateway asks the enterprise policy engine whether this exact subject may invoke this exact federated tool.

The design uses two independent authorization gates. `PermissionPlugin` reads the `groups` claim from `jwt_claims` and applies a default-deny group-to-tool map. `UnifiedPDPPlugin` calls an external OPA-compatible policy decision point for every tool invocation. The first gate answers “is this group allowed to reach this tool at all?” The second answers “does the current enterprise policy allow this subject, action and resource now?” The detailed plugin behaviours are covered in [access policies](policies-access.md).

**What it proves.** One Virtual Server can verify tokens from an external IdP, use claims from that token, and call an external policy engine on the same request. In the reference environment, two identities saw the same catalogue but received opposite verdicts on selected tools. Changing the policy in OPA changed the decision without editing or pushing the Virtual Server.

**When to use it.** Use this pattern when identity, groups and entitlements are already owned by enterprise systems. Do not use it as a scale benchmark; it proves the call path, not bulk onboarding. Require authentication on the Virtual Server and write policies as default-deny allow-lists.

### Where to find it

| Setting | Value |
|---|---|
| Console path | MCP Hub → Gateways → Virtual Servers → select a Virtual Server |
| Authentication | Require authentication, OAuth 2.0, external identity provider |
| Claims source | Verified access token, exposed to governance as `jwt_claims` |
| First gate | Permission stage, `PermissionPlugin`, default deny |
| Second gate | Before Tool (`tool_pre_invoke`), `UnifiedPDPPlugin`, Block (`sequential`) mode |
| Attached to | Virtual Server |
| External systems | OIDC IdP and OPA-compatible policy engine |

### How to do it

1. Create and push the Virtual Server so it has a stable endpoint URL.
2. Configure the IdP application to issue JWT access tokens for that endpoint. The token must include `email`, `groups`, and an `aud` value equal to the Virtual Server endpoint URL.
3. In MCP Hub, enable external OAuth on the Virtual Server with an issuer such as `https://idp.example.com/realms/mcp-inbound` and scopes `openid profile email groups`.
4. Attach `PermissionPlugin` at the Permission stage, then attach `UnifiedPDPPlugin` at Before Tool.
5. Push to Gateway and wait for `synced` before testing.

For Keycloak-style claim mapping, keep group names unqualified so the strings in the token match the role map exactly:

```json
{
  "groups_mapper": {
    "protocolMapper": "oidc-group-membership-mapper",
    "config": {
      "claim.name": "groups",
      "full.path": "false",
      "access.token.claim": "true",
      "id.token.claim": "true"
    }
  },
  "email_mapper": {
    "protocolMapper": "oidc-usermodel-property-mapper",
    "config": {
      "user.attribute": "email",
      "claim.name": "email",
      "jsonType.label": "String",
      "access.token.claim": "true",
      "id.token.claim": "true"
    }
  },
  "audience_mapper": {
    "included.custom.audience": "https://<your-gateway-host>/mcp/<virtual-server>",
    "included.client.audience": ""
  }
}
```

A default-deny Permission configuration can allow each group to one normal tool and both groups to one shared tool. The shared tool is intentional: it lets the second gate be the component that refuses.

```json
{
  "claims_context_key": "jwt_claims",
  "group_claim": "groups",
  "default_decision": "deny",
  "group_role_map": {
    "support-team": ["support-role"],
    "billing-team": ["billing-role"]
  },
  "roles": {
    "support-role": ["orders-mcp-runquery", "orders-mcp-echoargument"],
    "billing-role": ["orders-mcp-readheaders", "orders-mcp-echoargument"]
  }
}
```

The external policy can stay entirely outside MCP Hub:

```rego
package mcpgateway
import rego.v1

default allow := false

agent_entitlements := {
  "support-assistant@example.com": {"runquery"},
  "billing-agent@example.com": {"readheaders"},
}

subject := lower(input.subject.email)
tool := lower(input.resource.id)

allow if {
  startswith(input.action, "tools.invoke.")
  some t in agent_entitlements[subject]
  endswith(tool, t)
}

deny contains msg if {
  not allow
  msg := sprintf("agent %v is not entitled to %v - decided by the enterprise policy engine", [input.subject.email, input.resource.id])
}
```

Bind `UnifiedPDPPlugin` to the Virtual Server with the policy engine URL and policy path:

```json
{
  "engines": [
    {
      "name": "opa",
      "enabled": true,
      "priority": 1,
      "settings": {
        "opa_url": "http://opa.policy.svc.cluster.local:8181",
        "policy_path": "mcpgateway",
        "timeout_ms": 5000,
        "max_retries": 3
      }
    }
  ],
  "combination_mode": "all_must_allow",
  "default_decision": "deny"
}
```

### What happens

| Caller | Tool | Result | Gate that decides |
|---|---|---|---|
| `support-assistant@example.com` | `orders-mcp-runquery` | Forwarded | Both gates allow |
| `support-assistant@example.com` | `orders-mcp-readheaders` | Blocked | Permission gate |
| `support-assistant@example.com` | `orders-mcp-echoargument` | Blocked | External policy gate |
| `billing-agent@example.com` | `orders-mcp-readheaders` | Forwarded | Both gates allow |

The refusals are intentionally different:

```text
Permission denied for tool 'orders-mcp-readheaders'
```

```text
tool_pre_invoke blocked by plugin UnifiedPDPPlugin: PDP_DENY - Policy decision: DENY
([all_must_allow] agent support-assistant@example.com is not entitled to orders-mcp-echoargument - decided by the enterprise policy engine)
```

Dashboard traces show the request stopping at the relevant stage. Security views can distinguish a local permission denial from a policy-decision denial by plugin and reason.

### Setup tips

- Audience is order-sensitive: create and push the Virtual Server, copy its endpoint URL, set that URL as the IdP audience, then mint a fresh token.
- If a group claim arrives as `/support-team` but the role map expects `support-team`, authorization silently misses. Use unqualified group values or configure matching keys.
- Load the OPA module at the same path that the plugin evaluates, for example `mcpgateway`; stale modules at a different path can continue answering.
- Tool argument validation runs before policy hooks. If the call never reaches either gate, verify the tool input schema first.
- Identity-aware policies need the caller's identity forwarded to the policy engine. Before relying on a rule that keys on the caller, confirm with a permitted and a denied caller that the engine receives the subject you expect.
- Reconcile old plugin bindings when recreating a Virtual Server. A leftover default-deny Permission binding can look like an identity problem.

### Considerations

This pattern deliberately couples the catalogue to the IdP and policy engine. That is the point: the enterprise systems remain authoritative. Design those systems for the same availability target as the Virtual Server, and choose timeouts and retries explicitly.

## External policy availability — When the engine is down

> “Fail closed is the correct security posture, but it means the policy engine is now on the critical path.”

**Example scenario.** A regulated platform team requires every agent tool call to be checked by an external policy engine. During an outage test, the engine becomes unreachable between two identical tool calls. The team wants to know whether calls continue, how fast a new request stops, and whether the error is recognizable without reading gateway logs.

**What it proves.** With `UnifiedPDPPlugin` in Block (`sequential`) mode and `default_decision: deny`, calls fail closed when the engine cannot be reached. In the reference environment, a new-session call stopped 1.7 seconds after the engine was removed. Restoring the engine allowed the same call again without editing the Hub configuration or pushing the Virtual Server.

**When to use it.** Use this posture when an external decision point must remain authoritative. If your business requirement is to continue serving during a policy-engine outage, you need a documented exception model and risk acceptance. Engine faults and timeouts deny regardless of `default_decision`.

### Where to find it

| Setting | Value |
|---|---|
| Console path | Virtual Server → Tool calls → Before Tool |
| Plugin | `UnifiedPDPPlugin` |
| Hook | `tool_pre_invoke`; also declares `resource_pre_fetch` |
| Mode | Block (`sequential`) |
| Failure posture | `default_decision: deny` |
| Important runtime settings | `timeout_ms`, `max_retries`, decision-cache TTLs |

### How to do it

Use the same policy binding as the previous section. The settings that shape outage behaviour are:

```json
{
  "default_decision": "deny",
  "engines": [
    {
      "name": "opa",
      "enabled": true,
      "priority": 1,
      "settings": {
        "opa_url": "http://opa.policy.svc.cluster.local:8181",
        "policy_path": "mcpgateway",
        "timeout_ms": 5000,
        "max_retries": 3
      }
    }
  ],
  "combination_mode": "all_must_allow"
}
```

Test the posture in a non-production environment by removing the policy endpoint, calling a tool, restoring the endpoint, and calling the tool again. For an in-cluster OPA deployment, the mechanics can be as simple as:

```bash
kubectl scale deploy/opa-policy -n <policy-namespace> --replicas=0
# call the Virtual Server and observe the refusal
kubectl scale deploy/opa-policy -n <policy-namespace> --replicas=1
kubectl rollout status deploy/opa-policy -n <policy-namespace> --timeout=180s
# call again and confirm the same tool is allowed
```

Decision caching matters. A client that opens a new MCP session for each call gets a cache miss and asks the engine every time. A client that holds one session open can replay a recent allowed decision until the allow-cache TTL expires. The default allow-cache TTL (`allow_ttl_seconds`) is 5 seconds; in the reference environment, the new-session fail-closed measurement was 1.7 seconds.

### What happens

| Engine state | Caller experience |
|---|---|
| Available | The policy engine returns allow and the tool runs. |
| Unreachable | The call is refused before the tool runs. |
| Restored | The call succeeds again without Hub edits. |

A typical client-visible refusal is:

```text
tool_pre_invoke blocked by plugin UnifiedPDPPlugin: PDP_DENY - Policy decision: DENY
([all_must_allow] opa: evaluation error – [opa] All connection attempts failed – failing closed (indeterminate))
```

Traces show the request ending at `tool_pre_invoke`. Security dashboards can identify the difference between a policy denial and an engine reachability failure because the plugin reason names the engine failure and the fail-closed posture.

### Setup tips

- Measure both patterns that your clients use: new session per call and long-lived session.
- Keep the timeout lower than your user-facing timeout, otherwise callers wait on the policy engine until the client gives up first.
- Set the evaluation timeout and retries explicitly, and test them against a policy engine that answers slowly before relying on them.
- Treat the policy engine as part of the production service: run more than one replica, monitor it, and test restore by making a real call.
- Do not read readiness from the first pod returned by the API during a rollout. Use rollout status or your platform's deployment health signal.

### Considerations

Fail-closed protects the catalogue at the cost of availability. The right setting depends on the sensitivity of the tools behind the Virtual Server. The Hub can expose the failure clearly, but it cannot remove the operational dependency created by an authoritative external decision point.

## Bring your own identity provider — Your IdP, not ours

> “Do we need a vendor-specific identity plugin?”

**Example scenario.** A platform team already operates Keycloak, Okta, Entra ID or another OIDC-compliant provider. Agents authenticate there today. The team wants the MCP Gateway to accept those tokens directly, without registering every agent inside the gateway and without a plugin for each IdP product.

**What it proves.** External IdP support is a standards-based OIDC configuration on the Virtual Server. In the reference environment, one gateway accepted a JWT from one external IdP on one Virtual Server while also serving another Virtual Server backed by a different IdP. Calls with no token and calls with a corrupt token returned 401; a valid JWT from the configured issuer was forwarded.

**When to use it.** Use this when your organization already owns login, lifecycle and group membership. Combine it with PermissionPlugin, and with UnifiedPDP where an external policy engine decides; a valid token proves who the caller is, not what every tool should allow. For JWT claims extraction details and policy use, see [access policies](policies-access.md).

### Where to find it

| Setting | Value |
|---|---|
| Console path | Virtual Server → Authentication |
| Require authentication | On |
| OAuth mode | External identity provider |
| Issuer URL | `https://idp.example.com/realms/mcp-inbound` or the issuer URL advertised by your provider |
| Scopes | `openid email profile groups` |
| Authorization gate | `PermissionPlugin` in Block (`sequential`) mode, always bound |

### How to do it

Configure the Virtual Server authentication settings:

```json
{
  "requireAuth": true,
  "oauthConfig": {
    "mode": "external_idp",
    "issuerUrl": "https://idp.example.com/realms/mcp-inbound",
    "scopes": "openid email profile groups"
  }
}
```

On the IdP side, issue JWT access tokens and publish a standard OIDC discovery document. The gateway uses the issuer metadata to find the JWKS endpoint, fetch signing keys, and cache them. If your IdP uses a public certificate trusted by the runtime, no extra trust configuration is needed. If it uses an internal certificate authority in a test or private environment, add that CA to the gateway's trust configuration as documented for your release; do not disable verification.

Always bind PermissionPlugin in Block mode on every Virtual Server that accepts tokens from an external identity provider. Authorization then follows the groups in the token: for example, the group `support-team` can invoke `orders-mcp-runquery` and still be refused for `orders-mcp-readheaders`.

### What happens

| Call | Result |
|---|---|
| No token | 401, authentication required |
| Corrupt token | 401, invalid credentials |
| Valid token plus Permission gate | Authorized according to groups in the token |

### Setup tips

- Requesting an `email` scope is not always enough. Many providers put email into the identity token only unless the access-token claim is configured explicitly.
- Opaque access tokens fail like invalid credentials because the gateway validates JWT signatures against JWKS.
- If your provider advertises a loopback or private-only issuer URL, discovery and JWKS fetch will fail from the gateway.
- A token minted before the correct audience is configured still carries the old audience. Mint a fresh token after changing `aud`.
- Start with three control calls: no token, corrupt token, valid token. A success is meaningful only if the first two calls are refused.

### Considerations

This proves interoperability with the OIDC contract, not every feature of every provider. Provider-specific lifecycle, multi-factor, conditional access and group governance remain owned by the IdP. MCP Hub consumes the resulting token and claims.

## Connect your identity provider — The settings that decide success

> “The form is short, but the token has to be shaped precisely.”

**Example scenario.** A team has created a Virtual Server and selected an external identity provider, but every call returns 401. The usual cause is not the Hub form; it is one of three IdP-side settings: reachable issuer metadata, JWT access tokens, or required claims in the access token.

**What it proves.** The IdP integration can be reduced to a repeatable checklist. There is no vendor selector in MCP Hub. The same checklist applies to Keycloak, Okta, Entra ID and other standards-compliant providers, with provider-specific names for the same OAuth/OIDC concepts.

**When to use it.** Use this checklist when connecting a new provider, troubleshooting 401 responses, or preparing a production IdP application for MCP clients.

### Where to find it

| Setting | Value |
|---|---|
| MCP Hub | Virtual Server → Authentication |
| IdP | OAuth/OIDC application, claim mapping, token settings, issuer metadata |
| Gateway dependency | HTTPS trust for issuer metadata and JWKS |
| IdP audience | The pushed Virtual Server endpoint URL, set after the push |

### How to do it

1. **Advertise a reachable issuer.** The issuer and `jwks_uri` in the discovery document must be reachable from the gateway.
2. **Issue JWT access tokens.** The gateway validates JWT signatures against the provider's JWKS.
3. **Create and push the Virtual Server.** Copy the resulting endpoint URL.
4. **Set audience to that endpoint.** The access token's `aud` must contain the Virtual Server endpoint URL.
5. **Put required claims in the access token.** Include `email` and, if you use group authorization, `groups`.
6. **Mint a fresh token.** Previously issued tokens do not change after the IdP application changes.

Provider consoles and APIs name these settings differently, but every provider needs the same two changes on the
client application: the access token format set to **JWT**, and the `email` and `groups` claims added to the
**access token** — not only to the identity token. Keycloak issues JWT access tokens by default; the protocol mappers
shown in [External identity plus external policy](#external-identity-plus-external-policy-two-gates-one-call-path)
add the claims.

Set the audience only after the Virtual Server endpoint exists:

```json
{
  "idToken": {
    "audience": ["https://<your-gateway-host>/mcp/<virtual-server>"]
  },
  "accessToken": {
    "audience": ["https://<your-gateway-host>/mcp/<virtual-server>"]
  }
}
```

### What happens

| Missing setting | Common symptom |
|---|---|
| Reachable issuer or trusted certificate | Metadata discovery or JWKS fetch fails |
| JWT access token | 401 invalid credentials |
| Email in access token | 401 token missing valid email claim |
| Groups in access token | Authentication succeeds, group authorization misses |

### Setup tips

- Production public certificates usually need no gateway trust change. Internal CAs do.
- Do not replace the full trust store with only your private CA; preserve public roots and add yours.
- Some providers default `aud` to the client ID. Set it to the Virtual Server endpoint URL.
- To restrict who may call a Virtual Server's tools, bind PermissionPlugin on it.
- Initialize an MCP session before `tools/list`; a direct list request without session context can return a protocol error that is not an authentication failure.
- Keep the IdP application persistent. A test IdP without durable storage can lose clients and users on restart.

### Considerations

The gateway accepts the OIDC contract. It does not manage IdP users, password policy, group lifecycle or token issuance rules. Treat IdP configuration as production infrastructure and version it with the same discipline as application configuration.

## On-premises deployment — Both planes on-prem

> “The Control Plane and the Data Plane must run on infrastructure we operate.”

**Example scenario.** An enterprise with data-residency requirements deploys TIBCO Control Plane, a Data Plane and MCP Hub into a Kubernetes cluster it manages. The team also installs BW6 and TIBCO Flogo capabilities in the same platform footprint. The question is whether MCP Hub is a special deployment case or another platform capability with the same chart-and-image model.

**What it proves.** MCP Hub can be installed as a Control Plane capability while the MCP Gateway runs on a Data Plane in the same customer-managed environment. Chart delivery and image delivery have different credential boundaries: chart access is as documented for your release, while commercial runtime images require authenticated image pull access.

**When to use it.** Use this deployment model when policy, networking or data-residency requirements require both planes to run inside customer-operated infrastructure. Do not treat it as an install-duration benchmark unless you run a clean-room installation from empty cluster to ready platform.

### Where to find it

| Setting | Value |
|---|---|
| Platform layer | TIBCO Control Plane capability charts |
| Runtime layer | Data Plane namespace with MCP Gateway |
| Related capabilities | BW6, TIBCO Flogo, TIBCO Developer Hub |
| Credentials | Chart access as documented for the release; image pull credentials for commercial registries |
| Hardening focus | Non-root containers, namespace-scoped pull secrets, ingress and TLS, safe kernel settings |

### How to do it

1. Install the supported TIBCO Control Plane release in the customer-managed cluster.
2. Add the MCP Hub capability chart for the same supported release line.
3. Install or register the Data Plane and deploy the MCP Gateway into its namespace.
4. Provide image pull credentials in each namespace that needs commercial runtime images.
5. Configure ingress, DNS and TLS so the Control Plane UI, gateway endpoints and IdP issuer URLs are reachable from their intended clients.
6. Validate with a small MCP server, a Virtual Server and a token-authenticated tool call.

Obtain the MCP Hub capability chart from the repository documented for your release. When checking chart availability, record both the fetch result and the match count. A zero match from a failed download and a zero match from a valid index are different outcomes:

```bash
curl -sSL -w '\nhttp=%{http_code} bytes=%{size_download}\n' <chart-repository-url>/index.yaml \
  | grep -E '<chart-name>|http=|bytes='
```

For hardened clusters, check the listener ports documented for the gateway release you install against your pod security settings: a non-root container under hardened pod settings cannot bind ports below 1024. Set this explicitly and verify it on your release before production.

### What happens

A healthy on-premises deployment has the same control flow as a hosted one. Platform users open MCP Hub in the Control Plane, register MCP servers, push them to the Data Plane gateway, compose Virtual Servers, attach governance, and test from an MCP client or a test agent client. Runtime traffic stays on the Data Plane path you expose.

### Setup tips

- Image pull secrets are namespace-scoped. Copying a secret name from the Control Plane namespace to a Data Plane namespace can fail if the generated names differ.
- Follow redirects when checking chart indexes or artifact endpoints; some repositories return a redirect before the actual index.
- Harden the pod network namespace you are testing, not the node only. Container runtime defaults can mask a privileged-port issue.
- When probing a gateway container, copy the required environment from the deployed workload; otherwise the process can exit before reaching the listener you intended to test.
- Delete disposable probes after testing and confirm the live gateway did not restart during the check.

### Considerations

An on-premises deployment still depends on the supported release artifacts, registry credentials, DNS, certificates and cluster policy you provide. Plan high availability for both planes, and test upgrades against the same hardening profile you use in production.

## Call accountability — Six settings, one governed call

> “Before an agent calls a business tool, we need to know who called, what was allowed, and what record remains.”

**Example scenario.** A payments operations team exposes a tool such as `orders-mcp-release-payment` to agents. The platform team needs identity from the access token, a group permission check, an external policy decision, business metadata on the trace, exported telemetry, and an outbound event after successful execution. The goal is not only to block unsafe calls; it is to leave a record that security, operations, audit and usage reporting can read consistently.

**What it proves.** A governed call path can be designed from six settings on one Virtual Server: Identity, Permission, Policy, Metadata, Telemetry and Event. The Virtual Server auth setting establishes the caller and seeds `jwt_claims`; Permission and UnifiedPDP decide; SpanAttributeCustomizer, ToolsTelemetryExporter and WebhookNotification record what happened. For ordering and priority guidance, see [the plugin pipeline](plugin-pipeline.md).

**When to use it.** Use this as a baseline for business tools where agent access must be attributable and reviewable. Do not rely on an after-tool webhook as the only alert for blocked calls: a blocked call never reaches the after-tool success path. Use Security dashboards, traces, or SIEM export (enabled when the gateway is deployed) for blocks.

!!! note "Illustrative configuration"
    The six settings below are a recommended composition. Individual settings were exercised in the reference
    environment; the complete six-setting call and webhook delivery were not measured end to end. Verify each
    record in your own environment.

### Where to find it

| Setting | Value |
|---|---|
| Console path | Virtual Server authentication settings and its **Tool calls** lanes |
| Identity | Virtual Server authentication; validated token claims are available as `jwt_claims` |
| Permission | Permission stage (`http_auth_check_permission`) |
| Policy | Before Tool (`tool_pre_invoke`) |
| Metadata | Before Tool transform |
| Telemetry | Before Tool / After Tool telemetry spans |
| Event | After Tool (`tool_post_invoke`) |
| External systems | IdP, policy engine, trace collector, webhook receiver |

!!! note "Identity on the Virtual Server path"
    For inbound OAuth on a Virtual Server, the gateway validates the access token and exposes the claims to governance. You do not need to add a separate identity plugin just to read the token on this path. Use JWT-claims plugins where the documented hook requires them; see [access policies](policies-access.md).

### How to do it

Configure the six settings in this order, then push and wait for `synced`.

1. **Identity.** Enable external OAuth/OIDC on the Virtual Server.
2. **Permission.** Add a default-deny Permission binding that maps token groups to allowed federated tools.
3. **Policy.** Add `UnifiedPDPPlugin` before the tool to call the enterprise policy engine.
4. **Metadata.** Add business attributes that should appear on traces and records.
5. **Telemetry.** Export tool spans and, only when required, payload detail.
6. **Event.** Send an after-tool event to the receiver that owns downstream reconciliation.

Example metadata configuration:

```json
{
  "global_attributes": {
    "business.service": "payments",
    "business.cost_centre": "finance-operations",
    "data_class": "restricted",
    "legal_basis": "contract"
  },
  "tool_overrides": {
    "orders-mcp-release-payment": {
      "attributes": {
        "business.operation": "release-payment",
        "approval_required": "true"
      }
    }
  },
  "attribute_mapping": {
    "tool.name": "mcp.tool"
  },
  "remove_attributes": ["authorization", "cookie"]
}
```

Example telemetry exporter configuration:

```json
{
  "export_full_payload": true,
  "max_payload_bytes_size": 100000
}
```

Example after-tool event configuration:

```json
{
  "webhooks": [
    {
      "url": "https://events.example.com/mcp/audit",
      "events": ["tool_success"],
      "authentication": {
        "type": "bearer",
        "token": "<webhook-bearer-token>"
      },
      "retry_attempts": 3,
      "timeout": 10
    }
  ],
  "include_payload_data": false
}
```

### What happens

A successful call leaves a trace with the tool name, the plugin stages and the final tool result; the configured business attributes are added to the gateway's tool-invocation span. If telemetry export is enabled, the trace collector shows the Before Tool and After Tool stages around the gateway call. Check which identity attribute your release records on exported spans rather than assuming the caller's identity appears there. If the webhook is enabled, verify delivery of the configured after-tool event with a test receiver before relying on it.

A blocked call stops at the gate that refused it. Permission denials, PDP denials and PDP reachability failures are visible as different reasons. Because the tool did not run, after-tool webhooks are not sent for that blocked call.

### Setup tips

- Priorities decide order when several plugins share a hook. Put deny decisions before transformations that assume the call will run.
- Keep payload export off unless the receiving system has a clear need and an approved data-handling posture. To keep sensitive arguments out of traces, mask them with a data-protection plugin that runs before the exporter.
- Use explicit span attributes for the business record you need; do not rely on a generic `user` field if your release does not document its exact source.
- The webhook receiver owns deduplication, retries beyond the configured attempt count, and downstream reconciliation.
- The webhook bearer token is part of the plugin configuration: restrict who can read and edit Virtual Server plugin bindings, and rotate the token on the receiver side. Keep IdP client secrets out of shared configuration.

### Considerations

MCP Hub can create an accountable call path, but it is not a billing engine, identity directory, policy authoring system or event reconciliation platform. It supplies governed execution points and records; downstream teams decide how those records are retained, correlated and reported.
