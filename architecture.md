# GB Studio Assistant - Complete Technical Architecture

## Purpose

GB Studio Assistant is a real-time image-to-Game-Boy graphics converter. It takes any input image and converts it into a Game Boy-compatible 4-color palette image suitable for import into GB Studio. The tool provides a full non-destructive editing pipeline: image transformation, color adjustment, dithering, region-based editing, manual pixel painting, tile budget management, and export with automatic palette remapping.

This document describes the **core graphics engine** - the algorithms, data structures, and processing pipeline that produce GB Studio-compatible 8-bit output. Meta functions (file management, naming, project integration) are handled by the surrounding ecosystem and are not covered here.

---

## Core Constraints (GB Studio / Game Boy Hardware)

These constraints drive every design decision in the app:

1. **4-color palettes only.** The Game Boy displays exactly 4 colors per palette. Every pixel in the output must map to one of 4 palette colors, ordered lightest to darkest.
2. **8x8 pixel tiles.** The Game Boy renders graphics as 8x8 pixel tiles. GB Studio imposes limits on unique tile count.
3. **197 unique tile limit.** For GB Studio "adventure mode" caption images, the maximum is 197 unique 8x8 tiles. Exceeding this causes import failure.
4. **DMG green export palette.** GB Studio expects assets in the classic DMG green palette (`#e0f8cf`, `#86c06c`, `#306850`, `#071821`). Regardless of what palette the user works in, export remaps to these 4 specific colors by index position.
5. **Standard canvas sizes.** Common dimensions: 160x144 (full GB screen), 80x72 (half), 20x18, 160x16, 256x224 (Super GB), 320x288 (double SGB). Custom sizes up to 512x512 supported.

---

## Application State

A single central state object (`S`) holds all mutable state. No framework - plain JavaScript with direct DOM manipulation.

```javascript
{
  // Source image
  image: HTMLImageElement | null,  // The loaded source image
  imgW: number,                     // Source image natural width
  imgH: number,                     // Source image natural height

  // Output canvas dimensions
  gbW: 160,                         // Output width in pixels (default: Game Boy screen)
  gbH: 144,                         // Output height in pixels

  // Image transform (how source maps onto output canvas)
  imgCX: number,                    // Image center X in output-pixel coords
  imgCY: number,                    // Image center Y in output-pixel coords
  imgZoom: number,                  // Scale factor (1.0 = native pixel size)
  imgRot: number,                   // Rotation in degrees (-180 to 180)
  flipH: boolean,                   // Horizontal flip
  flipV: boolean,                   // Vertical flip

  // Color adjustments (applied per-pixel before palette quantization)
  brightness: number,               // -100 to 100
  contrast: number,                 // -100 to 100
  saturation: number,               // -100 to 100
  bw: number,                       // Black & white mix, 0-100%
  gamma: number,                    // Gamma correction, 0.2 to 2.8 (1.0 = linear)

  // Palette & quantization
  posterize: boolean,               // Whether to apply palette quantization
  dither: string,                   // Dithering algorithm key
  palKey: string,                   // Palette preset key (or 'custom')
  customColors: string[4],          // 4 hex color strings for custom palette

  // Paint layer (manual pixel overrides)
  paintLayer: Uint8Array | null,    // One byte per pixel: 0-3 = palette index, 255 = unpainted
  bakedLayer: Uint8ClampedArray | null, // RGBA pre-palette frozen pixels from bake operations

  // Selection (marquee)
  marquee: Object | null,           // { type: 'rect'|'circle'|'free', ... }
  baseAdj: Object | null,           // Frozen adjustment snapshot for area outside selection
  freePoints: Array,                // Point array for lasso selection

  // Tools
  tool: string,                     // 'pan' | 'pencil' | 'msel' | 'csel' | 'free'
  pencilColor: number,              // 0-3 palette index for pencil tool

  // Undo/redo
  undoStack: Array,                 // Max 30 snapshots
  redoStack: Array,

  // Tile management
  tileCount: number | null,         // Current unique 8x8 tile count
  tileAgg: number,                  // Simplification aggressiveness (0=soft, 1=med, 2=hard)

  // Display
  displayScale: number,             // Auto-computed canvas display multiplier
  exportScale: number,              // Export size multiplier (1, 2, 4, or 8)
  showGrid: boolean,                // Grid overlay visibility
  zoomLocked: boolean,              // Locks all transform controls
}
```

---

## Image Processing Pipeline

The pipeline runs every frame via `requestAnimationFrame`. The core function `buildFrame()` produces a complete output `ImageData` object.

### Stage 1: Image Transformation

Draw the source image onto an offscreen canvas at the output dimensions (`gbW x gbH`):

```
1. Clear offscreen canvas (gbW x gbH)
2. Translate to (imgCX, imgCY) - the image center point
3. Rotate by imgRot degrees
4. Scale by imgZoom, applying flipH/flipV as negative scale
5. Draw source image centered at origin (-imgW/2, -imgH/2)
6. Extract ImageData (RGBA pixel array)
```

This maps an arbitrarily-sized source image onto the fixed Game Boy output canvas with pan, zoom, rotation, and flip.

### Stage 2: Color Adjustments (Dual Model)

Apply brightness, contrast, saturation, B&W mix, and gamma to every pixel. The **dual adjustment model** enables region-specific editing:

- If a **marquee selection exists**: pixels inside the selection use the live adjustment values; pixels outside use the frozen `baseAdj` snapshot (captured when the selection was created).
- If **no selection**: all pixels use the live values.

This allows the user to adjust a selected region without affecting the rest of the image.

**Adjustment order (per pixel):**

1. **Brightness**: `pixel += brightness * 2.55` (linear offset, maps -100..100 to -255..255)
2. **Contrast**: S-curve via `factor = 259*(contrast+255) / (255*(259-contrast))`, then `pixel = factor*(pixel-128) + 128`
3. **Saturation**: Convert RGB to HSL, scale S by `(saturation+100)/100`, convert back
4. **B&W Mix**: Blend toward luminance using ITU-R BT.601 weights: `lum = 0.299*R + 0.587*G + 0.114*B`, then `pixel = pixel*(1-t) + lum*t` where `t = bw/100`
5. **Gamma**: Power curve: `pixel = (pixel/255)^(1/gamma) * 255`

All values clamped to [0, 255] after each step.

### Stage 2b: Baked Layer Composite

If a `bakedLayer` exists (from prior bake operations), its non-transparent pixels override the adjusted pixels. The baked layer stores raw RGB values (pre-palette), making it palette-agnostic - changing the palette re-quantizes baked pixels naturally.

### Stage 3: Palette Quantization & Dithering

If `posterize` is enabled, quantize every pixel to the nearest of the 4 palette colors, using the selected dithering algorithm. Then apply the paint layer (manual pixel overrides).

---

## Color Science

### Color Distance Function

Uses a weighted Euclidean distance that approximates human color perception (CIE 1976 delta-E approximation):

```javascript
function colDist(r1, g1, b1, r2, g2, b2) {
  const rm = (r1 + r2) / 2;
  const dr = r1 - r2, dg = g1 - g2, db = b1 - b2;
  return (2 + rm/256) * dr*dr + 4 * dg*dg + (2 + (255-rm)/256) * db*db;
}
```

The red channel weight increases and blue channel weight decreases as the average red value increases. Green is always weighted highest (4x). This accounts for the human eye's greater sensitivity to green and varying sensitivity to red vs blue.

### Nearest Color Lookup

Exhaustive search over the 4-color palette using `colDist`. With only 4 candidates this is O(1) per pixel and not worth optimizing further.

### RGB <-> HSL Conversion

Standard conversion used for saturation adjustment. RGB values normalized to [0,1], standard hue/saturation/lightness calculation. HSL to RGB uses the standard piecewise linear function.

---

## Dithering Algorithms

11 methods, divided into three categories:

### Ordered Dithering (7 methods)

Add a threshold value from a repeating matrix to each pixel before nearest-color lookup. The matrix creates a regular pattern that simulates intermediate colors through spatial mixing.

**General formula:**
```
threshold = matrix[y % size][x % size] * strength
quantized = nearestColor(pixel + threshold)
```

where `strength = 80` (constant).

**Matrix construction:** All matrices are normalized to the range [-0.5, 0.5] via: `normalized = (raw_value + 0.5) / matrix_element_count - 0.5`

| Method | Matrix Size | Character |
|--------|------------|-----------|
| Bayer 2x2 | 2x2 | Coarse, visible pattern |
| **Bayer 4x4** (default) | 4x4 | Good balance of detail and pattern |
| Bayer 8x8 | 8x8 | Fine detail, less visible pattern |
| Cluster Dot | 6x6 | Halftone/newspaper print look |
| Horiz. Scanlines | 4x4 | CRT horizontal line effect |
| Vert. Scanlines | 4x4 | Vertical stripe effect |
| Diagonal Lines | 4x4 | 45-degree stripe pattern |

**Bayer matrix values:**

```
BAYER2 raw: [[0,2],[3,1]]  (divide by 4)
BAYER4 raw: [[0,8,2,10],[12,4,14,6],[3,11,1,9],[15,7,13,5]]  (divide by 16)
BAYER8: Generated via fractal bit-reversal algorithm (3-bit interleave of x,y coords)
CLUSTER6 raw: [[34,29,17,21,30,35],[28,14,9,13,15,31],[16,8,4,5,10,18],
               [20,12,3,0,6,22],[27,11,7,1,2,23],[33,26,19,24,25,32]]  (divide by 36)
```

Scanline and diagonal matrices use modified values - non-pattern rows/columns are set to 0.49 (near the max) to force nearest-color behavior, while pattern rows use the standard normalized threshold scaled by 0.8.

### Error Diffusion (2 methods)

Process pixels left-to-right, top-to-bottom. Quantize each pixel, compute the error (difference between original and quantized), and distribute that error to neighboring unprocessed pixels.

**Floyd-Steinberg:** Distributes to 4 neighbors:
```
         X    7/16
  3/16  5/16  1/16
```
Full error is distributed (sums to 16/16). Produces high-fidelity results with minimal artifacts.

**Atkinson:** Distributes to 6 neighbors, each receiving 1/8 of the error:
```
        X    1/8   1/8
  1/8  1/8   1/8
       1/8
```
Only 6/8 of error is distributed (2/8 is discarded). Produces a more stylized, "Mac classic" look with higher contrast and less color bleeding. Better for the 4-color constraint because it avoids over-diffusing into extreme palette entries.

Both use a shared `Float32Array` error buffer, lazily allocated and reused across frames.

### Special (2 methods)

**Checkerboard:** For each pixel, find the two nearest palette colors. Alternate between them in a checkerboard pattern `(x+y) % 2`. Creates a uniform 50/50 dither between the two closest colors.

**None:** Pure nearest-color quantization with no dithering. Each pixel maps to whichever of the 4 palette colors is closest.

---

## Palette System

### Built-in Palettes (15)

Each palette is exactly 4 colors, ordered lightest to darkest:

```javascript
{
  dmg:       ['#e0f8cf','#86c06c','#306850','#071821'],  // GB Classic green
  grey:      ['#f8f8f8','#a8a8a8','#585858','#080808'],  // Grayscale
  icecream:  ['#fff6d3','#f9a875','#eb6b6f','#7c3f58'],  // Warm pastel
  mist:      ['#c4f0c2','#5ab9a8','#1e606e','#2d1b00'],  // Cool green-blue
  rustic:    ['#fffbe7','#ead4aa','#e07855','#26160e'],  // Earth tones
  hollow:    ['#fbf7f3','#e5b083','#426e5d','#20283d'],  // Muted natural
  spacehaze: ['#f8e3c4','#f00041','#6621a3','#050007'],  // High contrast neon
  velvet:    ['#9775a6','#683a68','#412752','#2d1b2e'],  // Purple monochrome
  kirokaze:  ['#e2f3e4','#94e344','#46878f','#332c50'],  // Modern GBC
  links:     ['#ffffb5','#7bc67b','#6b8c42','#5a3921'],  // Link's Awakening
  pokemon:   ['#e0f8cf','#86c06c','#306850','#071821'],  // Pokemon GSC (same as DMG)
  red:       ['#ffffff','#ff9999','#cc0000','#1a0000'],  // Red theme
  ocean:     ['#d8f0f8','#7ecef4','#2c79be','#0d2b45'],  // Blue theme
  sunset:    ['#ffeecc','#f8a040','#d03038','#202040'],  // Orange to purple
  ayy4:      ['#f0f0e0','#b0a090','#705848','#281810'],  // Warm grey
}
```

### Custom Palette

Users can modify any of the 4 color values via color pickers. When a color is changed, the palette key switches to `'custom'` and the `customColors` array is updated.

### GB Studio Export Palette (Constant)

```javascript
const GBS_PAL = ['#e0f8cf', '#86c06c', '#306850', '#071821'];
```

This is the DMG green palette. During GB Studio export, every pixel is remapped from its position in the user's working palette to the corresponding position in GBS_PAL. The remapping is by **index position**, not by color similarity:
- User palette color 0 (lightest) -> GBS_PAL[0]
- User palette color 1 -> GBS_PAL[1]
- User palette color 2 -> GBS_PAL[2]
- User palette color 3 (darkest) -> GBS_PAL[3]

Implementation: For each pixel, find the nearest color in the user's palette (by `colDist`), get that index, then substitute `GBS_PAL[index]`.

---

## Selection System

Three selection types for region-based operations:

### Rectangle Selection
```javascript
{ type: 'rect', gx, gy, gw, gh }  // grid-pixel coordinates and dimensions
```
Point-in-selection test: `gx >= mx && gx < mx+gw && gy >= my && gy < my+gh`

### Circle/Ellipse Selection
```javascript
{ type: 'circle', gx, gy, gw, gh }  // bounding box
```
Point-in-selection test: ellipse equation using center and radii derived from bounding box:
```
center = (gx + gw/2, gy + gh/2)
radii = (gw/2, gh/2)
test: ((px - cx) / rx)^2 + ((py - cy) / ry)^2 <= 1
```

### Lasso (Free-form) Selection
```javascript
{ type: 'free', points: [{x, y}, ...] }  // array of grid-pixel vertices
```
Point-in-selection test: **ray casting algorithm** (point-in-polygon). Cast a horizontal ray from the test point to infinity, count how many polygon edges it crosses. Odd = inside, even = outside.

```javascript
function pointInPoly(pts, px, py) {
  const x = px + 0.5, y = py + 0.5;  // test pixel center
  let inside = false;
  for (let i = 0, j = pts.length - 1; i < pts.length; j = i++) {
    const xi = pts[i].x, yi = pts[i].y, xj = pts[j].x, yj = pts[j].y;
    if (((yi > y) !== (yj > y)) && (x < (xj-xi)*(y-yi)/(yj-yi) + xi))
      inside = !inside;
  }
  return inside;
}
```

### Selection Effects

When a selection exists:
1. **Adjustments** (brightness, contrast, etc.) only affect pixels inside the selection. Exterior pixels use the frozen `baseAdj` snapshot.
2. **Pencil tool** only paints inside the selection.
3. **Tile simplification** only modifies tiles/pixels inside the selection.
4. **Bake** freezes the selection's adjusted pixels into the baked layer.

### Marching Ants Animation

The selection boundary is rendered with animated dashed lines:
- White solid stroke (background)
- Black dashed stroke (4px on, 4px off) with a scrolling `lineDashOffset`
- Offset increments every 80ms, cycling through 12 positions

---

## Paint & Bake System

### Paint Layer

A `Uint8Array` with one byte per output pixel. Values:
- `0-3`: Forced palette color index (overrides the pipeline result)
- `255`: Unpainted (pipeline result passes through)

The paint layer is applied **after** palette quantization in the pipeline. This means painted pixels are always one of the 4 palette colors and are unaffected by adjustments or dithering.

### Baked Layer

A `Uint8ClampedArray` in RGBA format (4 bytes per pixel). Stores **pre-palette** pixel values - raw RGB colors after adjustments but before quantization. This is palette-agnostic: if the user changes the palette, baked pixels get re-quantized through the new palette.

**Bake workflow:**
1. User creates a selection
2. User adjusts brightness/contrast/etc. (affects only selection interior)
3. User clicks "Bake Adjustments into Selection"
4. Pipeline runs up to but not including palette quantization
5. Selection interior pixels are copied to `bakedLayer` as raw RGB
6. Image transform is **locked** (prevents accidental repositioning)
7. All adjustment sliders reset to defaults
8. Selection is cleared

Multiple bakes accumulate - previous baked pixels are composited before new bake.

---

## Tile Management

### Tile Counting

The image is divided into 8x8 pixel tiles. Each tile is hashed using **FNV-1a 32-bit** to detect uniqueness:

```javascript
hash = 0x811c9dc5  // FNV offset basis
for each pixel in tile (as 32-bit RGBA packed integer):
  hash ^= pixel
  hash = Math.imul(hash, 0x01000193)  // FNV prime
hash = hash >>> 0  // unsigned
```

All hashes are collected in a `Set`. The set size = number of unique tiles.

Tile counting is **debounced** (90ms) to avoid performance issues during rapid editing. A cached frame (`lastFrameForTiles`) stores the most recent posterized output for recount without re-rendering.

### Tile Budget UI

- Visual progress bar: green (<=85%), orange (85-100%), red (>100% of 197 limit)
- Numeric display: "X / 197"
- Warning text changes based on status

### Tile Simplification Algorithm

Reduces unique tile count by forcing tiles to use fewer colors:

**Parameters:** Aggressiveness level:
- Level 0 (Soft): Keep top 2 colors per tile; skip tiles already at <=2 colors
- Level 1 (Med): Keep top 2 colors per tile always
- Level 2 (Hard): Keep only the single most common color (monochrome tiles)

**Algorithm per tile:**
1. Build a histogram of the 4 palette colors used in the tile (only counting pixels inside the marquee if a selection exists)
2. Sort colors by frequency (most used first)
3. Keep the top N colors (N depends on aggressiveness)
4. For each pixel using a non-kept color: remap to the nearest kept color (by `colDist`)
5. Write remapped palette indices to the paint layer

This operates on the paint layer, so it's non-destructive to the underlying image - clearing the paint layer reverts simplification.

---

## Export System

### GB Studio Export

1. Force posterize ON (temporarily if it was off)
2. Build the full pipeline frame
3. Remap every pixel from the user's working palette to GBS_PAL by index:
   - For each pixel, find the nearest color in the user's palette -> get index `i`
   - Replace pixel with `GBS_PAL[i]`
4. Scale by export multiplier (1x, 2x, 4x, or 8x) using nearest-neighbor (pixelated) scaling
5. Export as PNG

**Filename format:** `{name}_{W}x{H}_gbstudio_{hex0}-{hex1}-{hex2}-{hex3}_{scale}x.png`

The palette hex values in the filename encode the user's **working** palette (not the DMG export palette), providing a reference for which colors were used during authoring.

### Preview Export

Same as GB Studio export but **without** the palette remap. Exports with the user's working palette colors as-is.

**Filename format:** `{name}_{W}x{H}_{palKey}_preview_{scale}x.png`

### Drag-to-Export

Both export buttons support HTML5 drag. On `dragstart`:
1. Build export canvas
2. Create a scaled ghost image for the drag preview
3. Set multiple data transfer formats:
   - `DownloadURL`: `image/png:{filename}:{blobURL}` (native browser download)
   - `text/html`: `<img>` embed tag with data URL
   - `text/uri-list`: Direct data URL

---

## Undo/Redo System

### Snapshot Contents

A snapshot captures all mutable editing state (not image data or transform):

```javascript
{
  paint: Uint8Array copy | null,
  baked: Uint8ClampedArray copy | null,
  brightness, contrast, saturation, bw, gamma,
  palKey, customColors: [...],
  dither, posterize,
  marquee: deep copy | null,
  baseAdj: copy | null,
  freePoints: [{x,y}, ...]
}
```

### Mechanics

- **Max stack depth:** 30 snapshots
- `pushUndo()` is called **before** any state-changing operation
- Any new edit clears the redo stack
- Undo pops from undoStack, pushes current state to redoStack, restores popped snapshot
- Redo is the reverse
- All typed arrays are `.slice()` copied (deep copy)
- Marquee objects are `JSON.parse(JSON.stringify())` deep copied

---

## Canvas & Display System

### Display Scaling

The output canvas is displayed at an integer multiple of its native size:

```javascript
displayScale = max(1, min(8, min(
  floor((containerWidth - 60) / gbW),
  floor((containerHeight - 60) / gbH)
)))
```

Auto-recomputed via `ResizeObserver` when the container resizes. The 60px padding prevents the canvas from touching edges.

### Zoom System

Image zoom (how large the source image appears within the output canvas) uses exponential slider mapping:

```javascript
sliderToZoom(v) = 0.01 * (800)^(v/1000)     // v: 0-1000 slider value
zoomToSlider(z) = log(z/0.01) / log(800) * 1000
```

Range: 1% to 800%. Exponential mapping gives fine control at low zoom and coarse control at high zoom.

**Snap points:** [1, 2, 5, 10, 15, 25, 33, 50, 67, 75, 100, 125, 150, 200, 300, 400, 600, 800]%
When the zoom slider is released, if the current zoom is within +/-2% of a snap point, it snaps to that value.

### Grid Overlay

Two grid levels:
- **Pixel grid:** Every pixel boundary, drawn at 8% black opacity
- **Tile grid:** Every 8th pixel, drawn at 20% indigo opacity (shows 8x8 tile boundaries)

### Render Loop

```
doRender() via requestAnimationFrame:
  1. Clear display canvas
  2. Draw ghost image (source image at 18% opacity showing full extent)
  3. Build frame (full pipeline)
  4. Draw frame to display canvas (nearest-neighbor upscale)
  5. Update pixel preview panel canvas
  6. Draw grid overlay (if enabled)
  7. Draw selection marching ants (if selection exists)
  8. Draw border frame and corner crosshairs
```

The ghost image layer shows the user where their source image extends beyond the output canvas boundaries.

---

## Interaction Model

### Pan Tool
- Mouse drag: translate `imgCX`/`imgCY` by delta divided by displayScale
- Mouse wheel: zoom centered on cursor position (multiply zoom by 1.1 or 1/1.1)
- Touch: single-finger drag for pan, two-finger pinch for zoom

### Pencil Tool
- Click/drag to paint individual pixels with the selected palette color (0-3)
- Respects selection boundary (only paints inside marquee)
- Locks transform on first use (prevents accidental repositioning)
- Pushes undo on mousedown

### Selection Tools (Rect, Circle, Lasso)
- Drag to define selection area
- On mousedown: captures `baseAdj` snapshot, starts selection
- On mousemove: updates selection dimensions
- On mouseup: finalizes selection
- Lasso collects points on mousemove, closes polygon on mouseup

### Transform Controls
- Fit: Scale to fit entire image within canvas (letterbox)
- Fill: Scale to fill entire canvas (crop)
- Center: Move image center to canvas center without changing zoom
- Flip H/V: Mirror the image
- Rotation: -180 to 180 degrees via slider or input, plus 90/180 degree preset buttons
- All transforms can be locked via the zoom lock button

---

## Preset System

Presets save adjustment and palette settings (not image data or transform) to `localStorage`:

```javascript
{
  name: string,
  adj: {
    brightness, contrast, saturation, bw, gamma,
    palKey, customColors: [...],
    dither, posterize
  }
}
```

Storage key: `'gbpal_presets'`. JSON stringified array.

---

## Key Constants

```javascript
TILE_SIZE = 8               // Game Boy tile size in pixels
TILE_LIMIT = 197            // Max unique tiles for GB Studio adventure captions
ZOOM_MIN = 0.01             // 1% minimum zoom
ZOOM_MAX = 8.0              // 800% maximum zoom
SNAP_ZONE = 2               // +/-2% zoom snap threshold
MAX_UNDO = 30               // Undo stack depth
DITHER_STRENGTH = 80        // Ordered dither threshold multiplier
MARCHING_ANT_INTERVAL = 80  // ms between ant animation frames
TILE_DEBOUNCE = 90          // ms debounce for tile recount
```

---

## Implementation Notes for Recreation

### Single-File Architecture
The current implementation is a single HTML file with inline CSS and JavaScript. For integration into a larger ecosystem, the natural decomposition is:
- **Pipeline module**: `buildFrame()`, `applyAdj()`, `applyPalette()`, `applyPaint()` - pure functions operating on ImageData
- **Dither module**: All matrix definitions and dithering algorithms
- **Tile module**: `countUniqueTiles()`, `simplifyTiles()`, FNV-1a hashing
- **Selection module**: `inMarquee()`, `pointInPoly()`, selection state management
- **Color module**: `colDist()`, `hexToRgb()`, `rgbToHsl()`, `hslToRgb()`, nearest-color lookup
- **State module**: Central state object, undo/redo, snapshots
- **Canvas/UI module**: Rendering, interaction handlers, display scaling

### No External Dependencies
Everything is vanilla JavaScript. The only external resources are two Google Fonts (Press Start 2P, IBM Plex Mono) used for UI styling.

### Performance Characteristics
- Full pipeline runs every animation frame (~16ms budget at 60fps)
- For 160x144 (23,040 pixels), this is comfortably within budget
- For 320x288 (92,160 pixels), ordered dithering stays fast; Floyd-Steinberg is heavier
- Tile counting is debounced to 90ms to avoid stalling during slider drags
- Error diffusion reuses a pre-allocated Float32Array buffer

### Canvas API Requirements
- `CanvasRenderingContext2D` with `getImageData`/`putImageData`
- `willReadFrequently: true` hint on offscreen canvas context
- `imageSmoothingEnabled = false` for pixel-perfect upscaling
- CSS `image-rendering: pixelated` for display canvas
