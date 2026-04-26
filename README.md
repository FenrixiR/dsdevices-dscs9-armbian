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
| Bluetooth | ❌ Not working | UART comms timeout; RX line may be unconnected |
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
│       └── dscs9_1g.dts  # Decompiled DTS source
├── firmware/                     # Board-specific firmware files (TBD)
├── boot/                         # Boot configuration templates (TBD)
└── docs/
    └── hardware.md               # Detailed hardware notes
```

## Known Issues

- **Bluetooth**: `hci_uart_bcm` attaches to the UART but times out on command `0xfc18` (baudrate negotiation). The BT UART RX line may not be connected to the SoC on this board revision. Investigation ongoing.
- **Boot from SD**: Requires patching `boot.cmd` as described above. The stock script only scans eMMC.

## Contributing

PRs welcome. If you have a serial/UART console attached and can capture u-boot or kernel output, that would be particularly helpful for debugging Bluetooth.

## License

GPL-2.0-only — consistent with the Linux kernel DTB licensing.
