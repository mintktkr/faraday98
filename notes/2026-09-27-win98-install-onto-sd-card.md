# Win98 SE installed onto the SD card (2026-09-27)

Goal: a Win98 SE install that never touches the Arch eMMC. The install target is the 16 GB SD
card in the R3-131T's reader (`mmc1`, `80860F14:01`), which was already proven to boot in Legacy
mode through the laptop's native CSM (F12) earlier the same day.

## how it was done (and why not on faraday)

The install was **driven from erogu inside QEMU**, against the raw SD card in a USB reader. QI's
installer is its own Linux/DOS environment; running it in a VM lets the answer flow be scripted
(screendump + sendkey over the QEMU monitor) instead of typed on the laptop, and leaves faraday
free to keep running Linux while we prepared the card.

1. **Image**: `win98qi_v1.1.0_ALL_usb.img`, sha256 `26abc408…`, from oerg866's releases; cached
   at `~/dev/faraday98-media/win98qi_v1.1.0_ALL_usb.img`.
2. **Card health first**: destructive `badblocks` random write+read over the whole device
   (26 min, blocks 0–3780607, **0 bad blocks**, no kernel I/O errors). It is a used camera card,
   its photos were backed up and sha256-verified beforehand — that is why overwriting it was safe.
3. **Flash**: image written to the card, then read back **byte-identical** (MBR, one bootable
   FAT16 1.2G partition, the rest unallocated).
4. **QEMU**:
   `sudo qemu-system-i386 -enable-kvm -machine pc -drive index=0,file=win98qi_v1.1.0_ALL_usb.img,snapshot=on -drive index=1,file=/dev/sda,cache=none -display none -monitor unix:/tmp/qi-vm/mon.sock`
   (index 0 = the QI image, snapshot so it is never written; index 1 = the SD card).
5. **Choices in the installer**: variant *98SE Patched 98Lite Micro DX8.1b*; in QI's `cfdisk`,
   the whole card as one bootable **FAT32 LBA (0x0c)** partition, sectors 2048–30244863; install
   options at their defaults **plus** the extended driver library (`DRIVER.EX`) and extras, with
   `CREGFIX`. No GPT, no NTFS, no UEFI options.
6. QI reported **Success**. We did **not** reboot inside the VM — first-boot hardware detection is
   meant to happen on faraday's real hardware, which is the whole point of the project.
7. **Verified on erogu** (read-only): 615 MB used; `IO.SYS`, `MSDOS.SYS`, `COMMAND.COM` at the
   root; `WINDOWS/` with 1396 files; `fsck.vfat -n` clean (2218 files). Card ejected.

## what to expect on first boot

- PnP detection will run and may ask for a driver disk; that is normal for QI images, and
  cancelling the prompts is fine until the network is up.
- Display is the **generic VBE** driver from QI's extras (VBEMP). No acceleration.
- **No wifi**: the Intel 3165 has no 9x or XP driver. Wired ethernet (RTL8168) is the network
  path, via the XP NDIS route or a hand-edited NDIS2 INF (see the state-of-the-art note).
- **No USB3** until `XHCIQUAL` says the machine qualifies for xHCI98 (Braswell has never been
  tested — we would be the first report).
- Touchpad should work: the BIOS has a Basic (PS/2) touchpad mode, already verified.

## next

1. SD card into faraday, `F2` → Boot Mode **Legacy**, `F12` → the SD card. Watch the first boot
   and note what PnP finds and what it cannot.
2. `XHCIQUAL` from real DOS (idle USB question).
3. Then the actual project: the drivers nobody has written for this machine.
