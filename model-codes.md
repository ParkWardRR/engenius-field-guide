# Model codes

Every EnGenius device has a 3-char model code at [serial](serial-numbers.md)
positions 5–7. The controller's model whitelist keys on this `number` field, so
you need it when [whitelisting a model](add-unknown-models.md).

## Known codes

| Model | Code | Type |
|-------|------|------|
| EWS377-FIT | `X45` | AP |
| ECW230 | `X42` | AP |
| ECS2510FP | `RCF` | Switch |

Confirm a device's own code by reading its serial. To find the code for a model
already in the controller DB, query its `models` entry (see
[add-unknown-models](add-unknown-models.md)) — the `number` field
is the code.

> Contributions welcome — this table is small on purpose (only codes verified
> from real devices are listed).
