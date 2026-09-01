# Self-hosting the Fit Controller on ARM / Raspberry Pi

Run the FitCon100 container stack on generic ARM (e.g. a Raspberry Pi) instead of
the appliance. A Pi 3B+ works but is slow — painfully so during DB init.

> ⚠️ Unofficial, at your own risk. Vendor images in an unsupported config.

## Quick path

1. Install Docker from Docker's apt repo.
2. Download `epc-prod.sh`, raise `INIT_TIMEOUT`, run `./epc-prod.sh install 1.6.15-arm`.
3. Create a `br-lan` bridge for `epc-mdns`.
4. Wait out DB init (finish by hand if it times out).
5. Apply post-install fixes, then snapshot edits so they survive restart.

## 1. Docker

```bash
sudo apt install curl wget net-tools apt-transport-https ca-certificates gnupg lsb-release
curl -fsSL https://download.docker.com/linux/$(lsb_release -si | tr '[:upper:]' '[:lower:]')/gpg \
  | sudo gpg --batch --yes --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
echo "deb [arch=arm64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/$(lsb_release -si | tr '[:upper:]' '[:lower:]') $(lsb_release -cs) stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list
sudo apt update && sudo apt install -y docker-ce docker-ce-cli containerd.io
```

## 2. Installer + timeout fix

The ARM/FitCon variant uses `epc-prod.sh`. Its DB-init timeout defaults to **60s**
— far too short on ARM (init can take **30+ min** on a Pi 3B+). Raise it first:

```bash
wget https://epc-release.s3.us-west-2.amazonaws.com/epc-prod.sh
chmod +x epc-prod.sh
sed -i -e 's/^INIT_TIMEOUT=60$/INIT_TIMEOUT=60000/' epc-prod.sh
```

## 3. `br-lan` bridge

`epc-mdns` uses host networking and expects a bridge named **`br-lan`** on ARM
(the FitCon100 bridges its two Ethernet ports). Create it and enslave your NIC:

```bash
sudo systemctl enable systemd-networkd
sudo nmcli connection add type bridge con-name 'br-lan' ifname 'br-lan'
sudo nmcli connection add type ethernet slave-type bridge con-name 'Ethernet' ifname eth0 master 'br-lan'
sudo nmcli con mod 'br-lan' ipv4.method dhcp
```

## 4. Install

```bash
./epc-prod.sh install 1.6.15-arm     # ARM builds carry the -arm suffix
```

**Slow DB init:** if the installer times out (*"Need to import default data
later…"*), init keeps running in the background — don't proceed until it's done.
Monitor, then finish by hand if needed:

```bash
watch "ps auwwx | grep pyc | grep -v grep"   # done when db-init.pyc stops and gunicorn appears

while [ ! $(docker exec -it epc-api sh -c 'python /app/db-init.pyc -t') -eq 1 ]; do sleep 1; printf "."; done
docker exec -it epc-api sh -c 'python /app/db-init.pyc -i >/dev/null 2>&1'
```

## 5. Post-install fixes (required)

```bash
# 1. prestart.sh resets Mongo to defaults on every start — delete it
docker exec epc-api rm /app/prestart.sh

# 2. fitdog expects /root/fitcon/epc; everything installs to /epc
mkdir -p /root/fitcon && ln -s /epc /root/fitcon/epc
```

3. On some builds `epc-raccoon` lacks its `/epc` volume and won't start. Add to
   its `volumes:` in `/epc/pipe/docker-compose.yml`:
   ```yaml
               - /epc:/epc
   ```

Then `./epc-prod.sh down && ./epc-prod.sh up`; check `docker container ls`. A
container stuck `restarting` failed at startup and will retry forever.

## 6. Persist container edits

Containers reset to their image on recreate, losing in-container changes (deleted
`prestart.sh`, swapped cert, etc.). Snapshot and repoint the compose `image:`:

```bash
docker commit "$(docker container ls | grep epc-api | awk '{print $1}')" epc-api:1.6.15-custom-01
```
```yaml
    epc-api:
        image: epc-api:1.6.15-custom-01
```

## Related

- [backend-access](backend-access.md) · [add-unknown-models](add-unknown-models.md) · [custom-https-cert](custom-https-cert.md)
