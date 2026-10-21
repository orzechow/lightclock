# Lightclock modernization plan

*Written 2026-10-21. State of the repo at that point: last commit Oct 2019 (`13ee73f`), branch `webapp`.*

## TL;DR

- **Bumping versions won't revive the old stack.** The webapp is built on picoweb, which needs the `uasyncio` that MicroPython replaced in 2020. The README's other steps fail too:
  - `pip2`/ampy needs Python 2, which Ubuntu 26.04 doesn't ship.
  - `upip` and `make axtls` no longer exist.
  - The micropython-lib paths the README symlinks have moved.
- **Recommendation: rewrite the firmware as an [ESPHome](https://esphome.io) configuration and keep the existing Feather HUZZAH ESP8266.** ESPHome provides most of what the device needs as maintained components:
  - `http://lightclock.local` (mDNS) and a live web UI
  - over-the-air (OTA) updates
  - a fallback hotspot with a captive portal, for entering new Wi-Fi credentials
  - a time zone with daylight saving time
  - settings that survive power loss
  - an MCP9808 driver

  The only project-specific logic is the alarm schedule and the fan/thermal rules: about 60 lines of C++ inside the YAML.
- **Build environment:** the official ESPHome Docker image, pinned to one version. The first flash goes over USB from Linux, or from a browser on any OS; every later update goes over Wi-Fi.
- **A draft config already passes `esphome config` with ESPHome 2026.10.0:** [`docs/lightclock.draft.yaml`](lightclock.draft.yaml). It has not been compiled or flashed yet, because the environment it was written in can't reach the PlatformIO registry.
- **Optional later:** move to an ESP32 Feather if the darkest part of the sunrise looks steppy, or if you want a 4-pin PC fan. With ESPHome that means changing one block of config and redoing the wiring.

---

## 1. Current state

### What's there

| | |
|---|---|
| Board | Adafruit Feather HUZZAH ESP8266, 4 MB flash, CP2104 USB-serial |
| Firmware | MicroPython **v1.9.3** (Nov 2017), with `alarmclock/` and `webapp/` frozen into the image |
| Web | picoweb + offline-compiled utemplate on port 8081; Bootstrap 3.3.5, jQuery 1.11.3, clockpicker |
| Tooling | Makefile wrapping `ampy` (Python 2), `esptool.py`, gzip, `sed` minification |
| Repo | 142 commits (Jan 2017 – Oct 2019). The latest code is on `webapp`. GitHub's default branch is still `master` (Jan 2017); `master` and `asynchron` (Jan 2018) are older and fully merged into `webapp`. No tags. **No GitHub releases**, although the README tells you to download one, so whatever is on the device is the only built copy |

**Pin map** (from `config.json` / `main.py`):

| Function | GPIO | Notes |
|---|---|---|
| LED driver, PWM | 15 | Boot-strap pin, must be low at boot, so the LED is off during boot (good) |
| Fan, on/off today | 13 | Plain digital output; will become PWM |
| I²C SCL / SDA | 5 / 4 | MCP9808 at `0x18`; the bus runs at 10 kHz (long wires?) |
| Status LED (red, active low) | 0 | Boot-strap pin |

**Runtime design:** two `machine.Timer` callbacks run beside picoweb's event loop:
- an alarm tick every 1 s, which calls `pre_action` / `main_action` / `post_action` on `[SunriseExp, Fan, TemperatureLogger, TemperatureWatcher]`
- an NTP sync every 30 s

**Features today:**
- **Sunrise:** an exponential ramp that starts at the alarm time, lasts `sunrise_time_sec` (20 min) and is capped at `max_intensity_percent`. The light switches off abruptly when the ramp ends.
- **Fan:** on during the sunrise, off otherwise, forced on when the LED overheats.
- **Thermal protection:** LED off and fan on at ≥ 40 °C; released below 38 °C.
- **Temperature log:** a CSV file in flash, one line per second, all day.
- **Webapp:** big current time, an alarm time picker that submits on change, a collapsible settings form, and an "offline mode" on the access point that shows the Wi-Fi fields.

### Why it no longer builds

| Piece | 2019 | Today | Effect |
|---|---|---|---|
| MicroPython | v1.9.3 | v1.29.0 (Aug 2026) | — |
| Web framework | picoweb on old `uasyncio` | `asyncio` rewritten in v1.13 (2020); picoweb unmaintained | **Hard blocker:** the webapp can't run on any current MicroPython |
| Packages | `upip` | `mip` (`upip` removed) | README steps fail |
| micropython-lib | `logging/logging.py`, `uasyncio.core/…` | Restructured | Symlink steps fail |
| ESP8266 build | `make axtls`, symlinks into `modules/` | `make BOARD=ESP8266_GENERIC`, frozen modules via `manifest.py`, upstream recommends the `larsks/esp-open-sdk` Docker image | Build steps fail |
| Host tools | Python 2, `pip2 install adafruit-ampy`, `esptool.py` | No Python 2; ampy archived (use `mpremote`); esptool v5 | Deploy steps fail |
| Front-end | Bootstrap 3, jQuery 1.11 from MaxCDN/Google CDNs | Both long end-of-life | Works only with the bundled copies |

### Bugs and design issues not to carry over

In order of importance:

1. **Overheat protection fails exactly when the sensor fails.**
   - `measure_temp()` returns `None` on a read error, and callers unpack it with `temp, frac = …`, which raises a `TypeError` (`alarmclock/temperature_sensor.py:75`, `:107`, `:116`).
   - If the sensor fails at init, `self.mcp` is never set, so `if self.mcp:` raises an `AttributeError` (`:34`).
   - The logger runs before the watcher in the same loop (`main.py:60-86`), so once the logger throws, the **watcher never runs**.
2. **Flash wear.** The logger appends to a file in flash every second, day and night (`temperature_sensor.py:75-86`). On v1.9.3 the ESP8266 filesystem was FAT without wear levelling, so the same sectors were rewritten roughly 86,000 times a day.
   - If the board behaves strangely, worn flash is a plausible cause (see Phase 0).
   - Rule for the new firmware: no fast periodic flash writes.
3. **Time zone:** `utc_delay` is a fixed offset with no daylight saving time, and the form's `min: 0` rules out negative offsets.
4. **Alarm semantics:**
   - The light *starts* at the alarm time instead of reaching full brightness *at* it (the lead time is forced to 0 at `main.py:95`).
   - It switches off abruptly at the end of the ramp.
   - There is no alarm on/off and no weekday selection.
   - A sunrise that crosses midnight is cut off, because `alarm.py:62-75` only ever computes "today".
5. **PWM:**
   - `freq=20000` is silently limited to 1 kHz. MicroPython on the ESP8266 supports 1–1000 Hz, with one frequency shared by all pins.
   - The PWM object is re-created every second.
6. **Config:**
   - UI metadata, pins, secrets and runtime state share one dict.
   - Saving writes `current_time` / `current_temp` into `config.json` (`webapp/__init__.py:75`).
   - In offline mode the Wi-Fi password is rendered as an input placeholder (`webapp_templates/config.html:62`).
7. **Small ones:**
   - `time.sleep(500)` after a config error sleeps 500 *seconds* (`main.py:46-48`).
   - The logger checks the size of the hard-coded `temp_log.csv` instead of the configured file (`:77`).
   - Most of `dev_tools/` and `main_*_test.py` are broken: missing imports, wrong signatures, a `StringPrinter` without `*_action` methods.

---

## 2. Choosing the approach

### Firmware options

| | **ESPHome** (recommended) | MicroPython 1.29 + Microdot | Rust (esp-hal/Embassy) / Arduino C++ |
|---|---|---|---|
| Runs on the existing ESP8266 | Yes | Yes, but RAM is tight; code must be frozen | Rust: no (ESP32 family only). Arduino: yes |
| `lightclock.local` | Built in | ESP32 only. On the ESP8266, only the DHCP hostname via your router | Arduino: ESP8266mDNS. Rust: extra crate |
| Web UI | Built in: live, works on phones, extensible | You write it | You write it |
| OTA updates | Built in | Not on the ESP8266 (USB or WebREPL only) | ArduinoOTA / write it |
| Fallback hotspot + captive portal | Built in | You write it | You write it |
| Time zone with daylight saving time | Built in (`Europe/Berlin` etc.) | You write it | Library |
| MCP9808 driver | Built in | Third-party driver | Library |
| What you maintain | ~250 lines of YAML, ~60 of them C++ lambdas | All of it: Python app, web front-end, Wi-Fi handling | All of it |
| Build environment | One pinned Docker image | Docker image for the firmware, plus mpremote/esptool on the host | Toolchain image |
| Upkeep | Monthly releases; you bump one image tag | You follow MicroPython and each library | You follow everything |

**Why ESPHome:**
- What the device needs is what ESPHome ships as maintained components: Wi-Fi, mDNS, a web UI, timekeeping, PWM outputs, an I²C sensor, persistence and OTA.
- The project-specific part shrinks to the alarm and thermal rules, so there is much less code to rot again.
- ESPHome generates C++ and builds it with PlatformIO. If a lambda grows too large, it can become a small custom component without changing tools.

**What you give up:**
- **The UI is ESPHome's generic page**, a list of entities in the groups "Alarm" and "Settings", not a hand-made clock face. A custom front-end can come later (Phase 5); it would use the same REST/event-stream API.
- **The logic is C++ in YAML instead of Python.** Phase 3 moves the non-trivial parts into a small header that is unit-tested on the host.
- **The first build downloads the ESP8266 toolchain** into a Docker volume.

**Pick MicroPython instead if** you want to stay in Python and build your own UI, and can accept no mDNS and no OTA on the ESP8266, or you upgrade to an ESP32. [Appendix A](#appendix-a--micropython-alternative) sketches that path.

### Hardware: keep the ESP8266 for now

Do Phases 0–6 on the existing HUZZAH. Consider an ESP32 Feather (for example ESP32-S3 or ESP32-C6: same form factor, different GPIO numbers) only if the bench tests show one of these:

- **The sunrise starts with a visible jump or steps.**
  - ESP8266 PWM is software-timed; at 1 kHz the shortest usable pulse is about 0.1 % duty.
  - With gamma 2.8 that is already ≈ 8 % on the brightness scale, so the first step is visible in a dark room.
  - The ESP32's hardware PWM (LEDC) has about 16-bit resolution at 1 kHz, so its first step is ≈ 2 %.
- **You want a 4-pin PC fan.** These expect 25 kHz PWM and report speed on a tach wire; the ESP32 does 25 kHz at about 11 bits.
- **The LED flickers during Wi-Fi traffic** (software PWM jitter).

With ESPHome the move means three changes: swap the `esp8266:` block for `esp32:`, change the `esp8266_pwm` outputs to `ledc`, and update the pin numbers. The rest of the config stays as it is.

---

## 3. Target design

### Features

**Alarm**
- **Settings, all stored on the device so they survive power loss:**
  - wake time (time picker)
  - alarm on/off
  - sunrise duration
  - "stay on" duration
  - maximum brightness
- **Sunrise ends at the wake time:**
  - It starts *sunrise duration* minutes earlier and ramps from 1 % to maximum brightness.
  - ESPHome's gamma-corrected linear transition has the same slow-start, steep-end shape as the old exponential curve. A custom curve is possible later if needed.
  - The light then stays on for the "stay on" duration and fades out over 1 min.
  - Midnight wrap is handled.
- **Buttons:** "Stop alarm" and "Test sunrise".
- **No time, no alarm:** the alarm doesn't fire until the clock has synced via NTP.
- **Stretch goals:** weekday selection, snooze.
- **Bonus:** the light also works as an ordinary dimmable lamp from the UI.

**Fan and thermal protection**
- **Fan curve:**
  - Fan speed follows the MCP9808 temperature: off below about 28 °C, rising linearly to 100 % at shutdown − 2 °C.
  - The fan runs at ≥ 30 % whenever the light is on.
  - `min_power` covers the fan's spin-up threshold.
- **Over-temperature:**
  - At or above the shutdown temperature (default 40 °C): light off, fan 100 %.
  - It releases below shutdown − 2 °C.
- **Sensor failure** (a read error, or no reading for 30 s) triggers the same fail-safe as over-temperature.
- **At boot** the light is always off.
- **No temperature log in flash.** The current value is live in the UI; Home Assistant can keep history if you ever want it.

**Webapp and local name**
- **Primary address:** `http://lightclock.local` (mDNS). Linux (Avahi), macOS, iOS and Windows 10+ resolve it.
- **Fallback name:** some Android versions don't resolve `.local`. The device also registers `lightclock` as its DHCP hostname, so the router's name works too, for example `lightclock.fritz.box` or `lightclock.lan`.
- **UI layout:** the "Alarm" group is on top, then "Settings", then the light, fan, temperature and live logs.
- **Fallback hotspot:** if the home Wi-Fi is unreachable for 1 min, the device opens the access point "Lightclock Setup" with a captive portal to enter new credentials. This replaces the old offline mode. The alarm needs NTP time, so it doesn't run in hotspot-only mode, same as before; a small real-time clock module would fix that if you ever want it.
- **Security:**
  - The OTA and hotspot passwords live in a git-ignored `secrets.yaml`.
  - Optional HTTP basic auth on the web UI.
- **Optional:** the Home Assistant native API, off by default.

### Repository layout

```
lightclock/
├── firmware/
│   ├── lightclock.yaml                 # device name + list of packages
│   ├── packages/
│   │   ├── board-huzzah-esp8266.yaml   # board, pins, PWM outputs, I²C, status LED
│   │   ├── base.yaml                   # Wi-Fi, hotspot, captive portal, OTA, web_server, SNTP time
│   │   ├── alarm.yaml                  # wake time, settings, sunrise script
│   │   └── thermal.yaml                # MCP9808, fan, fan curve, protection
│   ├── include/lightclock_logic.h      # pure functions: sunrise start minute, fan curve, hysteresis
│   ├── secrets.example.yaml
│   └── secrets.yaml                    # git-ignored
├── tests/test_logic.cpp                # host unit tests for lightclock_logic.h (doctest)
├── docs/hardware.md                    # wiring, LED driver, fan, photos
├── compose.yaml
├── Makefile
├── .github/workflows/firmware.yml
├── renovate.json
└── README.md
```

ESPHome `packages:` keep each concern in its own file. Later, a second board file (`board-esp32-feather.yaml`) can sit next to the first one.

**Old code:** tag the tip of `webapp` as `legacy-micropython-1.9.3`, then delete the old code on the rewrite branch. Git history keeps it.

### Build and flash environment

**`compose.yaml`**: one pinned image, toolchains cached in a named volume:

```yaml
services:
  esphome:
    image: ghcr.io/esphome/esphome:2026.10.0   # bumped by Renovate PRs
    volumes:
      - ./firmware:/config
      - esphome-cache:/cache                    # entrypoint keeps PlatformIO toolchains here
    network_mode: host                          # OTA + mDNS from inside the container (Linux)
  esphome-usb:                                  # same, plus the serial port for the first flash
    extends: esphome
    devices:
      - ${PORT:-/dev/ttyUSB0}:/dev/ttyUSB0
volumes:
  esphome-cache:
```

**`Makefile`**: thin wrappers, so you never have to remember ESPHome command lines:

| Target | Does |
|---|---|
| `make validate` | `esphome config lightclock.yaml` |
| `make build` | `esphome compile` |
| `make flash-usb PORT=/dev/serial/by-id/usb-Silicon_Labs_CP2104…` | First flash over USB (Linux) |
| `make ota` | Update over Wi-Fi (`--device lightclock.local`, or `DEVICE=<ip>`) |
| `make logs` | Live device logs over Wi-Fi |
| `make test` | Host unit tests in a `gcc` container |
| `make backup PORT=…` | `uvx esptool read-flash 0 0x400000 backup/…bin`: dumps the old firmware before erasing |

**Machine-agnostic in practice:**
- **Linux** (Ubuntu 26.04, Docker or Podman):
  - Put your user in the `dialout` group.
  - Use the `/dev/serial/by-id/…` path, which survives replugging.
  - With rootless Podman, add `group_add: [keep-groups]`.
- **macOS / Windows:** Docker can't pass USB serial ports through. Build in Docker, then do the first flash one of two ways:
  - [web.esphome.io](https://web.esphome.io) in Chrome or Edge (WebSerial, nothing to install), or
  - `uvx esptool write-flash 0x0 firmware.bin` on the host.
- **After the first flash:** `make ota` works from any OS. If `.local` doesn't resolve inside the container, pass the IP.
- **No host Python setup:** `uvx` runs esptool without installing it; Docker or Podman, plus `make`, is the whole toolchain.

**Continuous integration (GitHub Actions)** in `.github/workflows/firmware.yml`:
- **On every push and PR:**
  - copy `secrets.example.yaml` → `secrets.yaml`
  - `esphome config` and `esphome compile` with the same pinned image
  - host unit tests
- **On a `v*` tag:** attach the firmware binary to a GitHub Release, so the README's "download from the releases page" is finally true.
- **Optional:** an ESP Web Tools page on GitHub Pages, so anyone can flash the latest release from the browser.
- **Version bumps:** Renovate opens a PR for each new ESPHome image and GitHub Action version, and CI compiles it before you merge. This is what keeps the project from going stale again.

---

## 4. Phases

Each phase ends with something you can check on the real device.

### Phase 0 — Bench check and backup (before touching the firmware)

**Steps**
1. **Back up the current flash:** `uvx esptool --port … read-flash 0 0x400000 legacy-backup.bin`. This is the only copy of the old firmware.
2. **Note the hardware** in `docs/hardware.md`, with photos:
   - **LED:** what kind of LED and driver (MOSFET or constant-current driver with a dim input), the supply voltage, whether full brightness is PWM high (the old code says yes).
   - **Fan:** 2-, 3- or 4-wire, its voltage, and how GPIO13 drives it (MOSFET? flyback diode?).
   - **MCP9808:** where it is mounted (on the LED heatsink?).
3. **Do a full `erase-flash`** later, before writing the new firmware. If the board crashes randomly afterwards, suspect worn flash (issue 2 above); a new HUZZAH is cheap.

**Done when:** the backup exists and the open questions in [section 5](#5-decisions-i-need-from-you) are answered.

### Phase 1 — Repo and toolchain

**Steps**
1. Tag `webapp` as `legacy-micropython-1.9.3`. Make the rewrite the default branch on GitHub (fast-forward `master`, or create `main`), so visitors no longer land on the 2017 code. Delete `asynchron`, which is fully merged.
2. Add `compose.yaml`, the `Makefile`, `.gitignore` (`secrets.yaml`, `.esphome/`), `secrets.example.yaml` and `renovate.json`.
3. Add the CI workflow (validate + compile).

**Done when:**
- `make build` works on your Ubuntu 26.04 machine from a fresh clone, with only Docker installed.
- CI is green.

### Phase 2 — Bring-up firmware (no alarm yet)

**Steps**
1. Write `board-huzzah-esp8266.yaml` and `base.yaml`, using the matching parts of the draft config: Wi-Fi, hotspot, captive portal, OTA, web UI, SNTP with `Europe/Berlin`, status LED, I²C, MCP9808, the two PWM outputs, the light, and a manually controlled fan.
2. Do the first flash over USB; do every later flash over Wi-Fi.

**Done when:**
- `http://lightclock.local` shows the UI.
- The light dims smoothly from the slider.
- The fan speed changes from the slider. Measure the fan's lowest start-up duty and set `min_power`; pick the PWM frequency by ear (whine vs. ticking).
- The temperature shows up live.
- Unplugging the router brings up the "Lightclock Setup" hotspot.

### Phase 3 — Alarm

**Steps**
1. Add `alarm.yaml`: the wake time entity, alarm on/off, the duration and brightness numbers, the sunrise script, and the Stop and Test buttons.
2. Move the start-minute calculation (midnight wrap) into `lightclock_logic.h` and unit-test it, covering 00:05 wake with a 20 min sunrise and 23:59 wake.
3. Stretch: weekday selection, snooze.

**Done when:**
- A wake time 3 minutes in the future, with a 2-minute sunrise, ramps up and is at full brightness *at* the wake time.
- The light stays on, then fades out.
- Settings survive a power cycle.
- The alarm doesn't fire before the time has synced.
- **Check the darkest part of the ramp in a dark room.** If the first step is too visible, see the ESP32 option in section 2.

### Phase 4 — Fan PWM and thermal protection

**Steps**
1. Add `thermal.yaml`: the fan curve, the minimum speed while the light is on, over-temperature with hysteresis, and the sensor-failure fail-safe.
2. Move the curve and hysteresis into `lightclock_logic.h` and unit-test them.

**Done when** each of these fault tests passes on the real device:
- **Sensor unplugged** → within 30 s the light goes off and the fan runs at 100 %.
- **Sensor warmed with a hair dryer** past the shutdown temperature → light off, fan 100 %. After cooling below shutdown − 2 °C, the light can be switched on again.
- **Normal sunrise at maximum brightness** → the fan settles at a sensible speed.

### Phase 5 — Webapp polish

**Steps**
1. Tune the stock UI: entity names, icons, groups and sort order. Add basic auth if wanted.
2. *Optional:* a custom clock-face page (big current time, wake-time picker, temperature), similar to the old one.
   - Use the `web_server` options `js_include` / `css_include` to bundle a small vanilla-JS or Preact page into the firmware.
   - It talks to ESPHome's REST and event-stream API, so the firmware doesn't change.
   - Do this only if the stock page isn't good enough in daily use.

**Done when:** setting the alarm from your phone takes two taps, with no CDN or internet access needed.

### Phase 6 — Docs and release

**Steps**
1. Rewrite the README: what it is, hardware, first flash on each OS, `make` targets, how updates work.
2. Tag `v1.0.0`; CI attaches the firmware to the release.
3. Optional: the ESP Web Tools page.

**Done when:** a fresh clone on any machine gets from zero to a flashed device using only the README.

### Phase 7 (optional) — ESP32 Feather

**Steps**
1. Add `board-esp32-feather.yaml` (`ledc` outputs, new GPIO numbers; optionally the fan tach on a `pulse_counter`).
2. Rewire, flash, and rerun the Phase 3 and 4 checks.

---

## 5. Decisions I need from you

1. **ESPHome or MicroPython?** I recommend ESPHome for the reasons in section 2.
2. **Fan hardware:** 2-, 3- or 4-wire, and how is it driven from GPIO13? This decides the PWM frequency and whether the ESP8266 is enough.
3. **LED driver type,** and whether you would swap the board if the low end of the sunrise looks steppy.
4. **Time zone:** I assumed `Europe/Berlin` from `utc_delay: 1`.
5. **Home Assistant:** do you run it? If so, enable ESPHome's native API and get history, automations and a phone app for free.
6. **Old code:** only a git tag (recommended), or keep a `legacy/` folder on the default branch?

---

## Appendix A — MicroPython alternative

If you choose MicroPython, the plan keeps the same phases, with these pieces:

- **Firmware:** MicroPython v1.29.0, board `ESP8266_GENERIC`.
  - The app is frozen into the firmware via a `manifest.py`: `include("$(PORT_DIR)/boards/manifest.py")`, then `package("lightclock")` and `require(...)`.
  - Built with the upstream-recommended `larsks/esp-open-sdk` Docker image.
  - On an ESP32, use the official `espressif/idf` image instead.
- **Structure:** one `asyncio` loop with tasks for the alarm, thermal, NTP and web, instead of timer callbacks next to a web server.
  - Pure logic (schedule, fan curve) is kept free of `machine` imports and tested with pytest on CPython.
  - The MicroPython Unix port in Docker checks compatibility.
- **Web:** [Microdot](https://github.com/miguelgrinberg/microdot), async, with a JSON API, plus a small static front-end (vanilla JS or Preact + htm), gzipped at build time and served from flash. No CDNs.
- **Tooling:** `uvx mpremote` (replaces ampy) and `uvx esptool`; `mpremote mount .` for edit-and-run without reflashing.
- **Local name:**
  - ESP8266: no mDNS. Set `network.hostname("lightclock")` before connecting and rely on the router's DNS. Reports suggest this is unreliable on the ESP8266 port; verify early.
  - ESP32: mDNS is built in.
- **You build yourself:** daylight saving time rules (an EU-only rule is ~15 lines), the hotspot plus a captive portal, safe settings storage (write-and-rename JSON, only on change), and a watchdog.
- **PWM on the ESP8266:** 1–1000 Hz, **one frequency shared by the LED and the fan**, 10-bit duty.
- **Effort:** clearly more than ESPHome, and it is the same kind of code that went stale last time.

## Appendix B — Draft config

[`docs/lightclock.draft.yaml`](lightclock.draft.yaml) is a single-file draft of Phases 2–4 for the HUZZAH (pins from the old `config.json`). It passes `esphome config` with ESPHome 2026.10.0. It still needs:

- a compile and a flash on real hardware
- fan PWM frequency and `min_power` tuned on the bench
- splitting into packages plus the logic header (Phases 1–4)
