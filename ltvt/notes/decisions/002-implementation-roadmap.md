---
summary: Implementation roadmap for the LTVT port — phased milestones from repo setup through full feature parity and public release.
last_updated: 2026-05-26
tags: [ltvt, decision, roadmap, milestones, implementation, planning]
---

# ADR-002: Implementation Roadmap

**Status:** Proposed  
**Depends on:** [ADR-001 — Tech Stack](./001-tech-stack.md)

---

## Guiding Principles

Before listing milestones, three principles govern sequencing decisions throughout this roadmap:

1. **Vertical slices over horizontal layers.** Each milestone ships something that actually runs end-to-end, even if narrow in scope. We do not build the entire Rust core before touching the UI.
2. **Validate early against the original.** The Delphi source is the ground truth. Every piece of computation must be numerically validated against the original before being built on top of.
3. **CLI and UI share the same core.** The CLI is not a side project; it is a first-class output of every milestone from M1 onward. This enforces that the Rust core has a clean, headless API.

---

## Milestone Map

```
M0  Foundation & tooling
 └─ M1  Rust core + CLI (geometry)
     └─ M2  WASM bridge + web skeleton + dot mode
         └─ M3  GPU rendering (texture + DEM)
             └─ M4  Feature overlay + interactivity
                 └─ M5  Tauri desktop app
                     └─ M6  Event prediction + advanced tools
                         └─ M7  Polish, accessibility, public release
```

Each milestone is a shippable, testable checkpoint. Later milestones depend on the previous but can be developed in parallel for independent sub-tasks.

---

## M0 — Foundation and Tooling

**Goal:** An empty but fully configured monorepo that CI can build and test end-to-end.

### Deliverables

- [ ] Monorepo structure scaffolded:
  ```
  ltvt/
    Cargo.toml              ← workspace root
    crates/
      ltvt-core/            ← pure Rust library
      ltvt-cli/             ← clap binary
      ltvt-wasm/            ← wasm-bindgen wrapper
    packages/
      ltvt-web/             ← React + Three.js frontend (Vite)
      ltvt-app/             ← Tauri v2 application
    .github/
      workflows/
        ci.yml
        release.yml
  ```
- [ ] Rust: `cargo test` passes (empty test suites) for all three crates
- [ ] TypeScript: `pnpm build` and `pnpm test` pass for `ltvt-web`
- [ ] WASM: `wasm-pack build` produces a valid ES module from `ltvt-wasm`
- [ ] Tauri: `npm run tauri dev` opens a blank window
- [ ] CI: GitHub Actions runs `cargo test`, `wasm-pack test --headless`, and `vitest` on every push to `main` and all PRs
- [ ] Linting: `clippy` (Rust), `eslint` + `typescript-strict` (TS)
- [ ] A `Justfile` (or `Makefile`) with standard dev tasks: `just build`, `just test`, `just dev`

### Definition of Done

CI is green on the empty repo. A developer can clone and run `just dev` to see a blank Tauri window and a blank Vite dev server.

---

## M1 — Rust Core: Ephemeris and Geometry

**Goal:** The core astronomical computation layer exists as a Rust library and works from a CLI. No UI yet.

### Deliverables

**`ltvt-core` — DE405 Reader**
- [ ] Binary file reader for Unix-format `.405` files (Chebyshev coefficient blocks)
- [ ] `ReadEphemeris(tdb, target, center)` function matching the Delphi `H_JPL_Ephemeris.pas` interface
- [ ] Supports: positions of Moon, Sun, Earth, all planets; lunar librations
- [ ] File loading: lazy (load block on demand from disk), with block caching

**`ltvt-core` — Time System**
- [ ] UTC → Modified Julian Date (MJD) conversion
- [ ] MJD → TDB (Terrestrial Barycentric Time / Ephemeris Time) using TAI offset tables
- [ ] ΔUT1 and leap-second table (can be a compiled-in table initially; IERS fetch later)
- [ ] Light-travel-time correction (one-iteration, matching original behavior)

**`ltvt-core` — Coordinate System**
- [ ] `SetSelenographicSystem(tdb)` — builds Moon body-fixed frame from JPL libration Euler angles
- [ ] `SelenographicCoordinates(icrf_vector)` → lon/lat/radius
- [ ] `SubEarthPointOnMoon(utc_mjd, observer_lon, observer_lat, observer_elev)` → selenographic lon/lat
- [ ] `SubSolarPointOnMoon(utc_mjd)` → selenographic lon/lat
- [ ] `PolarToVector` / `VectorToPolar` utilities
- [ ] Orthographic basis vector calculation (XPrime, YPrime, ZPrime) for each orientation mode

**`ltvt-core` — Feature Database**
- [ ] CSV parser for `CratersAndSub-CratersFeaturesIMPs2021-JohnMoore.csv`
- [ ] `Feature` struct (name, lon/lat, diameter, USGS code, flags)
- [ ] Filter functions: by minimum diameter, by visibility given a sub-observer point, by USGS code
- [ ] `ColongitudeAt(utc_mjd)` utility

**`ltvt-cli` — Commands**
- [ ] `ltvt geometry --date <ISO8601> [--lat <deg>] [--lon <deg>] [--elev <m>]`  
  Outputs: sub-observer lon/lat, sub-solar lon/lat, colongitude, % illuminated, Moon elevation, Moon diameter
- [ ] `ltvt features [--min-km <n>] [--date <ISO8601>] [--output json|table]`  
  Lists visible features given a geometry
- [ ] `ltvt colongitude --feature <name> --date <ISO8601>`  
  Reports solar altitude / colongitude at a named feature
- [ ] Output formats: `--output json` (machine-readable), `--output table` (human-readable, default)
- [ ] `--ephemeris <path>` flag for specifying the `.405` file location

### Validation Criteria

- [ ] For at least 20 test dates spanning 1950–2050, `ltvt geometry` output matches the original LTVT's displayed colongitude, sub-observer point, and sub-solar point to within 0.01° — verified by running the original Delphi exe under Wine or using its published examples
- [ ] Round-trip test: `PolarToVector(VectorToPolar(v)) ≈ v` for 1000 random vectors
- [ ] Known colongitude values cross-checked against USNO Astronomical Almanac

### Definition of Done

`ltvt geometry --date 2026-05-26T22:00:00Z` produces numerically correct output from the command line. CI validates this with a fixture-based test.

---

## M2 — WASM Bridge and Web Skeleton

**Goal:** The Rust core is callable from a browser. A minimal React app shows sub-point data and renders the Moon as a shaded dot (no texture, no GPU shader yet) with feature dots overlaid.

### Deliverables

**`ltvt-wasm` — Bindings**
- [ ] `wasm-bindgen` exports for: `computeGeometry(date, obsLat, obsLon, obsElev)` → JS object
- [ ] `getVisibleFeatures(subObsLon, subObsLat, minDiamKm)` → array of feature objects
- [ ] WASM bundle size target: < 500 KB (gzipped)
- [ ] `wasm-pack build --target bundler` produces an npm-importable package
- [ ] Feature data bundled into the WASM binary (compiled-in) so no CSV fetch is needed

**`ltvt-web` — React App Skeleton**
- [ ] Vite project with `@vitejs/plugin-wasm` and `vite-plugin-top-level-await`
- [ ] Main layout: sidebar (controls) + canvas area (Moon display)
- [ ] Date/time picker (UTC), observer lat/lon/elev fields
- [ ] "Compute Geometry" button → calls WASM → displays sub-observer point, sub-solar point, colongitude, % illuminated
- [ ] "Now" button sets current UTC

**`ltvt-web` — Basic Three.js Scene (Dot Mode)**
- [ ] `react-three-fiber` canvas in the main panel
- [ ] Orthographic camera matching the LTVT coordinate system (not a perspective camera)
- [ ] Moon rendered as a flat disk, sunlit and shadow hemispheres in solid colors (no texture)
- [ ] Feature dots drawn as `Points` or instanced meshes at correct selenographic positions
- [ ] Dot color by feature type (crater size tiers, non-craters)
- [ ] Mouse hover over a dot → tooltip showing feature name, diameter, type
- [ ] Click on the disk → re-centers view on clicked lon/lat
- [ ] Zoom control (scroll wheel or slider)

**State management**
- [ ] Zustand store: `geometry` (sub-observer, sub-solar), `viewSettings` (zoom, center, orientation mode), `featureFilter` (min diameter, enabled types)
- [ ] URL search params encode current geometry + view state (shareable links)

### Definition of Done

A user can open the web app, set a date/time, click "Compute Geometry", see the correct sub-points displayed, and see named lunar features plotted as dots on the correct disk positions.

---

## M3 — GPU Rendering: Texture and DEM

**Goal:** The Moon looks photorealistic. Texture mode and DEM mode both work in the browser via WebGL2 fragment shaders.

### Deliverables

**Texture Mode**
- [ ] A `THREE.ShaderMaterial` implementing orthographic texture mapping:
  - Fragment shader: for each pixel, compute selenographic lon/lat from screen position, sample the texture
  - Gamma correction applied in shader
  - "Normal" illumination: sun-angle shading using dot(surface_normal, sun_direction)
  - "High Sun" and "Low Sun" illumination modes
  - "Constant Sun Angle" mode
- [ ] Texture loading: `lores.jpg` bundled with the app; hi-res textures loaded from user-provided URL or file (File System Access API)
- [ ] Three texture slots (Texture 1 / 2 / 3); Texture 3 supports partial lon/lat extents
- [ ] Progress indicator during texture load
- [ ] Terminator lines (red morning / blue evening) drawn as `THREE.Line` objects

**DEM Mode**
- [ ] DEM data uploaded as a `THREE.DataTexture` (float32, single channel)
- [ ] Fragment shader: ray-march from observer through DEM texture to find surface intersection
  - Orthographic ray direction per pixel
  - Sample DEM height at each step
  - Stop when ray hits surface (height ≥ DEM value)
- [ ] Surface normal computed from 4-neighbor DEM samples in the shader
- [ ] Three photometric models in shader (controlled by uniform):
  - Lambertian: `max(0, dot(N, L))`
  - Lommel-Seeliger: `μ₀ / (μ + μ₀)`
  - Lunar-Lambert: McEwen 1996 blend
- [ ] Cast shadows: secondary ray-march in solar direction (optional, behind toggle)
- [ ] DEM × Texture3 multiply mode
- [ ] 2D and 3D (oblique) DEM views
- [ ] DEM file loading from user file (PDS `.IMG` format), parsed in Rust WASM, uploaded to GPU

**Controls wired up**
- [ ] Texture/DEM/Dot mode selector
- [ ] Illumination mode selector (Normal / High Sun / Low Sun / Constant)
- [ ] Gamma slider
- [ ] Grid overlay (lat/lon lines) drawn in shader or as a `THREE.LineSegments` overlay
- [ ] Libration circle toggle

### Validation Criteria

- [ ] Terminator position matches colongitude reported by the geometry panel to within 0.1°
- [ ] Lommel-Seeliger renders noticeably more accurate than Lambertian for near-terminator craters (visual inspection against Clementine images)
- [ ] DEM shaded relief matches the original LTVT screenshot for `lores.jpg` at the same geometry

### Definition of Done

All three drawing modes work in the browser. Switching modes, changing illumination, and adjusting gamma produce correct, smooth results.

---

## M4 — Feature Overlay and Interactivity

**Goal:** Full feature labeling and interactive map tools match the original LTVT's capabilities.

### Deliverables

**Feature Overlay**
- [ ] "Overlay Dots" applied on top of the current texture/DEM image (non-destructive)
- [ ] "Label Dots" places text labels adjacent to each dot
- [ ] Label configuration: feature name, size, units, font, color, offset, discontinue-filter
- [ ] Size-scaled circles around craters (togglable)
- [ ] Feature threshold filter (min diameter, or "flagged only")
- [ ] Dot rendering: instanced mesh or `Points` for performance with 9,300 features

**Right-Click / Context Menu**
- [ ] Right-click on the image opens a context menu at the correct selenographic position
- [ ] Identify nearest feature (shows name, coordinates, diameter, type)
- [ ] Go To: re-center on the clicked point
- [ ] Set reference point → enables distance/bearing readout on hover
- [ ] Draw circle: dialog to draw an angular-radius circle at the clicked point

**Mouse Readout**
- [ ] Real-time overlay (HUD) showing lon/lat under the cursor while moving
- [ ] Sun angle at cursor position
- [ ] Distance and bearing from reference point (if set)
- [ ] Feature name tooltip when hovering near a dot

**Navigation**
- [ ] Zoom slider + scroll wheel zoom
- [ ] Click-to-recenter (redraws with new center, same geometry)
- [ ] Keyboard shortcuts: arrow keys pan, `+`/`-` zoom, `R` reset view
- [ ] Go To dialog: enter lon/lat or Xi/Eta coordinates

**LTO / Rukl display**
- [ ] Mouse hover shows the LTO chart designation (e.g. `62B3`) and Rukl atlas number for the cursor position
- [ ] Displayed in the HUD readout strip

**Image Export**
- [ ] "Save Image" button: exports the current canvas to PNG
- [ ] Optional annotation layer: sub-observer/sub-solar lon/lat, date/time, observer location printed on the exported image
- [ ] Annotation color configurable

**Orientation modes**
- [ ] Line of Cusps, Cartographic, Equatorial, Alt-Az — all four wired up
- [ ] Manual rotation angle input
- [ ] Invert L-R / Invert U-D toggles
- [ ] Shareable URL encodes all view state

### Definition of Done

The full interactive map works: a user can compute geometry, view a textured Moon, overlay dots, label them, hover for readouts, right-click for context actions, and export an annotated image.

---

## M5 — Tauri Desktop App

**Goal:** The web app runs as a native desktop application on macOS, Windows, and Linux, with access to local files.

### Deliverables

**Tauri Integration**
- [ ] `ltvt-app` package wraps `ltvt-web` in a Tauri v2 window
- [ ] Window: resizable, minimum 1024×768, remembers last size and position
- [ ] App menu: File, Tools, Help menus mirroring the original structure

**Local File Access (Tauri Commands)**
- [ ] `load_jpl_file(path)` → reads `.405` file from disk, passes binary data to WASM
- [ ] `load_dem_file(path)` → reads PDS `.IMG` file, passes data to WASM
- [ ] `save_image(path, data)` → writes exported PNG to disk
- [ ] `load_texture(path)` → reads local JPEG/BMP texture file
- [ ] `load_feature_csv(path)` → reads alternate feature database
- [ ] Recent files list for each file type

**Settings Persistence**
- [ ] All user preferences stored in a platform-appropriate config file (`~/.config/ltvt/config.toml` or OS equivalent via Tauri's `store` plugin)
- [ ] Observer location saved and restored
- [ ] File paths (texture, DEM, JPL) saved and restored
- [ ] Label and rendering preferences persisted

**Observatory List**
- [ ] Load/save `Observatory_List.txt` from disk
- [ ] Add/remove/rename named observer locations from the UI
- [ ] Quick-select from a dropdown

**Desktop Distribution**
- [ ] GitHub Actions `release.yml` builds signed packages on tag push:
  - macOS: `.dmg` (universal binary, code-signed)
  - Windows: `.msi` + `.exe` (NSIS, optionally code-signed)
  - Linux: `.AppImage` + `.deb`
- [ ] Auto-updater via Tauri updater plugin
- [ ] GitHub Releases hosts binaries

### Definition of Done

A macOS user can `brew install --cask ltvt` (or download a `.dmg`) and run the full app, including loading a local `.405` file and exporting images.

---

## M6 — Event Prediction and Advanced Tools

**Goal:** The science-oriented power features from the original LTVT are re-implemented. This milestone can be developed partly in parallel with M5.

### Sub-milestones

**M6a — Moon Event Predictor**
- Rust WASM: `searchColongitude(startDate, endDate, targetColongitude)` → array of matching UTC timestamps
- Rust WASM: `searchFeatureAtTerminator(featureName, startDate, days)` → when does the terminator sweep past a feature
- UI: date range picker, target colongitude or feature name, results list with "Go To" buttons
- CLI: `ltvt predict --feature "Tycho" --days 30 --from 2026-05-26`

**M6b — Libration Tabulator**
- Rust WASM: `tabulateLibrations(startDate, stepDays, count)` → array of {date, subObsLon, subObsLat}
- UI: date range + step size + copy-to-clipboard / download as CSV
- CLI: `ltvt librations --from 2026-01-01 --to 2026-12-31 --step 1d --output csv`

**M6c — Photo Session Search**
- UI: upload a CSV of historical photo session dates
- Rust WASM: compute colongitude for each session, rank by proximity to target colongitude
- Results shown sorted with "Load Geometry" action

**M6d — Shadow Measurement Tool**
- Click mode: user clicks shadow tip, tool draws a line in the solar direction from the reference point
- Rust WASM: compute height from shadow length, sun angle, and known Moon scale
- Results saved to a downloadable text file
- CLI: `ltvt shadow --ref-lon <deg> --ref-lat <deg> --shadow-length-px <n> --image-scale <km-per-px>`

**M6e — Height Contours**
- Rust WASM: `computeContours(demData, heightIntervalKm)` → array of line segments in lon/lat
- Rendered as `THREE.LineSegments` overlay on the current view

**M6f — Photo Calibration**
- UI: dialog to mark known features in an uploaded photo image
- Rust WASM: least-squares solve for sub-observer point + rotation
- Calibrated photo rendered as a texture overlay with dots aligned
- Supports both ground-based (orthographic) and satellite (perspective) photos

### Definition of Done

`ltvt predict --feature "Clavius" --days 60` returns the next 60-day colongitude schedule from the CLI. All six sub-milestones have passing integration tests.

---

## M7 — Polish, Accessibility, and Public Release

**Goal:** The app is ready for public use by amateur astronomers.

### Deliverables

**Accessibility**
- [ ] Full keyboard navigation (Tab order, focus rings, ARIA labels)
- [ ] Screen reader support for the geometry readout panel
- [ ] High-contrast mode (respects `prefers-color-scheme: dark`)
- [ ] Reduced-motion mode (no animated transitions when `prefers-reduced-motion` is set)

**Performance**
- [ ] Initial web page load < 3 seconds on a 10 Mbit connection (lazy-load WASM and Three.js)
- [ ] Texture mode renders at interactive frame rate (≥ 30 fps) on a 2020-era integrated GPU
- [ ] DEM mode: adaptive step size reduces ray-march cost on low-end hardware
- [ ] Feature dot overlay for 9,300 features renders in < 100 ms (instanced mesh)

**Help System**
- [ ] In-app help panel with markdown content (no external `.chm` file)
- [ ] F1 / `?` tooltip on every control
- [ ] "What is this?" tooltips on all major UI elements
- [ ] Getting Started walkthrough (first-run experience)

**Web Deployment**
- [ ] Deployed to `ltvt.app` (or similar) via Cloudflare Pages
- [ ] PWA manifest + service worker: works offline after first load
- [ ] Ephemeris file for 2000–2050 cached after first fetch
- [ ] Share URL: any view state is URL-shareable
- [ ] OG meta tags for social sharing (screenshot of Moon in current geometry)

**Documentation**
- [ ] `README.md` in repo root: quick start for developers
- [ ] User guide (markdown, rendered in-app): Getting Started, all features
- [ ] API docs for `ltvt-core` (rustdoc) and `ltvt-wasm` (TypeDoc)
- [ ] CLI man pages (`ltvt help <command>`)

**Telemetry / Error Reporting**
- [ ] Optional anonymous error reporting (Sentry or Cloudflare Analytics)
- [ ] No personal data collected; fully functional with reporting disabled
- [ ] Privacy policy page

### Definition of Done

The web app passes a Lighthouse score of ≥ 90 on Performance, Accessibility, and Best Practices. Desktop binaries are available on GitHub Releases. The app is announced publicly.

---

## Dependency Graph

```
M0 ──► M1 ──► M2 ──► M3 ──► M4 ──► M5 ──► M7
                │                    │
                └──► M6a ────────────┘
                └──► M6b ─────► M7
                └──► M6c ─────► M7
                └──► M6d ─────► M7
                └──► M6e ─────► M4 (contours depend on DEM from M3)
                └──► M6f ─────► M4
```

M6a–M6d can begin once M2 is done (they only need the Rust WASM bridge, not the GPU renderer). M6e requires the DEM from M3. M6f requires the dot overlay from M4.

---

## Scope Cuts (Explicitly Out of Scope)

To keep the project deliverable, the following features from the original are intentionally **not** ported in the initial roadmap:

| Feature | Rationale |
|---|---|
| Support for all 9 non-Moon planets | Adds complexity with low demand; Moon focus is the core value. Can be added post-release. |
| LTO chart image viewer | The LTO series is a niche tool; map designation labels (M4) cover the primary use case |
| Rükl atlas page viewer | Same reasoning; Rükl numbers in the HUD are sufficient |
| `.chm` help format | Replaced by in-app markdown help |
| `PhotoSessions.csv` for Consolidated Lunar Atlas | Historical artifact; general photo session search (M6c) covers the use case |
| Windows-only `.ini` file format | Replaced by platform-native config via Tauri store plugin |
| Fixed 1024×768 window | Replaced by responsive layout |
| Delphi VCL UI components | Not applicable |

These can be revisited post-M7 based on user demand.

---

## Estimates (Rough Order of Magnitude)

These are solo-developer estimates. With a team, parallelism significantly reduces calendar time.

| Milestone | Effort |
|---|---|
| M0 — Foundation | 1–2 days |
| M1 — Rust Core + CLI | 2–3 weeks |
| M2 — WASM + Web Skeleton | 1–2 weeks |
| M3 — GPU Rendering | 2–3 weeks |
| M4 — Overlay + Interactivity | 2–3 weeks |
| M5 — Tauri Desktop | 1–2 weeks |
| M6 — Advanced Tools (all sub-milestones) | 3–4 weeks |
| M7 — Polish + Release | 2–3 weeks |
| **Total** | **~16–22 weeks** |

The single highest-risk item is M1 (DE405 reader validation). Everything else builds on known technology.

---

## Success Metrics

By the end of M7, the new LTVT should satisfy:

- **Accuracy:** Sub-observer/sub-solar points match the original Delphi exe to within 0.01° for 99% of test cases across 1950–2150
- **Performance:** Texture mode renders in < 50 ms per frame; DEM mode in < 500 ms at 1024×1024 with no cast shadows
- **Portability:** Runs correctly in Chrome 120+, Firefox 120+, Safari 17+ (web) and on macOS 13+, Windows 11, Ubuntu 22.04 (desktop)
- **CLI:** All geometry + prediction functions available as documented `ltvt` subcommands
- **Bundle size:** WASM module ≤ 500 KB gzipped; initial JS ≤ 200 KB gzipped
- **Accessibility:** WCAG 2.1 AA compliant
