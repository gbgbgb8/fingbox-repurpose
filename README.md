# Fingbox FIN-B-001 → Alpine, for an agent

Turn a Domotz/Fing Fingbox into an always-on Alpine Linux board that boots from its own eMMC and joins Wi-Fi. This is the procedure that worked on a real FIN-B-001. Follow it in order. The hard rules below are there because violating them bricks the bootloader or destroys the only backup.

Ask the owner, before you start, for three things this guide does not contain: the Wi-Fi SSID and passphrase, the root password they want, and which LAN the board should join. 

## What the board actually is

- Model FIN-B-001. Allwinner H3, 1 GB RAM, Samsung eMMC about 7.3 GB (`/dev/mmcblk0` on Linux 6.18, controller `1c11000.mmc`).
- U-Boot 2017.01 says the model is "Xunlong Orange Pi Plus 2E". That string is a lie. The board's own device tree is `sun8i-h3-domotz-fingbox-v2.dtb`. Do not boot with `sun8i-h3-orangepi-pc.dtb` or any other Orange Pi DTB.
- Ethernet is an external Realtek RTL8211E, `phy-mode = "rgmii"`, MDIO address 0. It is not the H3's internal PHY.
- Wi-Fi is an Atheros AR9271 USB device, `0cf3:9271`, on the EHCI controller at `1c1d000`. It is not the Orange Pi SDIO Wi-Fi. The chip's own EEPROM is blank. Calibration lives in a 1024-byte file on the stock system, `eeprom_ar9271.bin`.
- The empty microSD slot is a second MMC controller, `1c0f000.mmc`. It is unused and will spam the log. The eMMC is the other controller. Never unbind `1c11000.mmc`.

## Hard rules

- UART is 3.3 V. Connect GND to GND, adapter TX to board RX, adapter RX to board TX. Never connect the adapter's VCC or 3.3 V pin. Power the board from its own supply.
- Serial is 115200 8N1, no flow control. Do not send Ctrl-C. Closing the host serial file descriptor can SIGHUP the foreground job and drop the login. Background work on the board must `trap '' HUP`.
- Do not `saveenv`. Do not write a new U-Boot. Do not `dd` anything over the first 256 MiB of the eMMC.
- Do not format `mmcblk0p1`. It holds `uboot.env` and the stock `boot.scr`, which is the fallback if extlinux fails.
- Do not touch `mmcblk0p2` (stock recovery) or anything before sector 524288.
- Copy `eeprom_ar9271.bin` off the stock system before you wipe the root filesystem. It is this board's radio calibration. Another board's file is the wrong file.
- Read MAC addresses from this board's U-Boot environment. Do not reuse another board's MAC. `ethaddr` is the Ethernet MAC. `extra_kernel_cmd_line` contains `ath9k_htc.domotz_ath_hwaddr=...`, which is the Wi-Fi MAC.
- Back up the whole eMMC and verify the backup before any write.
- Do not expose the board to the internet.
- Never print Wi-Fi passphrases, root passwords, or API tokens into a log. Do not `cat` `wpa_supplicant.conf` back over serial.

## What to have on the bench

- CP2102 (or similar) 3.3 V USB serial adapter. On macOS it appears as `/dev/cu.usbserial-*`.
- The board's own power supply.
- A USB stick with the Alpine 3.24 armv7 U-Boot tarball unpacked on a FAT filesystem: kernel, initramfs, modloop, and the `apks/` tree. Known-good source: `alpine-uboot-3.24.3-armv7.tar`. The kernel this procedure was proven with is `6.18.58-0-lts`. A newer kernel changes the module vermagic and may change the Ethernet binding; if you use one, rebuild the Wi-Fi modules against that exact tree and re-check the `emac` node below.
- On the host: `dtc`, `arm-none-eabi-gcc` (a bare-metal GCC is fine; do not use zig or clang as the cross compiler), and GNU make (`gmake` on macOS). Homebrew `qemu` cannot run the armv7 userland.

## Serial

Open the port yourself. `screen` is awkward from an agent. This works from Python on the host:

- `os.open(path, os.O_RDWR | os.O_NOCTTY | os.O_NONBLOCK)`
- termios: `iflag = oflag = lflag = 0`, `cflag = CS8 | CREAD | CLOCAL`, speed `B115200`
- Run the host Python with `python3 -u`. Block-buffered prints look like a hung board.
- The board's shell is BusyBox ash. A heredoc sticks at `PS2`. Send one command per line.
- After `stty -echo`, the command is not echoed. A waiter that demands the marker twice will time out even though the command succeeded. Wait for one copy.
- Cooked tty input truncates past a few kilobytes. To push a file: `base64 -d > /path`, send base64 wrapped at 76 columns, finish with a newline and Ctrl-D (`\n\x04`). Pace the writes (512-byte chunks, about 8 ms between them). Check `md5sum` before you trust the file.
- BusyBox `ip` has no `-br`. Use `ip addr` and `ip link show`. BusyBox `timeout` is `timeout SECS CMD`, not `timeout -t`.

To interrupt U-Boot, spam the space bar for the 2-second `bootdelay`. The prompt is `=>`.

## Stock boot, then stop

Let one stock boot happen and save the log, or interrupt U-Boot immediately if you already have a log. Record `version`, `printenv`, `mmc list`, `mmc part`, and `usb start; usb storage`.

Expect:

- `bootcmd=run distro_bootcmd`, `boot_targets=fel mmc0 mmc1 usb0 pxe dhcp`
- `fdtfile=sun8i-h3-domotz-fingbox-v2.dtb`, `console=ttyS0,115200`
- Distro boot looks for extlinux before `boot.scr`. extlinux `TIMEOUT` is in tenths of a second (`TIMEOUT 20` is 2 seconds).
- Environment is `CONFIG_ENV_IS_IN_FAT`: `uboot.env` on p1. Leave it alone.

eMMC map, DOS, p1 starting at sector 524288 so the SPL and U-Boot stay in the gap in front of it:

| Partition | Label | Type | Role |
| --- | --- | --- | --- |
| mmcblk0p1 | system-boot | FAT, type `0c`, 128 MiB | Boot files. Keep the stock files. Add Alpine beside them. |
| mmcblk0p2 | system-recovery | ext4, about 1.4 GiB | Stock recovery. Do not mount it for writes. |
| mmcblk0p3 | writable | ext4, about 5.3 GiB | Stock Ubuntu Core data. Reuse this filesystem for Alpine. Do not `mkfs` it. |

The stock OS is Ubuntu Core (snappy): `core` snap plus `domotz-platform-kernel`. The kernel snap is where `eeprom_ar9271.bin` and the stock copy of `htc_9271-1.4.0.fw` live. Copy both off before you delete anything on p3. Confirm the EEPROM file is 1024 bytes and starts with bytes `5a a5` (little-endian `0xa55a`, `AR5416_EEPROM_MAGIC`).

## Backup

Write the whole eMMC to the USB stick (or to another machine), gzipped. FAT32's 4 GiB file limit is fine for the gzip. Verify with `gzip -dc backup.gz | cmp - /dev/mmcblk0` until `cmp` reports EOF. Keep that stick. It is the rollback. Do not put the only copy of the backup on the eMMC you are about to edit.

## Boot Alpine from USB without replacing U-Boot

The stick's extlinux will not find `sun8i-h3-domotz-fingbox-v2.dtb`. Load the Domotz DTB from eMMC p1 and `bootz` the stick's kernel, initramfs, and that DTB. `fdtfile` in the environment is already the right name.

Once the live system is up:

- The stick is `/dev/sda1` only if probe order says so. Identify it by label, not by assuming `sda`.
- `apk` repositories: `-X /media/usb/apks` is correct. `-X /media/usb/apks/armv7` makes apk look for `armv7/armv7` and fail. Pass `--allow-untrusted` for the stick's packages.
- Install at least `alpine-base`. The live image has no `mkfs.ext4`. You do not need it.

## Install onto p3 and boot from eMMC

Mount p3 and p1. Delete the Ubuntu Core data on p3. Keep `lost+found`. Do not run `mkfs`. The filesystem UUID stays whatever it already is; use that UUID in extlinux and fstab. Read yours. Do not assume it.

Copy the Alpine kernel, initramfs, and modloop to `/boot/alpine/` on p1 (`vmlinuz-lts`, `initramfs-lts`, `modloop-lts`). Copy the stock Domotz DTB to `/boot/alpine/fingbox.dtb` and keep an untouched `fingbox.dtb.orig` next to it.

`/boot/extlinux/extlinux.conf`:

```
TIMEOUT 20
PROMPT 1
DEFAULT alpine
LABEL alpine
  MENU LABEL Alpine
  KERNEL /alpine/vmlinuz-lts
  INITRD /alpine/initramfs-lts
  FDT /alpine/fingbox.dtb
  APPEND console=ttyS0,115200 root=UUID=<p3-uuid> rootfstype=ext4 rw rootwait modules=loop,squashfs,sd-mod,usb-storage,ext4
```

Do not put `modloop=` on the `APPEND` line. The squashfs hides `/lib/modules`, and you need a real modules directory on the ext4 root.

On the new root:

- `/etc/fstab` mounts p3 on `/` (`ext4 rw,noatime`) and p1 on `/boot` (`vfat defaults`). Plus `proc`.
- `/etc/hostname` of the owner's choosing.
- `/etc/inittab`: a getty on `ttyS0` at 115200, and the usual OpenRC `sysinit` / `boot` / `default` lines.
- Copy `/lib/modules` off the modloop squashfs onto the ext4 root, run `depmod -a`, and remove the `modloop` service from the sysinit runlevel (`rm /etc/runlevels/sysinit/modloop`).
- `/etc/local.d/unbind-sd.start`, mode 755: `echo 1c0f000.mmc > /sys/bus/platform/drivers/sunxi-mmc/unbind`. This only disables the empty SD controller.
- `reboot -f` does not unmount cleanly, so p3 replays its journal on the next boot. That is noisy and harmless. `sync` first anyway. `fsck.ext4` is not installed; the boot warning can be ignored.

`apk` repositories on the installed system should be the network ones:

```
https://dl-cdn.alpinelinux.org/alpine/v3.24/main
https://dl-cdn.alpinelinux.org/alpine/v3.24/community
```

Drop `/media/usb/apks` once the stick is no longer part of boot. The board's clock is wrong until you set it, and `apk` over HTTPS then fails with "server certificate not trusted". Set the clock from the host (`date -u -s @<epoch>`) before `apk update`. Install `chrony` and add `chronyd` to the default runlevel so the next boot fixes itself. `hwclock -w` is worth trying; do not count on the RTC surviving a power cut.

## Ethernet

Stock `fingbox.dtb` probes nothing on Linux 6.18. `dwmac-sun8i` looks up the IRQ by name and returns `-ENXIO` ("IRQ macirq not found") when `interrupt-names` is missing. After that is added it still fails with "Cannot get mdio-mux node", because this H3 driver wants an `mdio-mux` child even when the PHY is external.

Decompile `/boot/alpine/fingbox.dtb` with `dtc -I dtb -O dts`. On `ethernet@1c30000` (`allwinner,sun8i-h3-emac`):

- Add `interrupt-names = "macirq"`. Leave the existing `interrupts` value.
- Set `clock-names = "stmmaceth"`. A missing clock only warns; the IRQ and the mux are what actually block the probe.
- Keep `reset-names = "ahb"`. Do not rename it to `stmmaceth`.
- Keep `phy-mode = "rgmii"`. Do not switch to `rgmii-id`.
- Keep `reg` at `<0x1c30000 0x104>`. Do not widen it.
- Keep `allwinner,leds-active-low` and the existing `phy-handle`.

Add an empty `mdio` child (`snps,dwmac-mdio`) and an `mdio-mux` (`allwinner,sun8i-h3-mdio-mux`) whose `mdio-parent-bus` points at it. Under the mux:

- `mdio@1` (`allwinner,sun8i-h3-mdio-internal`) with `ethernet-phy@1` at `reg = <1>`, carrying the clock and reset cells that used to describe the internal PHY (`clocks = <&ccu 0x43>`, `resets = <&ccu 0x27>` in the stock phandle numbering). `get_ephy_nodes` requires that clock and reset.
- `mdio@2` with `ethernet-phy@0` at `reg = <0>`. Point the existing `phy-handle` at this node.

Give the new `mdio` a phandle that is not already used. Recompile with `dtc`, confirm the size grew (the working DTB was 20570 bytes against a 20062-byte original), and replace `/boot/alpine/fingbox.dtb`. Leave `fingbox.dtb.orig` in place.

After reboot, `eth0` exists and has the MAC from `ethaddr`. With no cable its state is `NO-CARRIER`. A DHCP stanza in `/etc/network/interfaces` is enough for the day a cable appears:

```
auto lo
iface lo inet loopback

auto eth0
iface eth0 inet dhcp
```

## Wi-Fi

Two separate bugs. Fix both. A power cut is part of the procedure, not a last resort.

### The chip stays in firmware across a reboot

`ath9k_htc` downloads `htc_9271-1.4.0.fw` (51008 bytes once decompressed). The AR9271 only returns to its boot ROM when USB VBUS drops. A reboot does not drop VBUS on this board, so the next probe transfers firmware and then logs `Target is unresponsive` after about a second. One cold boot after a real power removal logs `HTC initialized with 33 credits`.

There is no VBUS GPIO for `1c1d000`. `usb1-vbus` is PH6 and `usb2-vbus` is PH3; both are `disabled` in the DTB, and the main pin controller's `ngpio` is 224, so PH is not exportable. PH6 is the USB stick's port. Do not toggle it. Unbinding the EHCI controller drops the chip onto the companion OHCI, which rejects it (`Device endpoint numbers are not the expected ones`). A USB port reset (`USBDEVFS_RESET`) does not recover a wedged chip and can leave the port unable to enumerate. The fix is to unplug the board's own supply for about ten seconds.

Do this power cut only after the patched modules and `eeprom_ar9271.bin` are installed. A successful HTC init against the unpatched driver still fails, and the chip is wedged again until the next power cut.

### The calibration file is loaded and then ignored

The on-chip EEPROM reads as magic `0x0000`. Stock Domotz fed the driver `eeprom_ar9271.bin`. Mainline `ath9k_hw_nvram_read()` will use `ah->eeprom_blob` if something sets it. The HTC driver never sets it. Request the file in `ath9k_init_priv`, after the target is ready and immediately before `ath9k_hw_init(ah)`:

```c
#include <linux/firmware.h>

/* in ath9k_init_priv, just before ath9k_hw_init */
{
	const struct firmware *eeprom = NULL;

	if (!firmware_request_nowarn(&eeprom, "eeprom_ar9271.bin", priv->dev))
		ah->eeprom_blob = eeprom;
}
```

Release `ah->eeprom_blob` with `release_firmware` on the `err_hw` path and in `ath9k_deinit_priv`.

That patch is not sufficient. AR9271 uses the 4k EEPROM layout. `ath9k_hw_4k_fill_eeprom()` takes the USB path, `ath9k_hw_usb_gen_fill_eeprom()`, which reads the blank chip through MMIO and never looks at `eeprom_blob`. The magic check then passes (it goes through `nvram_read`, which does see the file) and the checksum runs on zeros: `Bad EEPROM checksum 0x0`. In `ath9k_hw_4k_fill_eeprom()`, if `ah->eeprom_blob` is set, call `__ath9k_hw_4k_fill_eeprom()` instead of the USB filler. `__ath9k_hw_4k_fill_eeprom()` reads through `nvram_read`, which prefers the blob, starting at word offset 64.

The stock file's checksum is already valid. The 4k header starts at file offset 128. Its `length` field counts the bytes covered by the checksum, and the XOR of those little-endian `u16`s is `0xffff`. Do not "fix" the checksum. Do not replace the file with a generic one. The MAC inside the file is not the board's Wi-Fi MAC; set the MAC on the interface after `wlan0` exists.

`eeprom_4k.c` is linked into `ath9k_hw.ko`, not `ath9k_htc.ko`. You must install both modules: `ath9k_htc.ko` for the firmware request, `ath9k_hw.ko` for the fill. Put `eeprom_ar9271.bin` at `/lib/firmware/eeprom_ar9271.bin`. Leave Alpine's compressed `htc_9271-1.4.0.fw.zst` as the firmware the loader finds. `dmesg` reports the uncompressed size, 51008.

### Building the modules on the host

Take Alpine's `linux-lts-dev` headers for the exact running version (`Module.symvers`, `include/generated`, `scripts/module.lds`) and the matching kernel source. Copy `scripts/module.lds` into the source tree or the `%.ko` rule is discarded. Build with the headers' `Module.symvers` so modversions CRCs match. Do not regenerate it.

```
gmake ARCH=arm CROSS_COMPILE=arm-none-eabi- \
  HOSTCC=clang HOSTCFLAGS="-I/tmp/host-include" \
  -j8 M=drivers/net/wireless/ath/ath9k modules
```

`HOSTCC` is only for host tools. The arm compiler must be GCC. If `CONFIG_CC_IS_CLANG` is set, the arm Makefile adds `-meabi gnu` and `arm-none-eabi-gcc` rejects it. Force `CONFIG_CC_IS_GCC=y` and set `CONFIG_CC_VERSION_TEXT` to the real `arm-none-eabi-gcc` version string in both `.config` and `include/config/auto.conf`, then `olddefconfig`. A mismatch makes `make` run an interactive `syncconfig`.

Turn `CONFIG_MODULE_SIG_ALL` off for this build. `CONFIG_MODULE_SIG_FORCE` is not set, so the unsigned modules load. The taint line `module verification failed: signature and/or required key missing` is expected. Disable `CONFIG_GCC_PLUGINS` if the host compiler cannot build them. `RANDSTRUCT` is off, so that does not change struct layouts.

Strip with `arm-none-eabi-strip -g`. `--strip-unneeded` removes the modversions symbols and the module will not load.

The vermagic string must be exactly `6.18.58-0-lts SMP mod_unload modversions ARMv7 p2v8` for this kernel. Confirm with `strings` before copying.

Install over the files under `/lib/modules/6.18.58-0-lts/kernel/drivers/net/wireless/ath/ath9k/`. Unload in order, because each holds the next: `rmmod ath9k_htc`, `rmmod ath9k_common`, `rmmod ath9k_hw`. Then `modprobe ath9k_htc`. `rmmod ath9k_hw` while `ath9k_htc` is loaded returns "Resource temporarily unavailable".

### Join the network

Install `wpa_supplicant` (Alpine 3.24 armv7 pulls `libnl3`, `dbus-libs`, `pcsc-lite-libs`, and the OpenSSL libs). Ask the owner for the SSID and passphrase. Write `/etc/wpa_supplicant/wpa_supplicant.conf` mode 600:

```
ctrl_interface=/run/wpa_supplicant
update_config=0
network={
	ssid="<ssid>"
	scan_ssid=1
	key_mgmt=WPA-PSK
	psk="<passphrase>"
}
```

`scan_ssid=1` matters when the SSID is hidden. It is harmless when it is not.

`/etc/local.d/wifi.start`, mode 755, run from the `local` service in the default runlevel. Wait up to ~25 s for `wlan0` (the driver is slow on a cold boot). Bring the link down, set the Wi-Fi MAC from `domotz_ath_hwaddr`, bring it up, start `wpa_supplicant -B`, wait until `wpa_cli -i wlan0 status` shows `wpa_state=COMPLETED`, then `udhcpc -i wlan0 -t 12 -T 2 -b`. Echo a line to `/dev/console` so a serial watcher can see success without logging the passphrase.

Then unplug the supply for ten seconds and plug it back in. On the way up you want, in order: `HTC initialized with 33 credits`, no EEPROM error, `wifi: wlan0 is up`, and an address on the owner's LAN. `udhcpc` failing earlier in the boot is `eth0` with no cable. Ignore it if `wlan0` has an address.

## SSH, then the stick can go

The stick is not in `fstab` and is not the root. Once `wlan0` has an address and you have confirmed the eMMC backup file is still on the stick, unmount it if you mounted it and tell the owner to pull it. They should keep the stick. It is the backup.

Install `openssh-server` and add `sshd` to the default runlevel. Install the owner's SSH public key in `/root/.ssh/authorized_keys` (directory 700, file 600). For a home bench box the owner may also want password login: set the password with `chpasswd`, and set `PermitRootLogin yes` and `PasswordAuthentication yes` in `/etc/ssh/sshd_config`. OpenSSH uses the first occurrence of each keyword. Restart `sshd` and log in from the host over Wi-Fi before you call it done. `PermitEmptyPasswords` stays off.

The DHCP address can change. 

## What "done" looks like

- Power applied, no serial cable required: the board boots Alpine from eMMC, brings up `wlan0`, and answers SSH.
- `eth0` exists and will DHCP if a cable is plugged in.
- p1 still contains the stock `boot.scr` and `uboot.env`. p2 is untouched. The first 256 MiB is untouched.
- The gzipped eMMC image from before the install still matches what `cmp` verified, and it lives somewhere other than the eMMC.

## Failures already chased, and the answer

| What you see | What it means |
| --- | --- |
| `IRQ macirq not found` | DTB has no `interrupt-names`. |
| `Cannot get mdio-mux node` | DTB has no `mdio-mux`. `eth0` will not exist until it does. |
| `Target is unresponsive` about 1 s after the firmware transfer | AR9271 firmware is still resident. Unplug the supply for 10 seconds. Do not keep rebinding USB. |
| `Invalid EEPROM magic (0x0000)` | HTC came up and the driver read the blank chip. The firmware-request patch is missing or the file is not in `/lib/firmware`. |
| `Bad EEPROM checksum 0x0` after `HTC initialized` | Magic came from the file, body came from the blank chip. `ath9k_hw.ko` is still the USB fill path. |
| `Device endpoint numbers are not the expected ones` | The chip fell onto OHCI. Rebind the EHCI at `1c1d000`. |
| `apk`: certificate not trusted | Clock is wrong. Set it, then retry. |
| `rmmod ath9k_hw`: resource temporarily unavailable | `ath9k_htc` or `ath9k_common` is still loaded. |
| ext4 journal replay on every boot | The previous `reboot -f` did not unmount. Boot continues. |
| SD controller `fatal err update clk timeout` | The empty slot. Unbind `1c0f000.mmc` only. |
