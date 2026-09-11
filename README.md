# DALY smart BMS — UART protocol reference

Reverse-engineered reference for the UART interface of a DALY smart BMS, covering both
protocols the board speaks: the well-known 13-byte Daly frames **and** Modbus RTU on the
same wire — the latter is where all the settings, thresholds and the event journal live.

Everything here was read off a real board and cross-checked against the vendor apps.
Where a widely used open implementation disagrees, that is called out explicitly.

---

## ⚠ Applicability

Most of this was taken from **one** board; part was re-checked on a second board of the
same family:

| | Board 1 (most findings) | Board 2 (re-checks, Sept 2026) |
|---|---|---|
| Model | `R24TK1A-8S100A` (8S, 100 A) | `R24TM` (4–8S, 150 A) |
| Hardware string (`0x017F`) | `JHB-R24TK-V2.1` | `JHB-R24TM-V2.2` |
| Software string (`0x0178`) | `70_260316_01T6` | `70_260625_01T4` |
| Balancer | 1 A active | 1 A active |
| Temperature sensors | 4 | 2 |

Vendor apps: "Smart BMS" and "Smart BMS Pro" / "DALY BMS".

On board 2 the `0x81` and `0xD2` maps read back with the same layout, a write through
`0xD2` recalculated derived levels by the §7.3 rules (checked on cell over-voltage only),
and the measurement failure of §9.4 reproduces. Everything else is from board 1 unless
stated.

**DALY register maps differ between models and firmware revisions.** Frame formats and the
Daly command set are fairly universal; the Modbus register addresses in sections 4–8 are
not. Before writing anything, read the registers back and confirm the values match what
the vendor app shows for your board.

## ⚠ Writing is not reversible in the usual sense

Settings land in the EEPROM of the protection board. A wrong address does not fail loudly —
it silently corrupts a neighbouring threshold, and you find out when a protection trips at
a value you never set. Reading is safe and cannot damage anything; writing needs a verified
address.

Two command codes are outright dangerous to probe blind: `0xD9` / `0xDA` write the MOSFET
switches (a zero data byte disconnects the battery), and `0x00F0` over Modbus reboots the
board. Keep any command sweep inside `0x50–0x9F`.

---

## 1. Wiring

The 6-pin UART connector. **Labels are given from the point of view of the external
module, not the BMS** — this is the single most common wiring mistake:

| Pin | Label | Meaning | Connect to |
|-----|-------|---------|------------|
| 1 | GND | ground | host GND |
| 2 | 3.3V | 3.3 V supply | — |
| 3 | 12V | 8–12 V, up to 500 mA | host power (via regulator) |
| 4 | S1 | wake-up button input | — |
| 5 | TXD1 | **BMS transmitter** | host RX |
| 6 | RXD1 | **BMS receiver** | host TX |

3.3 V logic, 9600 8N1, no level shifter needed. Powering the host from pin 3 gives a common
ground, so no isolation is required.

Useful side effect for diagnostics: **a tripped protection and sleep look different from
outside.** On a protection trip the BMS logic keeps running — the 12 V rail stays up, frames
keep coming, only the power path opens. In sleep everything dies, including the 12 V rail.

---

## 2. Two protocols on one line

The board answers **both**:

| Protocol | Address | Notes |
|----------|---------|-------|
| Daly frames | request `0x40`, reply `0x01` | live measurements, 13-byte fixed frames |
| Modbus RTU | request `0x81`, reply `0x51` | full map, `0x0000`–`0x023F` |
| Modbus RTU | request `0xD2`, reply `0xD2` | compact settings map `0x008B`–`0x00A2` (§7.1) |

The two Modbus address spaces **do not overlap**: registers of the `0xD2` map read as
`FFFF` through `0x81` and vice versa. Modbus addresses `0x51` and `0x01` do not answer.

The stock Bluetooth module talks to the board over Modbus, not Daly frames. On the board
tested, that module sits on a **different port** — its traffic is not visible on the UART
connector above, so you cannot sniff the vendor app by tapping these pins. (Verified: a
60-second capture with the app connected and actively writing produced zero bytes.) To see
the app's frames you need an HCI snoop log from the phone.

---

## 3. Daly frame protocol

Fixed 13-byte frames, 9600 8N1:

```
A5 40 <cmd> 08 <8 data bytes> <checksum>
```

`checksum` is the 8-bit sum of bytes 0..11. In a reply the second byte is `0x01`; in a
request it is `0x40`.

> **Check the address byte on receive.** Your own request passes both the start byte and
> the checksum test, so with RX and TX shorted (or a floating line echoing) an echo is
> indistinguishable from a healthy answer.

### Commands

| Cmd | Contents |
|-----|----------|
| `0x90` | pack voltage ×0.1 V, current ×0.1 A **offset by 30000**, SOC ×0.1 % |
| `0x91` | max/min cell voltage (mV) and their cell numbers |
| `0x93` | state (0 idle / 1 charge / 2 discharge), charge MOS, discharge MOS, remaining capacity |
| `0x94` | cell count, sensor count, cycles, **MOSFET temperature** (byte 11, offset −40) |
| `0x95` | cell voltages — series of frames, 3 cells each, frame index in byte 4 |
| `0x96` | temperatures — series of frames, 7 sensors each, value = byte − 40 |
| `0x97` | byte 5 — **number** of the cell being balanced |
| `0x98` | protection flags, 7 bytes |

Other codes that answer but are not decoded here: `0x56`, `0x67`, `0x68`, `0x69`, `0x99`.
Silent on this board: `0x54`, `0x55`, `0x58`, `0x64`, and all of `0x6B`–`0x8F`.

Codes worth knowing: `0x50` capacity and nominal cell voltage, `0x52` lifetime amp-hour
counters through the shunt, `0x53` production date and sleep timeout, `0x59`–`0x5F`
thresholds, `0x60` short-circuit current, `0x6A` serial number, `0x9A` balancer current
in mA.

### Current encoding

`raw − 30000`, in 0.1 A. Do the subtraction in a **signed 32-bit** type: during discharge
the result does not fit in `uint16`.

Note the sign convention is **inverted** in the threshold frames (`0x5B`) relative to the
measurement frame (`0x90`): charge sits at 29500 (i.e. −50 A) and discharge at 31200
(+120 A). Direction is implied by position in the frame, so read the magnitude.

### `0x97` returns a cell number, not a bitmask

**This corrects a bug in the widely used `maland16/daly-bms-uart` implementation**, which
reads byte 5 as a bitmask.

Confirmed three independent ways: the vendor app displayed "balancing cell 8", the raw byte
was `0x08`, and a clamp meter showed the current on cell 8. Read as a bitmask, `0x08` would
mean cell 4.

### Balancer timing

The balancer runs in **bursts of roughly 3 s on, 3 s off** (measured with a clamp meter,
0.7–0.8 A on a 1 A active balancer). If you poll slower than that, the "balancing now" flag
will look like it flickers randomly — it is not a bug, and the cell number is worth latching
for a few seconds so it does not vanish between bursts.

This also means **a single poll is not a coherent snapshot**: cell voltages, the balancing
flag and the balancer current come from three different exchanges, and the burst phase can
change between them. Do not correlate "balancer current" with "cell spread" from one read.

### Frame pacing

The board drops requests that come too fast. With 20 ms between frames, 13 of 20 codes
answered — and the ones that stayed silent were the useful ones. With **120 ms** between
frames, 21 of 21 answered.

The same applies to Modbus block reads: expect an occasional block to go unanswered and
retry it, otherwise a whole page of your UI reads as "parameter does not exist" when in
fact it was simply never delivered.

---

## 4. Modbus RTU

Standard Modbus RTU framing with CRC16. Function `0x03` to read, `0x06` to write one
register, `0x10` to write several.

### 4.1 Live data, `0x0000`–`0x00BF` (address `0x81`)

| Register | Contents | Scale |
|----------|----------|-------|
| `0x0000`–`0x0007` | cell voltages (space reserved to `0x002F`, i.e. 48 cells) | mV |
| `0x0030`–`0x0033` | temperature sensors | °C + 40 |
| `0x0038` / `0x0039` / `0x003A` | pack voltage / current / SOC | 0.1 V / offset 30000, 0.1 A / 0.1 % |
| `0x003C` / `0x003D` | cell count / sensor count | — |
| `0x003E`–`0x0041` | max and min cell voltage with their numbers | mV |
| `0x0042` | cell spread | mV |
| `0x0043`–`0x0046` | max and min temperature with their numbers | °C + 40 |
| `0x0047` | temperature spread | °C |
| `0x0048` | **state**: 0 idle, 1 charge, 2 discharge | — |
| `0x004B` | remaining capacity | 0.1 Ah |
| `0x004D` | balancing in progress (2 = yes) | — |
| `0x004E` | **actual balancer current** | offset 30000, mA |
| `0x004F` | **balancing cell mask**: cell N is `256 << (N-1)` | bitmask |
| `0x0052` / `0x0053` | **actual** charge / discharge MOS position | 0–1 |
| `0x0057` / `0x0058` | mean cell voltage / power | mV / W |
| `0x005A` | MOSFET temperature | °C + 40 |
| `0x0061`–`0x0063` | board clock | one byte per field |
| `0x0072` | forced-closure state (see §8) | 0 or 4 |
| `0x00A9`+`0x00AA` | total lifetime uptime | `u32`, seconds |
| `0x00AB`+`0x00AC` | uptime since power-on | `u32`, seconds |

> **Trap when identifying registers by correlation:** currents are stored with a +30000
> offset, so an idle current register reads 30000, not 0. A filter looking for "non-zero
> only while active" silently skips it. Look for deviation from a reference value.

`0x00AB` is the reliable witness for "did the board actually reboot" — see §8.

### 4.2 Settings, `0x0100`+ (address `0x81`)

| Register | Contents | Scale |
|----------|----------|-------|
| `0x0101` / `0x0106` | cell count / sensor count | — |
| `0x0109`+`0x010A` | nominal capacity | `u32` mAh |
| `0x010B`+`0x010C` | second capacity value, purpose unknown | `u32` mAh |
| `0x010D`–`0x0110` | total charged / total discharged | `u32` mAh |
| `0x0114` | nominal cell voltage | mV |
| `0x0115` | **sleep timeout** (65535 = never sleep) — see below | 10 s |
| `0x0116` | **current SOC** (writable — this calibrates the counter) | 0.1 % |
| `0x0117` / `0x0119` | balancer current setting / balancing enabled | mA / 0–1 |
| `0x011A` / `0x011B` | balance start voltage / start delta (`0x011A` mirrored at `0x022A`) | mV |
| `0x0121` / `0x0122` | charge / discharge switch **permission** | 0–1 |
| `0x0123`–`0x0125` | board clock (second mirror at `0x00D4`) | one byte per field |
| `0x0126`–`0x0128` | **password**, 6 ASCII chars, plaintext | — |
| `0x0129`–`0x012A` | production date | YY MM DD |
| `0x012E` | forced closure: write 1 to arm, read for seconds left | s |
| `0x0130`+ | thresholds, 4 registers each (see §4.3) | |
| `0x0140` / `0x0145` | charge / discharge current — **5 registers**: L1, L2, delay, L3, delay | offset 30000, 0.1 A |
| `0x014A` / `0x014E` / `0x0152` / `0x0156` | charge and discharge temperature limits | °C + 40 |
| `0x015A` / `0x015E` | cell spread / temperature spread | mV / °C, **no offset** |
| `0x0162` | SOC thresholds | 0.1 % |
| `0x0166` | MOSFET over-temperature | °C + 40 |
| `0x016E` | short-circuit trip current | A |
| `0x016F` | **zero-drift current** — dead band of the current reading | 0.01 A |
| `0x0174` | vendor-app **heartbeat** — see below | — |
| `0x0178` / `0x017F` / `0x018D` / `0x0199` | software / hardware / serial / battery code | ASCII, 7 registers each |
| `0x01FB` | balance stop voltage | mV |
| `0x0227` / `0x0229` | SOC calibration: "0 %" and "100 %" cell voltage | mV |

There is **no password check before writing.** The password register is readable plaintext
and has no effect on write access.

**The sleep timeout is `0x0115`** (board 2, from an HCI log of the app plus read-back). The
"DALY BMS" app writes the entered number divided by 10 and truncated: 40000 → `4000`,
65535 → `6553`. The board then rewrites `6553` as `65535`, which means "never sleep" — so
reading back 65535 after writing 6553 is success, not a mismatch. The 10-second unit is
inferred from the app; the actual timeout was not timed, because a sleeping board also cuts
the 12 V rail that may be powering your host.

> **Correction.** An earlier revision of this document listed `0x0175` as the sleep timeout.
> That was wrong: `0x0175` stayed at 65535 while the real timeout was changed.

**`0x0174` is a heartbeat of the vendor app's session.** While connected, the app writes
`162` there every 2–6 s. Outside a session the register reads `255`; after the first write
it reads `2`. What `2` means is not known.

### 4.3 Threshold structure

Voltage and temperature thresholds occupy **four consecutive registers**:

```
[warning]  [protection]  [return]  [delay ms]
```

The vendor app shows only the protection value; the other three are visible only here.

The **return** value is the hysteresis point — the level at which a tripped protection
clears. For a low-side threshold it sits *above* the trip point, and for a high-side one
below it.

**Current is the exception** — five registers, and no return value:

```
[L1] [L2] [delay ms] [L3] [delay ms]
```

So current has three levels and two delays, while voltage and temperature have three
levels and one delay.

### 4.4 The second layer

The board maintains a parallel, duplicate set of thresholds. Two different layouts:

| Registers | Layout |
|-----------|--------|
| `0x01C3`–`0x01CD` | pairs: `[protection] [return]`, two registers per parameter |
| `0x01DD`+ | triplets, stride 3: `[500] [protection] [return]` |

Example triplets on board 1: `0x01DD`/`0x01DE`/`0x01DF` = 500 / 25 / 20 for
temperature spread, and `0x01E0`/`0x01E1`/`0x01E2` = 500 / 30 / 150 for low SOC (i.e.
3.0 % trip, 15.0 % return, in 0.1 % units).

**This layer is not decorative — for at least one parameter it is the one that acts.** See
§9.1.

---

## 5. Protection flags (`0x98`)

Seven bytes, index = `byte × 8 + bit`. The general pattern is **even bit = warning, odd bit
= protection**, and the two levels of one parameter sit next to each other.

| Idx | Meaning | Level |
|-----|---------|-------|
| 0 / 1 | cell voltage high — warning / protection | 1 / 2 |
| 2 / 3 | cell voltage low — warning / protection | 1 / 2 |
| 4 / 5 | pack voltage high — warning / protection | 1 / 2 |
| 6 / 7 | pack voltage low — warning / protection | 1 / 2 |
| 8 / 9 | charge temperature high — warning / protection | 1 / 2 |
| 10 / 11 | charge temperature low — warning / protection | 1 / 2 |
| 12 / 13 | discharge temperature high — warning / protection | 1 / 2 |
| 14 / 15 | discharge temperature low — warning / protection | 1 / 2 |
| 16 / 17 | charge current high — warning / protection | 1 / 2 |
| 18 / 19 | discharge current high — warning / protection | 1 / 2 |
| 20 / 21 | SOC high — warning / protection | 1 / 2 |
| 22 / 23 | **SOC low — warning / protection** | 1 / 2 |
| 24 / 25 | cell spread — warning / protection | 1 / 2 |
| 26 / 27 | temperature spread — warning / protection | 1 / 2 |
| 32–39 | MOSFET faults: charge/discharge over-temperature, sensor failure, short, open | 2 |
| 40 | AFE measurement module failure | 2 |
| 41 | voltage measurement module failure | 2 |
| 42 | temperature sensor failure | 2 |
| 43 | EEPROM failure | 2 |
| 44 | real-time clock failure | 2 |
| 45 | pre-charge module failure | 2 |
| 46 | no communication with external controller | 1 |
| 47 | internal BMS communication failure | 2 |
| 48 | current measurement module failure | 2 |
| 49 | total voltage measurement failure | 2 |
| 50 | short-circuit protection tripped | 2 |
| 51 | charge blocked: voltage too low | 1 |
| 52 | **a switch is open by command** — see below | state, not an alarm |

Bits 28–31 and 53–55 were never observed set.

**Bit 52 does not tell you which switch.** It was first read as "charge switch off", and a
second experiment refuted that: switching the *discharge* MOS off raises the *same* bit,
and bit 53 never rises at all. So it means "something was opened by command" and nothing
more. For the actual state read frame `0x93`, or registers `0x0052`/`0x0053`.

Treat bit 52 as a **state**, not an alarm: if you fold it into an "any alarm present" test,
re-enabling a switch will look like "all protections cleared".

**Confidence note.** Byte 0 was confirmed on hardware (the vendor app named exactly bits 1
and 4 when the raw value was `0x12`), byte 6 bit 2 by a live short-circuit trip, and byte 2
bits 6 and 7 by a deliberate deep-discharge experiment. The rest of the naming comes from
the reference open implementation; only the ordering was taken from it, because its byte 1
handling has a typo (all eight flags read bit 1).

---

## 6. Event journal (`0x3000`)

The board keeps an on-board journal of protection events and setting changes. The vendor
app walks it one record per second.

The request looks ordinary. The reply does not:

```
81 03 3001 0001            "give me ONE register at 0x3001"
51 03 85 <133 bytes>       the board returns a WHOLE RECORD
```

**The requested register count is ignored** — you always get 138 bytes
(`51 03 85` + 133 + CRC). If your Modbus buffer is smaller, the reply is truncated, CRC
fails, and it looks exactly like "this register does not exist".

**Ring of 400 records.** Index 1 is the newest, 400 the oldest, 0 is an alias for the
oldest. The ring size is in the record itself (bytes 2–3). Asking for index 1000 gets no
answer.

### Record layout (133 bytes)

Verified against live readings on 24 records — voltage, current, SOC and all cell voltages
matched.

| Offset | Contents | Scale |
|--------|----------|-------|
| 0–1 | position in the ring | changes on every shift; not content |
| 2–3 | ring size | 400 |
| 5–10 | **event time** | YY MM DD hh mm ss, board clock |
| 13–14 | pack voltage | V×10 |
| 15–16 | current | offset 30000, 0.1 A |
| 17–18 | SOC | 0.1 % |
| 20–21 / 23–24 | max / min cell voltage | mV |
| 31–62 | **cell voltages**, space for 16 | mV |
| 95–98 / 99 | four temperatures / MOSFET temperature | °C + 40 |
| 103–104 | **event code**: 0 = alarm, otherwise the changed parameter | — |
| 107–108 | new value of that parameter | parameter units |

**Do not trust the max/min cell numbers** at bytes 19 and 22 — they do not follow the
layout and disagree with the cell array. Use the array.

### Two kinds of record, one ring

Changing a setting writes **one record per changed register**, so a single action in the
app produces a burst of four records with the same timestamp (warning, protection, return
and the second layer).

The practical consequence: **setting changes evict alarms from the same ring.** A session
of tuning can burn dozens of the 400 slots. If the journal matters to you, mirror it to
your own storage.

Because the board clock jumps backwards when the board reboots, store your own timestamp
alongside each record; the event's own time lives inside the record body.

---

## 7. Writing settings

### 7.1 Two dialects

The two vendor apps write differently, and the difference matters:

| App | Address | Style |
|-----|---------|-------|
| "DALY BMS" | `0x81` | writes every level explicitly, in bundles |
| "Smart BMS" | `0xD2` | writes one register per setting |

The `0xD2` map is a compact block of **(warning, protection) pairs**, `0x008B`–`0x00A2`.
Write the protection register and the board derives the rest (§7.2).

| Parameter | Protection | Warning | Encoding |
|-----------|------------|---------|----------|
| Cell over-voltage | `0x008C` | `0x008B` | mV |
| Cell under-voltage | `0x008E` | `0x008D` | mV |
| Pack over-voltage | `0x0090` | `0x008F` | V×10 |
| Pack under-voltage | `0x0092` | `0x0091` | V×10 |
| Charge current | `0x0094` | `0x0093` | 30000 − A×10 |
| Discharge current | `0x0096` | `0x0095` | 30000 + A×10 |
| Charge over-temperature | `0x0098` | `0x0097` | °C + 40 |
| Charge under-temperature | `0x009A` | `0x0099` | °C + 40 |
| Discharge over-temperature | `0x009C` | `0x009B` | °C + 40 |
| Discharge under-temperature | `0x009E` | `0x009D` | °C + 40 |
| Cell spread | `0x00A0` | `0x009F` | mV |
| Temperature spread | `0x00A2` | `0x00A1` | °C, **no offset** |

The map ends at `0x00A2`: `0x00A3`+ read `3000 / 10 / 1 / 1`, which matches neither the SOC
thresholds nor MOSFET over-temperature. SOC, capacity and balancer settings have no `0xD2`
address. The whole table was read back on board 2 and matched the `0x81` view pair for
pair, except the discharge current of the factory-fresh board (§7.5); `0x008C` was found
there by position and value and then confirmed by writing it.

### 7.2 The board recalculates derived levels — but only through `0xD2`

Write a single protection value to the `0xD2` map and the board fills in the warning, the
return value and the second layer by itself:

```
d2 06 008E 0A28      cell low = 2600
   → next block read shows 0A5A 0A28: warning became 2650 (protection + 50) by itself

d2 06 0094 73F0      charge current = 32.0 A
   → next block read shows 7430 73F0: L1 became 25.6 A (0.8 × 32) by itself
```

**Through `0x81` this does not happen.** Verified: writing the low-SOC protection at
`0x0163` from 30 to 0 via `0x81` left the second layer `0x01E1` at 30, permanently.

So "just write the protection value" is a property of the `0xD2` path, not of the board.
For parameters that have no address in the `0xD2` map — SOC, capacity, balancer settings —
you must write the derived registers yourself.

### 7.3 Recalculation rules

| Parameter | Warning | Second layer |
|-----------|---------|--------------|
| Cell, low | protection **+50 mV** | protection−50 / protection |
| Cell, high | protection **−50 mV** | protection+50 / protection |
| Pack, low | protection **+0.8 V** | protection−0.8 / protection |
| Pack, high | protection **−0.8 V** | protection+0.4 / protection−0.4 |
| Currents | **0.8 × protection** | no second layer; `L3` = **1.2 × protection** |
| Temperature spread | unchanged | unchanged; return = protection − 1 |

Re-checked on board 2 for cell over-voltage: writing 3650 through `0xD2` gave warning and
return 3600 and second layer 3700 / 3650.

**The "DALY BMS" app uses different constants** — it sets `L3` to `⌊4/3 × protection⌋` and
the low-cell warning to `protection + 100`. Its writes therefore overwrite the board's own
arithmetic, and afterwards the derived values are not where the board would have put them.

Consequence worth internalising: **"changed it and changed it back" is not a no-op.** The
app shows only protection values, so a round trip can leave a dozen derived registers
shifted while the UI looks untouched. Writing the same protection value once through `0xD2`
restores the board's arithmetic.

### 7.4 The `0x10` frame from the vendor app is non-standard

```
81 10 01 40 00 02 74 30 73 F0 2C A9
└adr└fn└─reg─┘└─cnt─┘└──data───┘└CRC┘     canonical Modbus would carry a byte count (04) here
```

There is no byte-count field. If you parse app traffic with a strict Modbus decoder, these
frames will not decode.

### 7.5 Echo is not a success criterion

Write acknowledgement is inconsistent, and it depends on the register:

| Register | Parameter | Echoed | Value actually applied |
|----------|-----------|--------|------------------------|
| `0xD2` `0x0094` | charge current | 8 of 8 | yes |
| `0xD2` `0x0090` | pack over-voltage | **0 of 8** | yes |
| `0x81` `0x011B` | balance start delta | 3 of 3 | yes |

Both are on the same address, both trigger recalculation, both take effect. No explanation
was found. **Judge success by reading the register back**, and treat the echo as
informational only.

Two caveats on reading back:

- **Do not read back immediately.** After a write via `0xD2`, the view through `0x81` kept
  returning stale values for up to two minutes before catching up on its own.
- **Exact comparison gives false negatives on live values.** Writing an SOC of 16 % while a
  67 A discharge was running read back as 15.9 % — the counter ticked between the write and
  the check. The write was fine.
- **A factory-fresh board can disagree with itself.** Board 2 out of the box read discharge
  current as 96 / 120 / 144 A through `0x81` but 120 / 150 A through `0xD2`; the two views
  converged only after the first round of setting changes. Cross-check both views on a new
  board before trusting either.

---

## 8. Control commands

| Action | How |
|--------|-----|
| Charge MOSFET | Daly command `0xDA`, data byte 1 = on / 0 = off |
| Discharge MOSFET | Daly command `0xD9`, same |
| Balancing enable | Modbus write to `0x0119`, 0 or 1 |
| Board reboot | Modbus write to `0x00F0`, **any value** |
| Forced closure | Modbus write `1` to `0x012E` |

### 8.1 Board reboot

Both vendor apps send a write to `0x00F0`; the value appears irrelevant (`0x0000` and
`0x0001` both work) — the write itself is the trigger.

**There is no echo**, because the board reboots before it can answer. Every other write is
acknowledged, so the silence here is easy to misread as failure. Judge by the consequence:
read the uptime counter `0x00AB` before and after. On the test board it went `961 → 0`.

A reboot clears a latched **warning** flag. It does **not** clear a latched current
protection — that one opens the switches and needs a charger, a B− to P− jumper, or a
physical reconnect.

All 576 registers were compared before and after a reboot: no threshold changed.

There is no Daly `0x00` "reset" command on this firmware — the uptime counter did not move
after sending it.

### 8.2 Forced closure (bypassing protections)

The vendor app has an "emergency forced start" switch that closes both MOSFETs regardless
of protection state. It is a **single register write**, captured from the app's own traffic:

```
81 06 01 2E 00 01 36 3F        function 0x06, register 0x012E, value 1
```

`0x012E` is dual-purpose: **write 1 to arm, read it for the seconds remaining.** The
observed hold time is about 300 s, counting down one per second. `0x0072` becomes `4` while
the mode is armed and returns to `0` when it lapses.

Three things worth knowing before you use it:

1. **There is no disarm command.** Writing again *extends* the timer instead of clearing it —
   the countdown jumps back to ~300. This is why pressing the app's switch a second time
   appears to do nothing.
2. **A MOSFET command does not clear it either.** Verified directly: armed, then sent a
   switch-enable command at 282 s remaining; the countdown continued undisturbed.
3. **It overrides the permission registers.** While armed you can end up with
   `0x0121`/`0x0122` = 0/0 (permission withdrawn) and `0x0052`/`0x0053` = 1/1 (switches
   physically closed). That divergence is the reliable "protection is bypassed" indicator —
   and it is also a trap: **when the timer lapses, the board applies the permission and
   opens both switches**, disconnecting the battery.

---

## 9. Behaviour observed on this firmware

This section is about how the board *acts*, not about the wire format. It is more likely
than the rest of this document to differ on other firmware revisions.

### 9.1 The low-SOC protection is advisory — it does not open the switches

Tested by falsifying the SOC counter (writing `0x0116`) while a real 67–69 A discharge ran,
across five threshold configurations:

| `0x0163` (layer 1) | `0x01E1` (layer 2) | Flag 23 raised at |
|--------------------|--------------------|-------------------|
| 0 | 30 (3.0 %) | 3.0 % — twice, in two independent runs |
| 0 | 2 (0.2 %) | passed 3.0 % silently, raised at 0.2 % |
| 0 | 0 | passed 0.2 and 0.1 silently, raised at 0.0 % |
| 20 (2.0 %) | 0 | 2.0 % |
| 20 (2.0 %) | 20 (2.0 %) | 2.0 % |

Findings:

- **The acting threshold lives in the second layer `0x01E1`**, not in `0x0163`. Zeroing the
  first layer alone changes nothing — and, because the `0x81` path does not recalculate
  (§7.2), that is exactly the trap it creates.
- **Both layers raise the same flag**, each at its own threshold; whichever is higher wins.
- **The comparison is non-strict**: the flag rose exactly *at* the threshold value in all
  five configurations, never below it.
- **Zero does not disable it** — it simply moves the trip to 0.0 %.
- **In none of the six trips did either MOSFET open.** The discharge ran through the
  threshold uninterrupted, including with the counter sitting at 0.0 %.

The protection also **latches**: lowering the threshold below the current SOC does not clear
it. It clears only when the SOC rises past the *return* threshold — and that return is
**strict**, where the trip is not: at exactly 15.0 % the latch still held, and released at
15.1 %.

Practical consequence: on this firmware, discharge depth is bounded by the pack/cell voltage
protections and by whatever the inverter does, **not** by the SOC setting.

### 9.2 SOC self-calibration

The counter snaps to 100 % when the **highest cell** reaches the `0x0229` threshold **while
a charge current is flowing**. Both conditions are required — this is why it never triggers
in a float/buffer regime no matter how the voltage threshold is set.

Caught live at 30-second resolution after the counter had been deliberately falsified:

```
23:37:37   SOC  47.3 %   27.60 V
23:38:07   SOC 100.0 %   27.90 V
```

Remaining capacity jumped to the full nominal value in the same step — the value is
assigned, not accumulated.

**Set the threshold high enough.** With `0x0229` at 3450 mV the latch fired while the pack
still held 247 Ah of 280 — under a 25 A charge the top cell runs tens of millivolts ahead of
the average and reaches a low threshold long before the pack is full. Calibrate at a low
current: the top-cell offset was within 13 mV at 10 A and up to 97 mV at 25 A.

### 9.3 The SOC counter drifts down at rest

On the test board the "total discharged" counter (`0x010F`) increments by 1 mAh every 128 s
with the pack at rest and the current reading at 0.0 A — a phantom **28 mA**, about
0.68 Ah/day. Independently confirmed over a 23.7-hour window.

The zero-drift dead band (`0x016F`, 0.5 A here) does **not** suppress it: the dead band
lives in the instantaneous current path, while the SOC integrator has its own offset. No
current-calibration register was found anywhere in `0x0000`–`0x023F`.

The counter is also not trustworthy near the bottom under heavy load — in the middle of the
range it is coulomb-accurate (98.9 → 84.4 % matched 40.6 Ah of 280 exactly), but close to
empty and under high current it collapses early.

### 9.4 Measurement failure under charge

Worth documenting because it is easy to misdiagnose as a wiring or inverter problem.

Under charge, this board intermittently loses communication on **both** interfaces, raises
flag 41 ("voltage measurement module failure") and **opens the charge MOSFET**. During the
dropout all eight cell voltages and the pack voltage read as zero while current is still
flowing — the measurement path fails, the power path does not.

- **It self-sustains.** After a trip the board re-closes the charge switch after ~50 s and
  the fault returns ~15 s later. Currents sampled *inside* that cycle say nothing about the
  current at which the fault occurs.
- **There is no stable current threshold.** Within one session the boundary is sharp; across
  sessions it moves (clean at 20 A one day, failing at 20 A the next, clean for seven hours
  at 19 A a third time).
- **Discharge does not trigger it** — three hours at 67–69 A produced one failed read; the
  first charge attempt dropped the link within 12 seconds.
- **A voltage spike on the charger output is a consequence, not a cause**: the board opens
  the switch, the charger loses its load, and its output jumps. The pack is behind the open
  MOSFET and does not see it.
- **The balancer is not involved** — reproduced with balancing disabled.
- **It is not a defect of one board.** Replacing board 1 with board 2 left the behaviour
  unchanged at charge currents above 15 A. What the two boards share is the DALY measurement
  front end and firmware base — and the charger.

Root cause not established.

### 9.5 A successful read is not a sane read

While flag 41 is raised, the board returns a **formally valid** Modbus reply in which the
pack voltage is `0.00 V`. A watcher that only checks "did the exchange succeed" will happily
ingest that as a measurement, and — if it also reads switch states from the same frame —
invent switch-opening events that never happened.

Check the transport, then sanity-check the values.

---

## 10. Gotchas, condensed

- Verify the reply address byte; an echo passes the checksum test.
- Space frames ~120 ms apart; retry unanswered Modbus blocks.
- Currents carry a +30000 offset; subtract in signed 32-bit.
- `0x97` is a cell **number**, not a mask.
- One poll is not one instant — the balancer bursts faster than a typical poll cycle.
- The journal reply ignores your register count and returns 138 bytes.
- Writes may not echo; verify by reading back, but not immediately.
- Derived levels are recalculated only on the `0xD2` path.
- "Changed it and changed it back" leaves derived registers shifted.
- A valid reply can still contain impossible values.
- Setting changes and alarms share one 400-record ring.
- The sleep timeout is `0x0115` in 10 s units, not `0x0175`; "never" reads back as 65535.
- On a fresh board, cross-check the `0x81` view against `0xD2`.

---

## Contributing

Corrections and data from other DALY models are welcome — especially register maps from
boards with a different hardware string, since that is where this document is weakest.
Please state the model, the hardware string (`0x017F`) and the software string (`0x0178`)
with any addition.

## License

Documentation released under CC0 / public domain. Use it, copy it, no attribution required.

The reference implementation cross-checked against is
[`maland16/daly-bms-uart`](https://github.com/maland16/daly-bms-uart); the `0x97` and byte-1
notes above are corrections to it, offered in that spirit.
