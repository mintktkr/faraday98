# `KRNL386.EXE` — what "Access to the file was denied" means here (2026-09-27)

After the install was moved from the SD card to a USB stick, the first boot on faraday gets past
the protected-mode switch and all VxD initialisation, and then stops at:

```
While initializing device SHELL:
Cannot find or load required file KRNL386.EXE. Access to the file was denied.
```

That is *further* than the SD card ever got (safe mode hung after `HIMEM`), so it is progress, not
a new wall. This note is about what the message is evidence of, and about the one check that
decides the question. It is not a fix.

## the short version

The install is not the variable — the media is. The stick carries the same install as the SD card
(2218 files, 611 MB, same QI run, `fsck` clean), and real-mode DOS reads the same stick happily.
What changes at exactly the moment of the failure is **who is doing the reading**: the message
comes from the protected-mode loader, and it lands on the first file the protected-mode file
system has to open.

## why that moment matters

Windows 9x can read a disk two ways:

- through a **protected-mode port driver** — a `PDR`/`MPD` in `WINDOWS\SYSTEM\IOSUBSYS`
  (`ESDI_506.PDR` for the chipset IDE, `SCSIPORT.PDR` plus a miniport for SCSI, and vendors'
  own). Fast and native; it is what the QEMU install has, because its boot disk is IDE/ATA.
- through **MS-DOS compatibility mode**: with no driver, the protected-mode file system keeps
  calling the firmware's `INT 13h`, from a V86 task.

This laptop has no PDR for any of its storage — eMMC/SDHCI, SD reader and USB all reach Windows
9x only as a BIOS disk. So the boot disk is on the compatibility path by construction, and that
path is a known weak spot. OSDev's *Virtual 8086 Mode* article states it flatly: *"Windows 9x
suffered from system freezing during disk access. Often due to an INT13-through-VM86 problem."*
A firmware `INT 13h` handler that is only ever exercised in real mode — the USB legacy module is
the obvious candidate — is exactly the kind of handler that can stop behaving once its caller is
a V86 task, with SMM or a private buffer underneath it.

Call that a hypothesis with a mechanism, not a proven cause. It does fit the observed boundary:
DOS boots, every VxD reaches the end of `DEVICEINIT` (that phase is hardware and memory setup,
not file I/O), and the failure lands on the loader's first file open.

## what it does not mean

- **Not a damaged install.** A file that real-mode DOS opens and the protected-mode loader cannot
  is a statement about the read path, not about the bytes on the disk.
- **Not QEMU lying to us.** The install boots in QEMU because QEMU gives the boot disk a
  protected-mode driver. The same image dying on real hardware says nothing about the image.
- **Not exotic.** This is an ordinary Win9x problem with an ordinary solution: the boot disk
  needs a protected-mode driver of its own. This repo's own mirror of Mike1978uk's Win95-on-5160
  project reaches "32-bit protected-mode disk access on the boot disk" by writing `XTIDEMP.MPD`
  so that `RMM.PDR` stands down and `C:` is served by `SCSIPORT` + `DiskTSD` — and ends with a
  `CONFIG.SYS` that has no real-mode storage drivers left. Same shape, solved by writing the
  driver.

Microsoft's generic article for this message (KB Q197685) lists damaged or misconfigured
hardware, IRQ steering, bus mastering and damaged system files. All four presume an IDE/PCI
machine with a chipset driver; none describes a BIOS-only boot disk. It is worth knowing mainly
for what it does not cover.

## the check that decides it

Read **the stick's own `C:\BOOTLOG.TXT`** from a DOS prompt on faraday. No unplugging, no Windows,
no QEMU. Set `Logo=0` in `MSDOS.SYS` first (the splash hides the lines) and boot once normally,
not safe mode.

- **Log present, with the VxD lines in it** → protected-mode disk I/O through the USB path
  *works*, and the refusal of `KRNL386.EXE` is narrower than "the disk is unreadable". Then the
  cheap things in order: the file's directory entry, its cluster chain, and a real-mode read of
  the file itself.
- **No log, or a log that stops before the VxD lines** → protected-mode reads through the USB
  legacy `INT 13h` are what is broken, and no QI option, registry tweak or `MSDOS.SYS` switch
  reaches that. It is a driver.

Two readings to avoid, both of which cost time on the SD card:

- **Never quote a `BOOTLOG.TXT` tail as the point of death.** The log is written through VFAT's
  write cache, so a hard hang takes the pending buffer with it. The tail is "the last line that
  got flushed", not "the line that killed it" — on the SD card the log stopped at `VDMAD` and the
  boot had in fact gone further.
- **Check the log for truncation before reading it.** The SD card's log was a three-cluster
  chain (134 → 17486 → 17866 = 24 KB, 402 lines), and its first 8 KB fragment ends mid-story at
  `PERF` while looking like a clean ending. Only the reassembled full file is worth reading.

## already ruled out

- **The 8.4 GB `INT 13h` boundary.** The stick's FAT32 partition runs 2048–16779263, and the
  CHS-addressable area (1024 × 255 × 63) ends at LBA 16450560 — so the top ~168 MB of the
  partition is past it. The install is ~611 MB ≈ 1.25 M sectors, nowhere near. And if geometry
  were the problem, real-mode DOS would trip on it too, and DOS does not.
- **A bad flash.** The stick's install matches the first SD-card install exactly (2218 files,
  611 MB, `fsck.vfat` clean) and QI reported Success. Nothing was rebooted inside the VM, so the
  first-boot hardware detection is genuinely happening on faraday for the first time.

## what this would mean for the project

Every storage path this machine owns is BIOS-only in Windows 9x, and none of them has a
protected-mode driver. If the check above lands on "reads through USB legacy are broken", then
running Win98 from a USB stick is not a configuration problem — it is the same class of work as
the wifi, touchpad and eMMC drivers already on the list, and the same class that xHCI98 and Janus
solved on other machines by writing the driver.

The eMMC should be tested rather than assumed to fail the same way. Its `INT 13h` handler is
ordinary firmware code, not the USB legacy module, and nothing above says the eMMC path inherits
this failure. The cheap version of that test is a small partition (1–2 GB) and a second install
on the eMMC — but that means shrinking the working Arch install, which is the human's call.

## sources

- Microsoft KB Q197685, *While initializing device shell: Cannot find or load a required file
  krnl386.exe* — <https://ftp.zx.net.nz/pub/Patches/ftp.microsoft.com/MISC/KB/en-us/197/685.HTM>
- OSDev wiki, *Virtual 8086 Mode*, "Using VM86 for disk access" —
  <https://wiki.osdev.org/Virtual_8086_Mode>
- Mike1978uk, *win95-intel-inboard-386pc* (protected-mode access to the boot disk) —
  <https://github.com/Mike1978uk/win95-intel-inboard-386pc>
- yeokm1, *xhci98* — the "no driver exists, so write one" precedent already cited in
  `research/2026-09-27-state-of-the-art.md`
