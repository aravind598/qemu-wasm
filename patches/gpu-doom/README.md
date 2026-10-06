# QEMU patches behind the browser GPU Doom run

Base: upstream QEMU `9ded45a67ff5daa7f30264bc8a9868b420da2a54` plus `../iommu-qemu/qemu-master-wasm64-jit.patch`, in this order.
Checked here: each applies with `git apply --index` on the one before it. Built as one Wasm64 QEMU and run in the browser with Alpine, VirGL and DSDA-Doom; not run through `scripts/qemu-ci.sh` or `--wasm`, and not rebased onto the repo's own AMD patches.

1. `qemu-master-wasm64-tci-direct-calls.patch` (tcg/wasm64.c): the cold-block interpreter calls helpers whose arguments are all 32/64-bit integers or pointers (up to 6, void/i32/i64 result) straight through a typed function pointer instead of libffi's JavaScript marshalling, and raises `MAX_INSTANCES` from 12,000 to 30,000. libffi stays for every other signature. Measured: libffi calls were 14-19% of the CPU thread's wall time in game 1; the instance cap 30,000 gave about +8-10% for about +100 MiB. fragile: Wasm traps on a signature mismatch, so only helpers the workload calls are tested.
2. `qemu-master-gpu-webgl-display.patch`: the browser display (`ui/webgl.c`: WebGL scanout, keyboard events, canvas resize), virtio-gpu/virgl glue (`hw/display`), build files, `util/qemu-timer.c`.
3. `qemu-master-gpu-amd-iommu-capture.patch`: the AMD-Vi capture trace events used to replay GPU DMA page walks against dmakit, the edu DMA-mask change and `memattrs.h`. It overlaps the repo's `qemu-master-amd-iommu.patch` and is not meant to be combined with it.

The JavaScript side (`pixelStorei`, `deleteProgram` and deferred `getError` patches for the virtual WebGL layer, the sleep-yield change) is in `../../gpu-doom-sleep-yield-20261005/`.

Note for this repository: these patches are written against upstream QEMU 9ded45a (11.1.50) with the wasm64 JIT patch from `aravind598/portable-media-stack` (`docs/evidence/iommu-qemu/qemu-master-wasm64-jit.patch`), not against this repository's default branch. They are kept here as a record; full write-up, measurements and the reproduction scripts are in portable-media-stack PR 68 (`docs/gpu-doom-pixelstore-20261005.md`).
