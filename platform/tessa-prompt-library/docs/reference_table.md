# Reference Table

Every prompt in the library, in one place. Follow the link for the full prompt text and what to expect
back.

Replace `<app-name>`, `<data-plane-name>` and `<owner-a>` / `<owner-b>` with values from your own
environment — see the [placeholder convention](index.md#placeholder-convention).

Two prompts are marked ⚠️ — they could not be confirmed working in a live environment. Everything else
on this page was run and verified.

---

## Inventory and Discovery

| # | Ask | Prompt |
| :---- | :---- | :---- |
| 1 | What can TESSA do? | [help](inventory.md#1-ask-tessa-what-it-can-do) |
| 2 | What applications exist? | [List all applications … with capability, data plane and status](inventory.md#2-list-every-application) |
| 3 | What data planes exist? | [Which data planes do I have, and what is the status of each?](inventory.md#3-list-your-data-planes) |
| 4 | What is this one app? | [What is the application `<app-name>`?](inventory.md#4-identify-a-single-application) |
| 5 | What runs here? | [What applications are running on `<data-plane-name>`? Group them by capability](inventory.md#5-see-what-runs-on-one-data-plane) |
| 6 | Who is behind on versions? | [Which data planes have an older version of BW5 installed?](inventory.md#6-find-version-drift-across-data-planes) |
| 7 | Where is this connector used? | [Get all provisioned Flogo connectors … group data planes by connector-version pair](inventory.md#7-map-connectors-to-the-data-planes-that-use-them) |
| 8 | Which runtimes are stale? | [List every capability instance with its version, and flag any behind the latest](inventory.md#8-audit-capability-instance-versions) |

## Health and Diagnosis

| # | Ask | Prompt |
| :---- | :---- | :---- |
| 9 | Summarise it for a leader | [Give me an executive view of my environment's status](health.md#9-the-executive-view) |
| 10 | Check everything | [How healthy is my whole environment? Cover every data plane, flag critical apps, scan for CPU violations](health.md#10-the-full-sweep) |
| 11 | Diagnose one app | [Diagnose the health of `<app-name>` — score it, name the cause and recommend a fix](health.md#11-diagnose-one-application) |
| 12 | What is not running? | [Which applications are not in a running state?](health.md#12-find-everything-that-is-not-running) |
| 13 | What is hiding? | [Are any applications reporting no status at all?](health.md#13-hunt-for-applications-with-no-status-at-all) |
| 14 | Any threshold breaches? | [Are there any CPU violations across my apps right now?](health.md#14-check-for-resource-violations) |

## Charts and Dashboards

| # | Ask | Prompt |
| :---- | :---- | :---- |
| 15 ⚠️ | One metric, one app | [Show me CPU usage for `<app-name>` for 24H](charts.md#15-a-single-metric) |
| 16 | Two metrics, one app | [Show me CPU and memory side by side for `<app-name>` for 24H](charts.md#16-two-metrics-side-by-side) |
| 17 | Everything at once | [Show me CPU, memory, request count and execution time for all apps as a dashboard](charts.md#17-the-full-dashboard) |
| 18 | Compare runtimes | [Show me CPU and memory for all BusinessWorks 6 and Flogo apps on all dataplanes](charts.md#18-compare-runtimes-across-your-data-planes) |
| 19 ⚠️ | Request vs limit | [Compare CPU request ratio and CPU limit ratio for all apps](charts.md#19-request-ratio-against-limit-ratio) |
| 20 | Find the worst | [Which apps are using the most resources? Show me as a chart](charts.md#20-let-tessa-pick-the-subject) |
| 21 | Change the chart | [Show it as a line chart](charts.md#21-refine-the-chart-conversationally) |

## Trends and Anomalies

| # | Ask | Prompt |
| :---- | :---- | :---- |
| 22 | Did it change? | [Compare CPU usage for all apps between last week Friday and today as a before/after chart](trends-anomalies.md#22-before-and-after) |
| 23 | Any spikes? | [Any spike in CPU or memory in the last hour?](trends-anomalies.md#23-look-for-spikes) |
| 24 | Real or noise? | [Show me memory for all apps over the last 7 days. Is any change sustained, or just a blip?](trends-anomalies.md#24-sustained-change-or-blip) |
| 25 | Am I being fooled? | [Zoom out — show the same metric over 24 hours instead of 4](trends-anomalies.md#25-zoom-out-before-you-conclude-anything) |
| 26 | When did it change? | [Find when it changed and tell me what most likely caused it](trends-anomalies.md#26-find-when-it-changed) |

## Troubleshooting

| # | Ask | Prompt |
| :---- | :---- | :---- |
| 27 | Open investigation | [Why is `<app-name>` slow?](troubleshooting.md#27-the-open-question) |
| 28 | Justify the conclusion | [Walk me through the evidence, then tell me the single most likely cause](troubleshooting.md#28-evidence-first-then-the-verdict) |
| 29 | Work around a limit | [What is the closest thing you can do instead?](troubleshooting.md#29-recover-when-tessa-says-it-cannot) |

## Reporting and Governance

| # | Ask | Prompt |
| :---- | :---- | :---- |
| 30 | Governance gaps | [Show me all apps missing tags grouped by owner](reporting.md#30-find-untagged-applications-grouped-by-owner) |
| 31 | Apply my own rules | [List all applications … with Application Name, Owner and Action columns](reporting.md#31-generate-a-report-from-your-own-rules) |
| 32 | Write it up | [Summarise this whole conversation into an executive report](reporting.md#32-turn-the-session-into-an-executive-report) |

---

## Prompt patterns at a glance

| Pattern | Why it works |
| :---- | :---- |
| Stay inside platform operations | Anything else is refused outright, however well phrased |
| Ask about all apps before one app | Broad questions route around missing telemetry and find what you did not suspect |
| Discovery before placeholders | Real names beat remembered ones |
| "Score it, name the cause, recommend a fix" | Turns a reading into a diagnosis |
| Instructions, not questions | "Show it as a line chart" renders one; "is it possible to?" gets a yes |
| Always name the time window | Defaults are short, and short windows hide step changes |
| Follow up rather than restart | Context is an asset — it is what makes the final summary possible |
| State your rules inline | TESSA applies your business logic, not a dashboard's |

Full explanations on [Prompting Tips](prompting-tips.md).
