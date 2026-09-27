# faraday98

Windows 98 SE on faraday's real hardware (Acer Aspire R3-131T, Pentium N3700 Braswell, 1.7G RAM,
29G eMMC), not a VM. Where drivers are missing, we write them.

Built in public by a human and an AI friend (Claude) working together. Expect rough edges,
and report anything that's wrong.

- `research/2026-09-27-state-of-the-art.md`: what exists, what's missing, how others did it (start here)
- `research/2026-09-27-faraday-hardware.md`: what's actually inside faraday, per device
- `research/readmes/` (local only, gitignored): raw READMEs of the relevant projects
- oerg866's QuickInstall: https://github.com/oerg866/win98-quickinstall

## status

- [x] hardware probe from the installed Linux (UEFI boot)
- [x] BIOS: Legacy boot exists, touchpad Basic mode = PS/2 (verified from Linux)
- [ ] Realtek RTL8168 rev 15: try the NDIS2 driver with REV_15 added to its INF
- [ ] XHCIQUAL from real DOS
- [ ] QuickInstall onto a spare disk or SD card (keep the Arch eMMC install intact)

## license

MIT for our own code and notes. Code ported directly from Linux drivers will be GPL-2 and marked
as such. This repo never contains Microsoft files (DDK, Windows 98 media, disk images).
