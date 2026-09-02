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

## Integrity fields are proprietary (re-heading caveat)

`mksenaofw` reads and re-encodes the header structure, but the **`md5sum` and
`chksum` in stock EnGenius images are not reproducible** from the OpenWRT tool's
formulas. Verified against `EWS377APv3-v3.9.3.2` and `ews377-fit-1.0.4-3`:

- `md5sum` (`0x28`) does **not** equal MD5 of the stored payload (`file[146:]`)
  for any offset/length slice tested — the payload is **encrypted**, and the
  hash is over the plaintext (computed pre-encryption / pre-header).
- `chksum` (`0x58`) does **not** match a byte-sum of the header (± CAPWAP), nor a
  16-bit one's-complement fold, nor a whole-image sum — the vendor uses a
  checksum the public tool doesn't implement.

Consequences for anyone trying to **re-head** a donor image (patch `product_id` +
`model` so the target accepts it):

- The donor `md5sum` stays valid *as long as you leave the payload untouched* —
  it covers the payload, not the header you're editing.
- You **cannot** recompute a vendor-valid `chksum` with public tooling. A pure
  header patch only works if the target's upgrade check ignores `chksum` (only
  gates on magic + `product_id` + `model` + `md5sum`). Confirm that in the
  device's `senao_image_header_check` before relying on it.
- Even an accepted re-headed image may not boot: the encrypted payload is
  decrypted on-device, so a donor payload keyed to a different model can fail to
  decrypt/boot. This is a separate failure mode from the header gate.
