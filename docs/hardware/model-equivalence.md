# Hardware equivalence

Models that share the same board can run each other's firmware — the basis for
[cross-flashing](../firmware/cross-flashing.md). Same board ≠ identical: enclosure,
ports, PoE, and per-device partitions can differ.

## Known equivalent groups

### ENS620EXT ≈ ENH1350EXT ≈ ECW160

All "11ac Wave 2 MU-MIMO Dual-Band AC1300 outdoor AP" (867 Mbps 5 GHz / 400 Mbps
2.4 GHz), same CPU, WiFi chips, RAM, and flash. ENH1350EXT and ECW160 share an
FCC ID.

| | ENS620EXT | ENH1350EXT / ECW160 |
|--|-----------|---------------------|
| Ethernet ports | 2 | 1 |
| PoE | proprietary 24V | standard 802.3at |
| Rating | IP55 | IP67 |
| `cert` MTD partition | no | yes (ECW160) |

Practical use: an ENS620EXT can be moved toward the ECW160 (a Fit-Controller line)
— but the missing `cert` partition means firmware alone won't register it; see
[cross-flashing](../firmware/cross-flashing.md).

### ECW230 (v3) ≈ EWS377-FIT

The ECW230v3 and EWS377-FIT are cross-flashable siblings — e.g. run EWS377-FIT
firmware on ECW230 hardware to use it with a local Fit Controller. Firmware
product IDs: EWS377-FIT `300`, ECW230v3 `284`
([firmware-format](../firmware/firmware-format.md)).

## Adding to this list

Equivalence is usually inferred from a shared FCC ID plus matching CPU/WiFi/RAM/
flash on a root shell. Verify before flashing — a wrong guess bricks hardware.
