<div align="center">

# EnGenius Field Guide

Unofficial field notes for EnGenius/Senao WiFi hardware and the self-hosted
controllers that manage it — running the local Fit Controller / EnGenius Private
Cloud (EPC), reaching the device backend, decoding serials, and cross-flashing
hardware-equivalent models.

[![License: Blue Oak 1.0.0](https://img.shields.io/badge/License-Blue_Oak_1.0.0-0a7bbb.svg)](LICENSE)
[![Status: unofficial](https://img.shields.io/badge/status-unofficial-orange.svg)](#scope)
[![PRs welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](#contributing)
![Last commit](https://img.shields.io/github/last-commit/ParkWardRR/engenius-field-guide)
![Repo size](https://img.shields.io/github/repo-size/ParkWardRR/engenius-field-guide)
![Commit activity](https://img.shields.io/github/commit-activity/m/ParkWardRR/engenius-field-guide)
[![Stars](https://img.shields.io/github/stars/ParkWardRR/engenius-field-guide?style=social)](https://github.com/ParkWardRR/engenius-field-guide/stargazers)

![OpenWrt](https://img.shields.io/badge/OpenWrt-00B5E2?logo=openwrt&logoColor=white)
![Podman](https://img.shields.io/badge/Podman-892CA0?logo=podman&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-2496ED?logo=docker&logoColor=white)
![AlmaLinux](https://img.shields.io/badge/AlmaLinux-0D597F?logo=almalinux&logoColor=white)
![Raspberry Pi](https://img.shields.io/badge/Raspberry_Pi-A22846?logo=raspberrypi&logoColor=white)
![MongoDB](https://img.shields.io/badge/MongoDB-47A248?logo=mongodb&logoColor=white)
![Redis](https://img.shields.io/badge/Redis-FF4438?logo=redis&logoColor=white)

</div>

## How it fits together

```mermaid
flowchart LR
    EWS["EWS<br/>managed APs"] --> EZ["ezMaster<br/>(EOL)"]
    ENS["ENS / ENH<br/>outdoor + indoor"] --> EZ
    FIT["-FIT variants<br/>EWS377-FIT, EWS850-FIT"] --> FC["Fit Controller / EPC<br/>(self-hostable)"]
    ECS["ECS / ECS-Lite<br/>switches"] --> FC
    ECW["ECW<br/>cloud APs"] --> CLOUD["EnGenius Cloud<br/>(hosted)"]
    ECW -.->|since early 2024| FC
```

## Start here

| Goal | Go to |
|------|-------|
| Self-host a controller | [landscape](docs/controllers/landscape.md) → [EPC on Podman/AlmaLinux (x86)](docs/controllers/epc-podman-almalinux.md) or [Fit Controller on ARM](docs/controllers/self-hosting-fitcon-arm.md) |
| Adopt a device the controller rejects | [whitelist the model](docs/controllers/add-unknown-models.md) or [cross-flash it](docs/firmware/cross-flashing.md) |
| Decode/generate a serial | [serial-numbers](docs/firmware/serial-numbers.md) |
| Understand product lines / shared hardware | [overview](docs/overview.md), [model-equivalence](docs/hardware/model-equivalence.md) |

### Adopting a device the controller won't take

```mermaid
flowchart TD
    A["Controller won't adopt a device"] --> B{"Device works,<br/>just not on the list?"}
    B -->|Yes| C["Whitelist the model<br/>in Mongo + sync Redis"]
    B -->|"No — wrong firmware line"| D{"Hardware-equivalent<br/>Fit/ECW sibling?"}
    D -->|Yes| E["Cross-flash to the sibling"]
    D -->|No| F["Not supported"]
    C --> G(["Device adopted"])
    E --> G
```

## Contents

| Doc | Covers |
|-----|--------|
| [overview](docs/overview.md) | Product lines and which controller manages what |
| [controllers/landscape](docs/controllers/landscape.md) | ezMaster vs Fit Controller/EPC; device compatibility |
| [controllers/epc-podman-almalinux](docs/controllers/epc-podman-almalinux.md) | EPC 1.8.8 as Podman containers on AlmaLinux |
| [controllers/self-hosting-fitcon-arm](docs/controllers/self-hosting-fitcon-arm.md) | Fit Controller stack on ARM / Raspberry Pi |
| [controllers/backend-access](docs/controllers/backend-access.md) | MongoDB + Redis access |
| [controllers/add-unknown-models](docs/controllers/add-unknown-models.md) | Whitelist an unsupported model |
| [controllers/custom-https-cert](docs/controllers/custom-https-cert.md) | Replace the controller's TLS cert |
| [firmware/serial-numbers](docs/firmware/serial-numbers.md) | Serial format + Code27 check char |
| [firmware/model-codes](docs/firmware/model-codes.md) | Known 3-char model codes |
| [firmware/firmware-format](docs/firmware/firmware-format.md) | Senao image header, mksenaofw, validation |
| [firmware/cross-flashing](docs/firmware/cross-flashing.md) | Flashing foreign firmware: method + risks |
| [hardware/model-equivalence](docs/hardware/model-equivalence.md) | Which models share hardware |
| [examples/epc-podman/](examples/epc-podman/) | Podman compose + env template |

## Scope

Unofficial community notes for education and interoperability, not endorsed by or
affiliated with the vendor. "EnGenius" and "Senao" are trademarks of their
owners. Expect gaps; verify before relying on anything.

## Contributing

Corrections and additions welcome — especially verified [model codes](docs/firmware/model-codes.md)
and hardware-equivalence pairs. Open an issue or PR.

## License

[Blue Oak Model License 1.0.0](LICENSE). Underlying knowledge credited in
[CREDITS.md](CREDITS.md).
