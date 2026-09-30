# Goodle — Product & Technical Spec

## Context
Goodle is a small, playful drawing app. You doodle on a graph grid and each stroke becomes an equation Desmos can render. It snaps to clean shapes where it can and falls back to smooth parametric curves where it can't. You press one button to copy everything into desmos.com.

Decisions made with the user:
- **Platform:** a native macOS app only.
- **Desmos hand-off:** Goodle renders its own preview and has a "Copy for Desmos" button. There's no API key and no network.
- **Fitting:** "smart snap" to shapes, with a freeform fallback.
- **Coercion rules:** auto-close near-loops, fill closed shapes, snap to other strokes and the axes.

This spec is the source of truth. It is detailed enough to implement in one pass on a Mac.

> **Touch caveat (important):** Macs have no touchscreens. "Works with touch" on macOS means drawing with the **trackpad, mouse, Apple Pencil via Sidecar (an iPad used as a second display), or a Wacom-style tablet**. All of these arrive as ordinary mouse events. A real finger-touch build would be an iPad target later. That target reuses the Rust core unchanged and most of the SwiftUI code (see Future).

---

## 1. Language choices (and why they hold up)
| Layer | Choice | Rationale |
|---|---|---|
| Core engine (stroke → equation) | **Rust** crate `goodle-core` | The user knows Rust. Numeric code is fast and safe. It's platform-agnostic, so it builds and unit-tests on Linux CI and on this machine. The same crate would power a future iPad, Windows, or web build (via WASM). |
| App shell | **Swift + SwiftUI (macOS 14+)**, with one AppKit `NSView` for input | This is the only first-class native macOS UI stack. It gives precise trackpad pinch and scroll, haptics, and access to tablet and Sidecar events. |
| Bridge | **UniFFI** (proc-macro mode) → generated Swift + an `.xcframework` | Standard, low-ceremony way to call Rust from Swift, with no hand-written C headers. |
| Project gen | **XcodeGen** (`apple/project.yml`) | Keeps a hand-edited `.pbxproj` out of the repo and avoids merge pain. |

## 2. Product spec (minimal and fun)
**One window, one canvas.** There are no documents. The scene autosaves to `~/Library/Application Support/Goodle/scene.json`.

- **Canvas:** a Desmos-like grid and axes. The default view is x ∈ [-10, 10] with y scaled to the window's aspect ratio. Grid steps follow 1/2/5×10ⁿ, and the axes are labeled.
- **Floating pill toolbar** (bottom center):
  - Tools: ✏️ Draw (D), 🧽 Erase (E), ✋ Pan (H).
  - Commands: Undo/Redo, Clear.
  - The big primary button **"Copy for Desmos" (⇧⌘C)**, which shows a "Copied N equations!" toast.
- **Equations panel** (right side, collapsible with ⌘E). It holds one chip per stroke. Each chip has:
  - a color dot (Desmos palette: #c74440, #2d70b3, #388c46, #6042a6, #fa7e19, #000000, cycling);
  - a kind icon;
  - the LaTeX rendered with **SwiftMath** (for freeform curves it shows a "Freeform · 6 curves" summary instead);
  - a context menu:
    - **Reinterpret as →** Line / Circle / Ellipse / Polygon / Function / Freeform (only the kinds that fit are listed);
    - Toggle fill;
    - Copy this;
    - Delete.
  
  Hovering a chip highlights its curve on the canvas, and clicking a curve highlights its chip.
- **The fun bit:**
  - The raw ink stays visible while you draw.
  - On release it morphs (about 180 ms ease-out) into the fitted shape.
  - A soft trackpad haptic plays (`NSHapticFeedbackManager`) when the stroke snapped to a primitive.
  - The empty state shows the hint "Doodle something ✍️".
- **Navigation:**
  - Pan: two-finger scroll, Space+drag, or the Pan tool.
  - Zoom: pinch, or ⌘+ / ⌘-.
  - ⌘0 resets the view.
- **Erase:** dragging deletes any curve whose rendered path passes within 10 pt of the pointer.
- **Tap vs. stroke:** a stroke shorter than 6 pt is a tap. In Draw mode a tap selects a curve.
- **Settings** (⌘,):
  - Fill closed shapes (default on);
  - Snap to strokes and axes (default on);
  - Smart shapes (default on; off means everything is freeform).

## 3. Core pipeline (`goodle-core`)
World coordinates use y-up. The Swift side converts screen points to world coordinates before calling the core. It also passes `pt_to_world` (world units per screen point), so every tolerance below is written in **screen points** and behaves the same at any zoom.

1. **Preprocess**
   - Drop duplicate points.
   - Resample to uniform arc length (a spacing of 2 pt).
   - Smooth with a moving average (window 5, endpoints pinned).
   - If the total length is under 6 pt, return `Tap`.
2. **Coerce: snap** (on by default)
   - Each endpoint moves to the nearest target within 12 pt. Targets in priority order:
     1. endpoints of existing open curves and polygon vertices;
     2. the origin;
     3. the x or y axis (projection).
   - The last or first 8% of the stroke is blended toward the snapped point so no kink appears.
3. **Coerce: auto-close**
   - Let `gap` be the distance from the stroke's start to its end.
   - The stroke becomes `closed` when `gap < max(16pt, 0.12·length)` and `length > 3·gap`.
   - **Overshoot trim:** if the tail passes within 16 pt of the start after 75% of the stroke, cut the tail at the closest point.
   - Finally, set the start and end to their midpoint.
4. **Fit candidates.** Each candidate returns its geometry, a normalized error (RMS / bbox diagonal), and `accepted: bool`.

   | Kind | Applies to | Method | Accept when |
   |---|---|---|---|
   | Line segment | open | Total least squares (PCA) | max deviation < 2.5% of length |
   | Circle / arc | closed or open | Kåsa algebraic fit → Gauss-Newton geometric refine | RMS radial error < 4% of r; an arc also needs a sweep > 45° |
   | Ellipse | closed | Fitzgibbon direct least squares | RMS < 4% and axis ratio > 1.15 (otherwise it's a circle) |
   | Polygon / polyline | both | Corners via turning angle > 35° plus Ramer–Douglas–Peucker (tolerance 3% of diagonal). Every edge must pass the line test. | 3–8 vertices if closed, 2–8 if open |
   | Function y=f(x) | open | Stroke must be x-monotonic (backtracking < 1% of width). Least-squares polynomial of degree 1–5 on centered x, with the degree chosen by BIC. | RMS < 1.5% of diagonal |
   | Freeform | always | Schneider's Bézier fit (Graphics Gems I). It splits at detected corners, and closed curves get C1 continuity at the seam. | always accepted (tolerance 1.5% of diagonal) |

5. **Classify**
   - The winner is the first accepted kind in simplicity order.
     - Open strokes: Line → Arc → Polyline → Function → Freeform.
     - Closed strokes: Circle → Ellipse → Polygon → Freeform.
   - All accepted candidates are stored as `alternatives`; the chip's "Reinterpret" menu reads from them.
   - With Smart shapes off, the winner is always Freeform.
6. **Emit.** Each stroke produces:
   - Desmos LaTeX: one expression per stroke;
   - a `render_path` sampled from **the emitted equation itself**, so the preview always matches what Desmos draws;
   - an optional `fill_polygon`.
7. **Number formatting**
   - Decimals = clamp(ceil(-log10(pt_to_world)) + 1, 0, 4).
   - Trailing zeros are stripped and `+-` becomes `-`.
   - Scientific notation is never used.

### LaTeX templates (Desmos's default parametric domain is t ∈ [0, 1]; every parametric uses it)
- **Line:** `y=mx+b\{x_0\le x\le x_1\}`. A near-vertical line (|slope| > 20) becomes `x=c\{y_0\le y\le y_1\}`.
- **Circle:**
  - outline: `(x-h)^2+(y-k)^2=r^2`
  - filled: `\le`
  - arc: `(h+r\cos(a+(b-a)t),k+r\sin(a+(b-a)t))`
- **Ellipse** (implicit rotated form, so fill works): `\frac{((x-h)\cos θ+(y-k)\sin θ)^2}{a^2}+\frac{(-(x-h)\sin θ+(y-k)\cos θ)^2}{b^2}=1`, with `\le 1` when filled. When |θ| < 3° the rotation terms are dropped.
- **Polygon:**
  - closed: `\operatorname{polygon}((x_1,y_1),…)`. Desmos always fills polygons, which matches the default fill setting. When fill is off, use the polyline form below with the loop repeated.
  - open polyline: one parametric expression with inline lists that Desmos broadcasts, `([x_s]+([x_e]-[x_s])t,[y_s]+([y_e]-[y_s])t)`.
- **Function:** `y=a_0+a_1(x-c)+…+a_n(x-c)^n\{x_0\le x\le x_1\}`. It's kept in centered form for numeric stability.
- **Freeform:**
  - outline: one cubic-Bézier parametric expression over inline lists, `((1-t)^3[X_0]+3(1-t)^2t[X_1]+3(1-t)t^2[X_2]+t^3[X_3],\ …y…)`.
  - closed and filled: `\operatorname{polygon}([xs],[ys])` over 96 samples of the Bézier.
- **Export:** all expressions joined with `\n` and placed on `NSPasteboard` as plain text.

### Scene & undo (in core, so it's testable)
- `Scene { curves: Vec<Curve>, next_color }`.
- Commands (Add, Remove, Reinterpret, SetFill, Clear) sit on undo and redo stacks.
- Serialization uses serde JSON with a `version: 1` field.

### FFI surface (UniFFI)
- `Engine` object with these methods:
  - `new(settings)`;
  - `add_stroke(points: Vec<Pt>, pt_to_world) -> StrokeOutcome`, where `StrokeOutcome` is either `Tap{hit_id?}` or `Curve(CurveView)`;
  - `remove(id)`, `reinterpret(id, kind)`, `set_fill(id, bool)`;
  - `undo()`, `redo()`, `clear()`;
  - `hit_test(pt, radius) -> Option<id>`;
  - `curves() -> Vec<CurveView>`;
  - `export_desmos() -> String`;
  - `to_json()` and `load_json()`;
  - `set_settings()`.
- `CurveView { id, kind, color_idx, latex, render_path: Vec<Pt>, fill_polygon: Option<Vec<Pt>>, closed, filled, alternatives: Vec<Kind>, segment_count }`.

## 4. Swift app structure
- `GoodleApp.swift`: the app entry point, the menu commands, and the Settings scene.
- `SceneStore.swift`: an `@Observable` wrapper around `Engine`. It mirrors `curves` and handles debounced autosave.
- `Viewport.swift`: world↔screen transform, pan and zoom, and nice grid steps.
- `InkCanvasView.swift`: an `NSView` subclass wrapped in `NSViewRepresentable`. It handles:
  - mouse down, drag, and up (with coalesced points);
  - `magnify(with:)` and `scrollWheel`, plus Space+drag;
  - drawing the grid, the live ink, the curves, fills (18% opacity), the highlight, and the morph animation, all with Core Graphics.
- `EquationPanel.swift` and `EquationChip.swift`: the chip list, SwiftMath `MTMathUILabel`, and the context menu.
- `Toolbar.swift`, `Toast.swift`, `Palette.swift`, `Settings.swift`.

## 5. Repo layout & build
```
Cargo.toml                 # workspace
core/                      # goodle-core (lib + cdylib/staticlib), src/{preprocess,coerce,fit/*,classify,latex,scene,ffi}.rs
core/tests/                # golden tests on synthetic strokes
uniffi-bindgen/            # tiny bin for Swift binding generation
scripts/build-apple.sh     # cargo build for aarch64+x86_64-apple-darwin → lipo → xcframework → apple/GoodleCore (local SwiftPM pkg)
apple/project.yml          # XcodeGen; app target depends on local package GoodleCore
apple/Goodle/…             # Swift sources + Assets (doodly app icon)
.github/workflows/ci.yml   # ubuntu: cargo fmt/clippy/test · macos-14: build-apple.sh + xcodegen + xcodebuild
SPEC.md, README.md
```

## 6. Verification (on a Mac, at implementation time)
1. **Core tests** (`cargo test`, which also runs here on Linux). Synthetic strokes with ±1.5% jitter must produce:
   - a circle, ellipse (rotated 30°), triangle, and square → the right kind, with parameters within 3%;
   - a sine-ish open stroke → Function (degree ≤ 5);
   - a spiral or scribble → Freeform;
   - a near-closed circle with a 10 pt gap → closed Circle;
   - an overshooting loop → trimmed.
   
   Two more checks: snapping works (an endpoint within 12 pt ends up exactly on the target), and `render_path` sampled from the emitted parameters stays within tolerance of the input.
2. **LaTeX golden tests:** fixed shapes → exact expected strings.
3. **Manual:** `scripts/build-apple.sh && cd apple && xcodegen && open Goodle.xcodeproj`, then run the app.
   - Draw one of each shape with the trackpad and with Sidecar + Pencil.
   - Press Copy for Desmos, paste into desmos.com, and confirm every expression appears and matches the preview.
   - **Check specifically that pasting multiple lines creates one expression per line.** If it doesn't, change the fallback so the button copies the expressions one at a time and the chip "Copy this" becomes the main path.
4. Undo, redo, and reinterpret round-trip. Quit and relaunch, and check that autosave restores the scene.

## 7. Out of scope / future
- iPad target: real finger touch and Pencil (same core, SwiftUI shared).
- Embedded live Desmos.
- Rounding coefficients to "nice" numbers.
- Multiple documents.
- Windows or Linux builds.

