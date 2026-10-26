# GUI acceptance checklist

This is the consolidated manual test plan. Manual acceptance is deferred until
implementation is ready; an unchecked item means **not yet verified**, not failed.
Automated results in [GUI_DEVELOPMENT.md](GUI_DEVELOPMENT.md) do not replace these
checks. Use [GUI.md](GUI.md) for installation, controls and documented limits.

## Record each test run

Copy this record into a private test report for each machine. Record pass, fail,
blocked or not applicable for every section, with a reason for skipped checks.

| Field | Result |
| --- | --- |
| Date and Git commit (`git rev-parse HEAD`) | |
| OS/version and architecture | |
| Python version and executable | |
| Tcl/Tk version | |
| Installed dependency versions (`python -m pip list`) | |
| Desktop/session, X11 or Wayland, tray host (where applicable) | |
| Filesystem and native locking available | |
| Test workspace and engine/worker arrangement | |
| Sections completed; failures or blocked checks | |

Keep reports, configurations, logs and traces private until reviewed for account,
email, proxy and assignment information. Report failures with exact steps,
expected/actual behavior and the relevant local time; redact credentials.

## 1. Environment and offline configuration

Use a disposable working directory with copied or synthetic work files. Do not
select an active computation directory for this section. Previewing a copied
configuration is separate from starting its backend: copied credentials and work
can still cause real operations if monitoring or an action is confirmed.

- [ ] Install using the matching OS instructions in GUI.md; `pip check` succeeds
  and `python -m tkinter` opens a working window.
- [ ] Launch the GUI with the venv interpreter. Verify tabs, scrolling, dialogs,
  keyboard navigation, resizing and readable text at the machine's display scale.
- [ ] Select the disposable directory; create, validate, save, close and reload
  a configuration. Verify worker count/directories and program selection persist.
- [ ] Edit Advanced, Hardware, Behavior and Connection settings; Cancel preserves
  the previous values, Apply stages changes, and Save persists them.
- [ ] Verify invalid values explain the problem without replacing the saved file.
  Unsaved changes block operations that require saved configuration.
- [ ] Verify comments and unrelated configuration entries survive saving. Change
  the file externally before saving and confirm the conflict is detected.
- [ ] Exercise password Keep/Replace/Remove/Use saved with disposable secrets;
  saved secrets are not displayed in forms or ordinary GUI activity messages.
- [ ] Preview synthetic assignments and exponent batches; changing the worker,
  workspace or draft invalidates the previous registration preview. Cancel every
  server operation in this offline section.
- [ ] Open CLI reference/help/About, local logs and private diagnostic export.
  Check missing/empty logs, and review an export privately before sharing it.

## 2. Desktop lifecycle without monitoring

- [ ] Launch with `--no-tray`; closing minimizes to a reachable taskbar/Dock item.
  Reopen and Quit successfully.
- [ ] With optional tray dependencies installed, verify Open, status and Quit.
  Closing hides only when the tray helper is ready; reopening preserves edits.
- [ ] Without a usable tray host/dependency, closing retains a reachable window
  through the documented fallback. Retry tray integration gives a usable result.
- [ ] On Linux/FreeBSD where supported, remove/restart the tray host while hidden:
  the window becomes reachable and Retry can establish integration again.
- [ ] Repeated Open/Retry/close actions do not create duplicate windows/helpers.

## 3. Controlled monitoring and ownership

This section starts the real backend and can contact services. Use a designated
test computer/workspace and review the Start confirmation before proceeding.
Do not use copied real assignments as throwaway monitoring fixtures.

- [ ] Start once, verify state transitions and inspect the native log. Repeated
  Start does not create another runner. Confirm outcomes from the native log and
  service state rather than assuming a local zero exit proves acceptance.
- [ ] Hide/reopen during monitoring; polling continues and no duplicate runner
  appears. Tray Start/Stop agrees with the window state.
- [ ] Verify a positive polling interval separately from progress check-ins, then
  verify polling interval zero completes one pass.
- [ ] With a controlled valid result/proof fixture, verify native file watching
  reacts; verify disabled watching retains scheduled processing.
- [ ] Stop while idle, then during an actual controlled upload/report: the GUI
  stays responsive and waits for owned workers to finish. Verify Quit follows
  the same drain policy and independent computation programs remain running.
- [ ] Exercise Force only on disposable work where interruption is acceptable;
  verify the owned runner exits and review any interrupted work before reuse.
- [ ] Attempt a second GUI runner against the same workspace: ownership is
  refused. Test conflict with an independent CLI instance where native locks work.
  Do not remove guards until all relevant processes are confirmed inactive.
- [ ] If testing a filesystem without native locks, follow the explicit no-lock
  limitations in GUI.md; record that independent CLI exclusion is unverified.

## 4. Deliberate live operations

Perform these only on explicitly chosen accounts, recipients and assignments.
Recovery/unreservation and registration can alter server state; local work-file
snapshots are not server rollback. Cancel each confirmation first, verify no
operation occurred, then perform only the intended operation. Mark unavailable
operations blocked with a reason instead of fabricating live data.

- [ ] Normal PrimeNet ping and experimental IPv6 ping; distinguish connectivity
  failure from UI failure.
- [ ] Register/update the designated computer; verify hardware/worker identity
  and the resulting saved configuration/service record.
- [ ] GPU detection, if supported hardware exists: review detected values before
  applying. Unsupported detection leaves manual hardware editing usable.
- [ ] SMTP provider discovery: review disclosure and proposed settings, Cancel,
  then Apply if appropriate. Test email goes only to a controlled recipient;
  verify receipt and TLS/authentication behavior with the actual provider.
- [ ] Pause/resume new-work requests and verify persisted/backend behavior.
- [ ] Register a small intended exponent batch and verify worker-file contents,
  private snapshot and remote reservation state.
- [ ] Report a genuine controlled result and upload a valid proof; verify remote
  acknowledgement and expected local file/archive handling.
- [ ] Recover assignments, recover all, unreserve one and unreserve all only
  where those actions are intended; verify snapshots and actual remote state.
- [ ] Test relevant proxy/SOCKS and authentication settings on machines that need
  them. Record absent provider/network prerequisites as blocked.
- [ ] If exercising debug levels 2–3, inspect owner-private raw traces, bounded
  capture and viewer behavior. Review/delete sensitive fixtures afterward.

## 5. Platform and compatibility completion

Repeat applicable sections on macOS, Windows and the Linux/FreeBSD desktops being
supported. Record GTK native libraries and actual panel/display-server results;
simulated GTK tests do not establish a working desktop installation. Validate
the exact Python **3.15.0** floor separately from later maintenance versions.
Record OS-specific unavailable features instead of treating every platform as
identical.

- [ ] Run the automated GUI suite on each available supported environment using
  the commands below; enable desktop tests only in an interactive desktop.
- [ ] Record manual results and attach reproducible failures to the development
  backlog; fix failures and repeat affected checks.
- [ ] Update GUI.md, GUI_PARITY.md and GUI_DEVELOPMENT.md with actual evidence,
  unresolved limitations and the tested commit before declaring release readiness.

Installers/bundled applications, experimental hidden Prime95 mode, computation
engine management and automatic server-reservation rollback are outside this
acceptance scope. They require separate implementation decisions.

From the repository root, the default suite skips desktop-dependent cases:

```sh
.venv/bin/python -B -m unittest discover -s tests -p 'test_gui_*.py' -q
```

For the full suite on an interactive macOS/Linux/FreeBSD desktop:

```sh
AUTOPRIMENET_GUI_DESKTOP_TESTS=1 .venv/bin/python -B -m unittest discover -s tests -p 'test_gui_*.py' -q
```

Windows PowerShell, with desktop tests enabled:

```powershell
$env:AUTOPRIMENET_GUI_DESKTOP_TESTS = '1'
.\.venv\Scripts\python.exe -B -m unittest discover -s tests -p 'test_gui_*.py' -q
Remove-Item Env:AUTOPRIMENET_GUI_DESKTOP_TESTS
```
