# Running EnGenius Private Cloud (EPC) on Docker / Ubuntu

Field notes for deploying **EPC 1.9.0** on **Docker CE / Ubuntu 24.04** — the
vendor-supported runtime. This is the **simpler, more stable** alternative to
the [Podman/AlmaLinux path](epc-podman-almalinux.md), which we ran in
production for four months before switching (see §5 for the post-mortem).

> If you followed the Podman guide first and it worked in your lab, great —
> this doc explains why you might still want to switch, and gives the Docker
> deploy end-to-end.

## 1. Why Docker over Podman

EPC's containers are designed for Docker. The `epc-api` container bind-mounts
`/var/run/docker.sock` and uses the Docker API to manage its own stack. On
Podman this requires **8+ shims** (see §5), each a potential regression vector.
Docker needs **zero** — the vendor compose works as-is.

| Concern | Docker | Podman |
|---------|--------|--------|
| Container DNS | `127.0.0.11` (matches the shipped nginx.conf) | `10.89.0.1` (requires nginx patch) |
| Docker socket | Native | Symlink `podman.sock` → `docker.sock` |
| SELinux volume labels | Not needed (Ubuntu default: AppArmor) | Required: `container_file_t` fcontext + `:z` + `label=disable` |
| Compose compatibility | `docker compose` plugin (native) | `podman-compose` (behavioral differences) |
| `epc.sh` installer | Works directly | Requires `ENVIRONMENT=1` + `-it` removal + shims |

## 2. VM setup (Proxmox example)

| Setting | Value |
|---------|-------|
| OS | Ubuntu 24.04 LTS (cloud-init image) |
| CPU | 8 vCPU, `cpu: host` |
| RAM | 16 GB ceiling / 8 GB balloon floor |
| Disk | 64 GB thin on NVMe, `ssd=1,discard=on` |
| NIC | virtio |
| Machine type | q35, `--vga std` |
| `onboot` | **1** — critical, the default is 0 and the VM won't autostart |

For official sizing, see [the Podman guide §2](epc-podman-almalinux.md#2-official-sizing).

## 3. Deploy procedure

### 3.1 Prerequisites

```bash
sudo apt-get update && sudo apt-get install -y curl wget net-tools

# Docker CE
curl -fsSL https://get.docker.com | sudo sh
sudo systemctl enable --now docker

# docker-compose standalone (EPC scripts call `docker-compose`, not `docker compose`)
sudo ln -sf /usr/libexec/docker/cli-plugins/docker-compose \
  /usr/local/bin/docker-compose

# host directories EPC expects
sudo mkdir -p /epc /root/cert \
  /srv/docker/mongodb/data/db /srv/docker/redis \
  /var/log/epc/{mongodb,redis,nginx,gunicorn,api,raccoon,otter,agent}

# pre-stage images
EPC_VER=1.9.0
for i in agent api db raccoon otter mdns radius; do
  sudo docker pull public.ecr.aws/d3g4m7o9/epc-$i:$EPC_VER
done
```

### 3.2 Installer

```bash
wget -q "http://engenius-epc.s3.us-west-2.amazonaws.com/dev/$EPC_VER/epc.sh" \
  -O /tmp/epc.sh

# patch: skip Docker/apt install (already done), remove -it (no TTY over SSH)
sed -i 's/^ENVIRONMENT=0/ENVIRONMENT=1/' /tmp/epc.sh
sed -i 's/docker exec -it/docker exec/g' /tmp/epc.sh
```

### 3.3 Create config files manually

**The installer won't create these on a fresh install** — its `setup_env` has
`keep_license=1` hardcoded (upgrade path), so `sed` silently fails on missing
files and `create_uuid.pyc` crashes with `NoSectionError: No section: 'ocu'`.

```bash
# config.ini
printf "[ocu]\nstage = production\nversion = $EPC_VER\n" \
  | sudo tee /epc/config.ini

# .epc-prod env file
cat <<EOF | sudo tee /epc/pipe/.epc-prod
REPOSITORY_URI=public.ecr.aws/d3g4m7o9/
VERSION=$EPC_VER
HOST_IP=$(hostname -I | awk '{print $1}')
ENABLE=False
MASTERIP=
SLAVEIP=
EOF

# .agent env file
cat <<EOF | sudo tee /epc/pipe/.agent
REPOSITORY_URI=public.ecr.aws/d3g4m7o9/
VERSION=$EPC_VER
HOST_IP=$(hostname -I | awk '{print $1}')
EOF
```

### 3.4 Bring up the stack

```bash
# main EPC stack
sudo docker-compose -p epc -f /epc/pipe/docker-compose.yml \
  --env-file /epc/pipe/.epc-prod up -d

# agent (separate compose file)
sudo docker-compose -p agent -f /epc/pipe/agent.yml \
  --env-file /epc/pipe/.agent up -d
```

### 3.5 DB init

```bash
# create UUID
sudo docker exec epc-api python /app/create_uuid.pyc

# wait for DB readiness
while [ "$(sudo docker exec epc-api python /app/db-init.pyc -t 2>/dev/null)" != "1" ]; do
  sleep 2; printf "."
done; echo

# import defaults (creates 69 models, 1 org, 1 network, 1 user, etc.)
sudo docker exec epc-api python /app/db-init.pyc -i
```

The installer also generates Mongo/Redis credentials and appends them to
`.epc-prod`. After the installer runs, verify:

```bash
grep MONGO_USER /epc/pipe/.epc-prod    # should have a random hex user
grep REDIS_PASS /epc/pipe/.epc-prod    # should have a random hex password
```

### 3.6 Post-init steps

These are required before the deployment is usable:

```bash
source /epc/pipe/.epc-prod

# 1. Enable fitregister (the device registration gate — defaults to false)
sudo docker exec epc-db mongo -u "$MONGO_USER" -p "$MONGO_PASSWORD" \
  --authenticationDatabase admin main --eval \
  'db.fitregister.updateOne({}, {$set: {enable: true, wan_ip: "<YOUR_IP>"}})'

# 2. Disable the license cron (reverts fitregister.enable every minute)
sudo docker exec epc-api sh -c \
  'crontab -l | sed "s|^\(\*/1.*--fitregister\)|# \1|;s|^\(\*/1.*--license\)|# \1|" | crontab -'

# 3. Add [redis] section to config.ini (raccoon needs it)
if ! grep -q '^\[redis\]' /epc/config.ini; then
  printf "\n[redis]\nredis_host = epc-db\nredis_port = 6379\n" \
    | sudo tee -a /epc/config.ini
fi

# 4. Copy notification templates
cp /epc/pipe/notification_templates.json /epc/notification_templates.json

# 5. Named pipes + fitdog (required for device onboarding)
for p in req msg host host_msg; do
  [ -p /epc/pipe/$p ] || sudo mkfifo /epc/pipe/$p
done
sudo chmod +x /epc/pipe/fitdog /epc/pipe/host.sh 2>/dev/null
sudo sh -c 'nohup /epc/pipe/fitdog >/var/log/epc/fitdog.log 2>&1 &'
```

### 3.7 First-run signup

Open `https://<vm-ip>/` or `http://<vm-ip>:8080/` in a browser. The first user
to sign up gets **system_admin** role automatically — the signup flow modifies
the seed user created by `db-init.pyc`. After signup, login via the web UI.

The dashboard should show "Everything is OK!" with empty device counts.

## 4. Gotchas

### EPC bugs (both Docker and Podman)

| # | Issue | Fix |
|---|-------|-----|
| 1 | `config.ini` not created on fresh install | Write manually before running installer (§3.3) |
| 2 | `.epc-prod` / `.agent` env files not created | Create manually (§3.3) |
| 3 | `fitdog` holds SSH TTY | Background the installer or redirect output |
| 4 | Redis device hashes flushed on every `epc-api` restart | Re-seed devices after restart |
| 5 | `fitregister.enable` reverts to false every minute | Disable the license cron (§3.6) |
| 6 | Vendor compose overwrites your hardened config | Rename: `mv docker-compose.yml docker-compose.yml.vendor-backup` |
| 7 | Expired vendor TLS cert (CN `www.engeniusnetworks.com`, expired 2025-07-10) | Replace with real cert via [custom-https-cert](custom-https-cert.md) |
| 8 | `epc-raccoon` logs "Missing redis in env file" | Cosmetic — the binary reads `.epc-prod` for `REDIS_PASS` and works fine |
| 9 | Database name is `main`, not `epc` | All Mongo queries must target the `main` database |
| 10 | `POST /api/v1/jwt-token` is NOT a login endpoint | It refreshes an existing token. Login is handled by the SPA internally |
| 11 | `notification_templates.json` missing | Copy from `/epc/pipe/` to `/epc/` |
| 12 | `prestart.sh` resets Mongo to defaults (some builds) | Delete it: `docker exec epc-api rm /app/prestart.sh` |
| 13 | Mongo `IndexOptionsConflict` on version upgrade | `db.users.dropIndex('profile.email_1')`, then restart |
| 14 | HMAC signature check rejects all AP check-ins | Bytecode-patch `chicken.pyc`: set `valid_signature` + `valid_signature_ex` to `return True` (see §7) |
| 15 | Caddy drops AP check-ins (status 0 / NOP) | AP connects by IP → Host header doesn't match hostname block. Add a `:443 { tls internal }` catch-all (§6.3) |
| 16 | Redis `device/<mac>` key must use lowercase MAC | Check-in code calls `mac.lower()` before Redis lookup |
| 17 | Mongo Device model field is `serial_number`, not `serial` | MongoEngine `Device.objects(serial_number=...)` maps to db field `serial_number` |
| 18 | `validate_license` blocks config delivery | Without license bypass, check-in returns 200 with empty body — AP never gets config. Patch alongside signature (§7.1) |
| 19 | `get_checkin_reply` crashes on empty `checkin_reply` | `'NoneType' object has no attribute 'decode'` — config must be generated and pushed before first check-in (§7.3) |
| 20 | Device missing `ap` embedded document → config generation fails | `get_ap_device_config` accesses `device.ap.profile.category`. Set the `ap` subdocument with `profile.band` + `profile.category` from the `models` collection |
| 21 | `config_compressed` format mismatch | `checkin_reply` stores **base64-encoded JSON** (not zlib compressed). Storing raw zlib bytes causes `UnicodeDecodeError` on check-in |
| 22 | Default SSID is open (auth_type: disabled) | The seed SSID profile "EnGenius WiFi" ships with no security. Set WPA2-PSK before pushing config (§7.4) |

### Infrastructure

| # | Issue | Fix |
|---|-------|-----|
| I1 | Proxmox `onboot=0` (default) — VM doesn't autostart | `qm set <vmid> --onboot 1` immediately after creation |
| I2 | Proxmox `--vga serial0` — noVNC blank | Use `--vga std`; delete serial0 device |

## 5. Why we moved off Podman — a post-mortem

We ran EPC 1.8.8 on **Podman 5.8.2 / AlmaLinux 10.2** for four months
(June–September 2026). It was lab-functional under SELinux Enforcing and
adopted one real AP. We switched to Docker after two production incidents and
too many close calls.

### 5.1 The shim stack

Running EPC on Podman requires layering these on top of the vendor stack:

| Shim | Why |
|------|-----|
| `podman-docker` package | `docker` CLI → `podman` |
| `podman-compose` symlinked as `docker-compose` | compose compatibility |
| `podman.sock` symlinked to `/var/run/docker.sock` | Docker-API socket |
| `/etc/containers/nodocker` | silence the Podman emulation banner |
| SELinux `container_file_t` fcontext on 4 trees | volume access under Enforcing |
| `label=disable` on `epc-api` | let it reach the Podman socket |
| `:z` labels on all volume mounts | SELinux shared access |
| nginx `resolver` patch `127.0.0.11` → `10.89.0.1` | Podman DNS is not at Docker's address |
| `checkin.pyc` bytecode patch | bypass HMAC-SHA256 signature mismatch |

Each shim is a regression vector. Vendor updates can break any of them.

### 5.2 What broke

**nginx resolver (the #1 Podman-on-EPC failure):** Docker's embedded DNS lives
at `127.0.0.11`; Podman's at `10.89.0.1`. The image ships `resolver
127.0.0.11` in `nginx.conf`. Without a bind-mounted patched config, every
`/device/register` request → **502** (30s timeout, indefinite retry).

**Redis FLUSHALL on epc-api restart:** `epc-api` flushes ALL Redis keys on
startup, then re-creates org/network/model data from MongoDB — but does NOT
re-create `device/<mac>` hashes. After any container restart, all device
check-ins 404 until someone manually re-seeds the Redis device hashes.

**Vendor compose landmine:** The installer stages the vendor compose file
alongside any hardened version. If `epc.sh`'s background retry, `fitdog`, or a
self-update runs `up` against the vendor file, it **silently reverts every
customization** — including replacing your real TLS cert with EnGenius's
baked-in **expired** self-signed one (CN `www.engeniusnetworks.com`, issued
2015, expired 2025-07-10).

### 5.3 Production incidents

1. **epc-api restart → all APs offline.** Redis device hashes flushed, no
   automatic re-seeding. Required manual intervention to re-seed each device.
2. **Self-update reverted TLS cert.** A background process ran `up` against the
   vendor compose file, replacing the ACME cert with the expired vendor cert.
   Browsers showed certificate warnings.

### 5.4 Recommendation

The Podman guide in this repo documents a working port and is still accurate
for lab use. For a production homelab where reliability matters, Docker on
Ubuntu eliminates the entire shim stack and just works.

## 6. TLS with Caddy reverse proxy

The vendor cert is expired (CN `www.engeniusnetworks.com`, 2025-07-10). Use
Caddy as a reverse proxy with automatic Let's Encrypt via DNS-01.

### 6.1 Remap container ports

```yaml
# docker-compose.yml:
epc-api:     "443:443" → "8443:443"   (keep "8080:8080")
epc-raccoon: "80:80"   → "8081:80"
```

```bash
docker-compose -p epc -f /epc/pipe/docker-compose.yml \
  --env-file /epc/pipe/.epc-prod up -d --no-deps epc-api epc-raccoon
```

### 6.2 Install Caddy

Build Caddy with the Cloudflare DNS module (or copy one that already has it):

```bash
xcaddy build --with github.com/caddy-dns/cloudflare
sudo mv caddy /usr/local/bin/caddy
```

### 6.3 Configure

```bash
useradd -r -s /usr/sbin/nologin -d /var/lib/caddy caddy
mkdir -p /etc/caddy /var/lib/caddy/.local /var/lib/caddy/.config /var/log/caddy
chown -R caddy:caddy /var/lib/caddy /var/log/caddy

echo 'CF_API_TOKEN=<your-cloudflare-dns-api-token>' > /etc/caddy/env
chmod 600 /etc/caddy/env
```

`/etc/caddy/Caddyfile`:

```caddyfile
{
    default_sni your-epc.example.com
}

your-epc.example.com {
    tls {
        dns cloudflare {env.CF_API_TOKEN}
        resolvers 1.1.1.1 8.8.8.8
    }
    handle /device/* {
        reverse_proxy 127.0.0.1:8081
    }
    handle {
        reverse_proxy 127.0.0.1:8080
    }
    encode gzip
}

# Catch-all for AP connections by IP (Host header won't match the hostname block)
:443 {
    tls internal
    handle /device/* {
        reverse_proxy 127.0.0.1:8081
    }
    handle {
        reverse_proxy 127.0.0.1:8080
    }
}

:80 {
    handle /device/* {
        reverse_proxy 127.0.0.1:8081
    }
    handle {
        reverse_proxy 127.0.0.1:8080
    }
}
```

`default_sni` is required because APs connect by raw IP (no SNI hostname).
The `:443` catch-all with `tls internal` handles the HTTP routing for those
requests — without it, Caddy completes the TLS handshake but drops the
request with status 0.

Create a systemd unit with `User=caddy`, `EnvironmentFile=/etc/caddy/env`,
`AmbientCapabilities=CAP_NET_BIND_SERVICE`. Then:

```bash
systemctl daemon-reload && systemctl enable --now caddy
```

Caddy obtains a cert via Cloudflare DNS-01 within ~15 seconds. Works even if
the hostname only resolves on your LAN.

> **Note:** After recreating `epc-api`, the license cron resets — re-disable it
> (§3.6 step 2) or `fitregister.enable` reverts to false within a minute.

## 7. Device adoption

### 7.1 Bypass HMAC signature check and license validation

The check-in handler in `chicken.pyc` validates two things that fail on
self-hosted EPC:

1. **HMAC signature** (`valid_signature` / `valid_signature_ex`): AP computes
   `mac.lower() + serial_number + "0@ne$"` as the HMAC key over
   `auth_header + url_path + body`. The EPC-side verification fails because
   the message format doesn't match.
2. **License validation** (`validate_license`): Without a valid license, the
   check-in returns 200 with an empty body — the AP never receives config.

Patch all three methods to `return True`:

```bash
docker exec epc-api python3 << 'PYEOF'
import marshal, struct, types, shutil

path = "/app/pkg/general/chicken.pyc"
shutil.copy2(path, path + ".bak")

with open(path, "rb") as f:
    data = f.read()

header, code = data[:16], marshal.loads(data[16:])
new_bc = bytes([100, 1, 83, 0])  # LOAD_CONST True; RETURN_VALUE

def patch(co):
    consts = list(co.co_consts)
    changed = False
    for i, c in enumerate(consts):
        if isinstance(c, types.CodeType):
            if c.co_name in ("valid_signature", "valid_signature_ex", "validate_license"):
                consts[i] = c.replace(co_code=new_bc, co_consts=(None, True), co_stacksize=1)
                changed = True
            else:
                r = patch(c)
                if r is not c:
                    consts[i] = r
                    changed = True
    return co.replace(co_consts=tuple(consts)) if changed else co

with open(path, "wb") as f:
    f.write(header)
    marshal.dump(patch(code), f)
PYEOF

docker restart epc-api
```

### 7.2 Register a device

After the signature/license bypass, register the device in both Mongo and
Redis. The check-in flow checks Mongo first (`Device.objects(serial_number=...)`),
then Redis (`device/<mac_lowercase>`).

Look up your org/network IDs first:

```bash
source /epc/pipe/.epc-prod

docker exec epc-db mongo -u "$MONGO_USER" -p "$MONGO_PASSWORD" \
  --authenticationDatabase admin main --eval \
  'db.orgs.find({}, {name:1}).forEach(printjson)'
# → gives you <org_id>

docker exec epc-db mongo -u "$MONGO_USER" -p "$MONGO_PASSWORD" \
  --authenticationDatabase admin main --eval \
  'db.networks.find({}, {name:1}).forEach(printjson)'
# → gives you <network_id>
```

```bash
# 1. Insert into Mongo (field must be serial_number, not serial)
#    The "ap" embedded document is required for config generation.
docker exec epc-db mongo -u "$MONGO_USER" -p "$MONGO_PASSWORD" \
  --authenticationDatabase admin main --eval '
  db.devices.insertOne({
    type: "ap", series: "cloud",
    name: "<device-name>",
    model: "<model>",
    serial_number: "<serial>",
    mac: "<MAC-lowercase>",
    org_id: "<org_id>",
    network_id: "<network_id>",
    hierarchy_view_id: "<network_id>",
    is_sync_config: false,
    is_config_up_to_date: false,
    ap: {
      profile: {
        band: "<band-from-models-collection>",
        category: "<indoor|outdoor>"
      },
      is_mesh_enable: false,
      radios: [], ssid_profiles: [], fast_handover: []
    },
    created_time: new Date(),
    modified_time: new Date()
  })'
```

Look up band/category from the models collection:

```bash
docker exec epc-db mongo -u "$MONGO_USER" -p "$MONGO_PASSWORD" \
  --authenticationDatabase admin main --eval \
  'db.models.find({name: "<model>"}, {band:1, category:1}).forEach(printjson)'
```

```bash
# 2. Seed Redis (MAC must be lowercase)
docker exec epc-db redis-cli -a "$REDIS_PASS" HMSET "device/<mac_lower>" \
  mac "<mac_lower>" sn "<serial>" serial_number "<serial>" \
  org_id "<org_id>" falcon_nid "<network_id>" network_id "<network_id>" \
  model "<model>" type "ap" name "<device-name>" \
  secret "preshared_initial" status "online" \
  config_version "" checkin_reply "" config_compressed "" \
  is_sync_config "false" is_in_trial_zone "false" \
  series "cloud" wan_ip "" device_id "<mongo_objectid>"

# 3. Re-disable license crons (container restart re-enables them)
docker exec epc-api sh -c \
  'crontab -l | sed "s|^\(\*/1.*--fitregister\)|# \1|;s|^\(\*/1.*--license\)|# \1|" | crontab -'

# 4. Re-enable fitregister
docker exec epc-db mongo -u "$MONGO_USER" -p "$MONGO_PASSWORD" \
  --authenticationDatabase admin main --eval \
  'db.fitregister.updateOne({}, {$set: {enable: true}})'
```

### 7.3 Generate and push config

The AP won't receive wireless config until it's explicitly generated and
stored in Redis. The `get_checkin_reply` function in `chicken.pyc` reads
`checkin_reply` and `config_version` from the device hash — if they're empty
it crashes with `'NoneType' object has no attribute 'decode'`.

Generate and push the config from inside the container:

```bash
docker exec epc-api python3 << 'PYEOF'
import sys, json, base64, time
sys.path.insert(0, "/app")

from pkg.general.redis_pool import RedisPool
redis = RedisPool.get_connection()
from squirrel.device_model import Device
from squirrel.network_collection_model import Network
from pkg.general.update_device_config import get_ap_device_config

MAC = "<mac_lowercase>"                  # e.g. 88:dc:97:04:44:07
SN  = "<serial_number>"                 # e.g. EPC1X420000000000000
NID = "<network_id>"                    # e.g. 60b83f5fdcf61564c17e9f2f

device  = Device.objects(serial_number=SN).first()
network = Network.objects(id=NID).first()

config     = get_ap_device_config(network, device, {})
config_str = json.dumps(config, separators=(",", ":"))
config_b64 = base64.b64encode(config_str.encode("utf-8")).decode("utf-8")

lua = '''
redis.call("HMSET", KEYS[1], "checkin_reply", ARGV[1], "diff", ARGV[2], "is_sync_config", ARGV[3])
redis.call("HINCRBY", KEYS[1], "config_version", 1)
'''
redis.eval(lua, 1, f"device/{MAC}", config_b64, "", "True")
print(f"Config pushed ({len(config_b64)} bytes)")
PYEOF
```

The AP picks up the config on its next check-in (~12 seconds), applies it
(radios restart, ~2 minutes of silence), then resumes checking in with the
new config version. Verify:

```bash
tail -f /var/log/caddy/access.log | grep checkin
# Expect: status=200 size=<non-zero> on first delivery, then size=0 after.
```

**Key format detail:** `checkin_reply` in Redis stores **base64-encoded JSON**
(not compressed). The `set_device_config` Lua script in `redis_api.pyc` calls
`HINCRBY config_version 1` on each push. `config_compressed` is unused by
the set path — don't store raw zlib bytes there or the check-in will crash.

### 7.4 Set SSID security

The default SSID profile has `auth_type: "disabled"` (open WiFi). Set
WPA2-PSK before pushing config:

```bash
source /epc/pipe/.epc-prod

docker exec epc-db mongo -u "$MONGO_USER" -p "$MONGO_PASSWORD" \
  --authenticationDatabase admin main --eval '
  db.networks.updateOne(
    {_id: ObjectId("<network_id>")},
    {$set: {
      "policy.ap_policy.ssid_profiles.0.name": "<SSID-name>",
      "policy.ap_policy.ssid_profiles.0.security.auth_type": "WPA2-PSK",
      "policy.ap_policy.ssid_profiles.0.security.wpa.passphrase": "<passphrase>",
      "policy.ap_policy.ssid_profiles.0.security.wpa.type": "aes",
      "policy.ap_policy.ssid_profiles.0.ieee_802_11w.is_enable": true,
      "modified_time": new Date()
    }})'
```

Valid `auth_type` values: `disabled`, `WPA2-PSK`, `WPA2-Enterprise`, `OWE`,
`WPA3-Personal`, `WPA2/WPA3-Personal`, `WPA3-Enterprise`.

After updating the SSID, regenerate and push config (§7.3).

### 7.5 Point the AP at the controller

```bash
curl -sk -u admin:admin -X POST "https://<ap-ip>/api/mgm/force_ac" \
  -H "Content-Type: application/json" \
  -d '{"force_ac_ip": "<epc-ip>", "force_ac_port": 443}'
```

The AP starts checking in every ~12 seconds. Verify with:

```bash
tail -f /var/log/caddy/access.log | grep checkin
```

## 8. Backend access


Credentials are in plaintext at `/epc/pipe/.epc-prod`:

```bash
source /epc/pipe/.epc-prod

# Redis
docker exec -it epc-db redis-cli -a "$REDIS_PASS"

# MongoDB (database is `main`)
docker exec -it epc-db mongo -u "$MONGO_USER" -p "$MONGO_PASSWORD" \
  --authenticationDatabase admin main
```

## 9. Source of truth

- Installer: `http://engenius-epc.s3.us-west-2.amazonaws.com/dev/<version>/epc.sh`
- Package (compose + configs): `.../dev/<version>/epc-pkg.tar.gz`
- Available versions: 1.8.8, 1.8.9, 1.9.0, 1.9.1 (others return 403)
- ARM/FitController installer: `https://epc-release.s3.us-west-2.amazonaws.com/epc-prod.sh`
- Docs: https://doc.engenius.ai/home-epc-quick-start-guide

## 10. See also

- [EPC on Podman/AlmaLinux](epc-podman-almalinux.md) — the full Podman port (lab use)
- [Backend access](backend-access.md) — MongoDB + Redis shell access
- [Custom HTTPS cert](custom-https-cert.md) — replace the expired vendor cert
- [Device onboarding](epc-podman-almalinux.md#8-device-onboarding--the-agent-the-pipes-and-the-mtls-gate) — agent, pipes, and the serial-keyed check-in flow (same on Docker)
- [Add unknown models](add-unknown-models.md) — whitelist an unsupported device model
