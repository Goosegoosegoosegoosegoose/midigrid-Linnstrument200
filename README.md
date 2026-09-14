# midigrid-Linnstrument200

**Goal:** turn a LinnStrument 200 + Norns (running on a [Freepoet XL Shield Pro](https://freepoet.io/), an open-source Norns-compatible sound computer) into a central performance/control brain — one surface to play, loop, and eventually sequence multiple synths and MPE gear live, instead of juggling each device separately.

## Current status

**Confirmed working on hardware (2026-09):**

- **Left split** — acts as a [midigrid](https://github.com/llllllll/midigrid) grid controller for [MLRE](https://github.com/thisiswhatidonow/mlre) (looping / audio mangling on Norns).
- **Right split** — a genuine MPE (MIDI Polyphonic Expression) playing surface, entirely custom-built in `linnstrument.lua.txt` (see below). Notes, pitch bend, timbre, and per-note pressure all confirmed audible on both the Nymphes and Micromonsta 2.
- **Device-select column (17)** — slots 1-4 pick which output device(s) the right split plays through: Nymphes, Micromonsta 2, both at once, or the iPad. Switching devices auto-silences whatever was playing so nothing gets stuck.
- **Octave-preset buttons (column 17, slots 5-8)** — four fixed presets (+2/+1/-1/-2 octaves) shift everything played on the right split after the press; pressing the active one again resets to no shift. Lit yellow/white so they read as a distinct control group from the device-select pads above them.
- **Per-note MPE** — each held note on the right split gets its own dynamically-allocated MIDI channel, with pitch bend, pressure, and timbre forwarded per-note (truly per-column, not just per-row) to the active device(s).
- **Legato slide** — sliding a held finger across column boundaries without lifting is one continuous glide (pitch bend accumulating from wherever the gesture started) instead of a run of retriggered blips. Toggle via `right_split_legato_slide`.
- **iPad bridge** — confirmed working: routed through a CME U6MIDI Pro (Norns side) + CME U2MIDI Pro (iPad side), the iPad receives MIDI in AUM.

**Bugs found and fixed getting there** (see the header comment in `linnstrument.lua.txt` for full detail):

1. Pitch bend/pressure/timbre never arrived at all at first — LinnStrument's "User Firmware Mode" doesn't send any continuous data per row until it's explicitly turned on via CC 9–12 sent *to* the LinnStrument. Fixed in `_activate_continuous_data`.
2. The LinnStrument's Z setting sends poly (key) pressure, but the Micromonsta 2 didn't respond to it audibly even though it arrived correctly. Fixed by re-encoding it as channel pressure on the note's own MPE channel — musically identical once every note has its own channel, and something more synths listen for.
3. Even after (1), pitch bend still never arrived: it turns out User Firmware Mode does **not** send X (pitch) as a standard MIDI pitchbend message at all — it sends a 14-bit absolute board position as a CC pair (MSB on CC 0-25, LSB on CC 32-57, column baked into the CC number). Y (timbre) is likewise a plain CC per column (CC 64-89), not a fixed CC74. Verified directly against the LinnStrument firmware source. Both are now decoded per-column and pitch bend is computed as a delta from a baseline captured on note-on — see `_handle_right_split_x_cc` / `_handle_right_split_y_cc`.
4. Columns 18-20's Note Lights kept coming out wrong on reset — the mode-switch SysEx was immediately followed by ~200 more CC messages with no pause, outrunning the LinnStrument's own processing of the SysEx. Fixed by running the reset sequence as a clock coroutine with small pauses between bursts.

**Resolved (previously a known limitation):** earlier versions of this file could only read pitch bend / timbre at per-row granularity (two notes on the same row couldn't be told apart), because of the wrong assumption in bug #3 above. Now that X and Y are correctly decoded per-column, all three expression dimensions (pitch, timbre, pressure) are genuinely per-note.

**Open / untested:**

- Whether CC74 (timbre, currently mapped to the Nymphes' filter cutoff) or CC1 (mod wheel) gives a better per-note expression result — configurable via `right_split_timbre_cc`, one line to change.
- Exact semantics of CC9 "slide mode" sent as part of the continuous-data activation — included per the LinnStrument firmware source but unconfirmed in practice.
- Legato slide detection can't perfectly distinguish a genuine slide from a deliberately-pressed adjacent-column dyad (both look identical at the raw column/row level) — an uncommon voicing in this 4ths tuning, but a place to add time-based gating later if it matters in practice.

### `linnstrument.lua.txt`

A patched drop-in replacement for midigrid's stock LinnStrument device file (original by @nightmachines, thanks to @jaggednz). Drop it into midigrid's device directory on Norns (typically `dust/code/midigrid/lib/devices/`) and select it as the LinnStrument's device profile.

Column 17 layout (top pad → bottom pad):

| Slot | Function | Notes |
|------|--------|-------|
| 1 | Nymphes | direct USB |
| 2 | Micromonsta 2 | direct USB (`MicroMonsta 2` — capital M, space before 2) |
| 3 | iPad (via U6) | confirmed working — routed through CME U6MIDI Pro / U2MIDI Pro into AUM |
| 4 | Both (Nymphes + Micromonsta 2) | sends to both at once, same pitch/channel |
| 5 | Octave +2 | toggles; notes played after the press are shifted, already-held notes unaffected |
| 6 | Octave +1 | toggles |
| 7 | Octave -1 | toggles |
| 8 | Octave -2 | toggles |

## Hardware / signal chain

**Control (MIDI/USB):** LinnStrument 200, Nymphes, Micromonsta 2, and a CME U6MIDI Pro (the iPad's bridge) all plug directly into Norns' own USB ports — no hub. (A powered USB hub caused Norns to crash on connection, likely power backfeed from the hub's own supply; plugging everything directly into Norns fixed it.) The U6MIDI Pro connects via 5-pin MIDI cable to a CME U2MIDI Pro on the iPad side, which the iPad reads inside AUM.

**Audio:** Nymphes and Micromonsta 2's direct analog outs, plus the iPad's audio (AUM → USB-C → UGREEN Revodok Pro dock → Focusrite Scarlett 2i2 → analog line out), all join at a **Behringer MX400** mixer, which feeds Norns' audio input. Norns' output feeds the main mixer.

- **LinnStrument 200** — left split as the grid, right split as the MPE surface
- **Norns (Freepoet XL Shield Pro)** — runs MLRE + midigrid, the actual brain

## Roadmap

Toward the "one surface controls everything" goal:

- **Confirm timbre CC choice** (CC74 vs CC1) and CC9 slide-mode behavior on hardware.
- **Sequencing, not just looping** — MLRE currently handles looping/mangling; extending control beyond that into sequencing multiple machines from the LinnStrument is the next big step toward "central brain."
- Revisit legato-slide detection if the dyad/slide ambiguity turns out to matter in practice.
