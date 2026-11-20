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
# From a root shell on the AP (preferred — also sets uci ezmcloud.minicloud=1):
cloudmode.sh 3 && reboot

# Or from the u-boot CLI over UART (append-only; works even when the OS is locked):
setenv cloud_id 3
saveenv
```

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
band."* 6 GHz needs **WPA3-Personal (SAE)** or OWE with PMF; a WPA2-PSK SSID
will not bring the third radio (`wifi2`) up, and the AP logs it skipping
`enable radio wifi2`. Avoid WPA3/WPA2 *mixed* mode for first bring-up — v1.8.112
fixed a bug where mixed mode wrongly disabled 6 GHz. In EPC the 6 GHz radio is a
dedicated `ap_policy.radio_6g` field, **not** an entry in the `ap_policy.radios`
list (whose `type` is limited to `2_4G`/`5G`/`5G-2`).
