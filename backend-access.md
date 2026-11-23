# Backend access (MongoDB + Redis)

Reach the controller's data stores directly — needed for
[adding models](add-unknown-models.md) and debugging.

## Credentials

Mongo and Redis passwords are stored in plaintext on the host at
`/epc/pipe/.epc-prod`. Source it to get `$MONGO_USER`, `$MONGO_PASSWORD`,
`$REDIS_PASS`:

```bash
source /epc/pipe/.epc-prod
```

## Redis

```bash
docker exec -it epc-db /usr/bin/redis-cli -a "$REDIS_PASS"
```

## MongoDB

Main database is `main`; the device whitelist is the `models` collection.

```bash
docker exec -it epc-db /usr/bin/mongo -u "$MONGO_USER" -p "$MONGO_PASSWORD" \
  --authenticationDatabase admin "mongodb://127.0.0.1:27017/main"
```

```javascript
db.models.find({}, {name:1, number:1, type:1})   // list known models
db.models.findOne({name:"EWS377-FIT"})           // one model's full doc
```

> Podman note: on a Podman host these `docker` commands work as-is if the
> `podman-docker` shim is installed; otherwise substitute `podman`.

## Newer EPC (1.10.x): `mongosh`, plural collections, and no stored admin password

Three things changed that break the recipes above on EPC 1.10.0:

- The shell is **`mongosh`**, not `mongo` (the old binary is gone).
- `--authenticationDatabase admin` is **required**; authenticating against
  `main` fails with `Authentication failed.`
- Device/network/org collections are **plural**: `devices`, `networks`, `orgs`
  (plus `redis_*` mirrors, `device_config_checksums`, `code27`, `licenses`,
  `fitregister`, `users`). `db.device` (singular) silently returns nothing.

```bash
U=$(sudo grep '^MONGO_USER='     /srv/appdata/epc/pipe/.epc-prod | cut -d= -f2-)
P=$(sudo grep '^MONGO_PASSWORD=' /srv/appdata/epc/pipe/.epc-prod | cut -d= -f2-)
sudo docker exec epc-db mongosh main --quiet \
  -u "$U" -p "$P" --authenticationDatabase admin --eval 'print(db.getCollectionNames().join(","))'
```

Note the compose/env path moved too — `/srv/appdata/epc/pipe/.epc-prod` here,
not `/epc/pipe/`. Locate it with
`sudo find / -maxdepth 5 -name '.epc-prod'`.

### Calling the REST API when nobody knows the web admin password

`.epc-prod` holds only the **Mongo/Redis** passwords. The web admin credential
is not stored anywhere on the host (it lives in `users` as a hash), and the
`tokens` collection is empty on a fresh install — so there is no token to
borrow. Resetting the admin password works but is intrusive.

Cleaner: the API signs its JWTs **HS256** with `routers/jwt_token.SECRET_KEY`
inside the `epc-api` container, so mint a short-lived admin token *there* and
the secret never leaves the container.

The claims that matter — getting these wrong is a silent `403 Permission
Denied`, not a `401`:

| claim | why |
|---|---|
| `sub` | the `authorization` decorator rejects a token without it |
| `scopes: ["system_admin","normal_user"]` | **the one everyone misses.** The decorator checks `sub` *and* `scopes`; no scopes ⇒ 403 |
| `user_email` / `operator_email` | `get_org_role(org_id, user_email, auth_source)` resolves the role from this |
| `auth_source: "local"` | same lookup; `local` for a normal install |
| `exp`, `iat` | ordinary expiry |

```python
# run inside the epc-api container
import sys, time; sys.path.insert(0, "/app")
import routers.jwt_token as j
from jose import jwt
now = int(time.time())
print(jwt.encode({
    "sub": ADMIN_EMAIL, "user_id": ADMIN_USER_ID,
    "scopes": ["system_admin", "normal_user"],
    "operator_name": "admin", "operator_email": ADMIN_EMAIL,
    "user_email": ADMIN_EMAIL, "auth_source": "local",
    "exp": now + 3600, "iat": now,
}, j.SECRET_KEY, algorithm=j.ALGORITHM))
```

Verify with a harmless read before relying on it:

```bash
curl -sk "https://127.0.0.1/api/v1/orgs/$ORG/inventory" -H "Authorization: Bearer $TOK"
```

`decode_token_user` also has a `DEBUG_MODE` bypass that impersonates a built-in
admin, but it is `False` on a normal install — don't count on it.

### Registering and assigning a device over the API

Same calls the UI makes. Bodies are **bare arrays**, which is easy to get wrong:

```bash
# register into org inventory — model is derived from the serial's model code
# (Code27Model), so no model field is needed: E4T -> ECW336
curl -sk -X POST "https://<epc>/api/v1/orgs/$ORG/inventory" \
     -H "Authorization: Bearer $TOK" -H 'Content-Type: application/json' \
     -d '[{"serial_number":"<SERIAL>"}]'

# assign to a network — AddDeviceValidator accepts only `id`
curl -sk -X POST "https://<epc>/api/v1/orgs/$ORG/hvs/$HV/networks/$NET/devices" \
     -H "Authorization: Bearer $TOK" -H 'Content-Type: application/json' \
     -d '[{"id":"<device_id>"}]'
```

`$HV` is the org's `hierarchy_view_id` — read it off any existing device.

A register call returns `mac: ""`: the API keys on the serial alone and binds the
MAC at first check-in. The record it writes is otherwise complete — the `ap`
subdoc's `profile.band` is derived from the model, so an `X42` serial comes back
as `ECW230v3` with `2_4G|5G` and an `E4T` one as `ECW336` with 6 GHz included.

Three prerequisites learned the hard way:

- **The registration API enforces the Code27 check character; the check-in path
  does not.** `POST /inventory` answers `400 {"status":"invalid"}` — with no hint
  as to which rule failed — for anything that is not 12 chars with a valid check
  char ([serial-numbers](serial-numbers.md)). Meanwhile a device whose `snextra`
  is a hand-made serial that fails that check still completes check-in and
  receives config, because check-in matches on the stored string only. That
  asymmetry is the whole reason hand-seeded Mongo records exist on this
  controller: a synthetic serial like `EPC2X420000000000000` can never be
  registered through the API, only written directly. Prefer generating a
  *valid* Code27 serial and going through the API — then the record is identical
  to a real unit's.

- **`fitregister.enable` must be `false`.** Otherwise registration tries to sync
  to EnGenius's hosted FitRegister and dies with *"Failed to get EPC info: EPC
  Sn not found in MongoDB"* — a self-hosted EPC has no FitCon serial. The
  `/fitregister/settings` PATCH is itself license-gated (`403 Connect License
  Not Found`), so set it in Mongo:
  `db.fitregister.updateOne({}, {$set:{enable:false}})`.
- **Licensing is not a prerequisite.** A device registers as
  `license_status: "expired"` and still adopts and receives config. Don't plan
  around "free license slots" — and be aware that device records can reference
  license IDs that no longer exist.

### A raw `$set` on a policy does not push

Editing a network policy with pymongo bypasses the model hooks, so no config
version is bumped and no device is notified. Trigger the real pipeline:

```python
from pkg.sublibs.trigger_update_device_config_sns import trigger_update_device_config_sns
trigger_update_device_config_sns({
    "org_id": ORG, "network_id": NET,   # omit device_id to cover the whole network
    "device_type": "ap",
    "headers": {"operator_email": ADMIN_EMAIL, "auth_source": "local"},
})   # returns None on success, an error string otherwise
```

Field-name trap: in the **policy** document an SSID's name is `name`;
`ssid_name` only exists in the *generated* device config. Likewise the policy
stores band bindings as `types` / `type_6g`, while the generated config uses
`ssid_types`.

### Never delete a live device record to free a licence

The check-in secret is a **shared** value: the controller holds a copy and the AP
holds a copy. Deleting a device record destroys the controller's copy, and the AP
keeps signing with a secret nothing else knows — producing a `463
INVALID_SECRET` / `462 INVALID_SIGNATURE` wall that looks like a firmware fault.
There is also no reason to do it: devices run happily as
`license_status: "expired"` (see above).

If it has already happened, do **not** try to recover the old secret. The
`Kaiwoo-signature` HMAC covers the POST body, which the reverse proxy does not
log, so it cannot be brute-forced from access logs — and it does not need to be.
An AP falls back to its initial pre-shared key once the controller rejects it, so
rebuilding a clean record is enough to make it re-pair:

1. Register the device (API if the serial is valid Code27, otherwise write
   `devices` + `redis_device` directly) with the serial the AP actually sends —
   read it from the `Kaiwoo-authentication` header's `sn=` field, not from your
   notes.
2. Fix the Redis key name. An API registration creates **`device/<serial>`**
   because `mac` is still `""` at that moment, while every working device is
   **`device/<mac_lower>`**. Until renamed, check-in returns `404 "Device Is Not
   In The Network"` — a *different* 404 from "not registered", distinguishable
   only by response size. Add the `mac` field too.
3. Write a **fresh** secret (`secrets.token_urlsafe(24)`) and
   `device_pairing = ""`.

The AP then re-pairs on its next check-in, walking `404 → 463 → 200`. A transient
`463` is **normal** mid-pairing, not a failure — only a `463` that persists past
a couple of check-in intervals means anything.
