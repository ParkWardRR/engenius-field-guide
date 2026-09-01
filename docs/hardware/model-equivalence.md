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

### ECW230 (v3) ≈ EWS377-FIT ≈ EWS377AP v3

The ECW230v3, EWS377-FIT, and the ezMaster/EnSky-managed **EWS377AP v3** are the
same Wi-Fi 6 4×4 board — cross-flashable siblings. Running EWS377-FIT firmware on
ECW230 or EWS377AP v3 hardware moves it onto a local Fit Controller. Firmware
product IDs: EWS377-FIT `300`, ECW230v3 `284`, EWS377AP v3 `282`
([firmware-format](../firmware/firmware-format.md)).

Caveat for the EWS377AP v3 → FIT direction: the v3's own web UI (LuCI
`admin/system/flashops`) rejects a FIT `.bin` at upload — its header check gates
on `product_id`/`model`, and `300`/`EWS377-FIT` ≠ the expected `282`/`EWS377APv3`
(rejected inline as *"Firmware upgrade is failed … make sure your firmware is
valid"*). Reaching FIT therefore needs either the vendor conversion package
(`ews377apv3-5.1.4-1.zip`), a forced/patched header check, or a bootloader-level
flash — not a plain web-UI upload. Per the encrypted-payload caveat in
[firmware-format](../firmware/firmware-format.md), a re-headed image may still
not boot.

## Adding to this list

Equivalence is usually inferred from a shared FCC ID plus matching CPU/WiFi/RAM/
flash on a root shell. Verify before flashing — a wrong guess bricks hardware.
