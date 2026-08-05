# Troubleshooting

Prompts for when something is wrong and you need the cause rather than the symptom.

Everything on this page has been run against a live environment.

---

## 27. The open question

!!! note "Prompt"
    ```
    Why is <app-name> slow?
    ```

Short, vague, and effective. You are not telling TESSA which metric to look at, which is the point —
it decides what is relevant and correlates across whatever it can reach.

Ask this before you ask anything more specific. A narrower question presumes you already know where
the problem is, and if you are wrong about that you will get a confident answer about the wrong thing.

Note that TESSA will not accept the premise just because you asserted it. If the metrics do not show
slowness, it says it cannot confirm the application is slow and shows you what it does have. That is
the right answer, and it saves you investigating a problem that is not there.

---

## 28. Evidence first, then the verdict

!!! note "Prompt"
    ```
    <app-name> is degraded. Walk me through the evidence, then tell me the single most
    likely cause.
    ```

Asking for the evidence *before* the conclusion gives you something you can check. Asking for a
**single** most likely cause forces TESSA to commit rather than hedging across five possibilities.

You get a labelled evidence block — state, health score, and the last observed CPU, memory, requests,
errors and response time — followed by one named cause. This is the version to use when you will have
to justify the conclusion to somebody else.

It is also the prompt that catches a false premise most cleanly. Tell TESSA an application is degraded
when it is merely stopped, and it lays out the evidence, disagrees with you, and explains that a
non-running application produces no signal to support a degradation finding.

---

## 29. Recover when TESSA says it cannot

!!! note "Prompt"
    ```
    You said that isn't available through this tool. What is the closest thing you can
    do instead?
    ```

TESSA tells you when something is out of reach rather than fabricating an answer — a scan the tool does
not support over that window, a chart service that is unavailable, or a write operation blocked by the
read-only restriction. It usually suggests an alternative unprompted; this prompt asks for one
explicitly.

The reply is specific rather than a shrug: it names what it *can* still establish — current state,
health score, and the recent CPU, memory, request, error and response-time metrics — and is clear
about where that stops short of what you asked for.

The recovery is often trivial. A window that is refused will frequently work at a different length,
and a chart that will not render for one application will render for another that has telemetry.

!!! info "Read-only is a hard boundary, not a preference"
    Ask TESSA to switch to write mode and it will decline, explain that platform policy restricts it
    to read-only for safety and governance, and then pivot to what it *can* do — listing connectors,
    providing configuration templates, producing a ready-to-run manifest for you to apply yourself.

    That last part is the useful bit. Blocked from making a change, TESSA will still prepare it.

---

## Working a real incident

The prompts above compose. A sequence that works:

1. **Scope it** — `How healthy is my whole environment? Cover every data plane, flag critical apps and
   scan for CPU violations.` ([Health](health.md#10-the-full-sweep))
2. **Pick the worst** — `Diagnose the health of <app-name> — score it, name the cause and recommend a
   fix.` ([Health](health.md#11-diagnose-one-application))
3. **See it** — `Show me CPU and memory side by side for <app-name> for 24H.`
   ([Charts](charts.md#16-two-metrics-side-by-side))
4. **Widen the window** — `Zoom out — show the same metric over 24 hours instead of 4, and tell me
   whether the pattern changes.`
   ([Trends](trends-anomalies.md#25-zoom-out-before-you-conclude-anything))
5. **Timestamp the change** — `Find when it changed and tell me what most likely caused it.`
   ([Trends](trends-anomalies.md#26-find-when-it-changed))
6. **Write it up** — `Summarise this whole conversation into an executive report.`
   ([Reporting](reporting.md#32-turn-the-session-into-an-executive-report))

Run these in one conversation, not six. TESSA carries everything it learns forward, so by step 6 it has
the whole investigation to summarise.

---

## Where next

[Reporting and Governance](reporting.md) — turning what you found into something you can send to
somebody.
