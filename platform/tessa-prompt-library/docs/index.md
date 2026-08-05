# TESSA Prompt Library

**TESSA** — the TIBCO Enterprise Smart System Agent — is an AI Ops agent for the TIBCO Platform that
you talk to in plain language. It works out what you are asking, calls the right governed tools,
analyses the results and hands back an answer. No dashboards to build, no queries to write.

This library exists because the hardest part of using TESSA is the empty prompt box. It gives you a
curated set of prompts that are known to work, organised by what you are trying to find out.

!!! warning "TESSA is a preview capability"
    TESSA is currently in preview on the TIBCO Platform, and the interface is badged **PREVIEW**
    throughout. Capabilities, tool coverage and phrasing preferences change between releases, so
    treat these prompts as a strong starting point rather than a fixed contract. If a prompt does not
    behave as described, ask TESSA `help` to see what it can currently do.

---

## What the prompt library gives you

| Benefit | Description |
| :---- | :---- |
| **Know what to ask** | TESSA reaches across your whole platform. These prompts show you the shape of question it answers well, so you are not guessing at its edges |
| **Get diagnoses, not dumps** | The difference between "show me memory for this app" and "diagnose this app" is enormous. The prompts here are phrased to get the analysis, not the raw numbers |
| **Learn the follow-up habit** | TESSA keeps context across a conversation. Several prompts here are deliberately written as chains, because the second question is where the value is |
| **Skip the traps** | A flat metric over one hour hides a step change over six. Prompts here encode the lessons that cost other people an afternoon |
| **Standardise across the team** | A shared vocabulary for health checks, incident reviews and capacity questions, so two engineers asking "is it healthy?" get comparable answers |

---

## How TESSA works

Every question runs through a guarded pipeline:

1. A **capability mapper** works out what can be answered and which tools belong together.
2. The **agent** plans the work, picks its own tools and calls them.
3. When an answer needs an exact calculation, TESSA **writes and runs code** rather than estimating.
4. A **response guard** checks the result is safe and grounded before you see it.

Its abilities come from **MCP servers** across the platform — Control Plane, BusinessWorks, Flogo and
Observability. It can also learn packaged **skills**: bundles of know-how for working the platform,
which you can extend or replace with your own.

!!! info "Read-only by default"
    TESSA observes, inspects and explains — it does not change your platform. Ask it to modify
    something and it will tell you it cannot, then offer what it *can* do instead. That is why it is
    safe to point at production.

!!! warning "It answers platform operations questions, and only those"
    TESSA is scoped to inventory, runtimes, health and metrics. Ask it to explain a Control Plane
    screen, or to offer general commentary on what your estate implies, and it declines — politely,
    consistently, and regardless of how you phrase it. The test is whether a tool can answer the
    question. See [Prompting Tips](prompting-tips.md#stay-inside-what-tessa-is-for).

---

## Using these prompts

Open a page, copy the prompt text, paste it into TESSA and send. Prompts are written to be copied
verbatim, with one exception: **placeholders**.

### Placeholder convention

Prompts that need to name something in *your* environment use angle brackets. Replace the whole
placeholder, brackets included.

| Placeholder | Replace with | Example |
| :---- | :---- | :---- |
| `<app-name>` | An application in your subscription | `order-service` |
| `<data-plane-name>` | One of your data planes | `dp-production-eu` |
| `<owner-a>`, `<owner-b>` | Names of people or teams | `Platform Team` |

Placeholders are taken literally — leave one in and TESSA will happily fill a whole table column with
`<owner-a>`.

### Discovery first

Do not guess at names. Start every session with a discovery prompt from
[Inventory and Discovery](inventory.md) to get the real application and data plane names out of your
own environment, then substitute them into the prompts that need them.

Better still, most prompts here are scoped to *all* applications rather than one. That is deliberate:
a broad question routes around applications that have no telemetry, and *"which apps are using the most
resources?"* finds the one you did not suspect, which naming an app yourself never will.

### These prompts have been tested

Every prompt in this library was run against a live TIBCO Platform environment. Two chart prompts could
not be confirmed working and are marked with a **Not yet verified** admonition explaining what happened
— everything else did what its page says it does.

---

## Where to start

| Page | Use it when |
| :---- | :---- |
| [Set Up TESSA](setup.md) | TESSA is not activated yet, or you need to configure a model and API key |
| [Inventory and Discovery](inventory.md) | Finding out what you actually have — apps, data planes, versions, connectors |
| [Health and Diagnosis](health.md) | Answering "is everything all right?" across the estate, or diagnosing one app |
| [Charts and Dashboards](charts.md) | You want to see the metric, not read about it |
| [Trends and Anomalies](trends-anomalies.md) | Comparing time windows, hunting spikes, separating a blip from a real change |
| [Troubleshooting](troubleshooting.md) | Something is wrong and you need the cause, not the symptom |
| [Reporting and Governance](reporting.md) | Untagged apps, ownership, custom reports, executive summaries |
| [Prompting Tips](prompting-tips.md) | Your prompts are not landing and you want to know why |
| [Reference Table](reference_table.md) | You know what you want and just need the prompt |

---

## Notes

- TESSA answers from real tool output. If it cannot reach something, it says so — it does not invent
  an answer.
- Answers reflect the moment you asked. Re-running the same prompt an hour later is expected to give
  a different result.
- As with any AI assistant, verify anything you are about to act on. TESSA is a very good first
  responder, not a substitute for your own judgement on a production change.
