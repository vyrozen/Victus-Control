# HP OMEN / Victus fan bridge 0.5.0 (boards 8C78 and 8BC2)

Tested physical machines:

- HP OMEN 16-wf1xxx, board 8C78, BIOS F.15 (original project). Fixed fan control was confirmed in 0.3.1; 0.4.0 retains its write transport and adds lower bounded targets and CPU temperature telemetry. 0.5.0 (this fork) adds board 8BC2, fan-table tolerance and read-only mode.
- HP Victus 16 (i7-13700HX), board 8BC2 (this fork). RPM readback and manual control were confirmed with the same transport. See [Victus 16 notes](../Docs/VICTUS-16.md).

## Transport and scope

Board identity is checked using PCI subsystem vendor 103C and subsystem ID 8C78 or 8BC2, independently of spoofed SMBIOS. The matched ID is published as `PhysicalBoard`. The WMI BIOS GUID is `5FB7F034-2C63-45E9-BE91-3D44E2C707E4`. Resolve its method object from `_WDG` rather than assuming a name (this machine uses WMAA).

Requests contain four little-endian u32 fields: signature `0x55434553`, command `0x20008`, type, input-byte count, then 128 allocated payload bytes. Output has an eight-byte header, including the firmware return code at offset four. Require successful transport, zero firmware status, and adequate length.

MID 2 is used for four-byte acknowledgements; MID 3 for 128-byte reads. **Writes use a 16-byte input payload.** On this firmware, WHCM creates the DAIN BufferField from the input size. Short fields evaluate as an Integer under macOS ACPICA; GC27/GC2E then fail when indexing it as a Buffer. Padding to 16 bytes fixes the transport without changing the first two fan bytes.

| Type | Purpose | Details |
|---|---|---|
| `0x2D` | Actual fan readback | 128 bytes; first two bytes ×100 RPM |
| `0x2F` | Fan table | Two-byte header; CPU/GPU/noise triplets; independent minimum/maximum units. Rows with CPU 0 and GPU 0 (the 8BC2 "fans off" row) are skipped. |
| `0x2E` | CPU/GPU target | First two bytes in 100 RPM units, zero = firmware Auto; pad to 16 bytes |
| `0x27` | Clear maximum override | 16 zero bytes; four-byte acknowledgement |
| `0x10` | Manual-mode keepalive | Side effect, not a passive query; four input/output bytes |
| `0x2A` | Thermal/design read | 128 bytes; byte 0 is this board's CTMP field; byte 2 is not labelled as GPU |

Percentages 1–100 scale each fan's own maximum, rounded to firmware units and clamped to that fan's minimum. A zero request returns that fan to firmware Auto. The 8C78 table has CPU 2000–6000 and GPU 2300–6300 RPM; the 8BC2 table has CPU 1800–5800 and GPU 2100–6100 RPM. Measured readback may differ from these target bounds; it is not cosmetically clamped.

The apparent high-resolution `GC11` read and fan-stop `GC31` are stubs in this board's ACPI. Success alone is not evidence of fan stop. No arbitrary EC read/write API is exposed.

## Discovery and read-only mode

The driver tries discovery up to ten times, three seconds apart. After the board matches it reads the fan table (`0x2F`) and RPM (`0x2D`).

- Both succeed: `FanWritesSupported = Yes`.
- RPM succeeds but the table does not validate: `TelemetryReadOnly = Yes`. RPM keeps updating and every write selector is refused.
- Diagnostics: `DiscoveryTableOK`, `DiscoveryTableTransport`, `RawFanTable` (128 bytes), `RawFanTelemetry` (first 16 bytes), `TelemetryTransport`.

## Ownership and fallback

A local-console user client exposes only apply, automatic, and heartbeat selectors. It holds a 20-second lease; the service refreshes it every three seconds. The driver refreshes manual-mode firmware keepalive every 30 seconds. Disconnect, sleep/shutdown, failed/repeated stalled manual telemetry, or lease expiry requests Auto. A fan targeted to zero/Auto is exempt from the manual stalled-fan check. The firmware's own keepalive timeout remains enabled.

The CPU firmware sample is marked valid only for a successful read in 1–125°C. While under manual control, a validated CPU sample of 85°C or more makes the bridge request maximum fans and stay in manual (`ThermalBoost = Yes`); it returns to the requested targets below 75°C. Auto is requested only if the maximum request itself fails. (Versions up to the first 0.5.0 build requested Auto at 90°C, which cancelled the app's own full-speed boost.) The app additionally boosts fan requests under serious/critical thermal pressure. Missing GPU temperature is never replaced with a CPU reading for curve control.

No HP CPU/GPU power-limit, clock, raw EC offset, or graphics-driver writes are made.

Protocol references: [Linux hp-wmi.c](https://github.com/torvalds/linux/blob/master/drivers/platform/x86/hp/hp-wmi.c) (board mapping and protocol), [OmenMon](https://github.com/OmenMon/OmenMon) (protocol investigation), and this machine's decompiled ACPI GCxx/WHCM routines. This document describes an independent implementation, not bundled third-party code.
