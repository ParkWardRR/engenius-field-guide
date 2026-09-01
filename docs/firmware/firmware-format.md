# Senao firmware image format

EnGenius firmware is Senao-format: a header wraps the payload, and the device
validates that header on upload. Understanding it is the basis of
[cross-flashing](cross-flashing.md).

## Decode with mksenaofw

`mksenaofw` (from the OpenWRT `firmware-utils` source) decodes/encodes images:

```bash
mksenaofw -d input.bin -o decoded.bin
```

## Header fields

The image is a **96-byte (`0x60`) main header**, an optional **CAPWAP
sub-header** (present when `firmware_type == 0`, the "combo" type used by
EWS/ECW APs), then the payload. Offsets below are verified against stock
`EWS377APv3-v3.9.3.2` and `ews377-fit-1.0.4-3` images; all multi-byte fields are
**big-endian**.

### Main header (`0x00`–`0x5F`)

| Offset | Field | Size | Notes |
|--------|-------|------|-------|
| `0x00` | `head` | 4 | `0` |
| `0x04` | `vendor_id` | 4 | `257` for EnGenius/Senao |
| `0x08` | `product_id` | 4 | per model — `282` = EWS377AP v3, `300` = EWS377-FIT, `284` = ECW230v3 |
| `0x0C` | `version` | 16 | ASCII, NUL-padded (e.g. `3.9.3.2`) |
| `0x1C` | `firmware_type` | 4 | `0` = combo (has CAPWAP sub-header); `0xFF` = none |
| `0x20` | `filesize` | 4 | payload size = total − header length |
| `0x24` | `zero` | 4 | `0` |
| `0x28` | `md5sum` | 16 | integrity hash of the payload (see caveat below) |
| `0x38` | `pad` | 32 | zero |
| `0x58` | `chksum` | 4 | header checksum (see caveat below) |
| `0x5C` | `magic` | 4 | `0x12345678` |

### CAPWAP sub-header (`0x60`…, when `firmware_type == 0`)

Mirrors OpenWRT's `struct capwap_header`. For EWS377/ECW-class images it is 50
bytes (`model_size = 10`), so the **payload starts at offset 146 (`0x92`)**.

| Offset | Field | Size | Example |
|--------|-------|------|---------|
| `0x60` | `mod` | 4 | `"all"` |
| `0x64` | `sku` | 4 | |
| `0x68` | `firmware_ver[3]` | 12 | `3.9.3` |
| `0x74` | `datecode` | 4 | `211203` |
| `0x78` | `capwap_ver[3]` | 12 | `1.9.51` |
| `0x84` | `model_size` | 4 | `10` |
| `0x88` | `model` | `model_size` | `EWS377APv3` / `EWS377-FIT` |

The `product_id` (`0x08`) and the `model` string (`0x88`) are the two fields the
device's upload check keys on.

## On-device validation

The upgrade scripts (`/lib/upgrade/*.sh`) gate a flash on the header. Relevant
functions: `get_magic_long`, `senao_get_vender_id`, `senao_get_product_id`,
`senao_image_header_check`. An upload is accepted only if:

- the **magic long** matches,
- **vendor_id** and **product_id** match what the device expects, and
- an **md5** integrity check passes.

A mismatch is rejected and logged as `ap_fw_upgrade_failed reason='Model
mismatch'`. The upgrade path also has a `--force` branch that bypasses the model
check.

**Why this matters for cross-flashing:** to run a foreign image, the target must
accept its header — i.e. the image's `product_id` (and friends) must match, or
the check must be forced/patched. That header gate, plus per-device secrets like
the [`cert` partition](cross-flashing.md), is what makes a crossflash more than
just writing a `.bin`.

## Re-heading a donor image (what's actually checked)

What gates an upload is narrower than the header suggests. Confirmed by reading
the live rootfs of an EWS377AP v3 (`/lib/upgrade/`, `/bin/header`) and running
its own tools against a re-headed `ews377-fit-1.0.4-3` image:

**The upload check tests only `vendor_id` + `product_id`** (`check_senao_image_header`):

```sh
check_senao_image_header(){
	local SN_VENDOR_ID=0101
	local SN_PRODUCT_ID=011a          # 282 = EWS377AP v3
	[ "$senao_vendor_id" = "$SN_VENDOR_ID" -a "$senao_product_id" = "$SN_PRODUCT_ID" ] && return 0
	[ "$senao_magic_long" = "27051956" -o "$senao_magic_long" = "d00dfeed" ] && return 0
	return 1
}
```

Not the `model` string, **not** `md5sum`, **not** `chksum`. So to make a donor
image accepted, patch **only** `product_id` at `0x08` to the target's value
(e.g. `300 → 282`, two bytes). `vendor_id` is already `257` across the line.
`senao_image_header_check` also rejects `firmware_type == 0` images whose
`firmware_ver` major is `3` and minor `< 7`.

**`md5sum` / `chksum` are still not reproducible via `mksenaofw`** (verified: the
stored values match no payload-MD5 slice nor any header byte-sum/fold) — but you
don't need to recompute them:

- `md5sum` covers the **payload**, so a `product_id`-only header patch leaves it
  valid. The on-device de-wrap tool **`/bin/header -x`** re-checks it and prints
  `MD5 check OK!` on a patched image — proof the hash is payload-scoped.
- `chksum` is **not read** by the upgrade path at all, so its opacity doesn't
  matter.

**The payload is obfuscated with a universal (not per-model) key.** `sysupgrade`
runs `header -x` on any image whose first-4-byte magic is `00000000`; that strips
the 146-byte header and de-obfuscates the payload into a plain **FIT** (`d00dfeed`).
A FIT-model payload de-wraps cleanly on EWS377AP v3 hardware, so cross-model
decryption is **not** a failure mode here (it can be on lines that lack the tool
or use keyed images). After de-wrap, `platform_check_image` requires that FIT to
carry the board's mandatory sections — trivially true between same-board siblings
(see [model-equivalence](../hardware/model-equivalence.md)).
