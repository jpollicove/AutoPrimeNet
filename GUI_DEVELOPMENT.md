# GUI development and validation

This document records development evidence for the optional Python 3.15.0-or-later
GUI. See [GUI.md](GUI.md) for environment requirements and the agreed behavior.
The complete GUI is not available yet; the desktop foundation now includes local
configuration forms, encrypted password replacement, validated persistence,
scheduled monitoring process control, confirmed one-shot commands and read-only previews. Live PrimeNet
monitoring remains unvalidated.

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
- `protect_credentials`: validates configuration offline and strictly encrypts
  explicit SMTP/proxy password replacements using the original backend. Returns
  only selected ciphertext and the hardware GUID needed to retain its binding.
  It does not write files; the existing save layer patches and validates the
  result before atomic replacement.

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

## Milestone: advanced assignment configuration

Added ten graphical settings for trial-factoring bit limits and target bits,
P−1 tests saved/retry multiplier/optimal bounds, ECM bounds/curve limits, and
LL/PRP conversion directions. Existing exponent and certification controls now
share a scrollable Advanced dialog. Dialog edits are staged separately: Cancel
discards them, Apply updates the main form, and Save uses the existing offline
backend and atomic persistence path. Applying while monitoring or local work is
active is refused. Mutually exclusive conversions are rejected in the dialog
and again by the original backend at save time. No new ranges beyond the
original backend's rules were imposed on the added numeric options.

The CLI reference maps all current configuration fields to their tab and label,
searches those labels, and supports double-click/Return navigation. Mapped
advanced rows open and focus the relevant control; unmapped entries remain
reference-only. This does not establish complete CLI parity.

Validated October 25, 2026 on macOS/Python 3.15.1/Tk 9.1.0: **62 tests passed**,
including nine desktop tests. Added tests round-trip the new settings through
the real offline backend, verify conflicting/invalid settings cannot alter the
file or create backups, and exercise dialog cancellation, application, reference
navigation and saving. All server cycles remain simulated or network-blocked.
No live assignments were changed and no original upstream files were edited.
Other platforms remain untested. Notifications, full operation coverage and
immediate-watch lifecycle remain future milestones.

## Milestone: local notification configuration

Added Notifications forms for send enablement, From address, recipients, SMTP
server, SMTP username, SSL/TLS and StartTLS. Field metadata now identifies its
INI section and option independently of the form key, so `[Email] username`
cannot collide with `[PrimeNet] username`. Existing field mappings still use
the same PrimeNet keys. CLI reference navigation includes the new mapped flags.

Save remains an offline, validated configuration operation. Changed recipient
lists normalize comma separators and surrounding spaces; unchanged values retain
their original formatting. SSL/TLS and StartTLS cannot both be enabled. The
backend checks email syntax, server scheme and required sender/server fields.
Send enablement uses the original backend's `[Email] send` flag; it does not
introduce a separate notification implementation. Blank boolean fields remove
explicit overrides and retain existing INI/backend inheritance semantics.

Validated October 25, 2026 on macOS/Python 3.15.1/Tk 9.1.0: **66 tests passed**,
including ten desktop tests. New coverage checks cross-section identity,
recipient normalization, TLS conflicts, invalid address/server rejection,
disable/reset behavior, preservation of plaintext and real upstream-encrypted
SMTP credentials, GUI save/reload and reference navigation. No SMTP message,
server discovery or live PrimeNet cycle was performed. No original upstream
files changed, and no dependencies were added.

Password editing, test-message sending, automatic SMTP discovery and per-event
notification controls remain unfinished. Saving other notification fields does
not encrypt plaintext passwords or replace the password for a changed SMTP
account. Those limits are documented in GUI.md. Full operation coverage,
immediate-watch lifecycle and other-platform validation remain future work.

## Milestone: hardware and resource configuration

Added eleven registration settings: processor model/features/frequency, total
physical memory, maximum stage 2 memory, proof residue disk limit, L1/L2/L3
cache sizes, physical cores and threads per core. They live in a scrollable
dialog opened from PrimeNet and are mapped in the searchable CLI reference.
The advanced assignment dialog and hardware dialog share the same staged
Apply/Cancel implementation. A modal dialog cannot apply edits while monitoring,
local operations or Quit are active.

Blank fields preserve INI inheritance and the backend's automatic detection or
defaults without materializing machine-specific values during an unrelated save.
The case-sensitive `memory` and `Memory` options remain distinct. Save uses the
original backend's type conversion and maximum-versus-total-memory check; the
GUI also rejects malformed/nonfinite numeric input. Hardware-only edits preserve
existing timing estimates, matching the absence of an upstream setup reset for
these settings. No computation-program settings or system resource limits are
changed by the forms.

Validated October 25, 2026 on macOS/Python 3.15.1/Tk 9.1.0: **69 tests passed**,
including eleven desktop tests. Added coverage includes all hardware fields,
case-sensitive memory keys, no-op saves, clearing overrides, estimate retention,
malformed input, backend memory conflicts, dialog cancellation, CLI-reference
navigation and GUI save/reload. Existing advanced-dialog tests pass after reuse.
No registration, hardware update, SMTP send or live PrimeNet cycle was performed.
Original upstream files remain unchanged; no dependencies were added.

Detected-hardware preview, automatic GPU selection, standalone graphical
registration, credential editing, selective server operations, immediate-watch
lifecycle and other-platform validation remain unfinished. GUI.md documents
units, detected/default behavior and the distinction between reported resource
settings and actual computation-program configuration.

## Milestone: connection settings and encrypted password management

Added proxy type/server/username and backend encryption policy in a Connection
dialog under Settings. Added masked SMTP/proxy password management under Settings
and Notifications, with CLI reference navigation for the corresponding flags.
Keep preserves pending edits, Replace stages a new secret, Remove stages deletion
of plaintext and encrypted entries, and Use saved cancels a pending change.
Cancel discards only that dialog's new edits. No saved password is loaded into
an entry; entries are cleared on Apply/Cancel. Successful Save or workspace
reload clears pending replacements. The removed legacy account-password feature
is intentionally not implemented.

The new offline `protect_credentials` operation reuses the original `encrypt()`
with strict encryption and limits its working mapping to explicitly replaced
passwords. Selected entries are removed from a private candidate before
preflight, allowing a damaged selected credential to be removed without first
decrypting it. No plaintext fallback is allowed. The existing atomic save layer
patches only returned ciphertext and any newly needed hardware GUID, then uses a
fresh offline validation process to decrypt/check the final candidate before
writing. Existing hardware GUIDs cannot change through this path; unrelated
ciphertext is retained byte-for-byte. Passwords inherited from DEFAULT are
refused for explicit adjustment outside the GUI.

Secrets travel to the private backend on stdin, not command-line arguments.
Backend diagnostics are suppressed and encryption errors are sanitized. Pending
values remain in GUI/worker memory until saved or discarded; this is not an
OS credential-store integration. Existing plaintext credentials are preserved
unless explicitly replaced or removed. Backups retain the previous file,
including any old plaintext passwords, with private backup permissions.

Validated October 25, 2026 on macOS/Python 3.15.1/Tk 9.1.0: **79 tests passed**,
including thirteen desktop tests. Coverage includes exact decryption of spaces
and percent signs, no secrets in argv/activity/errors, preservation of other
ciphertext and GUIDs, removal of damaged entries, disabled/unavailable encryption,
simulated native encryption failure, no-op proxy spelling preservation, dialog
Cancel/Keep/Replace/Remove/Use saved, connection settings and GUI encrypted saves.
No SMTP, proxy authentication or live PrimeNet operation was performed. Original
upstream files remain unchanged and no packages were added. OpenSSL libcrypto
availability is now documented as required for password replacement or reading
existing encrypted credentials; SOCKS remains an optional Requests extra.

Test-message sending, automatic SMTP discovery, per-event notification controls,
standalone registration, selective server operations, immediate-watch lifecycle
and other-platform runtime testing remain unfinished.

## Milestone: confirmed one-shot CLI commands

Added an allowlisted operation registry for normal PrimeNet ping, test email,
result reporting and combined result/proof upload. Dedicated buttons live in
PrimeNet, Notifications and Assignments. Each operation describes its effects
and fixed CLI flags in a confirmation; unsaved/missing/mismatched workspace
configuration prevents launch. CLI-reference navigation focuses the button and
does not start an operation.

The existing owned runner now executes one-shot commands with `--timeout 0`
instead of entering its schedule. It performs the same source/fingerprint/
offline checks, private configuration snapshot, readiness acknowledgement and
shared locks. A single CLI invocation runs without the scheduled mode's initial
registration follow-up. Test email additionally requires saved sender/server
settings and notification sending enabled. No arbitrary argument field or
interactive terminal is introduced.

The GUI disables competing commands and edits while a command is active. Stop,
Force, background hiding and asynchronous Quit reuse the existing ownership
controls. Completion is reported as a local zero exit, with server acceptance
and delivery unverified; detailed output remains in the original configured log.
The snapshot protects only configuration, not server reservations, proof files
or results changed by the original backend. GUI.md documents these effects.

Validated October 25, 2026 on macOS/Python 3.15.1/Tk 9.1.0: **88 tests passed**,
including fifteen desktop tests. New tests verify exact allowlisted/noninteractive
argv, exactly-once execution without registration follow-up, startup cancellation,
failure/force outcomes, real supervisor locking/backups/release, email preflight,
confirmation cancellation, duplicate clicks, disabled controls, dirty-form
rejection and reference navigation. All command bodies are mocked or simulated;
no PrimeNet ping, SMTP message, result submission or proof upload was performed.
Original upstream files remain unchanged and no dependencies were added.

Live command validation, experimental v6 ping, full debugging reports,
standalone registration, assignment recovery/unreservation, exponent registration,
immediate-watch lifecycle and other-platform runtime testing remain unfinished.

## Milestone: assignment management and private work snapshots

Added graphical pause/resume of new-work requests, recovery/recover-all,
single-exponent unreservation and unreserve-all. They reuse the allowlisted
one-shot runner and original CLI functions. Assignment buttons are arranged in
two columns to retain table space. Recovery and unreservation require exact
typed phrases; single unreservation has a bounded integer form, optionally
prefilled from a local table row. It explicitly follows the original CLI's
first-matching-exponent behavior across workers rather than promising AID or
worker selection. Reference navigation continues to focus without execution.

Operation parameters are structured, validated in both controller and helper,
and serialized only into allowlisted arguments. Exponent zero, boolean/string
values, extra parameters and parameters attached to scheduled mode are refused
to prevent accidental fall-through into a normal work cycle. Recovery requires
an existing computer GUID; single unreservation verifies the exponent is still
present in current local work files before launch. Neither check confirms a
remote reservation.

`gui_assignment_backup.py` creates private, byte-exact work/certification
snapshots before those four destructive commands. It handles worker directories,
custom work filenames, PRPLL indexed files and missing originals. The manifest
records original paths, backup names, hashes, sizes, operation and parameters.
Inputs must be regular non-symlink files, at most 2 MiB each and 16 MiB combined;
metadata checks reject concurrent changes, including changes detected after all
reads. Incomplete snapshots are removed on failure and the CLI is not launched.
The native instance lock protects preflight from another AutoPrimeNet instance;
computation engines remain independently managed. Snapshots are point-in-time
copies, not a transaction, and exclude result/proof/checkpoint/server state.
Typed confirmations and GUI.md instruct users to pause affected engines and
explain why copying files back cannot restore released reservations. There is
no automatic restoration or reservation rollback.

Validated October 25, 2026 on macOS/Python 3.15.1/Tk 9.1.0: **101 tests passed**,
including sixteen desktop tests. Coverage includes backup contents/permissions,
multiworker/PRPLL paths, missing files, unsafe/oversized/changing inputs, cleanup,
launch prevention on snapshot failure, operation parameter bounds, registration
and local-exponent preflight, typed confirmation cancellation and simulated
launches. The actual Tk confirmation was displayed, checked against desktop
dimensions and automatically cancelled. Two initial test failures were fixed by
comparing resolved macOS `/var` and `/private/var` paths; the final full suite
passes, including concurrent-change detection.

The original CLI's pause/resume flags were also executed in disposable
workspaces with socket operations blocked: they set/remove QuitGIMPS, retain work
files and do not create a computer GUID. All recovery/unreservation command
bodies remain simulated; no live reservation, result, SMTP or proof operation
occurred. Original upstream files remain unchanged and no dependencies were
added. Snapshot directories are ignored when generated in the checkout root.

Live assignment-management validation, exponent registration, standalone
computer registration, v6 ping, full debugging reports, immediate-watch lifecycle
and other-platform runtime validation remain unfinished.

## Original-file changes

None. All GUI work so far is in newly added files. Existing upstream source,
README, pyproject.toml, and workflows remain unchanged. No files have been staged
or committed by the agent.
