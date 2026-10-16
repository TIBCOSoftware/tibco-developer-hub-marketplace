# Policies — Observability & audit

Observability policies create the record that explains what an agent-facing tool call did and how that record is named. They do not usually change the client-visible answer; their value is in traces, receipts, and audit fields that downstream teams can query. For identity and call-accountability concepts, see [identity and governance capabilities](capabilities-identity-governance.md); for how priorities and hooks interact, see the [plugin pipeline](plugin-pipeline.md).

| Plugin / capability | Scenario | Hook(s) | What it proves |
|---|---|---|---|
| `ToolsTelemetryExporter` | An auditor needs the record of one specific tool call | Before Tool (`tool_pre_invoke`), After Tool (`tool_post_invoke`) | A governed call emits a two-span receipt with tool, arguments, server, request ID, and optional result content |
| `SpanAttributeCustomizer` | Trace readers need the receipt in their own vocabulary | Before Tool (`tool_pre_invoke`); also declares Resource pre-fetch (`resource_pre_fetch`) and post-fetch (`resource_post_fetch`) | Static business attributes, renamed keys, and removed gateway internals can be applied to the gateway's `tool.invoke` span |

## ToolsTelemetryExporter — The line the auditor asks for

> The call worked and nothing was blocked, but months later someone must reconstruct exactly what the assistant did.

**Example scenario.** A regulated operations team uses an assistant that calls account, case, or service tools through MCP Gateway. Later, an auditor asks for the record of one call: which tool ran, with what arguments, against which upstream server, and what the tool returned. Aggregate dashboards cannot answer that question.

`ToolsTelemetryExporter` writes the per-call receipt into the trace backend. It emits `tool.pre_invoke` and `tool.post_invoke` spans around the governed tool call. The Hub can read those traces through its observability proxy once the Hub's observability connection to your trace backend is configured, while the client receives the same tool answer as before. The exporter's spans go to the trace backend; they do not appear in the gateway's built-in trace store.

**What it proves.** A governed Virtual Server can leave a two-span OpenTelemetry receipt for every tool invocation. The baseline path runs the same tool but has no exporter receipt. The difference is visible in traces, not in the user's chat response.

**When to use it.** Use it when you need per-call audit or forensic reconstruction and you already operate an OpenTelemetry collector and trace store. Do not use it as an identity proof by itself: the receipt proves what the service did; confirm which caller identity your data path records before relying on it for attribution.

### Where it sits

| Setting | Value |
|---|---|
| Plugin | `ToolsTelemetryExporter` |
| Hooks | Before Tool (`tool_pre_invoke`), After Tool (`tool_post_invoke`) |
| Mode | `transform` or another inline, state-preserving mode |
| Priority | 200 in the reference environment |
| Attached to | Virtual Server |
| Trace destination | OTLP collector such as `https://<your-otel-collector>:4318`, then a trace backend the Hub can read |

### Configuration

1. Configure the MCP Gateway's OpenTelemetry export to your collector, for example `https://<your-otel-collector>:4318`.
2. In MCP Hub, open the Virtual Server that needs call receipts.
3. In Command Center, add `ToolsTelemetryExporter` on the observability/tool-call path.
4. Enable full payload export only if returned content is allowed in your trace backend.
5. **Push to Gateway** and wait for `synced`.

```json
{
  "export_full_payload": true,
  "max_payload_bytes_size": 100000
}
```

The shipped defaults are `export_full_payload: false` and `max_payload_bytes_size: 10000`.

`export_full_payload: false` records metadata and errors without the returned content. When it is `true`, the post span can include the raw result, truncated at `max_payload_bytes_size` with a truncation suffix. To keep sensitive arguments out of traces, mask them with a data-protection plugin that runs before the exporter.

### What happens

A governed tool call emits a pre span with fields such as `tool.name`, `tool.target_tool_name`, `tool.description`, `tool.invocation.args`, `target_mcp_server.name`, `target_mcp_server.url`, `request_id`, `user`, `tenant_id`, and `server_id`. Header values are redacted by pattern.

The post span carries the same `request_id`, whether the invocation had an error, and, when enabled and successful, the result content. The `request_id` is the practical link between pre and post. The plugin never blocks the call and never rewrites the request or response; if the collector is unavailable, the call still succeeds and the receipt is simply missing.

### Setup tips

- Put data-protection plugins before the exporter if arguments or results can contain sensitive content.
- Keep the Hub's default mode, Transform (`transform`), or another inline mode such as `sequential`. The `audit` and `fire_and_forget` modes can run outside the active recording span and may drop the exporter spans.
- Verify the collector and Hub observability proxy before relying on receipts. A successful tool response is not proof that the trace was stored.
- Search traces by service and operation, then use `request_id` to connect pre and post spans.

### Considerations

The exporter records in real time only; it does not buffer and retry locally. It records what happened mechanically; the human caller is identified only if your data path supplies that identity. Full-payload mode writes result content. Treat the trace backend as a sensitive data store and align retention, access, and masking accordingly.

## SpanAttributeCustomizer — The receipt, in your words

> The receipt exists, but the people who read it need cost centre, data class, and legal basis, not gateway implementation keys.

**Example scenario.** An illustrative insurer uses an assistant against claims tools: claim lookup, billing summary, and fraud alerts. The trace store is read by several teams. Finance wants cost centre, the security office wants data class, and a compliance liaison wants the declared legal basis for sensitive lookups.

`SpanAttributeCustomizer` stamps those operator-owned attributes onto the gateway's own `tool.invoke` span. It can add global attributes, override them per tool, rename gateway keys, and remove gateway internals. The tool answer does not change.

**What it proves.** Static configuration can rewrite the observability vocabulary for the main gateway span. In the reference environment, with an equivalent configuration, the governed receipt carried `insurer.cost_centre`, `insurer.data_class`, `insurer.legal_basis`, `insurer.service`, and `insurer.protocol`, while the baseline receipt carried `tool.name`, `tool.id`, `tool.gateway_id`, `has_headers`, and `arguments_count`.

**When to use it.** Use it when your observability or audit program already has naming standards and tool metadata that the gateway cannot infer. Do not use it to redact payloads or identify callers; it reads static configuration, not request content or user identity.

### Where it sits

| Setting | Value |
|---|---|
| Plugin | `SpanAttributeCustomizer` |
| Hooks | Before Tool (`tool_pre_invoke`), Resource pre-fetch (`resource_pre_fetch`), Resource post-fetch (`resource_post_fetch`) |
| Mode | `transform` |
| Priority | 10 in the reference environment |
| Attached to | Virtual Server |
| Affected span | Gateway `tool.invoke` span at span close |

### Configuration

1. List the federated tool names exposed by the Virtual Server. Per-tool overrides must use those names, not the upstream function names.
2. In Command Center, add `SpanAttributeCustomizer` on **Tool calls → Before Tool**. One assignment activates the plugin on every hook it declares, including its resource pre-fetch and post-fetch hooks; the lane you choose is where the Hub shows it.
3. Keep the mode as `transform` so plugin state is merged back into the active span context.
4. Type the static attributes, overrides, mappings, and removal list.
5. **Push to Gateway** and wait for `synced`.

```json
{
  "global_attributes": {
    "insurer.cost_centre": "CLM-4410",
    "insurer.data_class": "internal"
  },
  "tool_overrides": {
    "claims-claim-lookup": {
      "attributes": {
        "insurer.data_class": "claimant-personal",
        "insurer.legal_basis": "contract"
      }
    },
    "claims-billing-summary": {
      "attributes": {
        "insurer.data_class": "billing"
      }
    }
  },
  "attribute_mapping": {
    "tool.name": "insurer.service",
    "tool.integration_type": "insurer.protocol"
  },
  "remove_attributes": [
    "tool.id",
    "tool.gateway_id",
    "has_headers",
    "arguments_count"
  ]
}
```

Global attributes apply to every matching call. Tool overrides win on shared keys. Attribute mapping renames keys, and the removal list drops the original gateway internals from the affected span.

### What happens

For a governed call to `claims-claim-lookup`, the `tool.invoke` span carries `insurer.cost_centre: CLM-4410`, `insurer.data_class: claimant-personal`, `insurer.legal_basis: contract`, `insurer.service: claims-claim-lookup`, and `insurer.protocol: MCP`. The removed keys are absent. A billing-summary call receives the billing data class, while a tool without an override keeps the global `internal` class.

The baseline span keeps the gateway vocabulary. Both paths return the same tool content. If `ToolsTelemetryExporter` is also bound, to keep sensitive arguments out of traces, mask them with a data-protection plugin that runs before the exporter.

### Setup tips

- Key `tool_overrides` on federated names such as `<server>-<tool>`. An upstream function name that does not match is ignored.
- Keep the customizer before receipt exporters when you want the main `tool.invoke` span ready by span close.
- Avoid audit-style modes for this plugin. They can execute on a discarded context and leave the receipt unchanged.
- Confirm both the trace export and any DB-backed trace view you rely on, because the customizer applies when the gateway closes the span.
- On resource spans the customizer adds and renames attributes, but its removal list does not apply there. The reference measurement covered the `tool.invoke` span only.

### Considerations

The customizer is static. It does not inspect arguments, results, headers, or the caller. It rewrites the gateway's own span, not every span in the trace, and fields written after the transform, such as success and duration, remain. Like the exporter, it depends on the real-time trace pipeline; if the collector is unavailable, the customized receipt is lost with no local retry.
