# ECW cloud APs: `cloud_id` is the private-controller switch

Verified on hardware 2026-11-20 with an **ECW336** (firmware 1.8.118.12) against
a self-hosted **EPC 1.10.0**.

A stock ECW cloud AP pointed at a self-hosted controller will quietly ignore
**`force_ac`, DHCP option 43, and mDNS** and keep talking to EnGenius Cloud. The
reason is how the on-device agent is routed at boot, and the fix is a single
u-boot environment variable.

## Why the usual overrides do nothing on a stock unit

`/etc/init.d/ezmcloud` picks the agent's whole code path from `cloud_mode`:

```sh
cloud_mode=$(cloudmode.sh n)            # reads the u-boot env var `cloud_id`
if   [ "$cloud_mode" = "3" ];          then . /lib/cloud/minicloud-init.sh   # private / Fit
elif [ "$(check_controller)" = "1" ];  then . /lib/cloud/minicloud-init.sh
else                                        . /lib/cloud/ezmcloud-init.sh     # public cloud
fi
```

`check_ac()` — the function that reads `force_ac` / `dhcp_ac` / mDNS, with the
documented priority **Redundancy > Force AC > DHCP option > mDNS** — lives only
in `minicloud-init.sh`. A stock unit has **no `cloud_id`**, so `cloud_mode`
defaults to `0`, the *public-cloud* init is sourced, and that function is never
loaded. Setting `force_ac` on such a unit writes a value nothing reads.
`check_controller` returns private-mode only when an `epc_secret` or `fitmode=2`
already exists — chicken-and-egg on a fresh device.

## The fix

`cloudmode.sh` stores the selector in the u-boot env (`fw_setenv cloud_id`).
Value **`3` = "Private Cloud"** (the full map: `0` prod, `1` staging, `2` dev,
`3` private, `4`/`5`/`6` fit prod/staging/dev).

```sh
# From the u-boot CLI over UART — in practice the ONLY way on a stock cloud
# unit (append-only; works even when the OS is locked):
setenv cloud_id 3
saveenv

# If you somehow have a root shell, this also sets uci ezmcloud.minicloud=1:
cloudmode.sh 3 && reboot
```

> **You will not get that root shell on a stock ECW build — UART is mandatory.**
> Verified on ECW336 fw `1.8.118-12`: `POST /api/mgm/ssh {"enable":true}`
> genuinely starts dropbear (port 22 opens, banner `SSH-2.0-dropbear`, a real
> change from older ap-hk07 builds where the daemon never started) — but **every
> login is refused**: `root` → `"Not support for root account."`, `admin` →
> `"Not support for admin account."` It is not a passwd problem, since
> `/etc/passwd` gives both uid 0 and `/bin/ash`; the firmware patches the SSH
> auth path to whitelist neither.
>
> Nor is there an API substitute: `senao-openapi-server` dispatches to
> `ezmconfig`, which exports `get_cloudmode` (read-only) and the SSH toggle but
> **no cloud-mode setter**, and `/api/mgm/force_ac` returns
> `404 API URL NOT MATCH` on a stock unit because that handler lives in
> `www/cgi-bin/api.fcgi` — the *lighttpd/LSP* stack, which is inactive while the
> unit is in cloud mode. Budget one UART session per unit.

**Verify the env is still COMPLETE after `saveenv`** — `ethaddr`, `sn`,
`snextra`, `hw_id` all present. On some ECW/FIT units the real MAC exists *only*
in the env, so a corrupt/incomplete env is identity loss. Revert with
`cloud_id 0`.

In mode 3 the AP **never contacts EnGenius Cloud**, which is also the clean way
to handle a second-hand unit whose serial still sits in a previous owner's cloud
organisation — ownership there becomes irrelevant.

## After the mode is set

The AP sources `minicloud-init.sh`, honours `force_ac=<epc-ip>` (or discovers
the controller by mDNS), populates `fit_con_address`, and begins hitting
`/api/v1/checkin` on the controller. From there it's the normal self-hosted
onboarding (register the device, push config) — see
[EPC on Docker](epc-docker-ubuntu.md) §7.

### `force_ac` becomes settable here — and it removes the cable shuffle

Mode 3 also flips the web stack: `switch_lighttpd_lsp` enables the LSP GUI, so
the route that 404'd on the stock unit now answers.

```sh
GET  /api/mgm/force_ac                    # -> {"ip": ""}
POST /api/mgm/force_ac {"ip":"<epc-ip>"}  # -> 200, persists in UCI
# reboot to make the cloud agent re-evaluate
```

`force_ac` does **two** jobs, both worth knowing:

- `libcloud-init.sh:check_controller()` returns `1` when `cloud_mode=3` **or**
  `uci ezmcloud.ezmcloud.force_ac` is non-empty — so it is itself a second,
  independent door into the private-controller branch.
- `minicloud-init.sh` assigns `ac_addr=$force_ac_addr` — it is also the
  **controller address**.

Practical payoff: provision second-hand units on an **isolated segment** (so
they can never reach EnGenius Cloud before `cloud_id=3` is set), point them at
the controller with `force_ac`, and they check in **directly** across a routed
path. mDNS never has to cross the segment and the AP never has to be moved to
the controller's L2. A firewall permitting only the controller IP plus UDP 123
(NTP — the clock matters, see the `460` code below) is enough.

Setting `force_ac` live does **not** take effect on its own: `check_controller`
is evaluated when the cloud agent starts, so reboot after setting it.

### An unknown device adds itself to Pending Approval

A device the controller does **not** know checks in, gets **`404`**, and
`handle_auth_request` calls `add_pending_device`, writing a Redis hash:

```
pending → <serial>  {"model":"ECW336","type":"ap","mac":"…","sn":"…","ip":…,"timestamp":…}
```

That is the queue behind *Inventory → Register Devices → Pending Approval*.
Verified 2026-11-20: a freshly flashed ECW336 in `cloud_id=3` mode populated it
by itself, with the model correctly identified, **no manual DB seeding**.
Approve in the UI and the normal secret-negotiation / config-apply cycle runs.

Note the two failure codes separate cleanly, and looking for `pending` keys
while a device is already hand-written into Mongo finds nothing — a device that
*is* found never goes to pending:

| code | meaning | where the device goes |
|---|---|---|
| `404` | serial not in Mongo/Redis | **added to `pending`** → approve in UI |
| `463` | found, but its signing secret ≠ the stored one | stuck; fix the record |

**Prefer the pending/approve path over hand-seeding.** A hand-seeded device can
reach `200` check-ins and still never complete the config-apply/report loop
(`is_config_up_to_date` stays empty).

### Check-in status codes you'll see

The controller's check-in handler returns these (from `chicken.pyc`); knowing
them turns a stuck adoption into a one-line diagnosis:

| code | meaning | usual cause |
|---|---|---|
| `460` | invalid timestamp | AP clock >900 s off — fix NTP first |
| `461` | invalid nonce | replay / clock |
| `462` | invalid signature | pre-shared-key mismatch (mac/sn wrong) |
| `463` | invalid secret | device's stored secret ≠ what it signed with |
| `468` | invalid JSON | malformed body |

A freshly-seeded device signs with its pre-shared key
`lower(mac) + snextra + "0@ne$"` (note: `snextra`, not `sn`), so its controller
record must let that validate before check-in flips to `200`.

Concretely, what clears a `463`: the device's Redis record
(`device/<mac_lower>`) must carry `secret=preshared_initial` — the placeholder
that tells the controller to validate the AP's **pre-shared-key** signature
rather than compare against an already-negotiated secret. With that in place the
check-in goes to a steady `200`/online. And note the codes are checked in order,
so a `463` means the timestamp and the signature both passed: the mismatch
really is the stored secret, not the clock or the key derivation.

Because that key is derived from the MAC and the serial, those two values
together **are** the credential — a `printenv` paste that includes both is a key
disclosure, not just an identity leak.

### Config-delivery debounce (why a pushed config sometimes isn't sent)

Config is delivered in the check-in *response*, but `Chicken.get_checkin_reply`
gates it with a per-device latch in Redis:

```lua
-- cfg_ver = device/<mac>.config_version ; lsp_cfg_ver = device/<mac>.lsp_config_version
if not lsp_cfg_ver or (now - lsp_config_time) > 300 then
    HSET lsp_config_version = cfg_ver, lsp_config_time = now
    return false                         -- latch, send nothing this time
elseif lsp_cfg_ver < cfg_ver then return true   -- send the config
else return false end
```

So the correct order is **let one check-in latch `lsp_config_version`, then bump
`config_version` above it** (push the config). If you push *while* the latch is
empty, each check-in just re-latches `lsp = config_version` and never falls
behind, and the config (check-in response `size`) stays `0` forever — which
looks like the push failing when it is actually the debounce. Watch the caddy
log: the delivering check-in shows a non-zero `size`, then `0` on the next.

## Tr-band note (ECW336 / Wi-Fi 6E)

6 GHz is gated by SSID security: ECW336 firmware v1.6.71 —
*"Disable 6GHz radio if selected Wi-Fi encryption is not supported on the 6GHz
band."* 6 GHz needs **WPA3-Personal (SAE)** or OWE with PMF. Avoid WPA3/WPA2
*mixed* mode for first bring-up — v1.8.112 fixed a bug where mixed mode wrongly
disabled 6 GHz. In EPC the 6 GHz radio is a dedicated `ap_policy.radio_6g`
field, **not** an entry in the `ap_policy.radios` list (whose `type` is limited
to `2_4G`/`5G`/`5G-2`); the SSID's band binding is a separate `type_6g`
sub-object.

**Confirmed working on air** (ECW336, fw `1.8.118-12`, EPC 1.10.0): a Wi-Fi 6E
client associated on **channel 5 (6 GHz), 80 MHz, 802.11ax, WPA3-Personal**,
`Country Code: US`, with 2.4 GHz and 5 GHz up at the same time. **No AFC grant
was needed** — channel 5 is a 6 GHz **PSC** channel and Low-Power Indoor is
sufficient, so don't go hunting for a power-mode setting first.

Two traps, both of which cost real time:

1. **An ineligible SSID bound to 6 GHz disables the radio** — and it does so
   *even when* a separate WPA3 SSID is also bound to 6 GHz. Adding a WPA3 SSID
   is not enough; the WPA2 one must come **off** the band.
2. **`ap_policy.radio_6g` can silently fail to persist.** mongoengine hands back
   a default-populated embedded document for an unset field, so
   `if ap.radio_6g is not None` is True, `.save()` computes no delta, and
   nothing is written — a naive check reports success while Mongo still has no
   `radio_6g`. Verify against the raw collection and write it with pymongo
   `$set`.

A single SSID name across all three bands works well: same name and same
passphrase, with WPA2-PSK bound to 2.4/5 GHz and WPA3-Personal bound to 6 GHz.
Clients see one network and pick the band; older 2.4 GHz IoT gear keeps working.

One stale observation to ignore: earlier notes read "the AP logs `enable radio`
for `wifi0/1/3` but never `wifi2`, therefore 6 GHz is held down." That inference
was **unsound** — the `wifiN`↔band mapping was never established, and three
enabled radios is exactly what a working tri-band AP looks like.
