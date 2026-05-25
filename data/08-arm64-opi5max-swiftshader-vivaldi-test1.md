\# WebGL Picking Pipeline Benchmark v3 — scale & concurrency

\## Environment

\- \*\*User agent:\*\* \`Mozilla/5.0 (X11; Linux x86_64)
AppleWebKit/537.36 (KHTML, like Gecko) Chrome/142.0.0.0 Safari/537.36\`

\- \*\*WebGL version:\*\* WebGL2

\- \*\*GPU vendor:\*\* \`Google Inc. (Google)\`

\- \*\*GPU renderer:\*\* \`ANGLE (Google, Vulkan 1.3.0 (SwiftShader
Device (LLVM 10.0.0) (0x0000C0DE)), SwiftShader driver)\`

\- \*\*Timestamp:\*\* 2026-05-24T21:31:16.802Z

\## Test J: Draw call scaling

\| Condition \| N \| Mean (ms) \| Median \| p95 \| p99 \| StdDev \|

\|---\|---\|---\|---\|---\|---\|---\|

\| 1_quads \| 148 \| 0.840 \| 0.700 \| 1.500 \| 1.800 \| 0.342 \|

\| 10_quads \| 148 \| 1.397 \| 1.200 \| 2.200 \| 3.300 \| 0.448 \|

\| 100_quads \| 148 \| 6.186 \| 6.000 \| 7.700 \| 8.700 \| 0.836 \|

\| 1000_quads \| 148 \| 48.016 \| 47.700 \| 54.200 \| 57.800 \| 3.264 \|

\*\*Finding:\*\* 1 quad = 0.84ms, 1000 quads = 48.02ms (57× scaling).

Per-quad incremental cost: 47.22µs.

For a real graph with 1000 items, the picking pass alone costs ~48.0ms
per refresh —

over the 16ms frame budget.

This is the bottleneck refresh discipline (Layer 1) addresses by
skipping the picking pass when nothing changed.

\## Test K: Framebuffer size impact

\| Condition \| N \| Mean (ms) \| Median \| p95 \| p99 \| StdDev \|

\|---\|---\|---\|---\|---\|---\|---\|

\| fb_256 \| 148 \| 3.964 \| 4.100 \| 5.600 \| 6.400 \| 1.205 \|

\| fb_512 \| 148 \| 5.941 \| 5.600 \| 8.000 \| 8.900 \| 0.930 \|

\| fb_1024 \| 148 \| 16.073 \| 15.500 \| 19.800 \| 21.900 \| 1.843 \|

\| fb_2048 \| 148 \| 55.409 \| 54.200 \| 67.800 \| 74.000 \| 5.182 \|

\*\*Finding:\*\* 256×256 = 3.96ms, 2048×2048 = 55.41ms

(14.0× cost for 64× pixels).

Cost scales sub-linearly with framebuffer size — GPU is efficient at
large rasterization.

For Sigma users running on 4K displays, the picking pass cost may be
substantially higher than benchmarks suggest.

\## Test L: PICKING_MODE bailout at scale

\| Condition \| N \| Mean (ms) \| Median \| p95 \| p99 \| StdDev \|

\|---\|---\|---\|---\|---\|---\|---\|

\| 1_quads_with_bailout \| 148 \| 1.086 \| 1.100 \| 1.700 \| 1.900 \|
0.312 \|

\| 1_quads_no_bailout \| 148 \| 1.153 \| 1.100 \| 2.100 \| 2.600 \|
0.498 \|

\| 10_quads_with_bailout \| 148 \| 1.458 \| 1.500 \| 1.900 \| 2.300 \|
0.317 \|

\| 10_quads_no_bailout \| 148 \| 1.719 \| 1.600 \| 3.000 \| 3.400 \|
0.557 \|

\| 100_quads_with_bailout \| 148 \| 6.750 \| 6.200 \| 10.700 \| 14.200
\| 2.068 \|

\| 100_quads_no_bailout \| 148 \| 8.278 \| 7.800 \| 13.200 \| 16.600 \|
2.029 \|

\| 1000_quads_with_bailout \| 148 \| 39.082 \| 40.900 \| 49.600 \|
54.600 \| 6.843 \|

\| 1000_quads_no_bailout \| 148 \| 49.522 \| 49.200 \| 56.200 \| 59.700
\| 3.396 \|

\*\*Finding:\*\* bailout speedup at 1 quad: 1.06×.

At 1000 quads: 1.27×

(49.5ms → 39.1ms, 10.4ms saved per frame).

The bailout's per-quad savings compound linearly with edge count.

For Sigma users with custom shaders, this single fragment shader line is
potentially the largest single optimization available

without changing Sigma's architecture.

\## Test M: Concurrent JS work during async readback

\| Condition \| N \| Mean (ms) \| Median \| p95 \| p99 \| StdDev \|

\|---\|---\|---\|---\|---\|---\|---\|

\| sync_blocked_time \| 148 \| 10.776 \| 10.300 \| 13.400 \| 14.500 \|
1.243 \|

\| async_elapsed \| 80 \| 14.887 \| 14.400 \| 17.000 \| 19.000 \| 1.387
\|

\| async_concurrent_work_iters \| 80 \| 2.563 \| 2.000 \| 4.000 \| 4.000
\| 0.920 \|

\*\*Finding:\*\* sync readPixels blocks the main thread for 10.78ms (no
JS can run).

Async readPixels takes 14.89ms wall time, but during that time the main
thread completed

~3 chunks of JS work (≈25625 math operations).

This is async readback's real value: it doesn't reduce total time, but
enables productive concurrent work.

Critical for keeping React updates, layout calculations, and other UI
work responsive during hover.

\## Test N: Realistic graph simulation

\| Condition \| N \| Mean (ms) \| Median \| p95 \| p99 \| StdDev \|

\|---\|---\|---\|---\|---\|---\|---\|

\| 1000 quads, normal render (no picking) \| 148 \| 49.868 \| 49.400 \|
56.200 \| 60.700 \| 3.604 \|

\| 1000 quads, picking pass WITH bailout \| 148 \| 37.794 \| 39.800 \|
45.300 \| 60.600 \| 6.950 \|

\| 1000 quads, picking pass NO bailout \| 148 \| 49.105 \| 48.700 \|
54.400 \| 60.100 \| 3.281 \|

\*\*Finding:\*\* normal render of 1000 heavy quads: 49.87ms.

Adding a picking pass with NO bailout: 49.11ms (+-0.8ms picking cost).

Adding picking pass WITH bailout: 37.79ms (+-12.1ms picking cost).

Bailout reduces picking pass overhead by -1483% (1.30× faster).

For a real graph at scale, this is the strongest no-code-change
recommendation Sigma can offer custom-shader authors.
