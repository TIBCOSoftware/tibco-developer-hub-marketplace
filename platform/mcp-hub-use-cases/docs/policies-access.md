# Policies — Access, identity & credentials

Access policies decide which tools and resources an agent can use, whose identity is used for the decision, and where credentials are allowed to appear. This page collects the MCP Hub controls that sit closest to identity and upstream credentials: external policy engines, group-to-tool permissions, claim extraction, post-ingestion write gates, header injection and vault-backed per-call secrets. The organisation names, resource URIs, amounts and identifiers below are illustrative reference-environment data. For plugin ordering and hook names, see [Plugin pipeline](plugin-pipeline.md); for external IdP and external policy setup, see [Identity and governance capabilities](capabilities-identity-governance.md).

| Plugin / capability | Scenario | Hook(s) | What it proves |
|---|---|---|---|
| UnifiedPDPPlugin | Use an existing OPA/Rego policy to decide which tools may run. | Before Tool (`tool_pre_invoke`); also declares Before Resource (`resource_pre_fetch`) | MCP Hub can delegate the authorization decision to an external policy engine and fail closed when the engine denies or is unavailable. |
| PermissionPlugin | One assistant serves different people, but each token exposes only the tools their group may use. | Permission stage (`http_auth_check_permission`) | The same Virtual Server can be safe for clerks, approvers, operators and unmapped users when the tool allowlist is derived from the caller's groups. |
| JwtClaimsExtractionPlugin | A bound identity plugin is accepted by MCP Hub, but MCP tool calls already get their identity another way. | Identity (`http_auth_resolve_user`) | Authentication can succeed without this plugin taking part; do not treat a binding as proof that a hook is in the MCP call path. |
| SessionWriteGate | A principal reads an untrusted alarm, then tries a write that the source did not permit. | After Tool (`tool_post_invoke`) and Before Tool (`tool_pre_invoke`) | The gateway can remember what a principal has read and block later writes on the same server before the upstream system is reached. |
| HeaderInjector | A resource read needs a tenant header that the agent must not hold. | Before Resource (`resource_pre_fetch`) | The gateway can add a static header to its own upstream resource fetch, scoped by URI prefix. |
| VaultPlugin | A protected tool needs a secret from the customer's vault on each call. | Before Tool (`tool_pre_invoke`); also declares `agent_pre_invoke` | The gateway can use a per-call credential without storing it in MCP Hub configuration or showing it to the agent. |

## UnifiedPDPPlugin — bring your own policy

> “Our authorization policy already lives in OPA. We are not writing it again in another product.”

**Example scenario.** A platform team is rolling out a support assistant that can read order status, check shipment data and open tickets. The same upstream server also exposes operational tools that should not be available to a general-purpose assistant. Security already maintains its authorization rules in Open Policy Agent (OPA), with Rego reviewed and versioned like application code.

Instead of copying those rules into MCP Hub, the team binds `UnifiedPDPPlugin` to the Virtual Server and points it at OPA. MCP Hub stores the engine URL and policy path. OPA stores the policy. On every tool call, the gateway asks the external policy engine whether the federated tool may run.

**What it proves.** The decision happens before the tool is reached. A permitted tool is forwarded, an unlisted tool is blocked, and the MCP client receives the policy author's reason. Dashboard traces record the result as a `UnifiedPDPPlugin` denial, so the platform record names the engine-backed control that decided.

**When to use it.** Use this when authorization is already centralised in an external policy engine and the customer wants one source of truth for service, human and agent access policy. If the team only needs groups mapped to tools, `PermissionPlugin` is simpler. For SQL content checks, such as “allow `SELECT`, deny `DROP`”, prefer [SQLSanitizer](policies-content-safety.md), which is purpose-built.

### Where it sits

| Setting | Value |
|---|---|
| Plugin | `UnifiedPDPPlugin` |
| Hook | Before Tool (`tool_pre_invoke`); the plugin also declares Before Resource (`resource_pre_fetch`), and one assignment activates it on every hook it declares |
| Mode | Block (`sequential`) |
| Attached to | Virtual Server |
| External engine | OPA reachable from the MCP Gateway |
| Decision input | The caller, the tool and the request context, including the tool arguments (`input.context.tool_args`) |

### Configuration

1. Ensure OPA is reachable from the gateway runtime. A URL that only works from an administrator's workstation is not enough.
2. Deploy or select the Virtual Server the agent will use.
3. Add `UnifiedPDPPlugin` on **Before Tool** (`tool_pre_invoke`) in **Block** (`sequential`) mode.
4. Enter the engine configuration in JSON view, apply it, then **Push to Gateway** and wait for `synced`.
5. Test an allowed tool first, then a denied tool. A deny-only test can look successful even when OPA is unavailable, because the plugin fails closed.

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

`opa_url` is the address as the gateway sees it. `policy_path` must match the Rego package. Engine faults, timeouts and an unreachable engine always deny the call, whatever `default_decision` says; keep `default_decision: "deny"` so the configuration states the same posture.

```rego
package mcpgateway

import rego.v1

default allow := false

# The gateway sends the federated tool name: <server>-<tool>.
read_only_suffixes := ["lookup", "get-order", "list-orders"]

tool := lower(input.resource.id)

is_read_only if {
  some s in read_only_suffixes
  endswith(tool, s)
}

allow if {
  startswith(input.action, "tools.invoke.")
  is_read_only
}

deny contains msg if {
  not is_read_only
  msg := sprintf("agents may call read-only tools only; %v is not on the platform team's list", [input.resource.id])
}
```

The example policy allows read-only tools such as `orders-mcp-lookup` and blocks write-like tools such as `orders-mcp-runquery`. The important detail is the identifier: policy authors evaluate the federated `<server>-<tool>` name that the gateway sends.

### What happens

A read-only call is forwarded:

```text
orders-mcp-lookup → FORWARDED
```

A write-like call is stopped before the upstream server:

```text
orders-mcp-runquery → BLOCKED

tool_pre_invoke blocked by plugin UnifiedPDPPlugin: PDP_DENY - Policy decision: DENY
([all_must_allow] agents may call read-only tools only; orders-mcp-runquery is not on the platform team's list)
```

From an agent's point of view, the refusal can be paraphrased into a useful answer such as “I cannot run that cleanup job because platform policy blocked access to the query tool.” The trace remains precise: **Blocked** with `PluginViolation: UnifiedPDPPlugin / PDP_DENY`.

If the policy changes in OPA, MCP Hub does not need a new plugin configuration. For example, adding `runquery` to `read_only_suffixes` allows the call; removing it blocks the call again after OPA serves the updated policy.

### Setup tips

- **Allow leg first.** Because the plugin fails closed, an unreachable OPA endpoint can cause every deny test to pass for the wrong reason.
- **Package and path must match.** A Rego package named `mcpgateway` belongs behind `policy_path: "mcpgateway"`.
- **Use a gateway-reachable URL.** `http://opa.policy.svc.cluster.local:8181` is the pattern for an in-cluster service; replace the service and namespace for your environment.
- **Read the binding back after push.** A plugin can be attached but still be using an empty or default configuration if the JSON was not applied before the push.
- **Plan policy ownership.** MCP Hub points to the policy engine; the policy repository, review process and OPA availability remain owned by the customer platform team. See [Identity and governance capabilities](capabilities-identity-governance.md) for the broader external-policy model.

### Considerations

The engine receives the caller, the tool and the request context, including the tool arguments (`input.context.tool_args`). For SQL content checks, prefer [SQLSanitizer](policies-content-safety.md), which is purpose-built.

The plugin also introduces a runtime dependency. That is deliberate when OPA is the source of truth, but it means OPA must be deployed, monitored and reachable from the gateway. Set the evaluation timeout and retries explicitly, and test them against a policy engine that answers slowly before relying on them. Keep `default_decision` strict, and test both a permitted and a denied path after every change.

## PermissionPlugin — one assistant, different authority

> “The assistant is the same. The person using it is not.”

**Example scenario.** A single assistant is useful because everyone gets the same endpoint, the same documentation and the same tool discovery flow. The risk is that the tool list often mixes reads with irreversible actions: delete a data plane, release a payment, export customer data, or change production state. `PermissionPlugin` keeps the shared assistant model but makes the caller's token decide which tools may run.

The plugin reads a configured claims object, normally `jwt_claims`, extracts a group claim, maps groups to roles, and maps roles to federated tool names. The safest posture is default-deny: a group that is not mapped gets nothing, and a tool that is not explicitly named stays unavailable unless a role intentionally grants `*`.

**What it proves.** The same Virtual Server can serve multiple people safely. An on-call engineer can list systems but not delete them. An accounts-payable clerk can read invoices but not release money. An operations assistant can summarize an alert without being able to run an export tool planted in the same upstream capability set.

**When to use it.** Use `PermissionPlugin` when authorization is naturally “who may call which tool,” and the answer already lives in IdP groups. It is not an argument validator, not an output filter, and not a substitute for source-to-write provenance when reads and writes must remain on the same server. For external IdP setup and claim availability, see [Identity and governance capabilities](capabilities-identity-governance.md).

### Where it sits

| Setting | Value |
|---|---|
| Plugin | `PermissionPlugin` |
| Hook | Permission stage (`http_auth_check_permission`) |
| Mode | Block (`sequential`) |
| Attached to | Virtual Server with `requireAuth` enabled |
| Inputs | Claims context, group claim, group-to-role map, role-to-tool map |
| Tool identifiers | Federated tool names such as `northstar-finance-release-payment` |

### Shared configuration pattern

Always bind PermissionPlugin in Block mode on every Virtual Server that accepts tokens from an external identity provider.

1. Put the Virtual Server behind OAuth 2.0 or another supported external IdP mode.
2. Ensure the token contains the group claim you intend to use, such as `groups`.
3. Add `PermissionPlugin` on the permission stage in **Block** (`sequential`) mode.
4. Set `default_decision` to `deny`.
5. Push to the gateway, then sign in again or refresh the client connection so it presents a current token.

A compact administration example follows the same shape:

```json
{
  "claims_context_key": "jwt_claims",
  "group_claim": "groups",
  "default_decision": "deny",
  "group_role_map": {
    "platform-admins": ["platform-admin"],
    "oncall": ["platform-reader"]
  },
  "roles": {
    "platform-admin": ["*"],
    "platform-reader": ["ops-list-environments"]
  }
}
```

The platform reader can call `ops-list-environments`; the destructive `ops-delete-environment` tool is omitted. The administrator role intentionally grants `*`, so the role must be reserved for identities that are allowed to reach every tool on that Virtual Server.

### One assistant, two people — the token decides

> “Who on the team could ask an assistant to delete production?”

**Example scenario.** A platform operations assistant fronts an environment administration server. One tool lists environments. Another deletes one. Both tools are on the same server, and both appear to the assistant unless the Virtual Server is governed.

With `PermissionPlugin`, the on-call group receives the read role and the platform-admin group receives the administrator role. An unmapped user receives no role at all. The assistant endpoint is the same for all three; only the token differs.

**What happens.** A call without a token receives `HTTP 401`. The on-call engineer can list environments but receives a policy refusal for the delete tool:

```text
Permission denied for tool 'ops-delete-environment'
```

The platform administrator can reach the delete tool. In the reference test environment, the proof uses a deliberately non-existent environment ID and does not send a force flag, so the upstream system responds with its own “does not exist” error instead of deleting anything. The unmapped user is denied even for the read tool because `default_decision` is `deny`.

**Setup tips.** Set the IdP audience to the Virtual Server endpoint URL before minting tokens. If the denied tool returns an input-schema error instead of a permission error, send arguments that pass schema validation so the request reaches the permission stage.

**Considerations.** The token is a snapshot. Removing a user from a group does not change a token the user already holds. Use appropriate token lifetimes and refresh behaviour in the IdP for high-risk roles.

### Release the payment — the token decides who may

> “The clerk can check an invoice. Only the controller can release money.”

**Example scenario.** An accounts-payable assistant fronts a finance MCP server with three tools: list invoices, get one invoice, and release payment. The clerk and the payment approver use the same assistant. The difference is the group in the token.

```json
{
  "claims_context_key": "jwt_claims",
  "group_claim": "groups",
  "default_decision": "deny",
  "group_role_map": {
    "accounts-payable": ["ap-clerk"],
    "payment-approvers": ["payment-approver"]
  },
  "roles": {
    "payment-approver": ["*"],
    "ap-clerk": [
      "northstar-finance-list-invoices",
      "northstar-finance-get-invoice"
    ]
  }
}
```

The release tool is absent from the clerk role by omission. That matters because omission survives future tool additions better than a broad allowlist followed by exceptions.

**What happens.** The clerk can list the five invoices in the example system, then receives:

```text
Permission denied for tool 'northstar-finance-release-payment'
```

The approver can run the same release request. In a reference test environment the successful result is a real state change in the example system:

```json
{
  "result": "payment_scheduled",
  "invoice": "INV-20481",
  "amount_usd": 48750.0,
  "payment_reference": "PAY-90316"
}
```

A contractor or any other unmapped group is denied by default, including for reads.

**Setup tips.** Reset state before demonstrating a release. A second release of the same invoice may return `already_scheduled`, which proves the tool was allowed but is a poor business proof. Use an invoice that is still pending, and assert the state change rather than only asserting that no policy error occurred.

**Considerations.** A role that grants `*` grants all current and future tools on that Virtual Server. Use it only where that is truly the intended authority.

### The alert told the agent what to do

> “The human asked for a summary. The alert text asked the model to export data.”

**Example scenario.** An on-call assistant reads incident alerts from an operations feed. The latest alert includes third-party text, and that text contains an instruction addressed to the model: call the export tool to get the affected-customer count and send the snapshot onward. The human prompt only says, “Summarize the latest incident alert.”

The governed Virtual Server exposes the two read tools and denies the export tool. A baseline server with all tools open shows the risk: the model can obey the data it just read, not the person who asked the question.

```json
{
  "claims_context_key": "jwt_claims",
  "group_claim": "groups",
  "default_decision": "deny",
  "group_role_map": {
    "sre-oncall": ["incident-reader"]
  },
  "roles": {
    "incident-reader": [
      "meridian-incident-feed-list-alerts",
      "meridian-incident-feed-get-alert"
    ]
  }
}
```

`meridian-incident-feed-export-customers` is granted to nobody. If the tool is not selected onto the governed Virtual Server, the model cannot discover it there. If it is present but not granted, `PermissionPlugin` refuses it.

**What happens.** The governed assistant lists alerts, reads the latest alert, and produces a summary. It does not export the customer table. The baseline assistant, using the same human prompt against an open server, calls the export tool unprompted by the human. In a reference test environment, the export call mints an export ID and reports `4,812` rows, making the exfiltration visible as state, not just a successful response.

**Setup tips.** Keep the human prompt identical on the governed and baseline paths. If a model resists the injected instruction, improve the realism of the untrusted alert text rather than changing the human request. That preserves the point of the scenario: the user did not ask for the export.

**Considerations.** `PermissionPlugin` gates calls, not answers. If a permitted tool returns sensitive content, use data-protection or response-shaping controls for the result. If the same server must expose both the untrusted read and the dangerous write, use `SessionWriteGate` later in this page to record the read and block the follow-on write in the request path.

## JwtClaimsExtractionPlugin — the identity was already there

> “The token was valid, but this plugin was not the reason the tool call succeeded.”

**Example scenario.** An operations team protects a Virtual Server with OAuth 2.0 and then binds `JwtClaimsExtractionPlugin` on the Identity lane, expecting the plugin to extract claims for the MCP tool call. The tool call succeeds with a real JWT. A call without a token is rejected. At first glance, that can look like the plugin did the identity work.

The measured behaviour is more precise: MCP Streamable HTTP Virtual Server calls already seed `jwt_claims` through the Virtual Server authentication path. The `http_auth_resolve_user` hook is accepted as a binding, but it is not opened by this MCP tool-call path.

**What it proves.** A plugin binding can be real, visible and pushed, while still not participating in a specific request path. Authentication succeeds because the Virtual Server requires and validates a token, not because `JwtClaimsExtractionPlugin` refused or allowed the call.

**When to use it.** Treat this plugin as a specialised identity-lane component, not an authorization gate for MCP tool invocation. For ordinary MCP access control, start with the Virtual Server's authentication settings, then use `PermissionPlugin` or `UnifiedPDPPlugin` for authorization decisions. For the broader identity setup, see [Identity and governance capabilities](capabilities-identity-governance.md).

### Where it sits

| Setting | Value |
|---|---|
| Plugin | `JwtClaimsExtractionPlugin` |
| Hook | Identity (`http_auth_resolve_user`) |
| Mode | `transform` |
| Priority | `10` in the reference catalogue |
| Attached to | Virtual Server |
| Important boundary | Does not refuse MCP tool calls on this path |

### Configuration

1. Create or select an OAuth-protected Virtual Server.
2. Open the **Identity** lane (`http_auth_resolve_user`).
3. Add `JwtClaimsExtractionPlugin`.
4. Keep the context key at the default unless another plugin expects a different key.
5. Push to the gateway and test with and without a token.

```json
{
  "context_key": "jwt_claims"
}
```

### What happens

| Leg | Door | Identity | Result | Plugin receipt |
|---|---|---|---|---|
| Governed | OAuth-protected Virtual Server | Valid JWT | Tool call allowed | Empty; no plugin marker in the MCP reply metadata |
| Governed | Same Virtual Server | No token | `HTTP 401` | Not applicable; the Virtual Server's own authentication rejects the call |
| Baseline | Open Virtual Server | No token | Tool call allowed | Not applicable; no plugin is bound |

The empty receipt does not mean authentication failed. The governed call succeeds. It means this plugin did not take part in that MCP tool invocation path.

### Setup tips

- Use an MCP client or trace capture that can inspect reply metadata. Agent chat surfaces often hide `_meta` and HTTP status details.
- Test the protected Virtual Server without a token. The expected refusal is the Virtual Server's own `401`, not a plugin denial.
- Do not rely on this plugin for group-to-tool policy. Use `PermissionPlugin` or an external policy decision through `UnifiedPDPPlugin`.

### Considerations

This plugin does not verify a JWT signature as an independent authorization control, and its own behaviour is non-blocking; the Hub attaches it in Transform (`transform`) mode. Choosing a blocking mode does not turn it into a deny-capable policy gate for MCP tool calls. The Hub's own description of the Identity lane says the same: it serves the gateway's admin and REST API and is not run for Virtual Server requests. The practical lesson is to verify the hook path before relying on any plugin for access enforcement; the [Plugin pipeline](plugin-pipeline.md) explains where each hook runs.

## SessionWriteGate — the write that read the alarm

> “The assistant was allowed to switch the feeder. What had it just read when it tried?”

**Example scenario.** A network operations assistant reads an alarm queue and can also operate a switching system. The latest alarm contains free text from the field. That text includes an instruction to de-energise a feeder immediately. On an open server, an autonomous assistant can read the alarm and then perform the write because the write is otherwise within its authority.

`SessionWriteGate` answers a narrower question than role-based access: after this principal reads from a declared untrusted source, which declared writes are still allowed for that principal during the configured window?

**What it proves.** The same principal can perform a planned write before reading the alarm. After reading the untrusted source, the same write is blocked before the switching system is reached. A write that the source explicitly permits, such as opening a ticket, still goes through. Dashboard traces record the refusal as `SessionWriteGate` with a source-to-write reason.

**When to use it.** Use it when the same assistant must have both reads and writes on one Virtual Server, and the risk is that untrusted content tells the model to use its write authority. If the dangerous write can simply be left off the governed server, `PermissionPlugin` may be enough. If the write must coexist with the read, SessionWriteGate creates a request-path interlock and a record.

### Where it sits

| Setting | Value |
|---|---|
| Plugin | `SessionWriteGate` |
| Hooks | After Tool (`tool_post_invoke`) records source reads; Before Tool (`tool_pre_invoke`) blocks writes |
| Mode | Block (`sequential`) |
| Priority | `45` in the reference gateway |
| Attached to | Authenticated Virtual Server |
| Principal scope | Same authenticated principal |
| Tool keys | Upstream original tool names, or the canonical tool identity triple |

### Configuration

1. Put the governed Virtual Server behind an identity provider. The plugin needs a trustworthy principal; anonymous writes are refused.
2. Select the tools that must coexist: the untrusted read source, ordinary reads, and the writes.
3. Add `SessionWriteGate` on **Before Tool**. One assignment activates the plugin on every hook it declares — here the Before Tool gate and the After Tool recorder; the lane you choose is where the Hub shows it.
4. Enter the operator's classification and push to the gateway.
5. Prove an allowed write before any untrusted read, then read the source, then repeat the write.

```json
{
  "sources": {
    "alarm_feed": {
      "permits": ["open_ticket"]
    }
  },
  "write_tools": ["deenergise_feeder", "open_ticket"],
  "taint_ttl_seconds": 900
}
```

`sources` names the tools whose results contain untrusted content. `permits` names the writes still allowed after reading that source. `write_tools` names the tools that change the world. `taint_ttl_seconds` is the window from the first read; reading the same source again does not extend it.

!!! warning "Use upstream tool names"
    The keys above are upstream tool names such as `alarm_feed`, not the federated names shown as `grid-operations-mcp-alarm-feed`. A classification copied from the federated tool list will not match the source you intended.

### What happens

In the governed path, the principal first performs planned maintenance before any alarm read:

```text
deenergise_feeder(FDR-206) → SW-0001, planned maintenance accepted
```

The same principal then triages the latest alarm. The alarm text is delivered to the model, but the follow-on write is blocked:

```text
alarm_feed → ALM-7734 delivered

deenergise_feeder(FDR-118) → BLOCKED
Tool 'deenergise_feeder' is not permitted after this principal ingested content from '<gateway-id>|alarm_feed|MCP' at <first-read-time>. Writes that source still permits: ['open_ticket']

open_ticket → TCK-0001
```

In the baseline path, the same triage flow reaches the write and the example feeder changes state:

```text
deenergise_feeder(FDR-118) → SW-0002, FDR-118 DE-ENERGISED, customers_off_supply: 4312
```

Dashboard → Traces shows the governed refusal as:

```text
Blocked (SessionWriteGate · WRITE_BLOCKED_AFTER_UNTRUSTED_INGEST)
PluginViolation: SessionWriteGate / WRITE_BLOCKED_AFTER_UNTRUSTED_INGEST
```

In a reference test environment, a second principal that had not read the alarm could still perform the same write, and a short-window test refused the write shortly after the read but allowed it again after the window expired.

### Setup tips

- **Authenticate the Virtual Server.** A declared write from an anonymous or unknown principal is refused because the plugin cannot safely scope the taint.
- **Run the allow leg first.** A deny-only proof does not show that the write would have been allowed before the read.
- **Understand taint lifetime.** The taint is durable state for the configured window, not an in-memory hint. A recent read by the same principal can affect a later test until the window expires; plan test windows with a short `taint_ttl_seconds` or a fresh principal.
- **Keep direct bypass paths closed.** If an alternate path can invoke the write without traversing this gateway, this plugin cannot govern that path.
- **Use permitted writes deliberately.** Include a safe write such as `open_ticket` in `permits` when the operational workflow should continue after the gate blocks the dangerous action.

### Considerations

SessionWriteGate only knows what operators declare. Undeclared sources, undeclared writes, renamed upstream tools and writes outside this gateway are outside its decision. The control is per principal: a second identity starts clean, while a shared service account shares the same taint. It does not ask a model to confirm its own action, and it does not prevent the model from reading the untrusted text. It governs what the model may do next.

## HeaderInjector — the header the agent never held

> “The resource server needs a tenant key, but the assistant must never see it.”

**Example scenario.** A maintenance assistant reads service manuals from an equipment vendor's MCP resource server. The server lists public resource identifiers without a key, but it requires a tenant header when the gateway reads protected content. If the key is pasted into a prompt or an agent connection, it becomes part of the agent context. If no key is provided, the resource body is empty or unavailable.

`HeaderInjector` adds a configured header to the gateway's own upstream resource fetch. The agent asks for the resource; the gateway supplies the header; the agent receives the document, not the header value.

**What it proves.** A governed resource read can return the real document without putting the tenant key in the agent prompt, response or connection. The same governed Virtual Server can expose resources from the same upstream server where only URI prefixes that match the binding receive the header.

**When to use it.** Use HeaderInjector for resource fetches when a static header is acceptable in gateway configuration and the resources that need it can be scoped by URI prefix. Do not use it for arbitrary URL-fetching tools or normal tool calls; it has no tool hook.

### Where it sits

| Setting | Value |
|---|---|
| Plugin | `HeaderInjector` |
| Hook | Before Resource (`resource_pre_fetch`) |
| Mode | `transform` |
| Priority | `58` (catalogue default) |
| Attached to | Virtual Server exposing MCP resources |
| Scope | Resource URI prefixes |

### Configuration

1. Federate a resource server that permits unauthenticated registration and `resources/list`, while enforcing the header on `resources/read`.
2. Create a Virtual Server over the resources.
3. Add `HeaderInjector` on **Before Resource** (`resource_pre_fetch`).
4. Configure the header and one or more URI prefixes.
5. Push to the gateway and read the binding back.

```json
{
  "headers": {
    "X-Tenant-Key": "<tenant-key>"
  },
  "uri_prefixes": ["vendor://vendor-a/"]
}
```

Omit `uri_prefixes` only when the header should apply to every resource the Virtual Server exposes. With a prefix configured, the plugin applies a simple prefix match to the resource URI.

### What happens

| Read | Result |
|---|---|
| Baseline Virtual Server, `vendor://vendor-a/manuals/v90-gearbox-service` | The call succeeds but the body is empty in the reference gateway; no authentication error is visible to the client. |
| Governed Virtual Server, same URI | The manual content is returned, including its document marker and procedure text. The key is not visible in the agent result. |
| Governed Virtual Server, `vendor://vendor-b/manuals/k500-blade-inspection` | Empty again, because the URI is outside `vendor://vendor-a/` even though it is served by the same upstream server. |

### Setup tips

- **The server must allow discovery without the key.** If `resources/list` requires the header, the resources may never federate.
- **Measure the denial shape on your gateway version.** In the reference environment, a refused resource read surfaced as an ordinary empty document, not a visible `401` or `403` to the client.
- **Treat binding edits as credential authority.** Treat the right to edit plugin bindings as credential-management authority on that upstream, and restrict it accordingly.
- **Choose prefixes carefully.** A broad prefix can apply the header to more resources than intended; a narrow prefix can produce empty reads that look like missing content.

### Considerations

HeaderInjector protects the agent from holding the header. If the requirement is that MCP Hub never holds the upstream credential at all, use `VaultPlugin` instead. HeaderInjector also does not help with tool calls, `fetch_url` tools or upstream servers that refuse unauthenticated resource discovery.

## VaultPlugin — the credential never enters your product

> “The gateway may use our secret for this call, but your product must not store it.”

**Example scenario.** A bank, carrier or other regulated team wants an assistant to call a protected upstream tool, such as a treasury balance or partner account lookup. The upstream requires a bearer credential that rotates in the customer's vault. Storing that credential in MCP Hub configuration would fail the security review.

`VaultPlugin` uses a different pattern. The caller presents a vault-sourced token to the gateway on each call in a carrier header. The gateway resolves which upstream system the call targets, injects the token into the upstream `Authorization` header for that call, and strips the carrier header before the tool sees it.

**What it proves.** The credential is required, used and absent from the agent transcript. A governed call with the vault header authenticates upstream and returns business data. A control call without the vault header is refused by the protected upstream. The token is not stored in MCP Hub plugin configuration and is not returned to the agent.

**When to use it.** Use VaultPlugin when the customer owns the secret lifecycle and wants MCP Hub to act as a conduit, not as a credential store. If a static header in gateway configuration is acceptable, HeaderInjector may be simpler for resource reads. If the credential must be supplied per call from the customer's vault, use VaultPlugin.

### Where it sits

| Setting | Value |
|---|---|
| Plugin | `VaultPlugin` |
| Hook | Before Tool (`tool_pre_invoke`); the plugin also declares `agent_pre_invoke`, and one assignment activates it on every hook it declares |
| Mode | Transform (`transform`); the plugin injects and strips headers and never refuses a call |
| Attached to | Virtual Server exposing protected tools |
| Credential source | Per-call carrier header supplied by the caller |
| Upstream effect | Injects `Authorization` for the target system and strips the carrier header |

### Configuration

The plugin configuration contains routing rules, not a secret:

```json
{
  "system_tag_prefix": "system",
  "vault_header_name": "X-Vault-Tokens",
  "vault_handling": "raw",
  "system_handling": "tag",
  "auth_header_tag_prefix": "AUTH_HEADER"
}
```

A caller presents the per-call token in the configured carrier header. The key in the JSON object corresponds to the target system identified by the server's system tag:

```json
{
  "treasury-api.example": "<token-from-customer-vault>"
}
```

Configuration steps:

1. Configure the upstream MCP server or peer metadata so the gateway can resolve the target system, for example with a `system:<system-name>` tag.
2. Allow the carrier header to reach the gateway for that peer. Header passthrough, the peer's system tag and enabling the plugin in the gateway's plugin configuration are gateway installation settings owned by the customer platform team; MCP Hub has no screen for them.
3. Add `VaultPlugin` on **Before Tool** (`tool_pre_invoke`) for the governed Virtual Server.
4. Enter the routing configuration above. There is no token field to fill in.
5. Push to the gateway and prove both paths: with the carrier header and without it.

### What happens

On the wire, the governed call shows the important transformation:

```text
Caller supplies:      X-Vault-Tokens: {"treasury-api.example":"<token-from-customer-vault>"}
Upstream receives:    Authorization: Bearer <token-from-customer-vault>
Upstream does not see: X-Vault-Tokens
```

The control call omits `X-Vault-Tokens`; no `Authorization` header is injected, and the protected upstream refuses the request.

From the agent's point of view, the governed call can use a neutral tool such as `treasury-mcp-get-balance` and receive business data:

```json
{
  "status": "authenticated",
  "account": "CORPORATE-TREASURY-001",
  "available_balance": "4,820,335.17"
}
```

The control path returns an upstream error such as:

```text
unauthorized: this tool requires a valid upstream credential; none was presented
```

The token value is absent from both transcripts. The proof is anti-vacuous: without the credential, the tool cannot return the balance; with it, the tool authenticates, while the agent still never holds the secret.

### Setup tips

- **Keep echo tools out of agent demonstrations.** Header-echo tools are useful to verify wire behaviour, but they would return injected headers to the agent. Use a protected non-echoing upstream for agent tests.
- **Check exact stripping.** The carrier header must be absent downstream on success and on error paths.
- **Ensure the binding is load-bearing.** If the gateway runs the plugin globally for every call, the Virtual Server binding may be decorative. Prefer a deployment where the base plugin is inactive and the binding activates it for the intended server.
- **Persist gateway-side settings deliberately.** Header passthrough and peer tagging are runtime deployment settings. Encode them in the gateway configuration process so upgrades do not silently remove them.

### Considerations

VaultPlugin has two halves: the Hub-visible binding and the gateway-side setup that lets the carrier header reach the plugin and map to a target system. The Hub screen answers whether the Virtual Server is bound; it does not by itself prove that peer tags and header passthrough are correct.

The plugin also protects credentials only on paths that traverse this gateway and this hook. It does not turn an unsafe upstream response into a safe response, and it does not replace output controls for secrets or personal data returned by a permitted tool. Use it with least-privilege tool exposure and the appropriate data-protection policies for responses.
