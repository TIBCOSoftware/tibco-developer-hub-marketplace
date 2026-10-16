# Policies — Response shaping & protocol bridging

Response-shaping policies change what an agent receives without changing the upstream service contract. They can bridge older protocols, reduce bulky resources, tidy text, or summarize long replies, while MCP Gateway keeps the policy decision in one Virtual Server pipeline. Use these controls when the problem is the shape, size, or governance path of the response; use validation, data-protection, or content-safety policies when the content itself must be refused or redacted.

| Plugin / capability | Scenario | Hook(s) | What it proves |
|---|---|---|---|
| SOAP branch with before-tool policies | A BW5 SOAP operation imported from WSDL is governed like other tools | Before Tool (`tool_pre_invoke`) | SQL probes are blocked, rates are enforced, and PII can be masked before the SOAP call |
| `HTMLToMarkdownPlugin` | A large vendor portal page is read as an MCP resource | After Resource (`resource_post_fetch`) | Sanitized HTML can be converted to smaller Markdown that fits more reliably in an agent window |
| Output utilities | Code, JSON text, and Markdown are tidied on the way back | After Tool (`tool_post_invoke`), After Resource (`resource_post_fetch`) | Formatting utilities are useful for text, but language-blind headers can corrupt machine-readable output |
| `Summarizer` | Long tool and resource replies are replaced by model-generated summaries | After Tool (`tool_post_invoke`), After Resource (`resource_post_fetch`) | Summarization depends on real federated tool names and fails open when the model endpoint is unavailable |

## SOAP branch — The SOAP you kept, governed

> The estate keeps its BW5 SOAP services, but the agent-facing tool still needs the same policy coverage as every other tool.

**Example scenario.** A bank exposes a long-lived BusinessWorks 5 `getCityDistance` operation through TIBCO MCP Hub. The WSDL is imported, the endpoint is overridden to the routable service URL, and the Hub derives one Virtual Tool per operation and port. A Virtual Server selects the object-schema operation, while a second Virtual Server keeps the same tool ungoverned for comparison.

Security then applies the usual before-tool controls: SQL injection detection, a per-tool rate ceiling, PII masking, and a telemetry receipt. The SOAP service is unchanged. The governance is added at the Virtual Server, after the WSDL import described in [onboarding capabilities](capabilities-onboarding.md) and in the BW5 SOAP integration walkthrough in [integration](integration.md).

**What it proves.** On MCP Gateway 1.21 and later, before-tool plugins run before a WSDL-derived SOAP invocation. A clean request reaches the SOAP service, a SQL probe is refused before the SOAP call, rapid repeated calls hit the rate limit, and a PII filter can rewrite an argument before the exporter records it.

**When to use it.** Use this pattern when a SOAP service must stay in place but agent access must be governed centrally. Do not treat it as a modernization of the service itself: headers returned by a plugin are not passed through to the SOAP egress, caller network context is limited, and output policies that scan text content do not moderate SOAP tools that return only structured content.

### Where it sits

| Setting | Value |
|---|---|
| Capability | WSDL-derived Virtual Tool on a Virtual Server |
| Main hook | Before Tool (`tool_pre_invoke`) |
| Attached to | Virtual Server that selects the SOAP-derived tool |
| Minimum gateway level | MCP Gateway 1.21 for before-tool coverage on WSDL-derived tools |
| Related pages | [WSDL import](capabilities-onboarding.md), [BW5 SOAP walkthrough](integration.md) |

### Configuration

1. Import the WSDL in MCP Hub and override the endpoint if the address in the WSDL is not routable from the Data Plane.
2. Create a Virtual Server over the object-schema SOAP operation you want agents to call.
3. In Command Center, open **Tool calls → Before Tool**.
4. Add the policy plugins, keep the allow leg first in your testing, then **Push to Gateway** and wait for `synced`.

```json
{
  "SQLSanitizer": {
    "mode": "sequential",
    "priority": 40,
    "config": {
      "fields": null,
      "block_on_violation": true,
      "strip_comments": true
    }
  },
  "PIIFilterPlugin": {
    "mode": "sequential",
    "priority": 50,
    "config": {
      "detect_email": true,
      "default_mask_strategy": "redact",
      "redaction_text": "[PII_REDACTED]"
    }
  },
  "RateLimiterPlugin": {
    "mode": "sequential",
    "priority": 60,
    "config": {
      "by_user": null,
      "by_tenant": null,
      "by_tool": {
        "<your-federated-soap-tool-name>": "1/m"
      },
      "algorithm": "fixed_window",
      "backend": "memory",
      "fail_mode": "closed"
    }
  },
  "ToolsTelemetryExporter": {
    "mode": "transform",
    "priority": 200,
    "config": {
      "export_full_payload": true,
      "max_payload_bytes_size": 100000
    }
  }
}
```

`sequential` is the canonical id for the Hub's Block mode. `fields: null` tells SQLSanitizer to inspect every string argument. The rate-limiter example nulls the inherited user and tenant ceilings so the named tool limit is the deciding rule. The exporter is last so the receipt records the arguments after earlier transforms.

### What happens

A clean call returns the SOAP response. A request with a SQL probe in a string argument returns a policy refusal such as:

```text
Blocked by policy — tool_pre_invoke blocked by plugin SQLSanitizer: SQL_SANITIZER - Risky SQL detected
```

Three rapid calls to a tool configured as `1/m` return one success followed by two `RATE_LIMIT` refusals. An argument containing an email address can be masked to `[PII_REDACTED]` before the SOAP call; the successful response can also carry advisory plugin metadata stating that one value was masked. In Dashboard → Traces, blocked rows are marked Blocked and open to a detail pane that names the responsible plugin and code, and Jaeger-source traces show the before-tool plugin spans plus the exporter receipt on the SOAP call.

### Setup tips

- Confirm the gateway version before relying on before-tool coverage for WSDL-derived tools.
- Pick a SOAP operation with an object input schema for MCP clients; scalar-root operations are not invokable by MCP clients on this path.
- Override the WSDL endpoint to a URL reachable from the Data Plane, for example `http://<service>.<namespace>.svc.cluster.local:<port>/<soap-path>`.
- Consider placing the rate limiter after the sanitizer, so that injection probes the sanitizer blocks may not spend a request from the rate window.
- Treat the policy marker in a refused MCP result as advisory client context, not as an authorization token.

### Considerations

The SOAP branch is governed on what goes into the SOAP call. A plugin that returns headers does not add them to the SOAP egress. Policies that depend on caller IP address or User-Agent do not have that context on this path. SOAP replies are not served from the gateway cache, by design. ContentModeration scans text content; it does not scan `structuredContent`. Tools that return only structured content — including WSDL-derived SOAP tools — are not moderated. Summarizer leaves results that carry `structuredContent` unchanged.

## HTMLToMarkdownPlugin — The page the agent did not need

> The agent asked for the shipment status, not the full HTML page around it.

**Example scenario.** A logistics assistant reads a vendor portal page exposed as an MCP resource. The raw page contains a tracking table, scripts, event handlers, a widget, layout markup, and one hidden instruction carried as text. In this example the gateway also runs `SafeHTMLSanitizer` as a gateway-wide resource policy, which removes active browser content; the baseline resource is still bulky sanitized HTML.

!!! note "Reference measurements"
    Figures on this page come from an internal reference test environment (one gateway, one caller,
    test fixtures). They illustrate relative behaviour; they are not performance commitments. Measure
    in your own environment.

Binding `HTMLToMarkdownPlugin` converts the sanitized HTML resource to Markdown after the resource is fetched. In the reference environment, the baseline sanitized body was 126,273 bytes and 46,571 cl100k tokens. The Markdown body was 88,727 bytes and 29,315 tokens, about 70.3% of the baseline bytes and 62.9% of the baseline tokens.

**What it proves.** The plugin is a deterministic response-shaping control for resource fetches. It reduces markup, changes the MIME type to `text/markdown`, and can help the whole resource fit inside the text window an agent client sends to a model.

**When to use it.** Use it when agents read HTML pages as MCP resources and the value is in the words, not in the page structure. Do not use it as a model-safety control: hidden text remains text, tables are flattened, and a larger page may still exceed the client window.

### Where it sits

| Setting | Value |
|---|---|
| Plugin | `HTMLToMarkdownPlugin` |
| Hook | After Resource (`resource_post_fetch`) |
| Mode | `transform` |
| Priority | 120 in the reference environment |
| Attached to | Virtual Server that exposes HTML resources |
| Typical companion | `SafeHTMLSanitizer` at an earlier resource-post-fetch priority |

### Configuration

1. Federate the MCP server that exposes the HTML resource.
2. Create or open the Virtual Server that selects the resource.
3. In Command Center, open **Resource fetches → After Resource**.
4. Add `HTMLToMarkdownPlugin`. The plugin has no configuration.
5. **Push to Gateway** and wait for `synced`.

```json
{}
```

The important configuration fact is not a JSON key; it is the hook. The plugin declares only `resource_post_fetch`, so it only governs `resources/read`. A tool that fetches the same URL through `tools/call` is outside this control.

### What happens

The baseline path returns sanitized HTML: scripts, event handlers, iframes, objects, and styles are removed by the sanitizer, while headings, tables, spans, and layout divisions can remain. The governed path returns Markdown: headings become `#`, links become Markdown links, code blocks become fences, remaining tags are removed, entities are unescaped, and whitespace is collapsed.

In the reference environment, the baseline body had 125,227 characters and was truncated by a test MCP client that caps resource text at 100,000 characters. The Markdown body had 87,681 characters and arrived whole. The same hidden instruction was present on both paths, which is exactly the boundary of this plugin.

### Setup tips

- First confirm whether your gateway already runs an HTML sanitizer globally. If it does, the honest comparison is sanitized HTML versus Markdown, not raw HTML versus Markdown.
- Test with realistic page sizes. The value depends on the page and on the client window that feeds the model.
- Keep a baseline Virtual Server during rollout so you can compare bytes, tokens, and the body ending.

### Considerations

The converter is intentionally simple. It removes tags with regular rules rather than preserving complex page structure. Tables become one cell per line, hidden text can survive as ordinary text, and conversion does not detect prompt injection. If hidden content must be refused or flagged, bind an appropriate detection policy beside this one.

## Output utilities — What the agent is handed, tidied and stamped

> A text-cleanup policy should improve what the agent reads, not accidentally corrupt data the agent needs to parse.

**Example scenario.** An integration team exposes two plain-text tools and one Markdown resource to an agent. One tool returns a Python configuration snippet with messy whitespace. Another returns a JSON record as plain text, not as structured content. The resource is an untidy Markdown design note.

The governed Virtual Server binds `LicenseHeaderInjector` and `CodeFormatter` on the way back from tool calls, and `MarkdownCleanerPlugin` on the resource path. The Python snippet is correctly headered and normalized. The JSON text receives the same Python-style header in front of `{`, so it no longer parses as JSON. The Markdown note gets the same header, plus useful Markdown cleanup.

**What it proves.** Output utilities are deterministic and useful for text, but they are not semantic classifiers. In the reference environment, nothing supplied a language hint, so `default_language: python` applied broadly.

**When to use it.** Use these plugins for human-readable text that benefits from predictable whitespace, headers, and Markdown normalization. Avoid broad license-header injection on machine-readable text unless your upstream sets language metadata or you isolate the policy to known code-producing tools.

### Where it sits

| Setting | Value |
|---|---|
| Plugins | `LicenseHeaderInjector`, `CodeFormatter`, `MarkdownCleanerPlugin` |
| Hooks | After Tool (`tool_post_invoke`), After Resource (`resource_post_fetch`); `MarkdownCleanerPlugin` also declares `prompt_post_fetch` and has no tool hook |
| Mode | `transform` |
| Priorities | Set explicitly so the header is added before formatting: `LicenseHeaderInjector` 85, `CodeFormatter` 120, `MarkdownCleanerPlugin` 140 (the catalogue defaults, 185 and 180, would run the formatter first) |
| Attached to | Virtual Server exposing text tools and resources |

### Configuration

1. Federate the MCP server and create a Virtual Server over the selected tools and resources.
2. In **Tool calls → After Tool**, add `LicenseHeaderInjector`, then `CodeFormatter`, and set their priorities (85 and 120) so the header is injected first.
3. In **Resource fetches → After Resource**, add `MarkdownCleanerPlugin`.
4. Keep the default transform mode unless you have a specific reason to change it.
5. **Push to Gateway** and wait for `synced`.

```json
{
  "LicenseHeaderInjector": {},
  "CodeFormatter": {},
  "MarkdownCleanerPlugin": {}
}
```

The empty configs use shipped defaults. In the reference environment, that meant a Python-style license header and general text normalization; JSON pretty-printing did not run because the language was not explicitly `json`.

In that same environment, a Virtual Server assignment activated all hooks declared by the plugin, not only the lane where it was added. That is why the resource note also received the license header and formatter output. After pushing, verify the effective hooks in the binding read-back for your gateway.

### What happens

| Leg | Baseline | Governed |
|---|---|---|
| Python snippet | 478 bytes, messy whitespace | 531 bytes, headered, dedented, trailing whitespace trimmed, one trailing newline |
| JSON text record | 177 bytes, valid JSON text | 238 bytes, header before `{`, no longer valid JSON |
| Markdown resource | 382 bytes, untidy Markdown | 433 bytes, headered and normalized: heading spacing, bullet style, line endings, and blank-line runs |

The Hub catalogue correctly does not offer `MarkdownCleanerPlugin` on the After Tool lane because the plugin declares no tool hook. It appears on the resource lane, where it can clean Markdown resource content.

### Setup tips

- Separate code-producing tools from JSON-producing tools if you plan to inject license headers.
- Do not assume JSON pretty-printing will run on a plain string that happens to contain JSON. It needs the language to be known as `json`.
- Verify parseability after transformation, not only visual appearance.
- Read the plugin priorities from your gateway catalogue; do not assume every environment has the same numbers.

### Considerations

These plugins govern what comes back to the agent. They do not inspect or rewrite the tool call the agent sends. They also do not infer whether a reply is code, prose, or data. A broad text policy can be correct for snippets and wrong for machine-readable records in the same Virtual Server.

## Summarizer — The reply, summarized on the way back

> The agent should not receive a long document every time a short answer is enough, but the summary must be applied to the tool that actually ran.

**Example scenario.** A procurement assistant retrieves long partner documents from an MCP server: customs manuals, rate cards, and bulletins. The baseline Virtual Server returns the full text. The governed Virtual Server binds `Summarizer` so long replies can be replaced by a model-generated summary.

The important detail is the allowlist. For federated MCP tools, the name a client calls is the federated name, such as `partnerdocs-retrieve-customs-manual`, not the upstream server's original function name. Whether the shipped default allowlist matches your federated names depends on your gateway release, so set `tool_allowlist` explicitly and verify it on your release.

**What it proves.** The tool leg depends on exact federated tool names. The resource leg is different: it uses URI prefixes and length thresholds, so an `https://` resource can be summarized without a tool allowlist entry. Short replies, results that carry `structuredContent`, and non-matching resource URI schemes stay unchanged.

**When to use it.** Use it for long, human-readable text where a summary is acceptable and losing the original reply is acceptable. Do not use it when the exact original text must remain available in the MCP response, or when an unreachable model endpoint should block the call.

### Where it sits

| Setting | Value |
|---|---|
| Plugin | `Summarizer` |
| Hooks | After Tool (`tool_post_invoke`), After Resource (`resource_post_fetch`) |
| Mode | `transform` |
| Priority | 170 in the reference environment |
| Attached to | Virtual Server exposing long text tools or resources |
| External dependency | A model endpoint configured through gateway environment settings you own |

### Configuration

1. Call `tools/list` on the Virtual Server and copy the real federated tool names.
2. In Command Center, open **Tool calls → After Tool** and add `Summarizer`.
3. Set `tool_allowlist` to the federated names that are allowed to summarize.
4. **Push to Gateway** and wait for `synced`.

```json
{
  "tool_allowlist": [
    "partnerdocs-retrieve-customs-manual",
    "partnerdocs-search-bulletin",
    "partnerdocs-retrieve-rate-card-structured"
  ]
}
```

Replace those names with your own federated tool names. The model credentials belong in gateway environment configuration, such as a Secret referenced by your gateway deployment values, not in the Hub binding JSON.

### What happens

In the reference environment, the long tool reply changed from a full document of about 973 tokens to a summary of about 240 tokens. Latency increased from about 296 ms to about 2131 ms because the policy calls a model. A short bulletin stayed unchanged because it was under the threshold, and a structured reply stayed unchanged because Summarizer leaves results that carry `structuredContent` unchanged.

For resources, an `https://` document was summarized from 5095 bytes to 1028 bytes without any `tool_allowlist` entry, because the resource path does not check `tool_allowlist`. A `docs://` resource stayed unchanged because its URI scheme did not match the prefix gate. When the configured model endpoint was unreachable, the full document passed through with a `summarize_error` metadata field.

### Setup tips

- Copy tool names from `tools/list` or the Hub tool list after federation; do not type the upstream function name by memory.
- Keep start and end markers in test documents so you can prove whether the original text survived.
- Test short, structured, and resource cases separately. They follow different gates.
- Monitor latency as well as token count. Summarization can reduce size while increasing response time.

### Considerations

The summary replaces the original response; it is not added beside it. The plugin does not reject an allowlist entry that matches no tool; a misspelled name simply leaves that tool unsummarized, so verify each entry with a test call. If the model endpoint is unavailable, the plugin fails open and returns the original content with metadata rather than refusing the call. Treat summaries as convenience output, not as a fidelity guarantee.
