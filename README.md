# mishran

**मिश्रण — mixing / mixture**

Version: 0.5.5

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

**v0.4.x = a working mixer + routing server.** mishran fans many per-app
S16 streams into one mixed writer to a real `vani` sink, with per-stream
and master gain, linear resampling, and a routing server so separate app
processes can share the one hardware writer. Proven on agnos: a
**single-proc** mixed tone (`mishtone`) — an honest test, no rigging.

> ⛔ **The two-proc claim is RETRACTED (2026-08-03).** This paragraph used to
> also claim a proven "**two-proc** client→loopback→mixer→vani→HDA tone
> (`mishduplex` + `mishclient`)". That was a **FALSE GREEN**: the only proof,
> `mishran-duplex-audio-smoke.sh`, ran under a `MISHRAN_DUPLEX_SELFTEST` kernel
> hook that assigned `net_ip = 0x7F000001`; agnos puts `net_ip` in an outbound
> SYN's SOURCE, so before `net_src_for` (agnos 1.56.34) the client's loopback connect could not
> match a 4-tuple and the two procs could not talk at all. Hook and smoke are
> **deleted**. **TCP-on-loopback is not the local IPC transport** — the
> replacement is the agnos socket (`anu`), agnos
> `docs/development/planning/ipc.md` §9-§10. The routing server, the client API
> and the cooperative-yield pump are unaffected as *code*; they need re-targeting
> onto `anu` and re-proving there. Do not re-add the hook under any name.

- `src/error.cyr` — `MishranErr` codes + `mishran_err_*` helpers.
- `src/stream.cyr` — `MshStream`: id, `MshFormat` (S16/F32), sample rate,
  channels, per-stream Q8 gain, and a single-producer/single-consumer
  sample ring (`msh_stream_new` / `msh_stream_write` / `msh_stream_read`).
- `src/mix.cyr` — `msh_mix`: integer sum-and-clamp fan-in over N S16
  streams with per-stream gain + sink-rate reconcile; `msh_resample` is a
  real linear resampler.
- `src/route.cyr` — `MshRouter`: the **single-writer** owner of the one
  vani sink plus the registered stream table. `msh_router_pump` (blocking,
  single-proc paced) and `msh_router_pump_nb` (cooperative, non-blocking —
  for a multi-proc daemon on agnos) mix + write to the device.
- `src/proto.cyr` / `transport.cyr` / `server.cyr` — the TCP routing
  server + `msh_client_*` app API + a WRITE-payload backpressure latch.

Deferred: the F32 mix path (S16 is the working path) and a router-owned
thread for the pump loop.

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
