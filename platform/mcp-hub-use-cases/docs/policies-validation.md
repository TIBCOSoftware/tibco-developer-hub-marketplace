# Policies — Validation, contracts & normalization

TIBCO MCP Hub policies can turn quiet data drift into a visible block, normalize values before a tool sees them, or translate a response into the shape the reader expects. This page covers validation and normalization controls that sit in the MCP Gateway plugin pipeline on a Virtual Server. Use it with [response-shaping plugins](policies-response-shaping.md) when the response itself must be repaired or re-written, and with [plugin ordering](plugin-pipeline.md) when hook, mode and priority decide which policy runs first.

| Plugin / capability | Scenario | Hook(s) | What it proves |
|---|---|---|---|
| `SchemaGuardPlugin` | A downstream shipment service renames a required result field. | After Tool (`tool_post_invoke`) for this case; arguments can also be checked Before Tool (`tool_pre_invoke`). | A declared result contract can block a malformed `structuredContent` result before an agent reasons over it. |
| `SPARCStaticValidator` | A value that was valid yesterday is no longer allowed today. | Before Tool (`tool_pre_invoke`) | A per-tool policy override is different from the gateway's built-in input validation, even when both look like a refused call to an agent. |
| `ArgumentNormalizer` | A supplier invoice uses European number and date formats against a backend that expects another shape. | Before Tool (`tool_pre_invoke`); also Prompt Fetch (`prompt_pre_fetch`) | Locale-like formats can be normalized deterministically on the gateway, but the defaults are guesses that must be scoped. |
| `TimezoneTranslator` | A tool returns noon UTC to a reader who works in New York. | Before Tool and After Tool; `direction` selects which acts: After Tool (`tool_post_invoke`) for `to_user`, Before Tool (`tool_pre_invoke`) for `to_server` | A timestamp can be translated on the wire with no backend or agent change, but a wrong assumed zone is confidently wrong. |

## SchemaGuardPlugin — The contract changed

> A downstream service still returns a successful response, but the field the agent depends on is no longer there.

**Example scenario.** A logistics integration team exposes a shipment lookup tool through TIBCO MCP Hub. The tool returns machine-readable `structuredContent`, and the agent expects `structuredContent.shipment.id` to exist. A routine downstream release renames that field to `shipmentId` while leaving the call successful and `isError:false`.

Without a gateway-side contract, the malformed record reaches the agent. The model may notice the missing field, or it may reason over the remaining record and answer with confidence. That variability is the risk: the data contract changed, but nothing in the normal tool result path guarantees that the agent sees the change as an error.

**What it proves.** `SchemaGuardPlugin` validates a tool result against an operator-declared schema and, in Block mode (`sequential`) with blocking enabled, stops a non-conforming result on `tool_post_invoke`. The positive control matters: an intact result passes, while the same tool's drifted result is blocked. That proves the policy is checking the declared contract, not simply failing every call.

**When to use it.** Use `SchemaGuardPlugin` when the required contract can be expressed as required fields and types over `structuredContent`. Do not use it as an attack detector, a redactor, a rate control, or a full JSON Schema engine. If the work is to repair or reshape an otherwise acceptable response, use the [response-shaping plugins](policies-response-shaping.md) instead.

### Where it sits

| Setting | Value |
|---|---|
| Plugin | `SchemaGuardPlugin` |
| Hook | After Tool (`tool_post_invoke`) for result validation; the same assignment also runs Before Tool (`tool_pre_invoke`), where it checks arguments only if `arg_schemas` is set |
| Mode | Block (`sequential`) |
| Attached to | Virtual Server |
| Main config key | `result_schemas` |

### Configuration

1. Open the governed Virtual Server and go to **Command Center** → **Tool calls**.
2. On the **After Tool** lane, add `SchemaGuardPlugin`.
3. Configure the result schema using the live federated tool name as the map key.
4. Set the execution mode to **Block** (`sequential`) and keep `block_on_violation` enabled.
5. Apply the binding, then **Push to Gateway** and wait for the Virtual Server to show `synced`.

```json
{
  "result_schemas": {
    "shipment-service-getshipment": {
      "type": "object",
      "required": ["structuredContent"],
      "properties": {
        "structuredContent": {
          "type": "object",
          "required": ["shipment"],
          "properties": {
            "shipment": {
              "type": "object",
              "required": ["id"],
              "properties": {
                "id": {
                  "type": "string"
                }
              }
            }
          }
        }
      }
    }
  },
  "block_on_violation": true
}
```

The `result_schemas` key must match the federated tool name exactly. If the key does not match, the binding can look correct while validating nothing for that tool. The schema shown uses the subset that this plugin enforces: `type`, `properties`, `required`, and array `items` over basic types.

### What happens

| Call | Outcome |
|---|---|
| Ungoverned Virtual Server, changed result | The renamed field reaches the caller as a successful tool result. |
| Governed Virtual Server, intact result | The result passes and the agent can use `shipment.id`. |
| Governed Virtual Server, changed result | The gateway blocks the result before the agent reasons over it. |

The block text returned to the MCP client is:

```text
tool_post_invoke blocked by plugin SchemaGuardPlugin: SCHEMA_GUARD_RESULT - Schema validation failed (Result does not conform to schema)
```

The message deliberately does not name the failed field. The operational record is the blocked MCP response and the gateway trace for the tool call; triage starts by comparing that response with the schema you declared.

### Setup tips

- **Mis-key symptom:** all calls pass even when the schema should fail. **Cause:** `result_schemas` uses the wrong federated tool name. **Fix:** copy the tool name from the Virtual Server's discovered tool list.
- **Advisory-only symptom:** violations are recorded but the result still reaches the client. **Cause:** the binding is not in a blocking mode. **Fix:** use **Block** (`sequential`) for a blocking contract.
- **Text-only symptom:** the policy cannot see the business fields. **Cause:** the tool returns data only in `content[].text`. **Fix:** return machine-readable `structuredContent`, or validate a coarser envelope only.
- **Stale runtime symptom:** the console shows the new binding but the gateway still behaves as before. **Cause:** changes were not pushed. **Fix:** push and wait for `synced`.

### Considerations

`SchemaGuardPlugin` is not a full JSON Schema validator. `enum`, `pattern`, `format`, numeric ranges, `oneOf`, and `additionalProperties` can be stored but are not enforced by this plugin. The block also omits per-field details, so the caller sees only that the result failed the declared schema. Use it for structural result contracts, and use more specific validation policies when value-level constraints are required.

## SPARCStaticValidator — The value that was valid yesterday

> A blocked call can come from the platform's own input validation or from a policy you configured; the agent sees both as a refusal.

**Example scenario.** A retail bank uses two MCP tools in an operations workflow. One account-status tool already declares its own native enum, such as `open` or `closed`. Another case-notes tool accepts a loose string, but the operations team now wants only `confirmed` or `disputed` to pass through that particular tool.

A value such as `pending` on the account-status tool and a value such as `under-review` on the case-notes tool both end in refusal on the governed Virtual Server. They are not the same control. The native enum is caught by the gateway's normal input validation before policy plugins run; the case-notes value is caught only because `SPARCStaticValidator` has an explicit per-tool override.

**What it proves.** `SPARCStaticValidator` enforces explicit `tool_schemas` overrides on `tool_pre_invoke`. It does not need to be credited for every rejected input. For a native-schema mismatch, the gateway's input-validation layer can reject the call before `SPARCStaticValidator` is entered. For a loose tool with a policy override, `SPARCStaticValidator` is the layer that blocks.

**When to use it.** Use this plugin when the tool schema is too loose for a business rule you need to enforce at the gateway. Do not use it as a general claim that every native tool schema is revalidated by SPARC. Read the refusal message before attributing ownership.

### Where it sits

| Setting | Value |
|---|---|
| Plugin | `SPARCStaticValidator` |
| Hook | Before Tool (`tool_pre_invoke`) |
| Mode | Block (`sequential`) |
| Priority | Catalogue default |
| Attached to | Virtual Server |
| Main config key | `tool_schemas` |

### Configuration

1. Open the governed Virtual Server that exposes the target tool.
2. Go to **Command Center** → **Tool calls** → Before Tool (`tool_pre_invoke`).
3. Add `SPARCStaticValidator` from the plugin catalog.
4. Configure `tool_schemas` for the single federated tool you intend to override.
5. Push the binding to the gateway and verify that only the governed Virtual Server changed.

```json
{
  "tool_schemas": {
    "case-notes-service-addnote": {
      "type": "object",
      "properties": {
        "query": {
          "type": "string",
          "enum": ["confirmed", "disputed"]
        }
      },
      "required": ["query"]
    }
  },
  "block_on_violation": true
}
```

Key the override to only the tool that needs it. A second tool on the same Virtual Server, such as an account-status tool with its own native enum, can remain absent from `tool_schemas` so the comparison stays clear.

### What happens

| Leg | Governed Virtual Server | Ungoverned Virtual Server |
|---|---|---|
| Native enum, allowed value | `status=open` passes. | `status=open` passes. |
| Native enum, disallowed value | Refused by the gateway's built-in input validation. | Refused with the same native-validation message. |
| Policy override, allowed value | `query=confirmed` passes. | `query=confirmed` passes. |
| Policy override, disallowed value | Blocked by `SPARCStaticValidator`. | The raw value passes to the tool. |

Native-schema refusal appears as a gateway input-validation error, for example:

```text
Input validation failed for tool 'account-service-updatestatus' at 'status': failed 'enum' constraint
```

The explicit override has the plugin name and policy code, for example:

```text
tool_pre_invoke blocked by plugin SPARCStaticValidator: SPARC_ENUM_VIOLATION - 'under-review' is not one of ['confirmed', 'disputed']
```

In Traces, both are blocked tool calls, but the message tells you which layer acted.

### Setup tips

- **False-attribution symptom:** a team assumes SPARC blocked a value even when no SPARC code appears. **Cause:** native input validation ran first. **Fix:** compare with an ungoverned twin; the native refusal should repeat.
- **Vacuous test symptom:** every test is a refusal. **Cause:** there is no allowed control call. **Fix:** run allowed and disallowed calls for both the native-schema tool and the override tool.
- **Over-broad override symptom:** unrelated tools begin failing. **Cause:** too many federated tool names are included. **Fix:** configure the exact tool name only.
- **No-change symptom:** the policy appears in the console but the gateway allows the value. **Cause:** binding was not pushed or the key does not match the live tool name. **Fix:** push, then read back the binding and tool name.

### Considerations

The two validation layers are sequential, not redundant. If the platform's own input validation rejects a call, `SPARCStaticValidator` may never run for that call. The plugin is most useful when a tool is technically valid but business-invalid, such as a string field that now has an approved vocabulary. Treat the refusal text as the source of truth.

## ArgumentNormalizer — The answer was almost right

> The model copied the invoice faithfully, the backend parsed it faithfully, and the ledger was still wrong.

**Example scenario.** A store assistant posts a supplier invoice into an ERP tool. The supplier's document shows an amount as `1.250,00` and a due date as `12.03.2026`. The backend expects a US-style decimal string and an ISO date. With no policy, the backend refuses the non-ISO date; if a person fixes only the date, the backend can parse the amount as `1.25`.

This is not an attack and not a refusal problem. It is a normalization problem between the way humans write values and the way an older backend parses strings. `ArgumentNormalizer` moves that translation into the gateway so the same rule runs every time.

**What it proves.** `ArgumentNormalizer` walks string values in the argument tree before the tool call. At its defaults, it normalizes Unicode, removes control characters, trims and collapses whitespace, normalizes numbers, and normalizes dates with `day_first:false`. In a reference test environment, `1.250,00` became `1250.00`, while `12.03.2026` became `2026-12-03` until `day_first:true` changed the date to `2026-03-12`.

**When to use it.** Use it when the backend accepts string arguments and you cannot change the backend parser. Do not use it as a contract check or security control; it transforms and does not block.

### Where it sits

| Setting | Value |
|---|---|
| Plugin | `ArgumentNormalizer` |
| Hook | Before Tool (`tool_pre_invoke`); also Prompt Fetch (`prompt_pre_fetch`) |
| Mode | Transform (`transform`) |
| Priority | `40` (catalogue default) |
| Attached to | Virtual Server |

### Configuration

1. Open the governed Virtual Server and go to **Command Center** → **Tool calls**.
2. On the **Before Tool** lane, add `ArgumentNormalizer`. One assignment activates the plugin on every hook it declares, including Prompt Fetch; the lane you choose is where the Hub shows it.
3. Start with the default config only in a controlled test, then scope the real deployment by field.
4. Apply the config, push to the gateway, and test with raw values that the model has not already reformatted.

The default configuration is empty, which activates the plugin's shipped defaults:

```json
{}
```

A safer production pattern is to disable broad parsing and opt fields in explicitly:

```json
{
  "day_first": true,
  "enable_numbers": false,
  "enable_dates": false,
  "field_overrides": [
    {
      "pattern": "(^|\\.)amount$",
      "enable_numbers": true,
      "decimal_detection": "auto"
    },
    {
      "pattern": "(^|\\.)due_date$",
      "enable_dates": true
    }
  ]
}
```

The override pattern is a regular expression searched against the argument's field path. On a tool call the path is prefixed with the tool name (for example `<tool-name>.amount`, or `<tool-name>.lines[0].amount` for nested values), so anchor on the field name with `(^|\.)amount$` rather than `^amount$`; a bare `^amount$` matches nothing on a tool call. Values not named by an override fall back to the top-level setting, so switch the global feature off before enabling it for selected fields.

### What happens

| Call | Result |
|---|---|
| Baseline, raw amount and date | The backend refuses the date with `due_date must be YYYY-MM-DD`. |
| Baseline, date fixed by hand | The backend accepts the call but records the amount as `1.25`. |
| Governed, defaults | The backend records the amount as `1250.0`, but the date is `2026-12-03`. |
| Governed, `day_first:true` | The backend records the amount as `1250.0` and the date as `2026-03-12`. |

The trace is a successful tool call when the normalized arguments satisfy the backend. That is why this policy deserves careful scoping: a wrong normalization can arrive as a successful business action.

### Setup tips

- **Model-confounder symptom:** the gateway appears to do nothing. **Cause:** the model already reformatted the value before the tool call. **Fix:** test with a tool-call trace that shows the raw argument values entering the gateway.
- **Wrong-month symptom:** `12.03.2026` becomes December 3. **Cause:** the default is `day_first:false`. **Fix:** set `day_first:true` when your documents use day-first dates.
- **Over-normalization symptom:** part numbers or references change unexpectedly. **Cause:** broad defaults parse every string they can. **Fix:** use `field_overrides` and disable global date and number parsing.
- **Ordering symptom:** a response repair plugin appears unrelated to argument handling. **Cause:** argument normalization runs before the tool, while response repair runs after it. **Fix:** review the policy sequence in [plugin ordering](plugin-pipeline.md).

### Considerations

Normalization is a deterministic guess, not knowledge of the supplier. `1.234` is ambiguous and stays untouched in the reference measurement, while `12.03.2026` is interpreted according to `day_first`. A value that looks parseable but belongs to a different domain can be changed unless you scope the policy. Use validation policies beside it when an incorrect value must be refused rather than rewritten.

## TimezoneTranslator — It said noon

> The tool answered correctly in UTC, but the reader needed the local clock.

**Example scenario.** A control-room dispatcher asks an assistant for a clearance window. The upstream operations tool returns `2026-06-07T12:00:00Z`. The dispatcher works in New York, where that instant is `2026-06-07T08:00:00-04:00`. Without translation, the agent can surface the raw UTC value and the human reader may plan around noon.

The backend did not fail, the model did not need to invent anything, and no policy should block the call. The required behavior is a deterministic conversion on the result path, so the value presented to the reader is already in the expected zone.

**What it proves.** `TimezoneTranslator` rewrites ISO-shaped timestamps inside strings. A `to_user` binding on the result path translated `2026-06-07T12:00:00Z` to `2026-06-07T08:00:00-04:00` in a reference test environment, while the baseline Virtual Server returned the raw UTC value. The plugin is non-blocking: it transforms and forwards, with no refusal path.

**When to use it.** Use it when a tool has a known fixed server time zone and readers need values in another IANA time zone. Do not use it when the tool already returns localized, offset-bearing values for the correct audience, or when the problem is a stale, missing, or semantically wrong field.

### Where it sits

| Setting | Value |
|---|---|
| Plugin | `TimezoneTranslator` |
| Hooks | Before Tool and After Tool (`tool_pre_invoke`, `tool_post_invoke`); `direction` selects which one acts |
| Mode | Transform (`transform`) |
| Attached to | Virtual Server |
| Scope | One binding handles one direction |

### Configuration

1. Open the governed Virtual Server and go to **Command Center** → **Tool calls**.
2. For result translation, add `TimezoneTranslator` on **After Tool** (`tool_post_invoke`). One assignment activates the plugin on every hook it declares; the lane you choose is where the Hub shows it, and `direction` decides which hook acts.
3. Set `direction` to `to_user`.
4. Enter the user and server IANA time zones.
5. Apply, push to the gateway, and verify the translated value in the trace or client response.

```json
{
  "direction": "to_user",
  "user_tz": "America/New_York",
  "server_tz": "UTC"
}
```

For argument translation in the other direction, set `direction` to `to_server`. A Virtual Server holds one `TimezoneTranslator` policy: attaching the same plugin a second time on the same Virtual Server merges into one policy, so two-way translation for the same tool needs the other direction on a separate Virtual Server. The optional `fields` filter applies to argument translation, not to result translation.

### What happens

| Virtual Server | Tool result surfaced to the client |
|---|---|
| Baseline | `2026-06-07T12:00:00Z` |
| Governed with `to_user` | `2026-06-07T08:00:00-04:00` |

The call still succeeds. Traces show a successful tool invocation with a transformed value on the result path, not a blocked security event.

### Setup tips

- **Half-configuration symptom:** results translate but arguments do not, or the reverse. **Cause:** one binding handles one direction, and a second attach on the same Virtual Server merges into the first. **Fix:** set `direction` for the path you need, and use a separate Virtual Server when the other direction is also required.
- **Wrong-zone symptom:** a naive timestamp converts to a plausible but wrong instant. **Cause:** a timestamp with no offset is assumed to be in `server_tz`. **Fix:** confirm the upstream tool's real time zone before configuring the binding.
- **Unexpected rewrite symptom:** an identifier that looks like a timestamp changes. **Cause:** matching is regex-based over strings. **Fix:** avoid result-path translation for payloads that mix timestamps with timestamp-shaped identifiers, or adjust the upstream payload.
- **Downstream matching symptom:** log rules or clients no longer match the old `Z` format. **Cause:** the plugin re-emits timestamps with an offset. **Fix:** update consumers to compare instants rather than string shapes.

### Considerations

`TimezoneTranslator` does not infer meaning. It finds ISO-shaped strings and converts them. On the result path, it cannot be narrowed by field, so use it only when broad result translation is acceptable. Unparseable candidates pass through unchanged, but a well-formed timestamp in the wrong assumed zone parses successfully and becomes a confidently wrong value.
