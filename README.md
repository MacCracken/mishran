# mishran

**मिश्रण — mixing / mixture**

Version: 0.1.0

Sovereign software **audio mixer + routing daemon** for AGNOS, written in
pure Cyrius. mishran is the multi-stream fan-in layer the agnostic audio
graph lacks: many per-app PCM streams flow in, get mixed / resampled /
routed, and leave through a **single** output writer to
[`vani`](../vani) — Linux ALSA/OSS or the agnos `snd` syscalls (#64-69).

It is PulseAudio / PipeWire's job done sovereignly, with **no daemon
bloat** and no C.

## Why it exists

The agnos `snd` surface is **single-writer** — there is exactly one
`snd_id` slot, and a second `snd_appl` writer was deliberately rejected.
So a desktop with more than one sounding app cannot just have each app
open the device. mishran **owns that single hardware writer** and fans
every app's stream in behind it: each application talks to mishran, and
mishran talks to the device. That is the whole point of the mixer.

## Scope

**v0.1.0 = scaffold.** This is a buildable skeleton, not a finished mixer.
It compiles green (the `programs/smoke.cyr` banner links the full include
chain) and lays down the real type surface:

- `src/error.cyr` — `MishranErr` codes + `mishran_err_*` helpers.
- `src/stream.cyr` — `MshStream`: id, `MshFormat` (S16/F32), sample rate,
  channels, and a single-producer/single-consumer sample ring buffer
  (`msh_stream_new` / `msh_stream_write` / `msh_stream_read`, working S16).
- `src/mix.cyr` — `msh_mix`: a **real** (simple) integer sum-and-clamp
  fan-in over N S16 streams into one output frame buffer; `msh_resample`
  is a stub.
- `src/route.cyr` — `MshRouter`: the **single-writer** owner of the one
  vani sink plus the registered stream table (`msh_router_new` /
  `msh_router_add` / `msh_router_remove` / `msh_router_pump`). The pump
  loop stops short of the device write until the sink is wired.

Deferred to later cycles: per-stream gain, proper resampling, the F32
path, and opening the real `vani` sink in `msh_router_pump` (see
**Consumers** / the deferred cross-dep note below).

## Place in the stack

```
  app  app  app  app          (per-app PCM producers)
   \    |    |    /
    v   v    v   v
   MshStream ring buffers      (src/stream.cyr)
        |
        v
     msh_mix  ── fan-in: sum + clamp + (resample) ──  (src/mix.cyr)
        |
        v
   MshRouter  ── owns THE single writer ──            (src/route.cyr)
        |
        v
      vani  ── ALSA/OSS (Linux) | snd #64-69 (agnos) ── (deferred dep)
        |
        v
    hardware audio device
```

mishran sits **above** vani (it is a vani consumer) and **below** every
sounding application. It is a system library in the Sanskrit/Hindi naming
lane, a sibling of the other AGNOS audio pieces (`vani`, `shravan`,
`naad`).

## Consumers

- The AGNOS desktop / compositor (`aethersafha`) and any client that wants
  more than one app to make sound at once — they open an `MshStream`
  against a running `MshRouter` instead of the device directly.
- Pulls `dist/mishran.cyr` via a `[deps.mishran]` entry pointing at a git
  tag once the mixer is wired.

### Deferred cross-dep: `vani`

The output **sink** (`vani`, `../vani`) is **not wired at scaffold stage**.
`msh_router_pump` mixes into an internal buffer and returns
`MISHRAN_ERR_NO_SINK` instead of writing to a device. Add
`[deps.vani]` `path="../vani"` (or a git tag) to `cyrius.cyml` when
`msh_router_pump` opens the real hardware sink.

## Build

```sh
cyrius deps                                        # vendor stdlib into lib/
cyrius build programs/smoke.cyr build/mishran-smoke
./build/mishran-smoke
```

## License: GPL-3.0-only
