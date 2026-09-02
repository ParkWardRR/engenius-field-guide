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

> **The single biggest gotcha: on the EnSky/EWS build, SSH listens on port
> `8822`, not 22.** `etc/config/dropbear` has `option Port '8822'` (and
> `ssh_tunnel_port '8822'` in `ezmcloud`). Scanning port 22 shows "connection
> refused" and sends you down a rabbit hole thinking SSH is disabled — it isn't.

Telnet (23) lands in a **restricted CLI** that on newer builds is fully locked
("Please use 'Cloud GUI' to manage"). The way in is the **SSH exec channel on
:8822**, which runs as `uid=0`:

```sh
ssh -p 8822 -o HostKeyAlgorithms=+ssh-rsa -o PubkeyAcceptedAlgorithms=+ssh-rsa \
    root@<ap> 'id; cat /lib/upgrade/check_senao_image_header.sh'
```

SSH user is `root`, password = the **web admin** password (default `admin`). The
web LuCI login hashes `hex_md5(password + "\n")`; **the POST fields are
`username` + `password`** (not `luci_*`), and a valid login 302-redirects with a
`stok`. Protected POSTs first arm a token via `GET admin/system/ajax_setCsrf`.
The EWS web often force-redirects to **HTTPS** after the first apply.

**Cloud/FIT firmware has no shell** — but when *unadopted* it serves a local
React GUI with a JSON API (`admin`/`admin`), which is enough to point it at a
controller and to flash, with no shell at all:

```sh
TOK=$(curl -s -X POST http://<ap>/api/sys/login -H 'Content-Type: application/json' \
      -d '{"username":"admin","password":"admin"}' | jq -r .data.token)
curl -s http://<ap>/api/sys/sys_info -H "Authorization: Bearer $TOK"    # serial, mac, fw
curl -s -X POST http://<ap>/api/mgm/force_ac -H "Authorization: Bearer $TOK" \
     -H 'Content-Type: application/json' -d '{"ip":"<controller-ip>"}'  # set controller
curl -s -X POST http://<ap>/api/sys/apply -H "Authorization: Bearer $TOK" -d '{}'
# local flash: POST /cgi-bin/upload.cgi (multipart field `file`, Bearer) ->
#   GET /api/mgm/local_upgrade_image (validate) ->
#   POST /api/mgm/fw_upgrade {"mode":"Upgrade_locally"}   (bare {} = OTA no-op)
```
The GUI's diagnostic tools (`/api/mgm/tools/ping` …) strictly validate the target
as an IP, so there's **no command-injection shortcut** to a shell there.

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
Kaiwoo-authentication: id=88:DC:97:xx:xx:xx,timestamp=…,nonce=…,sn=00000000000000000000
```

**The `sn` is all zeros — a blank serial.** How the EPC check-in handler
(`routers/checkin.pyc :: handle_auth_request`, `pkg/general/chicken.pyc`) uses it:

1. **Match is by serial:** `device = Device.objects(serial_number=<sn>).first()`.
   `None` → `add_pending_device(mac, sn)` → HTTP `404` "Device Is Not Register".
   **A blank serial is silently skipped by `add_pending_device`** — so it never
   even shows up in the controller's "Pending Approval" list.
2. The **Redis operational key is `device/<mac_lower>`** (the serial is only the
   mongo match key). Reads `secret`, `falcon_nid`, `org_id`, etc.
3. The `Kaiwoo-signature` is HMAC-SHA256, verified by `valid_signature`
   (per-device `secret`) **OR `valid_signature_ex` (a global `DEFAULT_PRESHARED_KEY`)**.
   The `_ex`/default-key path is the new-device bootstrap: a device with no
   per-device secret still passes, so you don't need to know a per-device key.
4. Model is resolved from the serial's 3-char model code via the `models`
   collection (`{name: ECW230v3, number: X42}`).

Why blank: the cloud firmware reads its serial from **config field 19** —
`/etc/init.d/swallow`: `serial_number=$(setconfig -g 19)` — the "extra serial",
a **20-character** field that ships empty on a cross-flashed board. **This — not
the image — is the true adoption gate.**

### The fix: write field 19 with `fw_setenv` (the SAFE way)

Field 19 lives in the u-boot env (mtd7 "APPSBLENV"). `fw_setenv`, `fw_printenv`,
and `setconfig -g/-s` all operate on the same env and are mutually consistent —
and `fw_setenv` writes a proper CRC. Use it. The value must be **exactly 20
chars** with the target model code at positions 5–7 (`X42` = ECW230v3):

```sh
# on the EWS firmware, root via SSH :8822
fw_setenv snextra EPC1X420000000000000     # 20 chars, X42 at pos 5-7
setconfig -g 19                            # verify (reads the same env)
# the value survives a firmware flash (env is separate from rootfs), so re-flash
# the (product_id-patched) ECW230v3 image and it now presents EPC1X420000000000000.
```

> ⚠️ **Do NOT use `setconfig -s 19 …` casually, and never `setconfig -a 5`
> without a valid temp file.** `setconfig`'s set path is a temp-file workflow
> (`-a 1` copy flash→`/var/uboot_config`, `-a 2 -s N -d val` edit, then commit);
> **`setconfig -a 5` with no temp reads STDIN straight into `/dev/mtd7` and
> ERASES the whole u-boot env.** See §9 for why that (and the "recovery" after
> it) can stop the AP from booting.

Register that serial in the controller as an ECW230v3, and the check-in matches →
the AP adopts (validated via the `DEFAULT_PRESHARED_KEY` bootstrap). **The model
code in the serial matters** — give a cloud device an `X42` serial, not the
hardware's original `X44`, so the controller treats it as the model it's now
running. Same "forge a serial with the right model code" idea as
[serial-numbers](serial-numbers.md).

## 9. ⚠️ The u-boot env is a bricking hazard — the two-part trap

Two mistakes here turned a running AP into one stuck at the bootloader. Both are
easy to avoid once you know them.

**Part 1 — wiping the env.** `setconfig -a 5` with no temp file wrote empty STDIN
to `/dev/mtd7`, erasing every field (serial, product id, MACs). The device kept
running (values were already loaded), but a reboot now had no saved env.

**Part 2 — the fatal "fix".** Rebuilding the env with a **valid but incomplete**
set of vars (`fw_setenv bootcmd bootipq; active_fw 0; sn; snextra`) is *worse*
than leaving it erased. Rule:

- **Erased/invalid env → SAFE.** u-boot falls back to the **complete compiled
  default env** baked into the APPSBL (mtd8: `bootcmd=bootipq, active_fw=0,
  rootfsname=rootfs, ethaddr=00:aa:bb:cc:dd:10, snextra=00000000000000000000`).
  The board boots.
- **Valid-but-incomplete env → DANGEROUS.** u-boot now *trusts* your env and
  stops consulting the defaults, so any boot var you didn't set is missing → the
  active slot fails to boot.

So: only ever **`fw_setenv` individual fields on top of a known-good env**, or
**`env default -a`** to restore the whole default set. Never hand-assemble an env.

**MAC surprise.** Wiping the env also drops the stored MAC. EWS firmware reads the
MAC from ART and is unaffected, but **cloud firmware reads the MAC from env
`ethaddr`** — with it gone, cloud falls back to the raw Qualcomm default
(`00:03:7f:xx:xx:xx`) and pulls a *different* DHCP IP. An AP that "vanished" is
often just this: same device, new MAC, new lease. A managed switch's MAC table
(by port) + your DHCP leases will find it.

### Recovering a stuck AP (UART)

If the active slot won't boot, USB-TTL 3.3 V (GND↔GND, adapter RX↔AP TX, adapter
TX↔AP RX, **no VCC** — it's PoE-powered), 115200 8N1. Interrupt the boot
countdown to reach the u-boot prompt, then:

```
env default -a        # restore the COMPLETE default env -> the slot boots again
saveenv
reset
```

Then set your real identity the safe way (single fields), verify `fw_printenv`
shows a complete env (`bootcmd=bootipq`) *before* rebooting, and re-flash as
needed. A plain power-cycle does **not** help (same bad saved env); the
dual-image boot-count failsafe also won't trigger if the env can't persist the
counter. UART is the reliable path back.

## 10. Reverting

Flash the stock `EWS377APv3-*.bin` the same way (`product_id` already `282`, no
patch), or hold the factory-reset button to fall back to the intact EWS A/B slot.
