---
name: fpga-run
description: Run a compiled ELF or binary on a lab FPGA through the FPGA Run Queue (FRQ) web service. Collect the UART output and the results. Use this skill when the user asks to run, test, debug, or benchmark firmware on an FPGA. Also use it to program an FPGA bitstream or to examine FRQ runs, queues, or logs.
---

## What FRQ is

FRQ (FPGA Run Queue) is a web service on a lab host. The host connects to one or
more FPGA boards through JTAG. FRQ lets many users share these boards.

FRQ operates in this sequence:

1. A user uploads a firmware image to FRQ.
2. The user submits a *run* for an FPGA target.
3. FRQ puts the run in a queue. FRQ starts the run when the target is free.
   Only one run uses a target at a time.
4. The run executes a *pipeline*. A pipeline is a list of steps on the
   bitstream that is on the FPGA. Examples of steps:
   - `vio_set`: set a VIO probe, for example a reset probe or a mode probe.
   - `load_elf` or `load_bin`: write the image through an AXI port.
   - `mmio_write32`: write a 32-bit value to an address.
   - `sleep`: wait for a specified time.
5. During the run, FRQ records the output of the selected UART ports. FRQ can
   also record a trace or an ILA.
6. The run stops when its timeout is complete or when a user stops it.

To program a bitstream is a separate control action. It is not part of a run.

| FRQ interface | Path |
| --- | --- |
| Queue, history, and FPGA control | `/ui` |
| Run submit form | `/ui/submit` |
| Activity log | `/ui/activity` |
| Interactive API reference | `/docs` |
| API schema | `/openapi.json` |

If this guide and `/openapi.json` do not agree, use `/openapi.json`.

All host names, targets, pipelines, file paths, and UART names in this guide
are examples. Read the real values from the selected server.

## Select the FRQ server

Each FRQ server controls its own FPGAs. Do not guess the server.

1. Get the server name (host name or IP address) from the user. If the user
   did not give it, ask before you read or submit data.
2. Do not use a server from an earlier conversation or from an example in this
   guide.
3. Set `FRQ_BASE=http://<server>:8000`.
4. Make sure that the server answers:

   ```sh
   curl --fail --silent --max-time 10 "$FRQ_BASE/api/fpga"
   ```

5. If the command gives a timeout or "connection refused", open an SSH tunnel:

   ```sh
   ssh -N -L 8000:localhost:8000 <ssh_user>@<server>   # keep running; use run_in_background
   ```

   - If you do not know `<ssh_user>`, ask the user.
   - If local port 8000 is in use, use a different local port, for example
     `-L 8001:localhost:8000`.
   - Set `FRQ_BASE=http://localhost:<local_port>`.
6. If you use a tunnel, read `GET /api/fpga` through it. Make sure that the
   tunnel goes to the correct server. If the target is not what you expect,
   ask the user.

All examples below use `FRQ_BASE`.

## Examine the server state

Before you submit a run, read these endpoints. Old run settings and earlier
conversations do not show the current state.

| Endpoint | Data to examine |
| --- | --- |
| `GET /healthz` | `worker_alive`, `fpga_connected`, `unhealthy`, and `external_lock` for each worker. |
| `GET /api/fpgas` | One entry for each FPGA: `target`, `device`, `programmed`, `bitfile`, `probefile`, `busy`, `active_run`, `queued`, `connected`. |
| `GET /api/fpga?target=<target>` | The status of one target: `axi_ports`, `vio_probes`, `ilas`, `jtag_frequency`, `control_action`, `program_progress`. Without `target`, it shows the default worker. |
| `GET /api/queue` | The active run and the queued runs. |
| `GET /api/bitstreams` | The uploaded bitstreams (`bit_path`, `ltx_path`). |
| `GET /api/designs`, `GET /api/pipelines` | The built-in design pipelines and the saved pipelines. |
| `GET /api/uart/ports` | The serial devices on the server. |

Stop and tell the user if one of these conditions occurs. Do not try to repair
the server.

- `unhealthy` is set.
- A worker is not alive.
- An FPGA is not connected.
- `external_lock` is set.

### Get approval for shared actions

The FPGAs are shared. Get approval from the user before you do one of these
actions:

- Program a bitstream.
- Change the JTAG frequency (`POST /api/fpga/frequency`).
- Change the device selection (`POST /api/fpga/select`).
- Stop a run (`POST /api/runs/<id>/kill`).
- Delete a run or a bitstream.

If a different user has a run on the target (`busy`, `active_run`, or queued
runs), tell the user before you program it. A new bitstream replaces the design
under the run of the other user.

## Select the design and the pipeline

The pipeline must agree with the bitstream on the FPGA. VIO probe names, AXI
ports, and load addresses are different for each design. The pipeline name does
not tell you which bitstream is on the FPGA.

1. If the user gives a pipeline name, use that pipeline.
2. If the user does not give a name, use a saved pipeline
   (`GET /api/pipelines`) or a design (`GET /api/designs`). Get confirmation
   from the user that it is correct for the bitstream on the FPGA.
3. Read the `vio_probes` and `axi_ports` lists of the target from
   `GET /api/fpga?target=<target>`. `GET /api/fpga/probes?target=<target>` also
   gives the probe widths.
4. Make sure that each `vio_set` probe in `steps`, `reset_assert_steps`, and
   `reset_release_steps` is in `vio_probes`.
5. Make sure that each `axi_port` in the pipeline is in `axi_ports`.
6. If a probe or a port is not in the list, the pipeline is for a different
   design. Stop and ask the user.

To save a pipeline:

- Send `POST /api/pipelines` with `name` and `steps`. You can also send
  `description`, `params`, `reset_assert_steps`, and `reset_release_steps`.
- Ask the user before you create, change, or delete a saved pipeline. All users
  can see and use saved pipelines.

## Program the FPGA

If `GET /api/fpgas` shows the target as `programmed` with the correct
`bitfile`, skip this section.

1. If the bitstream is not on the server, upload it with
   `POST /api/bitstreams` (multipart field `file`).
2. Read `GET /api/bitstreams`. Record the `bit_path` and the `ltx_path`.
3. Program the target. This is the same request that the home page sends:

   ```sh
   curl --fail --silent --show-error --max-time 3900 \
     -H 'Content-Type: application/json' \
     -d '{"bitfile": "<bit_path>", "probefile": "<ltx_path>", "target": "<target>"}' \
     "$FRQ_BASE/api/fpga/program"
   ```

4. The program operation can take many minutes. If your client has a short
   timeout, run the request in the background.
5. Monitor `program_progress` in `GET /api/fpga?target=<target>`.
6. When the operation is complete, make sure that:
   - `program_progress.state` is `done`.
   - `program_progress.error` is empty.
   - `programmed` is `true`.
   - `bitfile` is the selected bitstream.
7. Read `GET /api/activity?target=<target>`. Make sure that the `fpga/program`
   action shows `success`, not `failed`.

> [!CAUTION]
> Do not send a second program request while `control_action` has a value.
> A client timeout does not stop the program operation on the server. Read the
> status again before you send the request again.

Notes:

- Submit runs with `program=false`. The run then uses the design on the FPGA.
- With `program=true`, the run programs the FPGA again. This procedure is
  slower and different. Use it only when the user asks for it.
- A program operation resets all the state in the design. This includes
  modeled memories, for example OTP or ROM. A pipeline reset does not reset
  this state. If the test needs state that a program operation erases or that
  a reset keeps, tell the user.

## Find the UART ports

The UART assignments change with the server, the target, and the cables. A
`/dev/ttyUSBn` name can have a different function on a different server. It can
also change after you disconnect and connect a cable.

- Do not use the device number to find the function of a port.
- Do not use a port map from a different server.
- `GET /api/uart/ports` also shows devices that are not UARTs of the target,
  for example JTAG adapters. Use `description` and `hwid` to remove these
  devices from the list.

Find the UARTs in this sequence:

1. **The user gives the ports.** If the user gives the console port and the
   download port for this server, use them.
2. **A saved port map exists.** Look for a map for this server, for example
   `.frq/uarts.json` (see step 4). The `uarts` list of a past run in
   `GET /api/runs` also shows a map. Make sure that the map agrees with the
   devices that the server shows now.
3. **No map exists.** Record all the devices in one run:
   1. Get the device list from `GET /api/uart/ports`.
   2. Submit a run with all candidate devices in `uarts`. Use the device name
      as the label (`ttyUSB0`, `ttyUSB2`, ...).
   3. After the run, read the log of each port.
   4. The port that shows the boot banner or the console output is the
      console port.
   5. The port that receives a download handshake is the download port. For
      example, an XMODEM receiver sends repeated `C` characters.
   6. Tell the user the result.
4. **Save the result.** Write the correct map. Then the next runs record only
   the necessary ports:

   ```json
   {"server": "<server>", "target": "<target>", "console": "/dev/ttyUSBn", "download": "/dev/ttyUSBm", "baud": 115200}
   ```

   - Ask the user before you write the file to the repository.
   - If the setup changes, delete the map and find the ports again. Examples:
     a new bitstream, a different board, or cables that you connect again.

The UART index is the position of the port in the `uarts` list of the run.

- If a run records one port, its UART index is `0`.
- If a run records more ports, record the index of each port in the sequence of
  the list.

## Submit a run

### Upload the image and set its path in the pipeline

The FRQ server cannot read files on the development host. Upload each image
first:

```sh
curl --fail --silent --show-error --max-time 120 \
  -F "file=@build/app.elf" "$FRQ_BASE/api/uploads"
```

The server accepts `.elf`, `.bin`, `.cfg`, and `.tcl` files. The response gives
the server path of the file:

```json
{"path": "<server_path>/app.elf", "sha256": "...", "size": 123456}
```

Use this `path` in the run:

1. Find the `load_elf` and `load_bin` steps in the pipeline.
2. Give one image for each of these steps, in the sequence of the steps.
3. Set the `path` of each step to the server path of its image.
4. Set the `elf` field of the run to the path of the first image.

Example: a pipeline with one `load_elf` step:

```json
{"op": "load_elf", "path": "<server_path>/app.elf", "axi_port": "<axi_port>"}
```

The FRQ submit page (`/ui/submit`) does the same steps: it shows one upload
field for each load step.

### Set the user name of the run

Submit each run with the login name of the current user. FRQ shows this name
in the queue and on the run page ("Submitted by"). Other users use it to know
who has the target.

1. Get the login name of the current user on the development host:
   - Linux or macOS: `whoami`
   - Windows PowerShell: `$env:USERNAME`
2. Set the `user` field of the run to this name. The name must have 1 to 64
   characters.
3. Do not use a different name, a shared name, or a name such as `agent`.
4. If the login name is not the lab identity of the user (for example, a
   shared build account), ask the user for the correct name.

The script below uses the login name when you do not set `user`.

### Submit the run with the script

1. Build the images.
2. Edit the settings at the top of the script below.
3. Run the script. The script needs only Python 3 and `curl`.

The script does these steps:

1. It sets `user` to the login name of the current user, if `user` is empty.
2. It makes sure that the target is connected and has the expected bitstream.
3. It finds the pipeline. It makes sure that the probes and ports of the
   pipeline are on the target.
4. It makes sure that there is one image for each load step.
5. It uploads each image and sets the server path in its load step.
6. It copies the bitstream fields from the live state of the target.
7. It submits one run. It shows the run ID and the UI link.

```sh
python3 - <<'PY'
import getpass
import json
import subprocess
from pathlib import Path
from urllib.parse import quote
from urllib.request import Request, urlopen

base = ""         # FRQ_BASE chosen by the user, e.g. http://<server>:8000
user = ""         # Empty: login name of the current user. See "Set the user name of the run".
target = ""       # A target from GET /api/fpgas.
expected_bitfile = ""  # The bitstream the pipeline was written for.
images = []       # Local images, one per load_elf/load_bin step, in step order.
pipeline_name = ""     # A saved pipeline or a design name, confirmed for this bitstream.
uart_devices = {}  # {"/dev/ttyUSBn": "console", ...}; see "Find the UART ports".
timeout_s = 120

user = (user or getpass.getuser()).strip()
if not 1 <= len(user) <= 64:
    raise SystemExit(f"User name must have 1 to 64 characters: {user!r}")
print("Submitting as user:", user)
if not all([base, user, target, expected_bitfile, images, pipeline_name, uart_devices]):
    raise SystemExit("Set every setting before submitting")
images = [Path(p).resolve(strict=True) for p in images]

def get_json(path):
    with urlopen(base + path, timeout=15) as response:
        return json.load(response)

fpga = next((f for f in get_json("/api/fpgas") if f["target"] == target), None)
if fpga is None or not fpga["connected"] or not fpga["programmed"]:
    raise SystemExit(f"Target is missing, disconnected, or not programmed: {fpga}")
if fpga["bitfile"] != expected_bitfile:
    raise SystemExit(f"Loaded bitstream differs: {fpga['bitfile']}")

saved = [p for p in get_json("/api/pipelines") if p["name"] == pipeline_name]
designs = [d for d in get_json("/api/designs") if d["name"] == pipeline_name]
if len(saved) + len(designs) != 1:
    raise SystemExit(f"Pipeline name must match exactly one saved pipeline or design")
pipeline = (saved or designs)[0]

status = get_json(f"/api/fpga?target={quote(target, safe='')}")
steps = (pipeline["steps"] + (pipeline.get("reset_assert_steps") or [])
         + (pipeline.get("reset_release_steps") or []))
missing = sorted({s["probe"] for s in steps if s["op"] == "vio_set"} - set(status["vio_probes"])
                 | {s["axi_port"] for s in steps if s.get("axi_port")} - set(status["axi_ports"]))
if missing:
    raise SystemExit(f"Pipeline does not match the loaded design; missing: {missing}")

load_steps = [s for s in pipeline["steps"] if s["op"] in ("load_elf", "load_bin")]
if len(load_steps) != len(images):
    raise SystemExit(f"Pipeline has {len(load_steps)} load steps; give one image for each")

def upload(path):
    return json.loads(subprocess.check_output([
        "curl", "--fail", "--silent", "--show-error", "--max-time", "120",
        "-F", f"file=@{path}", base + "/api/uploads",
    ]))

uploads = [upload(p) for p in images]
run_steps = [dict(s) for s in pipeline["steps"]]
for step, uploaded in zip((s for s in run_steps if s["op"] in ("load_elf", "load_bin")), uploads):
    step["path"] = uploaded["path"]

body = dict(
    user=user, elf=uploads[0]["path"], program=False, trace=False,
    fpga_target=target, fpga_device=fpga["device"],
    bitfile=fpga["bitfile"], probefile=fpga["probefile"],
    design=pipeline_name if designs else None, pipeline_name=pipeline_name,
    pipeline=run_steps,
    reset_assert_steps=pipeline.get("reset_assert_steps"),
    reset_release_steps=pipeline.get("reset_release_steps"),
    params=pipeline.get("params"), timeout_s=timeout_s,
    uarts=[{"device": dev, "baud": 115200, "label": label}
           for dev, label in uart_devices.items()],
)
request = Request(base + "/api/runs", data=json.dumps(body).encode(),
                  headers={"Content-Type": "application/json"})
with urlopen(request, timeout=30) as response:
    run = json.load(response)
print("Run:", run["id"], "status:", run["status"])
for image, uploaded in zip(images, uploads):
    print("Image:", image.name, "->", uploaded["path"], "sha256:", uploaded["sha256"])
print("UI:", f"{base}/ui/runs/{run['id']}")
PY
```

After you submit:

- If the submit request gives a timeout, read `GET /api/runs?q=<user>` before
  you try again. The server can have accepted the run. Do not submit the same
  run two times.
- `POST /api/runs/<id>/rerun` makes a **new** run. The new run uses the ELF,
  pipeline, timeout, and UART list of the old run. Use it only when all these
  items are still correct. To test a new image or to change the UART list,
  submit a new run.
- `/openapi.json` shows the other `RunRequest` fields, for example `priority`
  (default `100`), `trace`, `ila_capture`, and `debug`. Use their default
  values unless the user asks for a change. A debug run cannot also record a
  trace or an ILA.

## Monitor the run and get the results

1. Read `GET /api/runs/<id>`.
2. Examine `status`:
   - `queued`: the run waits in the queue.
   - `running`: the run is active.
   - All other values are final, for example `done`, `failed`, `error`, or
     `timeout`. The schema does not limit `status` to a fixed list. Use an
     unknown value as a final status. Do not use it as a pass.
3. Read `stop_reason` to find why the run stopped.
4. To wait, read the run again every few seconds. Or read the server-sent event
   stream `GET /api/events` (`run.status`, `queue.changed`, `fpga.status`).
   Do not create more runs to wait.

Get these results. Give them to the user with the run ID and the UI link.

| Result | Endpoint | Notes |
| --- | --- | --- |
| UART output | `GET /api/runs/<id>/uart/<idx>.log` | One log for each UART. |
| Hardware commands | `GET /api/runs/<id>/commands.log` | The pipeline commands and their output. If the run fails before the firmware starts, read this log first. |
| Trace and ILA | `trace.csv`, `ila.csv`, `ila.vcd`, `ila_annotated` | Only if the run recorded them. |

Status `done` shows only that the run completed its timeout without an error.
Use the UART output to decide if the test passed. Tell the user which output
you used.

## Send data to a UART

Some tests need input, for example console commands or an image download. FRQ
sends the bytes from `WS /api/runs/<id>/uart/<idx>/ws` to the serial port. The
socket also sends the port output to you. Use the UART index of the run.

1. Before you submit the run, set `timeout_s` long enough for the full
   transfer.
2. Read `GET /api/runs/<id>`.
3. Make sure that `status` is `running` and that `started_at` has a value.
   Status `running` alone can also include the pipeline preparation.
4. Send the data soon after the start. The run timeout and the handshake
   timeout of the firmware are limited.

> [!WARNING]
> Do not send data to a queued run. A queued run does not own the UART yet. The
> data can go to a different active run.

## Clean up

- Close all SSH tunnels that you opened.
- Keep the FPGA programmed.
- Do not delete runs, uploads, bitstreams, or saved pipelines unless the user
  asks.

