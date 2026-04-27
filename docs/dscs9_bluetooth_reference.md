# DSCS9 Android Bluetooth Configuration Reference

## Key Finding
Android uses **/dev/ttyS1** for Bluetooth HCI — this maps to:
- Android: `serial@c11084c0`
- Armbian: `serial@84c0` (uart_A, currently **disabled** in Armbian DTB)

This is **NOT** UART_AO_A (`serial@4c0` / ttyAML0) which is the console UART.

## Armbian DTB Changes Required
To enable BT on the correct UART, `serial@84c0` needs:
1. `status = "okay"`
2. `pinctrl-0 = <0x82>` (uart_a pinctrl, phandle for `uart_tx_a` + `uart_rx_a`)
3. `pinctrl-names = "default"`
4. A `bluetooth` child node with `compatible = "brcm,bcm4329-bt"` and `reset-gpios`

BT reset GPIO: pin `0x60` (96) on periphs bank (phandle `0x24`) = **gpio-619** in sysfs.

## Android bt_vendor.conf
Source: `/system/etc/bluetooth/bt_vendor.conf`

```
UartPort = /dev/ttyS1
FwPatchFilePath = /etc/bluetooth/
```

## Android bt_stack.conf
Source: `/system/etc/bluetooth/bt_stack.conf`
- BtSnoop logging: disabled
- All trace levels set to WARNING (2)

## UART Mapping Reference
| Android device | Armbian node     | Armbian alias | Role            |
|----------------|------------------|---------------|-----------------|
| /dev/ttyS0     | serial@c81004c0  | uart_AO       | Console         |
| /dev/ttyS1     | serial@c11084c0  | uart_A        | **Bluetooth**   |
| /dev/ttyS4     | serial@c81004e0  | uart_AO_B     | unused          |

## Firmware
BCM4345C0 firmware files present in Armbian at `/lib/firmware/brcm/`:
- `BCM4345C0.hcd` — generic
- `BCM4345C0_003.001.025.0162.0000_Generic_UART_37_4MHz_wlbga_ref_iLNA_iTR_eLG.hcd`

A device-specific firmware symlink may be needed:
```bash
ln -s BCM4345C0.hcd /lib/firmware/brcm/BCM4345C0.dsdevices,dscs9.hcd
```
