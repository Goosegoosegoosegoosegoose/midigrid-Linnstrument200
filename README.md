# midigrid-Linnstrument200

**Goal:** turn a LinnStrument 200 + Norns (running on a [Freepoet XL Shield Pro](https://freepoet.io/), an open-source Norns-compatible sound computer) into a central performance/control brain — one surface to play, loop, and eventually sequence multiple synths and MPE gear live, instead of juggling each device separately.

## Current status

**Working now:**

- **Left split** — acts as a [midigrid](https://github.com/llllllll/midigrid) grid controller for [MLRE](https://github.com/thisiswhatidonow/mlre) (looping / audio mangling on Norns).
- **Right split** — a genuine MPE (MIDI Polyphonic Expression) playing surface, entirely custom-built in `linnstrument.lua.txt` (see below), routed live to outboard synths.
- **Device-select column (17)** — 8 slots, each pad picks which output device(s) the right split plays through: currently Nymphes, Micromonsta 2, or both at once. Switching devices auto-silences whatever was playing so nothing gets stuck.
- **Per-note MPE** — each held note on the right split gets its own dynamically-allocated MIDI channel, and pitch bend / pressure / aftertouch are forwarded per-note to the active device(s).

**Known limitation:** LinnStrument's "User Firmware Mode" (required for the left split to act as a raw grid) overrides the whole instrument's MIDI behavior, so pitch bend / channel pressure on the right split can only be read at per-row granularity, not per-note. Two notes held on the *same* row can't be told apart for those two parameters — full detail is in the header comment of `linnstrument.lua.txt`. Poly key pressure (aftertouch) doesn't have this problem, since it carries a note number.

### `linnstrument.lua.txt`

A patched drop-in replacement for midigrid's stock LinnStrument device file (original by @nightmachines, thanks to @jaggednz). Drop it into midigrid's device directory on Norns (typically `dust/code/midigrid/lib/devices/`) and select it as the LinnStrument's device profile.

Output device mapping (column 17, top pad → bottom pad):

| Slot | Device | Notes |
|------|--------|-------|
| 1 | Nymphes | direct USB |
| 2 | Micromonsta 2 | direct USB (`MicroMonsta 2` — capital M, space before 2) |
| 3 | iPad (via U6) | not yet wired — see Roadmap |
| 4 | Both (Nymphes + Micromonsta 2) | sends to both at once, same pitch/channel |
| 5–8 | unused | placeholders, pads stay dark until configured |

## Hardware / signal chain

- **LinnStrument 200** — left split as the grid, right split as the MPE surface
- **Norns (Freepoet XL Shield Pro)** — runs MLRE + midigrid, the actual brain
- **Dreadbox Nymphes** and **Micromonsta 2** — hardware synths, both with native MPE support
- Audio from the Nymphes and Micromonsta routes through a **Behringer MX400** mixer into Norns' input; Norns' output feeds the main mixer

## Roadmap

Toward the "one surface controls everything" goal:

- **iPad bridge** — route slot 3 through a CME U6MIDI Pro (Norns side) + CME U2MIDI Pro (iPad side), so the iPad becomes another playable/sequenceable target from the right split.
- **On-hardware verification** of the per-note expression forwarding — confirming whether the LinnStrument's Z setting is actually sending key (poly) pressure vs. channel pressure, since that determines same-row note independence.
- **Sequencing, not just looping** — MLRE currently handles looping/mangling; extending control beyond that into sequencing multiple machines from the LinnStrument is the next big step toward "central brain."
- Wiring up output slots 5–8 as more gear gets added to the rig.
