# Install and run

## 1. Download and extract

Download the GEMS package for your platform from the TIBCO download site, then extract it. Everything
you need is inside the archive — there is no installer.

!!! note "Linux and Windows"
    ```sh
    # Linux
    unzip TIB_msg-gems_1.1.0_linux_x86_64.zip

    # Windows
    powershell Expand-Archive -Path TIB_msg-gems_1.1.0_win_x86_64.zip -DestinationPath .
    ```

Run every command below **from the extracted package directory**. `gemsctl` looks for the image file
relative to where it runs.

## 2. Start GEMS

!!! note "Linux and Windows"
    ```sh
    bin/gemsctl start          # Linux
    bin\gemsctl.exe start      # Windows
    ```

One command does the lot: it loads the container image, creates and starts the container, and opens
the UI in your default browser if your platform has one. On a headless host add `--no-browser` and
open `http://<host>:7513/` yourself.

![The GEMS landing page as it opens after `gemsctl start`](./images/gems-landing-page.webp)

That landing page is where every registration starts — the **Register EMS Servers** button in the
middle of it is step 4 below.

### How `gemsctl` finds the image

This is the step that surprises people, so it is worth knowing the order:

1. It searches the **current working directory and its parent** for a GEMS image file, and loads the
   one with the **highest version number**.
2. If that finds nothing, it falls back to `localhost/gems:latest` in the local registry.

Two flags override the search, and they are mutually exclusive:

- `--image-file <path>` — load this specific image file.
- `--image <name>` — use this image name from the local registry.

If you keep several GEMS versions side by side, pass `--image-file` explicitly rather than trusting
the highest-version rule to pick what you meant.

### Where things are written

| | Default |
| --- | --- |
| Data directory | `$HOME/.gems` (Linux), `%USERPROFILE%\.gems` (Windows) — override with `--data-dir` |
| UI port | `7513` — override with `--port` |
| Configuration file | `.gemsconfig` in the current working directory — override with `--config` |

The data directory holds the Prometheus time series and the component logs, so point `--data-dir` at
a volume with room to grow if you intend to keep history. It also holds
`<data-dir>/run/certs`, which is where MTLS material must be placed — see
[Register EMS servers](register-ems.md#mutual-tls-mtls).

## 3. Secure the UI before anyone else uses it

Started as above, the GEMS port is plain HTTP with no authentication. To lock it down, create a
`.gemsconfig` file with a `security` section. Two rules:

- Basic authentication requires a **password hash**, not a password.
- If you configure `basicAuth`, you **must** also configure `tls` — otherwise you would be sending
  credentials in the clear, and `gemsctl` will not let you.

Generate the hash first:

!!! note "Generate the password hash"
    ```sh
    bin/gemsctl hash-pwd 'the-password-you-want'
    ```

It prints a bcrypt hash to the console. Paste that into the configuration file:

!!! note ".gemsconfig — basic authentication and TLS"
    ```yaml
    security:
      basicAuth:
        userName: gems-admin
        password: "$2a$10$xyAlfmF040VEUsUC6.MKaujCMqJCNt8yCT3C8vHen/mfBlkgV7LN."
      tls:
        cert: /path/to/cert.pem
        key: /path/to/key.pem
    ```

Only PEM-encoded certificates and keys are supported. Restart GEMS to pick the change up, and the UI
is then served over HTTPS behind basic authentication.

## 4. Register your EMS servers

The first screen offers a **Register EMS Servers** button. That is a one-time step per server group,
driven by a YAML document you write or paste into the editor built into the page — covered in full on
[Register EMS servers](register-ems.md).

Once a group is registered and validated, **Done** returns you to the home page, which lists every
registered server.

## 5. Stop GEMS

!!! note "Linux and Windows"
    ```sh
    bin/gemsctl stop           # Linux
    bin\gemsctl.exe stop       # Windows
    ```

Stopping leaves the data directory, image and container in place, so the next `start` is fast and
your Prometheus history survives.

!!! warning "`--cleanup` is destructive"
    `bin/gemsctl stop --cleanup` deletes the data directory, the image **and** the container. That
    discards all collected metrics history. Use it to reclaim a machine, not as a normal shutdown.

## Repeatable runs

Passing the same flags every time gets old. Put them in `.gemsconfig` instead and both `start` and
`stop` will pick them up:

!!! note ".gemsconfig — repeatable runs"
    ```yaml
    global:
      dataDir: ./gems-data
      noBrowser: true
    container:
      port: 8513
    ```

Keep that file next to the extracted package, or point at it with `--config` from anywhere. The full
set of settings is in the [gemsctl reference](gemsctl.md#configuration-file).

---

Source: [TIBCO GEMS 1.1.0 documentation](https://docs.tibco.com/pub/msg-gems/1.1.0/doc/html/).
