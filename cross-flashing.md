# Cross-flashing

Running one model's firmware on a hardware-equivalent sibling — e.g. to move an
ezMaster-era AP onto a Fit/ECW line the modern controller supports.

> ⚠️ Advanced and destructive. A bad bootloader write bricks the device until
> recovered over UART. Do not attempt without serial access and a way back.

## When it works

Only between models that share hardware — see
[model-equivalence](model-equivalence.md). Even then, matching the OS
image isn't always enough (see the `cert` partition below).

**"Shares hardware" means the board, not the SoC.** The header gate is the only
thing a re-head satisfies; the payload still describes the donor's board. Two
models on the same Qualcomm SoC can differ in ethernet port mapping, PHYs and
PCIe radios, and an image carrying the wrong set of those will often **boot
normally and have no network** — a failure that looks like a cabling problem and
sends you hunting in the wrong place. The ECW336 is the documented case: same
IPQ8072A as the EWS377-FIT, same `machid`, same DT `model` string, same FIT
config slot name, and still a different board
([why](model-equivalence.md#ecw336--same-soc-not-the-same-board)). Before
flashing, confirm the two DTBs' `compatible` strings match.

The other half of "shares hardware" is whether there is a **way back**. Check
before you start, because it varies inside one generation: the ap-hk07 units
offered no network recovery at all (a reset-button hold at power-on triggered
neither TFTP nor a web recovery, and the AP emitted zero packets at the u-boot
console — UART was mandatory), while the ECW336's u-boot has a working TFTP
recovery path on boot-menu option `2`. Never assume the sibling's recovery story
transfers.

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

## Identity is a separate axis from the image

Flashing the sibling's firmware does not make the controller believe the unit is
the sibling. Those are two independent changes, and a migration usually needs
both:

- **The image header gates the upload.** `product_id` (EWS377AP v3 `282`,
  ECW230v3 `284`, EWS377-FIT `300`) is what the upgrade path checks — re-head
  for that, within the board family only.
- **The serial decides the model.** The controller reads the 3-char code at
  serial positions 5–7; EPC maps it with `Code27Model` when registering
  inventory, with no model field sent. Swapping those three characters (and
  recomputing the check char) is the whole of the identity change — see
  [migration serials](serial-numbers.md#migration-serials).

Three consequences that cost time if you meet them cold:

1. **The pre-shared key moves with the serial.** On the ECW line it is
   `lower(mac) + sn + "0@ne$"`, so a re-stamped serial means a new key and a
   controller record seeded with the new value
   ([ecw-private-cloud-mode](ecw-private-cloud-mode.md)).
2. **`cert` still holds the original identity** and stock firmware can write it
   back over the env — only when the env MAC is the dummy `00:aa:bb:cc:dd:10` or
   the serial starts `00000000`, but that is enough to make a re-stamp evaporate
   after an env wipe. The both-valid case never compares the two, which is why a
   re-stamp otherwise sticks.
3. **The target code has to exist in the controller's table.** `X42`, `X44` and
   `X45` all do; there is no `ECW230-FIT` code at all, so an ECW230v3 keeps
   `X42` even after a FIT flash. For anything absent, see
   [add-unknown-models](add-unknown-models.md).

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

## A cross-flashed unit loses the OEM local-upgrade path permanently

Plan for this before you cross-flash, because it decides whether you ever need
UART again. Once a unit runs another model's firmware, the GUI's
"upgrade from local file" stops working — and it fails in a way that looks like
your image is at fault when it is not.

Measured on an EWS377-FIT running ECW230v3 1.8.114-1 (2026-11-23), authenticated
as the controller-pushed local account:

```
POST /cgi-bin/upload.cgi
-> 400 {"error_code":3,"error_message":"INVALID VALUE OF ARGUMENTS:firmware"}
```

The same response came back for **all** of: the correct ECW230v3 image under
form field `file`, under `firmware`, under twelve other field names, a 2 KB junk
file, and **no file part at all**. Auth is genuinely enforced (401 without a
token), so the endpoint is reachable — it rejects before reading any content.
The likely cause is that the handler validates the request against the board's
own identity, which still reports the original model (`hw_id` here was
`0101012B`, an EWS377-FIT value, while the firmware was ECW230v3).

Consequences worth designing around:

- **Do not count on the GUI, or on a controller push, to move a cross-flashed
  unit between firmware versions.** Keep a console harness with the unit.
- `u-boot` TFTP + `nand write` at the offsets from the target build's own
  `flash.scr` remains the route that works, and it is unaffected by the
  identity mismatch.
- A controller-side firmware push is a separate question and was *not* available
  either on the self-hosted EPC tested — its firmware-upgrade policy reported
  `is_connect_license_expired: true` / `is_active: false`. Don't assume the
  controller is the fallback.

Corollary: **the thing that makes a cross-flash "finished" is not that it boots,
it is that you can still update it.** Flash both A/B slots with the version you
actually want while you have the console attached, rather than leaving one slot
on an older build you can no longer reach.

## Full write-ups

The detailed ENS620EXT↔ECW160 conversion, boot logs, and firmware analysis are in
DaveCorder's repo — see [CREDITS](../../CREDITS.md).
