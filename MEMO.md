# Empirical findings on the WebGL hit-detection pipeline

*A cross-platform investigation of `gl.readPixels`, picking-pass overhead, and where graph-visualization frameworks lose hover budget. Findings, methodology, and seven platform-specific measurement runs.*

— Ryan / bitmosh.dev — May 2026

---

## Abstract

Building a graph-visualization workbench on top of [Sigma.js](https://www.sigmajs.org/) and a custom plasma-edge fragment shader, I hit a hover-interaction performance wall I could not initially explain. The Chrome devtools profile pointed at a single `gl.readPixels` call costing 45.9ms per hover frame, but isolated benchmarks of `readPixels` showed sub-millisecond cost. The 450× gap between profile and benchmark turned out to be the whole story.

I built a reproducible benchmark suite to instrument the picking pipeline (region-size scaling, GPU load scaling, sync vs async readback, framebuffer-size effects, fragment-shader bailout, and concurrent main-thread work) and ran it across seven platform configurations: NVIDIA RTX 4070 (Linux/Chrome, Linux/Firefox), RTX 4060 (Windows 11/Chrome), Intel HD 4000 (Linux/Firefox, macOS Catalina/Safari), Mali-G610 ARM (Linux/Chromium), and SwiftShader software rendering (Linux/Vivaldi). 15 runs, roughly 600 individual measurements.

The findings:

1. `gl.readPixels` is not slow. What feels slow is the synchronous wait it forces on previously-queued GPU work. Cost in isolation: 0.05-0.15ms. Cost when the GPU is loaded with a heavy fragment-shader queue: 0.4-20ms.
2. Picking-pass cost scales with edge count, fragment-shader complexity, and framebuffer size, all three of which compound in real graph-viz workloads.
3. A single line of GLSL (early-returning from the fragment shader when picking-mode is set) produces measurable speedups on every platform tested. Magnitude scales inversely with pipeline efficiency. Top-tier hardware sees roughly 1.3× speedup. Older Intel hardware sees 2.0-2.3×. Safari on Catalina sees 4.6× (the difference between "under" and "over" the 16ms frame budget for a 1000-element graph).
4. WebKit on Apple Metal has a measurable additional overhead on the picking path, consistent with [WebKit bug #235002](https://bugs.webkit.org/show_bug.cgi?id=235002). Direct comparison: same hardware, same shader, Firefox/Linux 5-10ms vs Safari/macOS 20ms.
5. Async `readPixels` via `glFenceSync` + `glClientWaitSync` does not reduce total time. It releases the main thread, which is the actual win for hover-interaction responsiveness.

This document presents the methodology, the data, and a layered set of optimizations applicable to Sigma.js, Three.js, and any library that does GPU picking via colored framebuffer readback. The benchmark suite is hosted at [bitmosh.dev/labs/picking-benchmark](https://bitmosh.dev/labs/picking-benchmark) and is fully open-source. Run it yourself in under a minute.

---

## How I got here

I'm building [LumaWeave](https://bitmosh.dev), a graph visualization and theming workbench. It renders graphs of roughly 1000 nodes and edges using Sigma.js, with custom WebGL programs for node and edge rendering, including a plasma edge shader doing 5-octave FBM noise, gradient interpolation, and a bloom approximation in the fragment stage.

The plasma shader is expensive on purpose. It looks great. Edges flow, glow, and respond to hover. After integrating it into the in-development build, I noticed hover felt sluggish on a graph with 846 edges, even on an upper-tier consumer NVIDIA RTX 4070 Super. The performance budget felt off.

A Chrome devtools profile showed a tall column of `gl.readPixels` at 45.9ms in a single hover frame. That was the entire frame budget gone, plus most of the next frame, just to read one pixel of color data back from the GPU. Reading one pixel takes microseconds at the API level. Something else was happening.

The standard explanations didn't quite fit. Sigma already uses a texture-backed framebuffer for picking (the recommended pattern from [Gregg Tavares's WebKit-Metal slow-path work](https://bugs.webkit.org/show_bug.cgi?id=235002)). I'd already switched expensive `refresh()` calls to `scheduleRender()` where possible. Async readback would conceptually help but adds its own overhead and changes the contract.

So I built the benchmark suite, in pieces, over several days. What I found refined and in some places contradicted my initial hypotheses. The story isn't "readPixels is slow." The story is closer to: **`readPixels` is a synchronization point, and the cost you see at that point is everything the GPU still owes you.**

---

## How GPU picking works (briefly, for context)

In a 2D WebGL framework like Sigma, every renderable item (node, edge, label) gets assigned a unique numeric ID at draw time. To detect what's under the cursor, the framework runs a second draw pass (the *picking pass*) where each item renders its own ID encoded as an RGBA color. The result lives in a texture-backed framebuffer that isn't shown on screen.

When the cursor moves, the framework calls `gl.readPixels` at the mouse coordinates, reads back a single RGBA value from the picking framebuffer, decodes that back into an ID, and reports it as "the hovered item." Sigma's implementation lives in `packages/sigma/src/utils/colors.ts:158` and is a 1×1 RGBA read.

The system is conceptually simple. The performance characteristics are not.

There are three places cost accumulates:

1. **Rendering the picking framebuffer.** Every visible item gets drawn, with the same fragment shader (or a picking variant), producing a colored ID instead of visual output. If your fragment shader is expensive (and most custom shaders are), the picking pass costs roughly what the visible-scene render costs.

2. **GPU work synchronization.** `gl.readPixels` is, by spec, a blocking call. It cannot return until all previously-queued GPU commands have completed. If you've queued 50 expensive plasma-shader draws in the prior frame and then call `readPixels`, the call doesn't return until those 50 draws have finished. The wait shows up as the cost of `readPixels`.

3. **Driver and browser overhead.** Different browsers handle the readback path differently. Chrome uses Google's ANGLE translation layer (OpenGL→D3D11 on Windows, OpenGL→OpenGL on Linux, OpenGL→Metal on macOS). Firefox on Linux uses native OpenGL directly. Safari on Apple Metal has a known slow path for non-texture-backed framebuffers. The cost varies meaningfully across these.

The benchmark suite separates these three sources of cost.

---

## Methodology

The benchmark is a single self-contained HTML file. It creates a WebGL2 context, renders quads with three fragment shaders of increasing complexity (flat color → moderate sine math → 5-octave FBM noise + bloom), and measures `readPixels` cost under varying conditions.

Each test condition runs 148-196 iterations after a 20-50 iteration warmup. Statistics reported are trimmed mean (1% top/bottom outliers removed), median, p95, p99, and standard deviation. The suite displays which scene is currently rendering so the test conditions are visible.

What the suite measures:

| Test | Question it answers |
|---|---|
| A | Does payload size matter? (1×1 to 32×32 reads) |
| B | Does cost scale with queued GPU work? |
| C | Sync vs async readback: what's the actual difference? |
| D | Texture-backed FBO vs default canvas |
| E | Fragment shader complexity impact on picking |
| F | PICKING_MODE early-return savings |
| G | Frame-to-frame variance under sustained load |
| H | Warmup effects on first calls |
| J | Picking pass cost scaling with edge count (1-1000 quads) |
| K | Framebuffer size impact (256² to 2048²) |
| L | Bailout speedup at production scale |
| M | Useful main-thread work during async readback |
| N | Realistic 1000-quad graph simulation |

The full suite runs in about three minutes. Source: [github.com/lumaweave/picking-benchmark](https://github.com/lumaweave/picking-benchmark) (or whatever the eventual repo path is; view-source works as a fallback).

**Honest methodological caveats.** Browser `performance.now()` has variable granularity for fingerprint protection. Chrome reports ~0.1ms resolution. Firefox quantizes to 1ms. Safari quantizes to 1ms. This means sub-millisecond effects on Firefox/Safari are below the measurement floor, and individual samples cluster at integer values. The trimmed mean across 148+ samples still produces meaningful signal, but I avoid making claims tighter than the resolution allows.

Firefox also lies about the GPU it's running on (reports "GeForce GTX 980, or similar" on an RTX 4070, "Intel HD Graphics, or similar" on Intel HD 4000). Same fingerprint-protection rationale. I confirm hardware via the system itself, not the WebGL reporter.

Safari reports "Apple GPU" regardless of underlying hardware. I tested on a mid-2012 MacBook Pro 9,2 with an Intel HD 4000. Safari does not report that information to web content.

---

## Findings

### Finding 1: `readPixels` payload size doesn't matter for small reads

Tested 1×1 through 32×32 reads on RTX 4070/Chrome/Linux. Cost is flat across this range (0.085-0.105ms mean). The 32× pixel increase produces no measurable cost increase. There is no readback-cost benefit to reading exactly one pixel versus reading a small region. The per-pixel transfer cost is dominated by fixed overhead at this scale.

This matters because some optimization advice online suggests reading larger regions for "warmup" or "alignment" reasons. The data does not support that on modern hardware for typical picking workloads.

### Finding 2: `readPixels` cost scales with prior queued GPU work

This is the most consequential finding from the suite. The same 1×1 read becomes 5-9× more expensive when preceded by heavy GPU work in the same frame:

| GPU load (RTX 4070 / Chrome / Linux) | Mean readPixels cost |
|---|---|
| idle (1 draw, simple shader) | 0.088ms |
| light (1 draw, moderate shader) | 0.093ms |
| medium (1 draw, heavy shader) | 0.164ms |
| heavy (10 draws, heavy shader) | 0.614ms |
| extreme (50 draws, heavy shader) | 0.784ms |

The cost isn't the read. The cost is waiting for the GPU's command queue to drain so the read can return.

In real graph-viz workloads the queue can be very deep. LumaWeave renders 846 edges with plasma fragment shaders plus 276 nodes plus labels plus background overlays, every frame. The picking pass repeats that work, meaning by the time a hover-triggered `readPixels` fires, the GPU has roughly 2200+ draw calls queued ahead of it, many doing 5-octave noise math.

My initial production profile of 45.9ms is consistent with this. It is not a measurement of `readPixels` slowness. It is a measurement of how much GPU work was queued when `readPixels` was called.

The implication for optimization is significant. You can't make `readPixels` itself faster. You can:

- Reduce the GPU work that needs to drain before it (refresh discipline, picking-pass shader bailout, capped picking framebuffer)
- Move the wait off the main thread (async readback via fence)
- Avoid calling it as often (input throttling, picking-buffer caching)

The benchmark surfaces all of these as separate measurements.

### Finding 3: Fragment shader complexity dominates the picking pass at scale

For Sigma.js users with custom edge or node programs, this is likely the most actionable finding in the suite.

Test L measures the cost of rendering N quads with a heavy fragment shader, with and without a `PICKING_MODE` early-return in the shader:

```glsl
void main() {
  if (u_pickingMode) {
    gl_FragColor = u_pickingColor;
    return;
  }
  // ... expensive fragment work
}
```

The early-return skips the expensive math when the picking pass runs, since the fragment output during picking is just the encoded ID. The visible scene render still does the expensive work.

Speedup across seven platform configurations at 1000 quads:

| Platform | Bailout speedup @ 1000 quads | Time saved per frame |
|---|---|---|
| Firefox / Linux / RTX 4070 Super | 1.00× | 0.0ms |
| Chrome / Linux / RTX 4070 Super | 1.29-1.39× | 0.1ms |
| Chrome / Windows 11 / RTX 4060 | 1.34× | 0.3ms |
| Vivaldi / Linux / SwiftShader (CPU) | 1.27-1.34× | 10-13ms |
| Chromium / Linux / Mali-G610 (ARM) | 1.61-1.68× | 2.5-2.6ms |
| Firefox / Linux / Intel HD 4000 | 1.93-2.34× | 2.7-3.4ms |
| Safari / macOS Catalina / Intel HD 4000 | 4.64-4.65× | 16.3-16.5ms |

The pattern is monotonic: the slower the rendering pipeline, the larger the bailout's value. Safari/Catalina is the dramatic case: without bailout, a 1000-quad picking pass costs 20.7-21.1ms (over the 16ms frame budget). With bailout, 4.5ms (well under). The same single GLSL line crosses the playable/unplayable threshold.

On Firefox/Linux/RTX 4070 Super, the bailout shows no measurable effect. The most likely explanation is that Firefox on Linux uses native OpenGL directly (no ANGLE translation), and the fragment work at this scale is already so cheap that the bailout has nothing to save. This is a counterexample worth noting: not all users will see benefit. But the counterexample is the user least in need of help.

For library maintainers, this changes the framing. The bailout isn't a "performance tweak for power users." It's a quality-of-life fix that disproportionately helps users on weaker hardware, the users who feel the pain. It's also approximately zero code: a single conditional at the top of any custom fragment shader.

### Finding 4: Async readback shifts wait time off the main thread; it does not reduce total time

I initially hypothesized that switching from sync `gl.readPixels` to async (WebGL2's `glFenceSync` + `glClientWaitSync` polling pattern) would reduce hover latency. The data says otherwise.

| Condition (RTX 4070 / Chrome / Linux) | Mean wall time | Main thread blocked? |
|---|---|---|
| Sync read, idle GPU | 0.05ms | Yes, 0.05ms |
| Async read, idle GPU | 4.13ms | No |
| Sync read, heavy GPU | 0.43ms | Yes, 0.43ms |
| Async read, heavy GPU | 4.09ms | No |

Async has a fixed ~4ms minimum from the fence-poll loop (`setTimeout` minimum interval, fence resolution roundtrip). Under light load, async takes 80× longer in wall time. Under heavy load, async still loses on wall time but the gap narrows.

What async wins is concurrent main-thread work. Test M measures how much JavaScript work can complete on the main thread during an async readback. Under sync, the answer is zero. The main thread is blocked for the duration of the call. Under async, the main thread is free.

Concurrent work completed during async readback (chunks of ~10,000 math operations each):

| Platform | Sync block time | Async wall time | Concurrent work chunks |
|---|---|---|---|
| Firefox / Linux / RTX 4070 | 0.22ms | 8.4ms | 13-15 |
| Chrome / Linux / RTX 4070 | 0.18ms | 4.7ms | 8-10 |
| Chrome / Win11 / RTX 4060 | 0.58ms | 6.1ms | 5 |
| Mali-G610 | 2.2-2.4ms | 5.2-5.5ms | 2-3 |
| Firefox / Intel HD 4000 | 4.0-4.2ms | 10-15ms | 2-4 |
| Safari / Catalina | 5.0ms | 11.4ms | 2 |
| SwiftShader / CPU | 10.8-11.0ms | 14.9-15.3ms | 2-3 |

On slow setups, sync `readPixels` already blocks the main thread for 4-11ms per call. That's a perceptible pause in hover responsiveness. Async lets layout, React updates, and other UI work proceed during that wait. The total time isn't lower, but the user perceives the UI as responsive rather than stuck.

For Sigma users, the implication is conditional. On fast hardware (Chrome+top GPU), sync is cheaper and main-thread blocking is short enough not to feel like a stall. On slow hardware, the wait is long enough to feel sluggish, and async is the better trade even though it's "slower" in total time.

A library could pick adaptively. Measure the first few `readPixels` durations on init, and if they consistently exceed some threshold (say 3ms), switch to async readback. The benchmark doesn't validate this directly, but the conditions for adaptive selection are visible in the per-platform numbers.

### Finding 5: Texture-backed framebuffer for picking appears to be the safer cross-platform default

Test D compares reading from a texture-backed framebuffer object (FBO) versus reading directly from the default canvas:

| Platform | Canvas readPixels | Texture FBO readPixels | Ratio |
|---|---|---|---|
| Chrome / Linux / RTX 4070 | 0.058ms | 0.057ms | 0.98× |
| Chrome / Linux / RTX 4070 (heavy load) | 0.524ms | 0.439ms | 1.19× |

On ANGLE/OpenGL the difference is within noise. On WebKit/Metal (per Gregg Tavares's investigation), the texture-backed path can be 10× faster because Metal lacks an efficient row-by-row readback equivalent and the canvas-backed path falls back to a slow synchronous variant.

Sigma already uses a single texture-backed FBO for picking (verified in `packages/sigma/src/sigma.ts:1721`), which based on the cross-platform data appears to be the safer default. No change is needed at the framework level. But for any custom WebGL code doing GPU picking (Three.js examples, in-house renderers, framework starter templates), defaulting to texture-FBO appears to be the safer cross-platform pattern based on the data here. The bug report at WebKit makes this clear, but it's worth surfacing more visibly in MDN's WebGL best-practices guidance and in framework docs.

### Finding 6: Framebuffer size affects cost more on some backends than others

Test K renders 100 heavy quads at framebuffer sizes from 256×256 to 2048×2048 (a 64× pixel increase):

| Platform | 256² cost | 2048² cost | Ratio for 64× pixels |
|---|---|---|---|
| Chrome / Linux / RTX 4070 | 0.10ms | 0.41ms | 4.1× |
| Firefox / Linux / RTX 4070 | 0.17ms | 0.22ms | 1.3× |
| Chrome / Windows 11 / RTX 4060 | 0.38ms | 0.54ms | 1.4× |
| Mali-G610 | 1.2ms | 4.5ms | 3.8× |
| Firefox / Linux / Intel HD 4000 | 1.8ms | 11.2ms | 6.2× |
| Safari / Catalina / Intel HD 4000 | 2.3ms | 20.5ms | 8.9× |
| SwiftShader / CPU | 4.0ms | 55.4ms | 13.9× |

The ANGLE/OpenGL path on Chrome/Linux shows steeper framebuffer-size scaling than ANGLE/D3D11 on Windows. Firefox/Linux on top-tier hardware shows essentially flat scaling. Safari is dramatically scale-sensitive. At 2048², the picking pass alone is 20.5ms, over frame budget.

This surfaces a proposal that isn't currently in Sigma: **render the picking buffer at a capped resolution, independent of viewport size.** For hit detection, single-pixel precision isn't necessary. A picking buffer capped at 1024² or even 512², with mouse coordinates scaled at read time, would mean:

- Picking-pass cost is fixed regardless of display resolution
- Users on 4K displays don't pay 4× the picking cost for no UX gain
- The cost reduction is dramatic on platforms where framebuffer scaling is steep (Linux/Chrome, Safari, SwiftShader)

The trade-off is sub-pixel precision in hit detection. For nodes and edges with typical visible sizes (4+ pixels), this is invisible to users. There is also an unexpected side benefit: if the picking buffer is downsampled by 2× or 4×, each picking-pixel represents 2-4 viewport pixels, which naturally widens hit-test regions for narrow elements. Thin edges that are difficult to click in the visible scene become correspondingly easier to select. The optimization compensates for a UX paper cut that some custom-edge-program authors deal with separately.

### Finding 7: Software-rendering fallbacks are a real production hazard

Accidentally, the Orange Pi 5 Max test surfaced a real-world failure mode. On the same hardware (ARM64 with Mali-G610 GPU), Chromium correctly used Panfrost (the open-source Mali driver) for hardware acceleration. Vivaldi, on the default config, fell back to SwiftShader, which is software-rendered WebGL on the CPU.

The performance difference is severe:

| Backend on Orange Pi 5 Max | 1000-quad picking pass |
|---|---|
| Chromium / Panfrost (hardware) | 6.4-9.2ms |
| Vivaldi / SwiftShader (software) | 47.9-48.0ms |

That's a 7× slowdown from software fallback on otherwise-identical hardware. Real users on Linux setups (especially ARM, less-common GPU configs, browser-specific driver issues) can end up in this state without realizing it. Framework documentation should mention how to detect software rendering (`gl.getParameter(gl.RENDERER)` reveals it via "SwiftShader" or similar in the string) and how to advise users about it.

For LumaWeave specifically, this also explains why interactive performance on the Orange Pi 5 Max sits around 12-16 FPS on Chromium and 3-5 FPS on Vivaldi. That's the software-rendering tax compounded across every frame of the visible-scene render plus picking pass. Switching from Vivaldi to Chromium with Panfrost is one of the bigger single improvements available to users on that hardware class.

---

## Cross-platform summary

Consolidated view of the picking pass at production scale (1000 heavy quads, mid-resolution framebuffer):

| Platform | Normal render | Picking, no bailout | Picking, with bailout | Bailout savings |
|---|---|---|---|---|
| Firefox / Linux / RTX 4070 Super | 0.72ms | 0.70ms | 0.68ms | ~0ms |
| Chrome / Linux / RTX 4070 Super | 0.52ms | 0.46ms | 0.33ms | 0.13ms |
| Chrome / Windows 11 / RTX 4060 | 1.17ms | 1.53ms | 1.72ms | -0.2ms* |
| Chromium / Linux / Mali-G610 (ARM) | 6.3ms | 6.4ms | 4.0ms | 2.4ms |
| Firefox / Linux / Intel HD 4000 | 5.4ms | 5.3ms | 2.6ms | 2.7ms |
| Safari / Catalina / Intel HD 4000 | 21.2ms | 21.0ms | 4.8ms | 16.2ms |
| Vivaldi / Linux / SwiftShader (CPU) | 49.7ms | 50.7ms | 36.3ms | 14.4ms |

\* The Windows 11 result inverts slightly at this specific test, within run-to-run variance. The 1000-quad bailout test in the same run shows the expected 1.34× speedup. Single-machine, single-run results in tight ranges should be treated as directional.

Across this span (roughly 275× from fastest to slowest configuration), the consistent pattern is that the bailout helps more as the pipeline gets slower, with the dramatic case being Safari on older Apple hardware, where it crosses frame budget.

---

## Recommendations

I'll order these by audience and by impact, from broadest to narrowest.

### For developers using Sigma.js or any GPU-picking framework with custom fragment shaders

**Add a `PICKING_MODE` bailout to every custom fragment shader.** In my testing this was consistently the highest-impact change for the smallest amount of code. A single GLSL conditional:

```glsl
uniform bool u_pickingMode;
uniform vec4 u_pickingColor;

void main() {
  if (u_pickingMode) {
    gl_FragColor = u_pickingColor;
    return;
  }
  // ... your normal fragment work
}
```

Your custom node/edge program needs to set `u_pickingMode` to `true` when called for the picking pass and `false` otherwise, and set `u_pickingColor` to the per-instance picking color the framework expects.

In Sigma 3.x, this means wiring the picking-color logic in `processVisibleItem` (the abstract method on `NodeProgram` and `EdgeProgram` in `packages/sigma/src/rendering/node.ts:55` and `edge.ts:65`) into uniform updates rather than into vertex attributes. The implementation is a small refactor with documented benefit.

Expected savings (empirical, see cross-platform table above): from undetectable on top-tier hardware to ~16ms per hover frame on Safari/Catalina.

### For Sigma.js maintainers specifically

The following are framed as observations rather than requests. Sigma is a healthy and well-maintained project, and these are proposed additions to existing solid architecture, not corrections.

1. **Picking-buffer caching when nothing affecting picking has changed.** Camera-stable, position-stable, topology-stable frames don't need re-rendered picking buffers. Sigma already has a `needRedraw` and a `needRender` flag pair; a similar `needPickingRedraw` flag, set only on relevant state changes, would skip the picking re-render on most hover moves. This generalizes a pattern I implemented during recent LumaWeave development, applied at the library level rather than per-application.

2. **Capped-size picking framebuffer (proposed).** Render the picking buffer at min(viewport, configurable_cap), with mouse coordinates scaled at read time. Default cap of 1024² or 1536² would meaningfully reduce picking cost on large-display deployments. Trade-off is sub-pixel precision, which is irrelevant for typical node/edge hit-test sizes.

3. **`PICKING_MODE` documentation in custom-program guides.** The pattern above isn't currently documented as a recommended idiom for custom programs, but the data suggests it should be. A line in the [custom programs guide](https://www.sigmajs.org/docs/advanced/renderers/) plus updated examples would convert this from "trick I noticed while profiling" to "thing every Sigma user knows."

4. **Adaptive mousemove throttling, optional.** The captor's `handleMove` (`packages/sigma/src/core/captors/mouse.ts:238`) currently binds raw mousemove with `{ capture: false }` and no library-level throttling. An optional adaptive throttle, EWMA-based with hysteresis, could keep hover responsive on weak hardware without sacrificing precision on strong hardware. This is more architectural than the other items and might fit the v4 roadmap discussion rather than v3 maintenance.

### For Three.js maintainers and other WebGL framework authors

Async `readPixels` via fence is documented as a performance improvement, but as the data shows, it's a *responsiveness* improvement, not a throughput improvement. The Three.js issue [#23550](https://github.com/mrdoob/three.js/issues/23550) discusses this; the framing in linked example code occasionally suggests async will reduce total time, which is incorrect. Total wall time goes up under async; what goes down is main-thread blocking.

The clearest documentation framing is: *"Use `readPixelsAsync` when your application needs to do other JavaScript work during the readback (UI updates, layout, React reconciliation). Use synchronous `readPixels` when you need the result immediately and want minimum total time."*

### For WebKit / Safari engineers

[WebKit bug #235002](https://bugs.webkit.org/show_bug.cgi?id=235002) documents the Apple Metal slow path for `readPixels`. The empirical signature on Safari/Catalina/Intel HD 4000 matches: same hardware shows 2.4-4× higher picking-pipeline cost under Safari/macOS than under Firefox/Linux. This is data the bug filers could reference; to the best of my knowledge the issue is still open and active.

A specific note from the data: framebuffer-size scaling is dramatically worse on Safari than on other browsers. At 2048² framebuffer, Safari pays 20.5ms versus Firefox/Linux 11.2ms on the same hardware. The slow path appears to compound with rasterization work, not just be a fixed overhead. This may help isolate where in the Metal path the regression originates.

A more efficient row-by-row readback equivalent for Metal is presumably engineerable at the WebKit/Apple level. From the application-developer side we can only work around the slow path by using texture-backed framebuffers (which Sigma already does); a true fix would require changes upstream in the Metal compositor pipeline, which is beyond what framework or app authors can address.

### For MDN and WebGL documentation maintainers

A note in the WebGL best-practices guide about texture-backed framebuffers being safer than the default canvas for cross-platform `readPixels` performance would help library authors avoid the WebKit Metal trap pre-emptively. The existing guidance covers it but it can be easy to miss.

---

## Where I was wrong and what changed

I want to be explicit about places this investigation revised my initial hypotheses, because the corrections are themselves data:

**I initially thought async `readPixels` would be the headline optimization.** The data shows it doesn't reduce total time. It releases the main thread. That's still valuable, but it's a different value than I first claimed.

**I initially thought the `PICKING_MODE` bailout would show its dramatic value on top-tier hardware too.** It doesn't. On Firefox/Linux/RTX 4070 the effect is below measurement noise. The bailout shows its value inversely to pipeline efficiency, which makes intuitive sense once observed.

**I initially thought texture-backed framebuffer was a universal optimization.** It's not. It's a platform-specific safeguard. On ANGLE/OpenGL/Linux/Windows the difference is within noise. On WebKit/Apple Metal it's potentially 10×. Sigma already uses it appropriately for cross-platform compatibility, so the practical advice for Sigma users hasn't changed. The framing should probably be "this is platform-portable defense" rather than "this is faster everywhere."

**I initially overstated the LumaWeave production-profile correlation.** 45.9ms `readPixels` in production extrapolates from the benchmark roughly: heavy GPU queue depth from continuous plasma rendering plus picking-pass repetition plus React updates plus physics simulation. The benchmark doesn't replicate this exactly because it can't replicate a full application. The 45.9ms is consistent with the patterns shown, not directly proven by them.

**I initially missed software-rendering fallback as a category.** It only surfaced because of the Orange Pi data. Real users running browsers without hardware acceleration enabled see catastrophically degraded picking performance, and frameworks should help them detect that state.

---

## Open questions

A few things I haven't resolved and would value input on:

1. **Is there a sensible threshold for adaptive sync/async selection?** Above, I suggested 3ms sync-block time as a switching threshold. That number is hand-tuned, not measured. Empirically deriving the right cutoff would require user-perception data I don't currently have.

2. **What's the production cost of `processVisibleItem` itself at heavy edge counts?** This abstract method is called for every visible item in the picking pass and sits in Sigma's hot path (`packages/sigma/src/rendering/node.ts:55`). I haven't profiled it in isolation. It could be a worthwhile micro-optimization target, but I'd want measurements before recommending anything specific.

3. **Does Sigma's v4 architecture (currently in alpha at [v4.sigmajs.org](https://v4.sigmajs.org)) already address some of these?** The v4 roadmap discussion ([#1469](https://github.com/jacomyal/sigma.js/discussions/1469)) mentions performance improvements but the picking pipeline isn't explicitly called out. A maintainer perspective would help calibrate which of these recommendations are still relevant in v4 versus already addressed.

4. **WebGPU?** All measurements here are WebGL2. WebGPU has different readback semantics (`mapAsync` and friends). Whether the patterns I've identified persist or transform in WebGPU is unclear, and it's the direction most rendering work is heading. Re-running the benchmark with a WebGPU port would be valuable but is non-trivial work.

5. **Mobile browsers?** I haven't tested any phone or tablet directly. The Orange Pi data approximates mobile-class hardware (Mali-G610 is closely related to mobile Mali GPUs), but the browser environments differ. Real iOS Safari and Android Chrome measurements would refine the picture and surface mobile-specific quirks.

6. **Apple Silicon?** I tested Safari on Intel HD 4000 / Catalina, which exposed the Metal slow path. I have no data on Apple Silicon (M1/M2/M3/M4) under Safari. The Metal pipeline on Apple Silicon may be substantially different from the Intel-era Metal path, and the slow-path findings here may not generalize forward. Pending data from a friend running the suite on M4 hardware, which I plan to incorporate as a follow-up.

7. **Tile-based renderers vs immediate-mode renderers.** Mobile and Apple GPUs are typically tile-based deferred renderers, while desktop NVIDIA is immediate-mode. The cost model for `readPixels` is meaningfully different on tile-based architectures (tiles must be resolved to memory before readback can complete). The benchmark doesn't disambiguate this, and the patterns may need different framing on TBDR hardware.

8. **The 100% bailout case.** Test L's bailout shader does just `gl_FragColor = u_pickingColor; return;` at the top. Real applications may have setup work in the vertex shader that still runs during picking. The benchmark doesn't capture vertex-stage picking cost, which on instanced or vertex-heavy programs could be the dominant factor.

9. **High-density displays.** I tested at devicePixelRatio = 1 across all platforms. Retina, 4K, and HiDPI displays effectively multiply framebuffer pixel counts by 4-16×, depending on configuration. The framebuffer-size scaling data in Test K suggests this matters substantially on some backends, but I haven't run the benchmark at native DPR on any HiDPI machine.

10. **Background-tab behavior.** Browsers throttle requestAnimationFrame when tabs are unfocused. Whether this affects `readPixels` performance (e.g. by changing GPU power state) is something I noticed indirectly during testing but didn't measure. Could matter for applications that do background work involving GPU readback.

---

## Reproducibility

**Benchmark suite:** [bitmosh.dev/labs/picking-benchmark](https://bitmosh.dev/labs/picking-benchmark)

The full HTML source is viewable in DevTools. No external resources are loaded. No analytics, no telemetry, no tracking. The page generates the WebGL context, runs the tests, and produces results entirely client-side. Closing the tab discards everything.

**Source for the suite:** included on the same page as a `<details>` section, and viewable via View Source. Anyone can save the HTML and run it offline. The MIT license applies.

**Raw data:** all 15 benchmark runs (markdown format, ~200 measurements each) are available at [bitmosh.dev/labs/picking-benchmark/data/](https://bitmosh.dev/labs/picking-benchmark/data/). Each file includes the environment fingerprint, timestamp, and trimmed statistics for each test condition.

**Methodology details:**

- `performance.now()` resolution: 0.1ms on Chrome, 1ms on Firefox/Safari (fingerprint protection)
- Sample sizes: 148-196 iterations per condition, 20-50 warmup iterations discarded
- Statistics: trimmed mean (1% top/bottom), median, p95, p99, stddev
- All async tests use the `glFenceSync` + `glClientWaitSync` polling pattern recommended by [Khronos](https://www.khronos.org/opengl/wiki/Sync_Object) and [MDN](https://developer.mozilla.org/en-US/docs/Web/API/WebGL2RenderingContext/clientWaitSync), with 1ms `setTimeout` polling interval
- Test scenes are deterministic (same seed produces same quad layout across runs)
- Three identical runs per platform where possible, to capture run-to-run variance

**Platforms tested:**

1. NVIDIA RTX 4070 Super, x86_64, Ubuntu Linux, Chrome 148 (ANGLE/OpenGL backend)
2. NVIDIA RTX 4070 Super, x86_64, Ubuntu Linux, Firefox 151 (native OpenGL backend)
3. NVIDIA RTX 4060, x86_64, Windows 11, Chrome 148 (ANGLE/D3D11 backend)
4. Rockchip RK3588 Mali-G610, ARM64, Ubuntu 24.04 LTS, Chromium 114 (Panfrost driver)
5. Rockchip RK3588 Mali-G610, ARM64, Ubuntu 24.04 LTS, Vivaldi (SwiftShader software fallback)
6. Intel HD Graphics 4000, mid-2012 MacBook Pro 9,2, Ubuntu Linux, Firefox 147
7. Intel HD Graphics 4000, mid-2012 MacBook Pro 9,2, macOS Catalina 10.15.7, Safari 15.6.1

---

## About this document

I'm Ryan, a solo developer building LumaWeave. This investigation came out of trying to understand why hover performance felt off in my own product, and grew into a broader empirical look at how the WebGL picking pipeline behaves across consumer hardware.

LumaWeave is in active development. The picking-pipeline findings here are already partially implemented in the current private build (refresh-discipline cleanup, scheduleRender for hover paths, hover-state caching) and partially queued for future versions (capped picking framebuffer, PICKING_MODE bailout in custom programs, adaptive readback). The benchmark suite predates the memo and was built as a debugging tool. It just turned out to be useful as a reproducibility artifact.

I owe credit to Alexis Jacomy and the Sigma contributors for building a library careful enough that this kind of investigation is even possible. The texture-backed FBO usage, the lifecycle separation of `refresh`/`scheduleRender`, the abstract program base classes. These are good architectural decisions; I'm making proposals on top of solid foundations, not trying to fix anything broken.

Gregg Tavares's [WebKit bug investigation](https://bugs.webkit.org/show_bug.cgi?id=235002) was foundational for understanding the Metal slow path. The async readback pattern I use comes from his Three.js [issue thread](https://github.com/mrdoob/three.js/issues/23550) and the Babylon.js [forum thread](https://forum.babylonjs.com/t/how-to-speed-up-readpixelsasync/33240).

Questions, corrections, and disagreement are welcome at ryan@bitmosh.dev or via reply to this document wherever it landed for you (Sigma Discussion, X, my blog, etc.).

— Ryan

[bitmosh.dev](https://bitmosh.dev) · [@bitmosh](https://x.com/bitmosh) · May 2026
