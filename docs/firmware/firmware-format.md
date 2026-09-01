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

| Field | Notes |
|-------|-------|
| `vendor_id` | `257` for EnGenius/Senao |
| `product_id` | per model — e.g. `300` = EWS377-FIT, `284` = ECW230v3 |
| `firmware_type` | image type |
| `filesize` | payload size |
| `checksum` | header/image checksum |
| `magic` | `0x12345678` |
| `firmware_ver` | version triplet |
| `datecode` | build date |
| `model_size` | length of the model string |

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
