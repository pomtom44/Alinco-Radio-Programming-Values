# Alinco DR-138 MKII

VHF mobile, 144–148MHz amateur band. Programmed over Alinco's ERW-7 clone cable using the vendor's
own `DR_X38.exe` Windows software — no other official programming path exists.

**Not the same radio as the DR-138T.** Same cable, similar model name, different write protocol —
see `alinco-dr138-mkii_protocol.md` for the exact wire-frame difference. Don't carry notes between
the two.

Board/chip identifies itself as `DJ-138` over the wire — that's expected, not a typo.

## What's in this folder

- `alinco-dr138-mkii_protocol.md` — byte-exact wire frames and full memory map, every offset from
  `0x0000` to `0x3fff` accounted for (confirmed value, or explicitly `unknown`).
- `alinco-dr138-mkii_write_factory_defaults.py` — example programmer.
- `alinco-dr138-mkii_factory_default_0100_3ff0.bin` — raw bytes captured from a real "write factory
  defaults" session, `0x0100`–`0x3ff0`, address-ordered, no gaps. Primary source for the factory
  values in the protocol doc — if the two ever disagree, trust this file.

## Open question

One region of the captured factory-default session doesn't match already-verified data for this
radio: the 5-Tone Encode used-flag bitmap (`0x0390`) and the leftover content sitting in Encode
memory row 0 (`0x0d00`). The protocol doc keeps the previously-verified value (`0x0390` = `fe`) —
the capture is the one in question, not the doc. Needs a clean re-capture (fresh software launch,
no project file loaded first) before that region of the `.bin` can be trusted.

## Safety notes specific to this radio

- Every write blanks `0x3900`–`0x3ff0` and forces `0x0160` to `0xff`, regardless of what changed —
  expected radio behavior, not data loss.
- Skipping the mandatory `0x0040` probe read doesn't break the session, but the radio won't
  auto-reset after a write — you'll need a manual power-cycle for the write to actually take.
- Boot-strap password protection ships **off** by default (`0x0238` = `00`), with `"12345"` sitting
  in the password field either way (`0x0210`-`0x0214`) — enabling it later doesn't need a fresh
  password, just flipping that one byte.
