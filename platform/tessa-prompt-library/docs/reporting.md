# Reporting and Governance

TESSA will apply rules you supply and turn a working session into a document you can send to somebody.

Everything on this page has been run against a live environment.

---

## 30. Find untagged applications, grouped by owner

!!! note "Prompt"
    ```
    Show me all apps missing tags grouped by owner, with a one-line summary per owner
    listing their untagged apps.
    ```

A four-step analysis from a single sentence: retrieve every application, filter for missing tags, group
by owner, then summarise. TESSA tells you which field it used as the owner — typically `created-by` —
so you know what the grouping actually means.

The result is a table you can act on directly, each owner beside exactly what they need to fix, plus a
note on who has the largest backlog. That is a route to a human, which a flat list of untagged
applications is not.

This is the governance prompt to run monthly.

---

## 31. Generate a report from your own rules

!!! note "Prompt"
    ```
    List all applications in the system and present them in a table with the following
    columns: Application Name, Owner, and Action.

    For the Action column:
    - If the application belongs to Flogo, assign <owner-a>.
    - If the application belongs to BusinessWorks, assign <owner-b>.
    ```

TESSA combines live platform data with business logic you define inline. You are not limited to the
rules somebody built into a dashboard — you state them in the prompt and TESSA applies them across the
whole inventory, then tells you how it interpreted them.

Extend the pattern freely. Route by data plane rather than runtime, add a priority column driven by
application status, or assign a review date. Anything you can express as an if/then rule in a sentence,
TESSA can apply.

!!! tip "Substitute real names"
    Replace `<owner-a>` and `<owner-b>` with actual people or teams. TESSA takes the placeholders
    literally and will happily fill a column with `<owner-a>` if you leave them in.

---

## 32. Turn the session into an executive report

!!! note "Prompt"
    ```
    Summarise this whole conversation into an executive report: which data plane needs
    the most attention, the evidence for that, and prioritised next steps.
    ```

Run this at the end of an investigation, with no new query. TESSA reasons over the entire session —
every chart, every diagnosis, everything it learned along the way — and produces an overall finding,
the evidence behind it, what it suggests, and numbered next steps.

It will not manufacture a verdict the conversation does not support. Asked which data plane needs the
most attention after a session that never went down to data plane metrics, it says so plainly and
redirects to what the evidence *does* show. If you want a data plane named, make sure the conversation
covered data planes first — [the full sweep](health.md#10-the-full-sweep) is the way to do that.

!!! tip "Export it as a PDF"
    The conversation exports as a clean, co-branded PDF report — ready for an incident review or a
    leadership update, without anyone rebuilding the analysis in slides.

---

## Where next

[Prompting Tips](prompting-tips.md) — the patterns underneath all of these, so you can write your own.
