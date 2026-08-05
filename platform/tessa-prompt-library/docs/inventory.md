# Inventory and Discovery

Start here. These prompts tell you what you actually have, and they produce the real application and
data plane names that every other page asks you to substitute into a placeholder.

Everything on this page has been run against a live environment.

---

## 1. Ask TESSA what it can do

!!! note "Prompt"
    ```
    help
    ```

TESSA answers immediately with a structured summary of what it can do for you — listing and reviewing
platform inventory, inspecting BusinessWorks and Flogo environments, monitoring application health and
performance, and retrieving operational detail — and closes by suggesting a concrete next request.

Worth running as your very first message in any environment: it confirms the model connection works
and tells you which capabilities are live in the build you are on.

---

## 2. List every application

!!! note "Prompt"
    ```
    List all applications in my subscription with their capability, data plane and
    current status.
    ```

The workhorse prompt. Returns a table of every application TESSA can see, with its runtime, data
plane, status, last-modified time and a clickable application ID that takes you straight to the app in
the Control Plane.

Run this first in any session. The names it returns are what you substitute into `<app-name>`
elsewhere in this library.

---

## 3. List your data planes

!!! note "Prompt"
    ```
    Which data planes do I have, and what is the status of each?
    ```

Returns each data plane with its status, its ID, and any message attached to it — which is where you
find things like *"At least one Capability in DP is running with Errors"* on a data plane that is
otherwise green.

The names here are what you substitute into `<data-plane-name>`.

---

## 4. Identify a single application

!!! note "Prompt"
    ```
    What is the application <app-name>? Which capability does it run on, and is it
    running?
    ```

Useful when you have inherited an environment and found a name you do not recognise. TESSA tells you
the runtime it uses, the data plane it sits on and its current state.

---

## 5. See what runs on one data plane

!!! note "Prompt"
    ```
    What applications are running on <data-plane-name>? Group them by capability.
    ```

Asking for the grouping matters. Without it you get a flat list; with it you immediately see whether a
data plane is running a single runtime or a mix, which is usually the more interesting fact.

TESSA typically offers to reformat the same answer as a full app-by-app list with IDs — take it up on
that if you need the IDs.

---

## 6. Find version drift across data planes

!!! note "Prompt"
    ```
    Which data planes have an older version of BW5 installed?
    ```

One of TESSA's own suggested prompts. It compares the installed version against the latest available
on every data plane and tells you directly whether anything is behind, rather than making you compare
version numbers by hand.

Swap `BW5` for another capability to ask the same question about a different runtime.

---

## 7. Map connectors to the data planes that use them

!!! note "Prompt"
    ```
    Get all provisioned Flogo connectors. For each unique connector-version pair, group
    together all data planes that use it.
    ```

Also one of TESSA's own suggestions, and a good demonstration of multi-step analysis: it retrieves the
connector inventory, deduplicates by connector *and* version, then inverts the relationship to group
data planes underneath.

This is the prompt to run before a connector upgrade — it tells you the blast radius. It also surfaces
the case that matters most: the same connector provisioned at two different versions on different data
planes.

---

## 8. Audit capability instance versions

!!! note "Prompt"
    ```
    List every capability instance with its version, and flag any that are behind the
    latest.
    ```

Broader than prompt 6: covers every runtime rather than one. You get a row per capability instance with
its installed version, the latest available, and an explicit behind-latest column.

Asking it to *flag* what is behind, rather than just listing versions, is what turns a table into
something actionable.

---

## Where next

Now that you have real names, go to [Health and Diagnosis](health.md) to find out whether any of it
needs your attention.
