# Contributing

Thanks for your interest. This is a research-artifact repo — the primary contribution that helps is **running the benchmark on hardware/software combos that aren't yet in the data set**.

## What's most useful

Roughly in order of how much it would refine the picture:

1. **Apple Silicon (M1, M2, M3, M4) on Safari** — the existing Safari data is from Intel HD 4000 / Catalina. The Metal pipeline on Apple Silicon may behave substantially differently.
2. **iOS Safari and Android Chrome on actual phones** — current "mobile-class" data is approximated from a Rockchip RK3588 (Mali-G610), which is GPU-architecturally close but lives in a different browser environment.
3. **AMD GPUs** — there is currently no AMD data in the set. RDNA2/RDNA3 desktop, AMD integrated, anything.
4. **High-DPI displays at native `devicePixelRatio`** — all current measurements are at DPR=1. Retina / 4K behavior is interesting because framebuffer-size scaling differs by backend.
5. **Browsers not yet represented** — Brave, Arc, mobile browsers, etc.
6. **Re-runs of platforms already in the set** — variance characterization helps separate signal from noise.

## How to run the benchmark

1. Open [bitmosh.dev/labs/picking-benchmark](https://bitmosh.dev/labs/picking-benchmark), or open `readpixels-benchmark.html` locally.
2. Close other tabs and heavy apps if you can — keeps the GPU queue clean and the measurements honest.
3. Click **Run full v3 suite**. Quick run is fine for a smoke test but the full suite is what produces useful comparison data.
4. When it finishes, click **Copy as markdown** (or **Email results to bitmosh**).

## How to submit

Two paths, both fine.

### Path A — Pull request

1. Fork the repo.
2. Add your run as a markdown file in `./data/` following the naming convention:
   ```
   NN-browser-os-gpu-runR.md
   ```
   Use the next available number for `NN` and `1` for `R` (or `2`, `3` if you're contributing multiple runs on the same platform).
   Examples of valid filenames:
   ```
   16-safari-m4-macos15-run1.md
   17-chrome-android-mali-g78-run1.md
   18-firefox-windows-rx7900xtx-run1.md
   ```
3. Open a PR. Brief description: platform, anything unusual about the run (thermal throttling, background load, browser flags, etc.).

The file you commit should be exactly what the suite's **Copy as markdown** button produced. No editing needed — it already includes the environment fingerprint.

### Path B — Email

Click **Email results to bitmosh** in the suite. It opens your mail client pre-filled with the results and a sensible subject line. Send to `hello@bitmosh.dev`. I'll add the file to the repo and credit you in the commit message (let me know how you want to be credited — full name, handle, or anonymously).

## What if I disagree with a finding?

Open an issue or PR with the supporting data. The memo is explicit about the limits of the methodology and where I was wrong on first pass; contradicting evidence on any specific point is welcome.

## What I'm *not* looking for

- Changes to the benchmark code that aren't tied to a measurement issue. The suite is intentionally frozen for v3 to keep historical runs comparable. If the methodology needs to evolve, that'll be a v4 with its own run set.
- Stylistic cleanups, build-tool additions, framework conversions. The single-file HTML is a feature; please don't React-ify it.
- Cosmetic edits to the memo without new data behind them.

## Code of conduct

Be civil, technical, and concrete. Disagreements about measurements are great; arguments about anything else aren't the point of this repo.
