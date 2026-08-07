# Trends and Anomalies

A single chart tells you what is happening now. These prompts tell you what *changed* — which is
almost always the more useful question, and the one that takes longest to answer by hand.

---

## 22. Before and after

!!! note "Prompt"
    ```
    Compare CPU usage for all apps between last week Friday and today as a before/after
    chart.
    ```

TESSA fetches both periods, overlays them and tells you how the later one differs — higher peaks,
steadier baseline, and by roughly how much.

Relative dates work: *"last week Friday and today"* is understood. Absolute windows work too, for
example `yesterday 17:00-18:00 UTC` and `yesterday 19:00-20:00 UTC`.

This is the prompt for "did the deployment help?" and it replaces a substantial amount of dashboard
work. Expect an honest answer where the data is thin — TESSA renders the comparison per data plane and
will tell you outright that one of them had no series to compare.

---

## 23. Look for spikes

!!! note "Prompt"
    ```
    Any spike in CPU or memory in the last hour?
    ```

An open question rather than a chart request, and it is what you want when you do not yet know which
application to look at. TESSA scans, charts what it finds, and names the applications with their peak
as a ratio of requested resources — and says explicitly which of CPU or memory had *no* spike, so you
are not left wondering whether it checked.

---

## 24. Sustained change or blip?

!!! note "Prompt"
    ```
    Show me memory for all apps over the last 7 days. Is any change sustained, or just
    a blip?
    ```

The second sentence is the whole point. Without it you get a seven-day chart and have to judge for
yourself; with it TESSA sorts the applications into **sustained** and **blip** for you, with the
timestamp each sustained rise began and the peak it reached.

A week of data is also what turns an observation into evidence. "This application has been above its
requested memory continuously since the 31st" is something you can put in an incident report; "memory
looks high" is not.

Scoping to *all apps* rather than one is deliberate — it is both more reliable and more useful, since
you rarely know in advance which application drifted.

---

## 25. Zoom out before you conclude anything

!!! note "Prompt"
    ```
    Zoom out — show the same metric over 24 hours instead of 4, and tell me whether the
    pattern changes.
    ```

The most valuable habit on this page, because it corrects the most common mistake.

Over one hour, a metric can look completely flat. That flatness is the trap: widen the window and the
real story can appear — a metric flat at one level, then a step up at a specific time, and flat again
ever since.

A step change is not a spike. It is a deployment or a configuration change, and it has a timestamp you
can go and investigate. A short window hides it entirely, because the whole window sits *after* the
step.

!!! warning "A flat line over a short window proves nothing"
    Before concluding that a metric is stable, widen the window at least once. 

!!! note "Needs telemetry in both windows"
    In testing, the wider window came back empty because the application was stopped and had no series
    in any of the windows tried. That is TESSA being honest rather than the prompt failing — but it
    means this works only on an application that is actually running. Confirm there is data with
    [prompt 24](#24-sustained-change-or-blip) first.

    TESSA does not expose a maximum window; ask for what you want and it will tell you if it cannot
    render it.

---

## 26. Find when it changed

!!! note "Prompt"
    ```
    Memory for <app-name> stepped up at some point today. Find when it changed and tell
    me what most likely caused it.
    ```

The natural follow-up to prompt 25. TESSA identifies the time of the step and names the likely cause —
typically a deployment or configuration change — giving you a specific timestamp to correlate against
your change log.

Notice that you are not required to know when it happened. Stating that it *did* happen is enough.

Where the metrics do not support the premise, TESSA says so rather than inventing a step change. If it
tells you the application is stopped and its metrics are all zero, believe it — that is the grounding
working, and the answer you actually needed.

---

## Where next

You know what changed and when. [Troubleshooting](troubleshooting.md) turns that into a cause.
