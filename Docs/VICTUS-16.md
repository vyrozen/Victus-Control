# HP Victus 16 (board 8BC2) notes

This fork extends the original 8C78 driver to the HP Victus 16 with PCI subsystem `103C:8BC2`.

## Status

| Item | Result |
| --- | --- |
| Machine | HP Victus 16, Intel Core i7-13700HX, board 8BC2 |
| OS | Hackintosh on OpenCore, Darwin 25.6 |
| Install method | Manual kext install (not the combined PKG) |
| CPU/GPU RPM readback | Working |
| Manual fan control | Working, reported by the fork author after hardware testing |
| Not recorded | BIOS version, GPU model, sleep/wake and long-run behavior |
| Not tested | The combined PKG installer, and the power helper, on this board |

One working laptop is not certification for other Victus models.

## What was different from the 8C78 driver

The WMI transport and commands are the same (`0x2D` RPM, `0x2F` fan table, `0x2E` target, `0x27` clear maximum). Two things differed, and both stopped RPM from being read at all:

1. **Board check.** The kext only accepted subsystem `8C78`. With `8BC2` it never started reading, so `BoardVerified` stayed `No` and `CPUFanRPM`/`GPUFanRPM` never appeared. The check now accepts `8C78` and `8BC2`, and the matched ID is published as `PhysicalBoard`.
2. **Fan table layout.** The 8BC2 table begins with a "fans off" row `(CPU 0, GPU 0, noise 15 dB)`. The original validator required every row to be 10–100 and rejected the whole table. The validator now skips rows where both fan values are 0.

The decoded 8BC2 table:

| Fan | Minimum | Maximum |
| --- | --- | --- |
| CPU | 1,800 RPM | 5,800 RPM |
| GPU | 2,100 RPM | 6,100 RPM |

A table that still fails validation no longer disables RPM display. The driver then sets `TelemetryReadOnly = Yes`, keeps reading RPM, and refuses all writes.

## Verify your install

```sh
ioreg -r -c OmenFanBridge -l | grep -E 'BoardVerified|DiscoveryTableOK|FanWritesSupported|TelemetryReadOnly|CPUFanMinRPM|CPUFanMaxRPM|GPUFanMinRPM|GPUFanMaxRPM|CPUFanRPM|GPUFanRPM'
```

Expected on a working Victus 16:

```
BoardVerified      = Yes
DiscoveryTableOK   = Yes
FanWritesSupported = Yes
TelemetryReadOnly  = No
CPUFanMinRPM       = 1800     CPUFanMaxRPM = 5800
GPUFanMinRPM       = 2100     GPUFanMaxRPM = 6100
```

`CPUFanRPM` and `GPUFanRPM` can legitimately be `0` at idle, because the firmware may stop the fans in Auto.

## First manual-control test

1. Stay at the machine and watch temperatures.
2. Apply a moderate value such as 50% to both fans and confirm RPM follows within a few seconds.
3. Return to **Auto** and confirm RPM falls back.
4. Quit the app and confirm the fans return to Auto.
5. Sleep and wake the laptop, then confirm Auto after wake.

If Apply fails, the app reports the failing stage, transport status and firmware status. Include those values in a bug report.

## Troubleshooting

| Symptom | Likely cause |
| --- | --- |
| `BoardVerified = No` | Old kext still loaded, or a different board. Check `ioreg -l -w0 \| grep subsystem-id` against `103C:8BC2`. |
| `DiscoveryTableOK = No`, `TelemetryReadOnly = Yes` | Firmware table did not validate. Report `RawFanTable`. |
| `TelemetryHealthy = No` | The RPM query failed. Report `TelemetryTransport` and `LastFirmwareStatus`. |
| `RawFanTelemetry` all zero while fans spin | This board may not use query `0x2D`. Report it. |
| App says to reboot | The loaded `BridgeVersion` is not `0.3.1`, `0.4.0` or `0.5.0`. |

## Adding another board

A board needs more than an ID in the allowlist:

1. Read its PCI subsystem ID from `ioreg` and add it to `supportedBoard()` in `Driver/OmenFanBridge.cpp` and `verifyBoard()` in `Installer/Tools/main.swift`.
2. Install it first without expecting writes: if the table does not validate, the driver falls into `TelemetryReadOnly` and refuses writes. Check `RawFanTable` and `RawFanTelemetry` against the Linux `hp-wmi` driver's behavior for that board.
3. Add a regression test for the table in `Driver/test-fan-policy.cpp`.
4. Test manual control attended, with a recovery USB ready.

Linux maps 8BC2 to its `omen_v1` board parameters, which use the same `0x2D`/`0x2E`/`0x2F` fan queries as the other supported boards. That is why the transport worked unchanged here.
