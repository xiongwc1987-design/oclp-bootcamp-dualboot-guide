# OpenCore Legacy Patcher + Boot Camp dual-boot upgrade — hard-won lessons

**TL;DR — the upgrade path this guide is based on**: MacBookAir7,2 (2015 13" MacBook
Air, Broadwell, non-T2), Boot Camp dual-boot with Windows on the same disk, upgraded
from **macOS Monterey 12.7.6 → macOS Ventura 13.7.8** using **OpenCore Legacy Patcher
2.4.1**. Took about 8 hours mostly because each symptom looked like a different problem
than it actually was. This is the order I'd follow next time instead of re-deriving it
from scratch.

If your Mac is old enough that Apple's installer says "not compatible," you have a
Boot Camp Windows partition on the *same* disk, and OCLP's setup isn't behaving the way
the docs imply it should — this is probably for you.

---

## 0. Before touching EFI/bootloader anything: rule out a misbehaving background process

If the Mac starts panicking, needing multiple reboots to boot at all, or shows
`shutdown_stall` reports in `/Library/Logs/DiagnosticReports/`, **don't assume it's the
OCLP/EFI work in progress**. Check first:

```bash
uptime                          # load average >> core count = something's thrashing
ps aux | sort -rk3 -n | head -15
launchctl list | grep -v com.apple   # 2nd column non-zero = crash-looping
```

A crash-looping user LaunchAgent (`~/Library/LaunchAgents/*.plist`, exit code shown in
`launchctl list`'s 2nd column) can single-handedly cause exactly this symptom pattern.
Stop it (`launchctl bootout gui/$(id -u) <plist>`, then move the plist out of
`LaunchAgents/` so it doesn't reload) before doing any more boot diagnosis — it's easy
to waste hours chasing EFI ghosts that were actually caused by an unrelated runaway
process.

Also read the actual panic log before theorizing:

```bash
ls -lt /Library/Logs/DiagnosticReports/*.panic | head -3
cat "<latest>.panic"
```

Look at `Boot args:` in the panic — `-lilubetaall` / `-nokcmismatchpanic` present means
Lilu/OpenCore root patches *were* active for that boot, which can contradict an
assumption that "OpenCore never boots on this machine."

## 1. SIP must be FULLY disabled (all bits) before `bless --setBoot` will work

**Symptom**: `bless --mount /Volumes/EFI --setBoot --file .../OpenCore.efi` fails with
`Could not set boot device property: 0xe00002e2` / `bless exit code: 3`.

**Root cause**: SIP's NVRAM Protections bit (`0x10`) is still enabled. A prior
`csrutil disable` run from an old/different context, or `nvram -p | grep
csr-active-config` showing something like `0x803` (missing the `0x10` bit), is not
enough.

**Fix**: boot to Recovery (`Cmd+R` held from power-on until the spinner appears — on
non-T2 Macs there is no separate icon for Recovery in the Option-key Startup Manager,
`Cmd+R` is the only way in), open Terminal from the Utilities menu, run
`csrutil disable`, reboot. Confirm with:

```bash
csrutil status                  # must say "disabled" (not "Custom Configuration")
nvram -p | grep csr-active-config   # want 0x7f (%7f%00%00%00)
```

Only after this does `bless --setBoot` reliably return exit code 0.

**Caveat**: OpenCore's own config can silently reset `csr-active-config` to a different
value (e.g. back to `0x803`) on every boot it controls — this is normal/intentional
OCLP behavior for root-patched systems, not a sign the manual disable failed or
reverted by itself. Don't re-chase this once OpenCore is confirmed working.

## 2. Boot Camp + shared EFI partition = OpenCore's file gets fought over — use a dedicated partition

Official OCLP docs confirm Boot Camp/Windows on the same EFI System Partition as
OpenCore is fragile — Windows-side behavior can overwrite `EFI/BOOT/BOOTX64.EFI`, and
on some non-T2 firmware `bless --setBoot`'s NVRAM `efi-boot-device` registration is
*accepted* (persists across reboot, `nvram -p | grep efi-boot` shows it) but the
firmware still doesn't actually launch it — `bless --info --getBoot` only reports the
partition (`/dev/disk0s1`), not the specific file, and in practice it falls through to
whatever `EFI/BOOT/BOOTX64.EFI` currently contains.

**Fix (official, documented)**: create a dedicated ~200MB **MS-DOS (FAT)** partition
just for OpenCore, separate from the shared ESP and separate from the NTFS Windows
volumes. Then in OCLP's "Install to disk" disk picker, target this new partition
instead of the shared EFI partition. This makes a genuinely separate "EFI Boot" icon
appear in the native Option-key Startup Manager, distinct from "Windows".

Sources:
- https://dortania.github.io/OpenCore-Legacy-Patcher/WINDOWS.html
- https://dortania.github.io/OpenCore-Legacy-Patcher/TROUBLESHOOT-MISC.html

### How to create that partition safely

- Disk Utility's "+" (add partition) button is **greyed out when an NTFS volume is
  selected** — macOS Disk Utility doesn't support live-splitting NTFS. Don't fight
  this; it's a real limitation, not a UI bug.
- Instead, select the **APFS volume (Macintosh HD)** and shrink that by ~200MB — this
  works because APFS containers support live resize.
- Clicking "+" on an APFS-container disk prompts "Add Partition" vs "Add Volume" —
  choose **Add Partition** (a true separate partition, any format). "Add Volume" only
  creates another APFS volume inside the same container and can't be FAT32.
- **Never resize the currently-booted startup volume from within the running OS** —
  Disk Utility explicitly warns "this can cause the computer to stop responding...
  never turn off during resize." Cancel and instead: reboot to Recovery (`Cmd+R`), open
  Disk Utility from there (the startup volume isn't "in use" from Recovery), and do the
  resize/partition-add there. No risk warnings appear when done this way.

## 3. Even with efi-boot-device correctly set, the OpenCore menu can be invisible

**Symptom**: selecting the (now-correct, dedicated-partition) "EFI Boot" icon in the
native picker goes *directly* to whichever OS with **zero visible transition screen**
— no flash, no OpenCanopy menu, holding ESC/arrow keys does nothing.

**Cause**: OCLP's **"Show OpenCore Boot Picker"** checkbox (Build tab in OCLP
settings) was unchecked. Confirm via:

```bash
plutil -p /Volumes/<OC partition>/EFI/OC/config.plist | grep -A2 ShowPicker
# "ShowPicker" => 0   means hidden
```

**Fix**: OCLP → Settings → **Build** tab → check **"Show OpenCore Boot Picker"** →
rebuild ("Build and Install OpenCore") → reinstall to the dedicated OC partition again
→ reboot. After this, selecting "EFI Boot" shows OpenCore's own dark-themed menu (no
"Choose Network..." Wi-Fi selector — that's the tell you're in OpenCore's own UI, not
Apple's native one) listing both OS entries; select the target one from *there*.

To make an entry the permanent default within OpenCore's own menu (not just a one-time
boot), highlight it and press **⌘+Return** instead of plain Return.

## 4. "Not compatible" installer error needs SMBIOS spoofing, not just root patches

The Board ID exemption patch (visible in OCLP's build log as "Enabling Board ID
exemption patch") is **not** sufficient by itself to bypass the Install Assistant
app's own "macOS Ventura is not compatible with this Mac" gate. That check is separate
and needs actual SMBIOS model spoofing:

OCLP → Settings → **SMBIOS** tab → **SMBIOS Spoof Level: Moderate** (not None, not
Advanced unless you specifically need it — Advanced also spoofs the serial number and
risks Apple service blacklisting). Leave **SMBIOS Spoof Model: Default** (let OCLP
pick the closest/safest target model rather than manually choosing one). Rebuild +
reinstall + reboot through the OpenCore menu, then verify before retrying the
installer:

```bash
sysctl -n hw.model                 # should show the spoofed model, not the real one
kextstat | grep -iE "lilu|whatevergreen|airportbrcmfixup"   # should list all three
```

Only once both of these look right does the installer's compatibility gate actually
disappear.

## 5. "Installer is corrupted" / App Store "version not available"

- An old cached `Install macOS <X>.app` can fail signature/manifest validation after
  sitting around for a while and shows "installer is corrupted" — this is not real
  file corruption, just an expired local validation. Don't try to repair it, just
  regenerate it.
- App Store / System Settings → Software Update will often refuse to offer the newer
  macOS at all ("the requested macOS version is unavailable") even with SMBIOS
  spoofing active — Apple's server-side eligibility check uses deeper hardware
  identifiers that consumer-facing update UIs don't bypass.
- **Fix**: use OCLP's own **"Create macOS Installer" → "Download macOS Installer"**
  flow instead — it pulls a signed installer package directly from Apple's catalog
  servers, bypassing the consumer eligibility gate. Pick the target version explicitly
  from the list it shows (it lists several available versions with sizes/dates).
- That flow can leave large temp artifacts behind if it's interrupted (e.g. by
  insufficient disk space): a `payloads_overlay` file and/or a stray complete
  `Install macOS <X>.app` under `/private/var/folders/.../T/tmp*/`. Check there before
  re-downloading — a fully-formed installer app may already exist and just needs
  `mv` (not `cp`, since it's same-volume and instant) into `/Applications`.
- If it also offers to flash a USB installer: `df -h` reports **`/` separately from
  `/System/Volumes/Data`** on modern macOS (sealed system volume) — always check
  `df -h /System/Volumes/Data` for real free space, not `df -h /`, when diagnosing
  "insufficient space" errors, since `/Applications` and Downloads live on the Data
  volume via firmlink.
- USB flash step failing with "couldn't unmount disk ... in use by process N
  (mds_stores)": Spotlight is indexing the freshly-inserted USB. Fix:
  `sudo diskutil unmountDisk force /dev/diskN`, then retry disk search in OCLP.

## 6. After the OS upgrade completes: Post-Install Root Patch is mandatory

Installing the new macOS is not the end — WiFi/graphics/etc. on unsupported hardware
need OCLP's **"Post-Install Root Patch"** run *after* first boot into the new OS
(OCLP → main menu → Post-Install Root Patch → Start Root Patching → reboot when done).
This will likely be needed again after every future point-update of that macOS version
too (updates can overwrite the patched system files) — don't uninstall the OCLP.app
itself after finishing, keep it around for that reason. The generated OpenCore files
on the dedicated partition and the boot registration must also be treated as a
permanent, required part of this Mac's boot chain going forward — never delete/revert
them.

## Quick reference: full working sequence

1. Recovery (`Cmd+R`) → Terminal → `csrutil disable` → reboot → confirm `csrutil
   status` says disabled and `nvram -p | grep csr-active-config` shows `0x7f`.
2. Recovery → Disk Utility → shrink the APFS volume by ~200MB → Add Partition →
   MS-DOS (FAT) → confirms as a new small partition (e.g. `disk0sN`).
3. OCLP Settings: Build tab → check "Show OpenCore Boot Picker". SMBIOS tab → Spoof
   Level = Moderate, Spoof Model = Default.
4. OCLP → Build and Install OpenCore → Install to disk → pick the **new dedicated
   partition**, not the shared EFI partition.
5. Reboot → hold Option → select "EFI Boot" (a distinct icon from Windows now) →
   OpenCore's own dark menu appears → select target OS (⌘+Return to set as OpenCore's
   own default).
6. Verify: `sysctl -n hw.model` (spoofed) + `kextstat | grep -i lilu` (loaded).
7. Run the target-OS installer (via OCLP's own "Create macOS Installer" if App
   Store/cached installer fails) → install → reboot through OpenCore each time it
   auto-restarts (don't intervene).
8. After first boot into new OS: OCLP → Post-Install Root Patch → Start Root Patching
   → reboot. Done.

---

*Written up after a real (long) troubleshooting session on a MacBookAir7,2. Hope it
saves someone else the 8 hours it took the first time. Corrections/PRs welcome.*
