# DSDevices DSCS9 — Armbian Support

Community-maintained Armbian support files for the **DSDevices DSCS9**, an Android Digital Signage box based on the **Amlogic S912** SoC.

## Hardware

| Component | Details |
|-----------|---------|
| SoC | Amlogic S912 (Cortex-A53 octa-core) |
| RAM | 2GB DDR3 |
| eMMC | 16GB (Android) |
| WiFi/BT | Ampak AP6255 (BCM43455) — dual-band 802.11ac + BT 4.2 |
| Ethernet | Gigabit |
| Board ID | M12S-V1.1 (2017/03/18) |

## Status

| Feature | Status | Notes |
|---------|--------|-------|
| Ethernet | ✅ Working | Gigabit, RTL8211F PHY |
| WiFi 2.4GHz | ✅ Working | |
| WiFi 5GHz | ✅ Working | |
| SD card boot | ✅ Working | Requires patched `boot.cmd` — see below |
| USB | ✅ Working | |
| HDMI video | ✅ Working | |
| HDMI audio | ✅ Working | Via Amlogic AIU driver, works out of the box |
| IR receiver | ✅ Working | NEC protocol via `meson-ir` driver on GPIOAO_7. Keymap config required — see IR section |
| Bluetooth | ✅ Working | BCM4345C0, firmware loaded, scanning and pairing confirmed |
| HDMI CEC | ✅ Working | `meson_ao_cec` driver, `/dev/cec0` available |
| GPU (Mali-T820) | ✅ Working | Panfrost driver, scales 125–720MHz. glmark2 score ~145 |
| VPU | ✅ Working | Hardware decode: H.264, H.265, VP9 via `meson-vdec` |
| Hardware crypto | ✅ Working | AES-ECB, AES-CBC hardware accelerated via `gxl-crypto` |
| vRTC | ✅ Working | Wall-power only, no battery backup. Use NTP for accurate time |
| Audio 3.5mm | 🔲 Untested | T9015 DAC present, output only (no microphone) |
| Audio SPDIF | 🔲 Untested | Node present in DTB |
| USB OTG | 🔲 Untested | Hardware supports it, needs testing with USB-A to USB-A cable |
| HYM8563 RTC | 🔲 No battery | Chip may be present, battery holder not populated |
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
- `shutdown-gpios` on GPIO 96 (periphs bank), active high
- Firmware: `BCM4345C0.hcd` (already present in Armbian's `/lib/firmware/brcm/`)

This was discovered by cross-referencing with the Phicomm-T1 DTB (`meson-gxm-phicomm-t1.dtb`), which is another S912 board with the same AP6255 chip and working Bluetooth.

## IR Receiver

The IR receiver is functional out of the box via the `meson-ir` driver on GPIOAO_7. The driver initializes as `rc1` and supports multiple protocols — NEC must be explicitly enabled.

To use it:

```bash
# Enable NEC protocol
sudo ir-keytable -s rc1 -p nec

# Test and capture scancodes from your remote
ir-keytable -t -s rc1

# Create a keymap file at /etc/rc_keymaps/your_keymap.toml
# Load keymap
sudo ir-keytable -s rc1 -p nec -c -w /etc/rc_keymaps/your_keymap.toml
```

Any NEC protocol remote will work. For function binding (e.g., shutdown on KEY_POWER), use `triggerhappy` or `udev` rules.

## HDMI CEC

CEC works out of the box via the `meson_ao_cec` driver. No configuration needed.

```bash
# Install cec-utils
sudo apt install cec-utils

# List CEC devices on the bus
echo "scan" | cec-client -s -d 1

# Turn TV on
echo "on 0" | cec-client -s -d 1

# Set this device as active source (switch TV input)
echo "as" | cec-client -s -d 1
```

## GPU

The Mali-T820 MP3 GPU is supported by the open-source **Panfrost** driver, already loaded in the Armbian kernel. Dynamic frequency scaling works correctly — the GPU scales from 125MHz up to 720MHz under load.

```
panfrost d00c0000.gpu: mali-t820 id 0x820
[drm] Initialized panfrost 1.2.0 for d00c0000.gpu on minor 1
```

Available frequencies: 125, 250, 285, 400, 500, 666, 720 MHz

**glmark2-es2-drm score: ~145**

Note: The S912 datasheet lists 666MHz as the rated GPU frequency. 720MHz is present in the stock Android DTB and has been confirmed stable at the same voltage (950mV).

## VPU

Hardware video decoding is handled by `meson-vdec` and exposed as `/dev/video0`. Supported codecs:

- H.264
- H.265 (HEVC)
- VP9

Video playback does not consume GPU or CPU resources.

## Hardware Crypto

The `gxl-crypto` engine provides hardware-accelerated AES-ECB and AES-CBC. Useful for VPN (OpenVPN uses AES-CBC), HTTPS, and disk encryption workloads.

## eMMC Partition Map

The Android eMMC uses Amlogic's proprietary EPT format, invisible to standard tools (fdisk/gdisk/parted). Use `ampart` to read it. See `docs/dscs9_emmc_partitions.md` for the full partition map and mount instructions.

## Repository Structure

```
dsdevices-dscs9-armbian/
├── dtb/
│   ├── meson-gxm-dscs9.dtb          # Ready-to-use compiled DTB
│   └── src/
│       └── meson-gxm-dscs9.dts      # Decompiled DTS source
├── android-dtb/                      # DTB extracted from stock Android firmware
│   ├── dscs9_1g.dtb
│   ├── dscs9_dtb2.img
│   └── src/
│       └── dscs9_1g.dts              # Decompiled DTS source
├── docs/
│   ├── dscs9_bluetooth_reference.md  # BT investigation notes and solution
│   └── dscs9_emmc_partitions.md      # Android eMMC partition map
├── firmware/                         # Board-specific firmware files (TBD)
└── boot/                             # Boot configuration templates (TBD)
```

## Known Issues / TODO

- **Audio 3.5mm/SPDIF**: T9015 DAC node present, untested — 3.5mm is output only, no microphone input.
- **USB OTG**: Hardware supports it, needs testing with USB-A to USB-A cable.
- **HYM8563 RTC**: External I2C RTC chip likely present but battery holder not populated. DTB node (i2c-gpio bit-bang) not yet added.
- **Boot from SD**: Requires patching `boot.cmd` as described above.

## Contributing

PRs welcome. Hardware notes: GND/RX/TX debug pads are exposed on the PCB near the AP6255 module — a USB-UART adapter soldered here would allow u-boot and kernel console capture, useful for further debugging.

## License

GPL-2.0-only — consistent with the Linux kernel DTB licensing.
