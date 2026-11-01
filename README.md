# Ascend AICPU BLAKE3 — C / 6 logical workers (NEON-4)

This is a six-worker **AICPU benchmark**, derived from `ascend_blake3_aicpu-main`.
Each worker owns an independent Ascend stream and queues `queue_depth` BLAKE3 kernels.
Each kernel uses ARM NEON four-way SIMD to compute independent one-block BLAKE3
hashes. The host dispatches to **all six workers before waiting** and reports
measured wall-clock Mhash/s. This reduces the original one-wave-per-stream
synchronization overhead, but there is still a window-level barrier.

**Important:** Six streams / logical workers do **not** prove six distinct physical
AICPU cores. CANN scheduling decides mapping; AICPU `<<<6>>>` does not provide
AI-Core-style six-block partitioning. Verify actual parallelism by comparing
1, 2, 4, 6, 8 workers and AI Cpu(%) while holding groups/depth fixed.

## Usage

```bash
source /usr/local/Ascend/ascend-toolkit/set_env.sh
bash build.sh 6 1024 0 4 32
# thereafter:
./blake3_bench 6 1024 0 4 32
```

Arguments: `workers` (default 6), `groups` per kernel (default 1024),
`device` (default 0), `queue_depth` kernels queued per worker per window
(default 4), `windows` count (default 0 = until Ctrl+C).
For a comparable controlled benchmark, set `windows=32` and test
`./blake3_bench 1 1024 0 4 32`, then `2`, `4`, `6`, `8`.

`workers * groups * 4 * queue_depth` is the number of hashes per full window.
The reported `window_hash/s` and `overall_hash/s` are based on host steady-clock
elapsed time, including launch and synchronization overhead. `Ctrl+C` drains
submitted tasks and intentionally omits a partially submitted window from the
accounted total.

## Correctness / scope

- Uses real BLAKE3 compression and NEON `uint32x4_t` on four independent
  deterministic **64-byte** messages per group.
- On-device `BLAKE3("abc")` self-test prints **PASS** when successful.
  Expected digest: `6437b3ac38465133ffb63b75273a8db548c558465d79db03fd359c6cd5bd9d85`.
  Inspect the PASS text; synchronized launch alone is **not** proof of validity.
- This is unkeyed, single-block BLAKE3, **not PRL keyed BLAKE3**. No miner or
  PRL proof integration is claimed. It measures synthetic hash throughput.
- The generated messages use previous hash feedback, so the effective workload
  is SIMD-dependent across groups. Kernel output is not copied back to host.
- This environment did not have a CANN BiSheng compiler or Ascend hardware;
  build/runtime and six-core occupancy have not been verified here.

## Build prerequisites

A CANN environment with `bisheng`, `aicpu_api.h`, compatible AArch64 toolchain
and `dav-2201`. `build.sh` compiles `.aicpu` + `.asc` and runs the benchmark.

## How to interpret performance

Higher AICPU utilization alone is not the goal. Multiple worker streams may
serialize or contend, and added kernels can make the end-to-end PRL iter/s worse.
Use this stand-alone test to establish a baseline for how many AICPU tasks can
actually execute concurrently before attempting integration with Cube/Vector.
