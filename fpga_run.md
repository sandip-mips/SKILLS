---
name: fpga-run
description: Execute a compiled binary on the FPGA via the FPGA RUN Queue (FRQ). Trigger when the user requests to run, test, or benchmark a specific executable on the FPGA.
---

## Select the FRQ server

There are several FRQ servers, each fronting its own FPGA. Do not assume one.
The user must name the server (host name or IP) to use. If they have not, ask
before you read or submit anything. Do not pick a server from an earlier
conversation or from an example in this guide.

Call the chosen endpoint `FRQ_BASE`. Check that it answers:

```sh
curl --fail --silent --max-time 10 "$FRQ_BASE/api/fpga"
```

Use `FRQ_BASE=http://<server>:8000` when the server is reachable directly.

If the port is not reachable from this host (timeout or connection refused),
forward it over SSH and use localhost:

```sh
ssh -N -L 8000:localhost:8000 <ssh_user>@<server>   # keep running; use run_in_background
```

Then use `FRQ_BASE=http://localhost:8000`. Ask the user for the SSH user if you
do not know it. If local port 8000 is already in use, forward another local port
(for example `-L 8001:localhost:8000`) and use that port in `FRQ_BASE`. Make sure
the tunnel reaches the intended server before you submit: read `GET /api/fpga`
through it and confirm with the user if the target is unexpected. Close the
tunnel when the work is done.

All examples below use `FRQ_BASE`. The example hosts, pipelines, and UART names in
this guide describe one past setup (Bumblebee); verify every value against the
selected server.

Use the programming control on the home page to configure the FPGA. Use
`/ui/submit` to submit a test. The UI and the API show the run details and logs.

Before you submit a run, make sure that the FPGA target, device, bitstream,
probe file, pipeline, and UART ports are correct. Old run settings do not prove
the current FPGA state. Read `GET /api/fpga` and the run queue. Get explicit
approval before you reconfigure a shared FPGA or stop the run of another user.

The Boot ROM pipelines load `corvi_boot.elf` into ROM_P through `hw_axi_1`.
Each pipeline sets the BMODE VIO probe and then applies a hardware POR:

| Pipeline | BMODE | Boot mode |
| --- | --- | --- |
| `bb_0.3.1_rom_min_uart` | `1` | UART: receive a BOOT image through XMODEM |
| `bb_0.3.1_rom_min_nvm` | `7` | NVM: load the APP image from MRAM_1 |

The earlier SRAM boot flow used the saved `bb_sram_min` pipeline. Make sure that
a pipeline still exists and agrees with the selected design before you use it.
A pipeline name does not prove which bitstream the FPGA has.

### FPGA Programming

Use the home-page programming control for the fast configuration method.
Its API request is `POST /api/fpga/program` with these JSON fields:

```json
{
    "bitfile": "<server-side bitstream path>",
    "probefile": "<server-side probe file path>",
    "target": "<verified FPGA target>"
}
```

Use the paths and the target from a known-good reference run or from the current
FPGA inventory. Wait for the request to complete. The UI allows a 3900-second
request timeout. A client timeout does not prove that the configuration stopped.
If the request times out, read the FPGA status before you try again.

Read the returned `programmed`, `bitfile`, and `program_progress` fields.
Make sure that `bitfile` is the selected bitstream and that no configuration
error occurred. Then submit the test with `program=false`. The test then uses the
loaded design. Do not set `program=true` to request the fast method. Run-time
JTAG programming is slow and is a different path.

Each FPGA configuration clears modeled OTP. A SoC POR alone keeps modeled OTP and
ROM. Do not program a new patch image over the previous one. This procedure is
for modeled OTP, not physical OTP.

### UART discovery

UART assignments depend on the server, the target, and its cabling. The same
`/dev/ttyUSBn` name can have a different role on another server or after a
re-plug. Do not use the device suffix to find the role. Never reuse a mapping
from another server.

Resolve the UARTs in this order:

1. **The user states them.** If the user gives the console and download ports for
   this server, use those.
2. **A saved mapping exists.** Check for a mapping recorded for this server
   (for example `.frq/uarts.json`, see below, or a past run from
   `GET /api/runs`, whose `uarts` list shows what it captured). Use it, but
   confirm it still matches the devices the server lists.
3. **Discover by capturing all.** Otherwise list the USB serial devices on the
   server (from `GET /api/fpga` or the UI if it lists them; otherwise ask the
   user or run `ls /dev/ttyUSB*` over SSH). Submit the run with every device in
   `uarts`, labelled by its device name (`ttyUSB0`, `ttyUSB1`, ...). After the
   run, read the logs of each port: the port that prints the boot banner or
   console output is the console; the port that receives the XMODEM handshake
   (`C` characters) is the download port. Report the finding to the user.
4. **Save the result.** Write the verified mapping so later runs capture only
   the necessary ports:

   ```json
   {"server": "<server>", "console": "/dev/ttyUSB2", "download": "/dev/ttyUSB3", "baud": 115200}
   ```

   Ask the user before you write it to the repository. If the setup changes
   (new bitstream, different board, re-plugged cables), discard the mapping and
   discover again.

The WebSocket index is the position of the port in the `uarts` list of the run.
A run that captures only one port uses UART index `0`. When you capture all
ports, note the index of each in the order you listed them.

Past setups, for illustration only (do not reuse): an earlier SRAM/XMODEM setup
used `ttyUSB0` as console and `ttyUSB1` as download; a `bb_0.3.2` setup used
`ttyUSB2` as console with an unidentified download port.

### Submit a fresh run

Build first. The lab server cannot read files on the development host. Upload the ELF
with `POST /api/uploads` (multipart field `file`) and use the server-side `path` that it
returns.

Edit the settings, and then run the script below from the repository root. The
script uploads the current Boot ROM ELF. It submits a fresh run with the FPGA and
bitstream settings of a known-good run and the current saved pipeline.

Set `reference_run_id` to a known-good run for the intended FPGA. Set `user` to the lab
identity for this request. Set `pipeline_name` to the pipeline for the boot mode. Old run
IDs do not prove the current FPGA configuration. `program=False` requires that the FPGA
already has the correct bitstream.

```sh
uv run --project scripts/xmodem_util python - <<'PY'
import json
import subprocess
from pathlib import Path
from urllib.request import Request, urlopen

base = ""  # FRQ_BASE chosen by the user, e.g. http://<server>:8000 or http://localhost:8000 (tunnel).
reference_run_id = None  # Set a verified run ID for the intended FPGA.
user = ""  # Set the lab user identity.
pipeline_name = "bb_0.3.1_rom_min_uart"  # Example; pick the pipeline for the boot mode.
# UART capture. Set `uart_devices` to the verified ports, or list every USB serial
# device to discover which one carries data (see "UART discovery"). Labels are
# free-form; use "console"/"download" once known, else the device name.
uart_devices = {}  # e.g. {"/dev/ttyUSB2": "console"} or {"/dev/ttyUSB0": "ttyUSB0", ...}
preset = "bumblebee-public-fpga-debug"
elf = Path(f"build/{preset}/firmware/public/corvi/boot/corvi_boot.elf")

if not base or reference_run_id is None or not user or not uart_devices:
    raise ValueError("Set base, reference_run_id, user, and uart_devices before submitting")
elf = elf.resolve(strict=True)

def get_json(path):
    with urlopen(base + path, timeout=15) as response:
        return json.load(response)

reference = get_json(f"/api/runs/{reference_run_id}")
pipeline = next(item for item in get_json("/api/pipelines") if item["name"] == pipeline_name)
load_steps = [step for step in pipeline["steps"] if step["op"] == "load_elf"]
if len(load_steps) != 1:
    raise ValueError("Inspect the pipeline: expected one load_elf step")
upload = json.loads(subprocess.check_output([
    "curl", "--fail", "--silent", "--show-error", "--max-time", "30",
    "-F", f"file=@{elf}", base + "/api/uploads",
]))
load_steps[0]["path"] = upload["path"]
body = {key: reference[key] for key in (
    "fpga_target", "fpga_device", "bitfile", "probefile",
)}
body.update(
    user=user, elf=upload["path"], program=False, trace=False,
    timeout_s=120, pipeline_name=pipeline["name"], pipeline=pipeline["steps"],
    reset_assert_steps=pipeline["reset_assert_steps"],
    reset_release_steps=pipeline["reset_release_steps"],
    params=pipeline.get("params"),
    uarts=[{"device": dev, "baud": 115200, "label": label}
           for dev, label in uart_devices.items()],
)
request = Request(base + "/api/runs", data=json.dumps(body).encode(),
                  headers={"Content-Type": "application/json"})
with urlopen(request, timeout=15) as response:
    run = json.load(response)
print("Run:", run["id"], "status:", run["status"], "UARTs:", run["uarts"])
PY
```

For a BootROM-to-SBL test, upload `corvi_boot.elf` with `bb_0.3.1_rom_min_uart`, and
send `sbl.image` as shown below. For an independent SBL test, upload
`firmware/public/corvi/sbl/sbl.elf` with an SRAM boot pipeline, such as `bb_sram_min`,
that agrees with the current design. Use the same preset and build directory for all
artifacts. If the submission times out, read the run list in the UI before you try again.
The server can have accepted the run.

`POST /api/runs/<RUN_ID>/rerun` creates a **new** run with the old ELF, pipeline, timeout,
and UART capture list. Use it only when those settings are still correct. To test a new
ELF or to change the UART ownership, submit a fresh run.

Before you send, read `GET /api/runs/<RUN_ID>`. Make sure that the run is `running` and
that `started_at` has a value. The `running` status alone can include the pipeline
preparation.

A queued run does not own the UART yet. If you send directly to a queued run,
the data can go to a different active run. Use the UI or the `/api/events` stream to wait
for the start. Do not create more runs to wait. Send quickly, because the run timeout and
the receiver handshake timeout of the firmware are finite.

