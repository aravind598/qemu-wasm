# Browser GPU Doom findings — 2026-10-05

Doom runs through browser QEMU, a minimal Alpine guest and VirGL/WebGL2.
It is not yet reliably playable at 60 FPS. Repeated games still slow down, and the
original intermittent graphics fault is not conclusively explained.

## Work queue, highest priority first

1. Stabilize graphics context lifecycle and scanout. Verify repeated games,
   guest shutdown, turning/firing/focus release, and fresh GPU IOMMU walks.
2. Diagnose repeated-run slowdown and renderer private-memory growth.
   Separate JIT instance retirement/GC, interpreter fallback, and graphics lifetime.
3. Reduce measured Wasm/JavaScript overhead. Repeat the checked integer conversion
   comparison in reverse order before promoting it.
4. Reduce redundant graphics calls and synchronization while retaining correctness
   checks. Profile actual calls before changing the transport.
5. Measure distinct host frame content, frame-time distributions and guest ticks
   separately. Quantify tracing/capture overhead.
6. Tune fixed resolution and GPU selection after the execution path is stable.

## Verified execution path

The accepted runtime is JIT-enabled: HOST_WASM64=1 and CONFIG_TCG=1, without
CONFIG_TCG_INTERPRETER. Cold blocks intentionally use TCI; hot blocks compile to
Wasm after the threshold (1500 in this build). No interpreter-only integration
regression was found. The accepted Wasm SHA256 is
`3a8ec141d0459f09e6522609865a772e39d776d1a37a3a4d996cd5e50510e5a1`.

The JIT caps live instances at 12000, retires the oldest half and waits for
FinalizationRegistry callbacks to release the live-instance count. Retirement
can therefore prevent new compilation while existing compiled blocks still run.
This is a policy in the source, not a verified universal browser limit.

A separately rebuilt, hash-matched diagnostic confirmed substantial interpreter
and BigInt FFI residence in the first game, and compiled JIT execution in later
games. Graphics-worker getError samples also contain waits. Concurrent worker
profiles must not be added together as elapsed CPU time.

## Stability changes

The virtual WebGL layer now clears cached references when deleting textures,
renderbuffers, samplers and vertex arrays. Deterministic deleted-binding cases
failed before this change and passed afterward.

Drawingbuffer resize now temporarily unbinds PIXEL_UNPACK_BUFFER for private
texImage2D allocation and restores it. A bound-buffer resize reproduced error
1282 before the fix; afterward binding and expected output pixels were preserved.

Combined checks: 7/7 deletion/resize cases and 25/25 lifecycle checks, including
20 context recreations with flat VAO counts. The combined patch applies cleanly
and reproduces the tested source SHA256
`6efb393e2935a25d97c906c350c45034c35e9b421691fc3b33e9f93f17db8060`.

Two earlier controls failed at third-game startup with END_TRANSFERS 1282,
Illegal command buffer and scanout 0x502, including one with deletion fixes.
A native GL diagnostic completed three games without reproducing that failure.
The deterministic resize bug is fixed; it is not proven to explain every earlier
scanout failure. GL error checks remain enabled.

VirGL 1.3.0 source audit confirms vrend_decode.c:2099 checks GL errors after
command dispatch and propagates command failure; vrend_renderer.c:7552 implements
that check. Simply deleting it would change correctness behavior. Investigate
call/proxy overhead and redundant state changes instead.

## Current measured performance

Identical fixed 640x480, 350-tic demos, sound disabled, in one browser lifetime:

| Local report | Runtime | Guest FPS, games 1 / 2 / 3 | Completion |
| --- | --- | --- | --- |
| qemu-gpu-1791169451257 | Combined graphics fixes, original Wasm | 8.9 / 8.8 / 6.1 | All game exits zero; shutdown not observed |
| qemu-gpu-1791169786459 | Same build plus checked int53 fast path | 9.2 / 9.4 / 7.9 | All games and Alpine powerdown verified |
| qemu-gpu-1791170492802 | Unchanged control after candidate | 8.5 / 8.4 / 6.4 | All games and Alpine powerdown verified |
| qemu-gpu-1791170835734 | Repeated int53 candidate | 9.0 / 8.7 / 7.3 | All games and Alpine powerdown verified |

The control harness stopped at Doom exit before guest shutdown. It has been
corrected to wait for powerdown; the candidate passed that stricter check.
This does not invalidate completed gameplay timings, but the first report is
not a shutdown pass.

The conversion candidate passed 328701 differential checks, retains exact
inclusive int53 boundary checks, and changes one emitted JavaScript helper.
Wasm and guest are unchanged. The control/candidate/control/candidate sequence shows improved third-game guest
FPS in both candidate runs (7.3-7.9 versus 6.1-6.4), while earlier-game results
overlap. This is limited evidence, not a statistically established general gain.
Frame-time spikes remain: repeated candidate game-three canvas-copy p99 was
409ms versus 365ms in the adjacent control. These are compositor-copy intervals,
not unique displayed-frame timings. The candidate remains an experiment.

Earlier retirement at 9000 instances produced 6.7 / 7.6 / 8.6 guest FPS and
lower final renderer private memory (~2695 MiB versus prior ~2840 MiB), but
slowed the first game. It remains an experiment. Lowering compilation threshold
to 100 regressed and was rejected.

Guest FPS and canvas-copy rates do not establish unique host-visible FPS.
One earlier instrumented CDP capture observed about 9.52 changing decoded canvas
images per second; it may undersample and is not physical display refresh.
No 60 FPS result is claimed.

## IOMMU acceptance and limits

DMA remapping is enabled. Compatibility configuration advertises 16-bit PASID
and disables interrupt remapping. Cache/invalidation parity remains unverified.

The last completed original-runtime acceptance replay matched 1193/1193 fresh
GPU IOMMU walks, with missing/corrupt-memory negative checks passing. Earlier
captures matched 2749/2749, 1437/1437 and 1245/1245 independently; do not sum them.
Combined-fix acceptance qemu-gpu-1791170152678 passed turning, firing and focus
release. World-region image checks passed. Offline replay matched 930/930 fresh
GPU IOMMU walks across 196 GPU commands, with missing/corrupt-memory negatives
passing. The observed 1435 cache hits are not replayed; invalidation parity is
still unverified.

## Evidence and scope

All source experiments, scripts and reports are preserved locally in
gpu-doom-continuation. Original preview and old milestones remain unchanged.
This findings branch is documentation only: it does not change runtime code,
ship guest binaries, claim full CI passes, merge, or deploy. Existing PR 60 and
other existing PR branches are untouched.

The browser graphics API boundary remains even when computation is in Wasm.
Reduce measured crossings and redundant calls rather than assume a rewrite
removes the boundary:
https://emscripten.org/docs/optimizing/Optimizing-WebGL.html
