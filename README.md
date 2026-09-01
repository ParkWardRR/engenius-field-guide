<div align="center">

# EnGenius Field Guide

### Make EnGenius/Senao gear do what you want.

Self-host the controller. Adopt the devices it says it can't. Cross-flash between
models. Decode the hardware. The practical, unofficial notes EnGenius doesn't ship.

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

---

EnGenius's on-prem tooling is powerful and barely documented. This guide is the
missing manual: tested, no-fluff, straight to the commands.

## What you can do with it

🖥️ **Run the controller yourself.** Deploy EnGenius Private Cloud (EPC) as
Podman containers on AlmaLinux — hardened, SELinux **Enforcing**, reboot-proof —
or run the Fit Controller stack on a Raspberry Pi. No appliance, no cloud.
→ [EPC on Podman/AlmaLinux](epc-podman-almalinux.md) ·
[Fit Controller on ARM](self-hosting-fitcon-arm.md)

📡 **Adopt devices the controller rejects.** That "unsupported" AP usually works
fine — the controller just doesn't have it whitelisted. Add the model to its
database, or cross-flash the device to a supported sibling.
→ [Whitelist a model](add-unknown-models.md) ·
[Cross-flashing](cross-flashing.md) ·
[EWS377AP v3 → Cloud/FIT walkthrough](crossflash-ews377apv3-walkthrough.md)

🔧 **Go under the hood.** Decode or forge serials (the Code27 checksum), read the
Senao firmware header, and reach the controller's MongoDB/Redis directly.
→ [Serial numbers](serial-numbers.md) ·
[Firmware format](firmware-format.md) ·
[Backend access](backend-access.md)

## Two deep-dive walkthroughs

Fully worked, evidence-based, start-to-finish:

- **[Cross-flash an EWS377AP v3 → Cloud/FIT](crossflash-ews377apv3-walkthrough.md)** —
  the one-field `product_id` re-head, how to get root past the restricted CLI,
  flashing, reading firmware off-box, and why *adoption* (the blank-serial trap —
  the controller keys devices by serial, and a crossflash reports `sn=0000`) is
  the real gate.
- **[EPC on Podman/AlmaLinux](epc-podman-almalinux.md)** — the full 7-container
  deploy, the Docker-socket → podman-socket port, SELinux Enforcing, and (§8) the
  device-onboarding agent + pipes + the serial-keyed check-in flow.

## Start here

| Your goal | Go to |
|-----------|-------|
| Self-host a controller | [landscape](landscape.md) → [EPC on x86](epc-podman-almalinux.md) or [Fit Controller on ARM](self-hosting-fitcon-arm.md) |
| Adopt a device the controller rejects | [whitelist the model](add-unknown-models.md) or [cross-flash it](crossflash-ews377apv3-walkthrough.md) |
| Decode/generate a serial | [serial-numbers](serial-numbers.md) |
| Understand product lines / shared hardware | [overview](overview.md), [model-equivalence](model-equivalence.md) |

## All docs

| Doc | Covers |
|-----|--------|
| [overview](overview.md) | Product lines and which controller manages what |
| [landscape](landscape.md) | ezMaster vs Fit Controller/EPC; device compatibility |
| [epc-podman-almalinux](epc-podman-almalinux.md) | EPC 1.8.8 as Podman containers on AlmaLinux (incl. device onboarding) |
| [self-hosting-fitcon-arm](self-hosting-fitcon-arm.md) | Fit Controller stack on ARM / Raspberry Pi |
| [backend-access](backend-access.md) | MongoDB + Redis access |
| [add-unknown-models](add-unknown-models.md) | Whitelist an unsupported model |
| [custom-https-cert](custom-https-cert.md) | Replace the controller's TLS cert |
| [serial-numbers](serial-numbers.md) | Serial format + Code27 check char |
| [model-codes](model-codes.md) | Known 3-char model codes |
| [firmware-format](firmware-format.md) | Senao image header, mksenaofw, validation |
| [cross-flashing](cross-flashing.md) | Flashing foreign firmware: reference (method + risks) |
| [crossflash-ews377apv3-walkthrough](crossflash-ews377apv3-walkthrough.md) | Full worked EWS377AP v3 → Cloud/FIT conversion |
| [model-equivalence](model-equivalence.md) | Which models share hardware |
| [examples/epc-podman/](examples/epc-podman/) | Podman compose + env template |

## Scope

Unofficial community notes for education and interoperability — not endorsed by or
affiliated with the vendor. "EnGenius" and "Senao" are trademarks of their
owners. Expect gaps; verify before you rely on anything, especially firmware
flashing (it can brick hardware).

## Contributing

Corrections and additions welcome — especially verified
[model codes](model-codes.md) and hardware-equivalence pairs. Open
an issue or PR.

## License

[Blue Oak Model License 1.0.0](LICENSE). Underlying knowledge credited in
[CREDITS.md](CREDITS.md) — much of it re-documented from
[DaveCorder's notes](https://github.com/DaveCorder/EnGenius).
