# Policies — Data protection & privacy

Data protection in TIBCO MCP Hub is not only an After Tool concern. The same Virtual Server can govern three request flows — **Tool calls**, **Prompt fetches** and **Resource fetches** — so privacy rules can apply to tool answers, prompt arguments, rendered prompts and resource reads. Use these plugins with access policies when the question is "who may call this?" ([access policies](policies-access.md)), with content-safety policies when the question is "is this content safe?" ([content safety](policies-content-safety.md)), and with deliberate priority choices when several plugins share a lane ([plugin pipeline](plugin-pipeline.md)).

| Plugin / capability | Scenario | Pipeline and hook(s) | What it proves |
|---|---|---|---|
| PIIFilterPlugin | The same customer record is visible to one role and masked for another. | Tool calls — After Tool (`tool_post_invoke`); also declares `tool_pre_invoke`, `prompt_pre_fetch` and `prompt_post_fetch` | Masking can be applied per Virtual Server, so the server a role can reach determines whether PII is returned or redacted. |
| SecretsDetection + EncodedExfilDetector | A permitted partner-facing tool answer contains secret-looking material or encoded data. | Tool calls — After Tool (`tool_post_invoke`); SecretsDetection also declares `prompt_pre_fetch`, `tool_pre_invoke` and `resource_post_fetch`; EncodedExfilDetector also declares `prompt_pre_fetch` and `resource_post_fetch` | Response inspection stops unsafe answers while still allowing ordinary answers from the same tool set. |
| DenyListPlugin | An agent supplies a guarded word as a prompt argument. | Prompt fetches — Before Prompt (`prompt_pre_fetch`) | The plugin scans caller-supplied prompt arguments, not the prompt template itself. |
| FileTypeAllowlistPlugin | A document assistant tries to read a file type outside the intended corpus. | Resource fetches — Before Resource (`resource_pre_fetch`) plus MIME enforcement at `resource_post_fetch` | One binding confines resource reads by URI extension before fetch and declared MIME type after fetch. |
| ResourceFilterExample | A resource perimeter blocks protocols, hosts and over-large content, and redacts a marker. | Resource fetches — `resource_pre_fetch`, `resource_post_fetch`; plugin also declares `prompt_post_fetch` and `tool_post_invoke`, which it does not filter | Resource governance can protect the retrieval layer, but this example plugin is not a general DLP engine. |
| PrivacyNoticeInjector | Every rendered prompt must carry an operator-authored notice. | Prompt fetches — After Prompt (`prompt_post_fetch`) | A gateway-side transform adds a configured notice to the rendered prompt without blocking the fetch. |

## PIIFilterPlugin — the mask knows who asked

> "The support team needs useful answers; the fraud team needs the full record."

**Example scenario.** A service organisation exposes one customer-record tool through TIBCO MCP Hub. Support analysts need the company name, case context and contact fields, but they do not need to see raw personal data. Fraud investigators, using the same upstream record for a different job, are authorised to inspect the full record.

The important design choice is not two different tools. It is two Virtual Servers over the same tool: one server for investigators with no masking policy, and one server for support analysts with PIIFilterPlugin attached. Pair that with identity and access controls so the role reaches the right server.

**What it proves.** The same tool and same upstream row can return raw values to one authorised audience and `[PII_REDACTED]` to another. The decision is enforced by the gateway path the role can use, not by an instruction written into the agent prompt.

**When to use it.** Use PIIFilterPlugin when the same data has multiple audiences and one audience can work with masked values. If nobody who can reach the tool should see the raw data, use access controls instead; see [access policies](policies-access.md). If the answer should be refused rather than masked, use a blocking policy such as SecretsDetection.

### Where it sits

| Setting | Value |
|---|---|
| Plugin | `PIIFilterPlugin` |
| Pipeline | Tool calls |
| Hook | After Tool (`tool_post_invoke`) lane; the plugin also declares `tool_pre_invoke`, `prompt_pre_fetch` and `prompt_post_fetch`, and one assignment activates every hook it declares, so detected values in tool arguments and prompts are masked too |
| Mode | Block (`sequential`), the Hub's attach default; a detection still masks and the call continues unless `block_on_detection` is set |
| Attached to | Analyst-facing Virtual Server |

### Configuration

1. Create or select the Virtual Server used by the role that should receive masked answers.
2. Open the Virtual Server's Command Center and select the **Tool calls** flow.
3. Add `PIIFilterPlugin` from the After Tool lane.
4. Set the mode to **Block** (`sequential`).
5. Apply this JSON configuration, then Push to Gateway and wait for the server to show as synced.

```json
{
  "default_mask_strategy": "redact",
  "redaction_text": "[PII_REDACTED]",
  "detect_email": true,
  "detect_phone": true,
  "log_detections": true
}
```

`default_mask_strategy: "redact"` keeps the answer usable while replacing detected values with the `redaction_text`; set it explicitly, because the plugin's own default is `[REDACTED]`. The `detect_email` and `detect_phone` flags enable the detectors used in this scenario. `log_detections` records that a detection occurred; a successful redaction is not the same as a blocked call.

### What happens

The investigator-facing server returns the raw customer record. If your gateway enables PIIFilterPlugin gateway-wide, the baseline is masked too; compare against a server where it is not bound. The analyst-facing server returns the same record structure, but detected email and phone values are replaced with `[PII_REDACTED]`. In an agent conversation, the support analyst can still answer the business question, but the sensitive fields are masked.

Dashboard traces show the tool call succeeding. Expect detection or transformation evidence, not a policy-block event, because the plugin did not refuse the answer.

### Setup tips

- Test the unmasked control first. Capture the fact that the upstream record contains values that can be masked; otherwise a broken empty answer can look like success.
- Keep agent instructions neutral. If the agent self-censors before calling the tool, the plugin never runs.
- Wait for the gateway to report synced. A binding saved only in the Hub does not affect runtime traffic until it is pushed.
- Choose realistic test data for the detector you enable. For example, validate phone-number formats used in your region before relying on a phone mask; see Considerations.

### Considerations

PIIFilterPlugin is a pattern detector. Validate phone-detector coverage against the number formats your data contains before relying on it.

PIIFilterPlugin is a response transformer in this use case, not an access-control substitute. It does not decide who may call the server, and it does not prove that masked values are sufficient for every role. For strong separation, use separate Virtual Servers and bind PermissionPlugin on the unmasked server so that only the authorised role can reach it.

## SecretsDetection and EncodedExfilDetector — a fair question, an unfair answer

> "The partner was allowed to ask; the answer carried material that should not leave."

**Example scenario.** A partner-facing agent receives scoped access to a small set of diagnostic tools. Its token is valid, the tools are intentionally exposed, and a normal status question should be answered. The risk sits on the outbound side: a backend may echo credential-like text into a response, or a field may contain encoded data that does not look dangerous to a reader at first glance.

Attach response-side detectors to the governed Virtual Server and keep an ungoverned twin only while validating the policy. The control should show that the same upstream answer would otherwise leave the gateway.

**What it proves.** A permitted tool call can still be stopped after execution when the response is unsafe. SecretsDetection blocks secret-looking material, EncodedExfilDetector blocks encoded exfiltration patterns, and an ordinary answer from the same tool set still passes.

**When to use it.** Use these plugins when another party's agent can call tools that may return sensitive material. Use PIIFilterPlugin when masking known PII fields is enough. Use content-safety plugins for harmful generated text rather than credential or encoded-data detection; see [content safety](policies-content-safety.md).

### Where it sits

| Setting | Value |
|---|---|
| Plugin | `SecretsDetection` and `EncodedExfilDetector` |
| Pipeline | Tool calls |
| Hook | After Tool (`tool_post_invoke`) lane. One assignment activates each plugin on every hook it declares: SecretsDetection also scans prompt arguments (`prompt_pre_fetch`), tool arguments (`tool_pre_invoke`) and fetched resources (`resource_post_fetch`); EncodedExfilDetector also scans prompt arguments (`prompt_pre_fetch`) and fetched resources (`resource_post_fetch`) |
| Mode | Block (`sequential`) |
| Attached to | Partner-facing Virtual Server |

### Configuration

1. Open the partner-facing Virtual Server in Command Center.
2. Select the **Tool calls** flow and open the After Tool lane.
3. Add `SecretsDetection` in **Block** (`sequential`) mode.
4. Apply the blocking configuration.

```json
{ "block_on_detection": true, "redact": false }
```

5. On the same After Tool lane, add `EncodedExfilDetector` in **Block** (`sequential`) mode.
6. Apply the blocking configuration, then Push to Gateway and confirm the push.

```json
{ "block_on_detection": true }
```

For SecretsDetection, `block_on_detection: true` refuses the entire answer. `redact: false` means the caller receives a policy error rather than a modified answer. For EncodedExfilDetector, the policy blocks when its encoded-data detector fires.

### What happens

| Tool response | Ungoverned server | Governed server |
|---|---|---|
| Diagnostic answer containing credential-like material | Returns the raw answer | Blocked with `SECRETS_DETECTED` |
| Answer containing encoded exfiltration content | Returns the raw answer | Blocked with `ENCODED_EXFIL_DETECTED` |
| Ordinary status answer | Returns the answer | Returns the answer |

The refusal does not echo the detected secret or encoded content back to the caller. Dashboard traces show blocked rows with the plugin that decided the outcome and the corresponding violation details.

### Setup tips

- Attach EncodedExfilDetector on the After Tool lane for outbound answers. It has no Before Tool hook, and the Hub refuses that lane for it.
- Keep a control server during validation. Without a leaking control, a block alone does not prove the detector caused the difference.
- Review broad heuristics before enabling them. Some high-entropy or encoded-looking patterns can match benign IDs, hashes or other operational strings.
- The toolbar push may open a preview dialog. Confirm inside the dialog and wait for synced status.

### Considerations

Blocking is a business choice. It protects the boundary by refusing the answer, but it also means the partner receives no partial result. If a workflow needs safe partial answers, evaluate a redaction posture for secrets and document exactly which detector classes can redact safely.

## DenyListPlugin — the word the agent must never put into a prompt

> "A word list sounds simple until you ask what text it actually reads."

**Example scenario.** A semiconductor engineering assistant uses a Hub-authored prompt to draft customer-facing notes. The prompt template itself contains a sensitive codename in fixed wording, and the caller supplies values such as product and audience. The organisation wants to block the agent when it puts the guarded word into a prompt argument, and also wants rendered prompt text cleaned before it reaches the model.

This scenario uses two policies for two different jobs. `DenyListPlugin` refuses prompt fetches when caller-supplied arguments contain a guarded substring. `ReplaceBadWordsPlugin` rewrites rendered prompt text, including text that came from the template itself.

**What it proves.** DenyListPlugin scans `payload.args`; it does not scan the prompt template text. A codename written into the template can pass the deny list, while the same codename supplied as an argument is refused. A substring match is literal: a normal word that contains the denied substring is refused the same way.

**When to use it.** Use DenyListPlugin when the risk is a specific word or substring supplied by a caller as a prompt argument. Do not use it to police authored prompt templates; review prompt content separately or use a post-fetch rewrite. If the concern is harmful content rather than protected words, see [content safety](policies-content-safety.md).

### Where it sits

| Setting | Value |
|---|---|
| Plugin | `DenyListPlugin` |
| Pipeline | Prompt fetches |
| Hook | Before Prompt (`prompt_pre_fetch`) |
| Mode | Block (`sequential`) |
| Priority | `100` in the reference scenario |
| Attached to | Virtual Server exposing the prompt |

For the rewrite half of the scenario:

| Setting | Value |
|---|---|
| Plugin | `ReplaceBadWordsPlugin` |
| Pipeline | Prompt fetches |
| Hook | After Prompt (`prompt_post_fetch`) lane; the plugin also declares `prompt_pre_fetch`, `tool_pre_invoke` and `tool_post_invoke`, and one assignment activates every hook it declares |
| Mode | Transform (`transform`) |
| Priority | `150` in the reference scenario |

### Configuration

1. Open the governed Virtual Server in Command Center.
2. Select the **Prompt fetches** flow.
3. Add `DenyListPlugin` on the Before Prompt lane with priority `100`.
4. Apply the guarded word list.

```json
{ "words": ["nova"] }
```

5. Add `ReplaceBadWordsPlugin` on the After Prompt lane with priority `150`.
6. Apply the rewrite rule, then Push to Gateway and wait for synced status.

```json
{
  "words": [
    {
      "search": "(?i)nova[-\\w]*",
      "replace": "[internal codename]"
    }
  ]
}
```

The deny-list configuration is case-insensitive substring matching. The rewrite configuration is a regular expression replacement; in this example it rewrites `nova`, `Nova-7` and similar word forms.

### What happens

| Prompt fetch | Result |
|---|---|
| Clean arguments, but the template contains the codename | DenyListPlugin does not refuse; ReplaceBadWordsPlugin rewrites the rendered prompt. |
| Argument value contains the guarded word | Refused with `Plugin Violation: Denied words found in prompt`. |
| Argument value contains a normal word with the guarded substring | Refused with the same violation shape, for example `product: contains 'nova'`. |
| Ungoverned baseline | The same prompt fetches without refusal or rewrite. |

If ReplaceBadWordsPlugin is also bound, review [plugin ordering](plugin-pipeline.md). One assignment activates the plugin on every hook it declares; the lane you choose is where the Hub shows it. In the reference environment, the single After Prompt assignment also rewrote tool replies, and the plugin also rewrites prompt and tool arguments.

### Setup tips

- Put a guarded word in a prompt argument to test DenyListPlugin. A word hard-coded into the template tests a different concern.
- Test a benign substring case before rollout. Deny lists do not understand word intent.
- Place the rewrite after the prompt is rendered if template text must be cleaned.
- Verify priorities when both refusal and rewrite policies are present. A refusal before render prevents post-fetch rewriting from running.

### Considerations

DenyListPlugin is precise about where it reads and imprecise about meaning. It reads caller-supplied prompt arguments only, and it matches substrings rather than concepts. It does not catch paraphrases, synonyms or sensitive meaning expressed without the configured word.

## FileTypeAllowlistPlugin — the file it should not have read

> "The tools were safe; the corpus returned a file that was never meant to be read as a document."

**Example scenario.** A document-grounded assistant reads process notes, equipment instructions and reports through MCP resources. It resolves a URI and reads whatever the corpus returns. If an executable or opaque binary slips into that corpus, the assistant can read it unless the retrieval path is governed.

FileTypeAllowlistPlugin confines resource reads to document types you explicitly allow. It checks the URI extension before fetching the body and the declared MIME type after fetching it.

**What it proves.** One Virtual Server binding can stop a resource read at the retrieval layer. A `.exe` URI is refused before the fetch, and an extensionless binary blob is refused by the MIME backstop after fetch.

**When to use it.** Use FileTypeAllowlistPlugin when you can enumerate the document types a resource corpus should return. Do not use it as a malware scanner, content classifier or rate limiter. If protocol, host, size or simple resource-body redaction are the concern, evaluate ResourceFilterExample below.

### Where it sits

| Setting | Value |
|---|---|
| Plugin | `FileTypeAllowlistPlugin` |
| Pipeline | Resource fetches |
| Hook | Before Resource (`resource_pre_fetch`) lane; one assignment also activates the MIME check at `resource_post_fetch` |
| Mode | Block (`sequential`) |
| Attached to | Virtual Server exposing resources |

### Configuration

1. Open the governed Virtual Server in Command Center.
2. Select the **Resource fetches** flow (the default is **Tool calls**).
3. Add `FileTypeAllowlistPlugin` on the Before Resource lane.
4. Apply both allow-lists.

```json
{
  "allowed_extensions": [".md", ".txt"],
  "allowed_mime_types": ["text/markdown", "text/plain"]
}
```

5. Set the mode to **Block** (`sequential`).
6. Push to Gateway and wait for synced status.

`allowed_extensions` is the pre-fetch gate on the URI path. `allowed_mime_types` is the post-fetch backstop that checks the content type returned by the upstream server. Matching is case-insensitive.

### What happens

| Resource read | Result |
|---|---|
| `operating-note.md` with `text/markdown` | Allowed. |
| `firmware-loader.exe` | Refused before fetch with `Plugin Violation: Extension .exe is not allowed`. |
| Extensionless binary resource | Refused after fetch with `Plugin Violation: MIME application/octet-stream is not allowed`. |
| Ungoverned baseline | The same reads succeed, proving the policy caused the block. |

Only one chip appears on the Before Resource lane. The MIME check is still active at runtime: one assignment activates the plugin on every hook it declares; the lane you choose is where the Hub shows it.

### Setup tips

- Select **Resource fetches** before attaching the plugin; the default flow is **Tool calls**.
- Pair extension and MIME lists so that both gates apply.
- Both checks rely on declared metadata. Require upstream servers to return an accurate MIME type and configure both the extension and the MIME allow-lists; the plugin does not inspect file content.
- Use **Block** (`sequential`). `audit` or `transform` modes can record a violation while still forwarding the read.

### Considerations

This is an allow-list. Anything not listed is blocked because it is absent from the allowed set, not because the plugin recognises the type as malicious. Empty lists are allow-all for that gate, not deny-all. An empty `allowed_extensions` list disables extension checks; an empty `allowed_mime_types` list disables MIME checks.

## ResourceFilterExample — the resource has a perimeter too

> "A resource URI can be outside the perimeter even when it looks like just another document."

**Example scenario.** An operations assistant reads manuals, partner notices and service documents from many systems, all exposed as MCP resources. The organisation wants a simple perimeter: only allowed protocols, no reads from blocked hosts, no over-large resource bodies, and a literal sensitive marker redacted if it appears in a resource body.

`ResourceFilterExample` demonstrates that perimeter on the Resource fetches flow. Its catalogue name includes "Example", and that matters: treat it as measured behaviour to understand and validate, not as a complete data-loss-prevention product.

**What it proves.** A resource policy can refuse a URI before retrieval based on protocol or host, refuse content after retrieval based on size, and rewrite matching text in an allowed resource body. The same binding can also activate a declared tool hook, even when the tool hook is a pass-through.

**When to use it.** Use it to understand and validate resource-pipeline governance for protocol, host, size and simple regex replacement. Use FileTypeAllowlistPlugin when the core requirement is file type. Use a dedicated content-safety or DLP service when the requirement is semantic classification.

### Where it sits

| Setting | Value |
|---|---|
| Plugin | `ResourceFilterExample` |
| Pipeline | Resource fetches |
| Hook | `resource_pre_fetch` lane; one assignment also activates `resource_post_fetch`. The plugin also declares `prompt_post_fetch` and `tool_post_invoke`; it does not filter prompts or tool results |
| Mode | Block (`sequential`) |
| Priority | `75` in the reference scenario |
| Attached to | Virtual Server exposing resources |

### Configuration

1. Open the governed Virtual Server in Command Center.
2. Select the **Resource fetches** flow.
3. Add `ResourceFilterExample` on the `resource_pre_fetch` lane.
4. Apply this JSON configuration, then Push to Gateway and wait for synced status.

```json
{
  "max_content_size": 200,
  "allowed_protocols": ["file", "http", "https"],
  "blocked_domains": ["blocked-docs.example.com"],
  "content_filters": [
    {
      "pattern": "SENSITIVE-ID-\\d+",
      "replacement": "[REDACTED]"
    }
  ]
}
```

`max_content_size` sets the body-size ceiling in bytes. `allowed_protocols` accepts only the named URI schemes. `blocked_domains` rejects specific hosts. `content_filters` applies literal regex replacement to resource content that otherwise passes.

### What happens

| Resource or tool leg | Governed result |
|---|---|
| Allowed note | Reads successfully. |
| `ftp://` URI | Refused with `Plugin Violation: Protocol 'ftp' is not in allowed list`. |
| URI on `blocked-docs.example.com` | Refused with `Plugin Violation: Domain 'blocked-docs.example.com' is in blocked list`. |
| Body larger than `200` bytes | Refused with a maximum-size violation. |
| Body containing `SENSITIVE-ID-12345` | Reads successfully with `[REDACTED]` in place of the marker. |
| Ordinary tool reply | Passes through unchanged in the reference scenario. |

The baseline server should read all resources without these refusals, including the unredacted marker. That control proves the perimeter is applied by the bound plugin.

### Setup tips

- Validate every configured axis separately: protocol, host, size and redaction.
- Use a host value you control and document why it is blocked.
- Treat regex replacement as exact text handling. It does not infer meaning beyond the configured pattern.
- Review declared hooks before rollout. A single binding can activate more runtime hooks than the lane where it was attached; see [plugin pipeline](plugin-pipeline.md).

### Considerations

ResourceFilterExample is a perimeter example, not a general DLP engine, malware scanner or threat feed. It does not learn, classify or consult external reputation. Its "Example" label is a reminder to validate suitability, support posture and behaviour before relying on it for production governance.

## PrivacyNoticeInjector — the notice the gateway adds

> "A required line belongs in the rendered prompt even when a prompt author forgets it."

**Example scenario.** An organisation has a standing notice that every rendered prompt should carry. It might be a privacy hygiene instruction, an AI-disclosure line or another policy reminder. Relying on each prompt author to include the line is fragile, especially when prompts are federated from multiple servers.

PrivacyNoticeInjector adds the operator-authored notice after the prompt is rendered and before it is handed to the model. It is a transform, not a block.

**What it proves.** The same prompt fetched through a baseline server returns the authored text unchanged. Fetched through a governed server, the rendered prompt includes the configured marker and notice.

**When to use it.** Use it when the requirement is "this line must be present in the rendered prompt". Do not use it as proof that a model obeyed the notice or that an end user read it. For rules that must block unsafe prompt content, use a blocking prompt policy instead.

### Where it sits

| Setting | Value |
|---|---|
| Plugin | `PrivacyNoticeInjector` |
| Pipeline | Prompt fetches |
| Hook | After Prompt (`prompt_post_fetch`) |
| Mode | Transform (`transform`) |
| Attached to | Virtual Server default for prompts |

### Configuration

1. Open the governed Virtual Server in Command Center.
2. Select the **Prompt fetches** flow.
3. Add `PrivacyNoticeInjector` on the After Prompt lane.
4. Choose Transform mode.
5. Apply the notice text, placement and marker, then Push to Gateway and wait for synced status.

```json
{
  "notice_text": "Always tell the user they are speaking with an AI assistant, not a person.",
  "placement": "append",
  "marker": "[AI-NOTICE]"
}
```

`notice_text` is the operator-authored line. `placement` controls whether the notice is prepended, appended or placed in a separate user-role message. `marker` is the deduplication token; the plugin injects `{marker} {notice_text}`. With `prepend` or `append`, the notice is added to user-role messages; `separate_message` adds it as a new user-role message.

### What happens

| Prompt fetch | Result |
|---|---|
| Baseline server | Returns the rendered prompt without the notice. |
| Governed server | Returns the same rendered prompt plus `[AI-NOTICE] Always tell the user they are speaking with an AI assistant, not a person.` |
| Prompt already containing the marker | Left unchanged to avoid duplicate insertion. |

No violation is raised. The fetch succeeds, and the returned prompt content is modified.

### Setup tips

- Use a marker that is unique enough for your organisation. A prompt that already contains the marker is skipped, even if the existing notice is stale or incomplete.
- Confirm the configured default text. The shipped default is a PII and secrets hygiene reminder, not an AI-disclosure notice.
- Scope expectations carefully in rollout material. The plugin puts the line into the rendered prompt; it cannot force model behaviour or user acknowledgement.
- Test against a baseline server to prove the notice came from the gateway policy.

### Considerations

PrivacyNoticeInjector guarantees only that the configured line is present in the rendered prompt when injection occurs. It does not verify that the line is current, legally sufficient or read by a person. Deduplication is marker-based, so marker governance is part of the policy design.
