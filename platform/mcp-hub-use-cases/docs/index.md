# TIBCO MCP Hub — Scenarios & Policies

**Audience:** platform engineers, AI architects, integration leads and security teams who are putting AI agents in
front of enterprise systems with TIBCO MCP Hub.

This guide is a library of **worked scenarios**. Each one starts from a business problem — an agent that can reach
too much, a payload that costs too much, a credential that must never leave the vault, a service estate that is too big
to onboard by hand — and shows how MCP Hub answers it: which capability or policy plugin, where it sits in the
pipeline, the exact configuration, what an agent and an operator actually see, the traps teams hit on the way, and
where the control stops.

---

## Why MCP Hub

The Model Context Protocol (MCP) made it trivial to expose a tool to an AI application. It did not make it safe,
governed or observable to do so at enterprise scale. Teams that wire agents straight to tools end up with:

| Problem | What it looks like |
|---|---|
| **Sprawl** | Every team stands up its own MCP server. No inventory, no owner, no shared catalogue of what exists. |
| **No blast-radius control** | An agent that can reach a tool can call it. Nobody curates who sees which tools. |
| **Ungoverned data flow** | PII, secrets and unvalidated payloads move between models and back-end systems unchecked. |
| **Credential chaos** | Long-lived, over-scoped, hand-rolled tokens spread across client applications. |

**TIBCO MCP Hub** is the management console for MCP in the TIBCO Control Plane. From one place you **deploy** MCP
Gateways on your Data Planes, **connect** the MCP servers that expose your tools, **compose** curated Virtual Servers,
**govern** every call with policy plugins, **issue** scoped access tokens, and **observe** the traffic. Management is
central; runtime stays local to your data.

```text
            TIBCO Control Plane
  ┌──────────────────────────────────────────┐
  │  MCP Hub — console · catalogue · policy   │
  │            draft + push · tokens          │
  └───────────────┬──────────────────────────┘
                  │ push
    ┌─────────────┼─────────────────────┐
    ▼             ▼                     ▼
 Data Plane    Data Plane           Data Plane
 MCP Gateway   MCP Gateway          MCP Gateway
 federation · plugin pipeline · Virtual Servers
    │
    ├── MCP servers (Flogo, BW6, custom, registry)
    └── Virtual Tools (from OpenAPI / WSDL, e.g. BW5 SOAP / REST)
```

## The concepts used throughout this guide

| Concept | Meaning |
|---|---|
| **MCP Gateway** | An independent runtime on a Data Plane that fronts one or more MCP servers behind a single governed endpoint. Installed and managed from the Hub through the Control Plane API; day-to-day configuration happens in the Hub, while some deployment-level settings are applied when the gateway is installed. |
| **MCP server** | An upstream service that provides tools, prompts and resources. Registered on a gateway, then **pushed** so its tools **federate**. |
| **Virtual Server** | A curated, client-facing MCP endpoint — its own URL, its own authentication posture (open, gateway-issued token, or external identity provider), its own policy pipeline. |
| **Virtual Tool** | A tool built in the Hub straight from an OpenAPI (REST) or WSDL (SOAP) specification, or a plain REST call — no MCP server to write. |
| **Plugin** | A policy control assigned to a Virtual Server with a **mode** and a **priority**. Hooks include Before Tool (`tool_pre_invoke`), After Tool (`tool_post_invoke`), Before/After Prompt and Before/After Resource. One assignment activates the plugin on every hook it declares; the lane you choose is where the Hub shows it. |
| **Draft → push → synced** | Nothing goes live by accident. Servers, Virtual Servers, tools and policy are drafted in the Hub and only take effect once pushed and reported `synced` by the gateway. Removals need a push too: deleting a binding in the Hub leaves it active on the gateway until the next push. |
| **Test agent client** | An agent chat client used throughout this guide to test each control from the agent's side, not just from a raw MCP client. |
| **Dashboards** | Monitoring, Traces and Security views on each gateway. Blocked calls appear under **Traces → Blocked** with the plugin and violation code that stopped them. |

MCP Hub governs **three request pipelines** — **Tool Execution**, **Prompt Fetch** and **Resource Fetch**. Most
scenarios in this guide sit on the tool pipeline; several deliberately cover the prompt and resource pipelines, which
are just as exposed.

## How to read each scenario

Every scenario in this guide follows the same shape, so you can jump straight to the part you need:

| Section | What you get |
|---|---|
| **Example scenario** | The business situation, set in a realistic industry. Organisations, people and data are illustrative. |
| **What it proves** | The one claim the scenario demonstrates. |
| **When to use it** | When to reach for this control — and when a simpler one does the same job. |
| **Where it sits** | Plugin name, hook, mode, priority and scope — or, for capabilities, where to find it in the console. |
| **Configuration** | The click path and the exact JSON configuration. |
| **What happens** | The allowed and blocked (or transformed) behaviour, the exact text an MCP client or agent receives, and what the dashboards record. |
| **Setup tips** | Traps teams actually hit, as symptom → cause → fix. |
| **Considerations** | Where the control stops. These are stated plainly on purpose: a control is only useful if you know its boundary. |

!!! tip "Test the allowed path first"
    Many controls in this guide **fail closed**. With a misconfigured or unreachable dependency, *every* governed call
    is refused — so a test that only checks the refusal can pass with nothing working at all. Always assert a
    permitted call in the same run as the blocked one.

Except where a page marks a configuration as illustrative, scenarios were exercised on a live TIBCO Platform Control
Plane and Data Plane with real calls; numbers quoted are from a reference test environment and will vary with yours.

---

## The scenario map

### Capabilities — getting tools in, and running them

| Page | Scenario | The question it answers |
|---|---|---|
| [Onboarding & tool lifecycle](capabilities-onboarding.md) | API-spec discovery & bulk import — *No flow per service* | We have a large estate of existing services. How do they become MCP tools without hand-building one flow each? |
| | BW5 WSDL integration — *The SOAP you kept* | Can an agent call an existing BW5 SOAP service without touching the application? |
| | Sidecar registry auto-discovery — *By contract* | Can workloads appear in the Hub automatically when they meet the MCP discovery contract? |
| | TIBCO Flogo federation — *One governed catalogue* | Can native Hub tools and federated Flogo tools be governed through one Virtual Server? |
| | Runtime tool lifecycle — *One click, gone* | Can we pull one risky tool from one endpoint while everything else keeps running? |
| [Identity, external policy & deployment](capabilities-identity-governance.md) | External identity + external policy — *Two gates, one call path* | Can our IdP authenticate and our policy engine authorize the same call? |
| | External policy availability — *When the engine is down* | What happens to agent traffic when the policy engine is unavailable? |
| | Bring your own IdP — *Your IdP, not ours* / *Connect your identity provider* | How do we front Virtual Servers with our own identity provider, and which settings decide success? |
| | On-premises deployment — *Both planes, on-prem* | Can the Control Plane and the Data Plane both run on our own infrastructure? |
| | Call accountability — *Six settings, one governed call* | Which settings make a single call attributable, end to end? |
| [Composing the plugin pipeline](plugin-pipeline.md) | How the pipeline works | Hooks, priorities, modes, fail-open vs. fail-closed, draft-to-synced rollout. |
| | *Fourteen assignments, one call* | What does a realistic, fully governed tool call look like? |
| | *One at a time* | How do you add governance incrementally, one operational problem at a time? |
| | *Eight desks, one errand* | How do separate teams own separate checks on the same call? |
| | *The order decides* | Why plugin priority changes the outcome. |
| | *What the stack costs* | What latency does a governance stack add? |
| [Business compositions](business-compositions.md) | *Safe customer support* | Several plugins composed into one business outcome for a support assistant. |
| | *Outage containment* | Keeping an incident from cascading through agent traffic. |
| | *Trusted knowledge intake* | Letting agents read external knowledge without trusting it blindly. |
| | *Safe data operations* | Letting agents touch data without letting them damage it. |
| | *Controlled fulfillment* | Letting agents act on orders with interlocks that hold — even when a dependency is down. |

### Integration — TIBCO Flogo and BusinessWorks 5

| Page | Scenario | The question it answers |
|---|---|---|
| [Integration — Flogo & BW5](integration.md) | *Your Flogo agents, on the Hub* | How does a Flogo application become an MCP server, and its agents named participants in one chat? |
| | *Reason or execute* | Where do you draw the line between what the model decides and what actually acts — and why does that stop prompt injection structurally? |
| | *Keep the BW5 SOAP app, add MCP access* | How does an unchanged BW5 SOAP service become governed MCP tools? |

### Policy plugins — by what they protect

| Page | Plugins | Typical scenario |
|---|---|---|
| [Access, identity & credentials](policies-access.md) | UnifiedPDPPlugin, PermissionPlugin, JwtClaimsExtractionPlugin, SessionWriteGate, HeaderInjector, VaultPlugin | Your OPA decides which tools run; the caller's token decides what one shared assistant may do; the credential never enters the product. |
| [Data protection & privacy](policies-data-protection.md) | PIIFilterPlugin, SecretsDetection + EncodedExfilDetector, DenyListPlugin, FileTypeAllowlistPlugin, ResourceFilterExample, PrivacyNoticeInjector | The same record is masked for one role and visible to another; secrets and encoded data never leave in a tool answer. |
| [Content safety & threat protection](policies-content-safety.md) | SQLSanitizer, CodeSafetyLinterPlugin, ContentModeration, HarmfulContentDetector, URLReputationPlugin, VirusTotalURLCheckerPlugin, Provenance (RobotsLicenseGuard, CitationValidator) | The model wrote a `DROP`; a tool handed back dangerous code; a resource points at a lookalike host. |
| [Reliability, rate & cost control](policies-reliability-cost.md) | RateLimiterPlugin, token budget controls, CircuitBreaker, RetryWithBackoffPlugin, CachedToolResultPlugin, ResponseCacheByPrompt | An unattended agent exceeds its rate; a failing backend is not hammered; the bill nobody budgeted. |
| [Validation, contracts & normalization](policies-validation.md) | SchemaGuardPlugin, SPARCStaticValidator, ArgumentNormalizer, TimezoneTranslator | A downstream contract changed; a value valid yesterday is invalid today; "noon" meant a different noon. |
| [Response shaping & protocol bridging](policies-response-shaping.md) | SOAP branch, HTMLToMarkdownPlugin, Output utilities, Summarizer | A BW5 SOAP operation governed like any other tool; a vendor portal page reduced to what the agent needs. |
| [Observability & audit](policies-observability.md) | ToolsTelemetryExporter, SpanAttributeCustomizer | The one line an auditor asks for; the trace receipt in your own vocabulary. |

---

## Where to start

- **New to MCP Hub?** Read [Composing the plugin pipeline](plugin-pipeline.md) first — it explains hooks, order and
  modes, which every other page relies on.
- **Onboarding an existing estate?** Start with [Onboarding & tool lifecycle](capabilities-onboarding.md) and
  [Integration — Flogo & BW5](integration.md).
- **Security review?** Go to [Access, identity & credentials](policies-access.md),
  [Data protection & privacy](policies-data-protection.md) and
  [Content safety & threat protection](policies-content-safety.md), then
  [Identity, external policy & deployment](capabilities-identity-governance.md).
- **Want to see it end to end?** [Business compositions](business-compositions.md) shows several plugins composed
  into one business outcome.

## Further reading

- [TIBCO Platform](https://www.tibco.com/platform)
- [Model Context Protocol](https://modelcontextprotocol.io/)
- [TIBCO Developer Hub](https://github.com/TIBCOSoftware/tibco-developer-hub)
