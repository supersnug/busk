# busk

**busk is the RP2040 bootloader for [USK](https://github.com/supersnug/usk)**,
the Picofly modchip firmware. It selects a stored USK image, checks it, copies
it into RAM, and starts it. Its two-slot and watchdog retry scheme lets USK
try a firmware update while retaining an earlier image as a fallback.

Busk runs on the modchip, not the Switch's main processor. It does not glitch
the console, install console boot data, or load files from the SD card: those
are responsibilities of USK and its separate console-side loader.

## Boot sequence

The implementation is in [main.c](main.c):

1. Pull down the possible glitch-output pins and disable the reset pad's
   input/output paths to avoid unintentionally driving the console during boot.
2. Set the RP2040 regulator to 1.30 V and its system clock to 200 MHz.
3. Read the flash-backed selection counter and choose a firmware slot according
   to its parity and the watchdog retry state.
4. Arm a 100-ms watchdog and record the selected slot in watchdog scratch
   register 1.
5. Check that the image body size is nonzero and no greater than `0x38000`
   bytes (224 KiB).
6. Copy the body to SRAM at `0x20000000`, calculate its CRC-32, and compare it
   with the stored header value. A size or CRC failure forces a watchdog reset.
7. Call Thumb address `0x20000001` to enter the loaded firmware. USK disables
   the inherited watchdog at startup.

This is a direct call to the loaded image's entry point, not a generic vector
table handoff. The image must be built for USK's RAM execution layout. CRC-32
detects corruption; it is not a signature or proof of the image's origin.

## Flash layout and image format

These are offsets in the RP2040's external flash, mapped at `0x10000000`,
not addresses in the console's eMMC:

| Flash offset | Purpose |
| --- | --- |
| `0x00000` | busk and its RP2040 second-stage boot code |
| `0x08000` | USK configuration and timing records; not interpreted by busk |
| `0x0f000` | 256-byte slot-selection counter, within a dedicated erase sector |
| `0x10000` | Firmware slot 0 |
| `0x48000` | Firmware slot 1 |

Each slot begins with the [header defined in firmware.h](firmware.h):

```text
offset 0:  uint32 little-endian executable body size
offset 4:  uint32 little-endian CRC-32 of that body
offset 8:  executable body bytes
```

The CRC implementation is in [crc.c](crc.c). USK's packaging script produces
this header using `zlib.crc32()`. Busk copies only the body, not its header,
into RAM. The source's size check permits up to `0x38000` body bytes; that
check is not a comprehensive validation of the flash layout or image contents.

## Slot selection, retries, and rollback

`count_fuses()` counts cleared bits in the 256-byte counter area. Despite its
name, this is **ordinary RP2040 flash state**, not console hardware fuses.
An even count prefers slot 0; an odd count prefers slot 1. Busk reads this
counter but does not increment it or write firmware slots; USK does that.

For a fresh boot, busk sets watchdog scratch register 0 to zero and tries the
preferred slot. A watchdog reset advances the retry state using
`(preferred_slot ^ previous_attempt ^ 1) & 1`:

| Scratch register 0 on handoff | Selected slot |
| --- | --- |
| `0` | Preferred slot |
| `1` | Other slot |
| `2` | Preferred slot |
| `3` | Other slot |

If another watchdog reset arrives with the previous attempt already at 3,
busk halts instead of trying again. The source recognizes a fresh boot when
either `watchdog_enable_caused_reboot()` or `watchdog_caused_reboot()` is false;
otherwise it follows this retry path. A power-cycle starts a fresh sequence.

USK reads the attempt from scratch register 0 and the running slot from
scratch register 1. When USK stages an update into the other slot and forces
a watchdog reboot, busk can try that image without first changing the preferred
slot. If it cannot validate or start, subsequent retries can return to the
earlier image. After USK reaches glitch success, it adjusts the counter parity
to make the running slot preferred.

This mechanism is not a guarantee of recovery from every failure. Busk checks
size and CRC before calling the image; once USK disables the inherited
watchdog, busk is no longer supervising its progress.

## Terminal failure behavior

After exhausting retries, `halt()` disables the GPIO pads while retaining
pulldowns on the possible glitch outputs. `zzz()` reduces clocks and regulator
settings, disables oscillators, brownout detection, and SRAM banks, then loops.
Busk has no LED error-code interface, USB update service, or console shutdown
command. Firmware updates and normal status indication belong to USK.

## Building and packaging

[CMakeLists.txt](CMakeLists.txt) builds `main.c` and `crc.c` with the Pico SDK,
disables USB/UART stdio, and uses the default flash binary type. It selects a
second-stage bootloader with `PICO_FLASH_SPI_CLKDIV=4`.

**Busk must reserve the upper SRAM region for its own runtime**, leaving the
lower SRAM available for the USK image it copies. The historical build changes
the Pico SDK's `memmap_default.ld` RAM definition as follows:

```diff
-    RAM(rwx) : ORIGIN =  0x20000000, LENGTH = 256k
+    RAM(rwx) : ORIGIN =  0x20038000, LENGTH = 32k
```

This linker change is external to this repository's CMake configuration; a
plain default-layout build does not apply it automatically. The 32-KiB region
holds busk's RAM data and stack; it does not mean its executable is copied
there in the same way as USK. The required `pico_sdk_import.cmake` must also be
provided from the SDK.

The [USK build workflow](https://github.com/supersnug/usk/blob/main/.github/workflows/build.yml)
builds busk with that linker layout, restores the SDK linker script, and then
builds USK. USK's
[`prepare.py`](https://github.com/supersnug/usk/blob/main/prepare.py) combines
`busk.bin` at flash offset zero with a header-prefixed USK image in slot 0 to
produce `firmware.uf2`. Its separate `update.bin` contains the USK header/body,
not busk. Building busk alone does not produce that combined firmware package.

## Related documentation

- [USK overview](https://github.com/supersnug/usk): modchip firmware and boot-chain roles.
- [USK implementation](https://github.com/supersnug/usk/blob/main/IMPLEMENTATION.md): MCU-side updates, configuration, glitching, and packaging.
- [USK 2.80 build snapshot](https://github.com/supersnug/usk/blob/main/SNAPSHOT.md): pinned sources/toolchain and verified release hashes.
- [Independent copies and development](https://github.com/supersnug/usk/blob/main/FORKING.md): copying the required repositories and adapting the build.

Busk originates from [Rehius's busk](https://github.com/rehius/busk).
See [LICENSE](LICENSE) for the repository's license.
