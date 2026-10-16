# Composing the plugin pipeline

TIBCO MCP Hub lets teams assemble a governed MCP endpoint from multiple policy plugins rather than treating each plugin as a standalone control. This page shows how plugins compose on a Virtual Server, why order and hook placement matter, and what a realistic stack costs in a reference test environment. The examples are intentionally practical: each case describes the business reason for the stack, the configuration shape, the observable result, and the limits to plan around.

| Plugin / capability | Scenario | Hook(s) | What it proves |
|---|---|---|---|
| Plugin pipeline | Hooks, priorities, modes, draft-to-sync rollout | Auth, tool, prompt, resource hooks | A plugin stack is an ordered request pipeline, not a flat checklist. |
| Fourteen-assignment stack | A realistic governed tool call | Auth, Before Tool, After Tool | Fourteen assignments (twelve distinct plugins) can compose on one Virtual Server and run as a single governed call. |
| Incremental governance | Controls added one operational problem at a time | Auth, Before Tool, After Tool | Stacks grow because each control answers a distinct risk. |
| Accountable desks | Separate teams own separate checks | Before Tool, After Tool, audit | One pipeline can reflect multiple accountability boundaries without merging them into one gate. |
| Priority ordering | A guard and a sanitizer on the same lane | After Tool and Resource Fetch examples | A single priority change can silently disarm a dependent control. |
| Latency and overhead | In-process checks plus an external policy engine | Permission and Before Tool | In a reference test environment, external policy lookup adds modest latency; the figures do not isolate the cost of individual plugins. |

## How the pipeline works

> "We need the same MCP endpoint to authenticate, authorize, validate, redact, shape, and record calls without every team owning its own gateway."

**Example scenario.** A Virtual Server exposes a curated MCP endpoint. When a client calls a federated tool, fetches a prompt, or reads a resource, the MCP Gateway runs the matching plugin lane. Authentication and permission checks happen on auth hooks. Tool controls run on `tool_pre_invoke` before the upstream tool is called and on `tool_post_invoke` after the upstream result returns. Prompt and resource controls use their own pre-fetch and post-fetch hooks, so a resource-only plugin is not part of a tool execution path.

Each assignment has a mode and a priority. One assignment activates the plugin on every hook it declares; the lane you choose is where the Hub shows it. Attaching the same plugin a second time on the same Virtual Server merges into one policy rather than adding a second one. Lower priority numbers run first on the same hook. Mode controls the plugin's posture: a blocking mode (`sequential`, shown as "Block" in the Hub) can refuse the call, `transform` lets a redaction or shaping plugin continue the call with modified data, `audit` observes without changing anything, and `disabled` turns the assignment off. A stack is therefore a sequence of decisions and transformations, not a set of independent toggles.

**What it proves.** The pipeline lets independent controls operate on one Virtual Server, but the placement is the control. A plugin that reads another plugin's output must run after the producer and before anything that removes evidence it needs.

**When to use it.** Compose the pipeline when one MCP endpoint needs several checks: identity, policy, input validation, data protection, output shaping, and observability. Do not assume every catalogue plugin protects every surface; some plugins are for resource or prompt fetches rather than tool calls.

### Where it sits

| Setting | Value |
|---|---|
| Attached to | Virtual Server |
| Common tool hooks | `tool_pre_invoke`, `tool_post_invoke` |
| Auth hooks used in the cases | `http_auth_resolve_user`, `http_auth_check_permission` |
| Other governed paths | Prompt Fetch and Resource Fetch |
| Rollout state | Edit as draft, Push to Gateway, wait for `synced` |
| Ordering | Lower priority number runs first on the same hook |
| Hook scope | One assignment activates the plugin on every hook it declares |

### Configuration

1. Attach each plugin to the Virtual Server. One assignment activates the plugin on every hook it declares; the lane you choose is where the Hub shows it, so choose the lane that matches the data the plugin must inspect.
2. Choose the mode for the risk: Block (`sequential`) for controls that must refuse, Transform (`transform`) for masking and shaping, and Audit (`audit`) only when observation is intentional.
3. Set priorities explicitly when two plugins share a hook or one depends on another's output.
4. Apply the binding configuration.
5. Push to Gateway and wait until the Virtual Server reports `synced` before testing clients.

Important pipeline rules from the cases:

- A fail-closed gate such as [PermissionPlugin](policies-access.md), [UnifiedPDPPlugin](policies-access.md), or [SchemaGuardPlugin](policies-validation.md) can refuse before the upstream tool runs.
- A shaping or protection plugin such as [PIIFilterPlugin](policies-data-protection.md), [SecretsDetection](policies-data-protection.md), [EncodedExfilDetector](policies-data-protection.md), [HTMLToMarkdownPlugin](policies-response-shaping.md), or [OutputLengthGuardPlugin](policies-reliability-cost.md) may transform data instead of refusing it, depending on its mode and configuration.
- Observability plugins such as [ToolsTelemetryExporter](policies-observability.md) add evidence, but an audit binding that fires only on success or error is not a substitute for a security block event.
- If an unexpected transport error occurs, classify it as an error. Do not label every failure as a policy block, because that hides outages.

### What happens

A request moves through the applicable hooks in priority order. If a plugin refuses, later tool execution and output-side plugins may never run. If a plugin transforms, the next plugin reads the transformed value. If a plugin observes, it records what its hook can see. Draft changes do not govern traffic until they are pushed and synced.

### Setup tips

- Check each plugin's declared hooks before attaching it. The Hub rejects an assignment to a hook the plugin does not declare, and the error names both the plugin and the hook.
- Remember that a plugin declaring several hooks runs on all of them, even if you attached it on one lane. Attaching it again on another lane does not create a second policy.
- Test both an allowed leg and a refused leg. A refusal-only test cannot prove that a gate is configured correctly.
- For any composed lane, read the pushed priorities back and then test through the gateway.
- Treat a silent success after an ordering change as suspicious when a guard depends on content that another plugin can remove.

### Considerations

The pipeline cannot answer risks that are outside the bytes crossing the gateway. For example, an agent may combine individually authorized facts elsewhere and infer something sensitive. Plugin policy still needs model, application, and operational controls around it.

## Fourteen-assignment stack — one governed call

> "We tested plugins one at a time, but production will not run one plugin at a time."

**Example scenario.** An enterprise Virtual Server fronts tools for systems such as customer service, invoicing, warehouse inventory, and order management. Different agents use the same endpoint: a service assistant acting for a named person, an invoicing agent, an unattended warehouse monitor, and a partner agent outside the organization. The platform team wants to know whether a realistic stack composes before they discuss tuning it.

The reference stack is fourteen Hub assignments: two identity gates, seven input-side checks, and five output-side checks. [PIIFilterPlugin](policies-data-protection.md) and [SecretsDetection](policies-data-protection.md) each appear on both the Before Tool and After Tool lanes; because one assignment already activates a plugin on every hook it declares, the gateway receives twelve distinct plugin policies. The stack is chosen for business coverage, not to create a benchmark.

**What it proves.** Fourteen plugin assignments (twelve distinct plugins) can be active on one Virtual Server, push to `synced`, and run as one governed request path. In the measured run, the Hub showed 14 active assignments, 14 Virtual Server defaults, and zero tool overrides, and nothing failed because of composition.

**When to use it.** Use this pattern when a single client-facing MCP endpoint must carry identity, external policy, input validation, data protection, output shaping, and telemetry. Do not use this measurement as a throughput claim; it is one caller, one call at a time.

### Where it sits

| Setting | Value |
|---|---|
| Capability | Multi-plugin Virtual Server stack |
| Attached to | Virtual Server |
| Auth stage | [JwtClaimsExtractionPlugin](policies-access.md), [PermissionPlugin](policies-access.md) |
| Before Tool | [UnifiedPDPPlugin](policies-access.md), [RateLimiterPlugin](policies-reliability-cost.md), [ArgumentNormalizer](policies-validation.md), [SQLSanitizer](policies-content-safety.md), [SchemaGuardPlugin](policies-validation.md), [PIIFilterPlugin](policies-data-protection.md), [SecretsDetection](policies-data-protection.md) |
| After Tool | [PIIFilterPlugin](policies-data-protection.md), [SecretsDetection](policies-data-protection.md), [EncodedExfilDetector](policies-data-protection.md), [OutputLengthGuardPlugin](policies-reliability-cost.md), [ToolsTelemetryExporter](policies-observability.md) |
| Mode | Shipped defaults from the plugin catalogue in the reference run |

### Configuration

Build the stack from the plugin catalogue rather than from copied constants:

1. Start with a Virtual Server that requires authentication and has federated tools.
2. Add auth bindings for JWT claim resolution and permission checking.
3. Attach each plugin once: policy, rate limiting, normalization, SQL safety and schema validation on **Before Tool**; PII filtering, secret detection, encoded-exfiltration detection, output length handling and telemetry on **After Tool**. PIIFilterPlugin and SecretsDetection inspect both arguments and results through their declared hooks, so they need no second assignment.
4. Review the pipeline view to confirm each plugin appears on every hook it declares.
5. Push each rung of the stack and wait for `synced` before measuring or adding more controls.

The measured stack, as assigned in the Hub, is:

| # | Plugin | Lane (hook) | Stage |
|---:|---|---|---|
| 1 | [JwtClaimsExtractionPlugin](policies-access.md) | `http_auth_resolve_user` | Auth |
| 2 | [PermissionPlugin](policies-access.md) | `http_auth_check_permission` | Auth |
| 3 | [UnifiedPDPPlugin](policies-access.md) | `tool_pre_invoke` | Input |
| 4 | [RateLimiterPlugin](policies-reliability-cost.md) | `tool_pre_invoke` | Input |
| 5 | [ArgumentNormalizer](policies-validation.md) | `tool_pre_invoke` | Input |
| 6 | [SQLSanitizer](policies-content-safety.md) | `tool_pre_invoke` | Input |
| 7 | [SchemaGuardPlugin](policies-validation.md) | `tool_pre_invoke` | Input |
| 8 | [PIIFilterPlugin](policies-data-protection.md) | `tool_pre_invoke` | Input |
| 9 | [SecretsDetection](policies-data-protection.md) | `tool_pre_invoke` | Input |
| 10 | [PIIFilterPlugin](policies-data-protection.md) | `tool_post_invoke` | Output |
| 11 | [SecretsDetection](policies-data-protection.md) | `tool_post_invoke` | Output |
| 12 | [EncodedExfilDetector](policies-data-protection.md) | `tool_post_invoke` | Output |
| 13 | [OutputLengthGuardPlugin](policies-reliability-cost.md) | `tool_post_invoke` | Output |
| 14 | [ToolsTelemetryExporter](policies-observability.md) | `tool_post_invoke` | Output |

The lane column records where each assignment was made. At run time, each plugin runs on every hook it declares: rows 8 and 10 are one PIIFilterPlugin policy, rows 9 and 11 are one SecretsDetection policy, and a plugin such as ToolsTelemetryExporter also runs on `tool_pre_invoke`.

### What happens

!!! note "Reference measurements"
    Figures on this page come from an internal reference test environment (one gateway, one caller,
    test fixtures). They illustrate relative behaviour; they are not performance commitments. Measure
    in your own environment.

In the reference environment, the empty-stack p50 was 210.1 ms and the full 14-assignment stack p50 was 458.3 ms: the complete stack added about 248 ms at p50; these figures do not isolate the cost of individual plugins. Payload size was a second-order effect in the same environment: moving from 170 B to 58 KB increased the empty-stack floor by 26.6 ms and the 14-assignment (12-plugin) stack's added cost from 266.6 ms to 329.1 ms.

A refusal is not the same call shape as an allowed request. A tripping payload that contained a detected secret was blocked before the tool ran:

```text
tool_pre_invoke blocked by plugin SecretsDetection: SECRETS_DETECTED -
Secrets detected (Matched detectors: aws_access_key_id)
```

That blocked request was cheaper than a completed call because the upstream tool and output-side plugins did not run.

### Setup tips

- Assert an allowed leg in every measured configuration. A zero-allowed arm is a broken control, not a latency measurement.
- Account for [RateLimiterPlugin](policies-reliability-cost.md) call budgets during tests. Set `by_user` explicitly and verify it; the default depends on your gateway's plugin configuration. In the reference run, a `60/m` limit refused 4 of 20 calls on one rung.
- Vary payload with an empty-stack control if you need to separate payload cost from plugin overhead.
- Use each catalogue entry's supported hook list and default configuration summary.

### Considerations

This case does not measure throughput, WAN policy engines, payloads above 58 KB, or ordering as an independent variable. It also shows catalogue shape: some plugins are not part of a tool call at all. Resource or prompt plugins such as [HeaderInjector](policies-access.md), [FileTypeAllowlistPlugin](policies-data-protection.md), [PrivacyNoticeInjector](policies-data-protection.md), [DenyListPlugin](policies-data-protection.md), [URLReputationPlugin](policies-content-safety.md), and [HTMLToMarkdownPlugin](policies-response-shaping.md) protect the surfaces they support, not every request path.

## Incremental governance — one problem at a time

> "Nobody selected fourteen controls in one meeting; the stack grew because each incident exposed a different gap."

**Example scenario.** An illustrative logistics company runs several MCP servers over customer service, invoicing, warehouse inventory, and order management. Four agent populations use them: a service assistant with an end-customer context, an invoicing agent, an unattended warehouse monitor, and an external partner agent. Each population fails differently, so the organization adds controls for distinct risks rather than for plugin coverage.

The security team already keeps its authorization rules in its own policy engine and wants the gateway to ask it rather than duplicate the rules. Operations wants a ceiling on how often one agent can call a backend in a short window. Data protection wants both input and output inspection because filtering the prompt does not stop a tool from returning sensitive data. Security adds encoded-exfiltration detection because a fair question can produce an answer that hides data in base64. Audit asks who called which tool and what happened.

**What it proves.** A large plugin stack is easier to justify when each assignment maps to a problem that a non-platform owner recognizes. The stack is not "fourteen technologies"; it is a record of risks that arrived separately.

**When to use it.** Use this framing when architecture review focuses on why the stack has so many assignments. Use the measurement case above when the question is cost, and the ordering case below when the question is placement.

### Where it sits

| Setting | Value |
|---|---|
| Capability | Problem-to-plugin mapping |
| Attached to | Virtual Server |
| Identity and access | [UnifiedPDPPlugin](policies-access.md), [PermissionPlugin](policies-access.md), JWT claim extraction |
| Reliability | [RateLimiterPlugin](policies-reliability-cost.md) |
| Data protection | [PIIFilterPlugin](policies-data-protection.md), [SecretsDetection](policies-data-protection.md), [EncodedExfilDetector](policies-data-protection.md) |
| Validation | [ArgumentNormalizer](policies-validation.md), [SQLSanitizer](policies-content-safety.md), [SchemaGuardPlugin](policies-validation.md) |
| Response shaping and observability | [OutputLengthGuardPlugin](policies-reliability-cost.md), [ToolsTelemetryExporter](policies-observability.md) |

### Configuration

Convert each business problem into a binding decision:

1. Identify the owner of the risk: identity, policy, reliability, validation, data protection, response shaping, or audit.
2. Attach the plugin to the hook that sees the evidence for that risk.
3. Choose a mode that matches the consequence: refuse, redact, shape, or observe.
4. Set a priority that preserves dependencies.
5. Push and test the allowed and blocked paths.

A practical first inventory looks like this:

| Problem | Binding response |
|---|---|
| "We keep authorization in our own policy engine." | [UnifiedPDPPlugin](policies-access.md) on every tool call that needs the external decision. |
| "An agent can overwhelm a backend." | [RateLimiterPlugin](policies-reliability-cost.md), with a production-specific call budget. |
| "The leak can be in the answer, not the question." | [PIIFilterPlugin](policies-data-protection.md) and [SecretsDetection](policies-data-protection.md); each declares both input and output tool hooks, so one assignment inspects both. |
| "Encoded data can leave in an otherwise valid answer." | [EncodedExfilDetector](policies-data-protection.md) on the output hook. |
| "We need to know what was called." | [ToolsTelemetryExporter](policies-observability.md) after the call. |
| "The tool input and output must stay within expected shape." | [ArgumentNormalizer](policies-validation.md), [SQLSanitizer](policies-content-safety.md), [SchemaGuardPlugin](policies-validation.md), and output utilities. |

### What happens

The console shows the result as active assignments on one Virtual Server, grouped by the lane each was added to: two on the identity path, seven on Before Tool, and five on After Tool. The lanes are a display grouping; a plugin that declares both tool hooks runs on both. Four of twenty calls were refused by the rate limit during one rung, which is policy working, not composition failure.

### Setup tips

- Start with the business sentence, not the plugin name. This keeps the stack explainable during review.
- Tune defaults before production, in particular the per-user rate limit and the long-output control, which shortens rather than refuses.
- Treat telemetry as evidence about the paths it can observe, not a complete record of every blocked request.

### Considerations

The stack does not prove every future risk is covered. It proves that multiple assignments can coexist and that each one should have an accountable reason. Keep ownership visible so future teams know whether a proposed change removes a reliability guard, an authorization gate, a data-protection control, or an audit control.

## Accountable desks — eight checks for one errand

> "The controls are separate because different teams are accountable for different questions."

**Example scenario.** An agent in a regulated environment needs a signed qualification report for a delivery decision. The request sounds simple, but it crosses several desks: identity verifies whose name is on the request, personnel security checks what the sponsor can access, the programme office decides whether the errand is in scope, operations prevents overload, data protection inspects what enters and leaves, configuration management checks that required evidence is present, and audit records the result.

The key point is separation. The badge desk should not decide whether the assignment is valid. The rate limit should not decide what the document means. The schema gate should not decide whether the user is cleared. The pipeline lets each desk answer only its own question.

**What it proves.** A plugin pipeline can model accountability boundaries. The stack can contain repeated inspections on the way in and the way out because those are different moments with different evidence.

**When to use it.** Use this pattern for regulated workflows where a single MCP request must satisfy identity, authorization, scoping, load, input hygiene, configuration evidence, output protection, and logging. Do not present it as one scanner that solves all safety concerns; agents can still combine valid facts outside the gateway.

### Where it sits

| Setting | Value |
|---|---|
| Capability | Accountable multi-step governance |
| Attached to | Virtual Server |
| Stop shown in the case | [SchemaGuardPlugin](policies-validation.md) on `tool_pre_invoke` |
| Mode | Fail closed for missing required evidence |
| Observable refusal | Schema validation failure before tool execution |

### Configuration

The stop in this case requires a tool argument named `build`. The tool name below is illustrative; use the federated `<server>-<tool>` name your Virtual Server exposes:

```json
{
  "arg_schemas": {
    "evidence-records-get-report": {
      "type": "object",
      "required": ["build"],
      "properties": {
        "build": { "type": "string" },
        "query": { "type": "string" }
      }
    }
  },
  "block_on_violation": true
}
```

Important keys:

- `arg_schemas` maps a federated tool name to the JSON schema expected for that tool's arguments.
- `required` lists evidence the request must declare before the tool is invoked.
- `block_on_violation: true` turns the schema check into a fail-closed gate.

### What happens

Same identity, same tool, same endpoint, one field different:

| Request | Allowed | p50 in the reference environment |
|---|---:|---:|
| Declares the build | 3/3 | 459.0 ms |
| Omits the build | 0/3 | 223.9 ms |

The missing-field call is refused before the upstream tool runs:

```text
tool_pre_invoke blocked by plugin SchemaGuardPlugin:
SCHEMA_GUARD_ARGS - Schema validation failed (Arguments do not conform to schema)
```

Both legs matter. A refusal alone could mean the tool is unreachable or the control is misconfigured. The allowed leg proves the desk is checking a required field, not acting as a wall.

### Setup tips

- Test the allowed and refused versions in the same configuration, one argument apart.
- After renaming a Virtual Server, update every client URL and IdP audience entry that points at the old server URL. A rename issues a new gateway endpoint URL.
- If you use external identity, mint tokens for the Virtual Server URL currently under test.
- Keep the schema check focused on declared request evidence. This case validates the argument shape; it does not compare two documents.

### Considerations

No plugin can determine everything an agent may infer after it receives authorized data. The gateway can inspect the request and response it sees; it cannot control all reasoning outside the request path. This case also carries the same measurement limits as the larger stack: one caller, sequential calls, and no throughput claim.

## Priority ordering — the order decides

> "Every plugin can be enabled, and the guard you rely on can still read the wrong version of the content."

Run RobotsLicenseGuard before HTML sanitizers and other transforms so it evaluates the original metadata.

**Example scenario.** A governed Virtual Server uses a composed stack: inbound OAuth, [PermissionPlugin](policies-access.md), [SchemaGuardPlugin](policies-validation.md), [RateLimiterPlugin](policies-reliability-cost.md), [PIIFilterPlugin](policies-data-protection.md), and a WebhookNotification binding for audit. A baseline endpoint with the same tools but no plugins returns a customer record containing email and phone data in the clear. If your gateway enables PIIFilterPlugin gateway-wide, the baseline is masked too; compare against a server where it is not bound. The governed endpoint returns the same record with PII redacted.

That proves composition works, but it does not prove every ordering is safe. The sharper case uses a resource fetch with a license/provenance guard and an HTML sanitizer. The guard must inspect the policy marker before the sanitizer removes markup, and the test moves only the guard's priority.

**What it proves.** Priority is not cosmetic. It is the composition. A wrong priority can silently disarm a dependent control while every plugin still appears enabled.

**When to use it.** Review priorities whenever two bindings share a hook, especially when one plugin reads evidence another plugin may transform or remove. This applies to data protection, provenance, validation, and response shaping pipelines.

### Where it sits

| Setting | Value |
|---|---|
| Capability | Same-hook ordering |
| Tool stack example | Auth and Before Tool / After Tool governance |
| Ordering example | Resource Fetch post-processing |
| Priority rule | Lower number runs first |
| Required order | `RobotsLicenseGuard @60` before `SafeHTMLSanitizer @70` |

### Configuration

For the composed tool path, set priorities in dependency order. Priority orders plugins only among those running on the same hook; the auth hooks always run before any tool hook, whatever their numbers.

```text
http_auth_*        OAuth identity, then PermissionPlugin
tool hooks         SchemaGuardPlugin        priority 20
                   RateLimiterPlugin        priority 30
                   PIIFilterPlugin          priority 40
                   WebhookNotification      priority 50
```

Each plugin runs on the tool hooks it declares, in this relative order on each of them.

For the resource fetch dependency, give RobotsLicenseGuard a lower number than HTML sanitizers and other resource transforms:

```text
RobotsLicenseGuard @60  ->  SafeHTMLSanitizer @70
```

Confirm the effective order after pushing.

### What happens

With the guard ahead of the sanitizer, it reads the policy marker and blocks the restricted resource. Placed after a transform that removes that marker, it has nothing to evaluate.

On the tool stack, the practical contrast is equally direct: the same prompt and same model ask for contact data. The unguided path returns email and phone values in clear text; if your gateway enables PIIFilterPlugin gateway-wide, that baseline is masked too, so compare against a server where it is not bound. The governed path returns redacted values such as `[PII_REDACTED]` because the output-side [PIIFilterPlugin](policies-data-protection.md) runs before the final answer is exposed.

### Setup tips

- When a binding joins a lane that already has plugins, confirm its priority after Apply and again after Push to Gateway.
- Change one priority at a time during order tests.
- Classify resource tests structurally: a response body is served, a guard-specific refusal is blocked, and any other failure is an error.
- Do not rely on audit alone to catch ordering defects. A Before Tool block ends the pipeline before an After Tool hook runs.

### Considerations

A disarmed guard is quiet because the call succeeds. You cannot detect a bad order by watching only for errors. The ordering must be reviewed as part of configuration, tested with inputs that distinguish the controls, and rechecked whenever a new plugin joins the same hook.

## Latency and overhead — what the stack costs

> "What does each policy check add to a tool call, and is an external policy engine the expensive part?"

**Example scenario.** A platform team is weighing whether to consult its own policy engine on every MCP tool call. They need to know whether the network callout is the expensive part, whether a decision cache changes the answer, and how to test the policy settings against a slow engine.

The reference measurement separates the floor, an in-process permission check, an external policy check to an in-cluster OPA engine, and both gates together. Calls are issued from inside the cluster, deltas are compared rather than absolutes, and every configuration includes an allowed leg so a fail-closed gate cannot masquerade as a valid latency number.

**What it proves.** In the reference environment, an external in-cluster policy engine added about 27 ms p50 over the floor. The full two-gate path added about 86 ms. In the reference environment the complete stack added about 248 ms at p50; these figures do not isolate the cost of individual plugins.

**When to use it.** Use this case for latency planning, not throughput planning. It helps answer whether external policy lookup is viable for a sequential caller. It does not answer requests-per-minute capacity or WAN policy-engine behavior.

### Where it sits

| Setting | Value |
|---|---|
| Capability | Latency measurement for composed policy gates |
| Gate one | [PermissionPlugin](policies-access.md), in process |
| Gate two | [UnifiedPDPPlugin](policies-access.md) to OPA |
| Hook relationship | Different hooks, not two steps on one hook |
| Cache variable | Fresh session per call versus one reused session |
| Measurement posture | Sequential calls, one caller, inside the cluster |

### Configuration

This case uses the existing two-gate chain and changes only which bindings are active for measurement:

1. Measure the same tool with no plugins to establish the floor.
2. Enable [PermissionPlugin](policies-access.md) only.
3. Enable [UnifiedPDPPlugin](policies-access.md) only.
4. Enable both gates.
5. Repeat with fresh sessions and a reused session to bound possible decision-cache savings.
6. Test slow-policy behavior with an engine that accepts the request and responds after a delay, not with an unreachable URL.

### What happens

| Configuration | p50 in the reference environment | p95 | Added vs. floor |
|---|---:|---:|---:|
| No plugins | 217.3 ms | 244.7 ms | — |
| Gate one only | 238.6 ms | 270.9 ms | +21 ms |
| Gate two only | 244.5 ms | 265.7 ms | +27 ms |
| Both gates | 303.2 ms | 337.1 ms | +86 ms |

The two-gate total is more than 21 + 27 ms. The gates run on different hooks.

The decision-cache test bounded savings below 10 ms in the same environment:

| Configuration | Fresh session per call | One session reused |
|---|---:|---:|
| Gate two only | 244.5 ms | 239.3 ms |
| Both gates | 303.2 ms | 295.5 ms |

Set the evaluation timeout and retries explicitly, and test them against a policy engine that answers slowly before relying on them.

### Setup tips

- Measure from the same network location as the gateway path you care about. Laptop-to-cluster tunnels can swamp plugin deltas.
- Use deltas with the same tool, arguments, identity, and endpoint.
- Keep an allowed leg in every configuration.
- To test timeout behavior, use a slow responder that accepts the connection and then waits.
- Verify restore by calling the tool, not only by reading the configuration you just wrote.

### Considerations

These figures are not universal latency promises. They come from one reference test environment with one caller and sequential calls. A policy engine across a WAN, higher concurrency, different payloads, or different plugin order can change the result.
