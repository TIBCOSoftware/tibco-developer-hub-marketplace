# Policies — Reliability, rate & cost control

Reliability and cost policies help a TIBCO MCP Hub operator govern the behaviours that usually appear after the first successful agent rollout: loops, repeated reads, transient outages, long-running calls and unexpectedly large tool results. They run in the MCP Gateway plugin pipeline, so the decision is central, consistent and visible from every client that uses the Virtual Server. For hook ordering, priority and stack behaviour, see [Plugin pipeline](plugin-pipeline.md). For trace export, SIEM destinations and related controls, see [observability plugins](policies-observability.md).

| Plugin / capability | Scenario | Hook(s) | What it proves |
|---|---|---|---|
| `RateLimiterPlugin` | An unattended agent exceeds a safe call rate, and the operations team needs both a block and an alarm. | Before Tool (`tool_pre_invoke`); also Prompt Fetch (`prompt_pre_fetch`) | The gateway refuses over-limit calls before the upstream tool runs, records the block in Traces and can export enforced denials to a SIEM destination. |
| Token budget controls (`ToonEncoder`, `OutputLengthGuardPlugin`) | A fleet-scale agent programme returns large structured payloads that were never costed. | After Tool (`tool_post_invoke`) | Tabular JSON can be reduced before it reaches the model, while runaway results can be truncated or blocked by policy. |
| `CircuitBreaker` | A queue-processing agent keeps calling a backend that is already failing. | Before Tool and After Tool (`tool_pre_invoke`, `tool_post_invoke`) | The gateway learns from real failures, opens the circuit per Virtual Server and tool, then refuses further calls before they reach the dependency. |
| `RetryWithBackoffPlugin` | A backend has brief 503 blinks and occasional sustained outages. | After Tool (`tool_post_invoke`) plus a breaker on the same Virtual Server | A retry can hide a transient failure from the agent, but every retry is another backend call and needs a breaker beside it. |
| `CachedToolResultPlugin` | A fleet asks the same safe read repeatedly. | Before Tool and After Tool (`tool_pre_invoke`, `tool_post_invoke`) | With both serving keys enabled, the first call stores the result and the warm repeat is served from the gateway cache without touching the upstream. |
| `ResponseCacheByPrompt` | Operators see an approximate-cache flag and need to know whether work was skipped. | Before Tool and After Tool (`tool_pre_invoke`, `tool_post_invoke`) | The plugin annotates likely repeated prompts; it does not serve a cached response and does not suppress the underlying tool call. |

## RateLimiterPlugin — The silent alarm and the alarm that rings

`RateLimiterPlugin` gives a Virtual Server a per-caller ceiling. It runs before the tool call, so a refused call does not reach the upstream. The same policy also creates an operational question: after the gateway refuses a runaway agent, who is notified?

### Scenario 1 — The silent alarm

> “The agent looped all night. Did the gateway stop it?”

**Example scenario.** An operations team gives an unattended assistant a backlog to work through overnight. At 02:00 it hits a case it cannot complete, retries, then repeats the same failed path. By morning it could have called an internal tool tens of thousands of times. A human at a keyboard is naturally rate-limited; a loop is not.

!!! note "Reference measurements"
    Figures on this page come from an internal reference test environment (one gateway, one caller,
    test fixtures). They illustrate relative behaviour; they are not performance commitments. Measure
    in your own environment.

In the reference environment, a burst of eight calls against an ungoverned control passed all eight. The governed Virtual Server allowed four and refused four with `RATE_LIMIT — rate limit exceeded`. For a conversational test agent client, a per-second window was too high to trigger because model latency spaces calls out; a per-minute window such as `2/m` was the useful control.

**What it proves.** Rate limiting caps one caller without spending everyone else’s budget. TIBCO MCP Hub records the refusal in Traces, so the operator can prove the ceiling held.

**When to use it.** Use it on any Virtual Server that an unattended agent, scheduled process or batch integration can reach. Do not use it as a substitute for backend capacity planning: too low a limit breaks valid work, and too high a limit only documents the flood. Size the window from the caller’s actual behaviour.

#### Where it sits

| Setting | Value |
|---|---|
| Plugin | `RateLimiterPlugin` |
| Hook | Before Tool (`tool_pre_invoke`); the plugin also declares Prompt Fetch (`prompt_pre_fetch`) |
| Mode | Block (`sequential`) |
| Priority | `20` (catalogue default) |
| Attached to | Virtual Server |

#### Configuration

1. Open the governed Virtual Server and go to **Command Center** → **Tool calls**.
2. Add `RateLimiterPlugin` on **Before Tool**. One assignment activates the plugin on every hook it declares; the lane you choose is where the Hub shows it.
3. Set mode to **Block** (`sequential`).
4. Paste and apply the JSON policy.
5. Push to Gateway and wait until the Virtual Server is `synced`.
6. Test against an ungoverned control first, then against the governed Virtual Server.

```json
{ "by_user": "2/s" }
```

For an interactive agent, use the same policy shape with a slower window, for example:

```json
{ "by_user": "2/m" }
```

`by_user` is the important key: it gives each caller its own ceiling. The binding mode, not the JSON, decides whether an over-limit call is refused: **Block** (`sequential`) refuses it. Keys you leave out keep the gateway's configured defaults, so set the limit you want explicitly and verify it on your release.

#### What happens

A refused call returns a policy error to the MCP client. A test agent client that reads the structured denial shows it as **Blocked by policy**, with detail like:

```text
tool_pre_invoke blocked by plugin RateLimiterPlugin: RATE_LIMIT - Rate limit exceeded
```

Dashboard → Traces → Blocked shows rows such as `Blocked (RateLimiterPlugin · RATE_LIMIT)`. That trace is the first operational record to use when an owner asks whether the ceiling actually held.

#### Setup tips

- Always keep an ungoverned control Virtual Server over the same tool while proving the policy. If both paths fail, you are testing the upstream, not the limiter.
- Confirm the push inside the preview dialog. A policy that exists only in the Hub draft has no runtime effect.
- Tune the window to the caller. Back-to-back API bursts can hit `2/s`; a conversational agent often needs a per-minute threshold.
- Treat notification plugins separately from the rate limit. A Before Tool denial stops the call before any After Tool plugin runs, so `WebhookNotification` does not alert on a rate-limit block. Use SIEM export (Scenario 2) for denial alerts.

#### Considerations

Rate limiting does not choose the correct business limit for you. The safe ceiling comes from the upstream service’s capacity and the agent’s expected call pattern. A client can still retry after a refusal; the gateway will continue to enforce the ceiling, but the caller may need its own backoff behaviour.

### Scenario 2 — The alarm that rings

> “The ceiling held. Does the security desk see the denial where it already works?”

**Example scenario.** A security operations team does not monitor the Hub console all night; it monitors its SIEM. A policy denial that only appears in a product trace can be missed by the team responsible for audit and incident triage. The gateway can export enforced denials to an operator-owned destination so the SOC receives one event per block.

**What it proves.** With policy-denial export enabled, each enforced rate-limit block can become a SIEM event with the event type, category, action, hook, plugin, policy code and correlation id. The Hub trace remains the richer place to inspect the Virtual Server context. For the broader observability policy stack, see [observability plugins](policies-observability.md).

**When to use it.** Use this when a SOC reads SIEM data and must learn about enforced policy denials without opening TIBCO MCP Hub. If there is no SIEM workflow, Dashboard → Traces is already the operational alarm.

#### Where it sits

| Setting | Value |
|---|---|
| Runtime control | Gateway SIEM export |
| Exported event category | `policy_denial` |
| Rate-limit event type | `rate_limit_exceeded` |
| Hub policy | `RateLimiterPlugin` on Before Tool |
| Persistent configuration | Gateway deployment extra environment |

#### Configuration

1. Keep the `RateLimiterPlugin` configuration from the first scenario on the governed Virtual Server.
2. Configure the gateway's extra environment — a ConfigMap or Secret you own, referenced from the gateway deployment values.
3. Roll the gateway so the environment is active.
4. Trigger an over-limit call and verify both the SIEM collector and Dashboard → Traces.

```yaml
SIEM_EXPORT_ENABLED: "true"
SIEM_DESTINATIONS: '[{"name":"SOC","type":"webhook","url":"https://<your-collector>/hook","filters":{"categories":["policy_denial"]}}]'
```

#### What happens

When SIEM export is enabled, each enforced denial produces an event like this in shape:

```text
event_type=rate_limit_exceeded
severity=MEDIUM
category=policy_denial
action_taken=denied
hook=tool_pre_invoke
code=RATE_LIMIT
plugin=RateLimiterPlugin
```

The event intentionally carries little identity detail; join it with Hub traces by server and session timing rather than expecting it to name every caller, tool and tenant. The client still receives the same `RATE_LIMIT` denial, and the Hub still records the blocked trace.

#### Setup tips

- Verify on your release that an enforced denial reaches the collector before relying on it operationally.
- Test an allowed call first. The collector should stay quiet until there is an enforced denial.
- Monitor destination health and event failures, not only event receipt.
- Keep SIEM destination configuration in deployment values so it survives a gateway restart.

#### Considerations

The SIEM event may not include user, tenant, tool or server names, and the correlation id can identify the MCP session rather than the individual denied call. A flood of denial events is queued; if the queue is exhausted, older events can be dropped. Use SIEM export as the alerting path and Hub traces as the investigation path.

## Token budget — The bill nobody budgeted

> “The answer is correct, but the payload is too expensive to send to every model turn.”

**Example scenario.** An order-management assistant calls a federated tool that returns order lines, inventory rows or ledger entries as verbose JSON. One pilot looks affordable, then the same design is rolled out across hundreds of agents. Each tool result enters the model context and is then reasoned over; repeated keys, braces and formatting become recurring cost.

The gateway is the shared return path for those payloads. `ToonEncoder` rewrites JSON results into compact TOON only when the TOON representation is smaller; tabular data with repeated field names benefits most. `OutputLengthGuardPlugin` caps payloads that are too large for safe model use, either by truncating with an ellipsis or by blocking the result with a named violation.

**What it proves.** In the reference environment, the same 98-row order-lines result measured 8043 bytes as JSON and 1825 bytes as TOON. One widely used tokenizer counted 2588 tokens to 1328 tokens (counts vary by model): about 77% smaller in bytes and about 49% fewer tokens. The agent still answered the same business question correctly; the policy reduced the payload, not the data meaning.

**When to use it.** Use `ToonEncoder` for structured, tabular results with repeated field names. Use `OutputLengthGuardPlugin` where a tool can return an unexpectedly large body. Do not claim universal cost reduction: string-heavy payloads can be larger in TOON, and the encoder correctly leaves them unchanged.

### Where it sits

| Setting | Value |
|---|---|
| Plugins | `ToonEncoder`, `OutputLengthGuardPlugin` |
| Hook | After Tool (`tool_post_invoke`) |
| Mode | Transform (`transform`) for `ToonEncoder`, which never blocks; Block (`sequential`) for `OutputLengthGuardPlugin` when `strategy` is `block` |
| Attached to | Virtual Server |

### Configuration

1. Open the governed Virtual Server for the tool that returns large results.
2. Add `ToonEncoder` on **After Tool** and keep it in a non-blocking transform path.
3. Optionally add `OutputLengthGuardPlugin` on **After Tool** to cap individual text values in the result.
4. Apply the JSON configuration.
5. Push to Gateway and compare the governed result with an ungoverned control.

`ToonEncoder` configuration:

```json
{
  "min_size_bytes": 100,
  "max_size_bytes": 1048576,
  "include_tools": null,
  "exclude_tools": [],
  "add_format_marker": true
}
```

`OutputLengthGuardPlugin` configuration:

```json
{
  "strategy": "truncate",
  "max_chars": 15000,
  "limit_mode": "character",
  "ellipsis": "…"
}
```

Set `strategy` to `block` when the client should receive a refusal instead of a shortened result.

### What happens

For a tabular JSON result, `ToonEncoder` compares the original and encoded sizes. If TOON is smaller, the result is returned in TOON format and marked so a reader can tell that it was transformed. If it is not smaller, the original JSON is returned unchanged.

`OutputLengthGuardPlugin` applies `max_chars` to each text value in the result, not to the response as a whole. In `truncate` strategy it shortens an over-long value and marks it as truncated. With the `block` strategy, it refuses the result with `OUTPUT_LENGTH_VIOLATION`. That named code is the reliable signal that the gateway evaluated the policy; do not treat a generic transport error as a policy block.

### Setup tips

- Measure tokens as well as bytes. Byte reduction can overstate model cost reduction because tokenizers already compress some JSON structure.
- Keep a baseline Virtual Server with no transform so the comparison uses the same tool, arguments and data.
- Use production thresholds after the proof. Lowering `min_size_bytes` can help demonstrate the transform, but it is not always the right operational setting.
- For `OutputLengthGuardPlugin`, select `truncate` when the agent can work with a bounded partial answer; select `block` when partial output would be unsafe or misleading.

### Considerations

`OutputLengthGuardPlugin` does not set an aggregate response or model-context budget; validate total response size separately. The gateway reduces payload size; it does not issue the model invoice. The economic claim is a measured reduction in payload and tokens, not a billed-currency amount. The saving depends on data shape, and the transform should be validated against your own tool results. `OutputLengthGuardPlugin` protects the model context from an oversized result, but it runs after the tool completes, so it does not reduce the upstream work that produced the oversized payload.

## CircuitBreaker — Stop the cascade

> “The backend is already failing. The agent’s persistence must not deepen the outage.”

**Example scenario.** A batch agent works through a queue of work items, calling a backend once per item. The dependency starts returning failures. A human operator might stop after three 500 responses; an agent can continue through the queue and add load to a system that is already recovering.

`CircuitBreaker` reacts to failure, not volume. It records results after the tool call and opens when consecutive failures or error rate cross the configured threshold. While open, it refuses the next call at the gateway before the upstream is reached. `Watchdog` is the companion control for over-long single calls: it detects calls that exceed a duration limit and can either warn or block the result.

**What it proves.** In the reference environment with a lowered threshold, the baseline agent made six failing backend calls. The governed agent made three real calls, then the fourth was refused at `tool_pre_invoke` with `CIRCUIT_OPEN`; the observed agent stopped the remaining queue. The deterministic product behaviour is the gateway shield, not the model’s choice to stop.

**When to use it.** Use `CircuitBreaker` on queue processors, fan-out agents and integrations that call dependencies with known failure modes. Use `Watchdog` for tools that can hang or run beyond an acceptable response time. Do not use either as a replacement for upstream health checks, retries or capacity management; they control the client-facing gateway path.

### Where it sits

| Setting | Value |
|---|---|
| Plugin | `CircuitBreaker` |
| Hooks | Before Tool and After Tool (`tool_pre_invoke`, `tool_post_invoke`) |
| Mode | Block (`sequential`) |
| Attached to | Virtual Server; state is per Virtual Server and tool |

| Setting | Value |
|---|---|
| Plugin | `Watchdog` |
| Hooks | Before Tool and After Tool (`tool_pre_invoke`, `tool_post_invoke`) |
| Default action | `warn` |
| Blocking code | `WATCHDOG_TIMEOUT` |

### Configuration

1. Open the governed Virtual Server and go to **Command Center** → **Tool calls**.
2. Add `CircuitBreaker` once. One assignment activates the plugin on every hook it declares; the lane you choose is where the Hub shows it. Attaching it on **Before Tool**, where it refuses, keeps the display aligned with the behaviour.
3. Apply a threshold configuration that matches the dependency’s tolerance.
4. Add `Watchdog` if individual calls need a duration policy.
5. Push to Gateway and test the trip, open, half-open and close paths.

Production-oriented `CircuitBreaker` example:

```json
{
  "consecutive_failure_threshold": 5,
  "error_rate_threshold": 0.5,
  "window_seconds": 60,
  "min_calls": 10,
  "cooldown_seconds": 60
}
```

A short reference exercise used a lower threshold so the trip occurred mid-queue:

```json
{ "consecutive_failure_threshold": 3, "cooldown_seconds": 2 }
```

`Watchdog` example:

```json
{
  "max_duration_ms": 30000,
  "action": "block"
}
```

### What happens

When closed, calls pass through and the breaker records success or failure. When failure thresholds are met, the breaker opens. While open, the next call is refused at Before Tool with a stable code:

```text
CIRCUIT_OPEN - Circuit open
```

After `cooldown_seconds`, one half-open probe is allowed. A successful probe closes the circuit; a failing probe reopens it. A second concurrent probe can be refused with `CIRCUIT_HALF_OPEN_PROBE_IN_FLIGHT`.

`Watchdog` in default `warn` mode annotates an over-long result and still passes it. In `block` mode, it suppresses the result with `WATCHDOG_TIMEOUT`. It does not cancel the in-flight upstream call; it acts on the result path.

### Setup tips

- Test a healthy call, a failure sequence, an open-circuit refusal and a recovery probe. A single block is not enough to validate the lifecycle.
- Treat the agent stopping as observed behaviour. A different model may call again; the gateway still refuses while the circuit is open.
- Use `error_rate_threshold` and `min_calls` for busy tools where a percentage of failures is more meaningful than a consecutive count.
- In multi-replica deployments, remember that breaker state is local to each gateway process unless your deployment provides affinity or shared state.

### Considerations

The proof of shielding is the `CIRCUIT_OPEN` refusal at `tool_pre_invoke`, whose contract is before upstream invocation. If you need a measured backend-call count, instrument the backend as well. The breaker state is process-local and there may be no operational reset button; a restart clears it, and multiple replicas can learn different states. `Watchdog` limits the result that reaches the client; it does not interrupt work already executing upstream.

## RetryWithBackoffPlugin — The second request

> “A transient 503 should not drop the work, but a sustained outage must not become a hammer.”

**Example scenario.** A laboratory assistant queries an overnight diagnostic system. During an index rebuild, a request can briefly return 503 and then succeed on the next attempt. Some nights the rebuild stays down. Taking the first 503 loses useful work; retrying indefinitely adds load to the system that is already failing.

`RetryWithBackoffPlugin` runs after a tool returns a retryable status. It waits, then asks the gateway to invoke the tool again. It should be paired with `CircuitBreaker`, because each retry re-enters the plugin pipeline and is a real backend call. For how re-invocation interacts with hooks and priorities, see [Plugin pipeline](plugin-pipeline.md).

**What it proves.** In the reference environment, the baseline path returned a 503 on the first transient failure. The governed path retried and completed on attempt 2. Under sustained failure, the original call plus two retries produced three backend hits; the breaker then opened and the next logical call was refused with `CIRCUIT_OPEN` before the backend was touched.

**When to use it.** Use it for idempotent or safely repeatable reads and operations where transient 429, 500, 502, 503 or 504 responses are expected. Do not use it alone on non-idempotent writes or on a backend that is likely to stay down. A retry is a second request; it can have cost, side effects and capacity impact.

### Where it sits

| Setting | Value |
|---|---|
| Plugin | `RetryWithBackoffPlugin` |
| Hook | After Tool (`tool_post_invoke`) |
| Mode | Block (`sequential`), the Hub's attach default; the plugin re-invokes the tool rather than blocking |
| Priority | `170` (catalogue default) |
| Attached to | Virtual Server, beside `CircuitBreaker` |

### Configuration

1. Bind `CircuitBreaker` on the same Virtual Server first, with thresholds suitable for the dependency.
2. Add `RetryWithBackoffPlugin` on **After Tool**.
3. Apply the retry configuration.
4. Push to Gateway and verify both plugins are present on the pushed binding set.
5. Test a transient failure and a sustained failure separately.

Retry configuration:

```json
{
  "max_retries": 2,
  "backoff_base_ms": 200,
  "max_backoff_ms": 5000,
  "retry_on_status": [429, 500, 502, 503, 504],
  "jitter": true,
  "check_text_content": false,
  "tool_overrides": {}
}
```

Companion breaker configuration used for the short reference exercise:

```json
{ "consecutive_failure_threshold": 3, "min_calls": 5, "cooldown_seconds": 30 }
```

### What happens

A transient failure returns to the agent as a success after the retry budget is used once:

```text
baseline: failed (HTTP 503), attempt 1
governed: COMPLETED, attempt 2
```

A sustained failure spends the retry budget. With `max_retries: 2`, the backend sees up to three attempts for one logical request. The breaker counts those attempts and can open during or immediately after the failing sequence. The next call receives a policy block such as:

```text
Blocked by policy: CircuitBreaker: CIRCUIT_OPEN
```

The most reliable evidence of a retry is the retry metrics span (`retry_count`, delay) and the backend’s own hit counter. Do not assume which attempt’s payload is returned after the retry budget is exhausted; confirm on your release which attempt’s result is returned.

### Setup tips

- Verify on your release that a retry reaches the backend: check the backend's hit counter, not only the retry metrics span.
- Use idempotent tools for retries. If an operation charges a card, updates an order or sends a message, repeat safety must be proven by the upstream contract.
- Keep the breaker close to the retry in the same Virtual Server. Without it, retries can amplify an outage.
- Watch both logical requests and backend hits. The user may see one answer while the backend handled two or three calls.
- If `check_text_content` is off, a tool that returns HTTP 200 with an error in the body is not retried by status-based rules.

### Considerations

Retry improves reliability for a blink, but it spends backend capacity to do so. Each retry can run before-tool plugins again, can be counted by the breaker and can contribute to rate limits. The error-rate branch of a breaker may open earlier than a consecutive-failure threshold if the rolling window is already full of failures. For cost and incident response, count backend hits, not only agent-visible turns.

## CachedToolResultPlugin — The same question twice

> “The second identical read should not cost the system of record the same work as the first.”

**Example scenario.** A fleet of agents asks the same safe ERP read again and again: credit limit for an account, current inventory for a SKU, status for a shipment. The answer is safe to reuse for a short period, but the system of record still receives every request. The user sees the same answer, so the wasted work is easy to miss.

`CachedToolResultPlugin` stores a successful tool result after the first call and, when serving is enabled, answers a warm repeat from gateway memory before the upstream is called. It is intentionally narrow: use it only for idempotent reads.

**What it proves.** In the reference environment, the governed first call was a cold miss and stamped `cache_stored:true`. The governed second call returned byte-identical content with `cache_served:true` and `cache_hit:true`; the upstream was skipped. The baseline Virtual Server had no `cache_served` marker and ran the tool both times.

**When to use it.** Use it where many callers repeat the same read within a short `ttl`. Do not use it for writes or for reads whose result depends on who asks.

### Where it sits

| Setting | Value |
|---|---|
| Plugin | `CachedToolResultPlugin` |
| Hooks | Before Tool and After Tool (`tool_pre_invoke`, `tool_post_invoke`) |
| Mode | An awaited mode: Transform (`transform`, the Hub's attach default for this plugin) or Block (`sequential`) |
| Attached to | Virtual Server; one assignment activates the plugin on every hook it declares |

### Configuration

1. Confirm the target tool is a safe, idempotent read.
2. Enable serving in the gateway's extra environment — a ConfigMap or Secret you own, referenced from the gateway deployment values.
3. Add `CachedToolResultPlugin` to the governed Virtual Server in an awaited mode (Transform or Block).
4. Apply a binding configuration with `serve_from_cache:true` and a short `ttl`.
5. Push to Gateway and prove the warm repeat against a baseline Virtual Server.

Gateway extra environment:

```yaml
TOOL_CACHE_SERVE_ENABLED: "true"
```

Binding configuration:

```json
{
  "cacheable_tools": ["<federated-read-tool-name>"],
  "ttl": 300,
  "serve_from_cache": true
}
```

Both keys are required. `serve_from_cache:true` is the Hub binding half; `TOOL_CACHE_SERVE_ENABLED` is the gateway deployment half. If either is missing, the plugin can still observe a repeat but does not serve it from cache.

### What happens

Cold call:

```text
cache_stored: true
cache_hit: false
```

Warm repeat within `ttl`:

```text
cache_hit: true
cache_served: true
cache_store_skipped: served
```

The response body is the cached snapshot, so it can be byte-identical to the first response. If the upstream would have returned a fresh timestamp, a served response may visibly carry the earlier timestamp. That is expected; it is also why `ttl` must be small.

### Setup tips

- Prove serving with response metadata, not with a generic performance counter. Some gateway counters can increment for served calls too.
- Keep the cacheable tool list explicit. An empty list caches nothing.
- Use authentication consistently on the Virtual Server.
- Keep gateway topology in mind. The cache is in memory and process-local; restarts start cold, and multiple replicas or workers may miss unless routing is sticky.
- Serving applies to REST, MCP and A2A tools. SOAP and gRPC tool results are not served from cache.
- Verify on your release that a warm repeat carries `cache_served:true` before planning around skipped upstream calls.

### Considerations

Enable result serving only for idempotent reads where every caller uses the same upstream credential and the result does not depend on who asks. Keep the TTL short.

## ResponseCacheByPrompt — The repeated question that still ran

> “The metadata says this looks like a repeat. Did the backend actually run?”

**Example scenario.** A service-desk assistant receives the same claims or policy question several times in a shift. The wording is sometimes exact, sometimes the same words are reordered, and sometimes a user swaps in a synonym. Operators notice `approx_cache:true` in response metadata and wonder whether the plugin saved backend work.

`ResponseCacheByPrompt` is advisory. It compares configured request fields with earlier calls and annotates likely repeats. It does not short-circuit the tool call and does not serve a cached response. Use it as a signal for downstream logic, not as a backend-load reduction control by itself.

**What it proves.** In the reference environment, with a tool response that carried its own call counter, the counter advanced on every call: original wording, exact repeat, reordered words and synonym swap. The metadata changed; the backend work did not stop. The similarity logic was lexical bag-of-words cosine over whitespace tokens, not semantic understanding.

**When to use it.** Use it when a consumer will read the approximate-repeat signal and act elsewhere, for example to route, review or suppress work in a separate control. If the goal is to skip an upstream idempotent read, use `CachedToolResultPlugin` where its safety conditions are satisfied.

### Where it sits

| Setting | Value |
|---|---|
| Plugin | `ResponseCacheByPrompt` |
| Hooks | Before Tool and After Tool (`tool_pre_invoke`, `tool_post_invoke`) |
| Mode | Transform (`transform`); never blocks |
| Priority | Catalogue default |
| Attached to | Virtual Server; one assignment activates the plugin on every hook it declares |

### Configuration

1. Choose the tool and input fields that should be compared.
2. Add `ResponseCacheByPrompt` once, on either tool lane, with the JSON configuration. One assignment activates the plugin on every hook it declares; the lane you choose is where the Hub shows it. Attaching the same plugin a second time on the same Virtual Server merges into one policy.
3. Push to Gateway and verify the binding is present.
4. Test original, repeated, reordered and synonym-swapped prompts while reading both metadata and a backend-side counter or equivalent proof of execution.

```json
{
  "cacheable_tools": ["<federated-tool-name>"],
  "fields": ["query"],
  "ttl": 300,
  "threshold": 0.92
}
```

### What happens

| Call pattern | Metadata | Backend execution |
|---|---|---|
| First wording | `approx_cache:false` | Runs |
| Exact repeat | `approx_cache:true` | Runs |
| Same words reordered | `approx_cache:true` when similarity crosses the threshold | Runs |
| Genuine synonym swap | Often `approx_cache:false` because tokens changed | Runs |
| Baseline Virtual Server | No `approx_cache` key | Runs |

Some clients do not display response metadata. If the user interface hides `_meta`, capture the MCP response or trace data from a client path that exposes it.

### Setup tips

- Always pair `approx_cache` with proof that the tool did or did not run. The flag alone is not a receipt for skipped work.
- Use a unique first phrase in tests to avoid a false warm state from earlier traffic.
- Include a reordered-words case and a synonym case. That demonstrates the lexical nature of the match.

### Considerations

`ResponseCacheByPrompt` is not a cache in the serving sense. It is a metadata annotator. It can be useful in a larger design that has a downstream consumer, but by itself it does not lower backend load, reduce model cost or guarantee semantic similarity. Budget and capacity plans should not count `approx_cache:true` as saved work.
