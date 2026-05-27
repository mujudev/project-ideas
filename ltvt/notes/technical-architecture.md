---
summary: Deep-dive into how the LTVT Delphi 6 source code works — data pipeline, coordinate systems, rendering algorithms, photometric models, and ephemeris I/O.
last_updated: 2026-05-26
tags: [ltvt, architecture, delphi, algorithms, rendering, ephemeris, porting]
---

# LTVT Technical Architecture

Source version: **v0.21.4** (Borland Delphi 6, Windows only)  
Primary author: Jim Mosher (jimmosher@yahoo.com), 2008

---

## 1. Module Map

The codebase is split across three logical folders that roughly correspond to layers of the system:

```
Main Program/           ← UI, rendering, orchestration
  LTVT.dpr             ← entry point; creates and starts all forms
  LTVT_Unit.pas        ← ~8000 lines; the entire application logic
  MapFns_Unit.pas      ← LTO and Rukl atlas map-number helpers
  DEM_Data_Unit.pas    ← wrapper around the DEM data object
  LTO_Viewer_Unit.pas  ← embedded LTO chart image viewer
  *_Unit.pas (×17)     ← dialog/sub-form units (see §2)

Astro/                  ← pure astronomy and mathematics
  MoonPosition.pas     ← sub-observer/sub-solar point computation, coordinate systems
  H_JPL_Ephemeris.pas  ← JPL DE405 binary file reader and Chebyshev interpolator
  DE405d_EphemerisFile.pas ← data-block record definition (Delphi-native format)
  DE405_EphemerisFile.pas  ← same but for Unix big-endian binary format
  LunarDEM_Data.pas    ← Kaguya/JAXA PDS DEM file reader
  H_NOVAS.pas          ← port of the USNO NOVAS astrometry library
  H_ClockError.pas     ← UT1 and TDT offset tables (TAI–UTC leap seconds)
  IAU_FeatureNames_Info.pas ← lookup table of all 53 IAU feature-type codes
  MP_Defs.pas          ← shared constants (observer location, planet names, radii)
  TimLib.pas           ← time conversion utilities (sidtim, terra, etc.)
  MatLib.pas           ← matrix/rotation utilities
  MoonPosition.pas     ← full position pipeline

My Units/               ← reusable non-astronomical library code
  MPVectors.pas        ← TVector (3-element extended array), all vector math
  LabeledNumericEdit.pas ← custom numeric input component (requires Delphi palette install)
  JimsGraph.pas        ← simple 2-D plotting primitives (largely superseded by inline code)
  RGB_Library.pas      ← pixel-level color operations
  Zeroes.pas           ← root-finding (used for lunar age / new moon calculation)
  MATHOPS.PAS, MATRICES.PAS, MVECTORS.PAS ← older math utilities
  Win_Ops.pas          ← Windows-specific file/string helpers
  Constnts.pas         ← mathematical and astronomical constants
```

### Dialog Forms (all in `Main Program/`)

| Unit | Purpose |
|---|---|
| `H_Terminator_About_Unit` | About box |
| `H_Terminator_Goto_Unit` | Go-to a lon/lat or Xi-Eta coordinate |
| `H_Terminator_SetYear_Unit` | Set date by year number |
| `H_Terminator_SelectObserverLocation_Unit` | Observer lat/lon/elevation selector |
| `LibrationTabulator_Unit` | Tabulate librations over a date range |
| `H_PhotosessionSearch_Unit` | Search photo sessions file for matching geometry |
| `H_Terminator_LabelFontSelector_Unit` | Label font and placement preferences |
| `H_ExternalFileSelection_Unit` | Manage paths to texture, crater, JPL files |
| `DEM_Options_Unit` | DEM rendering photometric and shadow options |
| `H_MouseOptions_Unit` | Mouse click behavior configuration |
| `Satellite_PhotoCalibrator_Unit` | Calibrate satellite (orbital) photos |
| `LTO_Viewer_Unit` | Display and transfer LTO chart data |
| `H_PhotoCalibrator_Unit` | Calibrate ground-based photos |
| `CircleDrawing_Unit` | Draw circles / shadow profiles |
| `H_MoonEventPredictor_Unit` | Predict lunar events (e.g., grazes, occultations) |
| `H_CalibratedPhotoSelector_Unit` | Load a previously calibrated photo |
| `ObserverLocationName_Unit` | Name and save observer locations |
| `H_CartographicOptions_Unit` | Map orientation, terminator, libration circle |
| `DEM_ContourDrawing_Unit` | Draw height contours on DEM |
| `TargetSelection_Unit` | Switch target planet |
| `ExportTexture_Unit` | Export synthetic texture tiles |
| `CommentEntry_Unit` | Add text annotations |

---

## 2. Application Entry Point and Startup

`LTVT.dpr` creates all 22 forms upfront via `Application.CreateForm(...)`. All forms are allocated on startup; visibility is controlled by `Show`/`Hide`. The main form is `TLTVT_Form` (`LTVT_Unit.pas`).

`FormCreate` (`LTVT_Unit.pas:771`) runs the following startup sequence:

1. Set path globals (`BasePath`, `IniFileName`)
2. Configure locale (decimal separator `.`, no thousands separator)
3. Allocate `DEM_data : TDemData` object
4. Call `RefreshOptionsFromIniFile` — reads `LTVT.ini` and restores all user preferences
5. Optionally trigger "Now" button click (if `StartWithCurrentUT` is set), which immediately calls the ephemeris and draws
6. Allocate bitmaps for `UserPhoto` and `LTO_Image`

---

## 3. Data Pipeline Overview

```
User input (date/time + observer location)
         │
         ▼
EstimateData_ButtonClick
  → SubEarthPointOnMoon(MJD)     ← calls JPL ephemeris → PLEPH → STATE → INTERP
  → SubSolarPointOnMoon(MJD)     ← same
  → writes results to SubObs_*/SubSol_* numeric edit boxes
         │
         ▼
DrawTexture / DrawDots / DrawDEM button click
  → CalculateGeometry()
      - reads SubObs/SubSol lon/lat from form
      - converts to unit vectors (SubObsvrVector, SubSolarVector)
      - builds orthographic projection basis (XPrime, YPrime, ZPrime)
      - sets pixel↔coordinate mapping (SetRange)
         │
         ▼
  per-pixel rendering loop
      - for each screen pixel (i, j): XValue(i), YValue(j) → XProj, YProj
      - ConvertXYtoVector(XProj, YProj) → 3D point on unit sphere
      - dot-product visibility test with SubObsvrVector
      - look up texture pixel OR compute DEM brightness
      - write color to Image1.Canvas
         │
         ▼
  DrawTerminator / DrawGrid / MarkCenter overlaid on bitmap
```

---

## 4. Time System

All time handling flows through `H_ClockError.pas` and constants in `Constnts.pas`.

| Symbol | Meaning |
|---|---|
| `UTC_MJD` | Modified Julian Date in UTC: MJD = JD − 2400000.5 |
| `MJDOffset` | 2400000.5 — added to convert MJD to JD for JPL |
| `DUT` | UT1 − UTC in seconds (IERS table, `IERS_DUT`) |
| `EUT` | TDT − UTC in seconds (32.184 + leap seconds, `TDT_Offset`) |
| `TDB` | Barycentric Dynamical Time ≈ TDT; used as input to JPL PLEPH |

Time conversion in `MoonPosition.pas`:
```pascal
T0 := MJDOffset + UTC_MJD + TDT_Offset(UTC_MJD)*OneSecond/OneDay;
// T0 is now a Julian Date in Ephemeris Time (TDB)
```

Light-travel-time correction is performed explicitly:
```pascal
ReadEphemeris(T0, JPL_Moon, JPL_Earth, PositionsOnly, AU_day, Moon_Data);
LT1 := (OneAU * VectorMagnitude(Moon_Data.R) / c) / OneDay;
T1 := T0 - LT1;  // Moon was at T1 when the light we see now was emitted
```
This two-step light-time correction is applied separately for the sub-earth and sub-solar calculations.

---

## 5. JPL DE405 Ephemeris Engine

### File Format

Binary files (Unix big-endian, ~4.5 MB / 50-year block, extension `.405`). The record structure is defined in `DE405_EphemerisFile.pas` / `DE405d_EphemerisFile.pas`:

```
Block 1 (Header):  title lines, constant names/values, SS[1..3] (start/end JD + block length),
                   AU, EMRAT, IPT[1..12][1..3], NUMDE, LPT[1..3]
Block 2 (Constants): 400 named constant values (CVAL)
Blocks 3+: Data blocks, each covering SS[3] days of Chebyshev coefficients
           for 13 items: planets 1–9, geocentric Moon, Sun, nutations, librations
```

`H_JPL_Ephemeris.pas` provides a switch `{$DEFINE UnixFormat}` — when defined, byte-order conversion is done word-by-word via `JPLWordToDouble`; when undefined, the Delphi-native format can be read directly into the BUF array.

### Interpolation (Chebyshev)

The core is `INTERP` inside `STATE` (`H_JPL_Ephemeris.pas:359`):

1. Determine which sub-interval within the data block applies to the requested time
2. Compute normalized Chebyshev time: `TC ∈ [-1, 1]`
3. Evaluate Chebyshev position polynomial: `P(TC) = Σ cₙ Tₙ(TC)` using the recurrence `Tₙ = 2·TC·Tₙ₋₁ − Tₙ₋₂`
4. Optionally evaluate the derivative polynomial for velocity: `V(TC) = Σ cₙ Uₙ(TC)` using `Uₙ = 2·TC·Uₙ₋₁ + 2·Tₙ₋₁ − Uₙ₋₂`

Chebyshev polynomial arrays (`PC`, `VC`) are cached as typed constants and reused across calls to the same TC, giving a modest performance benefit.

### `ReadEphemeris` Interface

The public interface (`H_JPL_Ephemeris.pas:743`) wraps `PLEPH` with named enums:

```pascal
ReadEphemeris(TDB, JPL_Moon, JPL_Earth, PositionsOnly, AU_day, Moon_Data);
// returns Moon_Data.R : TVector  (position of Moon relative to Earth in AU)
```

`PLEPH` handles the Earth/Moon special case: the file stores the geocentric Moon and the Earth–Moon barycenter; heliocentric Earth coordinates are derived by combining these with EMRAT.

---

## 6. NOVAS Astrometry (`H_NOVAS.pas`)

A direct Delphi port of the USNO NOVAS-C library. Key functions used by LTVT:

| Function | Purpose |
|---|---|
| `tpplan` | Topocentric position of a planet (RA/Dec + distance) |
| `tpstar` | Topocentric position of a star |
| `sidtim` | Greenwich Mean/Apparent Sidereal Time |
| `zdaz` | Zenith distance and azimuth from RA/Dec |
| `terra` | Observer's position vector on Earth's surface |
| `pnsw` | Precession/nutation/sidereal-rotation transformation |
| `preces` | Precession of coordinates between epochs |

These are called only via `CalculatePosition` in `MoonPosition.pas`, which packages results into `PositionResultRecord`.

---

## 7. Selenographic Coordinate System

`SetSelenographicSystem(TDB)` in `MoonPosition.pas:329` constructs the transformation from the ICRF (International Celestial Reference Frame) to the Moon's body-fixed selenographic system.

For the Moon:
1. Read the three JPL libration Euler angles from the ephemeris (`ReadEphemeris(TDB, Librations, ...)`)
2. Apply `RotateCoordinateSystem` to get the Principal Axis (PA) system
3. Optionally correct PA → Mean Earth system using the Chapront et al. (1999) offsets `p1`, `p2`, `tau` (arcseconds)

```pascal
CorrectAxis(PA_System.UnitX, SelenographicCoordinateSystem.UnitX);
// uses RotateVector with p2, p1, tau about the three axes
```

For other planets (Mercury–Pluto), the IAU 2006 cartographic conventions are used: the pole direction (RA, Dec) and prime-meridian angle W are given as polynomial functions of time (from Seidelmann et al. 2007).

`SelenographicCoordinates(ICRF_Vector)` then projects any ICRF vector into body-fixed longitude/latitude/radius by dot products with the three unit axes.

---

## 8. Sub-Point Computation

Both `SubEarthPointOnMoon` and `SubSolarPointOnMoon` in `MoonPosition.pas` follow the same pattern:

1. Compute Moon–Earth vector at T0 (current observation time in TDB)
2. Back-propagate by light-travel time → T1 (when light was emitted)
3. For sub-solar: compute Sun–Moon vector at T2 (when solar light was emitted from the Sun toward the Moon)
4. Call `SetSelenographicSystem(T1)` to orient the body-fixed frame
5. Call `SelenographicCoordinates(RayDirection)` to convert the ray to selenographic lon/lat

For topocentric sub-earth: observer position on Earth is computed via `Terra()` using Greenwich Apparent Sidereal Time, then added to the geocentric Earth position before differencing.

---

## 9. Orthographic Projection

### Basis Vectors

`CalculateGeometry()` in `LTVT_Unit.pas:1287` constructs the three right-handed basis vectors for the projection:

- `ZPrime_UnitVector` — unit vector toward the observer (= sub-observer point on unit sphere)
- `YPrime_UnitVector` — "up" direction in projected image
- `XPrime_UnitVector` — "right" direction: `YPrime × ZPrime`

The "up" direction depends on `OrientationMode`:

| Mode | YPrime definition |
|---|---|
| `LineOfCusps` | `ZPrime × SubSolarVector` (normalized; terminator runs horizontally) |
| `Cartographic` | `Uy × ZPrime` (selenographic north pole projected upward) |
| `Equatorial` | `CelestialPole × ZPrime` (Earth north pole defines up) |
| `AltAz` | `ObserverZenith × ZPrime` (local vertical defines up) |

A manual rotation angle can be applied via `RotateVector(XPrime, θ, ZPrime)`.

### Screen-to-Sphere Mapping

`ConvertXYtoVector(XProj, YProj, MaxRadius)` (`LTVT_Unit.pas:1233`):
```
ZProj = sqrt(1 - XProj² - YProj²)   [only if XProj² + YProj² < 1]
3D point = XProj·XPrime + YProj·YPrime + ZProj·ZPrime
```

`ConvertXYtoLonLat` wraps this and extracts longitude/latitude via `ArcSin` and `ArcTan2`.

### Pixel ↔ Coordinate Mapping

`SetRange(Xmin, Xmax, Ymin, Ymax)` computes linear scale/offset factors. The coordinate range in the image spans `±ZoomFactor` (adjusted for image aspect ratio), centered on `(ImageCenterX, ImageCenterY)`.

```pascal
XPix(X) = fXScale * (X - fXOffset)
XValue(XPix) = XPix / fXScale + fXOffset
```

---

## 10. Drawing Modes

### Dot Mode (`DrawDotMode`)

`DrawDots_ButtonClick` iterates over the in-memory `CraterList[]` array. For each crater:
1. Convert lon/lat → 3D unit vector via `PolarToVector`
2. Check visibility: `DotProduct(CraterVector, SubObsvrVector) >= 0`
3. Project to screen: `XPix(CraterVector · XPrime)`, `YPix(CraterVector · YPrime)`
4. Draw dot and optionally a scaled circle representing crater diameter

### Texture Mode

`DrawTexture_ButtonClick` iterates over every pixel `(i, j)` of `Image1`:
1. Convert pixel to orthographic coordinates `(XProj, YProj)` via `XValue`, `YValue`
2. `ConvertXYtoVector` → unit vector in selenographic space
3. Extract lon/lat from the vector
4. Apply gamma: `cos(sun_angle)^(1/gamma)` for shading
5. Bilinear lookup in the loaded `TBitmap` texture (simple cylindrical projection)
6. Write pixel color to canvas

Texture files can be in simple cylindrical (equirectangular) projection spanning the full globe. Up to three textures can be configured (`Texture1`, `Texture2`, `Texture3`).

### DEM Mode (`DrawDEM_ButtonClick`, ~800 lines)

The most complex rendering path (`LTVT_Unit.pas:1929`). Per screen pixel:

1. **`SurfaceFound`**: Ray-marches along the observer direction until the ray intersects the DEM surface (elevation from `DEM_data.ReadHeight`). Both orthographic and perspective projections are supported.
2. **`ComputeSurfaceNormal`**: Numerically differentiates the DEM by sampling four neighboring points and taking the cross product of the two tangent vectors.
3. **`EvaluateBrightness`**: Applies one of three photometric models (see §11).
4. Optionally multiplies by a texture3 pixel value (`MultiplyDemByTexture3`).
5. Optionally ray-marches in the solar direction to compute **cast shadows** (`DemIncludesCastShadows`).

The DEM grid step size (resolution) is controlled by `DemGridStepMultiplier`.

---

## 11. Photometric Models

Defined in `DrawDEM_ButtonClick` (`LTVT_Unit.pas:1929`) and selected via `DemPhotometricMode`:

### Lambertian (`DotModeLambertian`)
```
Brightness = max(0, cos(sun_incidence_angle))
           = max(0, dot(SurfaceNormal, SubSolarVector))
```
Simple diffuse reflectance. Overly bright at high sun angles for the Moon.

### Lommel-Seeliger
```
μ  = cos(emission_angle)   = dot(SurfaceNormal, SubObsvrVector)
μ₀ = cos(incidence_angle)  = dot(SurfaceNormal, AdjustedSubSolarVector)
Brightness = μ₀ / (μ + μ₀)
```
The classical model for optically rough surfaces with single scattering. More physically accurate for the Moon than Lambertian.

### Lunar-Lambert (McEwen 1996)
```
L(α) = -0.019 + 0.242×10⁻³·α - 1.46×10⁻⁶·α²   [phase angle α in degrees]
Brightness = 2·L(α)·μ₀/(μ + μ₀) + (1 - L(α))·μ₀
```
A phase-angle-dependent blend of Lommel-Seeliger and Lambertian, calibrated against Clementine photometry. Constants `A`, `B`, `C` are hardcoded (`LTVT_Unit.pas:1931`).

---

## 12. DEM Data (`LunarDEM_Data.pas`)

The `TKaguyaData` (misnamed; actually reads any PDS-format DEM) class:

- Reads a PDS (Planetary Data System) label embedded in the same `.IMG` file
- Parses key-value pairs from the label: `LINE_SAMPLES`, `LINES`, `DUMMY_DATA`, `A_AXIS_RADIUS`, lon/lat extents
- Reads binary data as 4-byte single-precision floats in row-major order
- Missing data (NoDataValue) on polar rows is gap-filled from the adjacent row
- `ReadHeight(lon, lat)` performs nearest-neighbor lookup with longitude wrapping

Two DEM files are included in the distribution:
- `LDEM_16.IMG` — 16 px/deg (~1.9 km/px)
- `LDEM_64.IMG` — 64 px/deg (~0.47 km/px)

---

## 13. Feature Overlay System

### Data Structures

```pascal
TCrater = record
  UserFlag, Name, LatStr, LonStr : string;
  Lon, Lat : extended;          // radians
  NumericData : string;         // diameter in km
  USGS_Code : string;           // e.g. 'AA' = crater, 'ME' = mare
  AdditionalInfo1, AdditionalInfo2 : string;
end;
```

`CraterList[]` holds all features from the loaded CSV. `CraterInfo[]` is a subset — only features that were actually plotted in the current view (used for mouse hit-testing).

### CSV Format

The feature database (`CratersAndSub-CratersFeaturesIMPs2021-JohnMoore.csv`, ~9,300 entries) is a comma-delimited file parsed by `RefreshCraterList`. Expected columns:
```
UserFlag, Name, Lat, Lon, Diameter(km), USGS_Code, AdditionalInfo1, AdditionalInfo2
```

The `USGS_Code` determines dot color:
- `AA` (crater), `SF` (satellite feature), `CN`/`GD`/`CD` — size-colored dots
- All others — `NonCraterColor`

Crater circles are drawn only for features with a valid numeric diameter.

### IAU Feature Type Registry

`IAU_FeatureNames_Info.pas` contains a compile-time array of all 53 IAU feature type codes with singular/plural names and descriptions. Used for display/labeling only; the CSV uses the two-letter codes.

---

## 14. Map Reference Systems (`MapFns_Unit.pas`)

### LTO (Lunar Topographic Orthophotomap)

`LTO_String(lon, lat)` computes the standard LTO map designation for any selenographic point:
- 12 latitude bands (rows), each covering 10–16° of latitude
- Within each row, maps span variable longitude widths (20–45°)
- Each map is subdivided into quadrants A–D, then sub-quadrants 1–4

The function returns a string like `"62B3"`. `LTO_MapExists()` checks against a hardcoded list of 215 known LTO map identifiers.

### Rukl Atlas

`Rukl_String(lon, lat)` maps nearside features to Antonín Rükl's atlas numbering (1–76), based on an 8×11 grid in Xi-Eta orthographic space. Returns empty string for features on the far side.

---

## 15. Photo Calibration System

Two modes exist for overlaying calibrated photographs:

### Earth-Based (`H_PhotoCalibrator_Unit`)
- Orthographic projection: `UserPhoto_ZPrime` points from Moon center to sub-observer point
- Pixel coordinates calculated by dot products into the `XPrime`/`YPrime` plane
- `ConvertLonLatToUserPhotoXY` projects selenographic lon/lat to photo X/Y

### Satellite (`Satellite_PhotoCalibrator_Unit`)
- Finite-perspective projection: camera at `SatelliteVector` (position in km from Moon center)
- `ConvertLonLatToUserPhotoXY` computes the ray from camera to feature, normalizes it, and projects via `DotProduct` with the camera X/Y axes
- `ConvertUserPhotoXYtoLonLat` intersects the back-projected ray with the lunar sphere (quadratic formula) to find the surface point

Both modes support an arbitrary rotation angle (`UserPhotoSinTheta`, `UserPhotoCosTheta`) between image axes and the selenographic projection axes.

---

## 16. Coordinate Conversion Summary

| Function | Input | Output |
|---|---|---|
| `PolarToVector` (MPVectors) | lon, lat, radius | TVector [x,y,z] |
| `VectorToPolar` (MPVectors) | TVector | lon, lat, radius |
| `ConvertXYtoVector` | XProj, YProj (±1) | TVector in seleno system |
| `ConvertXYtoLonLat` | XProj, YProj | lon, lat (radians) |
| `SelenographicCoordinates` | ICRF TVector | lon, lat, radius (radians/km) |
| `ConvertLonLatToUserPhotoXY` | lon, lat, radius | photo X, Y |
| `ConvertUserPhotoXYtoLonLat` | photo X, Y, radius | lon, lat |
| `LTO_String` | lon, lat (radians) | e.g. `"62B3"` |
| `Rukl_String` | lon, lat (radians) | e.g. `"43"` |

---

## 17. Persistence (INI File)

`LTVT.ini` (same directory as the executable) stores all user preferences using Delphi's `TIniFile`. Key sections:

- `[FileOptions]` — paths to texture, crater, JPL, photo-sessions files
- `[LocationOptions]` — observer lat/lon/elevation
- `[CartographicOptions]` — orientation mode, terminator display, libration circle
- `[LabelOptions]` — dot colors, label font, placement
- `[DemOptions]` — photometric model, shadow, gamma
- `[MouseOptions]` — click behavior, cursor type

Options are read on startup by `RefreshOptionsFromIniFile` and written on demand via `Saveoptions_MainMenuItemClick`.

---

## 18. Test Coverage and Validation

**There are no automated tests in this codebase.** The author explicitly notes the code was written for personal research use and was not designed for formal verification.

### Implicit Validation Mechanisms

1. **Ephemeris cross-checks**: The code displays computed colongitude, percent illumination, and Moon elevation. Users can compare these against published ephemeris tables (e.g., the USNO Astronomical Almanac).

2. **Visual inspection**: The primary validation method — rendered images were compared against actual photographs of the Moon. The photo calibration tool was specifically designed for this purpose.

3. **LTO chart overlay**: Loading an actual LTO chart (a precisely calibrated orthophoto) and using "Transfer" to copy its geometry to the main window serves as an end-to-end validation of the projection math.

4. **Satellite photo calibration**: Projecting feature database dots onto calibrated satellite images validates the coordinate transforms.

5. **Shadow measurement**: The shadow measurement tool computes crater depths from shadow lengths and solar angle. Computed depths that agree with published values validate the geometry.

6. **Cross-platform consistency**: A `LinuxCompatibilityMode` flag exists, suggesting some testing was done against Wine or a Mono-era environment, though full support was never achieved.

### Known Precision Limitations

- Photometric models are approximations; `Lunar-Lambert` constants (McEwen 1996) are valid for Clementine data but may not generalize to other texture maps.
- Light-travel-time corrections are iterated only once (not converged), introducing negligible error at lunar distances.
- `ConvertXYtoLonLat` uses a nearest-neighbor texture lookup with no sub-pixel interpolation in the dot/texture modes, causing visible staircase artifacts on zoomed views.
- The DEM `ReadHeight` function uses nearest-neighbor sampling (no bilinear interpolation).

---

## 19. Porting Considerations

| Aspect | Delphi 6 implementation | Modern equivalent |
|---|---|---|
| UI framework | Delphi VCL (Windows-only forms) | Web (React/HTMX), Electron, Qt, GTK |
| Main rendering loop | CPU per-pixel on `TBitmap.Canvas` | WebGL/WASM shader; GPU fragment shader per pixel |
| Photometric models | Inline floating-point | GLSL fragment shaders |
| JPL ephemeris | Binary file, Chebyshev interp in Pascal | SPICE toolkit, `erfa`, `astropy`, or direct port |
| DEM data | PDS binary `.IMG` file | GeoTIFF or cloud-optimized formats |
| Feature database | CSV parsed at runtime | SQLite, PostGIS, GeoJSON |
| Persistence | Windows INI file | JSON config, localStorage (web), platform config dirs |
| Photo calibration | Per-session in-memory | Could be serialized as FITS WCS headers |
| Coordinate math | Custom Pascal vector library | `numpy`, `astropy.coordinates`, `erfa`, GLSL |
| Time handling | Custom TAI table + IERS_DUT | `astropy.time`, IERS auto-update |
| Thread model | Single-threaded, `Application.ProcessMessages` for UI | Web Workers, async/await, or native threads |
| Texture loading | `TBitmap` (Windows DIB) | Any image library (sharp, Pillow, three.js TextureLoader) |

The rendering inner loop is the most performance-critical part and is the strongest candidate for GPU acceleration. The per-pixel ray-march for DEM rendering is O(n·m·k) where n×m is pixel count and k is the number of DEM samples along the ray — this is a natural fit for a GPU compute shader.
