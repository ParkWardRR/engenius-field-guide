# Whitelisting an unknown model

Make the controller adopt a device whose model isn't in its default list — the
non-invasive alternative to [cross-flashing](cross-flashing.md). Edits
the controller, not the device.

The controller keeps known models in Mongo's `models` collection and mirrors them
into Redis. Add the model to Mongo, then sync to Redis.

## First: check it is actually unknown

"The controller won't adopt this device" is usually **not** a missing model, and
this page is the wrong tool for it. Query the collection before editing anything:

```javascript
db.models.find({}, {name:1, number:1, _id:0})          // everything it knows
db.models.findOne({name:/ECW336/})                     // or by model code: {number:"E4T"}
```

A worked example of getting this wrong: the ECW336 looked like a whitelisting
problem for a while, and it was not — EPC 1.10.0 already shipped it
(`{"name":"ECW336","number":"E4T"}`). The actual gate was on the **device**: the
ECW firmware's init reads the u-boot env `cloud_id`, and a stock unit has none,
so it defaults to the public-cloud path and never loads the code that reads
`force_ac`, DHCP option 43 or mDNS. Every controller-side override looked
ignored because nothing was reading it. See
[ecw-private-cloud-mode](ecw-private-cloud-mode.md).

Before concluding the model list is the problem, separate the failure modes —
the check-in response tells you which one you have. An unknown serial gets a
`404`; a device whose clock has drifted more than ~900 s gets a distinct
timestamp rejection (`460`), which is easy to mistake for an identity problem
because it turns up right after a bootloader env wipe resets the clock. Signature
and secret failures have their own codes again (`462`, `463`).

## Model document fields

| Field | Meaning |
|-------|---------|
| `type` | `"ap"` or `"switch"` |
| `name` | display name |
| `number` | 3-char [model code](model-codes.md) |
| `band` | e.g. `"2_4G\|5G"` |
| `category` | e.g. `"indoor"` |
| `ports` | array (switches); each `Port` has `id`, `poe_type`, `speed_cap` |
| `max_client_limit_24g` / `_5g` | AP client caps |
| `max_tx_power_limit_24g` / `_5g` | AP TX power caps |
| `dfs_support_type` | e.g. `["fcc","eu"]` |
| `support_poe`, `is_support_scanning_radio` | booleans |

## Method A — clone an existing model (Mongo shell)

Easiest: copy a similar model, change `name` + `number`, drop `_id`, reinsert.

```javascript
// in the mongo shell (see backend-access.md)
db.models.findOne({name:"EWS377-FIT"})    // copy the output
db.models.insertOne({
  type:"ap", name:"My Device", number:"XYZ", band:"2_4G|5G", category:"indoor",
  ports:[], max_client_limit_24g:128, max_client_limit_5g:128,
  max_tx_power_limit_24g:23, max_tx_power_limit_5g:23,
  dfs_support_type:["fcc","eu"], support_poe:false, is_support_scanning_radio:false
})
```

## Method B — from inside epc-api (Python)

Use the app's own model classes for less typo risk:

```python
# copy into the container, then: docker exec epc-api python /tmp/add_model.py
from squirrel.models_model import Model, Port
ap = Model(type=Model.type_ap, name="ECW230", band="2_4G|5G",
           category=Model.category_indoor, number="X42",
           dfs_support_type=["fcc","eu"])
Model.objects.insert([ap])
```

## Sync Mongo → Redis

After inserting, mirror to Redis:

```bash
docker exec epc-api python /app/db-init.pyc --model
```

The UI reads from Mongo, so you can add a device by serial without syncing — but
Redis drives **auto-discovery** into *Inventory → Register Devices → Pending
Approval*. Sync to get that.

Made a mistake? Delete the doc from Mongo, or reset the DB to defaults.
