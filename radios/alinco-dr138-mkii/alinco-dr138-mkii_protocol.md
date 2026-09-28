# Alinco DR-138 MKII — Protocol Reference

Raw values and bytes only. Status: `confirmed` / `unconfirmed` / `unknown` / `disputed` (two
independently-verified captures disagree, neither yet resolved as the error).

## Serial

9600 8N1, no flow control. ERW-7 cable (CP2102/FTDI). Half-duplex, every command echoed before
the real reply. 16KB image. Channels: 32 bytes each, 200 of them, base `0x2000`.

## Wire frames

| Step | Bytes | Notes |
|---|---|---|
| Handshake (PC->Radio) | `50 52 4f 47 52 41 4d` (`"PROGRAM"`) | 7 bytes |
| Handshake echo (Radio->PC) | `50 52 4f 47 52 41 4d` | same 7 bytes |
| Ready (Radio->PC) | `51 58 06` | unprompted |
| ID query (PC->Radio) | `02` | 1 byte |
| ID reply (Radio->PC) | `02` + model[7] + `00` | e.g. `02 49 44 4a 2d 31 33 38 00` = `IDJ-138` |
| Version reply (Radio->PC) | `05` + `V100` + `e0 00` + `06` | e.g. `05 56 31 30 30 e0 00 06` |
| Read frame (PC->Radio) | `52` addr_hi addr_lo `10` | 4 bytes |
| Read frame echo (Radio->PC) | `52` addr_hi addr_lo `10` | same 4 bytes |
| Read reply (Radio->PC) | `57` addr_hi addr_lo `10` + data[16] + checksum | 21 bytes |
| Read ack (Radio->PC) | `06` | separate byte |
| Write frame (PC->Radio) | `57` addr_hi addr_lo `10` + data[16] + checksum + `06` | 22 bytes — trailing `06` is MKII-specific, not present on reads, do not assume it applies to other Alinco models |
| Write frame echo (Radio->PC) | `57` addr_hi addr_lo `10` + data[16] + checksum + `06` | same 22 bytes |
| Write ack (Radio->PC) | `06` | separate byte |
| End (PC->Radio) | `45 4e 44` (`"END"`) | 3 bytes |
| End echo (Radio->PC) | `45 4e 44 06` | echo + ack |
| Final ack (Radio->PC) | `06` | |

`checksum = (addr_hi + addr_lo + length + sum(data)) & 0xFF`. Verified against 1008/1008 blocks
of a full factory-default write capture, 0 mismatches.

Mandatory: the first command after the version reply, every session, must be a read of `0x0040`
(16 bytes, all zero on the unit tested). Skipping it doesn't break reads/writes, but the radio
never auto-resets after a write — needs a manual power-cycle for the write to take effect.

Reads: `0x0010`-`0x3ff0`. Writes: `0x0100`-`0x3ff0` only (`0x0010`-`0x00ff` read-only). Every write
blanks `0x3900`-`0x3ff0` to zero and `0x0160` to `0xff`, regardless of what changed.

## Memory map

Grouped by vendor-software dialog, not by address — some dialogs (DTMF in particular) store
settings in byte ranges that otherwise belong to a different page.

### Identification (read-only)

| Offset | Field | Value | Status |
|---|---|---|---|
| `0x0000`-`0x000f` | unattributed, before the readable range starts | no data captured | unknown |
| `0x0010` | model string, length unconfirmed | ascii, e.g. `DJ-138` | confirmed |
| `0x0011`-`0x002f` | unattributed | no data captured | unknown |
| `0x0030` | date string, length unconfirmed | ascii, e.g. `2024/11/26` | confirmed |
| `0x0031`-`0x00ff` | unattributed | no data captured | unknown |

### Channel

Channel table: 200 x 32 bytes, channel N = `0x2000 + N*0x20`. Offsets below relative to a
channel's start (`+0xNN`) unless noted. Tested on channel 0.

| Offset | Field | Value | Status |
|---|---|---|---|
| `0x0100`-`0x011f` | used-flag bitmap, 1 bit/channel, bit N = channel N | `0`=used<br>`1`=empty. Factory: `fe` `ff`x31 | confirmed |
| `0x0120`-`0x013f` | skip-flag bitmap, same shape, matches "Skip" checkbox | `0`=not skipped<br>`1`=skipped. Factory: `fe` `ff`x31 | confirmed |
| `0x0140`-`0x015f` | unattributed | all `ff` | unknown |
| `0x0160` | unattributed, forced to `ff` on every write regardless of what's sent | `ff` | unknown |
| `0x0161`-`0x016f` | unattributed | all `ff` | unknown |
| `0x0170`-`0x01ff` | unattributed | all `ff` | unknown |
| `+0x00`-`+0x03` | RX Frequency: byte0 leading digit (raw), bytes1-3 BCD pairs, x100Hz | e.g. `01 44 50 00` = 144.50000. Factory `01 45 00 00` = 145.00000 | confirmed |
| `+0x04` | unattributed, constant | `00` | unknown |
| `+0x05`-`+0x07` | duplex offset magnitude (TX freq), 3 bytes BCD pairs x100Hz, kept even in simplex | e.g. `02 62 50` = 2.625MHz. Factory `00 60 00` = 0.6MHz | confirmed |
| `+0x08` | Step, 0-based index | `00`=2.5K `01`=5K(factory) `02`=6.25K `03`=8.33K `04`=10K `05`=12.5K `06`=20K `07`=25K `08`=30K `09`=50K | confirmed |
| `+0x09` bit0 | TX Off | `0`=off `1`=on | confirmed |
| `+0x09` bit1 | Reverse | `0`=off `1`=on | confirmed |
| `+0x09` bits2-3 (`0x0c`) | Channel Spacing | `00`=25KHz(factory) `04`=20KHz `08`=12.5KHz | confirmed |
| `+0x09` bit4 (`0x10`) | Encode DCS polarity | `0`=Normal `1`=Reverse | confirmed |
| `+0x09` bit5 (`0x20`) | Encode DCS carry (code+256) | `0`=<=255 `1`=+256 | confirmed |
| `+0x0a` bit0 (`0x01`) | Duplex direction | `0`=TX below RX `1`=TX above RX | confirmed |
| `+0x0a` bit1 (`0x02`) | unattributed, constant `1` in samples | `1` | unknown |
| `+0x0a` bits2-3 (`0x0c`) | TX Power | `00`=High(factory) `04`=Mid `08`=Low | confirmed |
| `+0x0a` bit4 (`0x10`) | Decode DCS polarity | `0`=Normal `1`=Reverse | confirmed |
| `+0x0a` bit5 (`0x20`) | Decode DCS carry (code+256) | `0`=<=255 `1`=+256 | confirmed |
| `+0x0a` bit6 (`0x40`) | Compander | `0`=off `1`=on | confirmed |
| `+0x0a` bit7 (`0x80`) | Talk Around (clears Reverse bit) | `0`=off `1`=on | confirmed |
| `+0x0b` bits0-1 (`0x03`) | Encode Type | `00`=Off `01`=CTCSS `02`=DCS | confirmed |
| `+0x0b` bits2-3 (`0x0c`) | Decode Type | `00`=Off `04`=CTCSS `08`=DCS | confirmed |
| `+0x0b` bits4-7 (`0xf0`) | DTMF memory slot index, 0-based, only meaningful when `+0x10`=DTMF | e.g. `d0`=slot14 | confirmed |
| `+0x0c` | CTCSS Encode value, 1-based index into CTCSS table | e.g. `1f`(31)=171.3. Factory `09`(9)=88.5 | confirmed |
| `+0x0d` | CTCSS Decode value, same table | e.g. `14`(20)=127.3. Factory `09`(9)=88.5 | confirmed |
| `+0x0e` | DCS Encode code, low 8 bits of decimal value of 3 octal digits; wraps + sets `+0x09` bit5 over 255 | e.g. `2d`=code055, `fd`=code775(carry), `40`=code500(carry). Factory `13` | confirmed |
| `+0x0f` | DCS Decode code, same encoding as `+0x0e`, carry at `+0x0a` bit5 | e.g. `2d`=code055, `40`=code500(carry). Factory `13` | confirmed |
| `+0x10` | Optional Signaling type | `00`=Off `01`=DTMF `02`=2-Tone `03`=5-Tone | confirmed |
| `+0x11`-`+0x12` | unattributed, constant | `00 00` | unknown |
| `+0x13`-`+0x19` | CH Name, 7 bytes ascii | e.g. `HELLO  ` (space-padded when set). Factory: all `00` (NUL, not space) | confirmed |
| `+0x1a` | Busy Channel Lock-out | `00`=Off(factory) `01`=Repeater `02`=Busy | confirmed |
| `+0x1b` | unattributed, constant | `00` | unknown |
| `+0x1c` low nibble | DTMF PTT ID | `0`=Off(factory) `1`=BOT `2`=EOT `3`=Begin+End | confirmed |
| `+0x1c` high nibble | 5Tone PTT ID | `0`=Off(factory) `1`=BOT `2`=EOT `3`=Begin+End | confirmed |
| `+0x1d` | Squelch Mode | `00`=Carrier(factory) `01`=CTCSS/DCS `02`=OptSignal `03`=CTCSS-DCS AND OptSignal `04`=CTCSS-DCS OR OptSignal | confirmed |
| `+0x1e` | Scrambler | `00`=Off(factory) `01`=On | confirmed |
| `+0x1f` | unattributed, constant | `00` | unknown |
| n/a | copy-channel action | copying ch0->ch113 populates ch113's record + sets used-flag bit | confirmed |

CTCSS table, 1-based index (`+0x0c`/`+0x0d`):

```
 1  67.0    11  94.8    21 131.8    31 171.3    41 203.5
 2  69.3    12  97.4    22 136.5    32 173.8    42 206.5
 3  71.9    13 100.0    23 141.3    33 177.3    43 210.7
 4  74.4    14 103.5    24 146.2    34 179.9    44 218.1
 5  77.0    15 107.2    25 151.4    35 183.5    45 225.7
 6  79.7    16 110.9    26 156.7    36 186.2    46 229.1
 7  82.5    17 114.8    27 159.8    37 189.9    47 233.6
 8  85.4    18 118.8    28 162.2    38 192.8    48 241.8
 9  88.5    19 123.0    29 165.5    39 196.6    49 250.3
10  91.5    20 127.3    30 167.9    40 199.5    50 254.1
```

### Functions

| Offset | Field | Value | Status |
|---|---|---|---|
| `0x0200`-`0x020f` | unattributed | all `00` | unknown |
| `0x0210`-`0x0214` | password, paired with enable flag `0x0238` | ascii. Factory `12345`, NUL-padded | confirmed |
| `0x0215`-`0x0219` | unattributed | all `00` | unknown |
| `0x021a` | Key Lock | `00`=off `01`=on | confirmed |
| `0x021b`-`0x021f` | unattributed | all `00` | unknown |
| `0x0220` | Display Mode | `00`=Freq(factory) `01`=Channel `02`=Name | confirmed |
| `0x0221` | VFO/MR | `00`=VFO(factory) `01`=MR (auto-forced when Display Mode=Channel) | confirmed |
| `0x0222` | MR Channel, raw channel number | e.g. `71`=113. Factory `00` | confirmed |
| `0x0223` | unattributed, constant | `04` | unknown |
| `0x0224` | unattributed, constant | `00` | unknown |
| `0x0225` | Frequency Scan | `00`=TO(factory) `01`=CO `02`=SE | confirmed |
| `0x0226` | Channel display lock (shown only when Display Mode=Channel) | `00`=No Lock(factory) `01`=Lock | confirmed |
| `0x0227` | unattributed, constant | `00` | unknown |
| `0x0228` | Backlight Brightness, 0-indexed | e.g. `1b`(27)->displayed"28". Factory `1d`(29) | confirmed |
| `0x0229` | unattributed, constant | `00` | unknown |
| `0x022a` | Back Light Color | `00`=Orange(factory) `01`=Blue `02`=Purple | confirmed |
| `0x022b` | TBST Frequency | `00`=1750Hz(factory) `01`=2100Hz `02`=1000Hz `03`=1450Hz | confirmed |
| `0x022c` | Time Out Timer, direct minutes | `05`=5min. Factory `00` | confirmed |
| `0x022d` | unattributed, constant | `00` | unknown |
| `0x022e` | Auto Power Off | `00`=Off(factory) `01`=30min `02`=1hour `03`=2hour | confirmed |
| `0x022f` | Voice Prompt | `00`=BeepOff `01`=BeepOn(factory) | confirmed |
| `0x0230` bit0 | Moni Key Function | `0`=SquelchOffMomentary(factory) `1`=SquelchOff | confirmed |
| `0x0230` bit1 | Eliminate Squelch Tail (no CTCSS/DCS) | `0`=off `1`=on | confirmed |
| `0x0231` | unattributed, constant | `00` | unknown |
| `0x0233` bit1 (`0x02`) | Inhibit To Setup Background Operations | `0`=off(factory) `1`=on | confirmed |
| `0x0233` bit3 (`0x08`) | Inhibit Initialize Operation | `0`=off(factory) `1`=on | confirmed |
| `0x0233` other bits | unattributed, inferred from OR-ing, not toggle-tested | - | unconfirmed |
| `0x0234` | Tail Eliminator Type (no 90 Degree option) | `00`=Off(factory) `01`=120Degree `02`=180Degree | confirmed |
| `0x0235` | Choose TX Power | `00`=60W(VHF)/45W(UHF)(factory) `01`=25W/25W | confirmed |
| `0x0236`-`0x0237` | unattributed | `00 00` | unknown |
| `0x0238` | Use Boot-Strap Password | `00`=off(factory) `01`=on (password at `0x0210`) | confirmed |
| `0x0239`-`0x023f` | unattributed | all `00` | unknown |
| `0x0240`-`0x029f` | unattributed | all `00` | unknown |
| `0x02a0` +0x00-0x03 | Identical layout/factory value to `0x02c0`, not a VFO A/B pair (radio has one VFO) | Factory `01 45 00 00` = 145.000 | unknown (purpose) |
| `0x02a0` +0x04-0x05 | unattributed, part of the 16-byte block | not individually isolated | unknown |
| `0x02a0` +0x06-0x0f | rest of the 16-byte block | `60 00 01 00 00 00 09 09 13 13` | unknown (purpose) |
| `0x02b0`-`0x02bf` | unattributed | all `00` | unknown |
| `0x02c0` +0x00-0x03 | Remembered VFO dial frequency, same encoding as channel RX freq. Only changes via radio's own front-panel dial | Factory `01 45 00 00` = 145.000 | confirmed |
| `0x02c0` +0x04-0x05 | unattributed, part of the 16-byte block | not individually isolated | unknown |
| `0x02c0` +0x06-0x0f | rest of the 16-byte block | `60 00 01 00 00 00 09 09 13 13` | unknown (purpose) |
| `0x02d0`-`0x02df` | unattributed | all `00` | unknown |
| `0x03c0`-`0x03df` | unattributed | all `00` | unknown |
| `0x03e0`-`0x03e6` | Starting Display, ascii, null-padded | e.g. `Potato\0` | confirmed |
| `0x03e7`-`0x03ef` | unattributed | all `00` | unknown |
| `0x03f0`-`0x03ff` | unattributed | all `ff` | unknown |

### DTMF

16-byte records (BOT/EOT, M1-M16) pack digits BCD 2/byte, odd count padded `0`, length byte at
`+0x0c`. Remotely Kill/Stun: 8-byte record, length at `+0x07`. Self ID: one raw decimal digit/byte,
not packed.

| Offset | Field | Value | Status |
|---|---|---|---|
| `0x0230` bit2 (`0x04`) | DTMF ANI | `0`=Off(factory) `1`=On | confirmed |
| `0x0232` | DTMF Transmitting Time, 0-based index | `00`=30ms `01`=50ms(factory) `02`=100ms `03`=200ms `04`=300ms `05`=500ms | confirmed |
| `0x02e0` | DTMF Interval Character (`a`=A..`d`=D, `e`=`*`, `f`=`#`) | `0e`=`*`(factory) `0f`=`#` | confirmed |
| `0x02e1` | DTMF Group Code (Off uses sentinel, not nibble scheme) | `0a`=A(factory) `0b`=B `ff`=Off | confirmed |
| `0x02e2` | DTMF Decoding Response | `00`=None(factory) `01`=BeepTone `02`=BeepTone+Respond | confirmed |
| `0x02e3` | DTMF Pretime, ms/10, range 10-1500ms | `14`=200ms(factory) `1e`=300ms | confirmed |
| `0x02e4` | DTMF First Digit Time, ms/10, range 10-1500ms | `14`=200ms(factory) `1e`=300ms | confirmed |
| `0x02e5` | DTMF Auto Reset Time, s x10, range 0.0-25.0s | `00`=0.0s(factory) `0a`=1.0s | confirmed |
| `0x02e6` | unattributed, constant | `00` | unknown |
| `0x02e7`-`0x02e9` | DTMF Self ID, one raw decimal digit/byte | e.g. `456`->`04 05 06` | confirmed |
| `0x02ea`-`0x02ec` | unattributed, constant | `00 00 00` | unknown |
| `0x02ed` | DTMF Side Tone | `0`=off(factory) `1`=on | confirmed |
| `0x02ee` | DTMF Time-Lapse After Encode, ms/10, range 10-1500ms | `14`=200ms(factory) `1e`=300ms | confirmed |
| `0x02ef` | DTMF PTT ID Pause Time, direct seconds (`Off`, or 5-75 step5) | `00`=Off `0a`=10s(factory) `14`=20s | confirmed |
| `0x02f0`-`0x02f7` | DTMF Remotely Kill code | e.g. `999`->`99 90`, length `03`@`+0x07` | confirmed |
| `0x02f8`-`0x02ff` | DTMF Remotely Stun code | e.g. `888`->`88 80`, length `03`@`+0x07` | confirmed |
| `0x0340`-`0x034f` | DTMF PTT ID Starting (BOT) code, BCD 2/byte, digits +0x00 onward | e.g. `111`->`11 10`, length `03`@`+0x0c` | confirmed |
| `0x034d`-`0x034f` | unattributed, record tail after length byte | all `00` | unknown |
| `0x0350`-`0x035f` | DTMF PTT ID Ending (EOT) code, BCD 2/byte, digits +0x00 onward | e.g. `222`->`22 20`, length `03`@`+0x0c` | confirmed |
| `0x035d`-`0x035f` | unattributed, record tail after length byte | all `00` | unknown |
| `0x0400`+N*0x10 (N=0-15, `0x0400`-`0x04ff`) | DTMF memory M(N+1) code, BCD 2/byte, digits +0x00 onward. M1 also written by Special Call wizard: `ANI`->`<OtherSideID>#<SelfID>`, `PTTID`->`#<SelfID>` (no dedicated storage for the wizard's own controls) | e.g. `123456`->`12 34 56`, length `06`@`+0x0c` | confirmed |
| `0x0400`+N*0x10 +0x0d-+0x0f | unattributed, record tail after length byte | all `00` (tested M1, row N=0) | unknown |

### 2-Tone

Decode tab: only 2 of 4 tone boxes editable at once (whichever Call Format uses). Editing a tone
also writes a big-endian mirror + a `48000000/freq` tuning word.

| Offset | Field | Value | Status |
|---|---|---|---|
| `0x0300`-`0x0301` | ATone Frequency, LE uint16, Hz x10 | e.g. `91 0c`=3217=321.7Hz | confirmed |
| `0x0302`-`0x0303` | Active 2nd-position tone freq (mirrors B/C/D), same encoding | e.g. `d6 4f`=20438=2043.8Hz(D active) | confirmed |
| `0x0304`-`0x0305` | ATone tuning word, LE uint16 = `floor(48000000/Hz_x10)` | `2880`->`16666`, `3217`->`14920`, `3333`->`14401`, `5000`->`9600`, `10000`->`4800` | confirmed |
| `0x0306`-`0x0307` | Active 2nd-position tuning word, same formula, tracks B/C/D | e.g. `288.0Hz`->`16666`, `500.0Hz`->`9600` | confirmed |
| `0x0308` | Decoding Response | `00`=None(factory) `01`=BeepTone `02`=BeepTone+Respond | confirmed |
| `0x0309` | 1st Tone Duration, s x10, range 0.5-10.0s | `05`=0.5s(factory) | confirmed |
| `0x030a` | 2nd Tone Duration, s x10, range 0.5-10.0s | `05`=0.5s(factory) | confirmed |
| `0x030b` | Long Tone Duration, s x10, range 0.5-10.0s | `0a`=1.0s(factory) | confirmed |
| `0x030c` | Gap Time, ms/100, range 0-2000ms | `00`=0ms(factory) | confirmed |
| `0x030d` | Auto Reset Time, s x10, range 0.0-25.0s | `00`=0.0s(factory) | confirmed |
| `0x030e` | Side Tone | `0`=off(factory) `1`=on | confirmed |
| `0x030f` | 2Tone Call Format, 0-based index | `00`=A-B(factory) `01`=A-C `02`=A-D `03`=B-A `04`=B-C `05`=B-D `06`=C-A `07`=C-B `08`=C-D `09`=D-A `0a`=D-B `0b`=D-C `0c`=LongA `0d`=LongB `0e`=LongC | confirmed |
| `0x0310`-`0x0311` | ATone Frequency, big-endian mirror of `0x0300` | e.g. `288.0Hz`(2880)->`0b 40` | confirmed |
| `0x0312`-`0x0313` | Active 2nd-position tone freq, BE mirror of `0x0302` | e.g. `288.0Hz`(2880)->`0b 40` | confirmed |
| `0x0314`-`0x0315` | CTone Frequency, BE, dedicated storage | e.g. `288.0Hz`(2880)->`0b 40` | confirmed |
| `0x0316`-`0x0317` | DTone Frequency, BE, dedicated storage | e.g. `288.0Hz`(2880)->`0b 40` | confirmed |
| `0x0318`-`0x031f` | unattributed | all `00` | unknown |
| `0x0380`-`0x0383` | Encode memory used-flag bitmap, 32 bits, bit0=row0..bit255=row31 | `0`=used `1`=empty. Factory `fe ff ff ff` | confirmed |
| `0x0384`-`0x038f` | unattributed | all `ff` | unknown |
| `0x0b00`+N*0x10 (N=0-31, `0x0b00`-`0x0cff`) | 2-Tone Encode memory row N. Freq fields LE uint16 Hz x10 (BCD elsewhere in radio, not here) | +0x00-01 1st tuning word LE = `floor(6000000/freq)` (`288.0Hz`->`2083`, `500.0Hz`->`1200`, `700.0Hz`->`857`)<br>+0x02-03 2nd tuning word same formula<br>+0x04-05 1st Tone Freq e.g. `40 0b`=2880=288.0Hz<br>+0x06-07 2nd Tone Freq e.g. `b8 79`=31160=3116.0Hz<br>+0x08-0e Name 7B ascii space-padded<br>+0x0f unused `00` | confirmed |

### 5-Tone

Codes are tone-standard digit sequences (`0`-`9`,`A`-`F`), BCD-packed 2/byte, odd count padded `e`
(not `0`). Repeat-tone rule: a digit repeating the previous one becomes `E`, but two `E`s never
appear back to back. `11`->`1E`, `111`->`1E1`, `11111`->`1E1E1`, `2222`->`2E2E`, `11112`->`1E1E2`,
`12111`->`121E1`. Exception: BOT/EOT field only, `1111` (4 ones, end of string) -> `1E1E1` (5
chars) instead of `1E1E`; same `1111` in Encode ID field -> clean `1E1E`.

| Offset | Field | Value | Status |
|---|---|---|---|
| `0x0320` | unattributed | `00` | unknown |
| `0x0321` | Decoding Response | `00`=None(factory) `01`=BeepTone `02`=BeepTone+Respond | confirmed |
| `0x0322` | Decode Standard, 0-based index | `00`=ZVEI1(factory) `01`=ZVEI2 `02`=ZVEI3 `03`=PZVEI `04`=DZVEI `05`=PDZVEI `06`=CCIR1 `07`=CCIR2 `08`=PCCIR `09`=EEA `0a`=EuroSignal `0b`=NATEL `0c`=MODAT `0d`=CCITT | confirmed |
| `0x0323` | Self ID length | e.g. `12345`->`05`. Factory `00` | confirmed |
| `0x0324` | Time Of Decode Tone, ms/10, range 30-100ms | `07`=70ms(factory) `08`=80ms | confirmed |
| `0x0325`-`0x0329` | Self ID digits, raw decimal/byte, max 5 | e.g. `12345`->`01 02 03 04 05`. Factory all `00` | confirmed |
| `0x032a` | unattributed | `00` | unknown |
| `0x032b` | Pretime, ms/10, range 10-2550ms | `14`=200ms(factory) `1e`=300ms | confirmed |
| `0x032c` | Time-Lapse After Encode, ms/10, range 10-2550ms | `14`=200ms(factory) `1e`=300ms | confirmed |
| `0x032d` | PTT ID Pause Time, direct seconds (`Off`, or 5-75) | `0a`=10s(factory) `14`=20s | confirmed |
| `0x032e` | Auto Reset Time, s x10, range 0.0-25.0s | `00`=0.0s(factory) `0a`=1.0s | confirmed |
| `0x032f` | First Delay, ms/10, range 10-2550ms | `14`=200ms(factory) `1e`=300ms | confirmed |
| `0x0330` | Side Tone | `0`=off(factory) `1`=on | confirmed |
| `0x0331` | unattributed | `00` | unknown |
| `0x0332` | Stop Code (`Off/B/C/D/F` only) | `00`=Off(factory) `0b`=B `0c`=C `0d`=D `0f`=F | confirmed |
| `0x0333` | Stop Time, ms/10, range 10-2550ms | `14`=200ms(factory) `1e`=300ms | confirmed |
| `0x0334` | Decode Time, ms/10, range 0-2000ms | `02`=20ms(factory) `32`=500ms | confirmed |
| `0x0335`-`0x033f` | unattributed | all `00` | unknown |
| `0x0340`-`0x035f` | shared with DTMF BOT/EOT, see DTMF section | | |
| `0x0360`-`0x0362` | unattributed | `00 00 00` | unknown |
| `0x0362` | PTT ID Starting (BOT) code length | e.g. `111`->`1E1`->`03` | confirmed |
| `0x0363` | unattributed, stray | `07` | unknown |
| `0x0364`-`0x0365` | PTT ID Starting (BOT) code digits, BCD, repeat-tone applies (`111`->stored`1E1`) | e.g. `1e 1e` | confirmed |
| `0x0366`-`0x0371` | unattributed | all `00` | unknown |
| `0x0372` | PTT ID Ending (EOT) code length | e.g. `222`->`2E2`->`03` | confirmed |
| `0x0373` | unattributed, stray | `07` | unknown |
| `0x0374`-`0x0375` | PTT ID Ending (EOT) code digits, same encoding as BOT | e.g. `2e 2e` | confirmed |
| `0x0376`-`0x037f` | unattributed | all `00` | unknown |
| `0x0390`-`0x039f` | Encode memory used-flag bitmap, same convention as `0x0100`/`0x0380` | `0`=used `1`=empty. Factory value disputed: `fe` (earlier verified reads) vs `ff` (2026-09-27 factory-reset-read-write capture, where row 0 at `0x0d00` also holds leftover non-zero content — see `alinco-dr138-mkii_readme.md`) | disputed |
| Info Function slot N (1-8), `0x0500+(N-1)*0x20` +0x00 | Function Option, 0-based index. Kill/Stun requires >=1 other slot = Remotely Wake Up | `00`=SquelchOff `01`=CallAll `02`=EmergencyAlarm `03`=RemotelyKill `04`=RemotelyStun `05`=RemotelyWakeUp `06`=GroupCall | confirmed |
| slot +0x01 | Decoding Response (SquelchOff: resp only, no name; CallAll: both; EmergencyAlarm: name only, no resp; Kill/Stun/WakeUp: forced BeepTone+Respond, name allowed; GroupCall: neither) | `00`=None `01`=BeepTone `02`=BeepTone+Respond | confirmed |
| slot +0x02 | Information ID length | e.g. `111`->`03` | confirmed |
| slot +0x03.. | Information ID digits, raw decimal/byte | e.g. `111`->`01 01 01` | confirmed |
| slot +0x10-+0x16 | Function Name, 7B ascii space-padded | e.g. `Test1  ` | confirmed |
| slot +0x17-+0x1f | unattributed, record tail after Name | all `00` (tested slot1, N=1) | unknown |
| `0x0600`-`0x0aff` | unattributed, gap between Info Function table and 2-Tone Encode memory table | all `00` | unknown |
| Encode memory row N, `0x0d00+N*0x20` (row count/upper bound not independently tested — 100 rows fits exactly against the Communication Notes table start at `0x1980`, treat as unconfirmed) +0x00 | Encode Standard, same index as `0x0322` | `00`=ZVEI1(factory) | confirmed |
| row +0x01 | unattributed | `00` | unknown |
| row +0x02 | Encode ID length | e.g. `12345`->`05` | confirmed |
| row +0x03 | Time Of Encode Tone, ms/10 (present even on unprogrammed rows) | `07`=70ms(factory) | confirmed |
| row +0x04.. | Encode ID digits, BCD 2/byte, odd padded `e` | e.g. `12345`->`12 34 5e` | confirmed |
| row +0x18-+0x1e | Name, 7B ascii space-padded | e.g. `test   ` | confirmed |
| row +0x1f | unattributed, record tail | `00` (tested row0) | unknown |
| Special Call wizard, targets Encode table row 0-99 by group number, Calling Type=PTTID | writes literal `E6` + 5-Tone Self ID | Self ID `12345`->`E612345` | confirmed |
| Calling Type=ANI | writes `<OtherSideID><IntervalChar><SelfID>` | OtherSideID`12345`,IntervalChar`D`,SelfID`12345`->`12345D12345` | confirmed |
| Calling Type=Send Message | digit portion `E`+`"1"`+OtherSideID w/ repeat-tone substitution, padded even with trailing `E`; message follows as plain ASCII, independent of digit portion | OtherSideID`12345`->`E1E2345E`; `67890`->`E167890E`; `34567`->`E134567E`; `11234`->`E1E1234E`. Message `Hello`->ASCII `48 45 4c 4c 4f` right after digits | confirmed |

### Information of Scanning

| Offset | Field | Value | Status |
|---|---|---|---|
| `0x03b0` | Scan Mode | `00`=Off `01`=On | confirmed |
| `0x03b1` | Priority Channel | `00`=Off(factory) `01`=Ch1 `02`=Ch2 `03`=Ch1&2 | confirmed |
| `0x03b2` | Revert Channel (index `03` unused/unseen) | `00`=Selected(factory) `01`=Selected+TalkBack `02`=PriorityCh1 `04`=LastCalled `05`=LastUsed `06`=PriorityCh1+TalkBack(forces `0x03b1`=`01`) | confirmed |
| `0x03b3` | Look Back Time A, s=(raw/10)+0.5, range 0.5-5.0s | `05`=1.0s(factory) `06`=1.1s | confirmed |
| `0x03b4` | Look Back Time B, same formula | `05`=1.0s(factory) `06`=1.1s | confirmed |
| `0x03b5` | Dropout Delay Time, s=(raw+1)/10, range 0.1-5.0s | `00`=0.1s(factory) `01`=0.2s | confirmed |
| `0x03b6` | Dwell Time, same formula | `00`=0.1s(factory) `01`=0.2s | confirmed |
| `0x03b7` | unattributed | `00` | unknown |
| `0x03b8` | Scan Enter Tone | `0`=off(factory) `1`=on | confirmed |
| `0x03b9` | unattributed | `ff` | unknown |
| `0x03ba` | unattributed | `00` | unknown |
| `0x03bb` | Priority Channel 1, raw channel number | e.g. ch5->`05`, ch10->`0a`. Factory `00` | confirmed |
| `0x03bc` | unattributed | `00` | unknown |
| `0x03bd` | Priority Channel 2, same encoding as Priority Channel 1 | e.g. ch10->`0a`. Factory `00` | confirmed |
| `0x03be`-`0x03bf` | unattributed | `00 00` | unknown |

### Emergency Info

Emergency Alarm mode gates edit access: Alarm mode enables Alarm Time only; both Transpond modes
and Both enable Duration of TX/RX instead.

| Offset | Field | Value | Status |
|---|---|---|---|
| `0x03a0` | Emergency Alarm | `00`=Alarm(factory) `01`=Transpond+Background `02`=Transpond+Alarm `03`=Both | confirmed |
| `0x03a1` | ENI Type Select | `00`=None(factory) `01`=DTMF `02`=5Tone | confirmed |
| `0x03a2` | Emergency ID, 0-based index into DTMF M1-M16 or 5-Tone Encode rows per ENI Type | `00`=M1/row0 `01`=M2/row1 | confirmed |
| `0x03a3` | Alarm Time[s], raw, range 1-255 | e.g. `01`=1s. Factory `01` | confirmed |
| `0x03a4` | Duration of TX[s], raw, range 1-255, `00` when grayed out | e.g. `01`=1s. Factory `00` | confirmed |
| `0x03a5` | Duration of RX[s], raw, range 1-255, `00` when grayed out | e.g. `01`=1s. Factory `00` | confirmed |
| `0x03a6` | Emergency ENI Send Select | `00`=Assigned(factory) `01`=Selected | confirmed |
| `0x03a7` | unattributed | `00` | unknown |
| `0x03a8` | Emergency Channel, raw channel number | e.g. ch5->`05`. Factory `00` | confirmed |
| `0x03a9` | Emergency Cycle | `00`=Continuous(factory) other=cycle count 1-255 | confirmed |
| `0x03aa` | unattributed | `00` | unknown |
| `0x03ab` | unattributed | `14` | unknown |
| `0x03ac` | changes once `01`->`05` on first departure from factory Alarm mode; further changes incl. reverting leave it `05` | Factory `01` | confirmed |
| `0x03ad`-`0x03af` | unattributed | `00 00 00` | unknown |

Emergency ID shows empty in the vendor UI when no matching DTMF/5-Tone code is programmed yet.

### Communication Notes

Rows `0`-`127` displayed. Call ID (5-digit) and Name stored in two parallel arrays, sorted
ascending by Call ID, packed from slot 0 (not by displayed row). No count field — in-use = packed
from slot 0 until first all-zero Call ID. Adding an entry re-sorts; deleting compacts the array.

| Offset | Field | Value | Status |
|---|---|---|---|
| `0x1980`-`0x1bff` | Call ID array, slot N (ascending Call ID order) at `0x1980+N*5`, 5B raw ASCII digits, no padding | e.g. `12345`->`31 32 33 34 35` | confirmed |
| `0x1c00`-`0x1fff` | Name array, slot N at `0x1c00+N*8`, 8B ASCII, null-padded | e.g. `TEST`->`54 45 53 54 00 00 00 00` | confirmed |

### Embedded Message

Not accessible in this vendor software installation — menu item shows an error dialog instead of
a config screen.

### Blanked / regenerated on every write

| Offset | Field | Value | Status |
|---|---|---|---|
| `0x3900`-`0x3ff0` | forced to zero by every write; non-zero values return after the radio's post-write auto-reset | `00` (as written) | unknown (purpose) |
