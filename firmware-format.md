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
| `0x08` | `product_id` | 4 | per model — the upload gate keys on this (see [table below](#known-product_ids)) |
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

### Known `product_id`s

All read directly from real vendor images (`vendor_id` is `257` across the line).
The value is the firmware-header product id — **not** the 3-char serial
[model code](model-codes.md); a device has both.

| `product_id` | Model | Board note |
|---|---|---|
| `182` | EWS377AP v2 (and the older `ews377ap-all` images) | prior generation |
| `275` | ECW230 | base cloud AP |
| `282` | EWS377AP v3 | **cross-flash sibling** |
| `284` | ECW230v3 | **cross-flash sibling** |
| `285` | ECW230S | related cloud AP (extra scanning radio); not a verified cross-flash target |
| `291` | ECW336 | **different board** (`ap-hk01-c6`) — same container, not a cross-flash target |
| `300` | EWS377-FIT | **cross-flash sibling** |

The three **siblings** (`282`/`300`/`284`) are the same `ap-hk07` board in three
series — the one-field re-head swaps between them. Note the older EWS377APv3
build `v3.8.2.2` carries the model string `EWS377AP` (no `v3`) yet still reports
`product_id 282`, so identify by the id, not the string.

**`291` is in the table to be recognised, not to be flashed.** The ECW336's
images parse identically — same `vendor_id 257`, same plaintext header, same
offsets — but it is a different board
([model-equivalence](model-equivalence.md#ecw336--same-soc-not-the-same-board)),
so a re-head in either direction produces an image whose payload describes the
wrong hardware. Its id is corroborated twice over: stable at `0x08` from
firmware `1.6.67-1` through `1.8.118-12`, and a live unit's u-boot env packs the
same number as `hw_id=01010123` — vendor `0x0101` (257) + `0x0123` (291). That
agreement is itself unusual: on the EWS377-FIT `hw_id`'s low half is `0x012B`
(299) while the header says 300, so do not assume the two numbering spaces line
up.

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

## The payload is XOR-obfuscated, not encrypted

An earlier version of this page concluded the payload was **encrypted**, because
the header `md5sum` matched no slice of the stored bytes. That was the right
observation and the wrong inference. The payload is obfuscated with a plain
**8-byte repeating XOR**:

```
key   = 783c9ecf67b359ac        # key[i % 8], phase 0 from the payload start
```

De-obfuscate it and you get an ordinary **u-boot FIT image** — no decryption, no
per-model key, nothing device-side. The header itself stays plaintext, which is
exactly why the file *looks* opaque and why scanning a stock `.bin` for `UBI#`
or gzip magic finds nothing.

The tell, if you ever meet an unfamiliar image: runs of `0x00` in the plaintext
surface the key repeating verbatim. Look around offset `0xA0`.

```python
key = bytes.fromhex("783c9ecf67b359ac")
raw = open("stock.bin", "rb").read()
hdr = 0x88 + model_len                      # header stays plaintext
fit = bytes(b ^ key[i % 8] for i, b in enumerate(raw[hdr:]))
# fit now starts with the FDT magic d0 0d fe ed
```

It is the **same key** OpenWrt's `senao-header` build step uses (`-m
783c9ecf67b359ac` in `ipq807x.mk`) — so one key covers both stock vendor images
and community-built ones.

**This resolves the `md5sum` puzzle.** `md5sum` (`0x28`) is the MD5 of the
*de-obfuscated* FIT, which is why it never matched the stored payload. Verified
on ECW336 `1.8.118-12`: the de-obfuscated FIT's MD5 equals the header's own
`md5sum` field exactly. So the hash is checkable with public tooling after all —
de-obfuscate first, then hash.

`chksum` (`0x58`) remains unexplained: it matches no byte-sum of the header
(± CAPWAP), no 16-bit one's-complement fold, and no whole-image sum.

### Consequences for re-heading

- The donor `md5sum` stays valid as long as you leave the payload untouched —
  it covers the payload, not the header you're editing.
- You still **cannot** recompute a vendor-valid `chksum`. A pure header patch
  works only if the target's upgrade check ignores it (gating on magic +
  `product_id` + `model` + `md5sum`). Confirm that in the device's
  `senao_image_header_check` before relying on it.
- **Drop the "it may fail to decrypt" worry — and replace it with a real one.**
  There is no decryption step to fail. The actual failure mode is that the
  payload is a *complete, correct image for the donor's hardware*: it carries
  the donor's DTS model, `qcom,ath11k-calibration-variant` and Wi-Fi
  `board-2.bin`. Re-headed onto a true sibling that is survivable (wrong model
  reported, radios on the donor's RF data — fine for a test, not a release).
  Re-headed across **boards** it is not: the image describes the wrong ethernet
  topology and PHYs, so the unit can boot perfectly and simply have no network.
  See [model-equivalence](model-equivalence.md#ecw336--same-soc-not-the-same-board).
- **Read the embedded `flash.scr` rather than guessing offsets.** The FIT
  carries the vendor's own flash script, which is the authoritative offset
  reference for that image; it gates on `machid` and `soc_hw_version`.
