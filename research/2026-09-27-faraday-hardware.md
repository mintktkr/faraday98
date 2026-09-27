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

## still to check (needs someone at the machine)

1. ~~BIOS: legacy boot and a touchpad Basic mode~~ Both exist, see above.
2. Boot something in Legacy mode: boot a Linux live USB in legacy mode and compare `lspci -nn`. Does the
   eMMC (`80860F14`) or the LPSS I2C (`808622C1`) show up as a PCI device?
3. `XHCIQUAL` from real DOS
