# PMW3360 SROM mods: no smoothing, fixed frame rate

Modified versions of PixArt's PMW3360 SROM `0x04`, the firmware blob most open-source mouse firmware uploads to the sensor at boot.

| File | Smoothing | Frame rate |
|---|---|---|
| `srom/pmw3360_0x04_nosmooth.bin` | **off** | stock (dynamic) |
| `srom/pmw3360_0x04_nosmooth_8k.bin` | **off** | **fixed ~8000 FPS** |
| `srom/pmw3360_0x04_nosmooth_11k.bin` | **off** | **fixed ~11000 FPS** |

Each `.bin` has a matching `.h`: the same bytes as a C array.

> **Untested on a production mouse.** These are static patches. They pass the sensor's SROM load check by construction (see below), but no tracking tests have been run.

## Using them

They are drop-in replacements for the stock `0x04` blob:
- Use the same SROM download sequence as the datasheet.
- `SROM_ID` still reads `0x04`, so drivers that check it are happy.
- No driver changes are needed.

Example: in QMK, replace the contents of `pmw33xx_firmware_data[]` in `drivers/sensors/pmw3360.c` with the array from the `.h` you want.

Recommended: keep rest mode off (Config2 = `0x00`). Most drivers already do this.

## What is changed

**Smoothing (all three).** The stock SROM runs motion through a moving average over the last 2^level frames, and picks the level from CPI:

| Condition | Level | Frames averaged |
|---|---|---|
| ≤ 2000 CPI | 1 | 2 |
| > 2000 CPI | 5 | 32 |
| Angle snap on | ≥ 6 | 64+ |

At high frame rates level 5 adds ~1.3 ms of lag, and level 6 ~2.7 ms. These builds force level 0, so each report is the raw motion of that frame. The filter carries its rounding remainder forward, so level 0 loses no counts.

**Frame rate (8k / 11k).** The stock SROM sets the frame period from exposure time: darker surface, longer exposure, lower FPS. The fixed builds:
- remove that override;
- replace the slow-mode constant (~3.4k FPS);
- re-apply a fixed period after every frame interrupt and every main-loop pass.

That last step also covers the mask ROM's own power-mode code, which reloads the period on mode changes and Config2 writes.

| Build | Period (sensor clock ticks) |
|---|---|
| 8k | `0x2009` |
| 11k | `0x174D` |

The FPS figures assume a 65.6 MHz sensor clock, measured on one PMW3360. The clock varies a little between chips, so expect the real rate to be within a few percent. The stock firmware never goes faster than period `0x163F` (~11.7k FPS), and both builds stay above that.

**Caveat: a fixed frame rate caps exposure.** The sensor can't expose longer than the frame period leaves room for. On dark or glossy pads the 11k build may track worse than stock. If SQUAL (register `0x07`) drops a lot compared with the stock SROM, use the 8k build or the dynamic one.

## Why the sensor still accepts them

The 2-byte trailer is not a CRC of the data. The sensor's mask ROM steps the decryption keystream once per stored byte and checks that the final state XORed with the trailer equals `0xBEEF`. That depends only on the seed (the first two bytes) and the length. These builds keep both, so the stock trailer stays valid.

## Credits

- Stock blob: [openinput-fw/sensor-blobs](https://github.com/openinput-fw/sensor-blobs).
- SROM encryption scheme: first described by [8051enthusiast](https://8051enthusiast.github.io/2020/04/14/002-Sensor_Firmware.html).

PMW3360 and its firmware are PixArt Imaging's. This project is not affiliated with PixArt.
