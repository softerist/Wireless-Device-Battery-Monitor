# Last-known battery readings

The resident app separates a battery value from the result of the latest query.
A confirmed percentage remains useful during idle periods, but its age and
reason must remain visible. A missing response does not prove that the keyboard
or mouse is asleep.

## States and reasons

- **Current** — `72%`: A confirmed reading remains current under the configured
  polling interval.
- **Last known** — muted `72%`: A confirmed historical reading exists, but no
  current reading is available.
- **Unknown** — `?`: No eligible confirmed reading exists for this binding.
- **Disconnected** — hidden, or a dimmed `×` when enabled: The driver explicitly
  reports an offline link.

The tooltip and resident JSON distinguish the reason separately:

- **`none`**: A current confirmed reading exists.
- **`sleep_reported`**: The protocol explicitly reported sleep.
- **`no_response`**: The battery query received no response; sleep is possible
  but unproven.
- **`unavailable`**: The protocol explicitly reported that a battery value is
  unavailable.
- **`query_failed`**: A query or transport operation failed.
- **`invalid_response`**: A received response failed validation.
- **`not_yet_read`**: No battery reading has been confirmed yet.
- **`reading_aged`**: A prior reading aged beyond the current-reading window
  without a newer outcome.
- **`disconnected`**: An explicit offline result is present.

For example, an unavailable reply after a confirmed 72% report produces a
last-known 72%, with reason `unavailable`. The same reply before any confirmed
report produces `?` with that reason. Neither case is labeled asleep. A
confirmed 0% is a valid value and remains eligible for historical display.

Only current readings carry active charging, full-charge, or low-battery
presentation. Historical values retain their percentage and receipt age without
implying present charging or generating a current low-battery warning.

## Display preferences

**Dim last known readings** controls historical text brightness. Its persisted
setting remains `DimSleepingDevices`, so existing preferences remain compatible.
Turning dimming off changes brightness only: the state, reason, and receipt age
remain historical. Existing hidden-device, last-known hiding, ordering, and
disconnected-device preferences still apply.

## Retention and identity

The extended retention policy applies to Lofree Flow 2 receiver bindings and
canonical device-reported Logitech MX Master 4 bindings. Other drivers retain
their existing retention limits. Flow 2
battery telemetry still requires the documented LF2-v1 firmware protocol; this
presentation change does not update firmware.

Confirmed values stay in memory while the matching receiver binding remains
present, without an arbitrary idle expiry. Receiver removal clears that binding.
Reconnecting requires a new confirmed reading before a value becomes available.
Failed enumeration does not establish physical removal. Query failures cannot
refresh a reading's receipt time.

Logitech ping errors are scoped to the affected receiver slot. An error from an
empty slot cannot change another device's query reason. A coarse legacy
estimate cannot seed a newly established raw-percentage binding; that binding
starts unknown until a validated percentage arrives.

The Flow 2 identity is bounded by the physical receiver and its paired keyboard:
the protocol does not establish an independent keyboard serial identity.
Re-pairing a different keyboard to the same continuously connected receiver
cannot be detected reliably. Model names alone are not physical identities.
Logitech bindings include a positive unit ID when available, plus receiver and
slot. Without a unit ID, the fallback identifies that receiver slot only. A
positively identified replacement in the same slot invalidates the superseded
binding. A failed model or unit lookup cannot silently downgrade an established
binding or borrow its percentage.

The cache is not restored across app restarts. A fresh process begins without
historical readings, even if an old snapshot file remains after an abnormal
exit. A new confirmed report, including the same percentage as before, refreshes
receipt time. Snapshot generation does not.

## JSON contracts

The running app's `%LOCALAPPDATA%\WirelessBattery\status.json` retains the
version-1 envelope and existing field types. Each device additionally contains
`state`, `reason`, `last_reading_at` (UTC or `null`), and `reading_age_seconds`
(integer or `null`). The percentage, `live`, `charging`, and `asleep` fields
follow the same resolved state as the app. `low` is false for historical data.
The legacy `connected` flag remains a driver connection indication rather than
proof of wakefulness.

`updated` describes snapshot generation. `last_reading_at` describes the
confirmed report, and `reading_age_seconds` is its age at snapshot generation.
Rewriting the file does not invent a new report. Unknown receipt metadata
remains `null` rather than a fabricated timestamp.

The independent `--json` command keeps its existing array contract. It polls
hardware once and does not consult the resident cache. It can therefore report
an unavailable battery while the running app displays a historical value. See
[YASB integration](yasb.md) for adapter limitations.

## Local checks and rollback

Use the existing x64 MinGW toolchain on `PATH`; no dependency installation is
required by these commands. From the repository root, the targeted serializer
check is:

```powershell
g++ -std=c++17 -O2 -Wall -Wextra -Werror `
  tests/status_json_test.cpp src/status_json.cpp `
  -o build/status_json_test.exe
if ($LASTEXITCODE -ne 0) { exit $LASTEXITCODE }
.\build\status_json_test.exe
```

The serializer cases cover historical 0%, each reason, offline output,
suppression of historical live/charging/low flags, additive legacy fields, and
unchanged report metadata across snapshot updates. These tests do not establish
hardware behavior.

The complete Windows checks keep assertions enabled:

```powershell
make test-last-known VERSION=0.0.0 CHANNEL=dev
if ($LASTEXITCODE -ne 0) { exit $LASTEXITCODE }
make test-sonix VERSION=0.0.0 CHANNEL=dev
if ($LASTEXITCODE -ne 0) { exit $LASTEXITCODE }
make VERSION=0.0.0 CHANNEL=dev
if ($LASTEXITCODE -ne 0) { exit $LASTEXITCODE }
make test VERSION=0.0.0 CHANNEL=dev
if ($LASTEXITCODE -ne 0) { exit $LASTEXITCODE }
git diff --check
Get-FileHash ./BatteryHub.exe -Algorithm SHA256
```

The regression fixtures are retained in source:

- [Battery state policy](../tests/battery_state_test.cpp): cache lifecycle,
  removal, slot replacement, clock age, effects eligibility, hide/dim options,
  notification recovery, and raw-report smoothing exclusion.
- [HID inventory](../tests/hid_inventory_test.cpp): the production enumeration
  decision path with partial, empty, failed, and unresolved-identity scans.
- [Flow 2 poller](../tests/sonix_driver_test.cpp): LF2-v1 validation, unavailable
  replies, unrelated traffic, deadlines, multiple receivers, and legacy SONiX.
- [Logitech poller](../tests/logitech_idle_test.cpp): idle omission, slots and
  unit IDs, unresolved model lookup, supported 0%, and percentage capability.
- [Resident JSON](../tests/status_json_test.cpp): shared presentation fields and
  stable report timestamps across snapshot generations.

In an already provisioned Linux environment, the native path is:

```sh
cmake -S . -B build-last-known-linux -G Ninja \
  -DWB_BUILD_LINUX=ON -DWB_BUILD_WINDOWS=OFF -DWB_BUILD_TESTS=ON \
  -DWB_VERSION=0.0.0 -DWB_CHANNEL=dev -DCMAKE_BUILD_TYPE=Debug
cmake --build build-last-known-linux
ctest --test-dir build-last-known-linux --output-on-failure
```

This path needs CMake, Ninja, GTK4, and Cairo development packages already
available. Portable policy tests and driver compilation alone do not establish
a passing Linux application build or Linux UI parity.
The CMake test directory keeps assertions enabled in Release builds as well;
production targets retain their normal build flags.

## Recorded local validation

### Final review-correction checkpoint

The 2026-11-22 follow-up validates the Windows review-correction executable,
`BatteryHub.exe`, with SHA-256:

```text
6E4C8A6D137ED8B549CCFB74AAF2E81071FAB4695A655B27C785DF3999063F42
```

Its application sources are commit
`bc342dbb2a370ffc57c2dae1fa85291000d46fc5`. The Windows application build and
75-executable canonical suite pass. The final strengthened Logitech fixture
also passes the focused target with assertions enabled and
`-Wall -Wextra -Werror`. Two warnings remain in the unchanged Soundcore fixture.
Independent semantic review has no remaining introduced findings.

The final executable's resident snapshot was observed 134 times across 33
distinct snapshot updates over approximately 11 minutes. Flow 2 retained 100%
and MX Master 4 retained 95% during unanswered queries. Historical receipts
remained fixed across repeated updates while their ages increased; exported
ages agreed with snapshot UTC age within one second. These readings used
`last_known` and `no_response`, without claiming protocol-reported sleep or
active charging, live status, or low-battery status. Both devices subsequently
returned fresh reports. Widget values and cleared historical hover wording
were manually confirmed.

Receiver removal was confirmed by Windows inventory: the keyboard entry
disappeared while the mouse remained present. Reconnection produced a new
current 100% keyboard report, dated after removal. The Windows PnP instance
set changed after reconnection, so this sequence does not establish stable
instance identity. Startup began without cached keyboard history and obtained
a valid report before retaining it. Early fresh reports were observed without
assuming deliberate input; the later visual/wake check and wireless typing
after reconnection were explicitly confirmed. Preferences, startup registration,
and the battery-history file hash remained unchanged.

The native Linux Debug configure, complete Ninja application build, and all
five CTest tests now pass. The first full link exposed six existing pollers
missing from the CMake driver library: Zebronics, Keychron, Lamzu, Aula,
Ninjutso, and Rapoo. The corrected source list registers their existing files
without changing protocol code, polling order, or Windows build inputs. A
separate independent review of that correction passes. The application sources
and Windows executable above remain unchanged by this Linux build correction.

The Linux build used Ubuntu 24.04 WSL, GCC 13.3, CMake 3.28.3, Ninja 1.11.1,
GTK4 4.14.5, and Cairo 1.18.0. The build prerequisites were installed before
this final gate. The commands in the preceding section reproduce the native
configure/build/CTest path. Linux GUI operation, physical-device parity,
packaging, and Flow 2 support on Linux are not established by that result.

This focused final check does not repeat the earlier themes, dimming, tray,
cable, extended idle, and Windows suspend/resume matrix below. Explicit
unavailable/error injection, multiple physical receivers, and protocol-reported
sleep retain synthetic coverage. Commands, exit records, failed and corrected
build logs, source identities, and observation results are retained locally;
generated binaries, raw host identifiers, and device backups are not committed.

### Earlier Windows checkpoint

The hardware observations below describe the preceding Windows candidate,
`BatteryHub.exe`, with
SHA-256:

```text
D3FE7D130906ED0D6DF222BC6F54D7CFD57143527CE45598AF551155A4D7F6E4
```

The local checkpoint has the following evidence:

- The Windows application build, targeted regressions, and complete canonical
  suite pass. The full suite executes 75 test programs. Two compiler warnings
  remain in the unchanged Soundcore test. Independent source review has no
  remaining findings.
- Controlled awake-idle-wake observations retain Flow 2 at 100% and MX Master 4
  at 95%. The controlled idle trial reaches historical ages of 739 and 247
  seconds respectively, without claiming protocol-reported sleep.
- Manual widget, tray, hover, Light/Dark, and dimming checks pass. Historical
  readings retain their reasons and ages; new reports clear historical hover
  wording. The original preferences and startup entry are preserved.
- Confirmed receiver removal removes its entry. After cable operation, the same
  Windows receiver identity returns with a new 100% report; the recorded return
  does not reuse the earlier receipt. Wired operation exports no keyboard
  battery value. Simultaneous wired and receiver presence was not established
  by that physical sequence.
- Two Windows S3 cycles are recorded. Across the first, Flow 2 keeps the same
  receipt while its age advances from 206 to 759 seconds. The second cycle
  retains the mouse receipt with a post-resume age of 705 seconds, consistent
  with its UTC age within integer-second precision. The first wake uses Flow 2;
  the second uses another keyboard. Flow 2 subsequently returns a new report
  without deliberate input in the second cycle. Both devices recover to fresh
  readings, with retained values and correct hover text manually confirmed.

A brief mouse-entry omission during USB receiver handling remains unclassified:
the same applied poll's inventory and cache decision were not captured.
Focused source review found no timeout or incomplete-scan deletion path for an
established binding. One complete inventory reporting its transport absent is
sufficient to invalidate that binding; there is no multi-poll removal debounce.

At this earlier checkpoint the complete Linux application build was **not
run**: no suitable pre-provisioned environment was found and dependencies were
not installed. The final checkpoint above closes that build/test gap.
Portable Linux policy tests and driver/backend compilation pass.
Release-equivalent compilation and execution of all five portable fixtures
also pass with assertions enabled. These checks do not replace the native
CMake build and CTest. Explicit unavailable/error injection, multiple physical
receivers, and protocol-reported sleep have synthetic coverage rather than
physical validation at this
checkpoint. Logs, commands, source diff, manifest, and observation helpers are
retained in the local evidence package; no release is published by this check.

## Further hardware checks and rollback

For a local candidate, check awake operation, ordinary idle, reported sleep when
available, explicit unavailable/error outcomes, receiver removal/reconnection,
app restart, and wake with the same percentage. Compare the display, tooltip,
and snapshot. Confirm that brightness and hiding preferences affect presentation
without changing the reading's age or meaning. Hardware checks requiring
unavailable/error injection should be recorded separately from observed idle.

Keep the previous executable and any temporary preference backup before testing.
To roll back, exit the candidate, restore temporary preferences if changed, and
reopen the previous executable. Installing a candidate, changing startup
registration, or updating firmware is a separate action.
