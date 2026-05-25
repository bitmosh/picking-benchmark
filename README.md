# picking-benchmark

A self-contained benchmark suite for the WebGL hit-detection (`gl.readPixels`) pipeline. Runs in your browser, sends nothing back. Measures region-size scaling, framebuffer-size impact, GPU load, fragment-shader bailout, and concurrent main-thread work across browsers, GPUs, and drivers.

> **Live**: [bitmosh.dev/labs/picking-benchmark](https://bitmosh.dev/labs/picking-benchmark)
> **Memo**: [bitmosh.dev/research/picking-pipeline](https://bitmosh.dev/research/picking-pipeline) (mirror of [`MEMO.md`](./MEMO.md))
> **Raw data**: [`./data/`](./data/)

---

## What this is

A single-file HTML page that instruments `gl.readPixels` and the surrounding picking pipeline used by graph-visualization frameworks (Sigma.js, Three.js, and anything else doing GPU picking via colored framebuffer readback). Built originally as a debugging tool while profiling [LumaWeave](https://bitmosh.dev), turned into a reproducibility artifact when the findings warranted a memo.

The investigation produced five findings worth maintainers' attention. Summarized:

1. **`gl.readPixels` is not slow in isolation** — sub-millisecond on every platform tested. What feels slow is the synchronous wait it forces on previously-queued GPU work. Loaded GPUs pay 0.4–20ms at the readback point.
2. **Picking-pass cost scales with edge count, shader complexity, and framebuffer size**, all three of which compound in real graph-viz workloads.
3. **A one-line `PICKING_MODE` early-return in the fragment shader** produces measurable speedups on platforms where the pipeline isn't already maxed out by efficiency gains elsewhere. Magnitude scales inversely with pipeline efficiency: ~1.3× on top-tier hardware, 2.0–2.3× on older Intel, 4.6× on Safari/Catalina, no measurable benefit on modern Apple Silicon or Firefox/Linux/native-OpenGL.
4. **WebKit/Apple Metal had a measurable additional overhead** on the picking path on Intel-era macOS, consistent with [WebKit bug #235002](https://bugs.webkit.org/show_bug.cgi?id=235002). Modern Apple Silicon (M4 Max / macOS 26) shows this resolved.
5. **Async `readPixels` via `glFenceSync` does not reduce total time.** It releases the main thread, which is the actual win for hover responsiveness.

Full methodology, per-platform data, and recommendations for Sigma / Three.js / WebKit / MDN are in [`MEMO.md`](./MEMO.md).

## Quick start

Three ways to run it. Pick the one that fits.

**1. Use the hosted version.** Easiest. No download.
```
https://bitmosh.dev/labs/picking-benchmark
```

**2. Download and open locally.** No internet required after download.
```bash
curl -O https://raw.githubusercontent.com/bitmosh/picking-benchmark/main/picking-benchmark.html
# Open the file in any modern browser.
```

**3. Clone the repo.** If you plan to modify or contribute.
```bash
git clone https://github.com/bitmosh/picking-benchmark.git
cd picking-benchmark
# Open picking-benchmark.html in any modern browser.
```

Click **Run full v3 suite** for the canonical run (≈30s on a modern desktop, longer on weaker hardware), or **Quick run** for a faster smoke test. When it finishes, the Copy / Email buttons format the results as markdown.

## What it measures

Five test groups, ~600 measurements per full run:

- **Test J — Draw call scaling.** `readPixels` cost as the GPU queue depth grows. Maps to: how does hover slow down as the graph gets denser?
- **Test K — Framebuffer size impact.** Cost from 256² to 2048² framebuffers. Maps to: what does a 4K display do to picking?
- **Test L — `PICKING_MODE` bailout at scale.** Cost with and without an early-return in the fragment shader during picking passes. Maps to: how much does a one-line shader change actually save?
- **Test M — Concurrent JS work during async readback.** What happens to total time vs main-thread time when async readback overlaps real JS work.
- **Test N — Realistic graph simulation.** Mixed-cost scene approximating a Sigma graph at typical sizes.

Each test reports N, trimmed mean (1% top/bottom), median, p95, p99, and standard deviation. Warmup iterations are discarded.

## Data format

Each file in [`./data/`](./data/) is one benchmark run on one platform. Filenames follow the convention:

```
NN-browser-os-gpu-runR.md
```

Where `NN` is a stable ordering number, `R` is the run index for that platform (multiple runs per platform capture variance). Examples:

```
01-chrome-linux-rtx4070super-run1.md
02-firefox-linux-rtx4070super-run1.md
03-chrome-windows11-rtx4060-run1.md
```

Each file includes the environment fingerprint (`navigator.userAgent`, `gl.getParameter(VENDOR/RENDERER)`, hardware concurrency, timestamp) followed by per-test result tables. The file is exactly what the suite's **Copy as markdown** button produces — paste-and-commit, no editing needed.

## Privacy

The HTML loads zero external resources. No CDN scripts, no fonts, no analytics, no telemetry, no `fetch()` calls. Open it offline, run it offline, close the tab — everything is discarded. The "Email results" button is the only mechanism that ever transmits anything, and it uses your own mail client with a `mailto:` link; you see and edit the message before sending.

If you're skeptical, that's the right posture. Open DevTools → Network tab, run the suite, observe zero requests. Or save the file and disconnect from the internet first.

## Contributing platform results

Genuinely interested in additional platform data, especially:

- Apple Silicon (M1/M2/M3, lower-tier M4 variants, older macOS versions on Apple Silicon)
- iOS Safari and Android Chrome on actual phones
- AMD GPUs (currently no AMD data in the set)
- High-DPI displays at native devicePixelRatio
- WebGPU port (see [open question 4 in the memo](./MEMO.md#open-questions))

Two ways to submit: PR with your run file added to `./data/`, or email `hello@bitmosh.dev`. Details in [CONTRIBUTING.md](./CONTRIBUTING.md).

## Citation

If you reference this work academically or in a blog post, here's a BibTeX-style entry:

```
@misc{bitmosh2026picking,
  author       = {Ryan, bitmosh.dev},
  title        = {Empirical findings on the WebGL hit-detection pipeline},
  year         = {2026},
  url          = {https://bitmosh.dev/research/picking-pipeline},
  note         = {Benchmark suite: https://github.com/bitmosh/picking-benchmark}
}
```

For informal references, "bitmosh.dev (2026), *Empirical findings on the WebGL hit-detection pipeline*" is fine.

## License

MIT. See [LICENSE](./LICENSE). Use the code freely. Cite the findings if you build on them — that's a professional courtesy, not a legal requirement.

## Acknowledgements

- **Alexis Jacomy** and the Sigma.js contributors, for building a library careful enough to be profiled at this level. The texture-backed FBO pattern, the `refresh` / `scheduleRender` separation, and the abstract program base classes all show up in the methodology section.
- **Gregg Tavares**, for the original WebKit Metal slow-path investigation ([WebKit #235002](https://bugs.webkit.org/show_bug.cgi?id=235002)) which framed how I thought about texture-backed vs canvas-backed framebuffers.
- The maintainers of the Three.js async readback discussions ([three.js #23550](https://github.com/mrdoob/three.js/issues/23550)) for prior art on the fence-sync pattern.

---

*Built as part of [LumaWeave](https://bitmosh.dev) development. Findings reported in good faith. If you find an error or want to challenge a finding with contradicting data, open an issue.*
