# Graphical RV Quickstart

TIBCO Graphical Rendezvous (GRV) is the monitoring, observability and management solution for TIBCO
Rendezvous. It ships as part of the Rendezvous product package, and it runs as a container that you
start with a command-line tool called `grvctl`.

!!! info "This is a companion to the product documentation"
    This guide condenses the Graphical Rendezvous 1.1.0 documentation into a path you can follow
    start to finish, and adds the decisions worth making before you point it at a production
    estate. The authoritative source is the official documentation —
    [TIBCO Graphical Rendezvous 1.1.0 documentation](https://docs.tibco.com/pub/grv/1.1.0/doc/html/).
    Where the two ever disagree, the official documentation wins.

## What you get

The `grv` container is not just the user interface. One image packages:

| Component | What it does |
| --- | --- |
| **Graphical Rendezvous user interface** | The web UI you use to explore, visualize and monitor the discovered Rendezvous estate |
| **TIBCO Rendezvous Monitor (`tibrvmon`)** | Discovers daemons through the registered transports and collects their metrics |
| **Prometheus** | Stores the collected time series — this is why the data directory needs room |
| **TIBCO-engineered MCP Server** | Exposes the estate to the AI Insights agent |
| **System services** | Supervise the above so the container keeps running |

A second, separate container runs the [AI Insights](ai-insights.md) agent when it is enabled. It is
deliberately isolated from both the host and the `grv` container, and you can turn it off entirely.

`grvctl` is the only thing you install on the host. It loads the image, creates and starts the
container, wires up the data directory, and opens the UI.

## Requirements

| | |
| --- | --- |
| **Container engine** | Podman or Docker, installed and running. `grvctl` detects what is available and **prefers Podman**. |
| **Operating system** | `grvctl` runs on Linux and Windows. The **container image is Linux-only** — on Windows you need a Linux container back end. |
| **Disk space** | Sized for Prometheus data plus component logs, in whatever you pass as the data directory. |

Nothing is installed into the system: everything lives in the extracted package directory and the
data directory (`$HOME/.grv` on Linux, `%USERPROFILE%\.grv` on Windows, by default).

## The whole path in three commands

!!! note "Extract, start, stop"
    ```sh
    unzip TIB_grv_1.1.0_linux_x86_64.zip   # 1. extract the package
    bin/grvctl start                       # 2. load the image, start GRV, open the UI
    bin/grvctl stop                        # 3. stop it again
    ```

Between steps 2 and 3 you register your Rendezvous transports once, from the UI, using a YAML
document. That registration is the only part with real decisions in it, and it is the part most
worth reading before you start — see [Register transports](register-transports.md).

## What to read next

| Page | Read it when |
| --- | --- |
| [Install and run](install.md) | Getting the container up for the first time, securing the UI before anyone else uses it, and upgrading |
| [Register transports](register-transports.md) | Pointing Graphical Rendezvous at your daemons, and working out why one will not validate |
| [AI Insights (Tech Preview)](ai-insights.md) | Deciding whether to run the operational agent, and how tightly to fence it in |
| [grvctl reference](grvctl.md) | Looking up a flag, a default, or a `.grvconfig` setting |

## Two things to know before you begin

**The UI is unauthenticated until you configure it.** `grvctl start` with no configuration exposes
the Graphical Rendezvous port with no basic authentication and no TLS. That is fine on a laptop and
not fine on a shared host — [securing the UI](install.md#3-secure-the-ui-before-anyone-else-uses-it)
takes two minutes and is worth doing before the first real registration.

**You will see transports you did not register.** Graphical Rendezvous creates *dynamic transports*
of its own to drive daemon discovery. They are hidden in the UI, but they count as client
connections and contribute to statistics — worth knowing before you go looking for the extra
connections on a daemon. See [dynamic transports](register-transports.md#dynamic-transports).

---

Source: [TIBCO Graphical Rendezvous 1.1.0 documentation](https://docs.tibco.com/pub/grv/1.1.0/doc/html/).
