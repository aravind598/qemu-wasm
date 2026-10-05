# Browser GPU Doom: retirement diagnosis and rejected CLOCK trial

The tested retirement cohort was reclaimable, but finalizer-coupled admission waited for browser collection. A minimal second-chance replacement for FIFO retirement did not improve the matched three-game workload and is **not recommended for promotion**. No runtime policy is applied by this documentation PR.

## Scope and source identity

Browser QEMU, minimal Alpine, DSDA-Doom 0.29.4, Freedoom 0.13.0, VirGL/WebGL2, 640x480, 350-gametic fixed demo, no audio. DMA remapping enabled;16-bit PASID compatibility advertisement; interrupt remapping disabled. Fresh-walk parity is tested; cache/invalidation-stream parity remains unverified.

The source archive directory identifies 9ded45a67ff5daa7f30264bc8a9868b420da2a54. Its actual `tcg/wasm64.c` SHA256 is 2def5df614d53ac80479280a8df2711a609c2dc5d6955972b51a96ca0f2ed213, verified byte-identical to the local approved source. The extracted source lacks Git metadata; do not infer identity with the older public `tcg/wasm.c` from this directory name. The actual SDK is Emscripten 6.0.11. The active build uses the Wasm64 JIT with TCI cold/cap fallback; it is not an interpreter-only integration regression.

## Owning-worker diagnostic

Pre-start page-scoped CDP auto-attachment and an `init_wasm_js` call-frame handshake identified the actual vCPU worker. A diagnostic-only startup yield before its first TB established an unpaused event-loop gate; owner `HeapProfiler.collectGarbage` preflight acknowledged in 4 ms with 0instances.

At a later native `trysleep` boundary, after dynamic TB execution returned, the existing timeout callback held only its continuation resolver and scalar/WeakRef audit data, then returned to the event loop. New allocations and retirements were frozen. No exported functions were logged or kept in diagnostic arrays. Collection was requested while unpaused, not inside a debugger nested loop.

| Observation | Created | Retired | Finalized callbacks | Counted | Pending | Unconsumed callbacks |
|---|---:|---:|---:|---:|---:|---:|
| Before collection |24000|18000|12000|12000|6000|0|
| After2.008s yield-only |24000|18000|12000|12000|6000|0|
| After owner GC acknowledged 472ms |24000|18000|18000|12000|6000|6000|
| First native sample after continuation release |26134|18000|18000|8134|0|0|

All6000pending objects finalized; after execution resumed, accounting consumed their callbacks and admitted5611newTBs in the six-second follow-up while remaining under12000counted. The cap-fallback counter stayed flat in that follow-up. No exact native-byte release or FPS improvement is inferred. This demonstrates reclaimable collection delay for this cohort, not a general leak diagnosis or a production forced-GC solution.

Separate all-retirement checks validated36000 table slots and JS mirror clears with 0failures. Callback firing and native consumption matched; no sustained accounting loss was observed. The pending-GC retirement guard already existed and is not a new fix.

## One policy variable: second-chance selection

The local candidate preserves stable slot addresses referenced by TB headers. Successful JIT lookup marks one reuse bit. At the unchanged cap, retirement gives referenced slots one second chance and removes half the active entries, using at most two fixed-array sweeps. Pending callbacks still block further retirement; only actual callback consumption decrements the counted budget. No earlier-retirement threshold, probation change, cap increase, forced GC or memory-pressure trick was combined. Extra metadata is about12KB per worker.

The single selected change tests selection independently of cohort size. The suggested256-member cohort was not tested. The scan remains a large bounded batch; before any future landing, slice-budgeted maintenance and the all-hot/no-victim case still need a separate candidate. This half-cohort candidate is rejected, so no claim that this maintenance scheme is production-ready is made.

## Matched results

Both fresh browsers used the same guest, resolution, settings, path counters and CDP screencast observer. Captured canvas images were decoded and compared; rates are instrumented observed canvas changes, possibly undersampled, not physical display refresh. Guest-clock FPS is separate.

| Metric |FIFO control|Second-chance half-cohort|
|---|---:|---:|
|Guest FPS, games1/2/3|[11.2, 9.0, 7.3]|[11.0, 9.2, 7.2]|
|Third-game decoded canvas changes/s|7.64|7.78|
|Third-game decoded intervals p95/p99(ms)|192.9/331.5|203.7/480.9|
|Third-game cap-fallback block entries(%)|62.38|62.63|
|Total instances created|42000|42000|
|Sampled peak renderer private MiB|2821.1|2874.0|

Both completed three games and shut down cleanly. There was no reduction in compilation churn or cap fallback, and no observed third-game performance gain. Candidate tail intervals and sampled memory were worse. This AB pair is not statistical proof, but offers no justification to promote the change. It does not rule out smaller-cohort CLOCK or other policies.

## Correctness checks

- Candidate input: turning/firing events, focus release and hidden-tab release passed; decoded world-region image checks passed.
- Fresh candidate GPU capture:1,638/1,638 untagged IOMMU walks matched offline dmakit; missing/corrupt-table negatives passed.299 GPU commands and2308cache hits observed; cache hits were not replayed. Captures from other milestones are independent and are not summed.
- Existing graphics regressions:25/25 lifecycle checks, including 20context recreations;7/7 deleted-binding/unpack-resize checks passed.
- Diagnostic no-finalizer stress: native callback notifications deliberately withheld; counted budget stayed12000, pending6000, no further instances admitted. Three game phases and shutdown passed. At least5888withheld callbacks observed from256-step reports. This modified workload is not a performance baseline.
- Static guest self-modification test:5000 calls return7, code changed to9 and5000calls verified, then restored7 and5000calls verified. All four executions passed: beforeGPU startup and after each game. No new tool installation or host execution of this guest ELF.

The intermittent original END_TRANSFERS1282/scanout0x502 fault remains unresolved. Passing these runs does not explain or fix it. GL error checks remain enabled.

## Artifacts and reproduction limits

Control report:`qemu-gpu-1791183358070`; candidate:`qemu-gpu-1791183658993`; input/IOMMU:`qemu-gpu-1791184097698`; stress:`qemu-gpu-1791184390193`; owner GC:`gc-capture-1791182778263`.

Control Wasm SHA256:`9d5f43706b9a0c721b3052d3f3d2d7dd6129b32c62dab644e617c664f1ab4965`. Candidate:`803b22bc3933c5de97d415f7b982f53fab830bf3e4c05bdd19639d8f463c1b7a`. Diagnostic startup-yield:`6de653fa36e5bdb03d28f9833d8e0250d197a43685d4f0e2f403aaf1f4a08b00`.

These findings summarize local recorded evidence; they are not an independently reproduced CI result. The complete source scripts, patches, logs and pinned build manifests remain in the isolated local handoff. Giant guest/runtime binaries are excluded. Repository CI replica, independent review and merge gates have not been claimed complete; this is a draft documentation handoff only. Existing PR branches, main/master, upstream repositories, deployment and credentials are untouched.

Next bounded work: retain this failed candidate as evidence. Selection alone cannot admit new modules while finalizer-coupled accounting stays at cap. Before broad integration, an offline two-TB then at-most-eight-TB packing probe can use the already installed Binaryen `wasm-merge`; preserve each TB's17mutable register/resume globals, worker-local helper bindings, embedded TCI return addresses, jump targets and generation checks. Do not share one mutable state bank or recycle an instance for arbitrary new TBs. Require original compiled-byte/TB-export budgets and the counted/native ceiling before any browser acceptance experiment. No packing implementation or speedup is claimed here.

