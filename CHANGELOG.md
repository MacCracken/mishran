# Changelog

All notable changes to **mishran** are documented here.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.2.0] - unreleased

### Added

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
