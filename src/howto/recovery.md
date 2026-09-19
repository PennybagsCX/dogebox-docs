# Recovery Mode

Recovery Mode is a minimal environment that lets you repair a Dogebox
whose main installation has stopped working — for example after a power
loss during an update, a bad system configuration, or a failed disk
upgrade.

# Getting into Recovery Mode

1. Power the device off. On the NanoPC-T6, plugging power in only lights
   the standby LEDs — press the power button once to boot, and note that
   a hung system will not respond to a short press (hold ~5 seconds to
   force-cut power as a last resort).
2. Insert a bootable Dogebox OS SD card — on the NanoPC-T6 the bootloader
   only boots microSD cards **32GB or smaller (SDHC, not SDXC)**; see
   [NanoPC T6](./installation/t6.md). Or, on devices with an
   internally-installed recovery image, hold the boot-media selection for
   your board.
3. Power the device on. The recovery environment boots from the SD card
   (or recovery partition) instead of the main installation.
4. The recovery UI is available in a browser at the address shown on the
   console (typically port `8080`), and the recovery API listens on port
   `3000`.

> **Note:** on the NanoPC-T6 the internal eMMC install boots via
> extlinux/U-Boot with a **very short (1-second) boot menu timeout**. If
> you need to select a different generation at boot time, watch the
> screen from the moment of power-on and press an arrow key as soon as
> the menu appears — or make the timeout permanent as described below.

# Repairing a broken boot default

If a system update left the device booting a broken configuration, the
fix is usually to boot an older (known-good) generation once, then make
it permanent again.

## Option A: from the boot menu

If your build's boot menu timeout is long enough (see the note above),
press an arrow key during the countdown, highlight a generation dated
before the problem, and press Enter. After booting, log in on the
console and make the selection permanent:

```console
$ sudo /run/booted-system/bin/switch-to-configuration boot
```

## Option B: from a bootable SD card

If the boot menu cannot be selected, boot from a Dogebox OS SD card and
log in on the console, then mount the internal installation and edit its
boot configuration:

```console
# Find the internal root partition (look for the ext4 "nixos" label,
# taking care not to confuse it with the SD card's own root partition):
$ lsblk -f

# Mount it and check the boot menu configuration:
$ sudo mount /dev/mmcblk0p8 /mnt
$ sudo vi /mnt/boot/extlinux/extlinux.conf
```

Two things worth changing in `extlinux.conf`:

- `TIMEOUT` is counted in tenths of a second (`TIMEOUT 10` = 1 second).
  A value of `100` gives a comfortable 10-second window to select an
  entry.
- `DEFAULT` names the entry that boots automatically. Generation labels
  look like `nixos-<number>-default`; the number is the generation index,
  and older indexes are older (pre-update) configurations.

Then unmount and reboot:

```console
$ cd /
$ sudo umount /mnt
$ sudo reboot
```

After booting the good generation, log in and run
`sudo /run/booted-system/bin/switch-to-configuration boot` so the good
generation remains the default across future boots.

# Recovery API

The recovery environment exposes a small API on port `3000`
(authenticate with your dashboard password to obtain a token):

- `GET /system/recovery-bootstrap` — reports whether the boot media is
  read-only, and the installation state (`notInstalled`,
  `unconfigured` or `configured`)
- `GET /system/disks` — block devices and their suitability for
  install/storage
- `GET/PUT /system/ssh/state`, `GET/PUT /system/ssh/key` — SSH
  configuration for the *installed* system
- `POST /system/host/reboot`, `POST /system/host/shutdown`

> **Caution:** `POST /system/install` performs a **full reinstall of the
> boot device**. Data on a separate storage disk survives a
> boot-device reinstall, but always verify which disk you are targeting
> (compare with `GET /system/disks`) before invoking it.

# Keeping recovery easy

- Keep a bootable Dogebox OS SD card for your board as part of your
  recovery kit.
- Raise the boot menu timeout on installed systems (a NixOS
  `boot.loader.timeout` setting of `10` in your custom configuration
  makes the menu reliably selectable).
- Avoid uninstalling or force-removing pups while the system is in a
  broken state — repair the boot first, then manage pups from the
  dashboard.
