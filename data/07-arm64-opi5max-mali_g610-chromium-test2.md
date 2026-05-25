\# WebGL Picking Pipeline Benchmark v3 — scale & concurrency

\## Environment

\- \*\*User agent:\*\* \`Mozilla/5.0 (X11; Linux x86_64)
AppleWebKit/537.36 (KHTML, like Gecko) Chrome/114.0.0.0 Safari/537.36\`

\- \*\*WebGL version:\*\* WebGL2

\- \*\*GPU vendor:\*\* \`Panfrost\`

\- \*\*GPU renderer:\*\* \`Mali-G610 (Panfrost)\`

\- \*\*Timestamp:\*\* 2026-05-24T21:52:13.147Z

\## Test J: Draw call scaling

\| Condition \| N \| Mean (ms) \| Median \| p95 \| p99 \| StdDev \|

\|---\|---\|---\|---\|---\|---\|---\|

\| 1_quads \| 148 \| 0.816 \| 0.700 \| 1.400 \| 2.900 \| 0.490 \|

\| 10_quads \| 148 \| 0.758 \| 0.800 \| 1.100 \| 1.300 \| 0.188 \|

\| 100_quads \| 148 \| 1.509 \| 1.500 \| 2.100 \| 2.800 \| 0.366 \|

\| 1000_quads \| 148 \| 6.421 \| 6.400 \| 7.200 \| 7.900 \| 0.510 \|

\*\*Finding:\*\* 1 quad = 0.82ms, 1000 quads = 6.42ms (8× scaling).

Per-quad incremental cost: 5.61µs.

For a real graph with 1000 items, the picking pass alone costs ~6.4ms
per refresh —

within frame budget.

This is the bottleneck refresh discipline (Layer 1) addresses by
skipping the picking pass when nothing changed.

\## Test K: Framebuffer size impact

\| Condition \| N \| Mean (ms) \| Median \| p95 \| p99 \| StdDev \|

\|---\|---\|---\|---\|---\|---\|---\|

\| fb_256 \| 148 \| 1.193 \| 1.200 \| 1.400 \| 2.500 \| 0.202 \|

\| fb_512 \| 148 \| 1.323 \| 1.300 \| 1.600 \| 1.800 \| 0.189 \|

\| fb_1024 \| 148 \| 2.177 \| 2.200 \| 2.600 \| 2.800 \| 0.304 \|

\| fb_2048 \| 148 \| 4.518 \| 4.400 \| 5.400 \| 5.700 \| 0.346 \|

\*\*Finding:\*\* 256×256 = 1.19ms, 2048×2048 = 4.52ms

(3.8× cost for 64× pixels).

Cost scales sub-linearly with framebuffer size — GPU is efficient at
large rasterization.

For Sigma users running on 4K displays, the picking pass cost may be
substantially higher than benchmarks suggest.

\## Test L: PICKING_MODE bailout at scale

\| Condition \| N \| Mean (ms) \| Median \| p95 \| p99 \| StdDev \|

\|---\|---\|---\|---\|---\|---\|---\|

\| 1_quads_with_bailout \| 148 \| 0.924 \| 0.900 \| 1.200 \| 1.700 \|
0.196 \|

\| 1_quads_no_bailout \| 148 \| 0.825 \| 0.800 \| 1.000 \| 1.400 \|
0.202 \|

\| 10_quads_with_bailout \| 148 \| 1.053 \| 1.000 \| 1.300 \| 1.500 \|
0.161 \|

\| 10_quads_no_bailout \| 148 \| 0.926 \| 0.900 \| 1.300 \| 2.100 \|
0.274 \|

\| 100_quads_with_bailout \| 148 \| 1.241 \| 1.200 \| 1.700 \| 2.600 \|
0.391 \|

\| 100_quads_no_bailout \| 148 \| 1.370 \| 1.300 \| 1.700 \| 2.000 \|
0.176 \|

\| 1000_quads_with_bailout \| 148 \| 3.778 \| 3.700 \| 4.400 \| 5.100 \|
0.304 \|

\| 1000_quads_no_bailout \| 148 \| 6.356 \| 6.300 \| 7.000 \| 7.500 \|
0.342 \|

\*\*Finding:\*\* bailout speedup at 1 quad: 0.89×.

At 1000 quads: 1.68×

(6.4ms → 3.8ms, 2.6ms saved per frame).

The bailout's per-quad savings compound linearly with edge count.

For Sigma users with custom shaders, this single fragment shader line is
potentially the largest single optimization available

without changing Sigma's architecture.

\## Test M: Concurrent JS work during async readback

\| Condition \| N \| Mean (ms) \| Median \| p95 \| p99 \| StdDev \|

\|---\|---\|---\|---\|---\|---\|---\|

\| sync_blocked_time \| 148 \| 2.434 \| 2.500 \| 2.800 \| 3.200 \| 0.332
\|

\| async_elapsed \| 80 \| 5.467 \| 4.900 \| 9.200 \| 11.900 \| 1.471 \|

\| async_concurrent_work_iters \| 80 \| 2.625 \| 3.000 \| 3.000 \| 4.000
\| 0.714 \|

\*\*Finding:\*\* sync readPixels blocks the main thread for 2.43ms (no
JS can run).

Async readPixels takes 5.47ms wall time, but during that time the main
thread completed

~3 chunks of JS work (≈26250 math operations).

This is async readback's real value: it doesn't reduce total time, but
enables productive concurrent work.

Critical for keeping React updates, layout calculations, and other UI
work responsive during hover.

\## Test N: Realistic graph simulation

\| Condition \| N \| Mean (ms) \| Median \| p95 \| p99 \| StdDev \|

\|---\|---\|---\|---\|---\|---\|---\|

\| 1000 quads, normal render (no picking) \| 148 \| 6.304 \| 6.200 \|
6.700 \| 9.400 \| 0.494 \|

\| 1000 quads, picking pass WITH bailout \| 148 \| 4.031 \| 4.000 \|
4.400 \| 5.100 \| 0.263 \|

\| 1000 quads, picking pass NO bailout \| 148 \| 6.402 \| 6.300 \| 7.000
\| 7.600 \| 0.352 \|

\*\*Finding:\*\* normal render of 1000 heavy quads: 6.30ms.

Adding a picking pass with NO bailout: 6.40ms (+0.1ms picking cost).

Adding picking pass WITH bailout: 4.03ms (+-2.3ms picking cost).

Bailout reduces picking pass overhead by 2420% (1.59× faster).

For a real graph at scale, this is the strongest no-code-change
recommendation Sigma can offer custom-shader authors.
