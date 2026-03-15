# Codex Heartbeat Automation Playbook

## Purpose
This playbook describes the live heartbeat/polling implementation used in `D:\Projects\mercenary`. It is written to match the current monitor scripts, not an older intermediate design.

## Current Reality
- Control plane: Windows Task Scheduler.
- Primary task: `CodexHandoffMonitor`.
- Current operator cadence: every `5` minutes when enabled.
- Tick script: `D:\Projects\mercenary\.tmp\codex_handoff_monitor_tick.ps1`.
- Dispatch runner: `D:\Projects\mercenary\.tmp\codex_dispatch_runner.ps1`.
- Heartbeat file: `D:\Projects\mercenary\docs\evals\CODEX_HEARTBEAT.txt`.
- Handoff file: `D:\Projects\mercenary\docs\evals\CLAUDE_CODEX_HANDOFF.md`.

## Core Working Model
- Use a scheduler-owned tick, not a long-running session-owned loop.
- Each tick acquires a lock, syncs the repo, reconciles any active dispatch run, scans the handoff, and decides whether to dispatch.
- WIP=1 is enforced by state, lock, and active-run reconciliation.
- Dispatch work is artifact-driven: each run writes `prompt.txt`, `stdout.log`, `stderr.log`, `last_message.txt`, and `exit_code.txt`.
- The monitor can be paused with a repo-local pause file and later auto-resume on a different Claude open item.

## Tick Algorithm
1. Acquire tick lock from `.tmp/handoff_monitor_tick.lock.json`.
2. Load pause state from `.tmp/handoff_monitor_pause.json`.
3. If paused:
- If there is no new distinct Claude open item, exit quietly with no heartbeat churn.
- If a different Claude open item appears, clear pause state and resume.
4. Sync current branch from origin:
- `git fetch origin <branch> --prune`
- `git pull --ff-only origin <branch>`
5. Load dispatch state from `.tmp/handoff_monitor_state.json`.
6. Reconcile `activeRun`:
- If `exit_code.txt` exists, write a handoff status entry and clear `activeRun`.
- If the runner is still alive but exceeds timeout threshold, kill it, write a timeout note, and back off.
- If the runner is gone and no exit artifact exists, mark it lost and back off.
7. Re-read handoff and find the newest Claude `[status: open]` entry anywhere in the file.
8. If a newer Codex response already exists for the same `item_id`, do not redispatch.
9. If the entry is still within retry backoff, do not redispatch.
10. If dispatchable, start one runner process and prepend a Codex `[status: in_progress]` handoff entry.
11. Append a heartbeat line describing what happened.
12. Release tick lock.

## Files Used
- `D:\Projects\mercenary\.tmp\codex_handoff_monitor_tick.ps1`
- `D:\Projects\mercenary\.tmp\codex_dispatch_runner.ps1`
- `D:\Projects\mercenary\.tmp\handoff_monitor_state.json`
- `D:\Projects\mercenary\.tmp\handoff_monitor_tick.lock.json`
- `D:\Projects\mercenary\.tmp\handoff_monitor_pause.json`
- `D:\Projects\mercenary\.tmp\monitor_dispatch\`
- `D:\Projects\mercenary\docs\evals\CLAUDE_CODEX_HANDOFF.md`
- `D:\Projects\mercenary\docs\evals\CODEX_HEARTBEAT.txt`

## State Files
### `.tmp/handoff_monitor_state.json`
Fields actually used by the tick:
- `lastDispatchKey`
- `activeRun`
- `lastFailureKey`
- `lastFailureAtUtc`

### `.tmp/handoff_monitor_pause.json`
Supported fields:
- `paused`
- `pauseEntryKey`
- `pauseItemId`
- `pausedAtUtc`
- `reason`

The tick normalizes missing fields, so a minimal file like this is valid:
```json
{"paused": true, "reason": "User requested pause"}
```

## Dispatch Runner: Live Implementation
The runner no longer launches `codex.ps1` through `pwsh`. That approach was replaced because redirected stdin was not reliably reaching Codex through the npm-generated PowerShell wrapper.

### What it does now
- Resolves `codex` location with `Get-Command codex`.
- Resolves `node.exe`.
- Resolves `node_modules\@openai\codex\bin\codex.js` relative to the installed codex command.
- Launches `cmd.exe /c` with an OS-level pipe:
```powershell
type "<PromptPath>" | "<nodeExe>" "<codexJsPath>" exec -C "<RepoRoot>" --dangerously-bypass-approvals-and-sandbox -s danger-full-access -o "<LastMsgPath>" - >"<OutPath>" 2>"<ErrPath>"
```
- Waits up to `TimeoutSeconds` (currently default `900`).
- Kills timed-out runs.
- Always writes `exit_code.txt` in `finally`.

## Why This Version Works
- Lockfile prevents overlapping ticks.
- State file prevents duplicate dispatch.
- Backoff avoids dispatch storms after failures.
- Artifact-based completion detection is reliable.
- Node + `codex.js` direct launch avoids the PowerShell wrapper stdin bug.
- Prompt is passed through stdin, not CLI arguments, which avoids tokenization breakage.
- `git fetch` + `git pull --ff-only` keeps the local handoff current before evaluation.
- Pause state allows controlled suppression without deleting task definitions.

## Known Failure Modes and Fixes
- Failure: heartbeats continue but no actual Codex work happens.
- Cause: logging-only loop with no dispatch integration.
- Fix: use artifact-backed dispatch runner plus active-run reconciliation.

- Failure: `%1 is not a valid Win32 application`.
- Cause: `Start-Process -FilePath codex` on Windows when `codex` resolves to `codex.ps1`.
- Fix: do not launch `codex` directly as a process target.

- Failure: stdin prompt never reaches Codex.
- Cause: npm PowerShell wrapper (`codex.ps1`) does not reliably forward OS-level redirected stdin in this automation path.
- Fix: bypass wrapper and call `node` + `codex.js` directly through `cmd.exe`.

- Failure: `error: unexpected argument 'are' found`.
- Cause: multiline prompt passed as a raw CLI argument.
- Fix: pass prompt over stdin with `exec ... -`.

- Failure: duplicate dispatch storms.
- Cause: legacy long-running monitor processes, no lock, no backoff, or both.
- Fix: enforce tick lock, use one scheduled task, clear/kill legacy monitors, and keep failure backoff active.

- Failure: stale handoff view.
- Cause: polling without a repo sync step.
- Fix: `git fetch` + `git pull --ff-only` at tick start.

- Failure: sync error on deleted or renamed remote branch.
- Cause: current local branch no longer has a matching remote ref.
- Current behavior: heartbeat logs a `status=stopped note=sync failed ...`, then the tick continues evaluating local handoff state.
- Operational fix: checkout a valid branch or recreate the remote branch if sync is required.

## Scheduler Commands
### Enable at current stable cadence (5 minutes)
```powershell
schtasks /Create /TN CodexHandoffMonitor /SC MINUTE /MO 5 /TR "pwsh -NoProfile -ExecutionPolicy Bypass -File D:\Projects\mercenary\.tmp\codex_handoff_monitor_tick.ps1" /F
schtasks /Change /TN CodexHandoffMonitor /Enable
schtasks /Run /TN CodexHandoffMonitor
```

### Use tighter troubleshooting cadence (2 minutes)
```powershell
schtasks /Create /TN CodexHandoffMonitor /SC MINUTE /MO 2 /TR "pwsh -NoProfile -ExecutionPolicy Bypass -File D:\Projects\mercenary\.tmp\codex_handoff_monitor_tick.ps1" /F
schtasks /Change /TN CodexHandoffMonitor /Enable
schtasks /Run /TN CodexHandoffMonitor
```

## Verification Commands
Task health:
```powershell
schtasks /Query /TN CodexHandoffMonitor /FO LIST /V
```

Recent heartbeat lines:
```powershell
Get-Content D:\Projects\mercenary\docs\evals\CODEX_HEARTBEAT.txt -Tail 40
```

Latest dispatch artifacts:
```powershell
Get-ChildItem D:\Projects\mercenary\.tmp\monitor_dispatch | Sort-Object LastWriteTime -Descending | Select-Object -First 5 Name,LastWriteTime
```

Pause state:
```powershell
Get-Content D:\Projects\mercenary\.tmp\handoff_monitor_pause.json -Raw
```

## Pause / Resume
### Pause by scheduler
```powershell
schtasks /End /TN CodexHandoffMonitor
schtasks /Change /TN CodexHandoffMonitor /Disable
```

### Pause without deleting the task
Write `.tmp/handoff_monitor_pause.json` with:
```json
{"paused": true, "reason": "User requested pause"}
```

### Resume
```powershell
schtasks /Change /TN CodexHandoffMonitor /Enable
schtasks /Run /TN CodexHandoffMonitor
```

If using the pause file, the tick will auto-resume on the next distinct Claude open item, or the file can be cleared manually.

## Legacy Tasks
`CodexProofLoop` and `CodexProofWrite` may still exist in Task Scheduler, but they are not required for the current working monitor design. They should be treated as legacy/optional.

## Operator Notes
- Use only one active `CodexHandoffMonitor` task.
- Do not manually run multiple background monitor processes.
- Trust `exit_code.txt`, `stderr.log`, and `last_message.txt` over guesswork when diagnosing failures.
- If the monitor looks alive but idle, inspect pause state, current branch sync health, and whether Claude actually posted a new distinct `open` item.
