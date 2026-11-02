# Omen Control Victus Edition

Native Intel macOS fan control for HP laptops running macOS on OpenCore. **Version 1.1.0** (fan bridge 0.5.0) is an unofficial community fork of [Omen-Control](https://github.com/DEATHLESSEVIL/Omen-Control), which targets the HP OMEN 16-wf1xxx (board 8C78). This edition adds support for the **HP Victus 16 (board 8BC2)**.

This project is not affiliated with, endorsed by, or supported by HP or the original author. Problems with this fork should be reported to this fork, not to the original project.

[Installation](INSTALLATION.md) · [Victus 16 notes](Docs/VICTUS-16.md) · [Limitations and risks](Docs/LIMITATIONS.md) · [Protocol](Driver/PROTOCOL.md) · [Release notes](Docs/RELEASE-NOTES.md) · [Power helper](Docs/POWER-HELPER.md) · [License](LICENSE)

> **Experimental software that loads a kernel extension.** A bad install can cause boot failure, crashes, or thermal problems. Keep a verified EFI backup and a bootable recovery USB, and watch temperatures the first time you use manual fan control.

## Supported hardware

| Laptop | PCI subsystem | Status |
| --- | --- | --- |
| HP Victus 16 (i7-13700HX) | `103C:8BC2` | RPM readback and manual fan control confirmed working by the fork author, using a manual kext install. See [Victus 16 notes](Docs/VICTUS-16.md). |
| HP OMEN 16-wf1xxx | `103C:8C78` | Tested by the original project (BIOS F.15). |

Any other board is refused by the driver. Adding a board check is not hardware validation; see [Adding another board](Docs/VICTUS-16.md#adding-another-board).

Requirements: Intel Mac or Hackintosh on macOS 13+ for the app, and a working OpenCore setup with a kext-capable build for the driver. The bundled driver build targets Darwin 25.5–25.99 (macOS Tahoe).

## Features

- Separate CPU and GPU fan readback (RPM) and manual requests.
- Profiles: **Auto**, **Eco** (30%), **Manual**, **Performance** (full fans) and **temperature curve**.
- Menu-bar panel with per-fan sliders, so the main window can stay closed.
- **Power Profile**: separate battery and plugged-in preferences, plus macOS Low Power Mode through an on-demand helper.
- Settings for °C/°F, Dock icon visibility, appearance and the background service.

### Fan behavior

| Profile | What it does |
| --- | --- |
| Auto | Firmware controls cooling. |
| Eco · 30% | Requests a lower speed, clamped to each fan's firmware minimum. |
| Manual | Requests a percentage per fan. 1–100% scales that fan's own maximum. |
| Performance · Full | Requests maximum fan speed. It does not raise CPU/GPU power limits. |
| Temperature curve | Follows the hotter of the CPU and GPU sensors. Needs both sensors. |

**0% means firmware Auto, not fan stop.** The fan-stop method in these BIOSes is a stub. At low load the firmware itself may stop the fans in Auto, so an RPM of 0 at idle is normal.

RPM readback is in 100 RPM steps. The usable range comes from each machine's firmware fan table (Victus 16 / 8BC2: CPU 1,800–5,800 RPM, GPU 2,100–6,100 RPM).

### Safety fallbacks

- The driver holds a 20-second lease that the app renews every 3 seconds. Disconnect, lease expiry, sleep, shutdown, or repeated stalled-fan telemetry returns the fans to firmware Auto.
- At a validated CPU temperature of 90 °C the driver requests Auto, and the app boosts manual cooling under serious or critical thermal pressure.
- If the firmware fan table cannot be validated, the driver stays in **read-only mode**: RPM is shown, and no fan writes are accepted.

These are fallbacks, not a guarantee against every firmware or driver fault.

## Install

Read [Installation](INSTALLATION.md) first. In short:

1. Build the app (`./build-app.sh`) and copy `dist/OmenControl.app` to `/Applications`.
2. Build the driver (`zsh Driver/build-fan-bridge.sh`), copy `OmenFanBridge.kext` to `EFI/OC/Kexts`, add one `Kernel → Add` entry, validate with your OpenCore's `ocvalidate`, and reboot.
3. Check the driver:

```sh
ioreg -r -c OmenFanBridge -l | grep -E 'BoardVerified|FanWritesSupported|CPUFanRPM|GPUFanRPM'
```

A working Victus 16 shows `BoardVerified = Yes` and `FanWritesSupported = Yes`.

The experimental combined installer (`Scripts/build-release.py`) also accepts board 8BC2, but it has not been tested end to end on a Victus 16.

## Diagnostics

Open **Diagnostics** in the dashboard for driver and sensor errors. These registry properties on `OmenFanBridge` help when reporting problems:

| Property | Meaning |
| --- | --- |
| `BoardVerified` | The PCI subsystem ID matched a supported board. |
| `DiscoveryTableOK` | The firmware fan table was read and validated. |
| `FanWritesSupported` | Table and RPM both validated, so writes are enabled. |
| `TelemetryReadOnly` | RPM works but the table did not validate, so writes are disabled. |
| `RawFanTable`, `RawFanTelemetry` | Raw firmware bytes for debugging a new board. |
| `LastApplyStage`, `LastApplyStatus` | Which write step failed and why. |

Remove serial numbers, EFI configs and other personal details before sharing a report.

## For developers

Open `OmenControl.xcodeproj` in Xcode (target **OmenControl → My Mac**), or build from the command line:

```sh
./build-app.sh
swift run --disable-sandbox OmenCoreChecks
clang++ -std=c++17 -IDriver Driver/test-fan-policy.cpp -o /tmp/omen-policy-checks && /tmp/omen-policy-checks
```

Quit the installed app first: only one instance can own the fan lease. Building the app does not install kernel code.

| Folder | Contents |
| --- | --- |
| `Sources/OmenControl` | SwiftUI/AppKit UI, telemetry, fan client, power client, fan renderer |
| `Sources/OmenCore` | Request validation, curve interpolation, animation, power-policy parsing |
| `Sources/PowerProtocol` | XPC contract and command allowlist for the power helper |
| `Sources/OmenPowerHelper` | On-demand privileged helper, limited to two power settings |
| `Sources/SMCReader` | Read-only SMC and NVIDIA thermal access |
| `Driver` | `OmenFanBridge` kext, fan-table policy and its boundary tests |
| `Installer`, `Scripts` | Experimental EFI installer and release tooling |

## License and credits

MIT. `LICENSE` keeps the original copyright line and adds a line for this fork; keep both, plus `THIRD-PARTY-NOTICES.md`, when redistributing. HP, OMEN and Victus are trademarks of their owners. Their names are used here only to identify compatible hardware, and the MIT license does not grant trademark rights.

**What comes from where**

- The app, UI, power helper, installer and the original driver design: the original [Omen-Control](https://github.com/DEATHLESSEVIL/Omen-Control) project, by Yashwanth / DEATHLESSEVIL and its contributors.
- This edition, by **vyrozen**: board 8BC2 support, fan-table tolerance, read-only mode and diagnostics, version bump, and the documentation rewrite. See [release notes](Docs/RELEASE-NOTES.md).

Protocol references: [Linux hp-wmi](https://github.com/torvalds/linux/blob/master/drivers/platform/x86/hp/hp-wmi.c) and [OmenMon](https://github.com/OmenMon/OmenMon). Original project: [DEATHLESSEVIL/Omen-Control](https://github.com/DEATHLESSEVIL/Omen-Control).
