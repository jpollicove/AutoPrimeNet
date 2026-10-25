# GUI development and validation

This document records development evidence for the optional Python 3.15.0-or-later
GUI. See [GUI.md](GUI.md) for environment requirements and the agreed behavior.
The complete desktop GUI is not available yet.

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
tests before exposing them in the GUI. The next prototype should address Tk/tray
event-loop integration and controlled backend initialization.

## Original-file changes

None. All GUI work so far is in newly added files. Existing upstream source,
README, pyproject.toml, and workflows remain unchanged. No files have been staged
or committed by the agent.
