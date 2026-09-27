# Win98 on faraday: state of the art (2026-09-27)

Question: can Windows 98 SE run on faraday's **real hardware** (Acer Aspire R3-131T, Pentium N3700
Braswell, 1.7G RAM, 29G eMMC), and where drivers are missing, how are people writing new ones
(mostly with AI help) in 2026?

Sources: X/Twitter search, GitHub search, web search, and a local mirror of oerg866's repos.
Every project below is linked; read the originals.

## TL;DR

- **This is much more feasible than it looked this morning.** The biggest blocker (USB on an
  xHCI-only machine) was solved **4 weeks ago** by xHCI98, a driver written by one person with AI
  help in 2 months.
- The Win98 driver scene is in an **AI-assisted boom**: xHCI98 (USB3 host), Janus (loads NT/XP
  drivers on 9x, about 500M Claude tokens), an Atheros L2 NIC driver for the Eee PC, a custom
  VKD.VXD for an IBM 5160, and oerg866 running XP NDIS 5.1 NICs on 98. All since March 2026.
- There's also tooling so an AI agent can drive a **real** Win98 box over the network (V9x Remote
  Agent, windows98-mcp, retro-agent). That makes an agent → build → deploy → screenshot loop
  possible on faraday itself.
- Still open, and probably our own work: **touchpad (I2C-HID)**, **eMMC boot**, **wifi**. Nobody
  has published Win9x drivers for any of these.

## faraday's blockers vs what exists now

| Blocker | State of the art | For faraday |
|---|---|---|
| Boot on a UEFI machine | QuickInstall ships **CSMWrap** (SeaBIOS as a CSM, run as an EFI app) plus the CREGFIX, TLB and >512MB-RAM patches in its reference images | Use the native CSM if the Acer BIOS has one. CSMWrap is the fallback |
| **USB (xHCI only)** | **xHCI98** (yeokm1, 2026-08-30): a USBPORT miniport under the NUSB or SweetLow USB 2.0 stack. USB 2.0 speeds only (about 34 MB/s). HID, mass storage, hubs, USB ethernet and UAC1 audio all work, and hotplug works. **Needs a legacy INTx interrupt pin** (Win9x has no MSI). Tested only on Skylake/Comet Lake/100–300-series PCHs, with AMD tested by Omores. **Braswell has never been tested.** | Run `XHCIQUAL` from real DOS first. If it qualifies, faraday gets a USB mouse, USB sticks, USB ethernet and USB audio. We'd also be the first Braswell report, which is worth sending upstream |
| Audio | **WDMHDA** (andrew-hoffman, in QI's driver-lib-extra), the "experimental HD Audio" in QI 1.1.0. Playback only, works with Realtek/VIA codecs, can screech or hang on some hardware | Try it. The fallback is a UAC1 USB dongle through xHCI98 |
| Display | Generic **VBE 2/3** drivers: VBEMP (in QI extra) and JHRobotics' **vmdisp9x** VESA mode (the display driver from SoftGPU, works on real hardware). No acceleration, software 3D through Mesa9x | Depends on the modes the VBIOS or SeaVGABIOS exposes. 1366×768 may not be there. A native Intel Gen8 modeset driver would be a huge project; nobody has done one |
| Multicore | **smp.vxd** (JHRobotics): SMP plus SSE/AVX context switching for 9x. Only apps using libsmp get the extra cores. "Expect large stability issues" | Optional. The N3700 has 4 cores |
| **eMMC boot disk** | Nothing native for Win9x. But **on Bay Trail, the eMMC shows up as a PCI device in legacy boot mode and ACPI-only in UEFI** (winraid thread). SeaBIOS has SDHCI support | Boot through the native CSM, then check `lspci` in legacy mode. If the eMMC is PCI, INT13 disk access plus Win98's compatibility-mode disk driver may just work |
| **Touchpad and touchscreen (I2C-HID)** | **Nothing for Win9x found anywhere.** | Check the BIOS for a "Touchpad: Basic" (PS/2) setting first. Failing that, a USB mouse through xHCI98 already makes the machine usable. A native I2C-HID VxD would be **our original driver**. Legacy mode may also turn the LPSS I2C controller into a PCI device, which is much easier to reach than ACPI |
| **Wifi** | No Win9x wifi drivers found. Janus and oerg866's work can load **XP NDIS 5.1** NIC drivers on 98, but only wired NICs have been proven | The card is an Intel 3165, which has no XP or 9x driver, so wifi is dead. faraday does have **wired ethernet** (Realtek RTL8168, see the hardware probe note), which is the real network path |

## The AI-assisted wave: who did what, and how

| Project | What | AI use | Worth copying |
|---|---|---|---|
| [yeokm1/xhci98](https://github.com/yeokm1/xhci98) ([blog](https://yeokhengmeng.com/2026/08/xhci98-usb-host-driver/)) | xHCI USBPORT miniport for 98SE through Win7 | Claude Fable 5 to plan, Opus 5 to code, Codex GPT 5.6 to review. About 2 months of spare time; the author estimates years without AI | **17 phases with a go/no-go spike first** (a stub that proves the USBPORT callbacks fire). **9 design docs act as the AI's memory**, and every session starts by reading the roadmap. **A DOS qualifier tool before any driver code.** Almost all dev done in QEMU (`qemu-xhci`) with a debug log on I/O port `0xE9`, plus an automated hotplug test matrix across 11 VMs. MSVC 6 and the Win2000 DDK. The undocumented USBPORT ABI came from **ReactOS**, and the command watchdog idea from Linux |
| [NellInc/Janus](https://github.com/NellInc/Janus) | Loads unmodified NT/XP `.sys` drivers inside a Win98 VxD wrapper. 18 drivers proven doing real I/O (NDIS NICs, ScsiPort, USB EHCI/MSC, i8042, serial…) | "About 500 million tokens of Claude (Opus 4.5–4.8) agentic engineering over 4 months" | A strict evidence standard: a claim only counts with an "unfakeable sentinel" (for example an ARP frame on the wire) plus adversarial review |
| oerg866 (tweets 25–26 Sep) | XP/2003 **NDIS 5.1** NIC drivers on 98, and an nForce MCP51 NIC for the first time ever | not stated | The XP driver pool could be reused for 9x |
| [kjellktbtr/asus-eee-900-win98-network-driver](https://github.com/kjellktbtr/asus-eee-900-win98-network-driver) | NDIS 5.0 miniport for the Attansic L2, confirmed on a real Eee PC 900 | (Probably AI assisted, not stated in the README) | **Builds under Wine on Linux** (`build.sh`, VC++5 plus XP DDK NDIS headers). A Linux-hosted toolchain is proven |
| [Mike1978uk/win95-intel-inboard-386pc](https://github.com/Mike1978uk/win95-intel-inboard-386pc) | Win95 on an IBM 5160 with an Inboard 386, including a DDK-built custom VKD.VXD. 14 PRs merged into 86Box | Keeps `.claude/skills/` in the repo (hardware debug methodology, Win9x DMA driver audit) | Methodology written as agent skills, with emulator-first development (their own 86Box hardware model) |
| @308Greenfield (tweet 2026-09-21) | A Win32 C app for 98 built with Claude Code | "I ran every build on the real machines and sent back what broke. Windows 98 breaks things in ways nobody has written down in 20 years, so that loop was the whole job." | The real-hardware feedback loop **is** the work |

The shared recipe: **spec plus open-source references (Linux, ReactOS, BSD) → AI writes the code →
emulator for fast iteration → real hardware as the final judge → documents as persistent memory.**

## Tooling for driving a real Win98 box from an agent

- [michaeldale/V9x-Remote-Agent](https://github.com/michaeldale/V9x-Remote-Agent): a 141 KB agent
  inside 95/98/Me, with an MCP server on the host. Exec, CRC-verified file push, screenshots, input
  injection, and a verified-reboot workflow. Has a physical-machine mode (IP allowlist) and a DOS build.
- [ido-pluto/windows98-mcp](https://github.com/ido-pluto/windows98-mcp): a C89 guest agent plus
  broker plus MCP, and it can also manage QEMU VMs and snapshots.
- [voidsstr/retro-agent](https://github.com/voidsstr/retro-agent): a 200 KB C agent for 98/2K/XP.
  They used it to build and tune a Voodoo 3/4/5 driver stack on real hardware.
- [ryandeering/claudewin9x](https://github.com/ryandeering/claudewin9x): Claude Code on 95/98 via a bridge server.

For faraday this needs a network. The first choice is the onboard **Realtek RTL8168** (see
`2026-09-27-faraday-hardware.md`). The fallback is xHCI98 plus a USB ethernet adapter (the ASIX
AX88772 is known to work; see also ijsf/AX88772B_WinME98SE). Once that works, the loop is: agent on
a Linux box builds → pushes to faraday → runs → takes a screenshot → reads the log.

## Toolchains in use

- xHCI98: **MSVC 6.0 plus the Windows 2000 DDK** (setup borrowed from WDMHDA). x64 builds use WDK 7.1
- Attansic L2: **VC++ 5.0 plus the XP DDK's NDIS 5.x headers**, run under **Wine**. The Win98 DDK's
  own `ndis.h` is NDIS 3.1 only
- XHCIQUAL (the DOS tool): **Open Watcom 2.0 plus the DOS/32A extender**
- Classic VxDs: the Win98 DDK plus MASM 6.11+. Books: Hazzah, *Writing Windows VxDs and Device
  Drivers* 2nd ed. 1997 (its disk is on bitsavers `Media_From_Books`), and Oney, *Programming the WDM*
  (2nd ed. has Win98/Me notes)

## Warnings

- **Malware lures.** GitHub search for "vxd" or "win98 driver" returns lots of throwaway accounts
  created the same day. Example: `Aerophilatelic-assault638/velocity9x`, a "from-scratch S3 driver"
  whose only download is `complanate.zip` and which has no source. Only trust repos with real
  history, source and a known author.
- **Licensing.** Porting Linux driver code directly makes the result GPLv2 derivative work (fine,
  just publish it). xHCI98's approach was spec plus references rather than line-by-line ports.
- xHCI98 says it is from "a solo human with AI assistance only, so bugs are not unexpected". Don't
  write logs to a drive on the controller being tested, and use a PS/2 keyboard during XHCIQUAL.

## Not yet verified (needs faraday on the desk)

*Update: the probe happened the same day. See `2026-09-27-faraday-hardware.md`.*

faraday wasn't on the desk yet on 2026-09-27. The first session should be a **read-only hardware
probe from a Linux live USB**:

1. BIOS: is there a Legacy/CSM boot mode? A PS/2 or "Basic" touchpad mode?
2. `lspci -nn` **in both UEFI and legacy boot**. Do the eMMC (SDHCI) and LPSS I2C controllers turn
   into PCI devices in legacy mode?
3. The xHCI controller's PCI ID plus its interrupt pin (`lspci -vv` → `Interrupt: pin A`). That's
   the xHCI98 go/no-go
4. The touchpad and touchscreen: I2C-HID vendor/product, I2C address (`/sys/bus/i2c/devices`, dmesg)
5. The wifi card's PCI ID and whether an XP driver exists
6. The HDA codec (`/proc/asound/card*/codec#*`), to check against WDMHDA's known-good list (Realtek/VIA)
7. The display modes offered by the VBIOS/GOP (for VBEMP and vmdisp9x)

Then: `XHCIQUAL` from real DOS (QuickInstall can boot DOS), then install QuickInstall to a spare
disk or SD card first so the Arch install on the eMMC stays untouched.
