---
summary: Analysis of LTVT's user-facing features, controls, and workflows — written from the perspective of porting to a modern stack.
last_updated: 2026-05-26
tags: [ltvt, user-guide, features, ux, porting]
---

# LTVT User-Facing Feature Analysis

Source version: **v0.21.4**  
Original distribution README: `ltvt/original_source/LTVT_ReadMe.txt`  
Source: `LTVT_Unit.pas` + all sub-form units

---

## 1. Program Overview

LTVT (Lunar Terminator Visualization Tool) renders simulated views of the Moon (or other solar system bodies) as seen from Earth at any date and time. Its central purpose is to let astronomers and observers answer the question: *"What does the Moon look like right now, and where is the terminator?"*

It does this by:
1. Computing the sub-observer and sub-solar points on the Moon's surface from a JPL planetary ephemeris
2. Using those points to construct an orthographic projection of the Moon onto a 641×641 px canvas
3. Shading the result using a texture map or a digital elevation model (DEM)
4. Overlaying IAU-named features as dots and labels

The program runs only on Windows and is fixed at 1024×768 display resolution. The window cannot be resized.

---

## 2. Main Window Layout

The window is divided into:

- **Left panel** — controls and inputs
- **Center** — 641×641 pixel Moon image canvas (`Image1`)
- **Right panel** — geometry readout, drawing buttons, status

The image is a `TImage` component backed by a `TBitmap`. All drawing is done to the canvas in memory and displayed in-place.

---

## 3. Core Geometry Inputs

These are the fundamental parameters that define the rendered view.

### Sub-Observer Point

| Control | Type | Purpose |
|---|---|---|
| Sub-Observer Longitude | `LabeledNumericEdit` | Selenographic longitude of the point on the Moon's surface directly facing the observer (degrees) |
| Sub-Observer Latitude | `LabeledNumericEdit` | Selenographic latitude of the same point |

In plain terms: this is the center of the Moon's visible disk as seen from the observer. For an Earth-based observer it is controlled by lunar libration.

The convention used depends on `PlanetaryLongitudeConvention` (East/West/Centered), configurable via **Cartographic Options**.

### Sub-Solar Point

| Control | Type | Purpose |
|---|---|---|
| Sub-Solar Longitude | `LabeledNumericEdit` | Selenographic longitude of the sub-solar point (the point on the Moon directly facing the Sun) |
| Sub-Solar Latitude | `LabeledNumericEdit` | Selenographic latitude of the same point |

These two points together determine where the terminator falls.

### Colongitude Display

Derived from sub-solar longitude: `Colong = 90 − SubSolLon`. Displayed live on the main form. The colongitude is the standard astronomical convention for describing which craters are near the terminator.

### Percent Illuminated

Computed from the angle between sub-observer and sub-solar points:  
`% = 50 × (1 + cos(angular separation))`

---

## 4. Date/Time and Ephemeris

### Date/Time Picker

Two `TDateTimePicker` controls: one for date, one for time (UTC). The date picker supports years as far back as 1601 (limited by Windows `SYSTEMTIME`). A "LinuxCompatibilityMode" relaxes this lower bound.

### "Now" Button

Sets date and time to the current system UTC, then immediately triggers **Estimate Geometry** (see below).

### Estimate Geometry Button

The core automation trigger. When clicked:
1. Reads the current UTC date/time from the pickers
2. Reads the observer location (lat/lon/elevation)
3. Opens the configured JPL ephemeris file if not already loaded
4. Calls `SubEarthPointOnMoon(MJD)` and `SubSolarPointOnMoon(MJD)` — full light-time-corrected, topocentric computation
5. Writes the resulting lon/lat values into the Sub-Observer and Sub-Solar input boxes
6. Updates the status labels: colongitude, % illuminated, Moon elevation, Moon diameter, ephemeris time

If no JPL file is loaded the button is non-functional and LTVT displays an error asking the user to locate a file.

### JPL Ephemeris Requirement

LTVT **cannot** compute lunar geometry without at least one JPL DE405 ephemeris file covering the requested date range. Files cover 50-year intervals (e.g., `unxp2000.405` covers 2000–2050). One file covering 2000–2050 is included with the distribution.

Users must download additional files from NASA/JPL for other date ranges:
```
ftp://ssd.jpl.nasa.gov/pub/eph/export/unix/
```

If the needed file is not loaded, LTVT prompts the user to locate it.

---

## 5. Observer Location

### "Set Location" Button

Opens the **Select Observer Location** dialog. The observer's geodetic coordinates are used for the topocentric computation:

| Field | Notes |
|---|---|
| Longitude (°E) | East-positive, −180 to +180 |
| Latitude (°N) | North-positive, −90 to +90 |
| Elevation (m) | Above sea level |

The dialog also supports loading from an `Observatory_List.txt` file (user-editable, plain text) which populates a drop-down. User can name and save custom locations.

The effect on the rendered image is the parallactic shift from geocenter to topocenter — visible as a small difference in sub-observer point that depends on the observer's distance from the Moon's center.

---

## 6. Drawing Modes

Three primary rendering modes, each with its own button. All three share the same orthographic projection geometry.

### 6.1 Texture Mode ("Texture" button)

Renders the Moon using a pre-loaded image (texture map). Three texture slots are available:

| Slot | Typical use |
|---|---|
| Texture 1 | Low-resolution USGS shaded relief (included: `lores.jpg`) |
| Texture 2 | High-resolution USGS or Clementine map (external) |
| Texture 3 | Any user-provided map; requires manual specification of lon/lat extent |

Radio buttons select which texture to use. Textures are loaded lazily (only when first selected). Hi-res textures take several seconds to load from disk.

**Gamma** control: applies `brightness^(1/gamma)` correction before rendering. Default 1.0 = no correction.

**Illumination modes** (radio buttons, active only for Texture and Dots rendering):
- **Normal** — use actual shading from sub-solar point
- **High Sun** — suppresses shadows; full disk illuminated
- **Low Sun** — enhances terminator region
- **Constant Sun Angle** — user specifies a fixed illumination angle, independent of actual solar position

### 6.2 Dot Mode ("Draw Dots" button)

Renders the Moon as a solid-colored disk (no texture), then overlays feature dots. Sunlit and shadowed hemispheres are shown in configurable colors (`DotModeSunlitColor`, `DotModeShadowedColor`).

The **Crater Threshold** field (km) filters the displayed feature set:
- `0` = show all named features
- `>0` = show only features larger than the specified diameter
- `-1` = show only features with the `UserFlag` column set in the CSV

### 6.3 DEM Mode ("Draw DEM" button)

Ray-traces the digital elevation model to produce a shaded-relief rendering. Much slower than texture mode. Options (from **DEM Options** dialog):

| Option | Effect |
|---|---|
| Photometric model | Lambertian / Lommel-Seeliger / Lunar-Lambert |
| Cast shadows | Ray-march in solar direction to find shadow-casting terrain |
| Multiply by Texture 3 | Multiply DEM brightness by Texture 3 albedo pixel |
| Correct for perspective | Use perspective (finite distance) rather than orthographic |
| DEM grid step | Controls sampling resolution (coarser = faster) |
| Gamma boost | Additional gamma applied to the final DEM image |
| Intensity boost | Multiplicative brightness boost |

The DEM is loaded from a separate `.IMG` file (PDS binary format). Two are included: `LDEM_16.IMG` (1.9 km/px) and `LDEM_64.IMG` (0.47 km/px).

**3D checkbox**: renders the DEM in pseudo-3D (isometric-like oblique view) rather than overhead.

---

## 7. Feature Overlay

### "Overlay Dots" Button

Plots dots for features from the loaded crater/feature CSV file, overlaid on the current image. Does **not** redraw the background — the existing texture or DEM image remains.

Features are filtered by the **Crater Threshold** value (same as in Dot Mode). A progress bar appears during rendering.

For each visible feature a dot is drawn in a size-dependent color:
- Small craters (< medium threshold): `SmallCraterColor`
- Medium craters: `MediumCraterColor`
- Large craters (>= large threshold): `LargeCraterColor`
- Non-craters: `NonCraterColor`

**Draw Circles checkbox**: enables drawing a scaled circle around each crater proportional to its diameter.

### "Label Dots" Button

Adds text labels to all currently plotted dots. Appears only after dots have been plotted.

### Label Preferences (menu: `Tools → Change Label Preferences`)

Configures how labels appear:

| Setting | Options |
|---|---|
| Include feature name | On/Off |
| Full crater names | On/Off (abbreviated vs. full IAU name) |
| Include feature size | On/Off |
| Include units | On/Off |
| Include discontinued names | On/Off (discontinued names are wrapped in `[brackets]`) |
| Dot offset | Radial displacement of label from dot center |
| Font | Face, size, color, style |
| Label placement | X and Y pixel offset from dot |

---

## 8. Right-Click Context Menu (on Image Canvas)

Right-clicking on the image opens a context menu with operations that depend on the current cursor position (i.e., on the lon/lat under the cursor):

| Item | Action |
|---|---|
| **Identify Nearest Feature** | Finds the closest plotted dot to the click point and shows its name, coordinates, and diameter |
| **Label Nearest Dot** | Adds a label to the nearest dot |
| **Label Feature and Satellites** | Labels a feature and all its lettered satellite craters |
| **Go To** | Centers the map on the clicked point |
| **Set Reference Point** | Sets the reference point for distance/bearing measurements |
| **Draw Lines to Pole and Sun** | Draws lines from the clicked point to the pole and to the sub-solar point |
| **Draw Circle** | Opens the circle-drawing tool centered at the clicked point |
| **Draw Contours** | Opens the DEM contour tool at the clicked point |
| **Nearest Dot to Reference Point** | Reports the closest dot to the reference point |
| **Nearest Dot Additional Info** | Shows additional data from the CSV for the nearest dot |
| **Count Dots** | Counts how many dots are currently plotted |
| **Record Shadow Measurement** | Enters shadow measurement mode |
| **Mouse Options** | Opens mouse configuration dialog |
| **Help** | Opens context-sensitive help |

---

## 9. Mouse Readout (Mouse Move)

While the cursor is over the lunar disk, the status area shows in real time:

- Selenographic longitude and latitude
- Sun angle at the cursor point (degrees above or below horizon)
- Distance and bearing from the reference point (if set)
- Pixel data from the underlying texture (if in texture mode)
- Elevation (if DEM is loaded)

---

## 10. Zoom and Pan

| Control | Effect |
|---|---|
| **Zoom** field | Zoom factor (1.0 = full disk; 2.0 = 2× magnification) |
| **Reset Zoom** button | Returns to zoom = 1.0 and center = (0, 0) |
| **Left-click on image** | Re-centers the map on the clicked point (redraw triggered) |

---

## 11. Cartographic Options Dialog

Opened via `Tools → Cartographic Options`. Controls how the map is oriented and what is overlaid:

| Setting | Options |
|---|---|
| **Orientation mode** | Line of Cusps / Cartographic / Equatorial / Alt-Az |
| **Rotation angle** | Manual clockwise rotation in degrees |
| **Invert L-R** | Mirror image left-right |
| **Invert U-D** | Mirror image upside-down |
| **Include libration circle** | Draws the ±6.7° libration boundary circle |
| **Libration circle color** | Color picker |
| **Include terminator lines** | Draws red (evening) and blue (morning) terminator circles |
| **No-data color** | Color used for pixels outside the texture area |
| **Annotate saved images** | Adds geometry text to the bottom of exported images |
| **Annotation colors** | Upper/lower label colors for saved images |
| **Start with current UT** | Automatically runs "Now" on program startup |

---

## 12. View Orientation Modes

| Mode | Description | Use case |
|---|---|---|
| **Line of Cusps** | Terminator runs horizontally across the image | Visual comparison with observed Moon |
| **Cartographic** | Selenographic north pole is always at the top | Atlas-style reference |
| **Equatorial** | Earth's celestial north pole defines "up" | Telescope with equatorial mount |
| **Alt-Az** | Observer's local zenith defines "up" | Alt-az telescope; matches naked-eye orientation |

Equatorial and Alt-Az modes require a prior "Estimate Geometry" computation from a specific time and location to be meaningful.

---

## 13. Grid Overlay

**Grid Spacing** field: if non-zero, draws latitude and longitude grid lines at the specified degree interval. Grid uses the libration circle color.

---

## 14. Image Save

| Control | Action |
|---|---|
| **Save Image** button | Saves the current canvas to a bitmap file; opens a save dialog |
| `Files → Save Image` | Same |

If "Annotate saved images" is enabled, geometry labels (sub-observer/sub-solar lon/lat, date/time, observer location) are printed along the top and bottom of the saved image.

---

## 15. Photo Sessions Search

**"Search Photo Sessions" button** opens the Photo Session Search dialog. This feature allows a user to search a database of historical photo session dates for sessions where the Moon's geometry (colongitude, libration) matched what would make a specific feature visible near the terminator.

Input CSV format: one row per photo session with at minimum a date and time. The ephemeris is used to compute colongitude for each session. Results are sorted by proximity to the requested colongitude. This was originally used for the Consolidated Lunar Atlas plates from the 1960s.

---

## 16. Moon Event Predictor

**"Predict" button** opens the Moon Event Predictor dialog. Searches over a date range for events such as:

- A specified feature being near the terminator (colongitude match)
- Feature rising above or setting below the terminator
- Specified illumination angle at a feature

This is useful for planning telescope observations.

---

## 17. LTO Chart Viewer

Opened via `File → Open an LTO Chart`. The Lunar Topographic Orthomap (LTO) series is a set of 1:250,000 scale cartographic charts published by the USAF/DMA in the 1970s. LTVT can load a scanned LTO chart image, and when "Transfer" is clicked, copies the chart's map projection parameters to the main window so that dots from the feature database are correctly projected onto the chart.

The LTO viewer supports three map projections:
- Orthographic
- Mercator
- Lambert Conformal Conic

The LTO map designation (e.g., `"62B3"`) is displayed in the mouse readout whenever the cursor is over a plotted feature.

---

## 18. Photo Calibration

Two calibrators let users overlay the feature database onto actual photographs:

### Ground-based Photo Calibrator (`Tools → Calibrate Photo`)
- User identifies at least two known features in the photo and provides their pixel coordinates
- LTVT solves for the photo's sub-observer point and orientation
- Feature dots from the database are then projected onto the photo

### Satellite Photo Calibrator (`Tools → Calibrate a Satellite Photo`)
- Additional parameter: the satellite's selenographic position (altitude, lon, lat)
- Uses a finite-perspective projection (camera at finite distance from Moon center)
- Otherwise same workflow as ground-based calibration

Once calibrated, `Load Calibrated Photo` shows the photo as a fourth "texture" source with dots aligned.

---

## 19. Shadow Measurement Tool

Right-click → **Record Shadow Measurement** enables a mode where:
1. The user clicks a point (tip of a shadow)
2. A line is drawn along the solar direction from the reference point
3. The tool reports the height of the object casting the shadow, derived from:
   - Sun angle at the reference point
   - Length of the shadow in pixels (scaled by the known angular size of the Moon)

Results are saved to `ShadowProfile.txt` in the LTVT folder.

---

## 20. Circle Drawing Tool

Right-click → **Draw Circle** (or `Tools → Draw Circle`) opens a dialog to draw a circle of specified angular radius centered on a given selenographic point. Used to:
- Outline crater rims
- Mark exclusion zones
- Illustrate angular distances

The circle is drawn using 1° arc steps, clipping the hidden portion (behind the limb).

---

## 21. DEM Height Contours

`Tools → Draw DEM Height Contours` opens a dialog to draw elevation contour lines on the current image at user-specified height intervals. Requires a DEM file to be loaded.

---

## 22. Libration Tabulator

`Tools → Tabulate Librations` opens a dialog that tabulates libration values (sub-observer lon/lat, libration in longitude and latitude) over a user-specified date range and step interval. Output is displayed in a scrollable text area and can be copied.

---

## 23. External File Management

`Files → Change External File Associations` opens a dialog listing all configurable file paths:

| File | Purpose |
|---|---|
| Texture 1 | Low-res texture map (default: `lores.jpg`) |
| Texture 2 | High-res texture map |
| Texture 3 | Supplementary texture / albedo map |
| Crater/Feature database | CSV feature list |
| JPL Ephemeris | DE405 binary file |
| Photo Sessions | CSV of historical observation dates |
| Calibrated Photos | List of previously calibrated photo calibrations |
| Observatory List | Named observer locations |

All paths can be absolute or relative to the LTVT.exe directory. Changes can optionally be saved as the new defaults.

---

## 24. Target Planet Selection

`Tools → Change Target Planet` opens a selection dialog for switching to a different solar system body:

- Moon (default)
- Mercury, Venus, Mars, Jupiter, Saturn, Uranus, Neptune, Pluto

When a non-Moon target is selected:
- The planet's physical radius and flattening are updated
- The coordinate system rotational parameters are derived from the IAU 2006 values (rather than JPL librations)
- The colongitude display is replaced by a "Central Meridian" display
- Feature overlays still work if a compatible CSV file is loaded

---

## 25. INI File and Settings

`Files → Save Options` writes all current settings to `LTVT.ini`. Settings persist across sessions. A different INI file can be specified with `Files → Change INI File`.

---

## 26. Keyboard Support

Almost every button and control has an associated `KeyDown` handler that processes **F1** to open context-sensitive help for that control. The main form also responds to:

- **Escape** — cancels a long-running rendering operation (sets `AbortKeyPressed := True`)

---

## 27. Platform Limitations (Relevant to Porting)

| Limitation | Notes |
|---|---|
| Windows-only | Uses VCL, Windows GDI, Windows `SYSTEMTIME`, `.chm` help format |
| Fixed 1024×768 window | Hard-coded; canvas is always 641×641 px |
| No multi-threading | Long renders (DEM mode) block the UI; only `Application.ProcessMessages` yields |
| No network access | JPL files must be downloaded manually and placed on disk |
| Single file per 50 years | Must manually chain multiple JPL files for long date ranges |
| No undo | Drawing operations are not undoable |
| No vector export | Output is always a raster bitmap |
| No scripting/automation | No command-line mode, no batch rendering |

---

## 28. Feature Summary Table

| Feature | Implemented | Notes |
|---|---|---|
| Orthographic Moon rendering | Yes | CPU per-pixel |
| Texture mapping (3 slots) | Yes | Simple cylindrical projection |
| DEM shaded relief | Yes | Ray-marching, 3 photometric models |
| DEM cast shadows | Yes | Optional; slow |
| Feature overlay (9,300 features) | Yes | IAU database |
| Feature labeling | Yes | Fully configurable |
| JPL ephemeris geometry | Yes | Full light-time correction |
| Observer location | Yes | Topocentric |
| Photo calibration (ground) | Yes | Orthographic |
| Photo calibration (satellite) | Yes | Perspective |
| LTO chart overlay | Yes | 3 projections |
| Shadow height measurement | Yes | |
| Moon event prediction | Yes | Colongitude-based search |
| Photo session search | Yes | Colongitude-based search |
| Libration tabulation | Yes | |
| Grid overlay | Yes | Configurable spacing |
| Terminator lines | Yes | Red/blue morning/evening |
| Libration circle | Yes | ±6.7° boundary |
| Distance/bearing measurement | Yes | From reference point |
| Height contours | Yes | Requires DEM |
| Circle drawing | Yes | Scriptable angular radius |
| Multi-planet support | Yes | Mercury–Pluto |
| Image save with annotations | Yes | BMP output |
| Texture export | Yes | Tiles in cylindrical projection |
| Context-sensitive help | Yes | F1 per control (`.chm`) |
| Automated tests | No | — |
| Batch/command-line mode | No | — |
| Scripting | No | — |
| Vector/SVG output | No | — |
| Network ephemeris | No | Manual file download |
| Undo | No | — |
| Resizable window | No | Fixed 1024×768 |
