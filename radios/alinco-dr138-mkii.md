# Alinco DR-138 MKII

VHF mobile, 144-148MHz. ERW-7 clone cable (CP2102/FTDI). Same cable as the DR-138T, different
write protocol - don't assume T-model notes carry over.

## Protocol

- 9600 8N1, no flow control.
- Handshake: PC sends `PROGRAM` (echoed back), radio sends `QX\x06`, PC sends `\x02`, radio sends
  a 16-byte ID block starting with the model string.
- Every command is echoed back before the real reply.
- 16-byte blocks. Checksum = `sum(header + data) & 0xFF`.
- First command after ID, every session, must be a read of `0x0040`. Radio won't answer anything
  else until this happens. Every read/write still ACKs fine if you skip it - the only symptom is
  the radio never auto-resets after a write, requiring a power-cycle for the write to take effect.
- Reads: `0x0010`-`0x3ff0`. Writes: `0x0100`-`0x3ff0` only (`0x0010`-`0x00ff` is read-only).
- Every write blanks `0x3900`-`0x3ff0` to zero and `0x0160` to `0xFF`, regardless of what changed.
  Looks like calibration data; the radio regenerates it after reset.
- 16KB image. Channels are 32 bytes each, 200 of them, starting at `0x2000`.

## Memory map

Field names match the vendor software's dialogs exactly, where known. Value lists every byte
value known or guessed for that field, paired with its setting name - untested ones are marked
`(guess)`. Status: `confirmed` / `unconfirmed` / `unknown`.

Sections below are grouped by vendor-software dialog, not by address - some dialogs (DTMF in
particular) store their settings inside byte ranges that otherwise belong to a different page, so
grouping by address alone hides which menu actually controls a given byte.

### Identification (read-only)

| Offset | Field | Value | Status |
|---|---|---|---|
| `0x0010` | model string | ascii, e.g. `DJ-138` | confirmed |
| `0x0030` | date string | ascii, e.g. `2024/11/26` | confirmed |

### Channel

Channel Edit dialog. Channel table is 200 x 32 bytes, channel N = `0x2000 + N*0x20`. Tested on
channel 0 - offsets below are relative to a channel's start (`+0xNN`) unless otherwise noted.

| Offset | Field | Value | Status |
|---|---|---|---|
| `0x0100`-`0x011f` | used-flag bitmap, 1 bit/channel, bit N = channel N. Not a vendor checkbox - tracks whether that channel slot has any data written to it at all, and gets set/cleared automatically as a side effect of programming or blanking a channel | `0`=used (channel has data)<br>`1`=empty (blank slot) | confirmed |
| `0x0120`-`0x013f` | skip-flag bitmap, same shape - matches the vendor's "Skip" checkbox in the Channel Edit dialog. Channel still exists and is selectable in MR mode, it's just left out of scan | `0`=not skipped<br>`1`=skipped | confirmed |
| `0x0140`-`0x015f` | unattributed, all `0xff` on every unit seen | - | unknown |
| `0x0160` | unattributed, forced to `0xff` on every write regardless of change | - | unknown |
| `0x0170`-`0x01ff` | unattributed, all `0xff` | - | unknown |
| `+0x00`-`+0x03` | RX Frequency: byte0 = leading digit (raw), bytes 1-3 = BCD digit pairs, x100Hz | numeric, e.g. `01 44 50 00` = 144.50000 | confirmed |
| `+0x04` | unattributed, constant `00` in every sample including both duplex directions | - | unknown |
| `+0x05`-`+0x07` | duplex offset magnitude backing TX Frequency: 3 bytes BCD digit pairs, x100Hz. Remembered even when TX=RX (simplex) - factory-default channel has RX=TX=145.00000 but this still reads `00 60 00` = 0.6MHz, the standard VHF repeater shift | numeric, e.g. `02 62 50` = 2.625MHz -> RX 144.500 + offset = TX Frequency 147.125 | confirmed |
| `+0x08` | Step - 0-based sequential index into the dropdown | `00`=2.5K<br>`01`=5K (factory default)<br>`02`=6.25K<br>`03`=8.33K<br>`04`=10K<br>`05`=12.5K<br>`06`=20K<br>`07`=25K<br>`08`=30K<br>`09`=50K | confirmed |
| `+0x09` bit 0 | TX Off | `0`=unchecked<br>`1`=checked | confirmed |
| `+0x09` bit 1 | Reverse | `0`=unchecked<br>`1`=checked | confirmed |
| `+0x09` bits 2-3 (mask `0x0c`) | Channel Spacing | `00`=25KHz (factory default)<br>`04`=20KHz<br>`08`=12.5KHz | confirmed |
| `+0x09` bit 4 (`0x10`) | Encode DCS Normal/Reverse polarity | `0`=Normal<br>`1`=Reverse | confirmed |
| `+0x09` bit 5 (`0x20`) | Carry bit for the Encode DCS code at `+0x0e` - adds 256 when the code's decimal value exceeds 255 | `0`=code <= 255<br>`1`=code + 256 | confirmed |
| `+0x0a` bit 6 (`0x40`) | Compander | `0`=unchecked<br>`1`=checked | confirmed |
| `+0x0a` bit 7 (`0x80`) | Talk Around - setting this clears the Reverse bit at `+0x09` bit 1 | `0`=unchecked<br>`1`=checked | confirmed |
| `+0x0a` bits 2-3 (mask `0x0c`) | TX Power | `00`=High (factory default)<br>`04`=Mid<br>`08`=Low | confirmed |
| `+0x0a` bit 0 (`0x01`) | Duplex direction - inferred from RX/TX sign, no dedicated UI field (dialog just shows two absolute frequency boxes) | `0`=- (TX below RX)<br>`1`=+ (TX above RX) | confirmed |
| `+0x0a` bit 1 (`0x02`) | unattributed, constant `1` in samples so far | - | unknown |
| `+0x0a` bit 4 (`0x10`) | DCS Decode Normal/Reverse polarity - CTCSS Decode has no Reverse option in the vendor dialog, this bit is DCS-only | `0`=Normal<br>`1`=Reverse | confirmed |
| `+0x0a` bit 5 (`0x20`) | Carry bit for the Decode DCS code at `+0x0f` - adds 256 when the code's decimal value exceeds 255, symmetric to the Encode carry bit at `+0x09` bit 5 | `0`=code <= 255<br>`1`=code + 256 | confirmed |
| `+0x0b` bits 0-1 (mask `0x03`) | Encode Type | `00`=Off<br>`01`=CTCSS<br>`02`=DCS | confirmed |
| `+0x0b` bits 2-3 (mask `0x0c`) | Decode Type - same shape as Encode Type above | `00`=Off<br>`04`=CTCSS<br>`08`=DCS | confirmed |
| `+0x0b` bits 4-7 (mask `0xf0`) | DTMF memory slot index, 0-based - only meaningful when Optional Signaling (`+0x10`) is set to DTMF. Selects one of the M1-M16 memories from the DTMF dialog (see DTMF section) | numeric, e.g. `d0` = slot 14 (nibble `d`=13, 0-based) | confirmed |
| `+0x0c` | CTCSS Encode value - 1-based index into the standard 50-tone list below. Only used when Encode Type=CTCSS; DCS Encode code lives at `+0x0e` instead | `1f`(31)=171.3 | confirmed |
| `+0x0d` | CTCSS Decode value - same table. Only used when Decode Type=CTCSS; DCS Decode code lives at `+0x0f` instead | `14`(20)=127.3 | confirmed |
| `+0x0e` | DCS Encode code - low 8 bits of the code's plain decimal value when its 3 octal digits are read as a number (not an index into a table). E.g. code `055` -> decimal 45 -> `2d`. Values over 255 (e.g. code `775` -> decimal 509, code `500` -> decimal 320) wrap and set the carry bit at `+0x09` bit 5 | e.g. `2d`=code 055, `fd`=code 775 (with carry bit set), `40`=code 500 (with carry bit set) | confirmed |
| `+0x0f` | DCS Decode code - same encoding as `+0x0e`, carry bit at `+0x0a` bit 5 | e.g. `2d`=code 055, `40`=code 500 (with carry bit set) | confirmed |
| `+0x10` | Optional Signaling type - selects DTMF/2-Tone/5-Tone codes configured in their own dialogs (see below) | `00`=Off<br>`01`=DTMF<br>`02`=2-Tone<br>`03`=5-Tone | confirmed |
| `+0x11`-`+0x12` | unattributed, constant `00 00` | - | unknown |
| `+0x13`-`+0x19` | CH Name, 7 bytes ascii, space-padded | free text, e.g. `HELLO  ` | confirmed |
| `+0x1a` | Busy Channel Lock-out | `00`=Off (factory default)<br>`01`=Repeater<br>`02`=Busy | confirmed |
| `+0x1b` | unattributed, constant `00` | - | unknown |
| `+0x1c` low nibble | DTMF PTT ID - selects BOT/EOT code configured in the DTMF dialog (see below) | `0`=Off (factory default)<br>`1`=BOT<br>`2`=EOT<br>`3`=Begin And End | confirmed |
| `+0x1c` high nibble | 5Tone PTT ID - independent of the low nibble | `0`=Off (factory default)<br>`1`=BOT<br>`2`=EOT<br>`3`=Begin And End | confirmed |
| `+0x1d` | Squelch Mode | `00`=Carrier (factory default)<br>`01`=CTCSS/DCS<br>`02`=Opt Signal<br>`03`=CTCSS-DCS AND Opt Signal<br>`04`=CTCSS-DCS OR Opt Signal | confirmed |
| `+0x1e` | Scrambler Switch | `00`=Off (factory default)<br>`01`=On | confirmed |
| `+0x1f` | unattributed, constant `00` | - | unknown |
| n/a | copy-channel action, not a dialog field or stored setting | copying ch0 -> ch113 populates ch113's record and sets its used-flag bit | confirmed |

Standard CTCSS table, 1-based index (`+0x0c`/`+0x0d` above):

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

Function Setup dialog.

| Offset | Field | Value | Status |
|---|---|---|---|
| `0x0210`-`0x0214` | password (paired with the enable flag at `0x0238`) | ascii, factory default `12345` | confirmed |
| `0x021a` | Key Lock | `00`=unchecked<br>`01`=checked | confirmed |
| `0x0220` | Display Mode | `00`=Freq (factory default)<br>`01`=Channel<br>`02`=Name | confirmed |
| `0x0221` | VFO/MR | `00`=VFO<br>`01`=MR (forced automatically when Display Mode is set to Channel) | confirmed |
| `0x0222` | MR Channel | raw byte = channel number directly, e.g. `71`(hex)=113 | confirmed |
| `0x0223` | unattributed, constant `04`, never toggled | - | unknown |
| `0x0224` | unattributed, constant `00` | - | unknown |
| `0x0225` | Frequency Scan - dropdown offers `TO/CO/SE`, abbreviation meaning not confirmed | `00`=TO (factory default)<br>`01`=CO<br>`02`=SE | confirmed |
| `0x0226` | Channel display lock (only shown when Display Mode is set to Channel) | `00`=No Lock<br>`01`=Lock | confirmed |
| `0x0227` | unattributed, constant `00` | - | unknown |
| `0x0228` | Backlight Brightness | numeric, looks 0-indexed - `1b`(27)=displayed "28" | confirmed |
| `0x0229` | unattributed, constant `00` | - | unknown |
| `0x022a` | Back Light Color | `00`=Orange (factory default)<br>`01`=Blue<br>`02`=Purple | confirmed |
| `0x022b` | TBST Frequency | `00`=1750Hz (factory default)<br>`01`=2100Hz<br>`02`=1000Hz<br>`03`=1450Hz | confirmed |
| `0x022c` | Time Out Timer | direct minutes, not an index - `05`=5minute | confirmed |
| `0x022d` | unattributed, constant `00` | - | unknown |
| `0x022e` | Auto Power Off | `00`=Off (factory default)<br>`01`=30min<br>`02`=1hour<br>`03`=2hour | confirmed |
| `0x022f` | Voice Prompt (only two options on this radio - no separate voice-vs-beep choice) | `00`=Beep Off<br>`01`=Beep On (factory default) | confirmed |
| `0x0230` bit 0 | Moni Key Function | `0`=Squelch Off Momentary (factory default)<br>`1`=Squelch Off | confirmed |
| `0x0230` bit 1 | Eliminate Squelch Tail When No CTCSS/DCS Signaling | `0`=unchecked<br>`1`=checked | confirmed |
| `0x0231` | unattributed, constant `00` | - | unknown |
| `0x0233` bit 1 (`0x02`) | Inhibit To Setup Background Operations | `0`=unchecked (factory default)<br>`1`=checked | confirmed |
| `0x0233` bit 3 (`0x08`) | Inhibit Initialize Operation | `0`=unchecked (factory default)<br>`1`=checked | confirmed |
| `0x0234` | Tail Eliminator Type. Radio only has 3 options - there is no 90 Degree | `00`=Off (factory default)<br>`01`=120 Degree<br>`02`=180 Degree | confirmed |
| `0x0235` | Choose TX Power | `00`=60W(VHF) 45W(UHF) (factory default)<br>`01`=25W(VHF) 25W(UHF) | confirmed |
| `0x0238` | Use Boot-Strap PassWord | `00`=unchecked<br>`01`=checked (password held at `0x0210`) | confirmed |
| `0x02c0` +0x00-0x03 | Remembered VFO dial frequency, same encoding as the channel table's RX frequency. Not editable from any dialog - only changes when the VFO dial is turned on the radio's own front panel | - | confirmed |
| `0x02a0` +0x00-0x03 | Identical layout and factory value (`145.000`) to `0x02c0`. Not a VFO-A/VFO-B pair - this radio only has one VFO. Purpose unexplained | - | unknown |
| `0x02a0` and `0x02c0` +0x06-0x0f | rest of each 16-byte block (`60 00 01 00 00 00 09 09 13 13`), unattributed | - | unknown |
| `0x03e0`-`0x03e6` | Starting Display, ascii, null-padded | free text, e.g. `Potato\0` | confirmed |

### DTMF

DTMF configuration dialog. Several settings live in spare bits/bytes inside the Function Setup
byte range even though they're edited from this separate dialog - see `0x0230` bit 2 and `0x0232`
below.

16-byte records (BOT/EOT codes, M1-M16 memories) pack digits BCD 2-per-byte, last nibble padded
with `0` on an odd digit count, with a length byte at `+0x0c` inside each record. Remotely
Kill/Stun use the same digit packing in a smaller 8-byte record with the length byte at `+0x07`.
DTMF Self ID is the odd one out - one raw decimal digit per byte, not packed.

| Offset | Field | Value | Status |
|---|---|---|---|
| `0x0230` bit 2 (`0x04`) | DTMF ANI | `0`=Off (factory default)<br>`1`=On | confirmed |
| `0x0232` | DTMF Transmitting Time - dropdown is `30/50/100/200/300/500ms`, 0-based index into that list | `00`=30ms<br>`01`=50ms (factory default)<br>`02`=100ms<br>`03`=200ms<br>`04`=300ms<br>`05`=500ms | confirmed |
| `0x02e0` | DTMF Interval Character - dropdown only offers `A/B/C/D/*/#`, standard DTMF nibble encoding (`a`=A, `b`=B, `c`=C, `d`=D, `e`=`*`, `f`=`#`) | `0e`=`*` (factory default)<br>`0f`=`#` | confirmed |
| `0x02e1` | DTMF Group Code - dropdown offers `Off/A/B/C/D/*/#`, same nibble encoding as Interval Character above for the lettered options, but `Off` breaks that scheme and uses a sentinel value instead | `0a`=A (factory default)<br>`0b`=B<br>`ff`=Off | confirmed |
| `0x02e2` | DTMF Decoding Response - dropdown offers `None/Beep Tone/Beep Tone and Respond` | `00`=None (factory default)<br>`01`=Beep Tone<br>`02`=Beep Tone and Respond | confirmed |
| `0x02e3` | DTMF Pretime, milliseconds / 10. Full range is 10-1500ms in 10ms steps (`01`-`96`) | `14`=200ms (factory default)<br>`1e`=300ms | confirmed |
| `0x02e4` | DTMF First Digit Time, milliseconds / 10. Full range is 10-1500ms in 10ms steps (`01`-`96`) | `14`=200ms (factory default)<br>`1e`=300ms | confirmed |
| `0x02e5` | DTMF Auto Reset Time, seconds x10. Full range is 0.0-25.0s in 0.1s steps (`00`-`fa`) | `00`=0.0s (factory default)<br>`0a`=1.0s | confirmed |
| `0x02e6` | unattributed, constant `00` | - | unknown |
| `0x02e7`-`0x02e9` | DTMF Self ID - one raw decimal digit per byte | e.g. `456` -> `04 05 06` | confirmed |
| `0x02ea`-`0x02ec` | unattributed, constant `00 00 00` | - | unknown |
| `0x02ed` | DTMF Side Tone | `0`=unchecked (factory default)<br>`1`=checked | confirmed |
| `0x02ee` | DTMF Time-Lapse After Encode, milliseconds / 10. Full range is 10-1500ms in 10ms steps (`01`-`96`) | `14`=200ms (factory default)<br>`1e`=300ms | confirmed |
| `0x02ef` | DTMF PTT ID Pause Time, direct seconds. Dropdown also offers `Off` and 5-75 in steps of 5 | `00`=Off<br>`0a`=10s (factory default)<br>`14`=20s | confirmed |
| `0x02f0`-`0x02f7` | DTMF Remotely Kill code | e.g. `999` -> `99 90`, length `03` at `+0x07` | confirmed |
| `0x02f8`-`0x02ff` | DTMF Remotely Stun code | e.g. `888` -> `88 80`, length `03` at `+0x07` | confirmed |
| `0x0340`-`0x034f` | DTMF PTT ID Starting (BOT) code | BCD-packed digits, e.g. `111` -> `11 10`, length `03` at `+0x0c` | confirmed |
| `0x0350`-`0x035f` | DTMF PTT ID Ending (EOT) code | BCD-packed digits, e.g. `222` -> `22 20`, length `03` at `+0x0c` | confirmed |
| `0x0400`+N*0x10 (`N`=0-15, table spans `0x0400`-`0x04ff` for M1-M16) | DTMF memory M(N+1) code. M1 specifically is shared with the Special Call dialog's Calling Type/Other Side ID controls - those two controls have no storage anywhere, not in the radio and not even in the software's own UI state (the dialog always reopens showing `ANI`/`000` regardless of what was set and saved). They're a one-shot formatting action that writes into M1: `ANI` -> `<Other side ID>#<DTMF Self ID>` (e.g. Self ID `456`, Other side ID `789` -> M1 becomes `789#456`); `PTTID` -> `#<DTMF Self ID>` (Other side ID dropped, M1 becomes `#456`) | e.g. `123456` -> `12 34 56`, length `06` at `+0x0c` | confirmed |

### 2-Tone

2-Tone config dialog. Has separate Encode and Decode tabs; the memory table (used by Encode) is
covered further down.

**Decode tab.** Only two of the four tone frequency boxes are ever editable at once - whichever
two letters the current Call Format uses. Editing a tone also writes a big-endian mirror of the
same frequency and a separate `48000000/freq` tuning word alongside the human-readable value (see
`0x0304`/`0x0310` below).

| Offset | Field | Value | Status |
|---|---|---|---|
| `0x0300`-`0x0301` | ATone Frequency, little-endian uint16, value = Hz x10, same encoding as the Encode memory table | e.g. `91 0c` = 3217 = 321.7Hz | confirmed |
| `0x0302`-`0x0303` | Active second-position tone frequency - mirrors whichever letter is currently the second half of Call Format (B/C/D), same encoding as ATone | e.g. `d6 4f` = 20438 = 2043.8Hz (D active) | confirmed |
| `0x0304`-`0x0305` | ATone tuning word, little-endian uint16 = `floor(48000000 / ATone_Hz_x10)` (a fixed reference clock divided by tone frequency) | `2880`->`16666`, `3217`->`14920`, `3333`->`14401`, `5000`->`9600`, `10000`->`4800` | confirmed |
| `0x0306`-`0x0307` | Active second-position tone's tuning word, same `48000000/freq` relationship as `0x0304`, tracks whichever of B/C/D is currently active | e.g. `288.0Hz` -> `16666`, `500.0Hz` -> `9600` | confirmed |
| `0x0310`-`0x0311` | ATone Frequency, same value and encoding as `0x0300`-`0x0301` but stored big-endian instead of little-endian | e.g. ATone `288.0Hz` (raw `2880`) -> `0b 40` | confirmed |
| `0x0312`-`0x0313` | Active second-position tone frequency, big-endian mirror of `0x0302`-`0x0303`, tracks whichever of B/C/D is currently active | e.g. `288.0Hz` (raw `2880`) -> `0b 40` | confirmed |
| `0x0314`-`0x0315` | CTone Frequency, big-endian, dedicated storage independent of whether C is currently the active second position | e.g. `288.0Hz` (raw `2880`) -> `0b 40` | confirmed |
| `0x0316`-`0x0317` | DTone Frequency, big-endian, dedicated storage independent of whether D is currently the active second position | e.g. `288.0Hz` (raw `2880`) -> `0b 40` | confirmed |
| `0x0308` | Decoding Response - dropdown offers `None/Beep Tone/Beep Tone and Respond` | `00`=None (factory default)<br>`01`=Beep Tone<br>`02`=Beep Tone and Respond | confirmed |
| `0x030f` | 2Tone Call Format - 0-based sequential index into the dropdown | `00`=A-B (factory default)<br>`01`=A-C<br>`02`=A-D<br>`03`=B-A<br>`04`=B-C<br>`05`=B-D<br>`06`=C-A<br>`07`=C-B<br>`08`=C-D<br>`09`=D-A<br>`0a`=D-B<br>`0b`=D-C<br>`0c`=Long A<br>`0d`=Long B<br>`0e`=Long C | confirmed |

**Encode tab settings** (memory table is further down):

| Offset | Field | Value | Status |
|---|---|---|---|
| `0x0309` | 1st Tone Duration, seconds x10. Full range is 0.5-10.0s in 0.1s steps | `05`=0.5s (factory default) | confirmed |
| `0x030a` | 2nd Tone Duration, seconds x10. Full range is 0.5-10.0s in 0.1s steps | `05`=0.5s (factory default) | confirmed |
| `0x030b` | Long Tone Duration, seconds x10. Full range is 0.5-10.0s in 0.1s steps | `0a`=1.0s (factory default) | confirmed |
| `0x030c` | Gap Time, milliseconds / 100. Full range is 0-2000ms in 100ms steps | `00`=0ms (factory default) | confirmed |
| `0x030d` | Auto Reset Time, seconds x10. Full range is 0.0-25.0s in 0.1s steps | `00`=0.0s (factory default) | confirmed |
| `0x030e` | Side Tone | `0`=unchecked (factory default)<br>`1`=checked | confirmed |
| `0x0380`-`0x0383` | Memory used-flag bitmap, 32 bits for the 32 memory rows (`0x0380` bit0 = row 0 ... `0x0383` bit7 = row 31), same convention as the channel used-flag bitmap at `0x0100` | `0`=used (row has data)<br>`1`=empty (factory default has only row 0 pre-populated, so factory value is `fe ff ff ff`) | confirmed |
| `0x0b00`+N*0x10 (`N`=0-31, table spans `0x0b00`-`0x0cff` for rows 0-31) | 2-Tone Encode memory row N. Frequency fields are little-endian uint16, value = Hz x10 (different encoding from every other frequency field in this radio, which use BCD) | +0x00-0x01 1st Tone tuning word, little-endian uint16 = `floor(6000000 / freq)` - `288.0Hz`->`2083`, `500.0Hz`->`1200`, `700.0Hz`->`857` (a different reference constant than the Decode tab's `48000000/freq`)<br>+0x02-0x03 2nd Tone tuning word, same `floor(6000000/freq)` relationship<br>+0x04-0x05 1st Tone Frequency, e.g. `40 0b` = 2880 = 288.0Hz<br>+0x06-0x07 2nd Tone Frequency, e.g. `b8 79` = 31160 = 3116.0Hz<br>+0x08-0x0e Name, 7 bytes ascii space-padded<br>+0x0f unused, stays `00` | confirmed |

### 5-Tone

5-Tone config dialog. Codes are sequences of standard-defined tones (digits `0`-`9`,`A`-`F`), not
arbitrary frequencies like 2-Tone - the dialog's right-hand panel showing a tone/frequency table
per Decode Standard is just a reference display, not editable data.

| Offset | Field | Value | Status |
|---|---|---|---|
| `0x0321` | Decoding Response - dropdown offers `None/Beep Tone/Beep Tone and Respond` | `00`=None (factory default)<br>`01`=Beep Tone<br>`02`=Beep Tone and Respond | confirmed |
| `0x0322` | Decode Standard - 0-based index into the dropdown | `00`=ZVEI1 (factory default)<br>`01`=ZVEI2<br>`02`=ZVEI3<br>`03`=PZVEI<br>`04`=DZVEI<br>`05`=PDZVEI<br>`06`=CCIR1<br>`07`=CCIR2<br>`08`=PCCIR<br>`09`=EEA<br>`0a`=Euro Signal<br>`0b`=NATEL<br>`0c`=MODAT<br>`0d`=CCITT | confirmed |
| `0x0323` | Self ID length | e.g. `12345` -> `05` | confirmed |
| `0x0324` | Time Of Decode Tone, milliseconds / 10. Full range is 30-100ms in 10ms steps | `07`=70ms (factory default)<br>`08`=80ms | confirmed |
| `0x0325`-`0x0329` | Self ID digits, one raw decimal digit per byte (not BCD-packed), max 5 digits | e.g. `12345` -> `01 02 03 04 05` | confirmed |
| `0x032a` | unattributed | - | unknown |
| `0x032b` | Pretime, milliseconds / 10. Full range is 10-2550ms in 10ms steps | `14`=200ms (factory default)<br>`1e`=300ms | confirmed |
| `0x032c` | Time-Lapse After Encode, milliseconds / 10. Full range is 10-2550ms in 10ms steps | `14`=200ms (factory default)<br>`1e`=300ms | confirmed |
| `0x032d` | PTT ID Pause Time, direct seconds. Dropdown also offers `Off` and 5-75 (step not confirmed) | `0a`=10s (factory default)<br>`14`=20s | confirmed |
| `0x032e` | Auto Reset Time, seconds x10. Full range is 0.0-25.0s in 0.1s steps | `00`=0.0s (factory default)<br>`0a`=1.0s | confirmed |
| `0x032f` | First Delay, milliseconds / 10. Full range is 10-2550ms in 10ms steps | `14`=200ms (factory default)<br>`1e`=300ms | confirmed |
| `0x0330` | Side Tone | `0`=unchecked (factory default)<br>`1`=checked | confirmed |
| `0x0331` | unattributed | - | unknown |
| `0x0332` | Stop Code - dropdown offers `Off/B/C/D/F` only, no digits or A/E, standard hex-nibble encoding | `00`=Off (factory default)<br>`0b`=B<br>`0c`=C<br>`0d`=D<br>`0f`=F | confirmed |
| `0x0333` | Stop Time, milliseconds / 10. Full range is 10-2550ms in 10ms steps | `14`=200ms (factory default)<br>`1e`=300ms | confirmed |
| `0x0334` | Decode Time, milliseconds / 10. Full range is 0-2000ms in 10ms steps | `02`=20ms (factory default)<br>`32`=500ms | confirmed |

Information Code Function table, slots 1-8, 32 bytes each at `0x0500 + (N-1)*0x20`. Some fields
lock out depending on Function Option: Squelch Off allows Decoding Response but not Function
Name; Call All allows both; Emergency Alarm allows Function Name but not Decoding Response;
Remotely Kill/Stun/Wake Up force Decoding Response to Beep Tone and Respond but allow Function
Name; Group Call allows neither. Configuring a Kill or Stun code requires at least one other slot
configured as Remotely Wake Up, enforced by the vendor software.

| Offset | Field | Value | Status |
|---|---|---|---|
| slot +0x00 | Function Option - 0-based index into the dropdown | `00`=Squelch Off<br>`01`=Call All<br>`02`=Emergency Alarm<br>`03`=Remotely Kill<br>`04`=Remotely Stun<br>`05`=Remotely Wake Up<br>`06`=Group Call | confirmed |
| slot +0x01 | Decoding Response, same 3-value enum as `0x0321` above. Only meaningful for Function Options that allow it | `00`=None<br>`01`=Beep Tone<br>`02`=Beep Tone and Respond | confirmed |
| slot +0x02 | Information ID length | e.g. `111` -> `03` | confirmed |
| slot +0x03 onward | Information ID digits, one raw decimal digit per byte | e.g. `111` -> `01 01 01` | confirmed |
| slot +0x10-+0x16 | Function Name, 7 bytes ascii space-padded | e.g. `Test1  ` | confirmed |

Encode memory table, rows 0-N, 32 bytes each at `0x0d00 + row*0x20`. Codes are hex-nibble digits
(`0`-`9`,`A`-`F`) per the Decode Standard's tone table, BCD-packed 2 per byte like every other
code field in this radio - except the padding nibble on an odd digit count is `e`, not `0`.

| Offset | Field | Value | Status |
|---|---|---|---|
| row +0x00 | Encode Standard for this row - same dropdown/index as `0x0322` | `00`=ZVEI1 (factory default) | confirmed |
| row +0x01 | unattributed | - | unknown |
| row +0x02 | Encode ID length | e.g. `12345` -> `05` | confirmed |
| row +0x03 | Time Of Encode Tone, milliseconds / 10 | `07`=70ms (factory default, present even on unprogrammed rows) | confirmed |
| row +0x04 onward | Encode ID digits, BCD-packed 2 per byte, odd count padded with `e` | e.g. `12345` -> `12 34 5e` | confirmed |
| row +0x18-+0x1e | Name, 7 bytes ascii space-padded | e.g. `test   ` | confirmed |
| `0x0390`-`0x039f` | Encode memory used-flag bitmap, same convention as `0x0100`/`0x0380` (`0x0384`-`0x038f` is a separate, unattributed range) | `0`=used (row has data)<br>`1`=empty (factory default has only row 0 pre-populated, so factory value is `fe`) | confirmed |
| `0x0362` | PTT ID Starting (BOT) code length | e.g. `111`->`1E1` -> `03` | confirmed |
| `0x0364`-`0x0365` | PTT ID Starting (BOT) code digits, BCD-packed 2 per byte, odd count padded with `e`. The software auto-substitutes a repeat marker (hex digit `e`, the standard's "Repeat Tone") into 3-in-a-row identical digits before storing - typing `111` actually stores `1E1`, not the literal digits | e.g. `111` -> stored as `1E1` -> `1e 1e` (second byte's low nibble is padding) | confirmed |
| `0x0372` | PTT ID Ending (EOT) code length, same 16-byte-record shape as BOT above, EOT sits immediately after it | e.g. `222` -> stored as `2E2` -> `03` | confirmed |
| `0x0374`-`0x0375` | PTT ID Ending (EOT) code digits, same encoding as BOT above | e.g. `222` -> stored as `2E2` -> `2e 2e` | confirmed |

Special Call wizard for the Encode table (targets any row 0-99 by group number, same record format
as the Encode table above). Behavior differs per Calling Type:

| Calling Type | Behavior |
|---|---|
| PTTID | Writes the literal prefix `E6` followed by the 5-Tone Self ID digits, e.g. Self ID `12345` -> stored digits `E612345`. Independent of the Special Call's group number. Self ID is fixed at exactly 5 digits on this radio, so length-dependence is untested | confirmed |
| ANI | Writes `<Other Side ID><Interval Character><Self ID>`, e.g. Other Side ID `12345`, Interval Character `D`, Self ID `12345` -> `12345D12345`. Same enum values as `0x02e0`/`0x0322`'s DTMF/5-Tone interval characters | confirmed |
| Send Message | Writes the Other Side ID digits, independent of the message text, followed by the message as plain ASCII (not BCD-packed) - message `Hello` landed as literal ASCII `48 45 4c 4c 4f` right after the digits, and an empty message leaves those trailing bytes untouched. Digit portion = literal `E` + the string `"1"` + Other Side ID, with the same repeat-tone substitution as BOT/EOT (adjacent identical digits: the second becomes `E`), then padded to even length with a trailing `E`. Other Side ID `12345` -> `E1E2345E`; `67890` -> `E167890E`; `34567` -> `E134567E`; `11234` -> `E1E1234E` | confirmed |

### Information of Scanning

Information Of Scanning Channel dialog.

| Offset | Field | Value | Status |
|---|---|---|---|
| `0x03b0` | Scan Mode | `00`=Off<br>`01`=On | confirmed |
| `0x03b1` | Priority Channel - dropdown offers `Off/Ch1/Ch2/Ch1&2` | `00`=Off (factory default)<br>`01`=Ch1<br>`02`=Ch2<br>`03`=Ch1&2 | confirmed |
| `0x03b2` | Revert Channel - dropdown offers `Selected/Selected+TalkBack/Priority Channel1/Last Called/Last Used/Priority Channel1+TalkBack`. Not a plain 0-based sequential index - `03` is skipped | `00`=Selected (factory default)<br>`01`=Selected+TalkBack<br>`02`=Priority Channel1<br>`03`=unused/unseen<br>`04`=Last Called<br>`05`=Last Used<br>`06`=Priority Channel1+TalkBack (also forces Priority Channel at `0x03b1` to `01`=Ch1) | confirmed |
| `0x03b3` | Look Back Time A, seconds = (raw/10)+0.5. Full range 0.5-5.0s in 0.1s steps | `05`=1.0s (factory default)<br>`06`=1.1s | confirmed |
| `0x03b4` | Look Back Time B, same formula as A above | `05`=1.0s (factory default)<br>`06`=1.1s | confirmed |
| `0x03b5` | Dropout Delay Time, seconds = (raw+1)/10. Full range 0.1-5.0s in 0.1s steps | `00`=0.1s (factory default)<br>`01`=0.2s | confirmed |
| `0x03b6` | Dwell Time, same formula as Dropout Delay above | `00`=0.1s (factory default)<br>`01`=0.2s | confirmed |
| `0x03b7` | unattributed | - | unknown |
| `0x03b8` | Scan Enter Tone | `0`=unchecked (factory default)<br>`1`=checked | confirmed |
| `0x03bb` | Priority Channel 1 - raw byte is the channel number directly | e.g. channel 5 -> `05`, channel 10 -> `0a` | confirmed |
| `0x03bd` | Priority Channel 2 - same encoding as Priority Channel 1 above | e.g. channel 10 -> `0a` | confirmed |

Still needed: Revert Channel index `03` - unused by any of the 6 dropdown options, real meaning
unknown.

### Emergency Info

Emergency Alarm mode gates which of Alarm Time / Duration of TX / Duration of RX are editable in
the vendor dialog (Alarm mode enables Alarm Time only; the two Transpond modes and Both enable
Duration of TX/RX instead).

| Offset | Field | Value | Status |
|---|---|---|---|
| `0x03a0` | Emergency Alarm | `00`=Alarm<br>`01`=Transpond+Background<br>`02`=Transpond+Alarm<br>`03`=Both | confirmed |
| `0x03a1` | ENI Type Select | `00`=None<br>`01`=DTMF<br>`02`=5Tone | confirmed |
| `0x03a2` | Emergency ID - 0-based index into whichever list ENI Type selects: DTMF M1-M16 (same convention as the Channel dialog's DTMF memory slot field) when ENI Type=DTMF, or the 5-Tone Encode memory table's rows when ENI Type=5Tone | `00`=M1/row0<br>`01`=M2/row1 | confirmed |
| `0x03a3` | Alarm Time[s] - raw value, range 1-255 | e.g. `01`=1s, `02`=2s | confirmed |
| `0x03a4` | Duration of TX[s] - raw value, range 1-255; `00` when Alarm mode has it grayed out | e.g. `01`=1s, `02`=2s | confirmed |
| `0x03a5` | Duration of RX[s] - raw value, range 1-255; `00` when Alarm mode has it grayed out | e.g. `01`=1s, `02`=2s | confirmed |
| `0x03a6` | Emergency ENI Send Select | `00`=Assigned<br>`01`=Selected | confirmed |
| `0x03a8` | Emergency Channel - raw channel number | e.g. channel 5 -> `05` | confirmed |
| `0x03a9` | Emergency Cycle | `00`=Continuous (default)<br>other=cycle count, raw value 1-255 (e.g. `01`=1 cycle) | confirmed |

`0x03ac` changes once, `01`->`05`, the first time Emergency Alarm mode moves away from its factory
default - further mode changes, including switching back to the default, leave it at `05`. Purpose
unexplained.

Emergency ID selects one of the DTMF/5-Tone codes programmed elsewhere (same pattern as Optional
Signaling's DTMF slot picker in the Channel dialog). It shows empty when no matching code has been
programmed yet.

### Communication Notes

Table, displayed rows `0`-`127`. Each row: Call ID (5-digit number), Name (text field). Call ID and
Name are stored in two separate parallel arrays, sorted ascending by Call ID and packed from slot 0
- not addressed by the displayed row number. Adding `12345` to a table that already held `99999`
placed `12345` in slot 0 and moved `99999` to slot 1. Deleting an entry compacts the array (later
entries shift down to fill the gap) - there's no separate count field; "in use" is just "packed
from slot 0 until the first all-zero Call ID."

| Offset | Field | Value | Status |
|---|---|---|---|
| `0x1980`-`0x1bff` | Call ID array - slot `N` (by ascending Call ID order, not displayed row number) at `0x1980 + N*5`, 5 bytes, raw ASCII digit characters (not BCD), no padding | e.g. `12345` -> `31 32 33 34 35` | confirmed |
| `0x1c00`-`0x1fff` | Name array - slot `N`, same indexing as Call ID array, at `0x1c00 + N*8`, 8 bytes, ASCII text, null-padded | e.g. `TEST` -> `54 45 53 54 00 00 00 00` | confirmed |

### Embedded Message

Clicking this menu item shows an error dialog ("DR_X38: Please check the link.") instead of any
config screen - not accessible in this vendor software installation.

### Unattributed - not yet mapped to a feature

Bytes with no known dialog or setting behind them yet.

| Offset | Field | Value | Status |
|---|---|---|---|
| `0x0360`-`0x037f` | unattributed, mostly zero with stray `0x07` bytes | - | unknown |
| `0x0384`-`0x038f` | unattributed, same fe/ff shape as `0x0380`-`0x0383`/`0x0390`-`0x039f` (2-Tone and 5-Tone's used-flag bitmaps) - purpose unknown, not part of either confirmed bitmap | - | unknown |
| `0x03a7`, `0x03aa`, `0x03ad`-`0x03af`, `0x03b9`-`0x03ba`, `0x03bc`, `0x03be`-`0x03bf` | unattributed, small integers, gaps left over inside the now-mapped Emergency Info (`0x03a0`-`0x03ac`) and Scanning (`0x03b0`-`0x03bd`) regions | - | unknown |
| `0x3900`-`0x3fff` | Blanked/regenerated region. Every write forces this to all-zero; non-zero values return after the radio's post-write auto-reset - meaning unknown, but confirmed non-destructive | - | unknown |

## Still missing

- `0x0233` bits other than 1/3 - bit splits inferred from OR-ing behavior across tests, not
  confirmed with a proper toggle-off test.
- `0x0360`-`0x0361`, `0x0376`-`0x037f`, `0x0384`-`0x039f`, `0x03a7`, `0x03aa`,
  `0x03ad`-`0x03af`, `0x03b9`-`0x03ba`, `0x03bc`, `0x03be`-`0x03bf` - genuinely unexplored.
- 5-Tone repeat-tone (`e`) substitution rule: scanning left to right, a digit that repeats the
  previous digit becomes `E`, but two `E`s never appear back to back (the digit after an `E` is
  always written literally, even if it also repeats). `11`->`1E`, `111`->`1E1`, `11111`->`1E1E1`,
  `111111`->`1E1E1E`, `1111111`->`1E1E1E1`, `2222`->`2E2E`, `11112`->`1E1E2`, `12111`->`121E1`. One
  known exception: in the BOT/EOT field specifically, `1111` (exactly 4 ones, ending the string)
  produces `1E1E1` (5 chars) instead of the rule's `1E1E` (4 chars) - reproducible, specific to
  digit `1` at that exact run length and position. The same `1111` typed into the Encode memory
  table's Encode ID field comes out clean as `1E1E` - the anomaly is BOT/EOT-field-specific, not a
  general substitution bug.
- Channel `+0x0a` bit 1 - unattributed, always seen as `1` so far.
- `0x03ac` - changes once on the first departure from default Alarm mode, purpose unexplained.
