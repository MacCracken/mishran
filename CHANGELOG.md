# Changelog

All notable changes to **mishran** are documented here.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased] — PCM over shared memory (lifts the 2 KB TCP-window ceiling)

The two-proc audio path now moves the PCM payload **off the TCP wire and onto a
shared-memory buffer** — the audio counterpart of setu's framebuffer present. Only a
tiny control message rides TCP; the samples travel through `sys_shm`. This lifts the
old inline-`MSH_WRITE` block ceiling (256 frames / 1024 B, forced under the ~2 KB agnos
loopback recv window): a client can now hand off blocks bounded by the shm buffer, not
the window. Validated on agnos with 1024-frame (4096 B) blocks — larger than the whole
TCP window — reaching the DAC (`mishran-duplex-audio-smoke.sh`, RMS 2269 / PEAK 4448, no
deadlock). Pure userland; no kernel change (the `sys_shm_*` syscalls #71-74 already ship,
the same ones setu's present uses).

### Added

- **`src/shm.cyr` — out-of-band PCM buffers (`msh_shm_create/write/read/free`).** COPY-
  based, kernel/OS-owned, keyed by an integer id sent over the wire (no fd passing).
  `#ifdef CYRIUS_TARGET_AGNOS` → the kernel shm syscalls; Linux → a tmpfs file
  `/dev/shm/mishran-pcm-<id>`, so the whole path is Linux-first testable. Mirrors setu's
  `src/buf.cyr` 1:1.
- **`MSH_WRITE_SHM` (7) + `MSH_ACK` (8) opcodes** (`src/proto.cyr`). `MSH_WRITE_SHM`
  carries `[nframes, buf_id]` — the PCM is in the shm buffer, not on the wire; the server
  replies `MSH_ACK(buf_id)` once it has copied the block into the stream ring, releasing
  the buffer for reuse.
- **Single-buffer credit/ACK flow control** (`src/server.cyr` `MshClient`). The client
  owns one shm buffer (`MSH_SHM_BLOCK_FRAMES = 2048`) and reuses it only after the ACK
  (`msh_client_wait_ack`), so the reuse is race-free (the server copies out *before*
  acking) and a full server ring simply defers the ack — automatic real-time pacing with
  no socket-desync risk. `msh_client_write` splits large writes into ≤ block-size shm
  handoffs. `MshClient` grew to 48 B (buf id + in-flight flag); `msh_client_close` frees
  the buffer.
- **Server-side shm consume + latch** (`src/server.cyr` `msh_server_service`,
  `msh_srv_ack`). `MSH_WRITE_SHM` copies the block out of shm into the ring and ACKs; if
  the ring is full it **latches** the block (frame count + shm buf id in the client slot,
  no ack) and the PCM stays safe in the client-owned buffer until a later tick drains the
  ring and the pending-retry consumes + ACKs it. The per-client slot grew 24 → 32 B
  (`MSH_SLOT_SZ`, `pending_buf` at +24).
- **`programs/shm_probe.cyr`** — host proof: shm round-trip · client write → `WRITE_SHM`
  → ring + ACK (reuse-guarded) · backpressure latch → pending-retry → ACK. All PASS.

### Changed

- **`programs/mishclient.cyr`** streams **1024-frame** blocks (was 256) over the shm path
  — deliberately above the TCP window, to exercise the ceiling lift.
- **cyrius pin 6.4.10 → 6.4.49** — the `sys_shm_*` floor is 6.4.33; 6.4.49 matches vani
  (mishran's vendored sink source). Feature-floor bump, not chasing.
- The legacy inline `MSH_WRITE` path is **unchanged** (`serve_probe` still PASS) — the
  server still handles it; only the client API (`msh_client_write`) moved to shm.

### Known

- **Long-running daemon heap growth (pre-existing, not from this change).** The serving
  loop allocs per-poll message structs + per-block PCM scratch from the bump allocator
  with no reclamation; a forever-running mixer daemon (`mishrand`) would eventually OOM.
  Masked by bounded smokes (which exit in ~1.5 s). Tracked for a focused
  reusable-buffer refactor; surfaced by a 2-lens adversarial-verify pass whose flow-
  control review otherwise found the shm path sound.

## [0.4.1] - 2026-07-10 — two-proc audio on agnos (cooperative yield)

Two concurrent processes can now share the one hardware writer through the mixer on
agnos — an app streams to the daemon over the loopback wire while the daemon mixes to
vani, with no deadlock. The fix is pure userland (no kernel-logic change); see the
agnos planning note `docs/development/planning/blocking-syscall-concurrency.md`.

### Added

- **Two concurrent procs play through the mixer on agnos — the cooperative-yield fix.**
  The two-proc audio path (a separate app streaming to the mixer daemon over the
  loopback wire) deadlocked on agnos: a blocking `audio_write` in the pump — and the
  client's blocking `sleep_ms` send-backoff — held the CPU with preemption disabled,
  and the kernel *cannot* preempt a blocking syscall (the shared per-CPU syscall kstack,
  the serial-kstack invariant), so whichever proc blocked starved the other. Fixed
  entirely in userland (no kernel-logic change) with the proven cooperative-yield
  pattern (`sched_yield` #44, the 1.53.9 setu-present cure):
  - `src/route.cyr` — new **`msh_router_pump_nb`** for multi-proc daemons: on agnos,
    emit a block **only when the DAC ring has room for a whole block** (`audio_avail`
    #69), write it NONBLOCK (`audio_write_nb` #66), else return without mixing so the
    daemon stays responsive to its clients; never a blocking, pump-hogging write (Linux
    delegates to the blocking pump). The original `msh_router_pump` stays **blocking** —
    single-proc feed loops (mishtone) rely on it to pace to real time. `mishrand` +
    `mishduplex` now pump via `_nb`; mishtone (single-proc, RMS 3700) is unchanged.
  - `src/transport.cyr` — the agnos send/recv would-block backoffs (`msh_write_all`,
    `msh_read_blk`) now `sys_sched_yield` instead of `sys_sleep_ms`, so a stalled proc
    donates its slice to the peer that must drain it instead of spin-sleeping preempt-off.
- **Server-side flow control — WRITE-payload backpressure latch.** `src/server.cyr`
  `msh_server_service` now defers a WRITE payload when the target stream ring can't hold
  it: it latches the frame count (`pending_frames` in the client slot) and leaves the
  bytes in the socket so TCP backs up and the client's write blocks — real-time pacing
  without dropping audio or desyncing the frame stream (the latched count gives the
  payload's exact length when the ring drains). A chunk larger than the whole ring is
  read lossily rather than deadlocking. Pairs with the client's `sched_yield` send
  backoff so the two procs stay in lockstep.
- **Two-proc audio proof — `programs/mishduplex.cyr` (server) + `programs/mishclient.cyr`
  (client).** The mixer server binds loopback:7701 + opens the vani sink, then
  `spawn_path`'s the client (the desktop server-first ordering — bind before the client
  connects); the client streams a square-wave tone over the real TCP wire; the server
  mixes it to vani. Driven on agnos by a `MISHRAN_DUPLEX_SELFTEST` kernel hook (agnos
  `kernel/core/main.cyr`, **post-`sched_active`** so two procs actually run concurrently)
  and captured by `agnos scripts/mishran-duplex-audio-smoke.sh`: **RMS 2116, PEAK 4448**
  (thresholds 800/3000), `hda: stream running`. PCM is chunked below the 2 KB agnos TCP
  loopback window (256 frames/write) + `sched_yield`-paced so `sock_send` #48 fits the
  recv buffer instead of blocking preempt-held — a wire constraint, not a mixer one (see
  Notes). Proves: two concurrent ring-3 procs, real loopback handshake, cooperative
  mix-to-DAC, no deadlock, no starvation.
- **Agnos audio proof — the mixer plays on the sovereign kernel.** `programs/mishtone.cyr`
  opens the vani sink via an `MshRouter`, registers two app streams at different Q8 gains
  (440 Hz @ unity + 660 Hz @ -6 dB), and mixes them frame-by-frame down to the sink. Driven
  on agnos by a `MISHRAN_AUDIO_SELFTEST` kernel hook (agnos `kernel/core/main.cyr` +
  `scripts/build.sh`) and captured by `agnos scripts/mishran-audio-smoke.sh`: the mixed tone
  reaches the HDA DAC — **RMS 3684, PEAK 6671** (thresholds 800/3000), `hda: stream running`.
  Proves the whole path — fan-in + gain + `msh_router_pump` → vani `audio_*` → kernel
  `snd_*` #64-69 — on-device, not just numerically.

## [0.4.0] - 2026-07-10 — the routing server

Apps can now route audio THROUGH mishran instead of grabbing the single hardware writer:
a TCP-loopback server + the app client library + a runnable daemon.

### Added

- **`mishrand` — the runnable mixing daemon** (`programs/mishrand.cyr`). Opens the vani sink,
  listens on loopback:7701, and runs the real-time loop: `msh_server_poll` (accept + drain each
  client's PCM into its ring) + `msh_router_pump` (mix every stream → the sink; `audio_write`
  paces the loop to real time). Composes the two proven halves — serve_probe (serving) and
  pump_probe (pumping). No audio device → a clean degrade (reports + exits 2, no busy-spin).
  Builds `--agnos`.
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
