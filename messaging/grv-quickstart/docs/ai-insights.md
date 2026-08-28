# AI Insights (Tech Preview)

TIBCO Rendezvous AI Insights is an AI-driven operational agent that reads the registered Rendezvous
estate and answers questions about it. It is new in Graphical Rendezvous 1.1.0, it is optional, and
it can be enabled or disabled at any time.

!!! warning "Tech Preview"
    AI Insights is a Tech Preview feature, for **non-production and internal use only**. Features
    and APIs may change without notice. Treat what it produces as input to an operational decision,
    not as the decision.

## What it does

| Capability | What it gives you |
| --- | --- |
| **Operational report generation** | Generates operational reports over the registered estate, with analysis rather than raw metric dumps |
| **Conversational natural-language interface** | Ask about the deployment in plain language instead of assembling the query yourself |
| **Recommendation engine** | Actionable recommendations and operational guidance, drawn from TIBCO engineering practice for Rendezvous |

The agent is purpose-built for Rendezvous rather than general-purpose, which is what makes its
answers repeatable and keeps token consumption — and therefore cost — down.

## How it is fenced in

The agent is the one part of Graphical Rendezvous that talks to something outside your network, so
it is worth knowing exactly what it can reach:

- It runs in its **own dedicated container**, isolated from both the host and the `grv` container.
- All outbound traffic goes through a **forward proxy with a configurable allowlist**. Anything not
  on the list is not reachable.
- It runs as a **non-root user with a read-only filesystem**, and its tooling is limited to data
  analysis and presentation.

The default allowlist is `api.openai.com,auth.openai.com,chatgpt.com`. Everything the agent can send
anywhere is bounded by that list, so it is the first thing to review with whoever owns egress policy
on the host.

## Turning it on and off

The agent runs by default. To disable it for a single run:

!!! note "Disable the agent for one run"
    ```sh
    bin/grvctl start --disable-agent          # Linux
    bin\grvctl.exe start --disable-agent      # Windows
    ```

To disable it permanently, or to change what it may reach, use the `agent` section of `.grvconfig`:

!!! note ".grvconfig — the agent section"
    ```yaml
    agent:
      disable: false
      allowedDomains: api.openai.com,auth.openai.com,chatgpt.com
    ```

| Name | Type | Default | Description |
| --- | --- | --- | --- |
| `disable` | Boolean | `false` | Do not start the AI agent container |
| `allowedDomains` | String | `api.openai.com,auth.openai.com,chatgpt.com` | Comma-separated list of domains the forward proxy will allow |

`allowedDomains` **replaces** the default list rather than adding to it. If you narrow it, keep the
endpoints the agent actually needs in the list or it will start and then fail every request.

## Resource limits

The agent container has its own limits, separate from the main container's, in the `container`
section:

| Name | Type | Default | Description |
| --- | --- | --- | --- |
| `agentMemLimit` | String | `0` (unlimited) | Memory limit for the agent container, e.g. `512m`, `2g` |
| `agentCpuLimit` | String | `0` (unlimited) | CPU limit for the agent container, e.g. `1.0`, `2.5` |
| `agentTmpFsSize` | String | `64m` | Size of the `/tmp` tmpfs inside the agent container |

On a shared host, set `agentMemLimit` and `agentCpuLimit` before you enable the agent — the defaults
are unlimited, and analysis workloads are bursty.

---

Source: [TIBCO Graphical Rendezvous 1.1.0 documentation](https://docs.tibco.com/pub/grv/1.1.0/doc/html/).
