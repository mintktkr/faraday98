# faraday hardware probe (2026-09-27)

Read-only probe from the installed Arch Linux, **booted in UEFI mode**. Everything below was
observed on the machine unless it says otherwise.

## machine

- Acer Aspire R3-131T, Insyde BIOS **V1.09** (2015-07-21)
- Pentium N3700 (Braswell), 4 cores, 1.7G RAM
- Panel: eDP 1366×768. GPU `8086:22b1` (Gen8)

## per device

| Device | IDs | Bus / Linux driver | Win98 outlook |
|---|---|---|---|
| USB | xHCI `8086:22b5`, **no EHCI** | PCI, `xhci_hcd`. **Legacy interrupt pin B exists** (disabled because Linux uses MSI) | xHCI98 needs a legacy INTx pin, and it's there. XHCIQUAL from DOS has to confirm it actually fires. Braswell is untested |
| Keyboard | i8042 | PS/2 (`isa0060/serio0`) | **Works natively** |
| Touchpad | **ELAN0501**, HID `04F3:3010` | BIOS **Touchpad: Advanced** → I2C-HID on LPSS I2C `808622C1:05` (ACPI). BIOS **Touchpad: Basic** → **PS/2** on the i8042 AUX port, IRQ 12 (Linux: `ETPS/2 Elantech Touchpad`, psmouse) | **Solved: set Basic.** Win98's standard PS/2 mouse driver should handle it (not tested in Win98 yet) |
| Touchscreen | Raydium `2386:0401` | **USB** full-speed HID | Could work through xHCI98 + the Win98 HID stack if it exposes a mouse collection. Unverified |
| eMMC (29.1G) | `80860F14:00` | **ACPI** SDHCI (`sdhci_acpi`) in UEFI mode | Legacy-boot behaviour is still unknown. A second SDHCI node is `80860F14:01` |
| **Ethernet** | Realtek RTL8111/8168 `10ec:8168` **rev 15**, subsys Acer `1025:1022` | PCIe, `r8169` | QuickInstall's driver library has the NDIS5 5.708 driver (INF lists REV_01–03) and the NDIS2 RTGND 1.54 wrapper (INF lists REV_04–0A). **Rev 15 isn't listed in either.** First experiment: add `REV_15` to the NDIS2 INF |
| Wifi | Intel Wireless 3165 `8086:3165` | PCIe, `iwlwifi` | No Win9x or XP driver. Dead |
| Bluetooth | Intel `8087:0a2a` | USB | Ignore |
| Webcam | `1bcf:2c81` | USB | No UVC on Win98. Ignore |
| Audio | HDA `8086:2284`, codec **Realtek ALC255** (+ Braswell HDMI) | PCI, `snd_hda_intel` | WDMHDA works best with Realtek codecs. Good candidate |

## what this changes

- **Wired ethernet exists.** So networking (and an agent driving the machine remotely) may not need
  USB at all. It depends on the Realtek rev 15 experiment.
- xHCI98's hard requirement (a legacy interrupt pin) is met on paper.
- Every input device has a path now: the keyboard (PS/2), the touchpad (PS/2 in Basic mode), and
  the touchscreen (USB, through xHCI98, unverified).

## BIOS (InsydeH2O Rev 5.0, V1.09), checked 2026-09-27

- **Main:** Touchpad Basic/Advanced (now **Basic**). F12 Boot Menu (now **Enabled**). Network Boot
  Disabled. D2D Recovery Enabled.
- **Boot:** Boot Mode **UEFI / Legacy** (Secure Boot disappears in Legacy). Boot order lists
  `EMMC: HBG4e 32G`, USB FDD, USB HDD, USB CDROM, Network Boot IPv4/IPv6.
- **Security:** a supervisor password is set, and Secure Boot is in Custom mode (enabled under UEFI).
- If you forget the supervisor password: after 3 wrong tries the BIOS shows a "System Disabled"
  code, and the standard InsydeH2O unlock-password algorithm works on it.

## legacy boot evidence (efivarfs, UEFI boot, 2026-09-27)

Read straight out of `/sys/firmware/efi/efivars`. No hardware change and no NVRAM write.

**The firmware publishes the eMMC as a legacy (INT13) boot device.** The EFI boot list contains

```
Boot0002* HBG4e    BBS(HD,,0x500)
```

and `HBG4e` is the eMMC's own model string (`/sys/block/mmcblk0/device/name`; `mmc0:0001 HBG4e` in
dmesg). Its device path decodes as ACPI(PCI root, PNP0A03) → PCI(dev 0x10, func 0) → the SDHCI at
`_ADR 0x00100000`, i.e. SDHCI0 = the eMMC. So that BBS entry is the internal eMMC, seen through the
firmware's legacy disk path.

**The firmware is actively enumerating legacy devices.** `LegacyDevOrder` is populated (entries with
connectivity 0x01/0x02/0x03/0x06/0x80 and device indices 0x11/0x12) and `TargetHddDevPath` ends in
`HD(1,GPT,93a633e0-a542-42ad-8b8c-8d0ba08b0046,0x800,0x32000)`, the leftover Windows Boot Manager
partition. `LegacyDevOrder` is an EDK2 variable that only exists when a legacy path is live.

**No CSM toggle shows up as its own variable.** The setup answers are one blob: `Setup`
(GUID `a04a27f4-df00-4d42-b552-39511302113d`, 874 bytes), with siblings `BootType` (1 byte, `0x02`),
`RestoreFactory` (`0x01`), `PhysicalBootOrder` (empty) and `Timeout` (0). `BootType` is very likely
the UEFI/legacy mode setting, but the encoding isn't decoded, so the fact is that the variable
exists, not what its value means. Baselines saved on faraday as
`~/bios-setup-baseline-2026-09-27.bin` and `~/bios-legacydevorder-2026-09-27.bin`: change one BIOS
setting at a time and diff to decode the layout empirically.

**Second SDHCI = a scratch boot medium.** `mmc1` (`80860F14:01`, ACPI `\_SB_.PCI0.SDHC`, status 15)
is a second SDHCI controller with no card in it, while `mmc0` (`80860F14:00`) is the eMMC. The R3-131T spec sheet lists an SD card reader, so an SD card is a Win98 install target that never
touches the Arch install (confirmed once a card shows up on `mmc1`).

### what this changes

The finding that would have killed bare-metal Win98 here was "the eMMC is ACPI-only, so INT13 cannot
see it". The firmware says the opposite: it publishes the eMMC *as an INT13 boot disk* and keeps a
legacy device order. The setup menu does offer Legacy and a Basic (PS/2) touchpad mode (see the BIOS
section). Still unverified: that INT13 actually reads the eMMC (a DOS disk editor listing the MBR,
read-only).

Also in the boot list: `Boot0005 Windows Boot Manager` and `Boot0006 ubuntu` (leftovers from earlier
installs), `Boot0000 Linpus lite` (the Arch install, which is `BootCurrent`), `Boot2001 EFI USB
Device` and `Boot0004 EFI Network` (remote-connect placeholders). There is no BBS entry for CD or
USB, but BBS entries only exist for devices attached at the time, so that proves nothing either way.

## still to check (needs someone at the machine)

1. ~~BIOS: legacy boot and a touchpad Basic mode~~ Both exist, see above.
2. Boot something in Legacy mode (e.g. a Linux live USB) and compare `lspci -nn`. Does the
   eMMC (`80860F14`) or the LPSS I2C (`808622C1`) show up as a PCI device?
3. `XHCIQUAL` from real DOS
4. Decode `BootType`: it was `0x02` under UEFI. Dump it again after booting once with Boot Mode =
   Legacy and diff it against the baseline
