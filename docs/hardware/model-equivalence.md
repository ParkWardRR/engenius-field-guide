# Hardware equivalence

Models that share the same board can run each other's firmware — the basis for
[cross-flashing](../firmware/cross-flashing.md). Same board ≠ identical: enclosure,
ports, PoE, and per-device partitions can differ.

## Known equivalent groups

### ENS620EXT ≈ ENH1350EXT ≈ ECW160

All "11ac Wave 2 MU-MIMO Dual-Band AC1300 outdoor AP" (867 Mbps 5 GHz / 400 Mbps
2.4 GHz), same CPU, WiFi chips, RAM, and flash. ENH1350EXT and ECW160 share an
FCC ID.

| | ENS620EXT | ENH1350EXT / ECW160 |
|--|-----------|---------------------|
| Ethernet ports | 2 | 1 |
| PoE | proprietary 24V | standard 802.3at |
| Rating | IP55 | IP67 |
| `cert` MTD partition | no | yes (ECW160) |

Practical use: an ENS620EXT can be moved toward the ECW160 (a Fit-Controller line)
— but the missing `cert` partition means firmware alone won't register it; see
[cross-flashing](../firmware/cross-flashing.md).

### ECW230 (v3) ≈ EWS377-FIT ≈ EWS377AP v3

The ECW230v3, EWS377-FIT, and the ezMaster/EnSky-managed **EWS377AP v3** are the
same board — **IPQ807x / `ap-hk07`**, Wi-Fi 6 4×4 — so they are cross-flashable
siblings. Running EWS377-FIT firmware on ECW230 or EWS377AP v3 hardware moves it
onto a local Fit Controller. Firmware product IDs: EWS377-FIT `300`, ECW230v3
`284`, EWS377AP v3 `282` ([firmware-format](../firmware/firmware-format.md)).
Confirmed on a v3 by root shell: `board_name = ap-hk07`, and a de-wrapped
EWS377-FIT image is an `openwrt-ipq-ipq807x-ubi-root.img` — same target.

**EWS377AP v3 → FIT is a one-field re-head.** The v3's LuCI
`admin/system/flashops` rejects a stock FIT `.bin` at upload (rejected inline as
*"Firmware upgrade is failed … make sure your firmware is valid"*), but its check
gates on `product_id` **only** — not the model string, md5, or chksum. Patching
`product_id` `300 → 282` (two bytes at `0x08`) is sufficient:

- verified: the device's own `senao_image_header_check` returns `0` on the
  patched image, and `/bin/header -x` de-wraps it (`MD5 check OK!`) into a valid
  same-board FIT — so `platform_check_image` passes and it flashes as a normal
  sysupgrade. No vendor `ews377apv3-5.1.4-1.zip`, forced flag, or bootloader
  work needed.
- the v3 **has** the per-device `cert` MTD partition (`mtd9`), which a rootfs
  sysupgrade preserves — so the converted AP keeps the cert it needs to register
  with a controller (unlike the cert-less ENS620EXT case above).

**Confirmed end to end.** Flashing a `product_id`-patched `ews377-fit-1.0.4-3`
via `sysupgrade -n` on an EWS377AP v3 succeeded: `header -x` de-wrapped it, the
FIT (`d00dfeed`) passed `platform_check_image`, and it wrote the UBI rootfs and
rebooted into EWS377-FIT (old EWS credentials/GUI gone, FIT firmware serving).
Two practical notes:

- **Dual-image (A/B) preserves the old firmware, temporarily.** The Senao
  upgrade writes to the *inactive* rootfs slot (`rootfs_1`) and switches the boot
  pointer, leaving stock EWS377APv3 intact on the other slot (`rootfs`) as an
  automatic fallback — until the first FIT firmware update reclaims that slot for
  its own A/B cycle. Keep a copy of the stock `.bin` off-box for a durable
  rollback; don't rely on the slot surviving an update.
- **A FIT AP serves no local web GUI.** After conversion `https://<ap>/` returns
  a bare `404` (lighttpd is up, but there's no LuCI/admin page — FIT firmware is
  controller-managed). That 404 is itself a quick confirmation the cross-flash
  took: EnSky/EWS firmware serves a login at `/`, FIT does not.
- **Controller series matters.** EWS377-FIT is series `fit`; an EnGenius Private
  Cloud (EPC) manages the `cloud`/`cloud-lite`/`fit`/`neutron` series and lists
  EWS377-FIT (`X45`) as a supported model, so a converted AP is adoptable there.

## Adding to this list

Equivalence is usually inferred from a shared FCC ID plus matching CPU/WiFi/RAM/
flash on a root shell. Verify before flashing — a wrong guess bricks hardware.
