# WebGL Picking Pipeline Benchmark v3 — scale & concurrency

## Environment

- **User agent:** `Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:147.0) Gecko/20100101 Firefox/147.0`
- **WebGL version:** WebGL2
- **GPU vendor:** `Intel`
- **GPU renderer:** `Intel(R) HD Graphics, or similar`
- **Timestamp:** 2026-05-24T18:46:15.877Z

## Test J: Draw call scaling

| Condition | N | Mean (ms) | Median | p95 | p99 | StdDev |
|---|---|---|---|---|---|---|
| 1_quads | 148 | 1.527 | 1.000 | 7.000 | 8.000 | 2.071 |
| 10_quads | 148 | 1.757 | 1.000 | 7.000 | 8.000 | 2.164 |
| 100_quads | 148 | 2.838 | 2.000 | 8.000 | 8.000 | 2.493 |
| 1000_quads | 148 | 10.014 | 11.000 | 13.000 | 18.000 | 2.840 |

**Finding:** 1 quad = 1.53ms, 1000 quads = 10.01ms (7× scaling). 
    Per-quad incremental cost: 8.49µs. 
    For a real graph with 1000 items, the picking pass alone costs ~10.0ms per refresh — 
    within frame budget. 
    This is the bottleneck refresh discipline (Layer 1) addresses by skipping the picking pass when nothing changed.

## Test K: Framebuffer size impact

| Condition | N | Mean (ms) | Median | p95 | p99 | StdDev |
|---|---|---|---|---|---|---|
| fb_256 | 148 | 2.345 | 1.000 | 8.000 | 8.000 | 2.418 |
| fb_512 | 148 | 2.919 | 2.000 | 8.000 | 8.000 | 2.616 |
| fb_1024 | 148 | 5.358 | 4.000 | 9.000 | 10.000 | 2.758 |
| fb_2048 | 148 | 11.831 | 12.000 | 18.000 | 19.000 | 2.786 |

**Finding:** 256×256 = 2.34ms, 2048×2048 = 11.83ms 
    (5.0× cost for 64× pixels). 
    Cost scales sub-linearly with framebuffer size — GPU is efficient at large rasterization.
    For Sigma users running on 4K displays, the picking pass cost may be substantially higher than benchmarks suggest.

## Test L: PICKING_MODE bailout at scale

| Condition | N | Mean (ms) | Median | p95 | p99 | StdDev |
|---|---|---|---|---|---|---|
| 1_quads_with_bailout | 148 | 1.541 | 1.000 | 7.000 | 8.000 | 2.077 |
| 1_quads_no_bailout | 148 | 1.615 | 1.000 | 7.000 | 8.000 | 2.207 |
| 10_quads_with_bailout | 148 | 1.851 | 1.000 | 8.000 | 8.000 | 2.361 |
| 10_quads_no_bailout | 148 | 1.716 | 1.000 | 7.000 | 8.000 | 2.125 |
| 100_quads_with_bailout | 148 | 2.304 | 1.000 | 8.000 | 8.000 | 2.415 |
| 100_quads_no_bailout | 148 | 2.980 | 2.000 | 8.000 | 9.000 | 2.508 |
| 1000_quads_with_bailout | 148 | 4.446 | 3.000 | 10.000 | 10.000 | 2.909 |
| 1000_quads_no_bailout | 148 | 8.588 | 9.000 | 12.000 | 13.000 | 2.463 |

**Finding:** bailout speedup at 1 quad: 1.05×. 
    At 1000 quads: 1.93× 
    (8.6ms → 4.4ms, 4.1ms saved per frame). 
    The bailout's per-quad savings compound linearly with edge count. 
    For Sigma users with custom shaders, this single fragment shader line is potentially the largest single optimization available 
    without changing Sigma's architecture.

## Test M: Concurrent JS work during async readback

| Condition | N | Mean (ms) | Median | p95 | p99 | StdDev |
|---|---|---|---|---|---|---|
| sync_blocked_time | 148 | 4.230 | 3.000 | 9.000 | 10.000 | 2.694 |
| async_elapsed | 80 | 14.787 | 15.000 | 18.000 | 21.000 | 1.849 |
| async_concurrent_work_iters | 80 | 3.750 | 4.000 | 8.000 | 13.000 | 2.188 |

**Finding:** sync readPixels blocks the main thread for 4.23ms (no JS can run). 
    Async readPixels takes 14.79ms wall time, but during that time the main thread completed 
    ~4 chunks of JS work (≈37500 math operations). 
    This is async readback's real value: it doesn't reduce total time, but enables productive concurrent work. 
    Critical for keeping React updates, layout calculations, and other UI work responsive during hover.

## Test N: Realistic graph simulation

| Condition | N | Mean (ms) | Median | p95 | p99 | StdDev |
|---|---|---|---|---|---|---|
| 1000 quads, normal render (no picking) | 148 | 7.662 | 7.000 | 12.000 | 13.000 | 2.540 |
| 1000 quads, picking pass WITH bailout | 148 | 3.865 | 3.000 | 9.000 | 10.000 | 2.376 |
| 1000 quads, picking pass NO bailout | 148 | 7.027 | 6.000 | 11.000 | 12.000 | 2.233 |

**Finding:** normal render of 1000 heavy quads: 7.66ms. 
    Adding a picking pass with NO bailout: 7.03ms (+-0.6ms picking cost). 
    Adding picking pass WITH bailout: 3.86ms (+-3.8ms picking cost). 
    Bailout reduces picking pass overhead by -498% (1.82× faster). 
    For a real graph at scale, this is the strongest no-code-change recommendation Sigma can offer custom-shader authors.