# Changelog

All notable changes to **mishran** are documented here.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added

- **Routing server + app client — apps route THROUGH the mixer.** mishran gains a TCP-loopback
  server (port 7701) so many apps share the one vani writer instead of each grabbing it — the
  reason the mixer exists. New modules: **`proto.cyr`** (wire messages — `HELLO`/`WELCOME`/
  `WRITE`/`GAIN`/`BYE`, 8-byte words plus a raw S16 PCM payload after `WRITE`), **`transport.cyr`**
  (listen/accept/connect/send/recv over cyrius `net.cyr` — cross-platform: Linux BSD sockets +
  agnos TCP band #47/#56/#57, modeled 1:1 on setu's transport), and **`server.cyr`** — an
  `MshServer` that accepts clients, maps each to one `MshStream`, and dispatches
  `HELLO`→register / `WRITE`→ring / `GAIN`→volume / `BYE`→drop, plus the app-facing
  `msh_client_connect`/`_write`/`_gain`/`_close` library a player like jalwa links.
  `msh_server_poll` services every client non-blocking alongside `msh_router_pump`. Verified by
  `programs/serve_probe.cyr` over real loopback (single-process, no fork): connect → HELLO/WELCOME,
  a WRITE + PCM payload lands in the router's stream ring (samples = 1234), and GAIN is applied
  (= 128) — all green; builds `--agnos`. Added `result` + `net` to the stdlib set.

## [0.3.0] - 2026-07-10

First tagged release — the pre-1.0 mixing kernel. Rolls up the previously-unreleased
0.1.0 scaffold and 0.2.0 sink/gain work plus the resampler below.

### Added

- **Linear resampler + per-stream rate reconciliation.** `msh_resample` is now a real
  per-channel linear interpolator (integer phase accumulator, no float): it up/downsamples
  an S16 block from any input rate to the sink rate. `msh_mix` gained a `sink_rate` argument
  and reconciles each stream to it — a stream at 44.1k (or any rate) is resampled to the
  router's 48k before the gain-scaled sum, so heterogeneous app streams fan in correctly
  (what a player like jalwa needs). Verified by `programs/resample_probe.cyr`: 2× upsample
  interpolates exactly, 2× downsample decimates, and a 24k stream through a 48k sink stays
  constant. Anti-alias filtering on downsample + an F32 path remain TODO.
- **Per-stream + master gain — "control outputs."** Each `MshStream` now carries a
  Q8 fixed-point gain (`+64`; 256 = unity, 128 = -6 dB, 512 = +6 dB, 0 = mute), and
  `msh_mix` scales every stream by its gain during fan-in. The `MshRouter` carries a
  **master** output gain (`+48`) applied to the mixed block in `msh_router_pump` via the
  reusable, sink-free `msh_apply_gain`. `msh_stream_set_gain` / `msh_router_set_gain` are
  the control surface — so a desktop can set each app's level and a master level before
  the single hardware write. Integer-only (matches the S16 mixing ethos; no float).
  Verified by `programs/gain_probe.cyr`: per-stream gain, master gain, the S16 clamp on a
  boosted overlap, and mute are all numerically exact; builds `--agnos` (the gain math is
  platform-identical, so the host proof carries).
- **vani output sink wired — first real device write.** The `MshRouter`'s
  single-writer sink slot now opens and drives a real `vani` PCM device:
  `msh_router_open(r, card, device)` (`audio_open_playback` → `audio_set_params`
  at the router's rate/channels/S16_LE → `audio_prepare`), `msh_router_pump`
  streams the mixed block through `audio_write` with `-EPIPE` XRUN recovery
  (re-prepare + replay once — the cyrius-doom / dhvani idiom, since the core
  `audio_*` shim leaves recovery to the caller), and `msh_router_close`
  drains + closes. `MISHRAN_ERR_SINK_WRITE` distinguishes a hard write failure
  from `MISHRAN_ERR_NO_SINK`.
- **`vendor/vani-core.cyr`** — vani's `dist/vani-core.cyr` core profile
  (`audio_*` PCM shim) vendored as a single committed file, compiled against
  mishran's own stdlib list. Not a git `[deps.vani]` (that drags in vani's full
  `[deps].stdlib` — yukti/patra/chrono — the shim never needs). Mirrors
  cyrius-doom / -polyomino / -bb. Provenance: vani 1.0.0.
- **`programs/pump_probe.cyr`** — a silent real-hardware probe: router +
  one silent stream → open sink → pump N ticks → drain + close. **Confirmed
  working on real hardware** (2026-07-06): sink open → pump → drain clean.
  Degrades clean (exit 2) when no audio hardware / `audio`-group access.

### Changed

- **cyrius pin `6.4.7` → `6.4.10`** (ecosystem parity with vani 1.0.0).

## [0.1.0] - unreleased

### Added

- Repo scaffolded: pure-Cyrius software audio mixer + routing server that
  fans many per-app PCM streams into one mixed/resampled writer to vani —
  buildable skeleton with `MshStream` ring buffers (S16), a real
  sum-and-clamp `msh_mix` fan-in, and the single-writer `MshRouter`
  (`error.cyr` / `stream.cyr` / `mix.cyr` / `route.cyr`; `smoke.cyr` links
  the full include chain green). Real resampling, per-stream gain, the F32
  path, and the `vani` sink are deferred to later cycles.
