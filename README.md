<div align="center">

# 🩺 pc-checkup

**A Windows PC technician, inside Claude Code.**

Type `/pc-checkup`, answer four questions, and get a faster, safer PC, with proof for every fix.

![Windows 10 | 11](https://img.shields.io/badge/Windows-10%20%7C%2011-0078D6?logo=windows&logoColor=white)
![Claude Code skill](https://img.shields.io/badge/Claude%20Code-skill-D97757)
![License: MIT](https://img.shields.io/badge/license-MIT-green)
![Free](https://img.shields.io/badge/price-free-brightgreen)

</div>

---

## Why this exists

Most "PC cleaner" apps delete things you didn't ask them to, run registry cleaners, and say "optimized!" with nothing to back it up. A good technician works differently. They **ask what's wrong, back up your files, measure, check everything, fix only what you approve, and prove it worked.**

This skill makes Claude work like that good technician. You only step in for the things only a human can do, like clicking **Yes** on an admin prompt.

## What you get

| | |
|---|---|
| 🗣️ **One round of questions** | Everything is asked up front. Then it gets to work. |
| 💾 **Your files are safe first** | It checks OneDrive and copies your folders to a USB drive, then confirms every file arrived. |
| 📏 **Before and after numbers** | Boot time, idle CPU, free space, and a number for *your* complaint, measured both times. |
| 🔍 **A full technician's checklist** | Disk health, memory errors, crashes, drivers, malware signs, bloat, updates, security settings, battery wear. |
| 🎯 **It hunts your actual problem** | "Everything looks fine" isn't an answer. It records the PC while you do the slow thing, then narrows down the cause. |
| ↩️ **Every fix can be undone** | A restore point comes first. Every change is logged with its exact undo command. |
| ✅ **Every fix is proven** | Did it stick? Did it help (median of 3 runs)? Did it break anything? Fixes that didn't help get undone. |
| 📄 **A plain-English report** | No jargon. What was fixed, what was undone and why, and what only you can do (like "clean the fans"). |

## How it works

```
 1. Ask        →  what bugs you, how hands-off you want it, how to back up
 2. Protect    →  back up your files and confirm the backup worked
 3. Measure    →  "before" numbers
 4. Check      →  hardware, logs, drivers, malware, bloat, updates, security
 5. Reproduce  →  catch your complaint in the act if the checks missed it
 6. Plan       →  one report and one yes/no for the risky fixes
 7. Fix        →  restore point first, one change at a time
 8. Prove      →  re-check, re-measure, look for new errors
 9. Report     →  before/after table, sent back to you
```

The order follows the **CompTIA A+ troubleshooting method** that working technicians are trained on. Every check and fix is sourced in [`REFERENCE.md`](skills/pc-checkup/REFERENCE.md).

## Example report

```markdown
## Before → after
| What                        | Before | After |
|-----------------------------|--------|-------|
| Boot to usable desktop      | 94 s   | 41 s  |
| CPU at idle                 | 23 %   | 4 %   |
| Free space on C:            | 6 %    | 31 %  |
| Crashes in the last 14 days | 5      | 0     |

## Fixed and proven
- Disabled 11 startup apps you don't use (undo: listed in fixes.md)
- Repaired corrupted Windows files (sfc now reports clean)

## Only you can do
- Battery holds 61% of its original charge, so a replacement is worth it (~$60)
```

*Illustrative example. Your numbers will differ.*

## Safety rules it never breaks

- 🚫 Changes nothing until you approve the plan
- 🚫 No registry cleaners, no "driver updater" apps, no defragmenting an SSD
- 🚫 Never turns off Defender, Windows Update or the page file "for speed"
- 🚫 Never deletes your files. Startup apps and services get **disabled**, not removed
- 🚫 Never fakes a result. A check that couldn't run is reported as "not checked", with the reason
- 🛑 If a disk shows signs of failing, everything else stops and backing up becomes the only job

## Install

1. Install [Claude Code](https://claude.com/claude-code) on the Windows PC you want checked.
2. Copy the skill folder into your skills directory:

   ```powershell
   git clone https://github.com/ch-code-ai/pc-checkup.git
   Copy-Item -Recurse pc-checkup\skills\pc-checkup "$env:USERPROFILE\.claude\skills\"
   ```

3. Open Claude Code and type:

   ```
   /pc-checkup
   ```

Everything it creates (the report, the logs, the undo list) goes in `%USERPROFILE%\pc-checkups\<date>\`. If the PC reboots or the session ends, run `/pc-checkup` again and it picks up where it left off.

## FAQ

**Does it need admin rights?** For some checks, yes. Windows will ask you to click **Yes**. All admin work is batched into one prompt for checks and one for fixes.

**Can I approve every single fix myself?** Yes. Pick "Ask me before every fix" at the start.

**What if I don't have a USB drive?** It uses OneDrive if you have it. With no backup at all, it limits itself to safe fixes and tells you why.

**Is my data sent anywhere?** The skill sends nothing anywhere itself. Claude reads the results of the checks the same way it reads anything else in your session.

## Disclaimer

This runs real commands on your PC. It's built to be careful (restore point, logged undo steps, your approval before changes), but you use it at your own risk. Read the plan before you say yes.

## License

[MIT](LICENSE). Free to use, change and share.
