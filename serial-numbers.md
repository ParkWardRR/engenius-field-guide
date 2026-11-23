# Serial numbers and the Code27 check character

Decode or generate an EnGenius serial. The embedded model code (positions 5–7) is
what the controller keys on, and the last char is a checksum.

## Format

```
Y Y M  F  M M M  S S S S  C
└─3─┘  └  └─3─┘  └──4──┘  └ check
 date  factory   sequence
       digit
```

- Positions 1–3: date — two digits of the year, then the month as `1`–`9`
  then `A`/`B`/`C` for October–December. `2290…` is 2022-09, `26A0…` is
  2026-10.
- Position 4: factory digit, `0` on everything seen so far.
- Positions 5–7 (`MMM`): the 3-char [model code](model-codes.md).
- Positions 8–11: the unit's sequence number, as 4 **base-27 digits** in the
  Code27 alphabet below (most significant first). `111D` is sequence 1, because
  index 0 is the character `1`.
- Position 12 (`C`): Code27 check char over the first 11 characters.

Validation only ever depends on the last character, so **a serial can satisfy
the check and still not be factory-shaped**. The grammar above is the vendor's
own, lifted from `/usr/sbin/snconverter.sh` in the stock firmware (below), and
it is what to generate when you want a serial that survives scrutiny. One useful
side effect: the Code27 alphabet omits `0`, `A`, `B`, `I`, `O`, `S`, `U`, `Y`
and `Z`, so any of those in positions 8–11 means the sequence field was not
machine-generated.

**Do not assume the model code starts with `X`**: the EWS377/ECW230 line's `X4n`
codes make that look like a rule, while the ECW336's is `E4T` and the switches
use `R__`/`G__`. Match on positions, not on a prefix.

One Code27 alphabet covers the whole line: the ECW336's own serial validates
under exactly this math, so there is no per-series variant to hunt for.

## Where it lives on the device

On the cloud/FIT firmware the serial the controller sees is the u-boot env
**`snextra`** (`setconfig` field 19), not `sn` — `sn` is frequently a
placeholder like `000000001`. `snextra` is a 20-char field; FIT units store the
12-char serial followed by eight `*`, while one ECW336's env held the bare
12-char value with the padding unconfirmed, so read the model code by position
rather than requiring a fixed length.

A second copy lives in the `cert` partition, as an `SN/MAC/HWID` record that
stock firmware's `uboot_protect.sh` syncs against the env at every boot. Worth
knowing both when recovering a wiped env and when re-stamping a serial on
purpose, since that restore path can silently undo one — conditions and limits
in [ecw-private-cloud-mode](ecw-private-cloud-mode.md). Nothing *on* the device
parses positions 5–7; the firmware treats the serial as an opaque string and it
is the controller that derives the model from the code (EPC does it with
`Code27Model`).

Treat a serial as **credential material, not just an identifier**: on the ECW
line the controller pre-shared key is derived from the unit's MAC and serial
together ([ecw-private-cloud-mode](ecw-private-cloud-mode.md)). A `printenv`
paste that includes both is a key disclosure.

## Code27 check

1. Sum the ASCII value of each of the 11 preceding chars.
2. Take `sum mod 27`.
3. Map that index (0–26) through the fixed table:

| idx | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 | 11 | 12 | 13 |
|-----|---|---|---|---|---|---|---|---|---|---|----|----|----|----|
| char| 1 | D | K | 3 | R | 5 | F | P | W | M | E  | 4  | 7  | G  |
| **idx** | **14** | **15** | **16** | **17** | **18** | **19** | **20** | **21** | **22** | **23** | **24** | **25** | **26** | |
| **char**| T | X | 8 | V | L | 2 | J | 6 | C | 9 | N | Q | H | |

## Generate one

```python
import random, string

CODE27 = "1DK3R5FPWME47GTX8VL2J6C9NQH"          # index 0..26 -> char

def check_char(s: str) -> str:                   # s = first 11 chars
    return CODE27[sum(map(ord, s)) % 27]

def make_serial(model_code: str) -> str:
    alnum = string.digits + string.ascii_uppercase
    body = "".join(random.choice(alnum) for _ in range(4)) \
         + model_code \
         + "".join(random.choice(alnum) for _ in range(4))
    return body + check_char(body)

print(make_serial("X45"))   # e.g. an EWS377-FIT serial
```

That generator is deliberately loose — random alnum in the free positions. For
something factory-shaped, encode the sequence properly:

```python
def seq4(n: int) -> str:                 # sequence -> 4 base-27 digits
    n %= 27 ** 4                         # 531441; the vendor wraps, not errors
    return "".join(CODE27[(n // p) % 27] for p in (19683, 729, 27, 1))

def make_factory_serial(date3: str, model_code: str, n: int, factory="0") -> str:
    body = date3 + factory + model_code + seq4(n)
    return body + check_char(body)

print(make_factory_serial("26A", "X45", 1))   # 26A0X45111DR
```

## The vendor's own converter

Stock firmware ships `/usr/sbin/snconverter.sh`, a 168-line `sh` script that
re-encodes a **legacy 9-char serial** into the 12-char format above:

```sh
snconverter.sh <old_sn> <model_name>     # e.g. 228001234 EWS377APv3 -> 2280X441DL2L
```

The old format was `YY` + month + a 6-digit decimal sequence. The script keeps
the 3 date chars verbatim, inserts the hardcoded factory digit `0`, maps the
model *name* to a code through a `case` table, re-encodes the decimal sequence
as 4 base-27 digits after `% 531441`, and appends the check char. It is the
authoritative source for everything in [Format](#format) above.

Two things to know before relying on it:

- **It is frozen SDK content, not per-series.** Byte-identical in a Cloud6E
  ECW336 1.8.118-12 image and a Fit6 EWS377-FIT 1.1.65-2 image, and its table
  covers only the ten ezMaster-era models that needed migrating — see
  [model-codes](model-codes.md) for the two codes whose names have since moved.
  A modern model name (`ECW336`, `EWS377-FIT`, `ECW230v3`) returns `XXX`.
- **Nothing calls it.** No file in the extracted rootfs, text or binary,
  references it; it is a factory/RMA utility. It also writes nothing — two argv
  values in, one line on stdout.

Its live companion is `/usr/sbin/sncheck.sh`, which is the device's own
validator: given no argument it reads `setconfig -g 19`, splits off the 12th
character and echoes `OK` or `NG`. Handy as a read-only self-consistency check
over UART after writing the field.

Worth knowing for migrations: **nothing on the device parses positions 5–7.**
All 14 consumers of `setconfig -g 19` in the stock rootfs — portal, hostapd
recovery, cloud connect, log filenames — treat the serial as an opaque string.
It is the controller that derives the model from the code (EPC does it with
`Code27Model` at inventory registration).

## Migration serials

To move a unit between [hardware-equivalent](model-equivalence.md) siblings,
**keep the unit's own date, factory digit and sequence, and swap only the
3-character model code** — then recompute the check character. The controller
decides the model from characters 5–7, so those three characters are the whole
difference.

| unit | code | EPC `models` name |
|---|---|---|
| EWS377AP v3 | `X44` | `EWS377APv3` |
| EWS377-FIT | `X45` | `EWS377-FIT` |
| ECW230v3 | `X42` | `ECW230v3` |

There is **no ECW230-FIT code** in the EPC table, so an ECW230v3 stays `X42`
after a FIT flash. An EWS377AP v3 becomes an EWS377-FIT only if those three
characters change `X44` → `X45`. An EWS377-FIT is already `X45`.

One sequence (1) on one lab date (2026-10), across the three identities:

| identity | serial |
|---|---|
| EWS377AP v3 | `26A0X44111D3` |
| EWS377-FIT | `26A0X45111DR` |
| ECW230v3 | `26A0X42111DD` |

Same date, same sequence; only the model code and the check character differ. On
a FIT unit the u-boot field is 20 chars — the serial plus eight `*`, e.g.
`26A0X45111DR********`.

> **Rewrite, don't mint.** Minting a fresh serial is only for a unit whose
> `snextra` is missing. If the unit still has one, read it and swap the three
> characters in place. A second serial for the same sequence and model code is a
> second device as far as the controller is concerned, and the old registration
> will not follow the hardware. Save `printenv` before writing the field.

### With `quarry`

```sh
quarry serial  --model X45 --prefix 26A0 --suffix 111D   # -> 26A0X45111DR
quarry snextra --model X45 --prefix 26A0 --suffix 111D   # -> 26A0X45111DR********
```

`--suffix` passes the sequence field through verbatim, which is what a migration
wants: copy the unit's own four characters across. To *number* a new unit
instead, `--seq N` encodes the base-27 digits for you (`--seq 1` → `111D`),
wrapping at 27⁴ like the vendor does. `snextra` emits the complete serial plus
eight `*`, the shape a FIT unit's field 19 actually holds.

> Both commands used to fall short here and the surrounding docs described that:
> `--suffix` defaulted to `0001`, which no factory serial can contain, and
> `snextra` zero-padded `prefix + code` to 20 characters with no serial or check
> character inside it (`sncheck.sh` on the device reads such a value as `NG`).
> Fixed in `quarry` / `pelegrun` as of 2026-11-23 — if you are on an older
> build, check what it emits before writing it to an env.
