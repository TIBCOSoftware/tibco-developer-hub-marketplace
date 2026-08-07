# Charts and Dashboards

TESSA renders interactive charts inline in the conversation. Ask for a metric and you get a real
chart — zoom, pan, hover for values — not a screenshot. Charts persist in the conversation and survive
a page reload.

Available chart types are **line**, **bar**, **area** and **scatter**, and TESSA can put several
metrics side by side in one dashboard.

!!! tip "Ask about all apps before you ask about one"
    Chart prompts scoped to *all apps* succeed far more reliably than prompts naming a single
    application, because a chart needs a time series that actually exists in the window you asked for
    — and a stopped or idle application often has none.

    Start broad, see which applications have telemetry, then narrow. If TESSA says
    *"no time series was available for that app in that window"*, that is what has happened.

!!! tip "Always name a time window"
    Say it plainly — `for 24H`, `for the last week`, `for the last 6 hours`. A prompt without a window
    gets a default you did not choose, and
    [that default is where the trap is](trends-anomalies.md#25-zoom-out-before-you-conclude-anything).

---

## 15. A single metric

!!! note "Prompt"
    ```
    Show me CPU usage for <app-name> for 24H.
    ```

The simplest chart prompt. Swap `CPU usage` for `memory`, `request count` or `execution time`, and the
window for whatever period you need.

---

## 16. Two metrics side by side

!!! note "Prompt"
    ```
    Show me CPU and memory side by side for <app-name> for 24H.
    ```

CPU and memory in one view, which is almost always what you want — the interesting cases are where one
is healthy and the other is not.

Verified working. TESSA renders the pair and reads them for you: a short CPU spike against steady
memory, for instance, with the peak values quoted as a ratio of what the application requested.

Unlike prompt 15, this one succeeded against a named application — so if you want a chart of one
application, this is the phrasing to try first.

---

## 17. The full dashboard

!!! note "Prompt"
    ```
    Show me CPU, memory, request count and execution time for all apps as a dashboard
    for the last week.
    ```

A four-panel dashboard covering all the key metrics, rendered per data plane. TESSA fetches every
series in a single pass rather than one chart at a time, so this is faster than asking four separate
questions as well as being easier to read.

It also tells you where coverage is thin — which data planes returned every metric and which returned
only some. That is useful in itself: a data plane where only `request_count` renders has an
observability problem, not a performance one.

---

## 18. Compare runtimes across your data planes

!!! note "Prompt"
    ```
    Show me CPU and memory usage for all BusinessWorks 6 and Flogo apps on all
    dataplanes for the last week as a side-by-side chart.
    ```

Plots every application of both runtimes on shared axes, colour-coded per application. This is how you
spot that one application behaves nothing like its neighbours — a couple of heavy memory users rising
towards or past 1.0× their requested memory while everything else sits near the baseline.

Naming both runtimes explicitly keeps the comparison open. If you name only one, TESSA filters to it
and you lose the contrast that made the question worth asking.

TESSA will tell you if some applications were excluded because their IDs could not be resolved — worth
reading, so you know the chart is not the whole estate.

---

## 19. Request ratio against limit ratio

!!! note "Prompt"
    ```
    Compare CPU request ratio and CPU limit ratio for all apps.
    ```

Two Kubernetes resource views side by side: usage against what the pod *requested*, and usage against
what it is *allowed*. A large gap between them means limits are generous relative to requests — useful
when you are sizing workloads or hunting for a capacity problem.

---

## 20. Let TESSA pick the subject

!!! note "Prompt"
    ```
    Which apps are using the most resources? Show me as a chart.
    ```

No placeholder, and one of the strongest prompts in the library. TESSA finds the applications worth
charting across every runtime and data plane, plots them, and gives you a ranked table alongside —
application, capability, data plane and peak usage as a ratio of what was requested.

Prefer this shape whenever you can. Naming an application means charting one you already suspected;
letting TESSA rank them means finding one you did not. It also sidesteps the missing-time-series
problem entirely, because TESSA only charts applications that have data.

---

## 21. Refine the chart conversationally

TESSA keeps the context of the conversation, so a follow-up does not need to repeat the application,
the metric or the window. A realistic sequence:

!!! note "Prompt 1"
    ```
    Can you draw a chart of hour-on-hour CPU usage for all BusinessWorks 6 apps?
    ```

!!! note "Prompt 2"
    ```
    Show it as a line chart.
    ```

!!! note "Prompt 3"
    ```
    Show it as an area chart instead.
    ```

Each follow-up re-renders the same data as a different chart type, and TESSA re-reads the new chart
for you. Five words — *"show it as a line chart"* — is a complete instruction.

!!! warning "Phrase follow-ups as instructions, not questions"
    Ask *"is it possible to plot a line chart?"* and TESSA answers the question you asked:
    **"Yes — I can plot line charts."** No chart. You then have to say *"ok do it"* to get one.

    *"Show it as a line chart"* renders it immediately. This is worth internalising for every
    follow-up, not just charts: a polite question gets a polite answer, an instruction gets the work.

That aside, the habit is the point: **refine, do not restart**. Starting a fresh prompt with all the
detail repeated throws away context TESSA already has, and is slower.

---

## Where next

To compare two periods rather than look at one, go to [Trends and Anomalies](trends-anomalies.md).
