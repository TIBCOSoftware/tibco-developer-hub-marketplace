# Prompting Tips

The patterns underneath every prompt in this library, and the boundaries worth knowing before you write
your own. Everything here comes from running the prompts against a live environment.

---

## Stay inside what TESSA is for

TESSA answers questions about **platform operations** — inventory, runtimes, health, metrics. Ask it
something outside that and it declines with a fixed message:

> I can only answer questions related to our business operations. Please ask about topics I can help
> with using the available tools and data sources.

This is not a phrasing problem you can work around, and the refusal is quick and cheap. Two kinds of
question that look reasonable but get refused:

| Refused | Ask instead |
| :---- | :---- |
| Explaining a Control Plane UI element — what a column means, how to read a page | Consult the product documentation |
| Open interpretation of your estate — what industry it suggests, which vendor an app belongs to, general commentary | Ask for the data and draw the conclusion yourself |

The dividing line is whether the answer comes from a tool. "Which apps are stopped" comes from a tool.
"What does this suggest about my business" does not.

---

## Discovery first

Do not guess at names. Run
[`List all applications in my subscription…`](inventory.md#2-list-every-application) at the start of a
session and use what comes back. A prompt built on a name you half-remember fails in the least helpful
way possible — TESSA tells you it cannot find it, and you are no wiser about what does exist.

---

## Ask about all apps before you ask about one

The most useful thing testing revealed. Prompts scoped to **all apps** succeeded consistently; prompts
naming a single application often came back with *"no time series was available for that app in that
window"*.

| Weaker | Stronger |
| :---- | :---- |
| "Show me CPU for `<app-name>`" | "Which apps are using the most resources? Show me as a chart" |
| "Is `<app-name>` unhealthy?" | "Which applications are not in a running state?" |
| "Show me memory for `<app-name>` over 7 days" | "Show me memory for all apps over the last 7 days" |

Two reasons this wins. Practically, TESSA only charts applications that have telemetry, so a broad
question routes around missing data instead of hitting it. Analytically, naming an application means
investigating one you already suspected, while asking TESSA to rank or filter finds the one you did not
know about — which is the whole reason to ask an agent rather than open a dashboard.

Use a specific application when you genuinely have one in mind, and expect to need a window where it
was actually running.

---

## Ask for the diagnosis, not the metric

The highest-leverage change you can make to any prompt. Three verbs do most of the work:

- **score it** — forces a judgement rather than a reading
- **name the cause** — forces an explanation
- **recommend a fix** — forces an action

`Diagnose <app-name> — score it, name the cause and recommend a fix` returns a health score, a verdict,
a cause and a recommendation. `Show me memory for <app-name>` returns a number and leaves the analysis
to you.

!!! warning "Read the score with the state"
    A stopped application scores **100/100 healthy** — zero CPU, zero memory, zero errors, nothing
    wrong with the metrics that exist. A perfect score can mean "nothing is happening" rather than
    "everything is fine".

---

## Give instructions, not questions

*"Show it as a line chart"* renders a line chart. *"Is it possible to plot a line chart?"* gets you
**"Yes — I can plot line charts."** and nothing else, because that is the question you asked.

TESSA answers questions literally and executes instructions. When you want work done, phrase it as
work.

---

## Always state the time window

`for 24H`, `for the last week`, `yesterday 17:00-18:00 UTC`. A prompt with no window gets a default you
did not choose — and defaults are short, which is exactly the case where
[a flat line hides a step change](trends-anomalies.md#25-zoom-out-before-you-conclude-anything).

Relative dates work: *"last week Friday and today"* is understood. There is no documented maximum
window — TESSA says the exact limit is not exposed by its tools — so ask for what you want, and it will
tell you if it cannot render it.

---

## Follow up, do not restart

TESSA carries context across the whole conversation. Once you have a chart, *"show it as a line chart"*
is a complete instruction — no need to repeat the application, the metric or the window.

This matters more than it sounds. A long conversation is not a cost, it is an asset: by the end of one,
TESSA can
[summarise the entire investigation](reporting.md#32-turn-the-session-into-an-executive-report)
because it still has all of it. Six separate conversations cannot do that — and the summary is only as
good as what the conversation actually covered.

---

## Spell out the scope when completeness matters

For a quick look, vague is fine — *"give me an executive view of my environment's status"* works
because TESSA decides what matters.

When you need to be certain nothing was skipped, list the parts: *"cover every data plane, flag
critical apps and scan for CPU violations"*. Naming the three things stops TESSA settling for a partial
answer.

---

## Do not expect it to accept your premise

Tell TESSA an application is slow, or degraded, or that its memory stepped up, and it checks before it
agrees. Where the metrics do not support what you said, it tells you it cannot confirm it and shows you
what it does have.

Treat that as the feature it is. An assistant that agreed with every premise would send you
investigating problems that do not exist.

---

## Supply your own rules

TESSA applies business logic you define in the prompt itself:

```
For the Action column:
- If the application belongs to Flogo, assign Alice.
- If the application belongs to BusinessWorks, assign Bob.
```

Anything expressible as an if/then rule can be applied across the whole inventory. Use real names —
placeholders left in the prompt are taken literally and end up in the output.

---

## Turn on Show TESSA Thoughts while you are learning

In **Settings → TESSA → LLM**, enable **Show TESSA Thoughts**. It reveals reasoning summaries, which
tools were invoked, execution timing per pipeline stage and other metadata, with secrets redacted.

When a prompt does not do what you expected, this tells you whether TESSA misunderstood the question,
was refused by the scope guard, or simply could not reach the data — three different problems with
three different fixes. Turn it off once you are comfortable.

---

## Know the boundaries

**TESSA is read-only.** It observes, inspects and explains; it cannot change your platform. Ask it to,
and it will say so plainly, explain that platform policy restricts it for safety and governance, and
then offer what it can do instead — including preparing the change for you to apply yourself.

**TESSA is grounded.** Every answer comes from real tool output. When a tool cannot answer, TESSA tells
you and suggests the nearest thing it can do. It does not fill the gap with a guess.

**Tools can be disabled.** If a prompt from this library fails and the phrasing looks right, check
**Settings → TESSA → MCP Servers** — the tool it needs may be switched off in your subscription.

---

## Verify before you act

TESSA is an excellent first responder. It is not a substitute for your own judgement on a production
change. Check anything you are about to act on — which is easy, because everything it tells you is
traceable back to real platform data.
