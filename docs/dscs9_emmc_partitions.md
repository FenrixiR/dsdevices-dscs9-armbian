# DSCS9 eMMC Partition Map

## Device Info
- **Block device:** `/dev/mmcblk2`
- **Total size:** 14.56 GiB (15,634,268,160 bytes)
- **Partition format:** Amlogic proprietary EPT (not MBR/GPT — invisible to fdisk/gdisk/parted)
- **Tool used:** `ampart`
- **SoC:** GXM (S912), platform `m12s`, variants `1g` and `2g`

---

## EPT (Amlogic Partition Table)

| ID | Name        | Offset (hex)  | Offset (human) | Size (hex)   | Size (human) | Masks |
|----|-------------|---------------|----------------|--------------|--------------|-------|
|  0 | bootloader  | `0x00000000`  | 0.00 B         | `0x00400000` | 4.00 MB      | 0     |
|  1 | reserved    | `0x02400000`  | 36.00 MB       | `0x04000000` | 64.00 MB     | 0     |
|  2 | cache       | `0x06C00000`  | 108.00 MB      | `0x20000000` | 512.00 MB    | 2     |
|  3 | env         | `0x27400000`  | 628.00 MB      | `0x00800000` | 8.00 MB      | 0     |
|  4 | logo        | `0x28400000`  | 644.00 MB      | `0x02000000` | 32.00 MB     | 1     |
|  5 | recovery    | `0x2AC00000`  | 684.00 MB      | `0x02000000` | 32.00 MB     | 1     |
|  6 | rsv         | `0x2D400000`  | 724.00 MB      | `0x00800000` | 8.00 MB      | 1     |
|  7 | tee         | `0x2E400000`  | 740.00 MB      | `0x00800000` | 8.00 MB      | 1     |
|  8 | crypt       | `0x2F400000`  | 756.00 MB      | `0x02000000` | 32.00 MB     | 1     |
|  9 | misc        | `0x31C00000`  | 796.00 MB      | `0x02000000` | 32.00 MB     | 1     |
| 10 | instaboot   | `0x34400000`  | 836.00 MB      | `0x20000000` | 512.00 MB    | 1     |
| 11 | boot        | `0x54C00000`  | 1.32 GB        | `0x02000000` | 32.00 MB     | 1     |
| 12 | system      | `0x57400000`  | 1.36 GB        | `0x40000000` | 1024.00 MB   | 1     |
| 13 | data        | `0x97C00000`  | 2.37 GB        | `0x30C200000`| 12.19 GB     | 4     |

---

## Mounting Partitions from Armbian

Since the eMMC uses Amlogic's proprietary partition format, partitions must be mounted using byte offsets:

```bash
# Mount Android system partition (read-only)
sudo mkdir -p /mnt/android_system
sudo mount -o ro,offset=$((0x57400000)) /dev/mmcblk2 /mnt/android_system

# Mount Android data partition (read-only)
sudo mkdir -p /mnt/android_data
sudo mount -o ro,offset=$((0x97C00000)) /dev/mmcblk2 /mnt/android_data

# Unmount when done
sudo umount /mnt/android_system
sudo umount /mnt/android_data
```

> **Note:** The `reserved` partition at offset `0x02400000` contains the Android DTB.
> It can be extracted with:
> ```bash
> sudo dd if=/dev/mmcblk2 of=android_reserved.img bs=512 skip=$((0x2400000/512)) count=$((0x4000000/512))
> ```

---

## Notes
- The `reserved` partition contains two DTB copies (gxm_m12s_1g and gxm_m12s_2g)
- Both DTB copies had invalid checksums when read by ampart — first copy was used
- The DTB offset within the `reserved` partition is at `0x4000000` (4MB in)
- `bootloader` and `reserved` have mask `0` — protected, do not overwrite
- `data` partition is AUTOFILL — takes all remaining space
