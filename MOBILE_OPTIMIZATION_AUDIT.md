# ComfyUI Mobile Optimization Audit

This audit pinpoints concrete areas in the current codebase that can be optimized for running ComfyUI in mobile-like environments (tight VRAM/RAM budgets, weaker CPUs/NPUs, and bandwidth-constrained clients).

## 1) Memory policy is global and static; mobile needs adaptive memory modes

### Where this shows up
- VRAM modes exist (`--lowvram`, `--novram`, etc.), but they are chosen at startup and remain static. See CLI flags in `comfy/cli_args.py` and load-time behavior in `comfy/model_management.py`.
- `minimum_inference_memory()` and `EXTRA_RESERVED_VRAM` are fixed heuristics, not dynamically tuned to workload or thermal/memory pressure.

### Why it matters on mobile
- Mobile devices frequently fluctuate available memory due to OS pressure, foreground/background transitions, and shared GPU memory behavior.
- Static thresholds can trigger unnecessary unload/reload churn or OOMs.

### Optimization direction
- Add a **dynamic memory governor** that continuously samples free memory and queue characteristics, then toggles between NORMAL/LOW/NO-VRAM-like behavior per prompt.
- Introduce adaptive `EXTRA_RESERVED_VRAM` and `minimum_inference_memory` scaling based on observed memory pressure and recent OOM near-misses.
- Persist model residency hints (hot models vs cold models) to reduce reload thrash.

## 2) Model loading/offloading is robust but not mobile-latency aware

### Where this shows up
- `load_models_gpu()` and `free_memory()` focus on fitting memory by unloading models, then reloading as needed.
- Offload/load decisioning is based largely on immediate memory constraints rather than prompt sequence prediction.

### Why it matters on mobile
- On mobile storage and memory bandwidth, repeated reloads are expensive in energy and latency.

### Optimization direction
- Add **predictive prefetch + delayed eviction**: if queue has multiple prompts sharing model families, keep those models warm longer.
- Add an eviction cost model (size × expected reuse × reload latency) rather than only free-bytes sorting.
- Provide a mobile preset that defaults to `cache-lru` tuned to lower memory ceilings.

## 3) Preview image pipeline can consume too much CPU/bandwidth

### Where this shows up
- `send_image()` always encodes with high JPEG quality (`quality=95`) or PNG (`compress_level=1`) for preview traffic.
- `preview-size` defaults to 512; high for phones on weak links.

### Why it matters on mobile
- Preview encoding/transmission competes with inference on thermally constrained systems.
- High-quality previews are often unnecessary for progressive updates.

### Optimization direction
- Add a **mobile preview profile**:
  - Lower default preview size (e.g., 256 or dynamic per client DPR/network).
  - Lower preview quality for intermediate frames; keep high quality only final outputs.
  - Support frame skipping / update coalescing when the websocket backlog grows.
- Add lightweight telemetry for preview encode time + socket send queue depth.

## 4) Execution path is effectively single worker; mobile may benefit from staged concurrency

### Where this shows up
- `prompt_worker()` runs in a single loop, executes prompts serially, and does periodic GC.

### Why it matters on mobile
- Full parallel inference is usually not possible, but mobile still benefits from **pipeline parallelism** (I/O, decode, postprocess) overlapping with GPU steps.

### Optimization direction
- Keep one inference worker but split non-GPU tasks (image encode/write, metadata, uploads) into bounded async worker pools.
- Backpressure preview/postprocess jobs if inference queue grows.
- Add a dedicated low-priority cleanup thread for predictable GC cadence.

## 5) Caching controls exist but there is no explicit mobile policy layer

### Where this shows up
- CLI exposes `--cache-classic`, `--cache-lru`, `--cache-none`, but no first-class “mobile mode” bundles sensible defaults.

### Why it matters on mobile
- Users frequently do not know the right combination of flags; a bad default can tank UX.

### Optimization direction
- Add `--mobile` preset that configures:
  - low preview bandwidth,
  - LRU cache with conservative size,
  - aggressive offload,
  - async offload and non-blocking copies when supported,
  - optional reduced metadata writes.
- Expose “Battery saver / Balanced / Performance” variants on top of `--mobile`.

## 6) Missing client capability negotiation for mobile-specific behavior

### Where this shows up
- There is partial feature detection (`supports_preview_metadata`) but no broader capability negotiation for network class, target FPS, decode preferences, or memory class.

### Why it matters on mobile
- Server cannot tailor preview cadence, image encoding format, or graph execution strategy to client constraints.

### Optimization direction
- Extend websocket handshake with a **client capability schema** (memory class, network estimate, preferred preview format, target update rate).
- Dynamically tune preview/image transport and intermediate node materialization based on capability profile.

## 7) Packaging/deployment opportunity: mobile-lean runtime profile

### Where this shows up
- Many optional nodes and APIs are loaded by default unless manually disabled.

### Why it matters on mobile
- Startup time, memory footprint, and binary/package size matter much more on device.

### Optimization direction
- Provide a **mobile distribution profile** that defaults to:
  - `--disable-all-custom-nodes` (with allowlist),
  - `--disable-api-nodes` when not needed,
  - curated minimal node set and lazy module imports for heavy optional paths.

---

## Suggested implementation order (highest impact first)
1. Mobile preset (`--mobile`) + preview profile + sane default cache mode.
2. Dynamic memory governor (adaptive reserve thresholds + offload policy).
3. Preview throttling/coalescing and quality tiers.
4. Predictive model residency + cost-based eviction.
5. Capability negotiation in websocket handshake.
6. Mobile-lean distribution profile and lazy loading.

## What you probably *actually* want next
If your goal is to make ComfyUI feel usable on phones/tablets rather than just “run,” the key is:
- **fast first image**,
- **stable memory behavior** (no sudden OOM),
- **responsive progressive previews**,
- **small thermal footprint**.

The first practical milestone is a "mobile balanced" mode that sacrifices a bit of peak quality/speed for predictability and battery.
