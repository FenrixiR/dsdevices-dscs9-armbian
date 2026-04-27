# DSDevices DSCS9 — Armbian Support

Community-maintained Armbian support files for the **DSDevices DSCS9**, an Android Digital Signage box based on the **Amlogic S912** SoC.

## Hardware

| Component | Details |
|-----------|---------|
| SoC | Amlogic S912 (Cortex-A53 octa-core) |
| RAM | 2GB DDR3 |
| eMMC | 16GB (Android) |
| WiFi/BT | Ampak AP6255 (BCM43455) — dual-band 802.11ac + BT 4.2 |
| Ethernet | 100Mbps |
| Board ID | M12S-V1.1 (2017/03/18) |

## Status

| Feature | Status | Notes |
|---------|--------|-------|
| Ethernet | ✅ Working | |
| WiFi 2.4GHz | ✅ Working | |
| WiFi 5GHz | ✅ Working | |
| SD card boot | ✅ Working | Requires patched `boot.cmd` — see below |
| USB | ✅ Working | |
| HDMI | ✅ Working | |
| IR Receiver | ✅ Working | NEC protocol, hardware works out of the box. Keymap configuration required for actual use — see IR section below |
| Bluetooth | ✅ Working | BCM4345C0 fully initialized, firmware loaded, scanning and pairing confirmed |
| Audio (HDMI) | 🔲 Untested | |
| Audio (3.5mm / SPDIF) | 🔲 Untested | |
| GPU (Mali-T820) | 🔲 Untested | |
| HDMI CEC | 🔲 Untested | Node present in Android DTB |
| eMMC install | ⚠️ Untested | Android eMMC left intact |

## Armbian Image

Use the ophub Armbian S912 server image:

**`Armbian_*_amlogic_s912_*_server_*.img`**

Available at: https://github.com/ophub/amlogic-s9xxx-armbian/releases

## Installation

### 1. Flash the image to SD card

Use Balena Etcher or `dd` to flash the Armbian S912 image to your SD card.

### 2. Replace the DTB

Copy `dtb/meson-gxm-dscs9.dtb` to the `BOOT` partition:

```
/dtb/amlogic/meson-gxm-dscs9.dtb
```

### 3. Update uEnv.txt

Edit `uEnv.txt` on the BOOT partition and set:

```
FDT=/dtb/amlogic/meson-gxm-dscs9.dtb
```

### 4. Fix SD card boot (boot.cmd)

The default `boot.cmd` only scans `mmc 1` for `uEnv.txt`, which is the eMMC in this u-boot's device numbering. The SD card slot is on a different mmc index. Edit `boot.cmd` and change:

```
setenv l_mmc "1"
```

to:

```
setenv l_mmc "0 1 2"
```

Then recompile:

```bash
mkimage -C none -A arm -T script -d /boot/boot.cmd /boot/boot.scr
```

### 5. u-boot

Rename `u-boot-s905x-s912.bin` to `u-boot.ext` on the BOOT partition.

### 6. Trigger boot from SD

Some devices may boot directly from SD card, but if your particular device doesn't, use the toothpick method (hold reset button in AV port while powering on) to trigger u-boot to boot from SD.

## WiFi Firmware

The AP6255 uses the `brcmfmac` driver. Required firmware files should already be present in Armbian. If WiFi does not appear, ensure the following exist in `/lib/firmware/brcm/`:

- `brcmfmac43455-sdio.bin`
- `brcmfmac43455-sdio.txt`

## Bluetooth

Bluetooth uses the BCM4345C0 chip (part of the AP6255 combo module) over UART_A (`serial@84c0`, `/dev/ttyAML1`).

The key to getting BT working was enabling CTS/RTS hardware flow control — without it the chip never responds. The DTB node requires:

- Both `uart_a` and `uart_a_cts_rts` pinctrl entries
- `uart-has-rtscts` property
- `compatible = "brcm,bcm43438-bt"`
- `shutdown-gpios` on GPIO 96 (GPIOX_23), active high
- Firmware: `BCM4345C0.hcd` (already present in Armbian's `/lib/firmware/brcm/`)

This was discovered by cross-referencing with the Phicomm-T1 DTB (`meson-gxm-phicomm-t1.dtb`), which is another S912 board with the same AP6255 chip and working Bluetooth.

## IR Receiver

The IR receiver is functional out of the box via the `meson-ir` driver on GPIOAO_7. To use it:

1. Enable NEC protocol: `sudo ir-keytable -s rc1 -p nec`
2. Test and capture scancodes: `ir-keytable -t -s rc1`
3. Create a keymap in `/etc/rc_keymaps/` (TOML format)
4. Load keymap: `sudo ir-keytable -s rc1 -p nec -c -w /etc/rc_keymaps/your_keymap.toml`

Any NEC protocol remote will work. For function binding (e.g., shutdown on KEY_POWER), use `triggerhappy` or `udev` rules.

## eMMC Partition Map

The Android eMMC uses Amlogic's proprietary EPT format, invisible to standard tools (fdisk/gdisk/parted). Use `ampart` to read it. See `docs/emmc_partitions.md` for the full partition map and mount instructions.

## Repository Structure

```
dsdevices-dscs9-armbian/
├── dtb/
│   ├── meson-gxm-dscs9.dtb      # Ready-to-use compiled DTB
│   └── src/
│       └── meson-gxm-dscs9.dts  # Decompiled DTS source
├── android-dtb/                  # DTB extracted from stock Android firmware
│   ├── dscs9_1g.dtb
│   ├── dscs9_dtb2.img
│   └── src/
│       └── dscs9_1g.dts          # Decompiled DTS source
├── docs/
│   ├── hardware.md               # Detailed hardware notes
│   └── emmc_partitions.md        # Android eMMC partition map
├── firmware/                     # Board-specific firmware files (TBD)
└── boot/                         # Boot configuration templates (TBD)
```

## Known Issues

- **Boot from SD**: Requires patching `boot.cmd` as described above. The stock script only scans eMMC.

## Contributing

PRs welcome. If you have a UART console (GND/RX/TX pads are exposed on the PCB near the AP6255 module) and can capture u-boot or kernel output, that would be helpful for further debugging.

## License

GPL-2.0-only — consistent with the Linux kernel DTB licensing.
