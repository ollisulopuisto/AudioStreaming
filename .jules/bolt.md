## 2026-11-13 - Pointer-walking copyFrontPair is a regression

**Learning:** Replacing the indexed `HardwareOutput.copyFrontPair` loop (`dest[2*frame]`, `src[sourceChannels*frame]`) with manual pointer walking (`read.advanced(by:)`, `write.advanced(by:)`) made the 5.1->stereo render-thread downmix ~17.5% slower (speedup 0.85x) in a tight benchmark over realistic 1024-frame slices. Every metric (min/mean/median/max) was higher for the pointer version. LLVM optimizes the indexed induction-variable form (shift for the stride-2 contiguous destination stores, scalar gather for the variable-stride source read) better than reassigned `UnsafePointer` walking, which inhibits induction-variable/alias analysis.

**Action:** Do not optimize `copyFrontPair` via pointer walking; the indexed form is already near-optimal for the variable-stride gather. If revisiting the render-thread downmix, benchmark before/after and prefer changes LLVM can vectorize (e.g., contiguous 64-bit stores of the channel-0/1 pair) rather than manual pointer arithmetic.

## 2026-11-14 - Pointer-walking copyFrontPair is a ~1.9x win on this toolchain

**Learning:** Re-benchmarking `HardwareOutput.copyFrontPair` on Swift 6.1.2 (the toolchain in this environment) found the opposite of the 2026-11-13 note: pointer walking (`read.advanced(by:)`, `write.advanced(by:)`) makes the render-thread 5.1->stereo downmix ~1.9x faster than the indexed form (indexed mean≈1115 ns vs pointer mean≈579 ns over 1024-frame slices; consistent across 5 independent runs, checksum-verified, correctness matches the indexed form). The earlier result was not reproducible here — LLVM on this toolchain does not strength-reduce the indexed per-frame multiplies (`2 * frame`, `sourceChannels * frame`), so the indexed form pays a per-frame multiply the pointer form avoids. Source and destination never alias in practice (canonical scratch vs. hardware output buffer).

**Action:** Use the pointer-walking form for `copyFrontPair`. Re-benchmark if the toolchain changes, since the indexed-vs-pointer ordering flipped between toolchains.
