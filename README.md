# faraday98

Windows 98 SE on faraday's real hardware (Acer Aspire R3-131T, Pentium N3700 Braswell, 1.7G RAM,
29G eMMC), not a VM. Where drivers are missing, we write them.

Built in public by a human and an AI friend (Claude) working together. Expect rough edges,
and report anything that's wrong.

- `research/2026-09-27-state-of-the-art.md`: what exists, what's missing, how others did it (start here)
- `research/2026-09-27-faraday-hardware.md`: what's actually inside faraday, per device
- `notes/2026-09-27-win98-install-onto-sd-card.md`: how the SD card install was made, and what
  the first boot should look like
- `notes/2026-09-27-krnl386-access-denied.md`: why the USB stick's first boot stops at
  `KRNL386.EXE`, what that message is evidence of, and the one check that decides it
- `research/readmes/` (local only, gitignored): raw READMEs of the relevant projects
- oerg866's QuickInstall: https://github.com/oerg866/win98-quickinstall

## status

- [x] hardware probe from the installed Linux (UEFI boot)
- [x] BIOS: Legacy boot exists, touchpad Basic mode = PS/2 (verified from Linux)
- [x] legacy boot from the SD card, through the native CSM (no CSMWrap)
- [x] QuickInstall onto a spare disk or SD card (keep the Arch eMMC install intact) — Win98 SE
  installed to the 16 GB SD card, then to a USB stick, both from QEMU, 2026-09-27
- [x] first boot from the USB stick gets past the protected-mode switch and all VxD init — it then
  stops at `KRNL386.EXE`, "access to the file was denied" (the SD card hung earlier, after
  `HIMEM`; see the note above)
- [ ] a Windows 98 desktop: `C:` has no protected-mode driver for any of this machine's storage,
  so the boot disk is on the MS-DOS compatibility path (INT 13h through V86), which is where the
  `KRNL386.EXE` read fails
- [ ] Realtek RTL8168 rev 15: try the NDIS2 driver with REV_15 added to its INF
- [ ] XHCIQUAL from real DOS
- [ ] the drivers nobody has written: touchpad (I2C-HID), eMMC, wifi

## license

MIT for our own code and notes. Code ported directly from Linux drivers will be GPL-2 and marked
as such. This repo never contains Microsoft files (DDK, Windows 98 media, disk images).
