# Serial numbers and the Code27 check character

Decode or generate an EnGenius serial. The embedded model code (positions 5–7) is
what the controller keys on, and the last char is a checksum.

## Format

```
X X X X  M M M  Y Y Y Y  C
└──4──┘  └─3─┘  └──4──┘  └ check
 free    model   free
```

- Positions 1–4, 8–11: any of `0-9 A-Z`.
- Positions 5–7 (`MMM`): the 3-char [model code](model-codes.md).
- Position 12 (`C`): Code27 check char over the first 11 characters.

The leading 4 are often a date-ish code (`YYWW`), but nothing depends on that —
treat them as free. And **do not assume the model code starts with `X`**: the
EWS377/ECW230 line's `X4n` codes make that look like a rule, while the ECW336's
is `E4T` and the switches use `R__`/`G__`. Match on positions, not on a prefix.

One Code27 alphabet covers the whole line: the ECW336's own serial validates
under exactly this math, so there is no per-series variant to hunt for.

## Where it lives on the device

On the cloud/FIT firmware the serial the controller sees is the u-boot env
**`snextra`** (`setconfig` field 19), not `sn` — `sn` is frequently a
placeholder like `000000001`. `snextra` is a 20-char field; FIT units store the
12-char serial followed by eight `*`, while one ECW336's env held the bare
12-char value with the padding unconfirmed, so read the model code by position
rather than requiring a fixed length.

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
