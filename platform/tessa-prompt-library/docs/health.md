# Health and Diagnosis

The prompts that answer "is everything all right?" — first across the whole estate, then down to a
single application.

The distinction that matters on this page: asking for a *metric* gets you a number, asking for a
*diagnosis* gets you a cause and a fix. These prompts are all phrased for the second.

Everything on this page has been run against a live environment.

---

## 9. The executive view

!!! note "Prompt"
    ```
    Give me an executive view of my environment's status.
    ```

Deliberately short and deliberately vague, and it works. TESSA decides what an executive view means:
it sweeps the environment, picks out what matters and summarises it at the level someone who is not
going to read a metrics table would want.

It is good at the discrepancy that a status column alone would hide — a data plane showing green while
a capability on it reports a service down. That distinction is exactly what you want surfaced before a
leadership update.

Good opening question when you do not yet know what you are looking for.

---

## 10. The full sweep

!!! note "Prompt"
    ```
    How healthy is my whole environment? Cover every data plane, flag critical apps and
    scan for CPU violations.
    ```

The same question with the scope spelled out, which is what you want when you need to be sure nothing
was skipped. Naming the three things explicitly — every data plane, critical apps, CPU violations —
stops TESSA settling for a partial answer.

You get a data plane table, then critical apps split into CPU violations, stopped applications and
applications with no status set, then a conclusion that says which of those is the actual risk.

---

## 11. Diagnose one application

!!! note "Prompt"
    ```
    Diagnose the health of <app-name> — score it, name the cause and recommend a fix.
    ```

The single most useful prompt in this library. Compare the two ways of asking:

| You ask | You get |
| :---- | :---- |
| "Show me memory for `<app-name>`" | A number, and the work of interpreting it |
| "Diagnose `<app-name>` — score it, name the cause, recommend a fix" | A health score out of 100, a verdict, the specific cause, and what to do about it |

The three verbs — **score**, **name the cause**, **recommend a fix** — are what produce that. Keep them.

!!! warning "A score of 100 does not always mean healthy"
    A stopped or idle application reports zero CPU, zero memory, zero requests, zero errors and zero
    response time — and scores **100/100 healthy**, because there is nothing wrong with the metrics
    that exist. TESSA says so when it happens, noting that the signal also matches a stopped or idle
    app.

    Always read the score together with the application's state. If you are diagnosing something that
    is not running, [prompt 12](#12-find-everything-that-is-not-running) is the more useful question.

---

## 12. Find everything that is not running

!!! note "Prompt"
    ```
    Which applications are not in a running state? For each, say what state it is in and
    what usually causes that state.
    ```

The second clause is what makes this worth asking. A list of stopped applications is a list; a list
that explains what each state typically means tells you which ones are deliberate and which are
incidents.

The answer includes applications whose state field is *empty* as well as those explicitly stopped —
see the next prompt for why that matters.

---

## 13. Hunt for applications with no status at all

!!! note "Prompt"
    ```
    Are any applications reporting no status at all? Explain why that is worth
    investigating.
    ```

Applications reporting *nothing* are easy to miss, because they do not appear in a list of failures.
A missing status means the platform view is incomplete, so the application's real runtime state is not
reflected in any reporting you are looking at — including the health sweep above.

If you only run one prompt from this page after a deployment, make it this one.

---

## 14. Check for resource violations

!!! note "Prompt"
    ```
    Are there any CPU violations across my apps right now?
    ```

A direct check against configured thresholds. You get the offending applications with their usage
against the threshold — for instance an application at 3.24× requested CPU against a threshold of 0.8
— or a clean answer, which is a useful thing to have on the record.

!!! tip "If TESSA says the time window is not available"
    Some scans are not offered over every window. TESSA will tell you rather than silently returning
    something else, and will suggest an alternative. See
    [Troubleshooting](troubleshooting.md#29-recover-when-tessa-says-it-cannot).

---

## Where next

Health answers tell you *that* something is wrong. To see it, go to
[Charts and Dashboards](charts.md); to work out *why*, go to [Troubleshooting](troubleshooting.md).
