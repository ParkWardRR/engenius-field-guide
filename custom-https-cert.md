# Custom HTTPS certificate

Replace the controller's default self-signed web-UI cert with your own. Applies to
the Linux/Docker EPC stack.

nginx in `epc-api` serves the UI from `/app/nginx.crt` + `/app/nginx.key`.

## 1. Copy your cert + key in

Get your PEM cert and key onto the host, then into the container:

```bash
docker cp cert.pem epc-api:/app/nginx.crt
docker cp key.pem  epc-api:/app/nginx.key
```

## 2. Make it persist

Container edits vanish on the next recreate. Snapshot the running container and
repoint the compose `image:`:

```bash
docker commit "$(docker container ls | grep epc-api | awk '{print $1}')" epc-api:1.8.7-custom-01
```

In `/epc/pipe/docker-compose.yml`:

```yaml
    epc-api:
        image: epc-api:1.8.7-custom-01     # was ${REPOSITORY_URI}epc-api:${VERSION}
```

Then `./epc-prod.sh down && ./epc-prod.sh up` (or your `podman-compose` equivalent).

> A private CA (e.g. one you run in OPNsense) issuing a cert for the controller's
> hostname is a common setup, so browsers on your network trust it without warnings.
