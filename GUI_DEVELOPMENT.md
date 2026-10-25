# GUI development and validation

This document records development evidence for the optional Python 3.15.0-or-later
GUI. See [GUI.md](GUI.md) for environment requirements and the agreed behavior.
The complete GUI is not available yet; a read-only desktop foundation is runnable.

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

The source SHA-256 must exactly match the reviewed backend. Source drift fails
closed before execution. An update to that guard requires a new integration
review, not just replacing the hash. No copy of the PrimeNet parser, serializer,
or API implementation is maintained by the bridge.

The executed prefix initializes native capabilities, locale, and an HTTP session;
it is therefore isolated from the Tk process. The CLI parser/startup block,
configuration loading, instance lock, file logger, prompted setup, and scheduler
are not executed. The bridge explicitly closes its session after parsing.

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

This is evidence for read-only helper reuse only. It does not establish correct
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

## Original-file changes

None. All GUI work so far is in newly added files. Existing upstream source,
README, pyproject.toml, and workflows remain unchanged. No files have been staged
or committed by the agent.
