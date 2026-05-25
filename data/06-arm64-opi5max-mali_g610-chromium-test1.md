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

\| 1_quads \| 148 \| 0.851 \| 0.800 \| 1.000 \| 1.600 \| 0.239 \|

\| 10_quads \| 148 \| 0.945 \| 0.900 \| 1.200 \| 1.300 \| 0.147 \|

\| 100_quads \| 148 \| 1.866 \| 1.900 \| 2.100 \| 2.700 \| 0.260 \|

\| 1000_quads \| 148 \| 9.184 \| 9.100 \| 9.800 \| 10.800 \| 0.368 \|

\*\*Finding:\*\* 1 quad = 0.85ms, 1000 quads = 9.18ms (11× scaling).

Per-quad incremental cost: 8.34µs.

For a real graph with 1000 items, the picking pass alone costs ~9.2ms
per refresh —

within frame budget.

This is the bottleneck refresh discipline (Layer 1) addresses by
skipping the picking pass when nothing changed.

\## Test K: Framebuffer size impact

\| Condition \| N \| Mean (ms) \| Median \| p95 \| p99 \| StdDev \|

\|---\|---\|---\|---\|---\|---\|---\|

\| fb_256 \| 148 \| 1.838 \| 1.700 \| 2.000 \| 6.900 \| 0.835 \|

\| fb_512 \| 148 \| 1.874 \| 1.900 \| 2.100 \| 2.200 \| 0.171 \|

\| fb_1024 \| 148 \| 2.353 \| 2.400 \| 2.600 \| 2.800 \| 0.247 \|

\| fb_2048 \| 148 \| 4.586 \| 4.500 \| 5.300 \| 5.600 \| 0.330 \|

\*\*Finding:\*\* 256×256 = 1.84ms, 2048×2048 = 4.59ms

(2.5× cost for 64× pixels).

Cost scales sub-linearly with framebuffer size — GPU is efficient at
large rasterization.

For Sigma users running on 4K displays, the picking pass cost may be
substantially higher than benchmarks suggest.

\## Test L: PICKING_MODE bailout at scale

\| Condition \| N \| Mean (ms) \| Median \| p95 \| p99 \| StdDev \|

\|---\|---\|---\|---\|---\|---\|---\|

\| 1_quads_with_bailout \| 148 \| 1.019 \| 1.000 \| 1.200 \| 1.800 \|
0.208 \|

\| 1_quads_no_bailout \| 148 \| 1.002 \| 1.000 \| 1.200 \| 2.700 \|
0.350 \|

\| 10_quads_with_bailout \| 148 \| 1.087 \| 1.100 \| 1.300 \| 2.100 \|
0.228 \|

\| 10_quads_no_bailout \| 148 \| 1.072 \| 1.100 \| 1.400 \| 2.400 \|
0.256 \|

\| 100_quads_with_bailout \| 148 \| 1.584 \| 1.600 \| 1.800 \| 2.500 \|
0.240 \|

\| 100_quads_no_bailout \| 148 \| 1.630 \| 1.600 \| 2.100 \| 3.200 \|
0.353 \|

\| 1000_quads_with_bailout \| 148 \| 4.048 \| 4.000 \| 4.500 \| 4.800 \|
0.223 \|

\| 1000_quads_no_bailout \| 148 \| 6.512 \| 6.500 \| 7.000 \| 7.400 \|
0.279 \|

\*\*Finding:\*\* bailout speedup at 1 quad: 0.98×.

At 1000 quads: 1.61×

(6.5ms → 4.0ms, 2.5ms saved per frame).

The bailout's per-quad savings compound linearly with edge count.

For Sigma users with custom shaders, this single fragment shader line is
potentially the largest single optimization available

without changing Sigma's architecture.

\## Test M: Concurrent JS work during async readback

\| Condition \| N \| Mean (ms) \| Median \| p95 \| p99 \| StdDev \|

\|---\|---\|---\|---\|---\|---\|---\|

\| sync_blocked_time \| 148 \| 2.178 \| 2.200 \| 2.700 \| 3.200 \| 0.423
\|

\| async_elapsed \| 80 \| 5.209 \| 5.000 \| 6.100 \| 6.600 \| 0.476 \|

\| async_concurrent_work_iters \| 80 \| 2.275 \| 3.000 \| 3.000 \| 3.000
\| 0.866 \|

\*\*Finding:\*\* sync readPixels blocks the main thread for 2.18ms (no
JS can run).

Async readPixels takes 5.21ms wall time, but during that time the main
thread completed

~2 chunks of JS work (≈22750 math operations).

This is async readback's real value: it doesn't reduce total time, but
enables productive concurrent work.

Critical for keeping React updates, layout calculations, and other UI
work responsive during hover.

\## Test N: Realistic graph simulation

\| Condition \| N \| Mean (ms) \| Median \| p95 \| p99 \| StdDev \|

\|---\|---\|---\|---\|---\|---\|---\|

\| 1000 quads, normal render (no picking) \| 148 \| 6.315 \| 6.300 \|
7.000 \| 7.400 \| 0.321 \|

\| 1000 quads, picking pass WITH bailout \| 148 \| 3.964 \| 3.900 \|
4.500 \| 5.200 \| 0.308 \|

\| 1000 quads, picking pass NO bailout \| 148 \| 6.303 \| 6.200 \| 6.800
\| 7.000 \| 0.234 \|

\*\*Finding:\*\* normal render of 1000 heavy quads: 6.31ms.

Adding a picking pass with NO bailout: 6.30ms (+-0.0ms picking cost).

Adding picking pass WITH bailout: 3.96ms (+-2.4ms picking cost).

Bailout reduces picking pass overhead by -20365% (1.59× faster).

For a real graph at scale, this is the strongest no-code-change
recommendation Sigma can offer custom-shader authors.
