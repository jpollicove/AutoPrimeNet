# GUI development and validation

This document records development evidence for the optional Python 3.15.0-or-later
GUI. See [GUI.md](GUI.md) for environment requirements and the agreed behavior.
The complete GUI is not available yet; the desktop foundation now includes local
configuration forms, validated persistence, scheduled monitoring process control,
and read-only previews. Live PrimeNet monitoring remains unvalidated.

## Milestone: environment and offline backend prototype

Validated on October 24, 2026 on the development Apple Silicon macOS machine:

| Component | Observed version/result |
| --- | --- |
| Homebrew Python / `.venv` interpreter | 3.15.1 |
| Native Tcl and Tk | 9.1.0 |
| Requests | 2.34.2 |
| urllib3 | 2.8.0 |
| certifi | 2026.10.23 |
| charset-normalizer | 3.5.2 |
| idna | 3.20 |
| `python -m pip check` | No broken requirements |
| Tk window smoke check | A real ttk window mapped, processed events, and closed successfully |
| Offline prototype suite | 10 tests passed |
| Existing CLI smoke checks | `autoprimenet.py --help` and `gimps_status.py --help` both passed |

Homebrew installed `python-tk@3.15` and its native `tcl-tk` and `libtommath`
dependencies. Requests and its dependencies were installed only inside `.venv`.
The graphical smoke check aborted inside the restricted tool sandbox; it passed
when run with access to the macOS graphical session. That first failure was not
treated as a successful GUI test.

Python 3.15.0 itself has not been runtime-tested. Windows, Linux, FreeBSD, tray
integration, and the full GUI have not been tested. Tested dependency versions
above are evidence from one environment, not cross-platform pins or a lockfile.

## Offline bridge

`gui_backend.py` is a short-lived private child-process prototype. It accepts one
JSON request on standard input and returns one JSON response on standard output.
It currently supports only:

- `capabilities`: reads the audited parser inventory without executing upstream
  definitions or startup. Includes hidden/inverse flags and automatic help.
- `parse_work`: executes the reviewed upstream definition prefix in a private
  namespace, then reuses `parse_assignment()` and `output_assignment()` to inspect
  supplied work lines. It does not open the user's work files.
- `validate_config`: initializes the existing argument parser and merges proposed
  configuration text in memory, reuses decryption and `check_options()`, and
  returns only non-secret effective settings. It does not write configuration,
  acquire locks, run prompted setup, register the computer or enter the scheduler.

The source SHA-256 must exactly match the reviewed backend. Source drift fails
closed before execution. An update to that guard requires a new integration
review, not just replacing the hash. No copy of the PrimeNet parser, serializer,
or API implementation is maintained by the bridge.

The executed prefix initializes native capabilities, locale, and an HTTP session;
it is therefore isolated from the Tk process. Normal CLI startup, disk-based
configuration loading, instance locking, file logging, prompted setup and the
scheduler are not executed. Configuration validation initializes the parser and
configuration in memory only. The bridge explicitly closes its session afterward.

Python socket audit guards deny DNS resolution, connections, and datagram sends
for these offline operations. This is a precaution around trusted, hash-checked
upstream code, not a general security sandbox for arbitrary code. No mutation or
server-operation request is accepted. Diagnostics are suppressed and failure
responses are sanitized to avoid reflecting secrets or corrupting the JSON stream.

Parsing is not mathematical validation or reservation confirmation. Even a line
with an AID is returned with `server_confirmation: "unverified"`.

## Running the tests

From the repository root, after following the environment setup in GUI.md:

```sh
.venv/bin/python -B -m unittest discover -s tests -p 'test_gui_*.py' -v
.venv/bin/python -m pip check
```

On Windows PowerShell:

```powershell
.\.venv\Scripts\python.exe -B -m unittest discover -s tests -p 'test_gui_*.py' -v
.\.venv\Scripts\python.exe -m pip check
```

The standard-library tests launch the bridge in isolated child processes. They
cover parser inventory, the ten work-file formats, canonical round trips,
unverified AIDs, malformed/multiline input, protocol/action rejection, source
drift, sanitized JSON errors, blocked network access, absence of generated
runtime files, and byte preservation of the upstream backend. No GIMPS account,
real assignments, live server, or test email is used.

In VSCode, use **Python: Select Interpreter** from the Command Palette and select
the repository's `.venv/bin/python` (Windows: `.venv\Scripts\python.exe`). The
repository does not require a committed VSCode settings file. The commands above
also work without activating the environment because they use its executable
directly.

The initial milestone was evidence for read-only helper reuse only. It did not establish correct
configuration precedence, encryption, lock coordination, registration, selective
server actions, process shutdown, or scheduler integration. Those require later
tests before exposing them in the GUI.

## Milestone: read-only desktop foundation and native tray helper

The new `autoprimenet_gui.py` launcher provides eight tabs, directory/configuration
selection, local identity/worker/disk/assignment previews, GUI activity, and a
searchable CLI reference. `gui_config.py` reads bounded, case-sensitive snapshots
without writing or decrypting. `gui_controller.py` serializes background jobs,
uses timeout-bounded private backend requests, and owns the tray helper process.

`gui_tray.py` uses pystray in a separate process to keep its native event loop
independent of Tk. The optional dependency file resolves to pystray 0.19.5,
Pillow 13.0.0, six 1.17.0 and PyObjC core/Cocoa/Quartz 12.2.2 on the development
Mac. All were installed inside `.venv`. Runtime readiness is established before
withdrawing the window; helper failure restores a hidden window. Taskbar/Dock
minimization is the fallback. Linux/FreeBSD tray integration is not enabled yet.

Validation on the same macOS/Python 3.15.1/Tk 9.1 environment:

- 18 core tests passed: the previous 10, plus read-only configuration/worker
  discovery, symlink conflict detection, PRPLL indexed files, exclusion of secret
  settings from snapshots, invalid values/paths, bounded request size, job
  serialization, and backend failure reporting.
- Five opt-in real-desktop tests passed: tab navigation/search/workspace loading
  and repeated refresh, tray hide/programmatic restore/helper failure recovery,
  reachable taskbar/Dock fallback without tray support, clean shutdown of a healthy
  tray helper, and macOS native Quit dispatch through application cleanup.
- The user confirmed the requested manual macOS menu-bar test passed on October
  24, 2026: closing the window, reopening it through the menu-bar icon, and Quit.
  This supplements the programmatic event-delivery checks above.

Run the desktop tests only in an accessible graphical session. On macOS:

```sh
AUTOPRIMENET_GUI_DESKTOP_TESTS=1 .venv/bin/python -B -m unittest discover -s tests -p 'test_gui_window.py' -v
```

On Windows PowerShell:

```powershell
$env:AUTOPRIMENET_GUI_DESKTOP_TESTS = '1'
.\.venv\Scripts\python.exe -B -m unittest discover -s tests -p 'test_gui_window.py' -v
Remove-Item Env:AUTOPRIMENET_GUI_DESKTOP_TESTS
```

The default suite skips desktop tests to avoid initializing Tk in headless or
restricted sessions. The earlier crash report confirms SIGABRT during macOS
application registration, with `TkMacOSXDisplayChanged` / `TkpOpenDisplay` in the
stack. This is consistent with the restricted-session failure, not proof of a
defect in application logic; subsequent desktop-access tests passed. Do not run
graphical checks in the same restricted context that caused that abort.

No monitoring, server mutation, notification delivery, configuration persistence,
or engine management is available in this milestone. Full initialization parity,
editable forms and process safety for an actual monitoring backend remain work
for later milestones. No live PrimeNet operations were performed.

## Milestone: configuration forms, safe persistence and offline initialization

Validated October 25, 2026 on macOS with Python 3.15.1 and Tk 9.1.0: **46 tests
passed**, comprising 39 core tests and seven opt-in desktop tests. No tests were
skipped in that combined desktop-enabled run. The dependency check also passed.
No new dependencies were required for this milestone.

`gui_options.py` defines typed metadata for the first fourteen configuration
fields. `gui_forms.py` presents them in the main tabs and adds program selection,
worker-count editing and a per-worker dialog. Saving is an explicit user action;
unsaved changes can be retained when cancelling reload or Quit. Forms and path
selectors are disabled during background operations to prevent competing edits.

`gui_config.py` now retains a loaded version fingerprint and patches only changed
single-line keys. It preserves comments, BOM/CRLF, unknown keys/sections,
untouched interpolation and ciphertext. New literals escape percent signs. It
uses the CLI-compatible instance lock, checks for external edits, writes a
private backup, flushes a same-directory temporary file, and atomically replaces
the target. Read-only/symbolic-link targets and known live-data filename conflicts
are refused. No-op saves preserve exact bytes without a backup. Old worker
sections remain when reducing the active count; program/hours changes reset the
same cached estimate keys as setup.

Offline validation executes the reviewed source's parser declarations and reuses
`merge_config_and_options()`, `decrypt()` and `check_options()`. It resolves
program-dependent defaults and support sets without running the CLI startup
block or scheduler. Python 3.15's lazy-import bindings required using the bridge's
standard-library ConfigParser rather than directly calling a potentially
unresolved dictionary binding. The audited upstream source itself was unchanged.

New verification covers:

- Effective defaults for all eight computation programs, and comparison of
  key effective values with the normal CLI's status path in a disposable
  workspace under a network-blocking audit hook.
- Real upstream OpenSSL encryption/decryption using a synthetic proxy password;
  an unrelated edit retains ciphertext and the original backup without storing
  the decrypted value. A separate fake-ciphertext formatting test mocks validation
  explicitly and is not counted as evidence of decryption.
- Initial creation, private-file modes, backups, comments/unknown/multiline
  preservation, literal percent paths, multiple workers and preferences,
  external edits (including restored mtime), lock contention, read-only files,
  failed replacement, temp cleanup, invalid input and engine-change resets.
- Real Tk interaction through the application methods for invalid input,
  successful save/reload, unsaved-change cancellation, and all prior tray tests.

The atomic rename is not a compare-and-swap against unrelated editors that ignore
the shared lock. Avoid concurrent writers; narrow races with such editors remain
possible after the final version check. Extended ACL/xattr preservation, network
filesystems, Windows/FreeBSD/Linux execution and Python 3.15.0 runtime validation
remain untested. Multiline values being edited are deliberately refused. Full
credential editing, registration, network initialization/proxy setup, monitoring,
selective server operations, notifications and complete graphical parity remain
later work. No live PrimeNet operations or real assignment changes occurred.

## Milestone: scheduled monitoring lifecycle

Validated October 25, 2026 on the same macOS/Python 3.15.1/Tk 9.1.0 environment:
**59 tests passed**, comprising 51 core tests and eight real-desktop tests. All
monitoring cycles in these tests are simulated or replaced with owned sleeping
test children; no live PrimeNet cycle, assignment change or notification send was
performed. The older CLI status comparison remains network-blocked.

`gui_monitor.py` is a private scheduled runner. It validates the selected
configuration offline, holds `~gui-monitor.lock`, and runs unmodified CLI passes
with `--timeout 0` at the configured check-in interval. This avoids leaving
daemon proof/result workers active when stopping at a cycle boundary. It is a
scheduled mode only, not implementation of native immediate file watching or
independent polling/check-in cadences. A completed initial registration can be
followed by a normal work pass; tests verify this glue with stubbed subprocesses,
not real registration.

Each Start preserves a private, byte-for-byte configuration backup before the
CLI can normalize the file. Tests verify the backup contents and POSIX private
file permissions. This backup does not undo server operations or checkpoints.

`MonitorController` owns the helper process, serialized control writes, lifecycle
events and restart generations. A ready/begin handshake prevents a Stop during
offline preflight from starting a server cycle afterward. The helper reads control
commands without buffered stdin read-ahead or buffered-reader shutdown locks.
Parent-pipe EOF requests a graceful stop. A confirmed force-stop command targets
only the CLI child created by this helper. No process discovery, arbitrary PID
killing, unreservation, or automatic stale-lock removal is implemented.

Dashboard Start/Stop/Force controls are integrated with tray state and start/stop
menu actions. Closing retains the GUI/runner; confirmed Quit waits asynchronously
for shutdown. Settings/path editing is disabled while monitoring, and the save
layer independently checks the runner lock. The CLI's native lock protects every
pass against independent CLI processes; the GUI-only lock does not prevent an
independent CLI from starting during a scheduled idle interval. A subsequent
native lock conflict stops the runner rather than killing the foreign process.

New tests cover duplicate starts/runners, native lock conflicts, changed config,
save exclusion, startup cancellation, idle/running-cycle Stop, parent EOF,
nonzero-cycle failure, initialization follow-up, private argv, and forced stopping
of an owned test child while an unrelated test process remains alive. The real
desktop test hides the window to the actual macOS menu-bar helper while a
simulated runner remains active, then tests cancelled and confirmed Quit.

Tray state updates/activation/shutdown now use PyObjC's native main-thread
dispatch on macOS; the Tk event loop remains in the GUI process. No new package
dependencies were needed. Native menu clicks for the new Start/Stop actions have
not been manually tested, and Windows/Linux/FreeBSD and Python 3.15.0 runtime
validation remain pending.

CLI stdout/stderr is not streamed into the GUI, avoiding pipe backpressure and
credential exposure. Detailed diagnostics remain in the existing rotating CLI
log; GUI cycle events are bounded and explicitly do not confirm server acceptance.
Long uploads, retries, or stale sidecar locks can delay graceful Stop. Force-stop
risks and scheduled-mode limitations are documented in GUI.md. The next work is
complete graphical operation coverage, immediate-watch lifecycle and safe live
validation after review; this milestone does not establish full parity.

## Original-file changes

None. All GUI work so far is in newly added files. Existing upstream source,
README, pyproject.toml, and workflows remain unchanged. No files have been staged
or committed by the agent.
