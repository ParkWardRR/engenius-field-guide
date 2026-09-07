# OpenWrt on the EWS377AP v3 (and `ap-hk07` siblings)

There's a third option beyond self-hosting the controller or cross-flashing to a
sibling OEM image: **replace the firmware with real [OpenWrt](https://openwrt.org/).**
A community port for the EnGenius **EWS377AP v3** (`ap-hk07`, Qualcomm IPQ8072A) now
boots and runs **persistently from NAND on real hardware** — kernel 6.18 with
Qualcomm NSS hardware offload.

> ## → Where to get it
> - **Firmware images + `SHA256SUMS`:**
>   [pelegrun-ap-hk07-firmware-tools · Releases](https://github.com/ParkWardRR/pelegrun-ap-hk07-firmware-tools/releases)
>   (tag `openwrt-ews377ap-v3-v0.1`)
> - **Install + back-to-stock guide:**
>   [`docs/openwrt-ews377ap-v3.md`](https://github.com/ParkWardRR/pelegrun-ap-hk07-firmware-tools/blob/main/docs/openwrt-ews377ap-v3.md)
> - **Source / engineering write-up:** fork branch `ews377ap-v3` of
>   [`openwrt-nss-edma`](https://github.com/ParkWardRR/openwrt-nss-edma) (see
>   `PORT-STATUS-ews377ap-v3.md`)

This page is a map, not the manual — follow the guide above for the actual steps.

## Status

Validated end-to-end on one unit: persistent NAND boot, Ethernet, both WiFi radios
(WPA2), and configuration surviving reboots. Secure boot is **not fused** on the
tested unit, so custom images boot. This is **unofficial** and **prerelease** — only
the UART/u-boot install is hardware-proven; the web-upload `.bin` is experimental.

## Why this is worth knowing

EnGenius devices already run a *private fork* of OpenWRT (see [overview](overview.md)),
but it's locked to the vendor's controller/cloud model. Mainline-style OpenWrt gives
you the normal OpenWrt userland, LuCI, and package feeds on the same hardware — no
controller, no cloud, no adoption/serial dance (the [blank-serial trap](crossflash-ews377apv3-walkthrough.md)
that gates cross-flashing simply doesn't apply).

## The two things that made it boot (for porters)

The generic OpenWrt `qualcommax`/`ipq807x` target needed exactly two board-specific
adjustments to boot under the stock EnGenius u-boot:

1. **FIT config name `config@hk07`.** The OEM `bootipq` selects the FIT
   configuration by *board name* and aborts ("Config not availabale") on anything
   else. The device recipe sets `DEVICE_DTS_CONFIG := config@hk07` (same as the
   in-tree ap-hk07 sibling `netgear_wax218`).
2. **Install to slot 0.** OpenWrt's qualcommax root-mount always targets the
   SMEM/DTS partition labeled `rootfs` (slot 0, `0x1000000`) regardless of which A/B
   slot u-boot loaded the kernel from — so OpenWrt must live on slot 0 (also the only
   slot the OEM installer ever writes).

Everything else (DTS, the 2.5G QCA8081 uplink on `port@6`/`uniphy2`, ath11k WiFi via
ART caldata, LEDs on GPIO 54/55/56) is standard qualcommax work.

## Install / restore, in one paragraph

Proven path: UART on header **J2** (3.3V, 115200 8N1) → interrupt u-boot → **back up
your own NAND first** → TFTP the bare `factory.ubi` and `nand write` it to slot 0
(`0x1000000`) → `setenv active_fw 0; saveenv; reset`. To go back to stock, `nand
write` your own OEM `rootfs` backup to slot 0, or re-apply an official EnGenius image
via the updater once stock boots. **Never touch the ART partition / bootloader region
(`0x0`–`0x1000000`)** — that's the one true brick. Full commands, the experimental
web-upload method, and the partition map are in the
[install & restore guide](https://github.com/ParkWardRR/pelegrun-ap-hk07-firmware-tools/blob/main/docs/openwrt-ews377ap-v3.md).

## Related in this guide

- [firmware-format](firmware-format.md) — the Senao image header (the web-upload
  OpenWrt image reuses it: vendor `0x0101`, product `0x011a`, type 0 combo).
- [cross-flashing](cross-flashing.md) / [EWS377AP v3 walkthrough](crossflash-ews377apv3-walkthrough.md)
  — the *OEM-to-OEM* alternative if you want to stay on EnGenius firmware.
- [model-equivalence](model-equivalence.md) — the `ap-hk07` hardware family (ECW230v3,
  EWS377-FIT) the port shares silicon with.
