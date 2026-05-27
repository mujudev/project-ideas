---
summary: Tech stack decision for the LTVT port — evaluating TypeScript, Go, and Rust options across ephemeris, rendering, deployment, and CLI requirements.
last_updated: 2026-05-26
tags: [ltvt, decision, tech-stack, architecture, rust, typescript, tauri, webgl, wasm]
---

# ADR-001: Tech Stack for LTVT Port

**Status:** Proposed  
**Decision required:** Before implementation begins

---

## Context

We are porting the Lunar Terminator Visualization Tool (LTVT) from a Windows-only Delphi 6 application into a modern, portable stack. The following requirements drive the decision:

1. **Portability** — must run on macOS, Linux, and Windows; ideally also as a web app in a browser
2. **Modern UI/UX** — the original fixed 1024×768 window and Windows-only UI are non-starters
3. **Stack constraint** — TypeScript, Go, or Rust (no Python, no Java)
4. **Dual interface** — every feature should be available as both a UI and a CLI command
5. **Deployment flexibility** — desktop app and/or hosted web app

The key technical challenges from the original implementation that constrain stack choices:

| Challenge | Why it matters |
|---|---|
| JPL DE405 Chebyshev interpolation | Precision astronomical computation — needs sub-arcsecond accuracy |
| Per-pixel orthographic rendering | ~640×640 pixels, each requiring a full 3D coordinate transform |
| DEM ray-marching | O(width × height × ray-depth) — extremely CPU/GPU intensive |
| Photometric models | Per-pixel Lommel-Seeliger / Lunar-Lambert math |
| Binary file I/O | DE405 `.405` files, PDS `.IMG` DEM files |
| Feature database queries | 9,300-entry IAU feature CSV |

---

## Options Evaluated

### Option A — Pure TypeScript (Browser-first)

All logic in TypeScript, deployed as a static web app, with a Node.js/Bun CLI.

**Ephemeris:** [`astronomy-engine`](https://github.com/cosinekitty/astronomy) (npm, MIT, no dependencies)
- Based on VSOP87 + NOVAS C 3.1
- ±1 arcminute accuracy (vs. DE405's sub-arcsecond)
- Has `Libration()` → sub-Earth selenographic lon/lat directly
- Has `GeoVector(Body.Moon)`, `IlluminationInfo` for sub-solar point
- Pure TypeScript, runs in browser and Node.js, 116 KB minified

**Rendering:** WebGL2 via Three.js r184 + custom fragment shaders
- Orthographic projection, texture sampling, photometric models → all natural GLSL
- DEM ray-marching → compute shader or fragment shader with DEM as texture
- Three.js abstracts WebGL boilerplate; `react-three-fiber` provides React integration
- Full GPU acceleration: the core bottleneck is completely solved

**UI:** React + Tailwind + shadcn/ui (or Solid.js + Kobalte for lower overhead)

**Desktop:** Tauri v2 — wraps the web frontend in a native window using the OS webview (WebKit/WebView2/WebKitGTK). Rust backend handles file system (reading `.405` files from disk), native OS integration.

**CLI:** Node.js or Bun script consuming the same core library, outputting to stdout/file.

**Deployment:**
- Web: static site on Vercel / Netlify / Cloudflare Pages — zero server cost
- Desktop: Tauri distributable (~3–10 MB binary) for macOS, Windows, Linux
- Both consume the same compiled JS/WASM bundle

**Pros:**
- Simplest path: one language dominates (TypeScript everywhere)
- GPU rendering is the default — no CPU fallback needed
- `astronomy-engine` covers all the event prediction / eclipse / phase features
- Rich ecosystem: Three.js, shadcn, Storybook, Vitest
- Tauri adds desktop without adding a second language to the *product* logic
- Easiest to iterate UI/UX

**Cons:**
- `astronomy-engine` accuracy (±1 arcminute) is lower than the original DE405 (sub-arcsecond). For casual use this is fine; for precision lunar observers it may not be
- No easy path to reading raw DE405 binary files from the browser (requires either a Tauri IPC call to Rust, or pre-processing data into a web-friendly format)
- DEM file reading (PDS binary) similarly requires either Tauri IPC or pre-conversion
- CLI story is workable but Node/Bun is heavier than a native Go/Rust binary

---

### Option B — Rust Core + TypeScript UI (Recommended)

Rust provides the precision computation layer (compiled to both a native binary and to WASM). TypeScript handles the UI. Tauri is the natural glue.

**Structure:**

```
ltvt/
  crates/
    ltvt-core/      ← pure Rust library: ephemeris, coordinates, DEM, photometry
    ltvt-cli/       ← clap CLI binary linking ltvt-core
    ltvt-wasm/      ← wasm-bindgen wrapper exporting ltvt-core to JS
  packages/
    ltvt-web/       ← React/Three.js frontend, consumes ltvt-wasm
    ltvt-app/       ← Tauri v2 app, uses ltvt-web + ltvt-core via Tauri commands
```

**Ephemeris (Rust):**
- Port the DE405 Chebyshev reader from the Delphi source — it's ~200 lines of pure math, straightforward in Rust
- The Rust `ltvt-core` crate owns: `ReadEphemeris`, `SubEarthPointOnMoon`, `SubSolarPointOnMoon`, all coordinate transforms
- For time handling: [`hifitime`](https://crates.io/crates/hifitime) crate — nanosecond-precision time, TDB/TT/UTC conversions, IERS tables — or manual port of the `TimLib.pas` / `H_ClockError.pas` logic
- For linear algebra: [`nalgebra`](https://crates.io/crates/nalgebra) (same vector/matrix operations as `MPVectors.pas`)
- Accuracy: matches the original DE405 exactly (same algorithm, same data)

**Rendering:**
- Fragment shader in GLSL/WGSL handles orthographic projection + texture + photometric models — runs on GPU via WebGL2
- DEM ray-marching: encode DEM as a floating-point WebGL texture; ray-march in a fragment shader — this is much faster than the original CPU loop
- Rust WASM is not used for rendering (GPU is faster); WASM is used only for pre-computation (geometry, sub-points)
- Three.js (or Babylon.js, or raw WebGL2) for the scene graph

**CLI (Rust, `ltvt-cli`):**
- `clap` for argument parsing
- Same `ltvt-core` functions as the WASM build
- Output: JSON (machine-readable) or formatted text (human-readable)
- Examples:
  ```sh
  ltvt geometry --date 2026-05-26T22:00:00Z --lat 34.05 --lon -118.25
  ltvt render --date 2026-05-26T22:00:00Z --output moon.png --width 1024
  ltvt features --min-diameter 50 --colongitude 45
  ltvt predict --feature "Tycho" --days 30
  ```

**UI (TypeScript):**
- React + Tailwind CSS + shadcn/ui for accessible, composable components
- `react-three-fiber` (R3F) for Three.js integration within the React tree
- `@react-three/drei` for common Three.js helpers (orbit controls, textures, etc.)
- Zustand or Jotai for lightweight state management

**Desktop (Tauri v2):**
- Tauri Rust backend = `ltvt-core` functions exposed as Tauri commands (IPC)
- Handles: reading `.405` files from disk, DEM file I/O, saving images, observatory list
- Frontend = same React/Three.js app, running in Tauri webview
- Single `npm run tauri build` produces macOS `.app`, Windows `.msi`/`.exe`, Linux `.AppImage`
- Bundle size: ~10–20 MB (webview provided by OS, not bundled)

**Web App:**
- `ltvt-wasm` compiled via `wasm-pack build --target web`
- The TypeScript frontend dynamically imports the WASM module
- JPL `.405` files fetched from a CDN or served from an API (see below)
- DEM files stored in IndexedDB after first download
- Deploy as static site on Cloudflare Pages / Vercel — no server required

**Ephemeris file strategy for web:**
- The 50-year DE405 block (`unxp2000.405`, ~4.5 MB) can be hosted as a static asset
- Fetch once, cache in IndexedDB / Cache API
- For other date ranges, prompt user to provide a file (File System Access API), matching the original behavior
- Alternative: pre-compute sub-point tables for a 200-year range into a compact binary format (~1–2 MB)

**Pros:**
- Full DE405 precision — matches the original exactly
- DEM rendering is dramatically faster via GPU fragment shaders vs. original CPU loop
- One Rust codebase powers CLI + desktop (Tauri) + web (WASM) — no logic duplication
- CLI is a native binary: fast startup, no runtime dependency
- Tauri's security model is a strong default (capability-based permissions)
- The WASM module can be used by other projects or a public API

**Cons:**
- Two languages (Rust + TypeScript) — requires knowledge of both
- WASM build pipeline (`wasm-pack`, `wasm-bindgen`) adds build complexity
- Initial Rust port of the ephemeris code requires careful validation against the original
- Longer initial setup vs. pure TypeScript

---

### Option C — Go Server + TypeScript Frontend

Go handles computation and serves a REST/gRPC API. TypeScript frontend consumes the API.

**Ephemeris (Go):**
- Port the DE405 reader to Go — feasible but no existing Go library covers DE405 at this level
- [`gosky`](https://github.com/soniakeys/meeus) (Meeus algorithms) or `go-astro` — lower precision than DE405
- No mature Go equivalent of `astronomy-engine`

**Rendering:** client-side WebGL (same Three.js story as above) or server-side with `go-draw` / `gg` — server-side rendering is impractical for interactive use

**CLI:** Go binary with `cobra` or `urfave/cli` — excellent CLI story

**Deployment:** Docker container on Fly.io / Railway / Render; static frontend served separately

**Pros:**
- Excellent CLI (Go produces tight, fast, zero-dependency binaries)
- Simple concurrent server model
- Single language for server + CLI

**Cons:**
- Ephemeris ecosystem is weak in Go — would need to port DE405 without prior art
- Desktop app story is poor: no mature equivalent to Tauri for Go (Wails exists but less mature)
- Server adds infrastructure cost and latency vs. pure client-side WASM
- Rendering must still be in the browser anyway; Go buys nothing for the core visual feature
- Splitting computation server-side breaks offline use

---

## Comparison Matrix

| Criterion | Option A (TS-only) | Option B (Rust+TS) ✓ | Option C (Go+TS) |
|---|---|---|---|
| Ephemeris precision | ±1 arcmin (VSOP87) | Sub-arcsecond (DE405) | ±1 arcmin or manual port |
| Rendering performance | GPU (WebGL) | GPU (WebGL) | Client GPU; server is slow |
| Browser deployment | ✓ Static | ✓ Static + WASM | Server required |
| Desktop app | ✓ Tauri | ✓ Tauri | Wails (less mature) |
| CLI | Node/Bun (ok) | Native binary (best) | Native binary (best) |
| Offline capable | ✓ | ✓ | ✗ (needs server) |
| Single codebase for CLI+web+desktop | Partial | ✓ (Rust core) | Partial |
| Dev complexity | Low | Medium | Medium |
| Ecosystem maturity | High | High | Medium |
| Mobile (future) | ✓ Tauri v2 | ✓ Tauri v2 | Difficult |

---

## Decision: Option B — Rust Core + TypeScript UI

### Rationale

The most important factor is that **the rendering and the computation can be cleanly separated**:
- Rendering → GPU (WebGL fragment shaders in the browser). TypeScript + Three.js owns this entirely. Rust is not needed here.
- Computation (ephemeris, coordinate transforms, sub-point calculation) → Rust. This is pure math that maps naturally to Rust, compiles to WASM without any changes, and can be a native binary for the CLI.

This separation means the "two language" cost is minimal in practice — the TypeScript layer never touches astronomy math, and the Rust layer never touches DOM/UI. The interface between them is a small, clean set of functions.

The DE405 precision argument is real but not a deal-breaker for casual use. However, since we're specifically porting a precision tool used by serious lunar observers, maintaining the original accuracy is the right default. The Rust port of the Chebyshev reader is ~200 lines and can be validated against the original output.

The CLI requirement is best served by Rust (`clap` + native binary) vs. Node/Bun (heavier, requires runtime). Go would also work for CLI, but Go buys nothing for the rendering path and splits the computation code across two languages.

---

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────┐
│                    User Interface                        │
│         React + Three.js + Tailwind (TypeScript)        │
│   Browser / Tauri WebView                               │
└────────────────────┬────────────────────────────────────┘
                     │ WebGL draw calls
                     ▼
┌─────────────────────────────────────────────────────────┐
│                GPU (WebGL2 / WebGPU)                     │
│  Fragment shaders: orthographic projection, texture      │
│  sampling, Lommel-Seeliger, Lunar-Lambert, DEM           │
│  ray-marching, cast shadows                             │
└─────────────────────────────────────────────────────────┘
                     │ sub-points, geometry params
                     ▲
┌─────────────────────────────────────────────────────────┐
│                   ltvt-core (Rust)                       │
│  ┌─────────────────┐  ┌────────────────────────────┐   │
│  │  ltvt-wasm      │  │  ltvt-cli                  │   │
│  │  (wasm-bindgen) │  │  (clap binary)             │   │
│  │  Browser / web  │  │  macOS/Linux/Windows       │   │
│  └────────┬────────┘  └────────────┬───────────────┘   │
│           │                        │                    │
│  ┌────────▼────────────────────────▼───────────────┐   │
│  │             ltvt-core library                    │   │
│  │  • DE405 Chebyshev reader                        │   │
│  │  • SubEarthPointOnMoon / SubSolarPointOnMoon     │   │
│  │  • Selenographic coordinate system              │   │
│  │  • Time conversions (UTC → TDB)                 │   │
│  │  • Feature database queries                     │   │
│  │  • Colongitude, libration tabulation            │   │
│  └─────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
                     │ Tauri IPC (file system, native OS)
                     ▼
┌─────────────────────────────────────────────────────────┐
│               Tauri v2 Backend (Rust)                   │
│  • Read .405 / .IMG files from disk                     │
│  • Save images                                          │
│  • Observatory list management                          │
│  • Auto-update                                          │
└─────────────────────────────────────────────────────────┘
```

---

## Proposed Library Choices

### Rust (`ltvt-core`)

| Purpose | Crate | Notes |
|---|---|---|
| CLI argument parsing | `clap` v4 | Derive macros; `--help` auto-generated |
| WASM bindings | `wasm-bindgen` + `wasm-pack` | Compiles to ES module or bundler target |
| Time handling | `hifitime` | TDB, TT, UTC, TAI; IERS tables; or manual port |
| Vector/matrix math | `nalgebra` | Mirrors `MPVectors.pas` precisely |
| Binary file I/O | `std::io` | DE405 reader; PDS DEM reader |
| Serialization | `serde` + `serde_json` | CLI JSON output; WASM data exchange |
| Error handling | `thiserror` + `anyhow` | |
| Testing | `cargo test` + `proptest` | Property-based tests for coordinate math |

### TypeScript (Frontend)

| Purpose | Package | Notes |
|---|---|---|
| Framework | React 19 | Concurrent mode; Server Components for SSG |
| Build | Vite + `@vitejs/plugin-react` | Fast HMR; WASM plugin |
| 3D/WebGL | Three.js r184 + `@react-three/fiber` | GPU rendering; custom `ShaderMaterial` |
| 3D helpers | `@react-three/drei` | Orbit controls, texture loaders, etc. |
| UI components | shadcn/ui + Tailwind CSS v4 | Accessible; Radix UI primitives |
| State | Zustand | Lightweight; no boilerplate |
| High-precision astronomy | `astronomy-engine` v2 | For event prediction, eclipses, phases |
| Date/time | `luxon` or native `Temporal` | UTC / ISO 8601 |
| Testing | Vitest + `@testing-library/react` | |
| Desktop | Tauri v2 (`@tauri-apps/api`) | IPC, file system, windows |

### Infrastructure

| Purpose | Choice | Notes |
|---|---|---|
| Monorepo | Cargo workspaces + npm workspaces | Rust crates + TS packages co-located |
| Build orchestration | `turbo` or `mise` task runner | Parallel builds |
| CI | GitHub Actions | `cargo test`, `wasm-pack test`, Vitest |
| Web deployment | Cloudflare Pages | Free tier; WASM support; edge CDN |
| Desktop distribution | Tauri `updater` plugin | Signed auto-update for macOS/Windows/Linux |

---

## Implementation Phases

### Phase 1 — Rust Core + CLI (no UI)
- Port DE405 Chebyshev reader from Delphi → Rust
- Implement `SubEarthPointOnMoon` and `SubSolarPointOnMoon`
- Validate output against original LTVT for 10+ test dates
- Build `ltvt-cli geometry` command
- Port IAU feature CSV reader + colongitude filter
- Build `ltvt-cli features` command

### Phase 2 — WASM + Basic Web Renderer
- Add `wasm-bindgen` wrapper for core functions
- Minimal React app: date/time picker, observer location, sub-point display
- Three.js scene: orthographic Moon disk, Lambertian shading, texture overlay
- Dot overlay from feature database

### Phase 3 — Full GPU Rendering
- GLSL fragment shader: Lommel-Seeliger + Lunar-Lambert photometric models
- WebGL float texture for DEM data
- DEM ray-march shader
- Cast shadow support in shader

### Phase 4 — Tauri Desktop App
- Wrap web app in Tauri v2
- Tauri commands: read local `.405` files, DEM files, save images
- macOS / Windows / Linux builds
- File association for `.405`

### Phase 5 — Feature Parity
- LTO chart designation display
- Rukl atlas numbers
- Moon event predictor (via `astronomy-engine` + Rust colongitude search)
- Photo session search
- Libration tabulator
- Shadow measurement tool
- Circle drawing tool
- Image export with annotation

---

## Key Risks and Mitigations

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| DE405 Rust port has bugs | Medium | High | Validate against original LTVT for 50+ known dates; add proptest round-trips |
| DEM ray-march shader is too slow on integrated GPUs | Medium | Medium | Add LOD: coarser step size on mobile/low-end GPUs; fall back to software rendering |
| WASM bundle too large for web | Low | Low | DE405 data is loaded lazily from CDN; core WASM should be <500 KB |
| `wasm-bindgen` bindings become a maintenance burden | Low | Medium | Keep the WASM API surface small — only expose computed results, not raw types |
| `astronomy-engine` and Rust core disagree on sub-points | Low | Low | Both are validated against JPL Horizons; use Rust for rendering geometry, `astronomy-engine` for event prediction only |
| Tauri WebView rendering differs cross-platform | Medium | Medium | Use CSS/WebGL (not DOM-specific features) for all Moon rendering; test on all three OSes |

---

## What We Are Not Doing

- **Not porting the NOVAS C library to TypeScript/Rust** — `astronomy-engine` covers the event prediction use cases at sufficient accuracy; the DE405 reader covers the precision geometry use case
- **Not building a Go backend** — adds infrastructure cost and network latency; loses offline capability; the CLI use case is better served by a native Rust binary
- **Not using Electron** — Tauri achieves the same result with a ~10× smaller bundle and better security model
- **Not doing server-side rendering of the Moon** — every pixel computation is O(width × height); GPU is the only viable path at interactive frame rates
- **Not porting the Delphi DFM forms** — clean-slate UI design in React
