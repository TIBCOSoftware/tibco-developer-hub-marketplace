# TIBCO GEMS Quickstart

TIBCO GEMS (Graphical EMS) is the monitoring, observability and management solution for TIBCO
Enterprise Message Service (EMS). It ships as part of the EMS product package, and it runs as a
single container that you start with a command-line tool called `gemsctl`.

!!! info "This is a companion to the product documentation"
    This guide condenses the GEMS 1.1.0 documentation into a path you can follow start to finish,
    and adds the decisions worth making before you register a production server group. The
    authoritative source is the official documentation —
    [TIBCO GEMS 1.1.0 documentation](https://docs.tibco.com/pub/msg-gems/1.1.0/doc/html/).
    Where the two ever disagree, the official documentation wins.

## What you get

The GEMS container is not just the user interface. One image packages:

| Component | What it does |
| --- | --- |
| **GEMS user interface** | The web UI you use to browse and administer registered EMS servers |
| **Monitoring application** | Collects metrics from each registered EMS server's monitor port |
| **Prometheus** | Stores the collected time series — this is why the data directory needs room |
| **System services** | Supervise the above so the container keeps running |

`gemsctl` is the only thing you install on the host. It loads the image, creates and starts the
container, wires up the data directory, and opens the UI.

## Requirements

| | |
| --- | --- |
| **Container engine** | Podman or Docker, installed and running. `gemsctl` detects what is available and **prefers Podman**. |
| **Operating system** | `gemsctl` runs on Linux and Windows. The **container image is Linux-only** — on Windows you need a Linux container back end. |
| **Disk space** | Sized for Prometheus data plus component logs, in whatever you pass as the data directory. |

Nothing is installed into the system: everything lives in the extracted package directory and the
data directory (`$HOME/.gems` on Linux, `%USERPROFILE%\.gems` on Windows, by default).

## The whole path in three commands

```sh
unzip TIB_msg-gems_1.1.0_linux_x86_64.zip   # 1. extract the package
bin/gemsctl start                            # 2. load the image, start GEMS, open the UI
bin/gemsctl stop                             # 3. stop it again
```

Between steps 2 and 3 you register your EMS servers once, from the UI, using a YAML document. That
registration is the only part with real decisions in it, and it is the part most worth reading before
you start — see [Register EMS servers](register-ems.md).

## What to read next

| Page | Read it when |
| --- | --- |
| [Install and run](install.md) | Getting the container up for the first time, and securing the UI before anyone else uses it |
| [Register EMS servers](register-ems.md) | Connecting GEMS to a server group — plain, TLS or MTLS |
| [gemsctl reference](gemsctl.md) | Looking up a flag, a default, or a `.gemsconfig` setting |

## Two things to know before you begin

**Registration changes your EMS servers.** Registering a group creates an administrative user
account and group on each server in it. The credentials you type into the registration YAML are used
*only* for that one-time registration; everything GEMS does afterwards uses the account it created.
Plan the registration user the way you would plan any admin account change.

**The UI is unauthenticated until you configure it.** `gemsctl start` with no configuration exposes
the GEMS port with no basic authentication and no TLS. That is fine on a laptop and not fine on a
shared host — [securing the UI](install.md#3-secure-the-ui-before-anyone-else-uses-it) takes two
minutes and is worth doing before the first real registration.
