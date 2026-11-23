# Model codes

Every EnGenius device has a 3-char model code at [serial](serial-numbers.md)
positions 5–7. The controller's model whitelist keys on this `number` field, so
you need it when [whitelisting a model](add-unknown-models.md).

Each model also belongs to a **series** — `cloud`, `cloud-lite`, `fit`, or
`neutron` (the ezMaster/EnSky line) — which is what determines how a controller
manages it. The codes and series below were read directly from an EnGenius
Private Cloud (EPC) `models` collection, so they're the controller's own values.

## Known codes

| Model | Code | Series | Type |
|-------|------|--------|------|
| ECW230 | `X41` | cloud | AP |
| ECW230v3 | `X42` | cloud | AP |
| ECW230S | `V41` | cloud | AP |
| ECW336 | `E4T` | cloud | AP (Wi-Fi 6E tri-band) |
| EWS377APv2 | `X43` | neutron | AP |
| EWS377APv3 | `X44` | neutron | AP |
| EWS377-FIT | `X45` | fit | AP |
| EWS276-FIT | `X46` | fit | AP |
| EWS357-FIT | `X29` | fit | AP |
| EWS356-FIT | `X25` | fit | AP |
| EWS375-FIT | `C42` | fit | AP |
| EWS850-FIT | `XC3` | fit | AP |
| ECS2510FP | `RCF` | — | Switch |
| EWS2510-FIT | `RC2` | fit | Switch |
| EWS2910P-FIT / EWS2910FP-FIT | `GC1` / `GCF` | fit | Switch |
| EWS7928P-FIT / EWS7928FP-FIT | `G21` / `G2F` | fit | Switch |
| EWS7952P-FIT / EWS7952FP-FIT | `G41` / `G4F` | fit | Switch |

Note the sibling hardware **ECW230v3 (`X42`), EWS377-FIT (`X45`), and EWS377APv3
(`X44`)** are the same board in three series — see
[model-equivalence](model-equivalence.md). (Watch the `X41`/`X42` split: `X42` is
ECW230**v3**; base ECW230 is `X41`.)

Codes are not all `X__`: `V41`, `C42`, `XC3`, `E4T`, and the `R__`/`G__` switch
codes show the first character varies. **ECW336 = `E4T`**, verified two ways —
EPC 1.10.0's `models` collection (`"name":"ECW336","number":"E4T"`) and the
position in a real unit's serial (`2290` + `E4T` + unit tail), the same slot
EWS377-FIT uses for `X45`. Its firmware-header `product_id` is **291**, and the
u-boot env packs both as `hw_id=01010123` (vendor `0x0101`=257, product
`0x0123`=291).

## A code is not unique across eras

The stock firmware carries the vendor's own legacy conversion table in
`/usr/sbin/snconverter.sh` — byte-identical in Cloud6E ECW336 1.8.118-12 and
Fit6 EWS377-FIT 1.1.65-2, so it is frozen SDK content, not per-series. It covers
the ten ezMaster-era models whose old 9-char serials were migrated into this
format, and two of its entries disagree with the controller table above:

| Code | `snconverter.sh` (legacy) | EPC `models` (current) |
|------|---------------------------|------------------------|
| `X25` | EWS330AP | EWS356-FIT |
| `CCE` | ENS620EXT | ECW160 |

Neither source is wrong. `CCE` is one board wearing two series names —
ENS620EXT and ECW160 are [hardware-equivalent](model-equivalence.md). `X25` is a
real retire-and-reuse: the 11ac-era EWS330AP is not the Wi-Fi 6 EWS356-FIT, and
neither EWS330AP nor ENS620EXT appears anywhere in EPC 1.10.0's 73-model table.
So **decode positions 5–7 with the device's era and series in hand** — a code
alone does not pin a model for hardware old enough to predate the controller.

The script agrees with EPC on every other entry it carries (`X23`, `X24`, `X26`,
`X28`, `X43`, `X44`, `XC2`, `CC2`), which is what makes those two stand out.

Confirm a device's own code by reading its serial. To find the code for a model
already in the controller DB, query its `models` entry (see
[add-unknown-models](add-unknown-models.md)) — the `number` field is the code.

> Contributions welcome — this table lists only codes verified from real devices
> or a controller's own `models` collection.
