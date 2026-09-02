# Cross-flashing

Running one model's firmware on a hardware-equivalent sibling — e.g. to move an
ezMaster-era AP onto a Fit/ECW line the modern controller supports.

> ⚠️ Advanced and destructive. A bad bootloader write bricks the device until
> recovered over UART. Do not attempt without serial access and a way back.

## When it works

Only between models that share hardware — see
[model-equivalence](model-equivalence.md). Even then, matching the OS
image isn't always enough (see the `cert` partition below).

## Levels of difficulty

1. **Firmware-only.** If the target accepts the donor image (its
   [Senao header](firmware-format.md) `product_id` matches, or you force the
   check), just flash it. Often works for the OS but may leave controller/cloud
   features broken.
2. **Missing per-device secret.** ECW/cloud devices store an auth cert+key in a
   dedicated **64 KB `cert` MTD partition**. Models like the ENS620EXT don't have
   one, so a firmware-only crossflash can't register with the controller.
3. **Bootloader swap.** MTD partition layout is hard-coded in the per-model u-Boot
   build (flash has no on-flash partition table). To change the layout you flash
   the donor's u-Boot **and** its env — the point of no return.

## Getting a real root shell

Stock SSH on ECW devices forces a restricted CLI. Run your own Dropbear on a
nonstandard port for a root shell + SCP:

```bash
dropbear -p 2022
```

Modern `scp` needs legacy flags against these devices:

```bash
scp -O -P 2022 -o HostKeyAlgorithms=+ssh-rsa -o KexAlgorithms=+diffie-hellman-group14-sha1 \
    admin@<device-ip>:/tmp/file ./file
```

## Reading/writing flash

```bash
cat /proc/mtd            # list partitions
cat /dev/mtd5 > /tmp/mtd5.bin
mtd write /tmp/mtd5.bin /dev/mtd5
```

## Example: ENS620EXT vs ECW160 MTD layout

The ECW160 carries `failsafe`, `failsafe_conf`, and **`cert`** partitions the
ENS620EXT lacks — the crux of why a plain crossflash won't register with the
controller.

| ENS620EXT | ECW160 |
|-----------|--------|
| SBL1, MIBIB, QSEE, CDT, DDRPARAMS, APPSBLENV, APPSBL, ART, HLOS, rootfs, rootfs_data, u-boot-env, userconfig | SBL1, MIBIB, QSEE, CDT, DDRPARAMS, APPSBLENV, APPSBL, ART, HLOS, rootfs, rootfs_data, **failsafe**, **failsafe_conf**, **cert** |

u-Boot lives at `APPSBL` (mtd6); its env at `APPSBLENV` (mtd5) — you need both
when swapping bootloaders. Example version strings:

```
ENS620EXT: U-Boot 2012.07-ENS620EXT-uboot_version:V1.0.4 (Mar 05 2021)
ECW160:    U-Boot 2012.07-ECW160-uboot_version:V1.0.2 (Jul 09 2019)
```

## Reverting: EWS377-FIT → ECW230 (back to stock)

Hardware v3 (FCC ID `A8J-EWS377APV3A`):

1. Disconnect from the network. Power via 12V; wire the AP to a PC NIC set to
   `192.168.1.22/24`, gateway `192.168.1.1`.
2. Factory-reset with the button.
3. Browse `http://192.168.1.1/`, log in `admin` / `admin`, set new credentials
   when prompted.

## Full write-ups

The detailed ENS620EXT↔ECW160 conversion, boot logs, and firmware analysis are in
DaveCorder's repo — see [CREDITS](../../CREDITS.md).
