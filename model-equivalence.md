# Hardware equivalence

Models that share the same board can run each other's firmware — the basis for
[cross-flashing](cross-flashing.md). Same board ≠ identical: enclosure,
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
[cross-flashing](cross-flashing.md).

### ECW230 (v3) ≈ EWS377-FIT ≈ EWS377AP v3

The ECW230v3, EWS377-FIT, and the ezMaster/EnSky-managed **EWS377AP v3** are the
same Wi-Fi 6 4×4 board — cross-flashable siblings. Running EWS377-FIT firmware on
ECW230 or EWS377AP v3 hardware moves it onto a local Fit Controller. Firmware
product IDs: EWS377-FIT `300`, ECW230v3 `284`, EWS377AP v3 `282`
([firmware-format](firmware-format.md)).

Caveat for the EWS377AP v3 → FIT direction: the v3's own web UI (LuCI
`admin/system/flashops`) rejects a FIT `.bin` at upload — its header check gates
on `product_id`/`model`, and `300`/`EWS377-FIT` ≠ the expected `282`/`EWS377APv3`
(rejected inline as *"Firmware upgrade is failed … make sure your firmware is
valid"*). Reaching FIT therefore needs either the vendor conversion package
(`ews377apv3-5.1.4-1.zip`), a forced/patched header check, or a bootloader-level
flash — not a plain web-UI upload. Per the encrypted-payload caveat in
[firmware-format](firmware-format.md), a re-headed image may still
not boot.

### ECW336 — same SoC, **not** the same board

The ECW336 (AX8400 Wi-Fi 6E tri-band) is the counter-example worth studying
before you add anything to the list above. It runs the **same silicon** as the
sibling trio — IPQ8072A, DTS `qcom,msm-id = <0x156 0x185>`, boot log
`CPU: IPQ8072A, SoC Version: 2.0`, identical to an EWS377-FIT — and its Senao
container is byte-identical in construction. Four separate signals say "same
board". All four are wrong:

| signal | what it says | why it's not identity |
|---|---|---|
| DT `model` string | `IPQ807x/AP-HK07` | a stale Senao label; the EWS377-FIT's DTB says the same thing |
| FIT config slot | boots `config@hk07` | both firmwares ship the same ~25 QSDK reference slots (ac01…oak03); each vendor writes its own board content into one |
| `machid` | `0x8010006` on both | selects the slot, not the board |
| `qcom,board_id` | `0x290` on both | drives the BDF name (`bdwlan.b290`), nothing else |

The machine-readable id disagrees with all of them:

```
ECW336:      compatible = "qcom,ipq8074-ap-hk01-c6", "qcom,ipq8074-hk01", …
EWS377-FIT:  compatible = "qcom,ipq8074-ap-hk07"
```

So OpenWrt's `board_detect` resolves an ECW336 to **`ap-hk01-c6`**, and
Qualcomm's own firmware agrees: `firmware_rdp_feature.ini`, shipped inside the
ECW336's own wifi-firmware volume, has a section headed
`#HK01-C6 - Pine RDPs : Pine6G + HK(2G+5G) in HK01 platform` and **no `ap-hk07`
entry at all**. (Two orthogonal reference designs are in play: **Hawkeye** `hk`
is the IPQ807x host board family, **Pine** is the 6 GHz PCIe radio card. The
`C6` is board revision 6 — it has nothing to do with 6 GHz, and there is no
"6E HK01".)

Diffing the two DTBs shows the difference is exactly where it hurts — 670 of
3347 DTS lines, concentrated in:

- ethernet on `dp4`+`dp5`, not `dp5`+`dp6`; `switch_lan_bmp`/`wan_bmp`
  `0x20`/`0x00` instead of `0x00`/`0x40`
- two `ieee802.3-c45` references for an **Aquantia 10G PHY** (the EWS377-FIT
  has zero), whose firmware u-boot CRC-checks and loads from an ECW336-only
  `0:ETHPHYFW` partition before the uplink works at all
- ECW336-only `aqr_int` / `dc_det` / `at_at_det` GPIOs and a `wlan_6g_led`
- a QCN9074 6 GHz radio on PCIe RC0 (`tgt 0x1104`, `qcn9000/bdwlan.ba4`)

**Consequence:** the ECW336 is *not* a cross-flash target for the trio, in
either direction. A re-headed ap-hk07 image would give it the wrong ethernet
topology and no Aquantia PHY — an AP that boots with no uplink, on hardware
whose only copy of its MAC is in the bootloader env. SoC-level work (NSS/EDMA
offload, the shared clock tree) *does* carry across; board DTS does not.

| | sibling trio | ECW336 |
|--|---|---|
| SoC | IPQ8072A | IPQ8072A (same) |
| board (`compatible`) | `ap-hk07` | **`ap-hk01-c6`** |
| header `product_id` | 282 / 284 / 300 | **291** |
| serial [model code](model-codes.md) | X44 / X42 / X45 | **E4T** |
| radios | 2 (2.4 + 5) | 3 (2.4 + 5 + **6 GHz** QCN9074) |
| uplink | — | Aquantia 10G PHY + `0:ETHPHYFW` |
| cross-flashable with the trio | yes | **no** |

## Adding to this list

Equivalence is usually inferred from a shared FCC ID plus matching CPU/WiFi/RAM/
flash on a root shell. Verify before flashing — a wrong guess bricks hardware.

**From firmware, read the DTB's `compatible` and nothing else.** As the ECW336
shows, the DT `model` string, the FIT config slot name, `machid` and
`qcom,board_id` can all four match between genuinely different boards. If you
have both images, diff the DTBs and look specifically at the ethernet
(`dp<n>` ports, `switch_lan_bmp`/`wan_bmp`), the PHY references, and the PCIe
radio nodes — those are what a wrong guess actually breaks, and they are the
hardest to spot after the fact because a mismatched unit often boots fine and
simply has no network.
