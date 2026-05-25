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

\| 1_quads \| 148 \| 0.752 \| 0.700 \| 1.200 \| 1.400 \| 0.205 \|

\| 10_quads \| 148 \| 1.336 \| 1.200 \| 2.000 \| 2.100 \| 0.336 \|

\| 100_quads \| 148 \| 6.751 \| 6.600 \| 9.400 \| 12.500 \| 1.436 \|

\| 1000_quads \| 148 \| 47.896 \| 47.600 \| 55.500 \| 61.100 \| 3.753 \|

\*\*Finding:\*\* 1 quad = 0.75ms, 1000 quads = 47.90ms (64× scaling).

Per-quad incremental cost: 47.19µs.

For a real graph with 1000 items, the picking pass alone costs ~47.9ms
per refresh —

over the 16ms frame budget.

This is the bottleneck refresh discipline (Layer 1) addresses by
skipping the picking pass when nothing changed.

\## Test K: Framebuffer size impact

\| Condition \| N \| Mean (ms) \| Median \| p95 \| p99 \| StdDev \|

\|---\|---\|---\|---\|---\|---\|---\|

\| fb_256 \| 148 \| 3.437 \| 3.000 \| 5.200 \| 5.800 \| 1.042 \|

\| fb_512 \| 148 \| 5.978 \| 5.700 \| 7.600 \| 11.500 \| 1.097 \|

\| fb_1024 \| 148 \| 15.903 \| 15.500 \| 18.700 \| 21.600 \| 1.553 \|

\| fb_2048 \| 148 \| 53.674 \| 53.600 \| 59.300 \| 61.800 \| 2.867 \|

\*\*Finding:\*\* 256×256 = 3.44ms, 2048×2048 = 53.67ms

(15.6× cost for 64× pixels).

Cost scales sub-linearly with framebuffer size — GPU is efficient at
large rasterization.

For Sigma users running on 4K displays, the picking pass cost may be
substantially higher than benchmarks suggest.

\## Test L: PICKING_MODE bailout at scale

\| Condition \| N \| Mean (ms) \| Median \| p95 \| p99 \| StdDev \|

\|---\|---\|---\|---\|---\|---\|---\|

\| 1_quads_with_bailout \| 148 \| 1.051 \| 1.000 \| 1.700 \| 2.100 \|
0.364 \|

\| 1_quads_no_bailout \| 148 \| 1.232 \| 1.200 \| 2.700 \| 3.800 \|
0.625 \|

\| 10_quads_with_bailout \| 148 \| 1.339 \| 1.500 \| 1.700 \| 1.900 \|
0.339 \|

\| 10_quads_no_bailout \| 148 \| 1.590 \| 1.400 \| 2.600 \| 3.900 \|
0.603 \|

\| 100_quads_with_bailout \| 148 \| 4.670 \| 4.900 \| 6.500 \| 7.600 \|
1.142 \|

\| 100_quads_no_bailout \| 148 \| 6.280 \| 6.000 \| 8.400 \| 9.200 \|
0.994 \|

\| 1000_quads_with_bailout \| 148 \| 37.134 \| 40.600 \| 44.600 \|
46.700 \| 7.067 \|

\| 1000_quads_no_bailout \| 148 \| 49.784 \| 49.500 \| 57.600 \| 63.300
\| 3.861 \|

\*\*Finding:\*\* bailout speedup at 1 quad: 1.17×.

At 1000 quads: 1.34×

(49.8ms → 37.1ms, 12.7ms saved per frame).

The bailout's per-quad savings compound linearly with edge count.

For Sigma users with custom shaders, this single fragment shader line is
potentially the largest single optimization available

without changing Sigma's architecture.

\## Test M: Concurrent JS work during async readback

\| Condition \| N \| Mean (ms) \| Median \| p95 \| p99 \| StdDev \|

\|---\|---\|---\|---\|---\|---\|---\|

\| sync_blocked_time \| 148 \| 10.956 \| 10.500 \| 13.200 \| 14.200 \|
1.181 \|

\| async_elapsed \| 80 \| 15.284 \| 14.400 \| 21.800 \| 23.700 \| 2.168
\|

\| async_concurrent_work_iters \| 80 \| 2.725 \| 2.000 \| 4.000 \| 5.000
\| 0.987 \|

\*\*Finding:\*\* sync readPixels blocks the main thread for 10.96ms (no
JS can run).

Async readPixels takes 15.28ms wall time, but during that time the main
thread completed

~3 chunks of JS work (≈27250 math operations).

This is async readback's real value: it doesn't reduce total time, but
enables productive concurrent work.

Critical for keeping React updates, layout calculations, and other UI
work responsive during hover.

\## Test N: Realistic graph simulation

\| Condition \| N \| Mean (ms) \| Median \| p95 \| p99 \| StdDev \|

\|---\|---\|---\|---\|---\|---\|---\|

\| 1000 quads, normal render (no picking) \| 148 \| 49.640 \| 49.700 \|
56.000 \| 58.700 \| 3.601 \|

\| 1000 quads, picking pass WITH bailout \| 148 \| 36.330 \| 38.000 \|
44.400 \| 45.100 \| 6.781 \|

\| 1000 quads, picking pass NO bailout \| 148 \| 50.726 \| 49.600 \|
62.300 \| 74.000 \| 5.708 \|

\*\*Finding:\*\* normal render of 1000 heavy quads: 49.64ms.

Adding a picking pass with NO bailout: 50.73ms (+1.1ms picking cost).

Adding picking pass WITH bailout: 36.33ms (+-13.3ms picking cost).

Bailout reduces picking pass overhead by 1325% (1.40× faster).

For a real graph at scale, this is the strongest no-code-change
recommendation Sigma can offer custom-shader authors.
