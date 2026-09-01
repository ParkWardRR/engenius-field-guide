# Controller landscape and device compatibility

Which on-prem controller manages which devices, and what to do when it refuses
to adopt one.

## Quick answer

- **ezMaster** — old on-prem controller for the managed EWS/ENS line. EOL.
- **Fit Controller / EPC** — current local controller. Manages **-FIT**, **ECW**,
  and **ECS/ECS-Lite**.
- Controller won't adopt your device? **Cross-flash** it to a supported sibling,
  or **whitelist its model** in the controller DB.

## ezMaster

EnGenius's earlier on-prem controller for the managed EWS/ENS generation.
Effectively end-of-life, displaced by the Fit Controller / EPC.

> ⚠️ Security: at least one 2018-era ezMaster build ships default `admin` /
> `password` and exposes management over cleartext HTTP and telnet. Keep any
> legacy ezMaster off untrusted networks and change defaults immediately.

## Fit Controller / EPC

The modern local controller. Sold as the ARM **FitCon100** appliance, but the
same Docker stack self-hosts on [x86](epc-podman-almalinux.md) or
[ARM](self-hosting-fitcon-arm.md). "Fit Controller" and "EPC" are the same
software across branding changes.

## Device compatibility

Designed for **-FIT** APs (EWS377-FIT, EWS850-FIT), **ECW** cloud APs, and
**ECS/ECS-Lite** switches. Two catches:

- **ECW gained local-controller support only in early 2024** — earlier it was
  cloud-only.
- **ezMaster-era devices aren't natively adoptable.** EWS377APv3, ENS620EXT,
  ENH1350EXT belong to the ezMaster generation; the Fit Controller won't take
  them as-is.

**Whitelist gotcha:** the controller only adopts models on its hardcoded
known-models list, which has odd gaps — e.g. ECS2510/ECS2510FP are missing while
ECS2512/ECS2512FP are present. An unlisted model is refused even if it would work.

## Two workarounds

| Route | What it does | When |
|-------|-------------|------|
| [Cross-flash](../firmware/cross-flashing.md) | Reflash the device to a hardware-equivalent Fit/ECW variant so the controller sees a supported model | Device is on the wrong firmware line ([equivalence table](../hardware/model-equivalence.md)) |
| [Whitelist the model](add-unknown-models.md) | Add the model to the controller DB; device stays untouched | Device works but its model is just missing from the list |
