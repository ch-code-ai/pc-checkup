---
name: pc-checkup
description: Give this Windows PC the full technician checkup — hardware health, startup bloat, malware, Windows integrity, drivers, security settings and performance settings — fix what the owner approves, and prove each fix worked. Use when the user invokes /pc-checkup or asks why the PC is slow, to tune it up, or to check it like a technician would.
disable-model-invocation: true
---

# /pc-checkup — check the PC the way a technician does

The owner wants a faster, safer PC with as little effort as possible. So: the agent does the work,
asks all its questions up front in one go, and only interrupts again for things only a human can do
(click Yes on an admin prompt, plug in a drive, reproduce a problem, approve a risky fix).

Order follows the CompTIA A+ troubleshooting method: ask → protect → measure → check → reproduce
the complaint → fix → **prove** → report. Sources for every check and fix live in
`REFERENCE.md` (next to this file); read it when a finding is unclear.

## Hard rules

- **Check first, change second.** Nothing changes before the plan is approved (step 6).
- **No change without a way back.** Restore point first; every change goes in `fixes.md` with its undo.
- **Disable, don't delete.** Startup entries, tasks and services get switched off, never removed.
- **No fix counts until it's proven** (step 8). "I ran the command" is not proof.
- **Never:** registry cleaners, "driver updater" apps, defragmenting an SSD, turning off Defender,
  Windows Update or the page file "for speed", BIOS updates on battery or with BitLocker active,
  deleting the owner's files, `robocopy /MIR` (it deletes), applying a full enterprise security baseline.
- **Never fake a result.** A check that couldn't run is "not checked", with why.
- Plain words; any technical term gets a comparison from normal life.
- **Work folder:** `$env:USERPROFILE\pc-checkups\<YYYY-MM-DD>\`. `state.md` there records the current
  step and what's done, so the checkup resumes after a reboot or a new session. Read it first if it exists.

## Admin commands

The agent's shell is not elevated. For admin work, write a `.ps1` into the work folder (each command
in its own `try`, all output appended to a results file), then launch it so the owner only clicks Yes:
`Start-Process powershell -Verb RunAs -Wait -ArgumentList '-ExecutionPolicy Bypass -File "<script>"'`
Tell the owner first: "A Windows prompt will pop up asking to allow changes — click Yes."
If they decline or it fails, ask once more; then mark those checks "not checked" and carry on.
Batch admin work: one prompt for all checks, one for all approved fixes.

## Flow

### 1. Intake — one message, all questions at once
- What bugs you about the PC? (slow at startup, slow in a certain app, crashes, noise/heat, battery, nothing — just a checkup)
- When does it happen, and since when? Did anything change then (update, new app, drop, spill)?
- **How hands-off do you want this?** (a) *Do every safe fix yourself, ask me only for risky ones* — default; (b) *Ask me before every fix.*
- **Backup:** (a) *You handle it* — I'll tell you when to plug in a USB drive; (b) *Give me the steps and I'll do it.*
Write the answers into `state.md`.

### 2. Protect the owner's files
Find out what already exists before asking for anything:
- OneDrive: is it running and backing up Desktop/Documents/Pictures? (`Get-Process OneDrive`; known-folder paths under `$env:OneDrive`)
- Free space on any external drive: `Get-Volume | Where DriveType -eq Removable`
- BitLocker status (in the admin batch).

**If the owner picked "you handle it":** ask them to plug in a USB drive with enough space (say the
size: total of the user folders). Copy `Desktop, Documents, Pictures, Videos, Music, Downloads` with
`robocopy <src> <drive>\pc-checkup-backup-<date>\<name> /E /R:1 /W:1 /LOG+:<work>\backup.log`.
Robocopy exit codes 0–7 mean success, 8 or higher means some files failed (read the log).
Prove it: compare file count and total size per folder between source and copy; report any gap.
If BitLocker is on, save the recovery key to the USB drive (`manage-bde -protectors -get C:`, admin).
**If "give me the steps":** write the exact steps for their setup (OneDrive, File History or USB copy)
and wait until they say done, then check what you can (files on the drive, OneDrive status).
**If no backup is possible** (no drive, no OneDrive): say what that risks, and limit fixes to *safe* ones.
**If a disk shows failing health at any point:** stop everything else; backup becomes the only job.

### 3. Measure "before" → `before.txt`
- `Get-ComputerInfo` (Windows build, uptime), `Get-CimInstance Win32_BIOS`, `Get-PSDrive C`
- Idle load: `Get-Counter '\Processor(_Total)\% Processor Time','\Memory\Available MBytes','\PhysicalDisk(_Total)\% Disk Time' -SampleInterval 2 -MaxSamples 10`
- Top 10 processes by CPU and memory; startup entries `Get-CimInstance Win32_StartupCommand`
- Battery (laptops): `powercfg /batteryreport /output <work>\battery-report.html`
- Plus one number for **each complaint** from step 1 (see step 5). Admin-only numbers (boot time,
  event 100 in `Microsoft-Windows-Diagnostics-Performance/Operational`) go in the admin batch.

### 4. Check everything
**Admin batch** (one prompt) → `admin-results.txt`:
`Get-PhysicalDisk` health + `Get-StorageReliabilityCounter` (Wear, Temperature, uncorrected errors);
`chkdsk C: /scan`; `DISM /Online /Cleanup-Image /ScanHealth`; `sfc /verifyonly`; `manage-bde -status`;
`Confirm-SecureBootUEFI`; `Get-Tpm`; `Get-ComputerRestorePoint`; `powercfg /energy /output <work>\energy-report.html /duration 60`; boot-time events.

**No admin needed** — what "bad" looks like is in the research note's tables:
- **Hardware:** logged memory errors (`System` log, provider `Microsoft-Windows-MemoryDiagnostics-Results`); battery full charge vs design capacity.
- **Bloat:** startup entries, `winget list` for OEM trialware, duplicate antivirus, toolbars; processes busy at idle.
- **Malware:** `Get-MpComputerStatus`, `Get-MpThreatDetection`; enabled scheduled tasks and services that run from `AppData`/`Temp`/user folders or have random names. Some will be the owner's own tools — list them for the owner to judge, never touch them on a guess. Browser extensions: list them.
- **Updates:** `Get-HotFix | Sort InstalledOn -Desc | Select -First 5`; `winget upgrade`; flag apps on CISA's Known Exploited Vulnerabilities list.
- **Logs (14 days):** `Get-WinEvent -FilterHashtable @{LogName='System';Level=1,2;StartTime=(Get-Date).AddDays(-14)}` grouped by provider+ID — disk (7/51/153), Kernel-Power 41, WHEA-Logger, BugCheck 1001; `Get-CimInstance Win32_ReliabilityRecords`; dumps in `C:\Windows\Minidump`.
- **Drivers:** `Get-PnpDevice -PresentOnly | Where Status -ne OK` (codes → Microsoft's Device Manager error code page); generic "Microsoft Basic Display Adapter"; one driver named in several crashes.
- **Security:** firewall profiles, `Get-MpPreference`, UAC `EnableLUA`, admin daily account, RDP `fDenyTSConnections`, remote tools in `winget list`, `Get-SmbServerConfiguration | Select EnableSMB1Protocol`, Memory integrity (`Win32_DeviceGuard`).
- **Settings:** `powercfg /getactivescheme`; TRIM `fsutil behavior query DisableDeleteNotify`; free space under ~15%; BIOS age vs the maker's support page.

### 5. When the owner's complaint doesn't show up in the checks
"Everything looks fine" is not an answer while the owner still feels a problem. Work it like a technician:
1. **Turn the complaint into a number.** Slow startup → seconds from power-on to usable desktop. Slow app → seconds to open it, or CPU/disk/memory while using it. Crashes → crashes per week in Reliability records. Hot/loud → CPU load and temperature at idle. Battery → drain per hour.
2. **Catch it in the act.** Start a recorder, ask the owner to do the slow thing now, then stop it:
   `typeperf "\Processor(_Total)\% Processor Time" "\PhysicalDisk(_Total)\% Disk Time" "\Memory\Available MBytes" "\Process(*)\% Processor Time" -si 1 -sc <seconds> -f CSV -o <work>\catch.csv -y`
   (not `Export-Counter`: it doesn't exist in PowerShell 7). The busiest process or resource during the slow moment is the lead.
3. **If it only happens sometimes:** set a scheduled task (owner's OK) that logs the same counters
   every minute for a day or two, tell the owner to note the time when it happens, then read the log
   around those times. Remove the task afterwards.
4. **Isolate.** Does it happen in a clean boot (non-Microsoft services and startup apps off — record
   what was off so you can restore it)? In Safe Mode? In a fresh test user account? Only on battery? Only
   on one network? Whatever makes it disappear points at the cause.
5. **Compare with what the hardware can do.** If the PC is behaving normally for its parts (e.g. 4 GB
   RAM with 30 browser tabs, a spinning hard drive), say so honestly and name the upgrade that would help
   and roughly what it costs. That is a real finding, not a failure.
6. If two theories fail, stop, tell the owner what was ruled out, and what would need deeper tools
   (Windows Performance Recorder trace, Process Monitor) or a hands-on technician.

### 6. Plan → `report.md`, then ask once
Group findings: **Fix now** (danger to data or security), **Worth fixing** (speed/stability), **Fine**,
**Not checked** (with why), **Only you can do** (buy a part, clean the fans). Hardware failures at the top.
Each finding: what it is in plain words, why it matters, the fix, its risk (*safe* / *needs restore
point* / *needs your OK*: hardware, reboots, uninstalls, deletions), and **how we'll prove it worked**.
Hands-off mode: list which fixes you'll just do (safe ones) and ask one yes/no for the risky ones together.
Write the approved list into `state.md`.

### 7. Fix
1. Create a restore point in the fix batch (`Checkpoint-Computer -Description "pc-checkup <date>"`).
   If System Protection is off, turn it on first (it's in the approved batch). If the restore point
   fails, stop — no fixes without it.
2. Each fix goes into `fixes.md` before it runs: what, why, exact command, **exact undo**, proof check.
3. One fix at a time where it matters (drivers, services, startup) so a result can be traced to its cause.
4. Windows files: `DISM /Online /Cleanup-Image /RestoreHealth`, then `sfc /scannow`.
   Drivers: PC maker's tool first, then chip maker, then Windows Update.
5. Needs a reboot? Update `state.md` with exactly where you are, tell the owner, and resume from it after.

### 8. Prove every fix
For each fix, run its proof check and record before → after in `fixes.md`:
- **Did the change stick?** Re-run the check that found the problem (the device is now OK, the startup
  entry is disabled, the setting reads the new value, sfc reports clean).
- **Did it help?** Re-measure the number tied to it (startup time, idle CPU, app open time, free space).
  Measure 3 times and take the middle value; a single run lies.
- **Did it break anything?** New errors in the event log since the fix; the owner's main apps still open.

Then one of:
- **Fixed** — stuck and helped. Keep.
- **Stuck but no difference** — undo it if it costs anything (a disabled app the owner uses, a removed
  feature); keep it if harmless. Either way, go back to step 5 with this theory ruled out.
- **Didn't stick** (it came back after reboot, something re-enabled it) — find what's undoing it:
  an app's own updater, a scheduled task, a policy, Windows Update reinstalling a driver. Fix that.
- **Made it worse** — undo immediately from `fixes.md`; if undo fails, use the restore point. Tell the owner.
- **Two failed tries at the same problem** — stop, report what was tried and what was learned.

### 9. Final report → `report.md`, then show it
Re-run step 3 into `after.txt`. The report opens with a before/after table (every number measured
both times), then: fixed and proven, tried and undone (and why), still open, only-you-can-do, and
where the backup and restore point are. Attach the HTML reports. Mark `state.md` done.
