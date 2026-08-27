# gemsctl reference

`gemsctl` is the whole command-line surface of GEMS: five commands, and a configuration file that
lets you stop typing flags. Run it from the extracted package directory.

```sh
bin/gemsctl <command> <options>          # Linux
bin\gemsctl.exe <command> <options>      # Windows
```

## Commands

### `start`

Starts the GEMS container, and opens the UI in the default browser if your platform has one.

Image resolution, in order: `gemsctl` searches the current working directory **and its parent** for a
GEMS image file and loads the highest version it finds; failing that it falls back to
`localhost/gems:latest` in the local registry. `--image` and `--image-file` override this and are
mutually exclusive.

| Flag | Default | Description |
| --- | --- | --- |
| `--config` | `.gemsconfig` in the working directory | Path to the configuration file |
| `--data-dir` | `$HOME/.gems` (Linux), `%USERPROFILE%\.gems` (Windows) | Path to the data directory |
| `--debug` | `false` | Output debug logging |
| `--no-browser` | `false` | Do not launch the default browser |
| `--image` | `localhost/gems:latest` | Container image name to create the container from. Mutually exclusive with `--image-file` |
| `--image-file` | none — search for the package image file | Container image file to load. Mutually exclusive with `--image` |
| `--port` | `7513` | Port exposed on the container for the GEMS UI |

### `stop`

Stops the GEMS container.

| Flag | Default | Description |
| --- | --- | --- |
| `--config` | `.gemsconfig` in the working directory | Path to the configuration file |
| `--data-dir` | `$HOME/.gems` (Linux), `%USERPROFILE%\.gems` (Windows) | Path to the data directory |
| `--debug` | `false` | Output debug logging |
| `--cleanup` | `false` | **Deletes the data directory, image and container** |

`--cleanup` throws away all collected Prometheus history along with everything else. It is for
reclaiming a machine, not for a routine shutdown.

### `hash-pwd`

Generates the basic-authentication password hash that secures the GEMS container port. The output
goes into `security.basicAuth.password` in the configuration file.

```sh
bin/gemsctl hash-pwd <STRING>
```

`<STRING>` is the plain password; the hash is printed to the console.

### `version`

Prints the `gemsctl` version to the console.

### `help`

Prints a brief help message.

## Configuration file

A YAML file, by default `.gemsconfig` in the current working directory, overridden with `--config`.
It has three top-level sections: `global`, `container` and `security`.

### `global`

| Name | Type | Default | Description |
| --- | --- | --- | --- |
| `dataDir` | String | `$HOME/.gems` / `%USERPROFILE%\.gems` | Path to the user data directory |
| `debug` | Boolean | `false` | Output debug logging |
| `noBrowser` | Boolean | `false` | Do not launch the default browser |
| `cleanup` | Boolean | `false` | Delete the data directory, image and container on `stop` |

Setting `cleanup: true` here makes **every** `stop` destructive. Prefer passing `--cleanup` on the
one occasion you mean it.

### `container`

| Name | Type | Default | Description |
| --- | --- | --- | --- |
| `imageName` | String | `localhost/gems:latest` | Image used to create the container |
| `imageFile` | String | `null` — search for the package image file | Image file to load |
| `port` | Integer | `7513` | Port exposed on the container for the GEMS UI |

### `security`

Two sub-sections, `basicAuth` and `tls`.

**`basicAuth`**

| Name | Type | Default | Description |
| --- | --- | --- | --- |
| `userName` | String | none | Basic authentication user name |
| `password` | String | none | Password **hash**, as produced by `hash-pwd` |

If `basicAuth` is given, `tls` must be given too.

**`tls`**

| Name | Type | Default | Description |
| --- | --- | --- | --- |
| `cert` | String | none | TLS certificate for HTTPS |
| `key` | String | none | TLS private key for HTTPS |

Only PEM-encoded certificates and keys are supported.

### Full example

Every option in one file:

```yaml
global:
  dataDir: ./gems-data
  debug: false
  noBrowser: false
  cleanup: false

container:
  imageName: localhost/gems:latest
  imageFile: null
  port: 7513

security:
  basicAuth:
    userName: user
    password: "$2a$10$xyAlfmF040VEUsUC6.MKaujCMqJCNt8yCT3C8vHen/mfBlkgV7LN."
  tls:
    cert: /path/to/cert.pem
    key: /path/to/key.pem
```

---

Source: [TIBCO GEMS 1.1.0 — gemsctl Reference](https://docs.tibco.com/pub/msg-gems/1.1.0/doc/html/gemsctl-reference).
