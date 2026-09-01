# Credits and attribution

Much of the firmware and controller reverse-engineering knowledge here traces to
research notes published by **DaveCorder** at
<https://github.com/DaveCorder/EnGenius>.

This repository re-documents those **facts** — commands, numbers, header layouts,
methods — in original wording and organization. Facts aren't copyrightable; the
specific expression of a write-up is. Where knowledge came from DaveCorder's
notes, it was rewritten from scratch, not reproduced.

For his original write-ups and raw artifacts not reproduced here (full boot logs,
firmware dumps, binary analysis, photos), see the source repo:
<https://github.com/DaveCorder/EnGenius>.

## Derived from DaveCorder's research

- Self-hosting the Fit Controller on ARM / Raspberry Pi
- Backend data-store access (MongoDB/Redis, credential location)
- Adding unsupported device models to the controller
- Serial format and the Code27 check-character algorithm
- The Senao firmware image header and on-device image validation
- Cross-flashing hardware-equivalent models: ENS620EXT ↔ ENH1350EXT ↔ ECW160 and
  ECW230(v3) ↔ EWS377-FIT

## Original to this repository

The **EnGenius Private Cloud (EPC) on Podman / AlmaLinux** material — compose
translation, Docker-socket→Podman-socket mapping, SELinux labeling, and the field
notes — is original work. See
[docs/controllers/epc-podman-almalinux.md](docs/controllers/epc-podman-almalinux.md)
and [examples/epc-podman/](examples/epc-podman/).

## Trademarks

"EnGenius" and "Senao" are trademarks of their respective owners. This project is
unofficial, unaffiliated, and not endorsed by either company.
