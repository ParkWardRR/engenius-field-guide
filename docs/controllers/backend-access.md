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
