# Changelog

All notable changes to **mishran** are documented here.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

> ⛔ **RETRACTION NOTICE — 2026-08-03. Every "validated on agnos" claim in this file that cites
> `mishran-duplex-audio-smoke.sh` is a FALSE GREEN.** That smoke built its kernel with the
> `MISHRAN_DUPLEX_SELFTEST` hook, which assigned `net_ip = 0x7F000001` in the kernel before
> launching the peer. agnos puts `net_ip` in an outbound SYN's SOURCE, so a client dialling
> `127.0.0.1` normally gets a SYN-ACK that `tcp_find_conn` can never match; the hook forced
> src == dst so the handshake closed **only under the hook**. On an ordinary boot mishran's
> two-proc loopback path could not connect at all. The hook, its `build.sh` define and the
> smoke script were all **deleted 2026-08-03**. TCP-on-loopback is not the local IPC transport —
> the replacement is the agnos socket (`naadi`), agnos `docs/development/planning/ipc.md` §9-§10.
> Individual claims below carry inline markers. History is preserved deliberately; do not cite it
> as proof, and do not re-add the hook under any name.

## [0.5.4] - 2026-08-03

### Fixed — the transport retraction over-reached and claimed more than the evidence supports

0.5.3 retired the loopback:7701 transport correctly, but its retraction text in `src/transport.cyr`,
`README.md` and this file stated the handshake **"could never match"** / **"was FALSE for agnos"** as
unbounded facts. Those absolutes are not supported, and `dist/mishran.cyr` ships them to consumers.

**The corrected, scoped statement:**

- **Before `net_src_for` (agnos 1.56.34)** the handshake could not close on an ordinary boot: every
  outbound segment claimed `net_ip` as its source, so a SYN to 127.0.0.1 was answered on a 4-tuple the
  client's own conn could not match. Every agnos green from that era came off the `MISHRAN_DUPLEX_SELFTEST`
  kernel hook's `net_ip = 0x7F000001` assignment and **remains retracted** — the two-proc audio results
  included.
- **`net_src_for` fixed the source-selection mechanism**, and setu's equivalent path did connect un-rigged
  afterwards (`aethersafha-clients-test.py`, `connected: 2, presented: 2`, QEMU `-smp 1`, 2026-08-02).
  **mishran's own 7701 wire was never re-tested post-fix.** So the honest statement is that it was
  **never honestly demonstrated on agnos** — not that it could not work. Neither claim should be made.
- **The transport is retired as the WRONG PRIMITIVE for local IPC, not because it was broken.**
  Replacement is the agnos socket (`naadi`), agnos `docs/development/planning/ipc.md` §9; the removal
  inventory is §10 of the same doc.

Changed: `src/transport.cyr` (file banner), `README.md`, this file, and `dist/mishran.cyr` regenerated
from source. **No code change** — comments, docs and the regenerated bundle only.

## [0.5.3] - 2026-08-02

### Changed — cyrius pin 6.4.61 -> 6.5.5

Toolchain catch-up across the whole desktop stack, cut together so the next burn runs binaries built
by ONE compiler rather than 6 different ones.

⚠ **The pin was documentation, not enforcement.** `cyrius build` compiles with the INSTALLED `cycc`,
prints a `toolchain drift` warning, and carries on — so this project was already being built by 6.5.5
before this bump. Verify provenance with `~/.cyrius/versions/<pin>/bin/cyrius` when it matters.

⭐ What the gap actually contained, for a reader deciding whether to care:
- **6.5.1** made overload-suffix arity a hard **error** where it used to warn. Latent arity
  mismatches are now build failures instead of silently-wrong code — good, and the reason this
  sweep surfaced real defects elsewhere in the stack.
- **6.4.75** fixed `fn_table` growth past 8192 silently corrupting six fn-indexed side tables.
- **6.5.0** added file-scoped `private` / per-item `public` — the first real answer to this
  ecosystem's duplicate-`fn`-silently-shadows hazard.
- **6.4.82** completed the agnos GPU syscall wrapper band to `#82`-`#95`, so `sys_gpu_shader_op`
  (#92) and `sys_gpu_modeset_op` (#93) no longer need a raw `syscall()` behind an `#ifdef`.

### Verification

Host + `--agnos` builds green; `gain_probe`, `resample_probe`, `shm_probe`, `client_leak_probe` all pass; every program in `programs/` builds. ⚠ `mishclient`/`mishduplex` build `--agnos` only, and always did — they call `sys_sched_yield`, which is agnos-only; this is unchanged by the bump (verified against the old 6.4.61 pin).

## [0.5.2] - 2026-07-12 — repin to cyrius 6.4.61 (the daemon is now leak-free end-to-end)

0.5.0/0.5.1 made mishran's own serving loop + client hot path alloc-free, but the
`client_leak_probe` still measured a non-zero `msh_server_poll` residual — the daemon's
accept loop dripping through **`net.cyr`'s `sock_accept`**, which allocated on every
would-block poll (a consumer-filed cyrius issue). **cyrius 6.4.61** fixes it
(`accept(NULL, NULL)` — the peer address was never read — plus a shared `_net_eagain()`
`Err` singleton for the would-block path, regression-gated by cyrius's
`tests/net_accept_no_leak.sh`). This patch repins mishran to receive it: the
`mishrand` mixer daemon that serves concurrent apps is now leak-free end-to-end.

### Changed

- **cyrius pin 6.4.49 → 6.4.61** — picks up the `net.cyr` `sock_accept` per-poll
  alloc-leak fix. With it, `programs/client_leak_probe.cyr` now measures
  **`msh_server_poll growth = 0 bytes`** over 20 000 blocks (was non-zero at 6.4.49) —
  the accept loop no longer drips. Probe labels + comments updated from
  "pre-existing residual" to "fixed in cyrius 6.4.61". All host probes
  (`serve_probe` / `shm_probe` / `client_leak_probe`) + the three `--agnos` daemon
  builds re-verified green.

## [0.5.1] - 2026-07-12 — the app-facing client's hot-path leak (mirror of 0.5.0)

0.5.0 closed the routing **server**'s per-poll / per-block heap leak; this patch closes
its exact **mirror on the app-facing `MshClient` API**. The client hot path allocated
from the bump allocator (which has no per-object free) on every audio block and every
ACK wait, so a long-running player — jalwa streaming a full playlist through the
two-proc mixer — slowly leaked the heap the same way the daemon did. Pure userland; the
wire and behaviour are unchanged (identical messages, identical flow) — only *where* the
message + wire buffers come from moved from a per-call `alloc` to client-owned reusable
buffers.

### Fixed

- **Unbounded client-side heap leak on the `MshClient` hot path (mirror of the 0.5.0
  serving-loop fix).** Every `msh_client_write` allocated a `MshMsg` (`msh_msg_new`,
  48 B) + a wire encode buffer (`msh_send` → `alloc(MSH_MSG_SZ)`, 48 B) per block, and
  every `msh_client_wait_ack` allocated a `MshMsg` + a wire read buffer (`msh_read_msg`,
  48 B) per ACK read — **~192 B/block** on a bump allocator that never frees (≈ 2 MB per
  4-min song; a full playlist accumulates on the agnos arena and eventually OOMs). Fixed
  with **client-owned reusable buffers** on `MshClient` (`MSH_CL_SZ` 48 → 64 B): one
  reused `MshMsg` (`MSH_CL_MSG`) + one reused wire scratch (`MSH_CL_WBUF`, `MSH_MSG_SZ`),
  both set once in `msh_client_connect` (the WELCOME message is repurposed as the reused
  `MshMsg`). New **alloc-free transport variants** `msh_send_buf` / `msh_read_msg_buf`
  take a caller-supplied wire buffer; `msh_send` / `msh_read_msg` stay as allocating
  wrappers for the one-shot connect handshake. The whole hot client path
  (`msh_client_write`, `msh_client_wait_ack`, and `msh_client_gain` / `_close`) now
  allocates **nothing** per block. A host leak probe (`programs/client_leak_probe.cyr`)
  measuring `alloc_used()` across **20 000 blocks**: `msh_client_write` growth **0 bytes**
  in steady state, versus ~192 B/block before. ~~Re-validated end-to-end on agnos
  (`mishran-duplex-audio-smoke.sh`: two concurrent procs, sustained tone to the DAC,
  RMS 2229 / PEAK 4448, no deadlock/regression).~~
  ⛔ **RETRACTED 2026-08-03 — this "re-validated on agnos" was a FALSE GREEN** produced by the
  `MISHRAN_DUPLEX_SELFTEST` kernel hook's `net_ip = 0x7F000001` assignment (see the notice at the
  top of this file). The **host** `client_leak_probe` result above stands; only the agnos leg is
  void. TCP-on-loopback is retired as a local transport — agnos `planning/ipc.md` §10.

### Added

- **`programs/client_leak_probe.cyr`** — host proof that the `MshClient` hot path is
  alloc-free. Single-process (no fork), it drives the real `msh_client_write` /
  `msh_client_wait_ack` against an in-process server, splitting the per-block
  `alloc_used()` delta into the client write (**must be 0**) vs the `msh_server_poll`
  residual. The poll residual is non-zero and **out of scope**: net.cyr's `sock_accept`
  allocs ~40 B on every EAGAIN poll (a `client_addr` + `addrlen` + `Err` Result) — a
  pre-existing per-poll leak in the materialized stdlib that also affects the `mishrand`
  daemon's accept loop, to be addressed in cyrius, not here.

## [0.5.0] - 2026-07-11 — PCM over shared memory + the serving-loop leak fix

The two-proc audio path now moves the PCM payload **off the TCP wire and onto a
shared-memory buffer** — the audio counterpart of setu's framebuffer present. Only a
tiny control message rides TCP; the samples travel through `sys_shm`. This lifts the
old inline-`MSH_WRITE` block ceiling (256 frames / 1024 B, forced under the ~2 KB agnos
loopback recv window): a client can now hand off blocks bounded by the shm buffer, not
the window. ~~Validated on agnos with 1024-frame (4096 B) blocks — larger than the whole
TCP window — reaching the DAC (`mishran-duplex-audio-smoke.sh`, RMS 2269 / PEAK 4448, no
deadlock).~~ Pure userland; no kernel change (the `sys_shm_*` syscalls #71-74 already ship,
the same ones setu's present uses).

> ⛔ **RETRACTED 2026-08-03 — the agnos validation above was a FALSE GREEN.**
> `mishran-duplex-audio-smoke.sh` passed only because the `MISHRAN_DUPLEX_SELFTEST` kernel hook
> assigned `net_ip = 0x7F000001`; without it the two procs could not complete a loopback
> handshake at all *in that era* (before `net_src_for`, agnos 1.56.34), so nothing was proven about block ceilings on agnos. Note also that "only a
> tiny control message rides TCP" is **not** the design going forward: local control messages
> move to the agnos socket (`naadi`). The shm PCM payload path is unaffected by this retraction.
> See agnos `docs/development/planning/ipc.md` §9-§10.

This cut also **closes the serving-loop heap leak** the shm bite's adversarial-verify
pass surfaced (the `### Known` item that was pending): the routing server no longer
allocates in its per-poll / per-block hot path, so a genuinely long-running mixer daemon
(`mishrand` — e.g. jalwa streaming a full song) no longer OOMs mid-stream. See **Fixed**.

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

### Fixed

- **Unbounded serving-loop heap leak (the pending `### Known` item, now closed).** The
  bump allocator has no per-object free, and nothing in the serving path called
  `alloc_reset`, so **every poll of an active slot leaked** a message struct
  (`msh_server_service`) + a 48 B wire read buffer (`msh_try_read_msg`, allocated even
  when nothing was pending) and **every consumed block leaked** a PCM copy-out (~4 KB /
  1024-frame stereo block) + the ACK's two structs (`msh_srv_ack` → `msh_msg_new` +
  `msh_send`). A forever-running `mishrand` daemon would OOM mid-stream; the bounded
  smokes (which exit in ~1.5 s) masked it. Fixed with **server-owned reusable buffers** on
  `MshServer` (`alloc(24) → alloc(56)`): one reused `MshMsg` for the poll/decode/reply
  path, one reused wire read buffer (`msh_try_read_msg` now takes it as a parameter), and
  a **grow-on-demand PCM copy-out scratch** (`msh_server_pcm` — alloc'd once, reused across
  blocks). `msh_srv_ack` is now **alloc-free**, encoding the fixed 3-word ACK straight into
  a caller-supplied buffer (the server's read buffer, free by the time an ACK is sent). A
  host leak probe measuring `alloc_used()` across **20 000 blocks**: server heap growth
  **0 bytes in steady state** (a one-time 1 KB scratch warmup), versus ~1.2 KB/block
  (≈ 24 MB) before. Deliberately **no per-tick `alloc_reset`**: `HELLO` allocates persistent
  stream/ring state (`msh_stream_new`) inside the loop, which a blanket reset would
  reclaim out from under live streams. ~~Re-validated end-to-end on agnos
  (`mishran-duplex-audio-smoke.sh`, incl. a ~10 s long-run variant, sustained tone to the
  DAC, no deadlock/OOM).~~
  ⛔ **RETRACTED 2026-08-03 — FALSE GREEN** (`MISHRAN_DUPLEX_SELFTEST` `net_ip = 0x7F000001`
  rigging; see the notice at the top of this file). The 20 000-block **host** leak measurement
  above stands — only the agnos long-run leg is void.
- **Orphaned client shm buffer on abnormal client drop.** A client that vanished without a
  `BYE` (crash) left its one out-of-band shm buffer unreclaimed for the daemon's lifetime.
  Low-severity on Linux (a stray tmpfs file), but a real hazard on agnos, whose kernel has
  only `SHM_MAX = 16` shm slots — 16 crashed clients would exhaust them and refuse all new
  connections. The server now tracks each client's last-announced buffer (`last_buf`; the
  per-client slot grew 32 → 40 B) and frees it on an **abnormal** drop; a clean `BYE` zeros
  it first so the client's own `msh_client_close` free stays authoritative (avoids a
  double-free that could hit a since-reused slot id).

## [0.4.1] - 2026-07-10 — two-proc audio on agnos (cooperative yield) ⛔ agnos claim RETRACTED

> ⛔ **RETRACTED 2026-08-03 — the headline claim of this version is a FALSE GREEN.** "Two
> concurrent processes share the hardware writer **on agnos**" was demonstrated exclusively by
> `mishran-duplex-audio-smoke.sh` under the `MISHRAN_DUPLEX_SELFTEST` kernel hook, which assigned
> `net_ip = 0x7F000001` so the client's loopback connect could match a 4-tuple it otherwise never
> could. Hook, define and smoke were deleted 2026-08-03. What **survives**: the cooperative-yield
> mechanism itself (`msh_router_pump_nb`, the `sched_yield` backoffs) is sound engineering and is
> unchanged — it simply has no valid agnos demonstration and must be re-proven over the agnos
> socket (`naadi`). What **dies**: the TCP-loopback wire as mishran's local transport, and the
> sub-window chunking workaround built to accommodate it. See agnos
> `docs/development/planning/ipc.md` §9-§10 and `planning/blocking-syscall-concurrency.md`.

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
  mixes it to vani. ~~Driven on agnos by a `MISHRAN_DUPLEX_SELFTEST` kernel hook (agnos
  `kernel/core/main.cyr`, **post-`sched_active`** so two procs actually run concurrently)
  and captured by `agnos scripts/mishran-duplex-audio-smoke.sh`: **RMS 2116, PEAK 4448**
  (thresholds 800/3000), `hda: stream running`. PCM is chunked below the 2 KB agnos TCP
  loopback window (256 frames/write) + `sched_yield`-paced so `sock_send` #48 fits the
  recv buffer instead of blocking preempt-held — a wire constraint, not a mixer one (see
  Notes). Proves: two concurrent ring-3 procs, real loopback handshake, cooperative
  mix-to-DAC, no deadlock, no starvation.~~

  ⛔ **RETRACTED 2026-08-03 — THIS PROOF PROVED THE HOOK, NOT THE PATH.** The
  `MISHRAN_DUPLEX_SELFTEST` hook assigned `net_ip = 0x7F000001` in the kernel, which is the only
  reason "real loopback handshake" ever closed; on an ordinary boot it could not. The hook, its
  `build.sh` define and `mishran-duplex-audio-smoke.sh` are **deleted** — do not look for the
  script and do not re-add the define under any name. The RMS/PEAK numbers are not evidence of a
  working two-proc path on agnos. `mishduplex.cyr` / `mishclient.cyr` remain in `programs/` as
  the shape to re-target onto the agnos socket (`naadi`); the sub-window chunking note above is
  a retired accommodation to a transport that is itself retired. See agnos
  `docs/development/planning/ipc.md` §9-§10.
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
