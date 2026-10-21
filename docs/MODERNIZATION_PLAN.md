# Lightclock modernization plan

Status: proposal, October 2026. Branch: `webapp` (50 commits ahead of `master`, last commit 2019-10-04).

## 1. Where the project stands

**What it is:** MicroPython firmware for an Adafruit HUZZAH ESP8266 Feather. A sunrise alarm drives a PWM LED,
switches a fan, and reads an MCP9808 temperature sensor over I2C with a thermal-shutdown watchdog. A small web app
(picoweb + precompiled utemplate + Bootstrap 3/jQuery) is used to set the alarm time and config.

**Why it no longer builds:**

| Piece | Pinned today | Status in 2026 |
|---|---|---|
| MicroPython | `v1.9.3` (2017) | Current stable is **v1.29.0**. The `make axtls` step, `upip` and the old `modules/` symlink approach are all gone. Frozen code is now declared in a `manifest.py`. |
| micropython-lib | `v1.9.3` (symlinked `uasyncio`, `logging`, `pkg_resources`) | `uasyncio` became built-in `asyncio` (v3 API) in v1.13+. The old symlink layout no longer exists. |
| picoweb | upip install | Unmaintained, and it needs the pre-v3 `uasyncio` API, so it **cannot run on current MicroPython**. |
| utemplate | git clone next to the repo | Still works as pure Python, but it is not in micropython-lib, so it has to be vendored. |
| mcp9808 driver | git clone + symlink | Has to be vendored. |
| Host tooling | Python 2 `ampy`, `esptool.py`, a native unix MicroPython build | Python 2 is gone from Ubuntu 26.04. `ampy` is deprecated in favour of **`mpremote`**. esptool is **v5.x** (commands renamed `write-flash`/`erase-flash`; the old names are deprecated). |
| CI / releases | none (README mentions a "releases page" that is not automated) | none |

**Code-level problems found while reading** (these need fixing before the clock goes back on the nightstand):

- `alarmclock/temperature_sensor.py`
  - `measure_temp()` returns `None` when the sensor fails. Callers then do `temp, frac = self.measure_temp()` and
    raise `TypeError` inside a timer callback, so the **thermal watchdog crashes exactly when it should act**.
  - `self.mcp` is never set when `MCP9808()` raises in `__init__`, so a later `if self.mcp:` raises `AttributeError`.
  - `uos.stat('temp_log.csv')` hardcodes the file name instead of using the `temp_log_file` config value.
  - `TemperatureLogger` and `TemperatureWatcher` each create their own I2C bus and sensor object and read the sensor
    twice per tick.
- `main.py` does blocking work (NTP with retries, file writes, PWM) inside `machine.Timer` callbacks while picoweb
  owns the event loop. In AP mode it blocks forever in `while not ap_if.isconnected()` before the web app starts.
- `machine.PWM(..., freq=20000)`: the ESP8266 software PWM tops out around 1 kHz, so this value is silently clamped.
  The code also re-creates the PWM object on every tick. Use one long-lived PWM object with an explicit, valid frequency.
- Config/secrets: the README says Wi-Fi credentials go in `config_wifi.json` with `static_ip`. The code actually reads
  `wifi_ssid`/`wifi_password`/`wifi_static_ip` from the committed `config.json`, and `wifi_static_ip` is not in it.
- Broken helpers: `main_webapp_test.py` (wrong `do_connect`/`WebApp` signatures), `main_test.py` + `dev_tools/clock_actions.py`
  (wrong import, outdated action API), `dev_tools/draw_profile.py` (outdated `SunriseExp` API), `dev_tools/test_fan.py`
  (missing `json` import).
- `utc_delay` is a manual offset, so it is wrong for half the year (no DST handling).
- Open TODOs: setting the time from the web app in offline mode (`main.py:129`), and static IP handling (`connect_to_router.py:39`).

The `asynchron` branch was rebased into `webapp` by hand and contains nothing that isn't already here. It can be deleted.

## 2. Target stack

| Concern | Choice | Why |
|---|---|---|
| Firmware | MicroPython **v1.29.0**, board `ESP8266_GENERIC` (default 2 MiB+ variant, which fits the Feather's 4 MiB) | Current stable. Pin the tag, bump on purpose. |
| Frozen code | `manifest.py` in this repo: `include("$(PORT_DIR)/boards/manifest.py")` + `package("alarmclock")`, `package("webapp")`, vendored `utemplate`, `mcp9808`, and the **compiled templates** | Freezing saves RAM, which is the scarce resource on ESP8266 (~35–40 KB free heap). |
| Async runtime | built-in `asyncio` | Replaces `machine.Timer` callbacks and picoweb's loop with one cooperative loop. |
| Web framework | **Microdot v2.7.0** (asyncio, has a utemplate integration), vendored and frozen | Maintained successor in the picoweb niche. **Risk:** RAM on ESP8266. Phase 2 starts with a spike; the fallback is a ~100-line hand-written `asyncio.start_server` handler, which is all this app needs (`GET /`, `POST /`, static gz files). |
| Templates | utemplate (vendored as a git submodule, pinned), compiled offline in the container | The existing approach still works. |
| Front end | Drop jQuery 1.11 + Bootstrap 3 + clockpicker (~68 KB gzipped, used only for a collapse and a time picker). Use native `<input type="time">`, `<details>` for the collapsible settings, and a small hand-written CSS file. Total ≈ 2–3 KB gzipped, **served from the board only, no CDN**. | The CDNs existed for speed in online mode (local copies took 6–8 s per file before gzip). With assets ~25× smaller that reason is gone, while CDNs fail in AP/offline mode, depend on third-party hosts (`maxcdn.bootstrapcdn.com` may already be retired; unverified), leak visits, and pin libraries with known CVEs. File size doesn't affect RAM, because static files are streamed in chunks. |
| Board I/O | **`mpremote`** (pip) | Official tool, replaces ampy. Supports `fs cp -r`, `mount` (run code from the PC without copying), `run`, and `repl`. |
| Flashing | **esptool v5.x** (pip) | |
| Host Python tooling | `uv` with a `pyproject.toml` (dev group: esptool, mpremote, pytest, ruff) | One lockfile, works the same on Linux, macOS and Windows. |
| Linting | `ruff` | |

## 3. Machine-agnostic build and flash environment

**Principle:** build in Docker (deterministic and identical everywhere). Flash/deploy with pure-Python tools that run
either in the same container (Linux, with `--device`) or natively via `uv` (any OS). Docker Desktop on macOS/Windows
cannot pass USB serial devices through, so a Docker-only flashing setup would not be machine-agnostic.

### 3.1 Layout

```
Dockerfile                 # FROM larsks/esp-open-sdk (toolchain recommended by upstream) + python3, git, uv
                           # bakes MicroPython v1.29.0 checkout + submodules + prebuilt mpy-cross
compose.yaml               # services: build, shell, flash (flash adds devices: ["${PORT:-/dev/ttyUSB0}"])
manifest.py                # frozen modules
pyproject.toml / uv.lock   # host tools: esptool, mpremote, pytest, ruff
vendor/                    # git submodules pinned to tags: utemplate, microdot, micropython-mcp9808
justfile (or Makefile)     # single entry point, see below
.devcontainer/             # optional: VS Code / Codespaces using the same Dockerfile
```

The `larsks/esp-open-sdk` image is old but is the toolchain upstream still documents for ESP8266. Pin it **by digest**.
If it ever disappears, fall back to the `jepler/esp-open-sdk` prebuilt toolchain tarball (also linked from the upstream
README) installed into a `debian:stable-slim` image.

### 3.2 Commands (replace the current Makefile)

```
just build          # docker: compile templates, gzip static, mpy-cross check, build firmware → build/firmware.bin
just flash          # esptool erase-flash + write-flash build/firmware.bin   (PORT=/dev/ttyUSB0 by default)
just deploy         # mpremote: copy config.json, config_wifi.json, static assets, boot.py, main.py
just config         # mpremote: push only config files
just repl / logs    # mpremote repl
just dev            # mpremote mount . + run main.py — iterate on Python without reflashing
just test           # pytest on the host (see 3.4)
just lint           # ruff
```

`flash` and `deploy` run natively through `uv run` by default. `just docker-flash` is the Linux-only variant through
compose, which needs the user in the `dialout` group. Ubuntu note: if the port disappears, `brltty` or `ModemManager`
grabbed it; document `sudo apt remove brltty` in the README.

### 3.3 GitHub Actions

- `ci.yml` (push/PR): ruff, pytest, `docker build`, firmware build, upload `firmware.bin` + `webapp-assets.zip` as
  artifacts. Cache the Docker layer with the MicroPython checkout.
- `release.yml` (tag `v*`): same build, attach firmware + assets to a GitHub Release. This makes the README's
  "download from the releases page" install path real.
- Dependabot for `github-actions`, `pip` (uv) and git submodules, so the stack can't silently rot for 7 years again.

### 3.4 Testing without hardware

Move pure logic out of hardware classes (alarm-window computation, sunrise intensity curve, temperature watchdog
state machine, config form parsing, DST calculation) and test it with **pytest on CPython** using small fakes for
`machine`, `time` and the sensor. Optionally also run the same tests under the MicroPython **unix port** built in the
container (`micropython -m unittest`) to catch MicroPython-only differences. This replaces the broken `main_test.py`
and `main_webapp_test.py`.

## 4. Feature work to finish on this branch

Ordered by importance for putting the clock back into service.

1. **Reliable thermal protection** (safety, must-have)
   - Merge logger and watcher into one `TemperatureMonitor` with a single I2C/sensor instance and one read per tick.
   - Treat "sensor missing or read failed" as unsafe: LED off, fan on, error state shown in the web app. Never raise.
   - Use the configured log file name. Keep the size-capped CSV rotation.
   - Unit-test the state machine: over temperature → shutdown, hysteresis → recover, sensor failure → shutdown.
2. **Async main loop** (replaces Timers)
   - `asyncio` tasks: `alarm_task` (1 s tick), `ntp_task` (non-blocking, retries with backoff), `web_server`.
   - The LED/PWM object is created once. The watchdog runs every tick, even while the web server is handling a request.
3. **Networking and offline mode** (closes both TODOs)
   - Move credentials to a git-ignored `config_wifi.json` (matches the README); `config.json` holds only defaults.
   - DHCP by default and `network.hostname("lightclock")`. Static IP becomes optional and is applied *before*
     `connect()`; if it is set but the network doesn't match, log it and stay on DHCP.
   - If the STA connection fails: start the AP with a configured password (not the MicroPython default) and start
     the web server right away (no blocking wait loop).
   - Offline web app: a Wi-Fi form that saves `config_wifi.json` and reconnects without a reboot, plus **manual
     time setting** (POST of the browser's current time → `machine.RTC().datetime(...)`) so the alarm works without NTP.
4. **Automatic time zone with DST**
   - Replace `utc_delay` with a `timezone` config value holding a POSIX TZ rule, e.g. `CET-1CEST,M3.5.0,M10.5.0/3`
     (default). A small pure-Python parser/evaluator for the `Mm.w.d/time` form (the common one) is unit-tested
     against known transition dates.
   - The RTC stays in UTC (as NTP sets it). Local time = UTC + offset from the rule, computed on demand. This removes
     the "shift the RTC when utc_delay changes" hack from the web app.
   - The web app detects the browser's zone (`Intl.DateTimeFormat().resolvedOptions().timeZone`), maps it to a POSIX
     rule via a small built-in table (Europe, US, a few others), and offers it as a one-click "use my time zone".
     There is also a free-text field for any rule.
   - No IP-geolocation or online time-zone API: it would add an external dependency and fail in offline mode.
5. **Web app refresh**
   - Port the routes to Microdot (or the fallback server). Keep the gzip static serving and the cache headers.
   - Native time picker, collapsible settings via `<details>`, server-side validation of min/max from the config schema.
   - Show the current temperature + sensor status, the next alarm, an **"alarm on/off" toggle**, a **"test sunrise
     now" / "light off" button**, and a link to download `temp_log.csv`. A tiny inline-SVG chart of the log is a
     nice-to-have.
6. **Cleanup**
   - Delete or fix the `dev_tools` scripts (keep `test_led`, `test_fan` and `record_temp` as `mpremote run` targets),
     remove the obsolete `webapp/README.md` upip instructions, rewrite the README for the new workflow, and delete the
     `asynchron` branch.

## 5. Execution order and milestones

| # | Milestone | Done when |
|---|---|---|
| M0 | Safety net | Current Feather filesystem and flash backed up (`esptool read-flash 0 0x400000 backup.bin`, `mpremote cp -r :/ backup/`). |
| M1 | Toolchain | `just build` produces a stock v1.29.0 firmware with the old `alarmclock` frozen (imports may fail at runtime). CI is green. |
| M2 | Microdot spike | A hello-world Microdot + one compiled utemplate page runs on the Feather with ≥ 10 KB free heap after start. Decide Microdot vs. the hand-written server. |
| M3 | Port to modern MicroPython | `asyncio` main loop, fixed temperature monitor, single PWM object, host tests passing. The clock runs a sunrise on the bench. |
| M4 | Networking/offline | STA + AP fallback, `config_wifi.json`, manual time setting, DST. |
| M5 | Web app refresh | New front end, toggles, temperature log download. |
| M6 | Release | Tag `v1.0.0`, release artifacts from CI, README rewritten, physical clock flashed and running for a few nights (watch temperature logs). |

Merge `webapp` → `master` after M3 (the first build that runs on the clock), then work on short-lived feature
branches from there.

## 6. Risks and open questions

- **RAM on ESP8266** is the main technical risk (Microdot + asyncio + templates). Mitigations: freeze everything,
  compile templates, stream responses, gzip static files, and keep the fallback server ready. The hardware stays on
  the HUZZAH ESP8266, so the fallback server is the real plan B, not a board swap.
- The `larsks/esp-open-sdk` image is unmaintained. Pin it by digest and keep the tarball fallback documented.
- PWM frequency: pick an explicit value ≤ 1 kHz and check on the bench that there is no audible coil whine or visible
  flicker at low duty. The current exponential curve starts at a very low duty, where 10-bit resolution steps are visible.
  Consider `duty_u16` with a gamma correction.
- Decisions (owner, October 2026):
  - Hardware stays the HUZZAH ESP8266.
  - Automatic time zone handling via a POSIX TZ rule, prefilled from the browser.
  - No CDNs; the minimal front end is served from the board.
