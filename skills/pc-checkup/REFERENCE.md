# How a professional technician checks up a Windows 11 PC

Purpose: raw material for a Claude Code skill that audits a Windows 11 PC (slowness, security, wrong config, drivers, failing hardware).
Source note: written from known docs, then every link was checked live on 2026-10-23 (dead ones replaced). Dell, Lenovo, Intel, Best Buy and the Windows blog block automated checks, so those 5 are unconfirmed. Secondary (shop) sources are labelled **[secondary]**.

Risk labels: **safe** (read-only or trivially reversible) / **restore point** (create one first) / **user OK** (ask before doing). **Admin** = needs an elevated prompt.

---

## Phase 0 - The method pros follow

CompTIA A+ troubleshooting methodology (Core 2): 1) identify the problem (question the user, note recent changes, back up before changes), 2) establish a theory of probable cause (question the obvious), 3) test the theory, 4) establish a plan of action and implement it, 5) verify full functionality and implement preventive measures, 6) document findings, actions and outcomes.
- Source: CompTIA A+ 220-1102 exam objectives, https://www.comptia.org/certifications/a

Skill implication: interview -> back up -> measure baseline -> diagnose read-only -> fix with approval -> re-measure -> report.

## Phase 1 - Intake / interview

| Check | How | Bad looks like | Fix / risk |
|---|---|---|---|
| Symptoms, when it started, what changed (update, new app, drop, spill) | Ask the user; CompTIA step 1 | "Slow since last update", "fan loud", "blue screens" | Narrows theory. safe |
| System identity | `winver`, `systeminfo`, `Get-ComputerInfo` | Unsupported/old build | Plan update. safe |
| Reliability history | `perfmon /rel` (Reliability Monitor) | Red X's clustered after a date | Points to the change. safe |

Sources: https://www.comptia.org/certifications/a ; systeminfo https://learn.microsoft.com/windows-server/administration/windows-commands/systeminfo ; Get-ComputerInfo https://learn.microsoft.com/powershell/module/microsoft.powershell.management/get-computerinfo

## Phase 2 - Protect the data first

| Check | Command/tool | Bad | Fix / risk |
|---|---|---|---|
| Backup exists (OneDrive, File History, Windows Backup) | Settings > Accounts > Windows backup | No backup at all | Back up before any change. user OK |
| BitLocker recovery key known | `manage-bde -status`, `manage-bde -protectors -get C:` (Admin) | Drive encrypted, key not in Microsoft account | Save key before BIOS/firmware/TPM work, or the user can be locked out. user OK |
| Restore point | `Checkpoint-Computer -Description "pre-checkup"` (Admin); System Protection must be on | System Protection off | Turn on + create point. restore point |

Sources: Windows Backup https://support.microsoft.com/windows/back-up-your-windows-pc-87a81f8a-78fa-456e-b521-ac0560e32338 ; BitLocker recovery key https://support.microsoft.com/windows/finding-your-bitlocker-recovery-key-in-windows-6b71ad27-0b89-ea08-f143-056f5ab347d6 ; manage-bde https://learn.microsoft.com/windows-server/administration/windows-commands/manage-bde ; Checkpoint-Computer https://learn.microsoft.com/powershell/module/microsoft.powershell.management/checkpoint-computer?view=powershell-5.1 ; System Restore https://support.microsoft.com/windows/system-restore-a5ae3ed9-07c4-fd56-45ee-096777ecd14e

## Phase 3 - Baseline measurement (before)

Record: boot time, idle CPU/RAM/disk, free space, startup app count, process count, battery capacity, a quick benchmark.
- Boot time: Event Viewer > Applications and Services > Microsoft > Windows > Diagnostics-Performance > Operational, event 100 (boot duration). https://learn.microsoft.com/windows-hardware/test/wpt/windows-performance-analyzer (WPR/WPA for deep boot traces)
- Live resources: Task Manager, Resource Monitor (`resmon`), Performance Monitor (`perfmon`). https://learn.microsoft.com/troubleshoot/windows-server/performance/troubleshoot-performance-problems-in-windows ; https://learn.microsoft.com/windows-server/administration/windows-commands/perfmon
- Built-in score: `winsat formal` (Admin) https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2012-r2-and-2012/cc770542(v=ws.11) (archived page; tool still ships, results in %systemroot%PerformanceWinSATDataStore, readable via `Get-CimInstance Win32_WinSat`).

## Phase 4 - Hardware health

| Area | Command/tool | Bad | Fix / risk |
|---|---|---|---|
| Drive health (SMART) | `Get-PhysicalDisk \| Get-StorageReliabilityCounter` (Admin): Wear, Temperature, ReadErrorsUncorrected; `Get-PhysicalDisk` HealthStatus | HealthStatus Warning/Unhealthy, rising uncorrected errors, wear near 100, reallocated sectors | Back up immediately, replace drive. user OK (hardware) |
| Drive health (vendor) | Settings > System > Storage > Disks & volumes > drive properties shows SSD "Drive health" (estimated remaining life, temp) | Low remaining life / warning | Same |
| File system | `chkdsk C: /scan` (online, Admin); `/f` or `/r` needs reboot | Errors found | `chkdsk /f` scheduled; `/r` slow on HDD. restore point + user OK (downtime) |
| RAM | `mdsched.exe` (Windows Memory Diagnostic), result in System log source MemoryDiagnostics-Results | Any error | Reseat/replace DIMM. user OK |
| Thermals / fans | Vendor diagnostics; Task Manager GPU temp; throttling seen as low CPU clocks under load | Throttling, fans at full at idle | Clean vents, repaste (hardware). user OK |
| Battery | `powercfg /batteryreport` (writes battery-report.html) | Full charge capacity well below design capacity, high cycle count | Replace battery. safe to measure |
| Vendor full diag | Dell SupportAssist / ePSA, Lenovo Vantage + Lenovo Diagnostics, HP PC Hardware Diagnostics (UEFI) | Failure code | Follow vendor code. safe to run |

Sources: Get-StorageReliabilityCounter https://learn.microsoft.com/powershell/module/storage/get-storagereliabilitycounter ; Get-PhysicalDisk https://learn.microsoft.com/powershell/module/storage/get-physicaldisk ; SSD drive health in Settings https://blogs.windows.com/windows-insider/2020/05/27/announcing-windows-10-insider-preview-build-19645/ (introduced feature) ; chkdsk https://learn.microsoft.com/windows-server/administration/windows-commands/chkdsk ; Memory Diagnostic https://support.microsoft.com/en-us/windows/experience/performance-optimization/how-to-fix-error-0xa-irql-not-less-or-equal (mdsched steps; Event ID 2001 = no errors) ; powercfg https://learn.microsoft.com/windows-hardware/design/device-experiences/powercfg-command-line-options ; Dell SupportAssist https://www.dell.com/support/contents/en-us/category/product-support/self-support-knowledgebase/software-and-downloads/supportassist ; Lenovo Vantage https://support.lenovo.com/solutions/ht505081 ; HP PC Hardware Diagnostics https://support.hp.com/us-en/help/hp-pc-hardware-diagnostics

## Phase 5 - Startup and background bloat

| Check | Tool | Bad | Fix / risk |
|---|---|---|---|
| Startup apps | Task Manager > Startup apps (Startup impact column); Settings > Apps > Startup | Many "High" impact items, updaters, trial AV | Disable (not uninstall). safe (reversible) |
| Everything that autostarts | Sysinternals Autoruns (`autorunsc -a * -h -s -c` for CSV, verify signatures) | Unsigned/unknown entries, missing-file entries | Uncheck first, delete later. restore point |
| Heavy processes | Task Manager, Process Explorer (VirusTotal check option) | Constant high CPU/disk at idle | Identify owner app, update or remove. user OK |
| Installed bloat | `winget list`; Settings > Apps > Installed apps | OEM trialware, duplicate AV, toolbars | Uninstall with user's OK. user OK |

Sources: Task Manager startup https://support.microsoft.com/windows/configure-startup-applications-in-windows-115a420a-0bff-4a6f-90e0-1934c844e473 ; Autoruns https://learn.microsoft.com/sysinternals/downloads/autoruns ; Process Explorer https://learn.microsoft.com/sysinternals/downloads/process-explorer ; winget list https://learn.microsoft.com/windows/package-manager/winget/list

## Phase 6 - Malware and persistence

| Check | Command/tool | Bad | Fix / risk |
|---|---|---|---|
| Defender status | `Get-MpComputerStatus` (AMServiceEnabled, RealTimeProtectionEnabled, signature age) | Real-time off, signatures old, third-party AV expired | Turn on / update. safe |
| Full scan | `Start-MpScan -ScanType FullScan`; offline: `Start-MpWDOScan` (reboots) | Detections | Quarantine (reversible). Offline scan = user OK (reboot) |
| Threat history | `Get-MpThreatDetection` | Repeated detections | Find source |
| Scheduled tasks | `Get-ScheduledTask`, `schtasks /query /fo LIST /v`; Autoruns Scheduled Tasks tab | Tasks running scripts from AppData/Temp, unsigned, random names | Disable task. restore point |
| Services | `Get-CimInstance Win32_Service` (PathName, StartMode); Autoruns Services tab | Service binary in user folders, unsigned | Disable. restore point |
| Browser extensions | Each browser's extensions page; Autoruns covers some | Unknown extensions, search hijack | Remove with user. user OK |
| Unsigned binaries | Sysinternals `sigcheck -u -e` | Unsigned exes in System32/startup | Investigate |

Sources: Get-MpComputerStatus https://learn.microsoft.com/powershell/module/defender/get-mpcomputerstatus ; Start-MpScan https://learn.microsoft.com/powershell/module/defender/start-mpscan ; Defender Offline https://support.microsoft.com/windows/help-protect-my-pc-with-microsoft-defender-offline-9306d528-64bf-4668-5b80-ff533f183d6c ; Get-ScheduledTask https://learn.microsoft.com/powershell/module/scheduledtasks/get-scheduledtask ; Sigcheck https://learn.microsoft.com/sysinternals/downloads/sigcheck ; Autoruns https://learn.microsoft.com/sysinternals/downloads/autoruns

## Phase 7 - Windows integrity and updates

| Check | Command | Bad | Fix / risk |
|---|---|---|---|
| Image health | `DISM /Online /Cleanup-Image /ScanHealth` (Admin) | Component store repairable | `DISM /Online /Cleanup-Image /RestoreHealth` then `sfc /scannow`. safe-ish (restore point) |
| System files | `sfc /scannow` (Admin), log in `%windir%\Logs\CBS\CBS.log` | "found corrupt files and was unable to fix" | Run DISM RestoreHealth first, then sfc again; last resort in-place repair. restore point |
| Windows Update | Settings > Windows Update; `Get-HotFix`; update history | Months behind, failing updates | Install; troubleshooter. user OK (reboots) |
| App updates | `winget upgrade` | Outdated apps (browsers, Java, Adobe, 7-Zip) | `winget upgrade --id X`; batch `--all` only with user OK. user OK |
| Component store bloat | `DISM /Online /Cleanup-Image /AnalyzeComponentStore` | Cleanup recommended | `/StartComponentCleanup` (removes rollback of old updates). user OK |

Sources: SFC/DISM https://support.microsoft.com/windows/using-system-file-checker-in-windows-365e0031-36b1-6031-f804-8fd86e0ef4ca ; DISM repair https://learn.microsoft.com/windows-hardware/manufacture/desktop/repair-a-windows-image ; component store https://learn.microsoft.com/windows-hardware/manufacture/desktop/determine-the-actual-size-of-the-winsxs-folder ; Windows Update https://support.microsoft.com/windows/update-windows-3c5ae7fc-9fb6-9af1-1984-b5e0412c556a ; winget upgrade https://learn.microsoft.com/windows/package-manager/winget/upgrade

## Phase 8 - Logs and crash dumps

| Check | Tool | Bad | Fix |
|---|---|---|---|
| Errors | `Get-WinEvent -FilterHashtable @{LogName='System';Level=1,2}` ; Event Viewer custom view "Administrative Events" | Recurring disk (Event 7/51/153), Kernel-Power 41, WHEA-Logger, service crashes | Map to phase (disk -> Phase 4, WHEA -> hardware) |
| Reliability | `perfmon /rel`; `Get-CimInstance Win32_ReliabilityRecords` | App/Windows failures | Update/reinstall culprit |
| Blue screens | Dumps in `C:\Windows\Minidump`, `MEMORY.DMP`; analyze with WinDbg `!analyze -v` | Same driver in many dumps | Update/roll back that driver |
| Unexpected shutdowns | Kernel-Power 41 | Power/thermal/PSU or driver | Investigate hardware |

Sources: Event Viewer / Get-WinEvent https://learn.microsoft.com/powershell/module/microsoft.powershell.diagnostics/get-winevent ; Kernel-Power 41 https://learn.microsoft.com/troubleshoot/windows-client/performance/event-id-41-restart ; WHEA https://learn.microsoft.com/windows-hardware/drivers/whea/ ; crash dump analysis https://learn.microsoft.com/windows-hardware/drivers/debugger/analyzing-a-kernel-mode-dump-file-with-windbg ; Reliability Monitor https://learn.microsoft.com/en-us/troubleshoot/windows-server/performance/reliability-monitor-shows-no-information (data source: RacTask; no current how-to page exists)

## Phase 9 - Drivers

| Check | Command/tool | Bad | Fix / risk |
|---|---|---|---|
| Problem devices | `Get-PnpDevice -PresentOnly \| Where Status -ne OK`; Device Manager yellow bang | Error codes: 10 (cannot start), 28 (no driver), 43 (device reported problem), 31, 39, 52 (unsigned) | Per code: reinstall/update driver, reseat device |
| Driver inventory | `pnputil /enum-drivers`, `driverquery /v` | Very old, generic "Microsoft Basic Display Adapter" | Install OEM driver |
| Source of drivers | OEM tools (Dell Command Update, Lenovo Vantage/System Update, HP Support Assistant), Intel Driver & Support Assistant, AMD Software auto-detect; Windows Update optional updates | Random "driver updater" apps | Prefer OEM > chip vendor > Windows Update. restore point |
| Regression | Device Manager > Roll Back Driver | New driver caused crash | Roll back. restore point |

Sources: Device Manager error codes https://support.microsoft.com/en-us/windows/hardware/drivers/error-codes-in-device-manager-in-windows ; Get-PnpDevice https://learn.microsoft.com/powershell/module/pnpdevice/get-pnpdevice ; pnputil https://learn.microsoft.com/windows-hardware/drivers/devtest/pnputil ; update drivers https://support.microsoft.com/windows/update-drivers-through-device-manager-in-windows-ec62f46c-ff14-c91d-eead-d7126dc1f7b6 ; Intel DSA https://www.intel.com/content/www/us/en/support/intel-driver-support-assistant.html ; AMD https://www.amd.com/en/support/download/drivers.html ; Dell Command Update https://www.dell.com/support/kbdoc/en-us/000177325/dell-command-update

## Phase 10 - Security configuration

| Check | Command | Bad | Fix / risk |
|---|---|---|---|
| Firewall | `Get-NetFirewallProfile` (Enabled) | Any profile off | Turn on. safe |
| Defender features | `Get-MpPreference`, Windows Security app: Tamper Protection, PUA blocking, Smart App Control, SmartScreen | Off | Turn on. user OK (Smart App Control can't be turned back on without reset) |
| BitLocker / Device encryption | `manage-bde -status` / `Get-BitLockerVolume` (Admin) | Off on a laptop | Enable after key is saved. user OK |
| Secure Boot / TPM | `Confirm-SecureBootUEFI` (Admin), `Get-Tpm` (Admin), `msinfo32` | Secure Boot off, TPM not ready | Enable in firmware. user OK |
| Core isolation / Memory integrity (HVCI) | Windows Security > Device security; `Get-CimInstance -Namespace root\Microsoft\Windows\DeviceGuard -ClassName Win32_DeviceGuard` | Off, or blocked by incompatible driver | Remove old driver, enable. restore point |
| UAC | `HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System` EnableLUA=1, ConsentPromptBehaviorAdmin | EnableLUA=0 | Re-enable. user OK (reboot) |
| Daily account is admin | `Get-LocalGroupMember Administrators` | User runs as admin daily | Recommend standard account. user OK |
| Remote access | RDP `fDenyTSConnections`, `Get-NetTCPConnection -State Listen`, installed TeamViewer/AnyDesk, SMBv1 `Get-WindowsOptionalFeature -Online -FeatureName SMB1Protocol` | RDP open without need, unknown remote tools, SMBv1 on | Disable/remove. user OK |
| Outdated software with CVEs | `winget upgrade`; compare against CISA Known Exploited Vulnerabilities catalog | Apps in KEV not patched | Update. user OK |
| Baseline comparison | Microsoft Security Baselines (Security Compliance Toolkit, Policy Analyzer); CIS Microsoft Windows 11 Benchmark; NIST NCP | Large drift | Home PCs: apply only the consumer-relevant items. user OK |

Sources: Windows Firewall https://learn.microsoft.com/windows/security/operating-system-security/network-security/windows-firewall/ ; Tamper protection https://learn.microsoft.com/defender-endpoint/prevent-changes-to-security-settings-with-tamper-protection ; Smart App Control https://support.microsoft.com/windows/smart-app-control-frequently-asked-questions-285ea03d-fa88-4d56-882e-6698afdb7003 ; BitLocker https://learn.microsoft.com/windows/security/operating-system-security/data-protection/bitlocker/ ; Confirm-SecureBootUEFI https://learn.microsoft.com/powershell/module/secureboot/confirm-securebootuefi ; Get-Tpm https://learn.microsoft.com/powershell/module/trustedplatformmodule/get-tpm ; Memory integrity https://support.microsoft.com/windows/core-isolation-e30ed737-17d8-42f3-a2a9-87521df09b78 ; UAC https://learn.microsoft.com/windows/security/application-security/application-control/user-account-control/ ; SMBv1 https://learn.microsoft.com/windows-server/storage/file-server/troubleshoot/detect-enable-and-disable-smbv1-v2-v3 ; CISA KEV https://www.cisa.gov/known-exploited-vulnerabilities-catalog ; Security Compliance Toolkit https://learn.microsoft.com/windows/security/operating-system-security/device-management/windows-security-configuration-framework/security-compliance-toolkit-10 ; CIS Windows benchmarks https://www.cisecurity.org/benchmark/microsoft_windows_desktop ; NIST NCP https://ncp.nist.gov/repository

## Phase 11 - Performance settings

| Check | Command | Bad | Fix / risk |
|---|---|---|---|
| Power plan / mode | `powercfg /getactivescheme`, Settings > Power mode | Laptop stuck on "Best power efficiency" while plugged in, or desktop on power saver | Set Balanced/Best performance. safe |
| Power problems | `powercfg /energy` (Admin, 60 s trace), `powercfg /sleepstudy`, `powercfg /requests` | Errors in energy report, devices blocking sleep, high drain in standby | Fix offending driver/app. safe to measure |
| Disk space | `Get-PSDrive C`; Storage Sense; `cleanmgr` | Under ~10-15% free | Storage Sense / Cleanup recommendations. user OK (deletes) |
| Visual effects | SystemPropertiesPerformance.exe | Only matters on weak hardware | Adjust for performance. safe |
| SSD TRIM / optimization | `fsutil behavior query DisableDeleteNotify` (0 = TRIM on); `Optimize-Volume -DriveLetter C -Analyze` | TRIM off; scheduled optimization off | Re-enable. safe. Never defrag an SSD manually; Windows retrims |
| Firmware/BIOS | `Get-CimInstance Win32_BIOS` vs OEM support page | Years old BIOS, OEM security advisory | Update via OEM tool on AC power, BitLocker suspended. user OK (brick risk) |

Sources: Power mode https://support.microsoft.com/windows/change-the-power-mode-for-your-windows-pc-c2aff038-22c9-f46d-5ca0-78696fdf2de8 ; powercfg options https://learn.microsoft.com/windows-hardware/design/device-experiences/powercfg-command-line-options ; Sleep study https://learn.microsoft.com/windows-hardware/design/device-experiences/modern-standby-sleepstudy ; Storage Sense https://support.microsoft.com/windows/manage-drive-space-with-storage-sense-654f6ada-7bfc-45e5-966b-e24aded96ad5 ; Disk Cleanup https://support.microsoft.com/windows/disk-cleanup-in-windows-8a96ff42-5751-39ad-23d6-434b4d5b9a68 ; performance tips https://support.microsoft.com/windows/tips-to-improve-pc-performance-in-windows-b3b3ef5b-5953-fb6a-2528-4bbed82fba96 ; fsutil https://learn.microsoft.com/windows-server/administration/windows-commands/fsutil-behavior ; Optimize-Volume https://learn.microsoft.com/powershell/module/storage/optimize-volume ; BitLocker suspend before firmware update https://learn.microsoft.com/powershell/module/bitlocker/suspend-bitlocker

## Phase 12 - After measurement and customer report

Repeat Phase 3 measurements, then report: what was found (by severity), what was fixed, what was left and why, hardware that needs replacing, before/after numbers, attached reports (battery-report.html, energy-report.html, sfc/DISM results). This is CompTIA steps 5-6 (verify, prevent, document). https://www.comptia.org/certifications/a

**[secondary]** What shops sell as a "tune-up": Geek Squad lists virus removal, computer tune-up and optimization, and data backup as services (https://www.bestbuy.com/site/services/geek-squad-services/pcmcat138100050018.c?id=pcmcat138100050018). Shop tune-ups typically bundle: startup cleanup, malware scan, updates, driver updates, disk cleanup, hardware diagnostic, and a written report; the steps match Phases 4-11 above.

## What pros never do blindly

- Change anything before a backup, restore point and saved BitLocker key exist. (Phase 2 sources)
- Run "registry cleaners" or third-party driver updaters; Microsoft does not support registry cleaners. https://support.microsoft.com/en-us/topic/microsoft-support-policy-for-the-use-of-registry-cleaning-utilities-0485f4df-9520-3691-2461-7b0fd54e8b3a
- Delete Autoruns/service/task entries outright; they disable first and confirm. https://learn.microsoft.com/sysinternals/downloads/autoruns
- Defragment SSDs or disable the page file / Windows Update / Defender "for speed". https://learn.microsoft.com/powershell/module/storage/optimize-volume
- Flash BIOS on battery or without suspending BitLocker. https://learn.microsoft.com/powershell/module/bitlocker/suspend-bitlocker
- Uninstall apps or delete user files (Downloads, old Windows installs) without the owner's OK. https://support.microsoft.com/windows/manage-drive-space-with-storage-sense-654f6ada-7bfc-45e5-966b-e24aded96ad5
- Apply a full enterprise baseline (CIS/Microsoft) to a home PC; it can break consumer features. https://learn.microsoft.com/windows/security/operating-system-security/device-management/windows-security-configuration-framework/windows-security-baselines
- Skip the verify-and-document step. https://www.comptia.org/certifications/a
