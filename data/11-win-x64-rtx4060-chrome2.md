# WebGL Picking Pipeline Benchmark v3 — scale & concurrency

## Environment

- **User agent:** `Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/148.0.0.0 Safari/537.36`
- **WebGL version:** WebGL2
- **GPU vendor:** `Google Inc. (NVIDIA)`
- **GPU renderer:** `ANGLE (NVIDIA, NVIDIA GeForce RTX 4060 (0x00002882) Direct3D11 vs_5_0 ps_5_0, D3D11)`
- **Timestamp:** 2026-05-24T19:35:21.025Z

## Test J: Draw call scaling

| Condition | N | Mean (ms) | Median | p95 | p99 | StdDev |
|---|---|---|---|---|---|---|
| 1_quads | 148 | 0.584 | 0.400 | 2.600 | 3.200 | 0.749 |
| 10_quads | 148 | 0.627 | 0.400 | 2.600 | 7.200 | 0.973 |
| 100_quads | 148 | 0.633 | 0.400 | 2.200 | 2.800 | 0.549 |
| 1000_quads | 148 | 1.598 | 1.300 | 3.300 | 3.700 | 0.715 |

**Finding:** 1 quad = 0.58ms, 1000 quads = 1.60ms (3× scaling). 
    Per-quad incremental cost: 1.02µs. 
    For a real graph with 1000 items, the picking pass alone costs ~1.6ms per refresh — 
    within frame budget. 
    This is the bottleneck refresh discipline (Layer 1) addresses by skipping the picking pass when nothing changed.

## Test K: Framebuffer size impact

| Condition | N | Mean (ms) | Median | p95 | p99 | StdDev |
|---|---|---|---|---|---|---|
| fb_256 | 148 | 0.493 | 0.300 | 1.100 | 3.000 | 0.717 |
| fb_512 | 148 | 0.528 | 0.400 | 1.100 | 2.900 | 0.707 |
| fb_1024 | 148 | 0.912 | 0.500 | 3.100 | 4.500 | 0.943 |
| fb_2048 | 148 | 1.517 | 1.300 | 3.800 | 5.100 | 0.825 |

**Finding:** 256×256 = 0.49ms, 2048×2048 = 1.52ms 
    (3.1× cost for 64× pixels). 
    Cost scales sub-linearly with framebuffer size — GPU is efficient at large rasterization.
    For Sigma users running on 4K displays, the picking pass cost may be substantially higher than benchmarks suggest.

## Test L: PICKING_MODE bailout at scale

| Condition | N | Mean (ms) | Median | p95 | p99 | StdDev |
|---|---|---|---|---|---|---|
| 1_quads_with_bailout | 148 | 0.226 | 0.200 | 0.300 | 0.500 | 0.082 |
| 1_quads_no_bailout | 148 | 0.225 | 0.200 | 0.300 | 0.500 | 0.076 |
| 10_quads_with_bailout | 148 | 0.239 | 0.200 | 0.300 | 0.400 | 0.063 |
| 10_quads_no_bailout | 148 | 0.256 | 0.200 | 0.300 | 0.500 | 0.341 |
| 100_quads_with_bailout | 148 | 0.367 | 0.400 | 0.500 | 0.500 | 0.074 |
| 100_quads_no_bailout | 148 | 0.347 | 0.300 | 0.500 | 0.500 | 0.076 |
| 1000_quads_with_bailout | 148 | 0.628 | 0.600 | 0.800 | 1.000 | 0.097 |
| 1000_quads_no_bailout | 148 | 1.301 | 1.300 | 1.600 | 1.900 | 0.181 |

**Finding:** bailout speedup at 1 quad: 0.99×. 
    At 1000 quads: 2.07× 
    (1.3ms → 0.6ms, 0.7ms saved per frame). 
    The bailout's per-quad savings compound linearly with edge count. 
    For Sigma users with custom shaders, this single fragment shader line is potentially the largest single optimization available 
    without changing Sigma's architecture.

## Test M: Concurrent JS work during async readback

| Condition | N | Mean (ms) | Median | p95 | p99 | StdDev |
|---|---|---|---|---|---|---|
| sync_blocked_time | 148 | 0.353 | 0.300 | 0.500 | 0.600 | 0.096 |
| async_elapsed | 80 | 5.448 | 5.100 | 6.800 | 8.400 | 0.675 |
| async_concurrent_work_iters | 80 | 5.888 | 6.000 | 10.000 | 11.000 | 2.554 |

**Finding:** sync readPixels blocks the main thread for 0.35ms (no JS can run). 
    Async readPixels takes 5.45ms wall time, but during that time the main thread completed 
    ~6 chunks of JS work (≈58875 math operations). 
    This is async readback's real value: it doesn't reduce total time, but enables productive concurrent work. 
    Critical for keeping React updates, layout calculations, and other UI work responsive during hover.

## Test N: Realistic graph simulation

| Condition | N | Mean (ms) | Median | p95 | p99 | StdDev |
|---|---|---|---|---|---|---|
| 1000 quads, normal render (no picking) | 148 | 0.703 | 0.700 | 0.900 | 1.300 | 0.133 |
| 1000 quads, picking pass WITH bailout | 148 | 0.811 | 0.800 | 1.100 | 1.400 | 0.152 |
| 1000 quads, picking pass NO bailout | 148 | 1.425 | 1.400 | 1.800 | 2.100 | 0.218 |

**Finding:** normal render of 1000 heavy quads: 0.70ms. 
    Adding a picking pass with NO bailout: 1.42ms (+0.7ms picking cost). 
    Adding picking pass WITH bailout: 0.81ms (+0.1ms picking cost). 
    Bailout reduces picking pass overhead by 85% (1.76× faster). 
    For a real graph at scale, this is the strongest no-code-change recommendation Sigma can offer custom-shader authors.
