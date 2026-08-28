# grvctl reference

`grvctl` is the whole command-line surface of Graphical Rendezvous: five commands, and a
configuration file that lets you stop typing flags. Run it from the extracted package directory.

!!! note "Usage"
    ```sh
    bin/grvctl <command> <options>          # Linux
    bin\grvctl.exe <command> <options>      # Windows
    ```

## Commands

### `start`

Starts the Graphical Rendezvous container, and opens the UI in the default browser if your platform
has one.

Image resolution, in order: `grvctl` searches the current working directory **and its parent** for
the packaged image file (`TIB_grv_1.1.0_linux_x86_64.zst`); failing that it uses the image named in
`container.imageName`. `--image` and `--image-file` override this and are mutually exclusive.

| Flag | Default | Description |
| --- | --- | --- |
| `--config` | `.grvconfig` in the working directory | Path to the configuration file |
| `--data-dir` | `$HOME/.grv` (Linux), `%USERPROFILE%\.grv` (Windows) | Path to the data directory |
| `--debug` | `false` | Output debug logging |
| `--disable-agent` | `false` | Do not start the AI agent container |
| `--image` | none | Container image name to create the container from. Mutually exclusive with `--image-file` |
| `--image-file` | none — search for the package image file | Container image file to load. Mutually exclusive with `--image` |
| `--no-browser` | `false` | Do not launch the default browser |
| `--port` | `7580` | Port exposed on the container for the Graphical Rendezvous UI |

### `stop`

Stops the Graphical Rendezvous container.

| Flag | Default | Description |
| --- | --- | --- |
| `--cleanup` | `false` | **Removes stopped containers and deletes the run state directory** |
| `--config` | `.grvconfig` in the working directory | Path to the configuration file |
| `--data-dir` | `$HOME/.grv` (Linux), `%USERPROFILE%\.grv` (Windows) | Path to the data directory |
| `--debug` | `false` | Output debug logging |

`--cleanup` is for reclaiming a machine, not for a routine shutdown.

### `hash-pwd`

Generates the basic-authentication password hash that secures the Graphical Rendezvous container
port. The output goes into `security.basicAuth.password` in the configuration file.

!!! note "Generate a password hash"
    ```sh
    bin/grvctl hash-pwd <STRING>
    ```

`<STRING>` is the plain password; the hash is printed to the console.

### `version`

Prints the `grvctl` version to the console.

### `help`

Prints a brief help message. Also available as `-h` / `--help`.

## Configuration file

A YAML file, by default `.grvconfig` in the current working directory, overridden with `--config`.
It has five top-level sections: `global`, `container`, `security`, `agent` and `rvmon`.

### `global`

| Name | Type | Default | Description |
| --- | --- | --- | --- |
| `dataDir` | String | `$HOME/.grv` / `%USERPROFILE%\.grv` | Path to the user data directory |
| `debug` | Boolean | `false` | Output debug logging |
| `noBrowser` | Boolean | `false` | Do not launch the default browser |
| `cleanup` | Boolean | `false` | Remove stopped containers and delete the run state directory on `stop` |

Setting `cleanup: true` here makes **every** `stop` destructive. Prefer passing `--cleanup` on the
one occasion you mean it.

### `container`

| Name | Type | Default | Description |
| --- | --- | --- | --- |
| `imageName` | String | `""` | Image used to create the container |
| `imageFile` | String | `""` — search for `TIB_grv_1.1.0_linux_x86_64.zst` in the current and parent directory | Image file to load |
| `port` | Integer | `7580` | Port exposed on the container for the Graphical Rendezvous UI |
| `promPort` | Integer | `0` (not exposed) | Port for the Prometheus server. **Deprecated** |
| `rvmonPort` | Integer | `0` (not exposed) | Port for the Rendezvous monitor. **Deprecated** |
| `logLevel` | String | `info` | One of `debug`, `info`, `warn`, `error` |
| `memLimit` | String | `0` (unlimited) | Memory limit for the main container, e.g. `512m`, `2g` |
| `cpuLimit` | String | `0` (unlimited) | CPU limit for the main container, e.g. `1.0`, `2.5` |
| `agentMemLimit` | String | `0` (unlimited) | Memory limit for the agent container |
| `agentCpuLimit` | String | `0` (unlimited) | CPU limit for the agent container |
| `agentTmpFsSize` | String | `64m` | Size of the `/tmp` tmpfs in the agent container |

`promPort` and `rvmonPort` are deprecated: exposing Prometheus and `tibrvmon` directly is not the
supported way to reach them. Leave them at `0` on anything you intend to keep.

### `security`

Three sub-sections: `basicAuth` and `tls` protect the Graphical Rendezvous UI; `rv` governs how
Graphical Rendezvous verifies the **daemons** it connects to. They are independent of each other.

**`basicAuth`**

| Name | Type | Default | Description |
| --- | --- | --- | --- |
| `userName` | String | none | Basic authentication user name. Requires `tls` to be configured |
| `password` | String | none | Password **hash**, as produced by `hash-pwd` |

**`tls`**

| Name | Type | Default | Description |
| --- | --- | --- | --- |
| `cert` | String | none | TLS certificate for HTTPS. Without it the UI is served over plain HTTP |
| `key` | String | none | TLS private key for HTTPS |

**`rv`**

| Name | Type | Default | Description |
| --- | --- | --- | --- |
| `skipVerify` | Boolean | `false` | Skip verification of the daemon's certificate |
| `serverCerts` | String | system certificate pool only | Additional server certificates to trust |
| `expectedHostname` | String | none | Server Name Indication used when verifying the daemon certificate |

`skipVerify: true` turns off daemon certificate verification for **every** transport, not just the
one that is giving you trouble. Reach for `serverCerts` first and keep `skipVerify` for a lab.

### `agent`

| Name | Type | Default | Description |
| --- | --- | --- | --- |
| `disable` | Boolean | `false` | Do not start the AI agent container |
| `allowedDomains` | String | `api.openai.com,auth.openai.com,chatgpt.com` | Comma-separated domains the agent's forward proxy will allow |

See [AI Insights](ai-insights.md) for what the agent does and how it is isolated.

### `rvmon`

| Name | Type | Default | Description |
| --- | --- | --- | --- |
| `disableDynamicTransports` | Boolean | `false` | Do not create dynamic transports for discovery |

Disabling dynamic transports removes the extra client connections they create, and removes the
automatic discovery they exist to provide — you then see only what your registered transports reach.
See [dynamic transports](register-transports.md#dynamic-transports).

## Environment variables

Read by `tibrvmon` inside the container, so set them before `start`:

| Variable | Description |
| --- | --- |
| `TIBRVMON_RV_ADMIN_NAME` | Basic authentication user name used to modify a Rendezvous daemon's configuration |
| `TIBRVMON_RV_ADMIN_PASSWORD` | The plaintext password for that user |

### Full example

Every section in one file:

!!! note "Every section in one .grvconfig"
    ```yaml
    global:
      dataDir: ./grv-data
      debug: false
      noBrowser: false
      cleanup: false

    container:
      imageName: ""
      imageFile: ""
      port: 7580
      logLevel: info
      memLimit: 2g
      cpuLimit: "2.0"
      agentMemLimit: 512m
      agentCpuLimit: "1.0"
      agentTmpFsSize: 64m

    security:
      basicAuth:
        userName: grv-admin
        password: JDEkUi5wU01wMXpUOEVnWVpERGlPckZiSE9kcmRhczBCbFAkXKThc6Txlh5pFikn+5IwkJ6qauv+1kNxFZJpNOzQ/8I=
      tls:
        cert: /path/to/cert.pem
        key: /path/to/key.pem
      rv:
        skipVerify: false
        serverCerts: /path/to/daemon-ca.pem
        expectedHostname: rvd.example.com

    agent:
      disable: false
      allowedDomains: api.openai.com,auth.openai.com,chatgpt.com

    rvmon:
      disableDynamicTransports: false
    ```

## Examples

!!! note "Linux"
    ```sh
    bin/grvctl start
    bin/grvctl start --config /tmp/config.yaml
    bin/grvctl start --port 8080 --no-browser
    bin/grvctl start --data-dir /tmp/grv --image my-registry/grv:latest
    bin/grvctl start --debug
    bin/grvctl stop
    bin/grvctl stop --cleanup
    ```

!!! note "Windows"
    ```sh
    bin\grvctl.exe start --config C:\tmp\config.yaml
    bin\grvctl.exe start --data-dir C:\tmp\grv --image my-registry/grv:latest
    ```

---

Source: [TIBCO Graphical Rendezvous 1.1.0 documentation](https://docs.tibco.com/pub/grv/1.1.0/doc/html/).
