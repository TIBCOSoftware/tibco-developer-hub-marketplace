# Policies — Content safety & threat protection

TIBCO MCP Hub can apply content-safety and threat-protection controls at several points in the MCP Gateway pipeline: before a tool runs, after a tool returns, before a prompt is fetched, and before or after a resource is read. The controls on this page focus on material an agent reads, writes, relays, or receives as code, SQL, URLs, or citations. For complementary policy families, see [access policies](policies-access.md), [data-protection policies](policies-data-protection.md), and the [plugin pipeline](plugin-pipeline.md) guide for hook and priority design.

| Plugin / capability | Scenario | Hook(s) | What it proves |
|---|---|---|---|
| SQLSanitizer | The model composes destructive SQL and sends it to a query tool. | Before Tool (`tool_pre_invoke`) and Prompt pre-fetch (`prompt_pre_fetch`) | Destructive SQL can be refused before the query tool receives it. |
| CodeSafetyLinterPlugin | A coding tool returns dangerous code to an agent. | After Tool (`tool_post_invoke`) | Tool output can be blocked or annotated before the agent reasons over it. |
| ContentModeration | The assistant sends off-brand content, or relays content returned by a tool. | Before Tool (`tool_pre_invoke`), After Tool (`tool_post_invoke`), Prompt pre-fetch (`prompt_pre_fetch`) | A moderation provider can refuse, redact, transform, or warn on text passing through the gateway. |
| HarmfulContentDetector | A known phrase appears in a prompt argument or tool reply. | Before Prompt (`prompt_pre_fetch`) and After Tool (`tool_post_invoke`) | A local regex lexicon catches known strings without an external classifier. |
| URLReputationPlugin | A resource read points at a lookalike host or plain HTTP URL. | Before Resource (`resource_pre_fetch`) | Resource reads can be denied by scheme, host, or URL pattern before fetch. |
| VirusTotalURLCheckerPlugin | A resource URL needs an external reputation verdict. | Before Resource (`resource_pre_fetch`); also declares `resource_post_fetch`, `prompt_post_fetch` and `tool_post_invoke` | A feed-based check can fail closed without a key and block URLs reported malicious. |
| RobotsLicenseGuard and CitationValidator | A research agent reads pages with declared AI restrictions and cites links in its answer. | RobotsLicenseGuard: Before Resource (`resource_pre_fetch`) and After Resource (`resource_post_fetch`); CitationValidator: After Resource (`resource_post_fetch`) and After Tool (`tool_post_invoke`) | Declared no-AI policy and dead citations can both stop the call, if ordered correctly. |

## SQLSanitizer — the model wrote the DROP

> A model that can compose raw SQL can also compose a destructive statement.

**Example scenario.** A clinical-operations assistant answers reporting questions against a warehouse through a federated query tool. The tool accepts a raw SQL string, so the assistant is not only retrieving data; it is writing the statement that the query tool will execute. An analyst asks for cleanup of an obsolete staging table. The model chooses `DROP TABLE archived_encounters;` and sends it as the `query` argument.

On an ungoverned Virtual Server, the query tool receives the statement exactly as written. On the governed Virtual Server, SQLSanitizer reads the tool arguments on the way in and refuses the call before the query tool receives the SQL.

**What it proves.** SQLSanitizer is an input guard. It scans SQL strings in prompt arguments and tool arguments, returns `SQL_SANITIZER` for destructive shapes, and stops the tool invocation in **Block** (`sequential`) mode.

**When to use it.** Use it when an agent composes raw SQL against a system of record and you need a deterministic gateway-side refusal for dangerous statements. If the tool accepts structured parameters instead of SQL, or if every destructive change is already human-approved outside the gateway, this control may not be the right fit. If the risk is who may run a tool at all, combine this with [access policies](policies-access.md).

### Where it sits

| Setting | Value |
|---|---|
| Plugin | `SQLSanitizer` |
| Hook | Before Tool (`tool_pre_invoke`) for tool arguments; Prompt pre-fetch (`prompt_pre_fetch`) for prompt arguments |
| Mode | Block (`sequential`) |
| Attached to | Virtual Server |
| Scope | The SQL text in selected argument fields, before the upstream tool runs |

### Configuration

1. Open the governed Virtual Server in TIBCO MCP Hub.
2. Open **Command Center** and select the **Tool calls** flow.
3. On the **Before Tool** (`tool_pre_invoke`) lane, add `SQLSanitizer`.
4. Set the mode to **Block** (`sequential`).
5. Paste the JSON configuration, apply it, then **Push to Gateway** and wait for `synced`.
6. Test with an allowed `SELECT`, a destructive `DROP`, and an unqualified `DELETE` or `UPDATE`.

```json
{
  "fields": ["sql", "query", "statement"],
  "block_on_violation": true,
  "strip_comments": true
}
```

Important keys:

| Key | Effect |
|---|---|
| `fields` | The argument keys to scan. If omitted or `null`, the plugin scans every string leaf, which improves coverage but can create false positives. |
| `block_on_violation` | `true` refuses the call. `false` records issues as advisory metadata and lets the call continue. |
| `strip_comments` | Removes SQL comments from benign forwarded values. The block decision is still based on the detection path, not on the stripped value. |
| `blocked_statements` | Optional list of DDL or privilege statements to block. Defaults include `DROP`, `TRUNCATE`, `ALTER`, `GRANT`, and `REVOKE`. |
| `block_delete_without_where` / `block_update_without_where` | Refuse unqualified `DELETE` and `UPDATE` statements. |
| `require_parameterization` | Optional heuristic signal for string interpolation. Treat it as a hint, not proof of parameter safety. |

### What happens

| Tool call | Governed outcome |
|---|---|
| `query: "SELECT id FROM encounters WHERE id = 5"` | Passes unchanged. |
| `query: "DROP TABLE archived_encounters;"` | Blocked before the tool runs. |
| `query: "DELETE FROM roster"` | Blocked as `DELETE` without `WHERE`. |
| `query: "DELETE FROM roster WHERE id = 1"` | Passes because a `WHERE` token is present. |

Only the configured `fields` are inspected.

The client receives a refusal in this shape:

```text
tool_pre_invoke blocked by plugin SQLSanitizer: SQL_SANITIZER - Risky SQL detected (Potentially dangerous SQL detected in tool args)
```

Dashboard → Traces shows the governed call as **Blocked** with `SQLSanitizer · SQL_SANITIZER`. The ungoverned or non-blocking path completes, which is useful when proving that the plugin, not the tool itself, produced the refusal.

### Setup tips

- Match `fields` to the tool contract. A narrow list avoids false positives, but any SQL sent under a different key is not scanned.
- Prove an allowed leg first. A clean `SELECT` should pass, so a later `DROP` block is not a blanket outage.
- Keep `block_on_violation` on for enforcement. Turning it off changes the plugin into an observer.
- Use a Before Tool lane for query tools. SQLSanitizer does not inspect tool results.
- Confirm the tool is only a query executor. SQLSanitizer is not a database proxy; it reads MCP arguments at the gateway.

### Considerations

SQLSanitizer catches accidental and obvious destructive SQL, not every malicious expression. It provides configured lexical checks, not database authorization or a guarantee of query safety; combine it with least-privilege database permissions and approved query interfaces. The blocklist is curated: `INSERT` is not blocked by default, scoped updates pass, and any pattern outside the configured set may pass. The detection path is stronger than a simple text match because it normalizes strings and comments before checking, but it remains a policy guard over SQL text, not a replacement for database privileges, review workflows, or least-privilege credentials.

## CodeSafetyLinterPlugin — what the agent was handed

> A coding agent may be handed dangerous code as tool output, even when it never asked to run it.

**Example scenario.** An engineering assistant searches repositories, returns snippets, and suggests shell or Python code for developers to inspect. One tool returns `os.system("id")` or `subprocess.run(...)`; another returns prose containing `eval(`, `exec(`, or `rm -rf`. The gateway must distinguish the patterns it is prepared to hard-block from patterns that are better treated as warnings because they occur often in legitimate documentation.

**What it proves.** CodeSafetyLinterPlugin inspects the code returned by tools on `tool_post_invoke`. A block-tier match refuses the tool result with `CODE_SAFETY`. A warn-tier match lets the result continue with advisory metadata.

**When to use it.** Use it when tools return code, scripts, snippets, command examples, or generated remediation steps. It is not a command sandbox and does not stop a command from executing. If the risk is whether an agent may call an execution tool, use [access policies](policies-access.md) and runtime sandboxing. If the risk is PII or secrets in the returned text, use [data-protection policies](policies-data-protection.md).

### Where it sits

| Setting | Value |
|---|---|
| Plugin | `CodeSafetyLinterPlugin` |
| Hook | After Tool (`tool_post_invoke`) |
| Mode | Block (`sequential`) |
| Attached to | Enforced gateway-wide by default in the reference gateway; the Virtual Server binding is where you view and tune it, not an on/off switch, and its blocking mode cannot be downgraded |
| Scope | Tool output returned to the MCP client |

### Configuration

1. Open the governed Virtual Server.
2. Open **Command Center** and select the **Tool calls** flow.
3. On the **After Tool** (`tool_post_invoke`) lane, add `CodeSafetyLinterPlugin`.
4. Use an empty config to view the shipped defaults, or configure explicit block and warning regexes for your environment.
5. Apply, **Push to Gateway**, wait for `synced`, then test one clean output, one blocked output, and one warn-only output.

```json
{}
```

Example patterns from the shipped defaults (the block tier on your release may list more execution sinks than shown here):

| Tier | Patterns | Effect |
|---|---|---|
| Block | `\bos\.system\s*\(`, `\bsubprocess\.(Popen\|call\|run)\s*\(` | Refuses the result with `CODE_SAFETY`. |
| Warn | `\beval\s*\(`, `\bexec\s*\(`, `\brm\s+-rf\b` | Lets the result pass and records an advisory. |

### What happens

A benign snippet passes. A tool result containing literal `os.system(` or `subprocess.run(` is blocked before the agent can use it. The client sees the category, not the exact matched pattern:

```text
tool_post_invoke blocked by plugin CodeSafetyLinterPlugin: CODE_SAFETY - Unsafe code pattern (Detected unsafe code constructs)
```

A result containing warn-tier strings such as `eval(`, `exec(`, or `rm -rf` passes with advisory metadata. In a test agent client, the blocked result appears as a policy block; advisory metadata may not be rendered visibly, so use traces or raw response metadata for audit.

### Setup tips

- Test all three outcomes: clean pass, block-tier refusal, and warn-tier advisory. That proves the plugin is active and discriminating.
- Do not claim the client is told which block pattern matched unless your client actually displays it. The standard block text exposes the category.
- Tune warn patterns carefully. Shell examples and stack traces commonly contain strings that should alert a human but not necessarily stop work.
- The plugin is enforced gateway-wide, so there is no plugin-free control Virtual Server on the same gateway. A false positive on the block tier is fixed by moving the rule to the warn tier, not by changing the mode. Confirm your release's catalogue and binding read-back.

### Considerations

The plugin inspects output as data; it does not execute or sandbox anything. The block tier matches literal execution-sink calls such as `os.system(`. The warn tier recognizes configured patterns for some dynamic imports and indirect calls and is advisory; review warn-tier hits and validate the patterns against your own code. This is a lexical scanner, not execution analysis. The warn tier is deliberately broader and more false-positive-prone; treat it as a signal for review, not as proof that code is unsafe.

## ContentModeration — what the assistant sends and what tools return

> The assistant can create harmful content itself, or relay harmful content written by someone else.

**Example scenario.** A retail assistant drafts replies to customer reviews and calls a posting tool to send them. In a separate flow, the same assistant reads review or support-ticket text returned by a system of record and relays it to another customer. The first risk is content the assistant sends into a tool; the second risk is content a tool returns to the assistant.

**What it proves.** ContentModeration can evaluate text with a configured moderation provider and then block, redact, transform, or warn. The provider can be IBM Watson NLU, IBM Granite via Ollama, or OpenAI. The reference scenarios used a stand-in classifier that flags a fixed test marker, so they demonstrate the block and redact mechanics, not classification quality. A built-in lexicon can provide a weaker fallback when a network provider fails, but production-grade moderation depends on the provider and credentials you configure.

**When to use it.** Use ContentModeration when assistant text crosses a brand, safety, or compliance boundary. Use the Before Tool case when the assistant is sending content outward. Use the After Tool case when the assistant is relaying content it did not write. Use [data-protection policies](policies-data-protection.md) instead when the issue is PII, secrets, or masking, and see [plugin ordering](plugin-pipeline.md) if moderation needs to run before or after another text transform.

### Where it sits

| Setting | Value |
|---|---|
| Plugin | `ContentModeration` |
| Hooks | Prompt pre-fetch (`prompt_pre_fetch`), Before Tool (`tool_pre_invoke`), After Tool (`tool_post_invoke`) |
| Mode | Block (`sequential`) |
| Attached to | Virtual Server |
| Common actions | `block`, `redact`, `transform`, `warn` |

### Configuration

1. Decide which surface you are protecting: prompt text, tool arguments, tool output, or more than one.
2. Open the Virtual Server in **Command Center**.
3. Add `ContentModeration` on the lane that matches the scenario, for example **Before Tool** for outgoing post text or **After Tool** for returned review text.
4. Select the provider and endpoint. Store credentials outside the JSON, for example in a secret or environment variable supported by the gateway deployment.
5. Configure categories, thresholds, and actions.
6. Apply, **Push to Gateway**, wait for `synced`, then test both flagged and benign content.

```json
{
  "provider": "ibm_granite",
  "ibm_granite": {
    "ollama_url": "http://<ollama-service>.<namespace>.svc.cluster.local:11434",
    "timeout": 30
  },
  "enable_caching": false,
  "categories": {
    "harassment": {
      "threshold": 0.5,
      "action": "block"
    }
  }
}
```

Important keys:

| Key | Effect |
|---|---|
| `provider` | Selects the moderation provider: `ibm_watson`, `ibm_granite` or `openai`. The Hub renders it as a dropdown. |
| Provider block | Endpoint and provider-specific settings. Do not paste API keys into shared documentation or source-controlled configs. |
| `categories.<name>.threshold` | Score above which the category is flagged. |
| `categories.<name>.action` | Action to take for the category: `block`, `redact`, `transform`, or `warn`. |
| `enable_caching` | Controls verdict caching by text. Disable during tests if you need every call to prove the provider path. |

### Scenario: The assistant sends content outward

#### What happens

The assistant calls a posting tool such as `retail-posts-publish-reply` and passes the reply text as a tool argument. On the baseline Virtual Server, the tool receives and publishes the text. On the governed Virtual Server, ContentModeration blocks the call before the posting tool runs.

```text
tool_pre_invoke blocked by plugin ContentModeration: CONTENT_MODERATION - Content policy violation in tool arguments (Harmful content detected in retail-posts-publish-reply arguments)
```

A benign reply still passes. Dashboard → Traces shows the governed call as **Blocked** with `ContentModeration · CONTENT_MODERATION`, while the baseline call completes.

#### Setup tips

- Attach to **Before Tool** when the content to protect is in tool arguments.
- Prove that the same prompt and same tool succeed on a baseline Virtual Server and fail only on the governed one.
- If you use a deterministic test token during setup, label it as a test signal. Do not present it as a real classifier judgment.
- Configure the category action intentionally. A category set to `warn` will not block even if the provider flags it.

#### Considerations

A block refuses the whole tool call, not just the offending sentence. If you need partial rewriting, test `redact` or `transform` and verify the returned shape. Also note the hook behaviour: one assignment activates the plugin on every hook it declares; the lane you choose is where the Hub shows it. A ContentModeration binding therefore moderates prompt arguments, tool arguments and tool output together.

### Scenario: The assistant relays returned content

#### What happens

The assistant calls a review tool such as `retail-reviews-list-reviews`. The returned content includes a flagged review. With `action: "block"` on the **After Tool** lane, the gateway refuses the tool result before the assistant receives it:

```text
tool_post_invoke blocked by plugin ContentModeration: CONTENT_MODERATION - Content policy violation in tool output (Harmful content detected in retail-reviews-list-reviews output)
```

With `action: "redact"`, the call can pass but with replacement text, for example:

```json
{
  "content": [
    { "type": "text", "text": "[CONTENT REMOVED BY MODERATION]" },
    { "type": "text", "text": "[CONTENT REMOVED BY MODERATION]" }
  ],
  "isError": false,
  "_meta": {
    "policy": {
      "content_modified": true,
      "structured_content_dropped": true
    }
  }
}
```

In the reference test environment, redaction was applied to the joined reply, not only the one flagged item, and structured content was dropped from the returned object.

#### Setup tips

- Attach to **After Tool** for returned text, and verify your gateway version actually scans standard MCP `content` results.
- Keep a clean returned item in the test set. A clean pass proves moderation is not refusing everything.
- Test `block` and `redact` separately. They have different operational consequences.
- Review traces and gateway logs, not only the assistant's final prose. An allowed result carries no moderation verdict in the client response; the allow decision is visible only in the plugin's audit log.

#### Considerations

ContentModeration scans text content; it does not scan `structuredContent`. Tools that return only structured content — including WSDL-derived SOAP tools — are not moderated. There is also a maximum text length for joined output, so content past the scan ceiling should not be treated as fully moderated. The classifier is the control: without a configured provider, any local fallback is only a basic lexicon.

## HarmfulContentDetector — a lexicon, not a classifier

> Use a lexicon when the phrase is known; use a classifier when the danger is meaning.

**Example scenario.** A patient-portal assistant reads clinical notes and drafts messages from prompt arguments. The organization has a small set of exact phrases or tokens it never wants the assistant to receive or return. Those phrases are known in advance and can be represented as regular expressions.

**What it proves.** HarmfulContentDetector scans configured regex categories on `prompt_pre_fetch` and `tool_post_invoke`. It can block with `HARMFUL_CONTENT`, or, if redaction is enabled, replace matches and continue. Redaction takes precedence over blocking for the same binding.

**When to use it.** Use it for exact, known phrases and local offline enforcement with no external provider. Do not use it as a semantic safety classifier. For provider-based meaning detection, use ContentModeration. For masking PII or secrets, use [data-protection policies](policies-data-protection.md).

### Where it sits

| Setting | Value |
|---|---|
| Plugin | `HarmfulContentDetector` |
| Hooks | Before Prompt (`prompt_pre_fetch`) and After Tool (`tool_post_invoke`) |
| Mode | Block (`sequential`) |
| Priority | `96` (catalogue default) |
| Attached to | Virtual Server; one assignment covers both hooks |

### Configuration

1. Open the governed Virtual Server.
2. Select the **Prompt fetches** flow and add `HarmfulContentDetector` on **Before Prompt**. One assignment activates the plugin on every hook it declares; the lane you choose is where the Hub shows it. It therefore also scans returned tool text on **After Tool**.
3. Paste the category config once. Attaching the same plugin a second time on the same Virtual Server merges into one policy, so the two hooks cannot carry different configs.
4. Push and verify the binding after sync.
5. Test a prompt argument and a tool reply.
6. Later, if you change `redact` from `false` to `true`, push again and retest because behavior changes from refusal to pass-with-mask.

```json
{
  "categories": {
    "test_signal": ["\\bTEST-SIGNAL-HARMFUL-\\d+\\b"]
  },
  "block_on": ["test_signal"],
  "redact": false
}
```

Use harmless test signals while validating the pipeline. Do not put real harmful phrases in screenshots, examples, or shared runbooks unless your governance process requires it.

### What happens

| Call | Governed outcome |
|---|---|
| Prompt argument with no match | Passes. |
| Tool reply with no match | Passes. |
| Prompt argument containing the configured pattern | Blocked before prompt fetch. |
| Tool reply containing the configured pattern | Blocked after tool return. |
| Same config with `redact: true` | Passes with matched text replaced, commonly as `[REDACTED]`. |

A block has this client-facing shape:

```text
HARMFUL_CONTENT - Harmful content (Detected categories: test_signal)
```

The baseline Virtual Server returns the same matched text raw, which proves the refusal is the plugin's effect.

### Setup tips

- Expect both prompt arguments and returned tool content to be scanned: one assignment activates the plugin on both of its hooks.
- Verify priority relative to other text-changing plugins. If a mask or transform runs first, the detector may see different text. The [plugin pipeline](plugin-pipeline.md) page explains ordering.
- Treat `redact` as a behavioral switch. It is not just presentation; it stops blocking for matched text and continues with masked content.
- If you supply any `categories` object, confirm whether your shipped defaults are replaced or merged in your release. In the reference behavior, custom categories replaced defaults.

### Considerations

HarmfulContentDetector does not understand meaning. It is case-insensitive regex matching over strings. A paraphrase, coded wording, or unseen phrase is not caught unless the pattern matches it. Conversely, a broad regex can catch benign text. Redaction is silent from the operator's point of view unless you monitor traces and returned content, so review it carefully before changing a block policy into a redact policy.

## URLReputationPlugin — where the agent may read from

> A resource read should not trust a one-character-off host just because the URL looks familiar.

**Example scenario.** A procurement assistant reads partner rate cards and compliance documents exposed as MCP resources. The legitimate portal is `https://partner-portal.example.com/ratecard.pdf`. A lookalike URL, `https://partner-porta1.example.com/ratecard.pdf`, differs by one character. Another resource uses the real host but downgrades to `http://`.

**What it proves.** URLReputationPlugin runs before resource fetch and inspects the requested URI's scheme and host. It can refuse a configured bad domain, refuse plain HTTP, and fail closed when a URI has no host to inspect.

**When to use it.** Use it when agents read resources by URL and you need local controls for known-bad hosts, URL patterns, or non-secure schemes. It is not a threat feed and does not judge the text inside a trusted source. For feed-based URL reputation, use VirusTotalURLCheckerPlugin. For resource-stage ordering, see the [plugin pipeline](plugin-pipeline.md).

### Where it sits

| Setting | Value |
|---|---|
| Plugin | `URLReputationPlugin` |
| Hook | Before Resource (`resource_pre_fetch`) |
| Mode | Block (`sequential`) |
| Priority | `60` in the reference test environment |
| Attached to | Virtual Server |
| Scope | `resources/read`, before the upstream resource is fetched |

### Configuration

1. Open the Virtual Server that exposes the resource set.
2. Open **Command Center** and select the **Resource fetches** flow.
3. On **Before Resource** (`resource_pre_fetch`), add `URLReputationPlugin`.
4. Configure denied domains or patterns and decide whether to refuse plain HTTP.
5. Leave `whitelist_domains` empty unless you intentionally want a bypass for that exact host.
6. Apply, **Push to Gateway**, wait for `synced`, then test allowed, denied, insecure, and no-host URIs.

```json
{
  "whitelist_domains": [],
  "allowed_patterns": [],
  "blocked_domains": ["partner-porta1.example.com"],
  "blocked_patterns": [],
  "use_heuristic_check": false,
  "entropy_threshold": 3.65,
  "block_non_secure_http": true
}
```

Important keys:

| Key | Effect |
|---|---|
| `blocked_domains` | Exact domains to refuse. This is the main denylist mechanism. |
| `blocked_patterns` | URL patterns to refuse. |
| `block_non_secure_http` | Refuses `http://` resource reads. |
| `whitelist_domains` | A bypass list, not an allow-list gate. A domain absent from this list is not automatically blocked. A domain present here can bypass other checks, including plain HTTP blocking. |
| `use_heuristic_check` | Optional heuristic checks. Test with your real domain set before using it in production. |

### What happens

| Resource URI | Governed outcome |
|---|---|
| `https://partner-portal.example.com/ratecard.pdf` | Allowed by default when no rule blocks it. |
| `https://partner-porta1.example.com/ratecard.pdf` | Blocked by `blocked_domains`. |
| `http://partner-portal.example.com/ratecard.pdf` | Blocked by `block_non_secure_http`. |
| `file:///compliance/certs/example-iso9001.pdf` | Blocked because no host can be parsed. |

The lookalike refusal names the rule:

```text
Plugin Violation: Domain 'partner-porta1.example.com' in blocked set
```

The plain HTTP and no-host cases can be more generic:

```text
Plugin Violation: URL http://partner-portal.example.com/ratecard.pdf is blocked
```

Dashboard → Traces records the resource read as blocked before fetch. A baseline Virtual Server over the same resources can read the lookalike and plain HTTP resources, which proves the denylist is doing the work.

### Setup tips

- Test the allowed resource first. A valid partner document should still read through the governed server.
- Do not populate `whitelist_domains` just to document the good host. Use it only when you intentionally want a host to bypass other checks.
- Remember that this plugin does not govern a tool that fetches arbitrary URLs inside its own implementation. It governs MCP `resources/read`.
- When using heuristic checks, rehearse with domains that resemble your production domains, not placeholder-only hostnames.

### Considerations

URLReputationPlugin is a local list, not an external reputation service. A lookalike domain you did not list is allowed unless another configured rule catches it. The refusal text does not always tell the agent which rule fired, so operators should record configuration changes and review traces. If you need external intelligence, combine this local denylist pattern with VirusTotalURLCheckerPlugin on the same resource pipeline, paying close attention to order and priority.

## VirusTotalURLCheckerPlugin — a feed, not a list

> A local list only knows what you wrote; a reputation feed can know what others have already reported.

**Example scenario.** An operations assistant reads vendor security bulletins before a maintenance window. Some resources use partner-owned URLs; one URL points to the public EICAR anti-malware test artefact at `https://secure.eicar.org/eicar.com.txt`; another points to a page the feed has never seen. The organization wants a gateway-side verdict before the assistant reads the content.

**What it proves.** VirusTotalURLCheckerPlugin runs on `resource_pre_fetch`. Without its API key, the gateway fails closed and refuses covered reads rather than silently allowing them. With a key, the plugin can allow a known-clean URL, block a URL reported malicious, cache verdicts, and pass an unscanned URL as unknown rather than safe.

**When to use it.** Use it when a local denylist is not enough and resource reads should be checked against VirusTotal's URL or domain intelligence. Use URLReputationPlugin for local host rules that do not need a network call. Account for external quota, caching, and outage behavior before you enforce it broadly.

### Where it sits

| Setting | Value |
|---|---|
| Plugin | `VirusTotalURLCheckerPlugin` |
| Hook | Before Resource (`resource_pre_fetch`); also declares `resource_post_fetch`, `prompt_post_fetch` and `tool_post_invoke` |
| Mode | Block (`sequential`) |
| Priority | `61` in the reference test environment |
| Attached to | Virtual Server |
| Scope | URL-shaped resource reads before fetch. The plugin also declares `resource_post_fetch`, `prompt_post_fetch` and `tool_post_invoke`, and one assignment activates every hook it declares, so URLs in fetched content, rendered prompts and tool output can also be looked up at VirusTotal |

### Configuration

1. Provision the API key as a gateway environment variable or secret. Do not put the key in the plugin JSON.
2. Open the governed Virtual Server and select the **Resource fetches** flow.
3. Add `VirusTotalURLCheckerPlugin` on **Before Resource**.
4. Set `api_key_env` to the environment variable name.
5. Decide whether to use local allow or deny lists, and whether to check URL, domain, or IP verdicts.
6. Push, wait for `synced`, and test no-key behavior, allowed local exceptions, known-clean URLs, known-bad test URLs, and unknown URLs.

```json
{
  "api_key_env": "VT_API_KEY",
  "allow_domains": [],
  "deny_domains": [],
  "block_on_verdicts": ["malicious"],
  "min_malicious": 1,
  "cache_ttl_seconds": 300,
  "scan_if_unknown": false,
  "check_domain": false,
  "check_ip": false
}
```

Important keys:

| Key | Effect |
|---|---|
| `api_key_env` | Name of the environment variable that holds the VirusTotal key. It is not the key. |
| `allow_domains` | Local bypass list. A matching domain can read without contacting the feed. |
| `deny_domains` | Local deny list. |
| `block_on_verdicts` and `min_malicious` | Verdict thresholds that turn feed stats into a block. |
| `cache_ttl_seconds` | Plugin verdict cache duration. |
| `scan_if_unknown` | Whether to submit unknown URLs for scanning. The reference scenario leaves this off. |
| `check_domain` / `check_ip` | Additional lookups. Disable if URL verdicts are enough and you need to preserve quota. |

### What happens

!!! note "Reference measurements"
    Figures on this page come from an internal reference test environment (one gateway, one caller,
    test fixtures). They illustrate relative behaviour; they are not performance commitments. Measure
    in your own environment.

No key present, no local allow:

```text
Plugin Violation: cold cache under missing-credentials
```

No key present, but the vendor host is in `allow_domains`, for example `advisory.example-vendor.com`: the read passes as a local allow and VirusTotal is not contacted. Other hosts remain refused while the key is missing.

Key present, default allow list empty:

| Resource | Governed outcome |
|---|---|
| A known-clean public page such as `https://www.iec.ch/` | Reads with a clean verdict in the reference test environment. |
| `https://secure.eicar.org/eicar.com.txt` | Blocked with `VT_URL_BLOCK`. |
| `https://advisory.example-partner.com/bulletin-2026-09.html` when the feed has no record | Reads as unscanned or no-input, which is not the same as safe. |
| Repeated blocked URL within cache TTL | Blocks from cache, usually faster and without another feed lookup. |

For the EICAR URL, the reference test environment observed `malicious: 9`, `suspicious: 2`, `undetected: 28`, and `harmless: 51` at the time of the run. Treat exact counts as feed data that can change.

### Setup tips

- Validate the missing-key path before rollout. A misconfigured key should fail closed, not pass silently.
- Keep `api_key_env` as a variable name and manage the actual secret through your platform's secret mechanism.
- Start with a small resource set. External feeds have quota and latency characteristics that differ by plan and time.
- If you add an `allow_domains` entry during an incident, document it and remove it deliberately.
- Read traces and raw error details for feed stats; some clients shorten the visible error text.

### Considerations

An unknown feed result is not proof of safety; it means the feed has no verdict available under the configured behavior. Verdicts are cached; after changing a list, verify the effective policy once the cache TTL has passed. Also account for feed quota: a measured burst of seven scored lookups in under four seconds did not trip the tested key's limit, but that is not a guarantee for your plan.

## Provenance — license and citation

> A research agent should respect declared source policy and should not ship dead citations.

**Example scenario.** A research assistant reads web resources and writes a sourced answer. One page declares that it should not be used for AI processing through machine-readable metadata. Another page is clean. The assistant then writes an answer containing citations, one resolving and one dead.

**What it proves.** Provenance controls can run at different stages of the same request. `RobotsLicenseGuard` evaluates the resource after fetch but before the agent receives it. `CitationValidator` evaluates the tool result after the answer is produced. The order matters because a sanitizer or transform that removes HTML metadata before the license guard runs can prevent the guard from seeing the policy it needs.

**When to use it.** Use these controls when agents read external pages or produce answers with citations. They address declared source policy and link resolvability. They do not decide whether your organization has a contract right to use a source, and they do not verify that a cited page supports the claim.

### Where it sits

| Setting | Value |
|---|---|
| Plugin | `RobotsLicenseGuard` |
| Hooks | After Resource (`resource_post_fetch`); also declares Before Resource (`resource_pre_fetch`) |
| Mode | Block (`sequential`) |
| Priority | `63` (catalogue default); keep it lower than any HTML sanitizer that removes the metadata it needs, because lower numeric priority runs earlier |
| Attached to | Virtual Server resource pipeline |

| Setting | Value |
|---|---|
| Plugin | `CitationValidator` |
| Hooks | After Tool (`tool_post_invoke`); also declares After Resource (`resource_post_fetch`) |
| Mode | Block (`sequential`) |
| Attached to | Virtual Server tool pipeline |

### Configuration

1. Open the Virtual Server that exposes the external resources and answer tool.
2. Select the **Resource fetches** flow and attach `RobotsLicenseGuard` on **After Resource**.
3. Give it a priority that lets it inspect HTML metadata before other resource transforms remove it.
4. Select the **Tool calls** flow and attach `CitationValidator` on **After Tool**. It also declares `resource_post_fetch`, so the same assignment checks links in fetched resources.
5. Apply both configurations, **Push to Gateway**, and wait for `synced`.
6. Test four legs: restricted page, clean page, answer with a dead citation, and answer with resolving citations.

`RobotsLicenseGuard`:

```json
{
  "respect_noai_meta": true,
  "block_on_violation": true,
  "allow_overrides": []
}
```

`allow_overrides` contains URI substrings that bypass the check for sources you have approved separately. Matching is a plain substring match on the resource URI, so a short entry bypasses broadly; keep this list small, specific and auditable.

`CitationValidator`:

```json
{
  "require_200": true,
  "block_on_any_fail": true,
  "max_links": 20
}
```

`require_200` performs real outbound checks. Confirm gateway egress and timeout behavior before using it on high-volume answer flows.

### What happens

| Case | Governed outcome |
|---|---|
| Resource declares no-AI policy | Blocked by `RobotsLicenseGuard`. |
| Clean resource | Reads successfully. |
| Tool result cites a dead link | Blocked by `CitationValidator` with `CITATION_INVALID`. |
| Tool result cites resolving links | Delivered. |

Representative client text for the resource block:

```text
MCP error -32602: Plugin Violation: robots/noai policy
```

Representative client text for the citation block:

```text
tool_post_invoke blocked by plugin CitationValidator: CITATION_INVALID - Invalid citations (One or more citations failed validation)
```

Dashboard → Traces records the blocked calls with the plugin name and code. A baseline Virtual Server is useful for proving that the upstream page or answer tool itself still works.

### Setup tips

- Switch the Command Center to **Resource fetches** before adding `RobotsLicenseGuard`; resource lanes are separate from tool lanes.
- Confirm ordering against HTML transforms. If another plugin strips the `<head>` first, the license guard may never see the no-AI metadata.
- Keep one clean page in the test. It proves the resource guard is not blocking every read.
- Confirm outbound network access for citation checks. A gateway that cannot reach the public URL cannot produce a trustworthy citation verdict.
- Use `allow_overrides` only for sources your organization has reviewed.

### Considerations

RobotsLicenseGuard reads declared policy; it does not know your legal agreement with the publisher. CitationValidator checks that links resolve; it does not prove that the cited content supports the generated sentence. Both controls are strongest when paired with clear pipeline ordering, source allowlists, and trace review. See [plugin-pipeline.md](plugin-pipeline.md) for ordering patterns across resource and tool hooks.
