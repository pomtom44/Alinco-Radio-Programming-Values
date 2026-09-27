# Radio Programming Values

Byte-level memory maps for radios programmed over a serial clone cable, documented well enough
that someone else can write their own programming tool without redoing the reverse-engineering.

## Table of Contents

- [About](#about)
- [Radios Documented](#radios-documented)
- [Disclaimer](#disclaimer)
- [How It's Made](#how-its-made)
- [Repo Structure](#repo-structure)
- [Table Format](#table-format)
- [Contributing](#contributing)
- [License](#license)

## About

Most cheap/OEM radios only have one way to program them: a vendor Windows app, usually old,
usually undocumented, sometimes barely functional. There's no public spec for the wire protocol
or the memory layout, so anyone wanting to automate programming (batch-provisioning a fleet,
scripting a repeater/digipeater build, or just not wanting to click through a GUI 200 times) has
to reverse-engineer it from scratch. This repo is that reverse-engineering, written down, so the
next person doesn't have to.

## Radios Documented

- [Alinco DR-138 MKII](radios/alinco-dr138-mkii.md) - VHF mobile, ERW-7 clone cable

## Disclaimer

- **Nothing here is official documentation.** It's reverse-engineered from observed behavior, not
  from a vendor spec. Treat every entry as a hypothesis to verify, not a guarantee - even the ones
  marked `confirmed` (see the Status tags below for what that word actually means here).
- **Test before you trust it.** If you're building a tool against these tables, validate against
  your own radio before relying on it for anything that matters. A field that behaves one way on
  the unit these were captured from may behave differently on a different firmware revision,
  region variant, or clone-brand board.
- **You can damage a radio by writing bad data to it.** Writing memory you don't understand -
  especially in unmapped/`unknown` regions - carries real risk: bricking the radio, corrupting
  calibration data, or putting it in a state only the vendor tool (or a full factory reset) can
  recover from. Work on a unit you can afford to have go wrong, and keep a known-good backup image
  before you start.
- **This is not legal/compliance advice.** Radio transmission is regulated. Frequency, power, and
  duplex/offset fields in particular can put a radio outside its type-approved operating
  parameters or outside what your license/region permits. That's on you to check, not this repo.

## How It's Made

Real hardware, a real vendor-cable connection, and a transparent serial sniffer sitting between
them:

1. Sit the sniffer between the real clone cable and the radio, with the vendor software talking
   through a virtual null-modem pair into one end of it. Every byte in both directions gets logged
   with a timestamp - this is what confirms the wire protocol (handshake, block read/write shape,
   checksums) without guessing.
2. For the memory map: start from a known baseline, change exactly one setting in the vendor
   software, write it to the radio, capture just that write session. Repeat for the next setting.
   Diffing each capture against the one before it isolates precisely which byte(s) that one
   setting touches - usually to a single byte or a small run of them.
3. Where possible, sampled values are also cross-checked directly against the vendor software's
   own settings dialogs, so the label and the value are both known for certain, not just the byte
   location.

Claude (Anthropic's AI) was used throughout as a reverse-engineering assistant: writing the
sniffer/diff tooling, spotting patterns across capture logs, and drafting these tables from the
results. Every byte/bit meaning here traces back to an actual capture from real hardware, not to
the AI's own guesses about what a radio "probably" does - but the write-up itself went through an
AI, and AI-assisted work makes mistakes that look plausible. See the Disclaimer above before
relying on any of it.

## Repo Structure

One markdown file per radio under `radios/`, covering everything for that radio: protocol notes,
then the full memory map in byte order, grouped by region.

Add a new radio by copying that shape. If it shares a protocol family with one already here
(common with rebadged/OEM radios), link back to the shared parts instead of duplicating them.

## Table Format

| Column | Meaning |
|---|---|
| Offset | Absolute address, or `+0xNN` relative to a channel record's start for per-channel settings |
| Field | Name matching the vendor software's UI label, where known |
| Value | Every byte value known or guessed for this field, each paired with the setting name it corresponds to, one per line (e.g. `00`=Freq / `01`=Channel / `02`=Name) |
| Status | `confirmed` / `unconfirmed` / `unknown`, see below |

Status tags:
- `confirmed` - isolated cleanly, or matched directly against a vendor-UI label and value.
- `unconfirmed` - a value or option is listed but hasn't been independently tested - inferred from
  context, a packed byte's OR-ing behavior, or general radio conventions. Validate before trusting.
- `unknown` - byte location is known to hold *something*, but no test has explained it yet.

Within a Value list, not every entry is necessarily tested - untested ones are marked `(guess)`
inline. Unless a table says an option set is complete, assume the radio accepts more values than
are listed.

## Contributing

- Filling in missing options for an existing setting: run one more single-setting-change capture,
  diff it against a known state, add it to the Value list.
- New radio: copy the file shape. Doesn't need to be complete - a protocol writeup with an empty
  settings table is still useful to the next person.
- Resolving an `unconfirmed` or `unknown` entry: usually just needs one more targeted test (e.g.
  toggling a suspected bit independently rather than always stacking it on top of other changes).

## License

No license chosen yet.
