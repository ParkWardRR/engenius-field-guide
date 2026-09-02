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

Confirm a device's own code by reading its serial. To find the code for a model
already in the controller DB, query its `models` entry (see
[add-unknown-models](add-unknown-models.md)) — the `number` field is the code.

> Contributions welcome — this table lists only codes verified from real devices
> or a controller's own `models` collection.
