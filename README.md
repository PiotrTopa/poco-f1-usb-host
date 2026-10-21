# poco-f1-usb-host

![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)
![postmarketOS](https://img.shields.io/badge/postmarketOS-edge-009900.svg)

Turn a **Xiaomi POCO F1 (beryllium, SDM845)** running **postmarketOS** into a
**self-powered USB host**: the phone switches its USB-C port to host mode
(`dr_mode = "host"`) *and* sources its own **5 V VBUS**, so RP2040 boards, USB
serial adapters, pendrives, keyboards etc. enumerate when plugged in directly,
with no Y-cable, powered hub or external supply. It is two small POSIX shell
scripts and one device-tree node. You don't need to rebuild the kernel or add a
new driver.

## Why / how it works

Out of the box the mainline device tree for beryllium leaves the dwc3 controller
in `peripheral` mode (that is what gives postmarketOS its USB-network login at
`172.16.42.1`) and describes **no VBUS source**. Switching to host mode alone is
not enough: the phone does not power the port, and a plugged device either stays
dark or shows up as floating D+/D- noise (phantom full-speed device,
`error -71`).

The fix relies on a register-layout coincidence:

- The **PMI8998** PMIC (the SDM845 companion PMIC on the POCO F1) has a
  **USB-OTG boost converter at SPMI offset `0x1100`**.
- Its register layout is **identical to the PM8150B** OTG block: `CMD_OTG` at
  `+0x40` (bit 0 = `OTG_EN`), current limit at `+0x52`, `OTG_CFG` at `+0x53`.
- The in-tree regulator driver **`qcom_usb_vbus-regulator`**
  (`drivers/regulator/qcom_usb_vbus-regulator.c`, compatible
  `qcom,pm8150b-vbus-reg`) is **chip-agnostic**. It has no per-chip data and
  only uses the `reg` base address and the compatible string from the device tree.

So a regulator node under the PMI8998 SPMI node (`/soc@0/spmi@c440000/pmic@2`)
with `compatible = "qcom,pm8150b-vbus-reg"` and `reg = <0x1100>` makes the
stock driver drive the PMI8998 boost unchanged. With `regulator-always-on`, the
regulator core sets `OTG_EN` at boot, and the PMI8998 boosts 5 V onto the
USB-C port for as long as the phone is on.

The second piece is the USB role. `dwc3-qcom` on this kernel has **no runtime
role switch** for beryllium, so the role is fixed in the DTB through `dr_mode`
on `/soc@0/usb@a6f8800/usb@a600000`. With `dr_mode = "host"`, the dwc3 xHCI
host enumerates whatever is plugged in.

postmarketOS boots beryllium with an **appended DTB**
(`deviceinfo_append_dtb="true"`). Both scripts therefore edit the DTB file in
`/boot/dtbs/qcom/` and run `mkinitfs`, which repacks and flashes `boot.img`.
After a reboot the change is live.

Tested on 2026-06-05: a USB pendrive enumerated at high speed (4 MB read at
13.4 MB/s), with no over-current and the regulator in state `enabled`.

## Repository contents

| File | Purpose |
|---|---|
| `poco-usb` | Sets dwc3 `dr_mode` to `host` or `peripheral` in the DTB (`fdtput`), then runs `mkinitfs` |
| `poco-vbus` | Adds or removes the PMI8998 VBUS regulator node (idempotent decompile, insert, recompile), then runs `mkinitfs`, with backups |
| `pmi8998-usb-vbus.dtso` | The device-tree node with an explanatory header (reference; `poco-vbus` inserts the same node) |

## Requirements

- Xiaomi POCO F1 (beryllium) running **postmarketOS edge**
- Kernel package **`linux-postmarketos-qcom-sdm845`**
- Appended DTB at `/boot/dtbs/qcom/sdm845-xiaomi-beryllium-tianma.dtb` or
  `/boot/dtbs/qcom/sdm845-xiaomi-beryllium-ebbg.dtb` (Tianma or EBBG panel)
- Packages: `dtc` (provides `dtc`, `fdtget`, `fdtput`); `mkinitfs` is already
  installed on postmarketOS. Optional: `usbutils` for `lsusb`.
- Root (`sudo`)
- **A working non-USB way in** (Wi-Fi + SSH, or the on-device terminal). Host
  mode turns off the USB-network login. See [Caveats](#caveats).

### DTB auto-detection

Both scripts pick the DTB in this order:

1. `POCO_DTB` environment variable, if set (full path, explicit override)
2. `/proc/device-tree/model`: contains `Tianma` or `EBBG`
3. `deviceinfo_dtb` in `/usr/share/deviceinfo/deviceinfo` (or `/etc/deviceinfo`),
   if it names exactly one variant
4. whichever of the two DTB files exists, if only one does

If none of these settles it, the scripts stop without changing anything. Both
scripts print the DTB they use on every run. Override example:

```sh
sudo POCO_DTB=/boot/dtbs/qcom/sdm845-xiaomi-beryllium-ebbg.dtb poco-vbus status
```

## Install

On the phone (over Wi-Fi SSH or in a local terminal):

```sh
sudo apk add dtc usbutils
sudo install -m 0755 poco-usb poco-vbus /usr/local/sbin/

sudo poco-usb host            # dr_mode -> host (DTB + boot.img)
sudo poco-vbus apply --reboot # add the VBUS regulator node, flash, reboot
```

Order doesn't matter, and each script is idempotent: running it again does
nothing if its change is already there. Both changes take effect on the next
boot. On that boot, VBUS is on and the port is a host.

## Switching back

```sh
sudo poco-usb device --reboot
```

This sets `dr_mode = "peripheral"` again and restores the postmarketOS
USB-network login (`ssh user@172.16.42.1` from a PC over the cable). The VBUS
node can stay. To also stop the phone sourcing 5 V, which you should do before
plugging it into a PC as a device, run:

```sh
sudo poco-vbus revert --reboot
```

## Verify

After the reboot:

```sh
sudo poco-usb status
sudo poco-vbus status
```

Or by hand:

```sh
ls /proc/device-tree/soc@0/spmi@c440000/pmic@2/ | grep vbus          # node present
grep -l usb_vbus /sys/class/regulator/*/name                         # regulator registered
cat /sys/class/regulator/<that-regulator>/state                      # -> enabled
tr -d '\0' < /proc/device-tree/soc@0/usb@a6f8800/usb@a600000/dr_mode # -> host
lsusb && ls /dev/sd* /dev/ttyACM* /dev/ttyUSB* 2>/dev/null           # plugged device enumerates
```

## Caveats

- **Host mode turns off the USB-network login** (`172.16.42.1`). Before running
  `poco-usb host`, make sure Wi-Fi and SSH work (or that you have a local
  terminal). Otherwise you lose your access path until you can get a shell another way.
- **Re-apply after kernel updates.** A `linux-postmarketos-qcom-sdm845` upgrade
  reinstalls a pristine DTB, which removes **both** changes: `dr_mode` goes back
  to `peripheral` and the VBUS node disappears. Run `sudo poco-usb host` and
  `sudo poco-vbus apply --reboot` again.
- **Tianma vs EBBG.** The DTB that gets patched must be the one your panel
  actually boots. Auto-detection handles the usual cases; if in doubt, check
  `cat /proc/device-tree/model` and set `POCO_DTB` explicitly.
- **VBUS is always on.** The phone sources 5 V permanently, even with nothing
  plugged in, which costs some battery. Do not connect the phone to another host
  (PC, charger acting as a source) while the VBUS node is active and the port is
  in host mode. Revert first.
- The current limit range in the node is 0.5 to 3 A. The real budget is limited
  by the PMI8998 boost and the battery. Use a powered hub for hungry devices.
- `mkinitfs` reflashes the boot partition. The scripts back up `boot.img` and the
  DTB before every change (see below).

## Rolling back

Every change first copies `/boot/boot.img` and the DTB to a timestamped
directory:

```
/root/boot-backup/usb-YYYYmmdd-HHMMSS/    # from poco-usb
/root/boot-backup/vbus-YYYYmmdd-HHMMSS/   # from poco-vbus
```

To undo the changes cleanly, run `sudo poco-vbus revert` and
`sudo poco-usb device`, then reboot. To restore a DTB from a backup by hand:

```sh
sudo cp -a /root/boot-backup/vbus-<ts>/sdm845-xiaomi-beryllium-<panel>.dtb.pre-vbus \
           /boot/dtbs/qcom/sdm845-xiaomi-beryllium-<panel>.dtb
sudo mkinitfs && sudo reboot
```

If the phone no longer boots, flash the saved `boot.img.pre-*` from fastboot:
`fastboot flash boot boot.img.pre-vbus`.

## License

MIT. See [LICENSE](LICENSE).

---

**Keywords:** POCO F1, beryllium, SDM845, PMI8998, PM8150B, postmarketOS, USB OTG host, VBUS, dr_mode, dwc3, qcom_usb_vbus-regulator, device tree, mainline Linux
