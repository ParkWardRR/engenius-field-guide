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

### Renaming devices and networks, and where a "room" label goes

Ask the running app for its route table rather than guessing — this saved a lot
of 405s:

```bash
docker exec epc-api python3 -c "
import sys; sys.path.insert(0,'/app')
import main
for r in main.app.routes:
    m=getattr(r,'methods',None)
    if m: print(','.join(sorted(m)), r.path)
"
```

Two traps it exposes:

- **AP updates live under an extra `/aps/` segment.** Renaming is
  `PATCH /orgs/{org}/hvs/{hv}/networks/{net}/devices/aps/{device_id}`. The
  sibling `/devices/{device_id}` is **DELETE-only** — PATCH and PUT there both
  answer `405 Method Not Allowed`, and so does every method on the `/devices`
  collection. Switches and gateways have their own
  `/devices/switches/{id}` and `/devices/gateways/{id}` PATCH routes.
- Network rename is `PATCH /orgs/{org}/hvs/{hv}/networks/{network_id}` with
  `{"name": "..."}`. It propagates to every device's `network_name`. Note
  `network.pyc` carries a `BLOCKED_ROOT_NAME` constant, so some names are
  refused.

Body is a plain object (not the bare array that register/assign want):

```bash
curl -sk -X PATCH ".../devices/aps/$DEV" -H "Authorization: Bearer $TOK" \
     -H 'Content-Type: application/json' \
     -d '{"name":"LivingRoomAP","description":"Living room"}'
```

**There is no tag or label field on a device.** The schema is `name`, `model`,
`mac`, `serial_number`, `description`, … — so a location/room belongs in
**`description`**, which renders in the device list. If you need real grouping,
the only structural tool is the hierarchy: org → hierarchy view → network.

Renaming a device **bumps its config version and triggers a push**, so expect
`is_config_up_to_date: false` for a minute afterwards and an offline unit to
stay false until it next checks in. That is normal, not a failed rename.

Keep EPC's device names identical to the DHCP/DNS reservations. Matching the two
means a name in the controller, a lease, and a DNS record all refer to the same
box, which is what makes "which AP is this?" answerable at 2am.

### Radio and roaming tuning via the AP policy API

`PATCH /orgs/{org}/hvs/{hv}/networks/{net}/policy/aps` with `{"radios": [...]}`
round-tripped from the GET. Four traps, all of which cost time:

- **`fast_handover` is the roaming gate, and it is per-radio.** It is an object
  with three independent thresholds, all `is_enable:false` by default:
  `force_disconnect` (shed a client below RSSI), `association_request` (refuse
  to associate), `probe_request` (don't answer probes). This is what makes
  802.11r actually fire — with it off, clients ride a link to failure and then
  re-associate from scratch instead of fast-transitioning.
- **On 6 GHz, `fast_handover` is the *only* gate.** The 6G radio object has no
  `rssi_threshold` key at all (2.4G and 5G do, defaulting to −95). So "there's
  no way to gate 6 GHz" is wrong — it is there, under a different name.
- **`channel_list` has an asymmetric read/write shape.** GET returns a plain
  list (`[]`), the *delivered* config wraps it as
  `channel_candidate_list: {"is_enable":…, "channel_list":[…]}`, and **PATCH
  wants the plain list**. Sending the wrapped object fails with
  `400 {"channel_list": "Only lists and tuples may be used in a list field"}`.
- **There is no per-AP radio override through the API.** `PATCH
  …/devices/aps/{device_id}` declares no body model — it validates inline and
  accepts only `name`, `description`, `profile`, `ethernet`, `ssid_profiles`.
  A `radios` key is rejected as `Invalid Body` in every shape. Per-AP channel
  pinning therefore needs the web UI (whose login is RSA-encrypted in browser
  JS, so not scriptable) or a direct write to `devices.ap.radios` with
  `is_override`. Network-wide policy plus a candidate list is the only
  API-supported lever.

**Judging channel assignment: wait for convergence first.** `ap_acs_result`
entries logged *during* a config push reflect the **old** config, because the
AP reloads wifi and re-runs ACS before applying what it just fetched. Right
after a push that excluded DFS, the ACS log still showed picks of 104/120/136
(all DFS); once `is_config_up_to_date` went 5/5, the same APs moved to
non-DFS 149/153. Reading the log too early looks exactly like "the firmware
ignores the constraint".

**ACS is choosing blind on this fleet.** Every 5 GHz and 6 GHz pick logs
`totalrssi=0` and every 2.4 GHz pick logs `totalrssi=65535` — a sentinel, not a
measurement. With no survey input its choices are close to arbitrary, which is
how it put three of four APs on 6 GHz ch 5 and the two highest-overlap APs on
the same 2.4 GHz channel. Constraining the candidate list bounds the damage but
does **not** make it assign distinct channels.

**Verifying the gates are live: watch for two new event types.** Once
`fast_handover` is enabled, EPC's `log` collection starts emitting
**`ap_fast_handover`** and **`ap_kick_station`** events. Neither appears at all
while the feature is off, so their presence is direct proof the gates are
firing — far better evidence than inferring it from an RSSI improvement, which
could just be clients moving. Expect `ap_kick_station` to account for the
disappearance of far clients.

**Band steering cannot cross SSIDs.** `band_steering` operates only within one
SSID that is enabled on multiple bands. With a split-SSID design (one SSID per
band) it can never move a client from the 2.4 GHz SSID to the 5/6 GHz one — only
removing the 2.4 SSID from that client's saved networks will.

### `force_ac` can be lost in place on an already-adopted AP

Known risk: a *freshly flashed* AP boots with an empty `force_ac`. Less obvious,
and seen on a live fleet 2026-11-24: **an AP that has been adopted and running
for hours can lose `force_ac` on its own**, without a reboot or a reset.

The symptom is deceptive. EPC showed the AP `raccoon.online: false` with a
`last_seen` an hour stale, which reads like a dead AP — but it was entirely
healthy: 0 % packet loss at 0.5 ms, local GUI `200`, **uptime 11 h 27 m** (so no
reboot), cpu 0 %, 54 % memory free, and still serving a client on 5 GHz. Only
its controller pointer was gone, so it had silently stopped checking in. With no
DHCP option 43 on that segment there was nothing to re-discover the controller
with.

Diagnosis is one call per AP — don't infer it from EPC's online flag:

```bash
curl -s "http://$AP/api/mgm/force_ac" -H "Authorization: Bearer $TOK"
# -> {"data":{"ip":""}}   <-- orphaned
```

Fix is the same as for a fresh unit (`POST /api/mgm/force_ac` + `/api/sys/apply`);
recovery was immediate, with the AP back `online: true` inside 30 s.

Two things worth taking from this. **An AP reporting offline in the controller
while answering ping and serving clients is almost always a controller-pointer
or agent problem, not a hardware one** — check `force_ok`/`force_ac` and its
uptime before touching power or cabling. And **audit `force_ac` across the whole
fleet when one AP goes quiet**, since the useful question is whether it is
spreading; here exactly one of five had lost it.

### A channel policy only takes effect when ACS next runs

Setting `channel_list` / `is_exclude_dfs_channel` does **not** move an AP that is
already on air. The constraint is applied by the AP's next auto-channel
selection, and ACS can go hours without re-running — on a five-AP fleet there
were **0 ACS runs in the four hours** after a policy push.

The result looks like selective non-compliance. After pushing
`is_exclude_dfs_channel: true` with `channel_list: [36,44,149,157]`, four APs
happened to be sitting on 149/153/40 (inside the set, so apparently obedient)
while one sat on **104 — a DFS channel outside the set**. The tempting
conclusion is that *that* unit's firmware ignores the policy, and it is wrong:
every AP was still on a pick made *before* the policy arrived. Rebooting the
odd one out made it re-select immediately, and it chose 149 and ch 6 — both
inside the lists, non-DFS.

Two practical consequences:

- **Verify a channel policy by forcing a re-selection, not by reading the
  current channel.** A reboot of one AP is the cheapest conclusive test, and
  costs nothing if you pick an AP with no associated clients.
- **A policy push leaves the fleet's channel distribution essentially
  unchanged** until each AP independently re-runs ACS. If you need the plan
  actually in force, nudge each AP; otherwise expect the old, arbitrary
  assignment to persist. Whether that matters depends entirely on utilization —
  at 1–2 % occupancy it does not.

Beware the related misread on the **channel-utilization endpoint**: it returns a
fixed 288-slot, 24-hour, 5-minute grid and **pads absent samples with `null`**.
Reading the tail of `details[]` therefore shows `curr_chan: null` for a
perfectly healthy AP. Compare a known-good AP over the same window before
concluding a unit reports no telemetry — a native ECW336 and a cross-flashed
ECW230v3 showed 161 and 182 nulls out of 288 respectively, with their last
populated sample at the *same* timestamp.

### Measure airtime before paying to fix a channel plan

The most useful thing EPC exposes for RF decisions is
`GET …/devices/aps/{id}/channel-utilization` — a 5-minute series of per-channel
utilization plus the AP's current channel per band. Pull it before acting on a
channel plan, because a plan that looks broken on paper can be free in practice.

Worked example from a five-AP fleet, 934 samples over 8.5 h. ACS had put **three
of four APs on 6 GHz ch 5**, which reads as the fleet's worst defect:

| 6 GHz channel | Occupancy | median | p90 | max |
|---|---|---|---|---|
| ch 5 | **3 APs sharing** | 1 % | 5 % | 16 % |
| ch 37 | 1 AP alone | 1 % | 5 % | 10 % |

Statistically identical. **Co-channel contention matters only in proportion to
utilization**, and at ~1 % occupancy there is nothing to contend for. Acting on
it would have meant an unsupported write to `devices.ap.radios` for no
measurable gain.

Per band on the same fleet:

| Band | median | p90 |
|---|---|---|
| **2.4 GHz** | **26 %** | **49 %** |
| 5 GHz | 2 % | 11 % |
| 6 GHz | 1 % | 5 % |

Two things fall out of that. The only loaded band was **2.4 GHz** — and its load
was **external**, since it carried just two IoT clients, so the airtime was
neighbours and interference rather than our own traffic. That makes channel
*choice* matter far more there than our own reuse does, and pooling per-channel
figures across APs is how you pick: ch 6 measured worst (median 32 %, p90 54 %)
against ch 1 at 23 %/36 %, so narrowing the candidate list to `[1, 11]` is a
real, zero-risk gain where per-AP pinning was not.

The corollary is about where to spend effort generally: the bands carrying all
the *client* traffic (5/6 GHz here, 92 % of clients) were the idle ones, and the
congested band carried almost no clients. Utilization and client count point in
opposite directions, so check both before concluding anything.
