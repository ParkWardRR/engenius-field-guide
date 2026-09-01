# Cross-flashing an EWS377AP v3 to Cloud/FIT — a full walkthrough

An evidence-based, end-to-end account of converting an EnGenius **EWS377AP v3**
(the ezMaster/EnSky "neutron" AP) to **EWS377-FIT** or **ECW230v3** firmware so a
modern controller can manage it. Every claim here was verified on real hardware
and stock firmware images. For the terse reference version see
[cross-flashing](cross-flashing.md); this is the "show your work" version.

**TL;DR:** it's a *one-field* patch. The three are the same board (IPQ807x /
`ap-hk07`); flip the firmware image's `product_id` and flash it. The flash is the
easy part — *adoption* is where per-device identity bites (see the last section).

## 0. The three siblings

| Firmware model | `product_id` | Model code | Series | Managed by |
|---|---|---|---|---|
| EWS377AP v3 | `282` (`0x11a`) | `X44` | neutron | ezMaster / EnSky |
| EWS377-FIT | `300` (`0x12c`) | `X45` | fit | FitController / Cloud |
| ECW230v3 | `284` (`0x11c`) | `X42` | cloud | EnGenius Cloud / EPC |

Same silicon (`board_name = ap-hk07`), `vendor_id = 257` across the line. Which
firmware you flash decides which controller world the AP lives in — pick the one
your controller speaks (an EPC/EnGenius-Cloud wants **cloud** ECW230v3 or **fit**;
a FitController wants **fit**).

## 1. The image format (and why `binwalk` fails)

EnGenius images are Senao format: a **96-byte main header** + a **CAPWAP
sub-header** (present when `firmware_type == 0`) + an **obfuscated payload**.
Big-endian. Verified offsets:

| Off | Field | Notes |
|--|--|--|
| `0x08` | `product_id` | the one field you patch |
| `0x1C` | `firmware_type` | `0` = combo (has CAPWAP sub-header) |
| `0x28` | `md5sum` | over the **payload** (pre-header), not recomputable via `mksenaofw` |
| `0x58` | `chksum` | vendor-proprietary; **not read** by the upgrade path |
| `0x5C` | `magic` | `0x12345678` |
| `0x88` | `model` string | `EWS377APv3` / `EWS377-FIT` / `ECW230v3` |

The payload begins ~offset 146 and reads as noise (`a831 6022…`) because it's
obfuscated — `binwalk` finds nothing. `mksenaofw -d` (OpenWRT `firmware-utils`)
de-obfuscates it back to a plain U-Boot FIT (`d00dfeed`).

## 2. What the AP actually checks on upload

Pulled from the running rootfs, `/lib/upgrade/check_senao_image_header.sh`:

```sh
check_senao_image_header(){
	local SN_VENDOR_ID=0101
	local SN_PRODUCT_ID=011a          # 282 = EWS377AP v3
	[ "$senao_vendor_id" = "$SN_VENDOR_ID" -a "$senao_product_id" = "$SN_PRODUCT_ID" ] && return 0
	[ "$senao_magic_long" = "27051956" -o "$senao_magic_long" = "d00dfeed" ] && return 0
	return 1
}
```

It gates on **`vendor_id` + `product_id` only** — not the model string, not
`md5sum`, not `chksum`. So a foreign image is rejected (web UI shows *"Firmware
upgrade is failed … make sure your firmware is valid"*) purely because its
`product_id` is `300`/`284`, not `282`.

## 3. The one-field re-head

Patch the stock FIT/cloud `.bin`'s `product_id` to `282`:

```python
import struct
d = bytearray(open("Cloud6_4x4_ECW230v3_firmware_v1.8.112-13.bin","rb").read())
assert struct.unpack(">I", d[8:12])[0] == 284     # ECW230v3
d[8:12] = (282).to_bytes(4, "big")                 # -> EWS377AP v3 gate
open("ecw230v3-282.bin","wb").write(d)
```

Nothing else needs fixing: `md5sum` covers the untouched payload (stays valid —
`/bin/header -x` on the patched image still prints `MD5 check OK!`), and `chksum`
is never read. The obfuscation key is **universal**, so a FIT/cloud payload
de-wraps fine on v3 hardware.

## 4. Getting in to do it

Telnet (23) and SSH (22, once enabled) both land in a **restricted CLI**
(`ews377apv3>`). Escape it with the **SSH exec channel**, which runs as `uid=0`
via `/bin/ash`:

```sh
ssh -o HostKeyAlgorithms=+ssh-rsa -o PubkeyAcceptedAlgorithms=+ssh-rsa \
    root@<ap> 'id; cat /lib/upgrade/check_senao_image_header.sh'
```

SSH user is `root`, password = the **web admin** password. The web LuCI login
hashes `hex_md5(password + "\n")`; protected POSTs first arm a token via
`GET admin/system/ajax_setCsrf`.

## 5. Flashing

**Web UI (two-step):** upload the patched `.bin` to `System Manager → Firmware`
(`POST admin/system/flashops`, field `image`, disable `Expect: 100-continue`);
it validates, then you confirm with a second POST (`step=2`, `keep=0` for a clean
cross-firmware flash). **Or** just `sysupgrade -n /tmp/patched.bin` from the exec
shell.

The Senao dual-image write goes to the **inactive** rootfs slot and flips the
boot pointer, so the reboot comes up on the new firmware.

## 6. What you'll see afterward

- It boots the new firmware. Old EWS credentials stop working; on cloud/FIT the
  web root returns **`403`/`404`** (no local GUI — controller-managed). That's the
  confirmation it took.
- **Dual-image = free rollback, temporarily.** Stock EWS377APv3 is still on the
  other slot. A **factory-reset button hold reverts to it** (handy way back in —
  it comes up with `admin`/`admin` LuCI). The slot is reclaimed by the first
  firmware update, so keep the stock `.bin` off-box.

## 7. Reading firmware off-box (creds, discovery logic)

```sh
mksenaofw -d fw.bin -o decoded.bin                       # de-obfuscate -> FIT
dumpimage -T flat_dt -p 1 -o rootfs.ubi decoded.bin      # extract rootfs UBI
ubireader_extract_images rootfs.ubi && unsquashfs <vol-ubi_rootfs>
```

Findings from the rootfs: `admin`/`admin` (MD5-crypt `$1$jLKJRxCv$…`), `root`
empty (but SSH off + empty-login blocked), no `/www` GUI, and the cloud agent
logic in `/lib/cloud/minicloud-init.sh`.

## 8. Adoption is the real work — the blank-serial trap (and the fix)

Flashing is 90%; adoption is the other 90%. A cloud AP:

1. **Discovers** the controller via mDNS (EPC `Minicloud_*._http._tcp` TXT →
   `/api/v1/checkin` + `/device/register`), via **DHCP option 43** (controller
   IP), or `force_ac`.
2. **Checks in** to `/api/v1/checkin` (server-auth TLS, `curl -k`), identifying
   itself in a header.

Capture one check-in (the AP connects with `-k`, so a self-signed TLS logger with
an iptables redirect of just the AP works) and you see the exact problem:

```
POST /api/v1/checkin HTTP/1.1
Kaiwoo-authentication: id=88:DC:97:04:44:07,timestamp=…,nonce=…,sn=00000000000000000000
```

**The `sn` is all zeros — a blank serial.** The EPC keys every device by serial:
the check-in handler runs a Redis `HMGET device/<sn> secret`; no record →
`DEVICE_NOT_FOUND`, surfaced as `handle_auth_request … checkin key error: 'id'`
and a permanent `404` loop. `devices` never leaves `0`.

Why blank: the cloud firmware reads its serial from **config field 19** —
`/etc/init.d/swallow`: `serial_number=$(setconfig -g 19)`. On a cross-flashed
board that field is empty (the EnSky/EWS firmware read the real serial fine; the
ECW firmware reads a different location and gets zeros). **This — not the image —
is the true adoption gate.**

### The fix: give it a serial with `setconfig`

`setconfig` writes the persistent factory config, which survives a firmware
flash. So, while you have EWS access (from the factory-reset revert):

```sh
# generate a VALID serial with the target model code so the controller accepts
# it as that model — X42 = ECW230v3 (Code27 check char; see serial-numbers.md)
#   e.g. EPC1X4200011

# on the EWS firmware (root via the SSH exec channel):
setconfig -s 19 EPC1X4200011      # write serial to field 19
setconfig -g 19                   # verify

# then re-flash the (product_id-patched) ECW230v3 image; it now reads field 19
# and presents EPC1X4200011 instead of zeros.
```

Register `EPC1X4200011` in the controller as an ECW230v3, and the check-in now
matches → the AP adopts. **The model code in the serial matters**: give a cloud
device an `X42` serial, not the hardware's original `X44` (EWS377AP v3 / neutron),
so the controller treats it as the cloud model it's now running. This is the same
"forge a serial with the right model code" idea as
[serial-numbers](serial-numbers.md), applied to close the identity gap a
firmware-only crossflash leaves open.

## 9. Reverting

Flash the stock `EWS377APv3-*.bin` the same way (`product_id` already `282`, no
patch), or hold the factory-reset button to fall back to the intact EWS A/B slot.
