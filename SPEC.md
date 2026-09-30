# Goodle — Product & Technical Spec

## Context
Goodle is a small, playful drawing app. You doodle on a graph grid and each stroke becomes an equation Desmos can render. It snaps to clean shapes where it can and falls back to smooth parametric curves where it can't. You press one button to copy everything into desmos.com.

Decisions made with the user:
- **Platform:** a native macOS app only.
- **Desmos hand-off:** Goodle renders its own preview and has a "Copy for Desmos" button. There's no API key and no network. (A Desmos API key is only needed to *embed* the calculator; pasting text into desmos.com needs neither.)
- **Fitting:** "smart snap" to shapes, with a freeform fallback.
- **Fit fidelity:** a fitted curve never strays more than a bounded on-screen distance from the ink (see §3 step 4b). If a shape can't meet the bound, the stroke falls back to a simpler faithful kind.
- **Overlay:** a live dashed preview of the predicted shape while drawing, plus a fading ghost of the raw ink after release (see §2 and §4a).
- **Coercion rules:** auto-close near-loops, fill closed shapes, snap to other strokes and the axes.
- **No TeX/LaTeX runtime.** The clipboard text uses Desmos's LaTeX-flavoured paste syntax, but it is built by plain string formatting and never rendered. Chips typeset equations from **MathML** with the system WebKit and the system **STIX Two Math** font. Nothing is bundled, installed, or downloaded for math.
- **Design approval:** the GUI design in §4a must be approved before implementation starts.

This spec is the source of truth. It is detailed enough to implement in one pass on a Mac.

> **Touch caveat (important):** Macs have no touchscreens. "Works with touch" on macOS means drawing with the **trackpad, mouse, Apple Pencil via Sidecar (an iPad used as a second display), or a Wacom-style tablet**. All of these arrive as ordinary mouse events. A real finger-touch build would be an iPad target later. That target reuses the Rust core unchanged and most of the SwiftUI code (see Future).

---

## 1. Language choices (and why they hold up)
| Layer | Choice | Rationale |
|---|---|---|
| Core engine (stroke → equation) | **Rust** crate `goodle-core` | The user knows Rust. Numeric code is fast and safe. It's platform-agnostic, so it builds and unit-tests on Linux CI and on this machine. The same crate would power a future iPad, Windows, or web build (via WASM). |
| App shell | **Swift + SwiftUI (macOS 14+)**, with one AppKit `NSView` for input | This is the only first-class native macOS UI stack. It gives precise trackpad pinch and scroll, haptics, and access to tablet and Sidecar events. |
| Math display | **Presentation MathML** rendered by the system **WebKit** (native MathML Core since Safari 16.4 / macOS 13.3) | Real typesetting (fractions, superscripts, radicals) with no LaTeX, no third-party package, and no bundled fonts. |
| Bridge | **UniFFI** (proc-macro mode) → generated Swift + an `.xcframework` | Standard, low-ceremony way to call Rust from Swift, with no hand-written C headers. |
| Project gen | **XcodeGen** (`apple/project.yml`) | Keeps a hand-edited `.pbxproj` out of the repo and avoids merge pain. |

## 2. Product spec (minimal and fun)
**One window, one canvas.** There are no documents. The scene autosaves to `~/Library/Application Support/Goodle/scene.json`. The visual design, layout, and states are specified in §4a.

- **Canvas:** a Desmos-like grid and axes. The default view is x ∈ [-10, 10] with y scaled to the window's aspect ratio. Grid steps follow 1/2/5×10ⁿ, and the axes are labeled.
- **Floating pill toolbar** (bottom center):
  - Tools: ✏️ Draw (D), 🧽 Erase (E), ✋ Pan (H).
  - Commands: Undo/Redo, Clear.
  - The big primary button **"Copy for Desmos" (⇧⌘C)**, which shows a "Copied N equations!" toast.
- **Equations panel** (right side, collapsible with ⌘E). It holds one chip per stroke. Each chip has:
  - a color dot (Desmos palette: #c74440, #2d70b3, #388c46, #6042a6, #fa7e19, #000000, cycling);
  - a kind icon;
  - the equation, typeset from the curve's MathML (see §4 `MathRenderer`). It shows exactly the numbers that will be copied. Freeform curves show a "Freeform · 6 curves" summary instead, because their Bézier lists are too long for a chip;
  - a tooltip with the exact Desmos text;
  - a context menu:
    - **Reinterpret as →** Line / Circle / Ellipse / Polygon / Function / Freeform (only the kinds that fit are listed);
    - Toggle fill;
    - Copy this;
    - Delete.
  
  Hovering a chip highlights its curve on the canvas, and clicking a curve highlights its chip.
- **The fun bit (and the fidelity overlay):**
  - **While dragging:**
    - The raw ink is drawn solid.
    - A dashed **preview** of the predicted fitted shape is drawn over it. It's throttled to about 30 Hz and computed off the main thread.
    - A small **magnet dot** marks any endpoint that will snap.
    - A **closing hint** (a short dashed bridge between start and end) appears when auto-close will fire.
    - Together these mean the result on release is never a surprise.
  - **On release:**
    - The fitted curve morphs in (about 180 ms ease-out; instant with Reduce Motion).
    - The raw ink stays as a faint **ghost** that fades out over 1.5 s, so you can compare the two.
  - **"Keep as drawn"** pill: a small pill appears near the stroke's end for 3 s or until the next stroke. It reinterprets the curve as Freeform.
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
   | Freeform | always | Schneider's Bézier fit (Graphics Gems I). It splits at detected corners, and closed curves get C1 continuity at the seam. | always accepted (tolerance **2 pt**, absolute, so the fallback always tracks the ink) |

   **4b. Fidelity gate.** The relative thresholds above allow large deviations on big strokes, and RMS hides local spikes. So every candidate that passed its own test is also checked on screen:
   - Sample its `render_path` from the **rounded, emitted** parameters (after step 7).
   - Compute the symmetric Hausdorff distance, in pt, to the coerced stroke (after steps 2–3).
   - Reject the candidate if that distance is greater than `clamp(4%·diag, 8pt, 20pt)`.
   - Store the distance as `fidelity_pt`.

   Snapping and auto-close are deliberate edits. They are measured against the coerced stroke and are shown live by the overlay (§2).
5. **Classify**
   - The winner is the first accepted kind in simplicity order.
     - Open strokes: Line → Arc → Polyline → Function → Freeform.
     - Closed strokes: Circle → Ellipse → Polygon → Freeform.
   - All accepted candidates are stored as `alternatives`; the chip's "Reinterpret" menu reads from them.
   - With Smart shapes off, the winner is always Freeform.
6. **Emit.** Each template below is first built as a small **math AST** in `core/src/math.rs`:
   ```rust
   enum Math { Num, Ident, Op, Row(Vec<Math>), Frac(a, b), Sup(base, exp), Sub(base, sub),
               Paren(inner), Func(name, args), Restrict(expr, conds), List(Vec<Math>) }
   ```
   Two emitters walk the same tree:
   - `to_desmos()` produces the plain-text paste format;
   - `to_mathml()` produces Presentation MathML (`<math display="block">…</math>`).

   Because both come from one tree, the chip and the clipboard can't disagree. Number formatting (step 7) is applied once, when the AST is built. Each stroke produces:
   - `desmos_text`: one Desmos expression per stroke;
   - `mathml`: the same expression as MathML, used by the chip;
   - a `render_path` sampled from **the emitted equation itself**, so the preview always matches what Desmos draws;
   - an optional `fill_polygon`.
7. **Number formatting**
   - Decimals = clamp(ceil(-log10(pt_to_world)) + 1, 0, 4).
   - Trailing zeros are stripped and `+-` becomes `-`.
   - Scientific notation is never used.

### Desmos expression templates (plain text)
These are strings for the clipboard, written in Desmos's LaTeX-flavoured paste syntax. They are never rendered; the chip renders the MathML form of the same AST. Desmos's default parametric domain is t ∈ [0, 1], and every parametric uses it.
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
- **Export:** all expressions joined with `\n` and placed on `NSPasteboard` as plain text. *Unverified:* that Desmos turns a multi-line paste into one expression per line (see §6.3).

### Scene & undo (in core, so it's testable)
- `Scene { curves: Vec<Curve>, next_color }`.
- Commands (Add, Remove, Reinterpret, SetFill, Clear) sit on undo and redo stacks.
- Serialization uses serde JSON with a `version: 1` field.

### FFI surface (UniFFI)
- `Engine` object with these methods:
  - `new(settings)`;
  - `add_stroke(points: Vec<Pt>, pt_to_world) -> StrokeOutcome`, where `StrokeOutcome` is either `Tap{hit_id?}` or `Curve(CurveView)`;
  - `preview_stroke(points: Vec<Pt>, pt_to_world) -> Option<PreviewView>`. It is pure: it runs steps 1–5 without touching the scene or undo stack;
  - `remove(id)`, `reinterpret(id, kind)`, `set_fill(id, bool)`;
  - `undo()`, `redo()`, `clear()`;
  - `hit_test(pt, radius) -> Option<id>`;
  - `curves() -> Vec<CurveView>`;
  - `export_desmos() -> String`;
  - `to_json()` and `load_json()`;
  - `set_settings()`.
- `CurveView { id, kind, color_idx, desmos_text, mathml, render_path: Vec<Pt>, fill_polygon: Option<Vec<Pt>>, closed, filled, alternatives: Vec<Kind>, segment_count, fidelity_pt }`.
- `PreviewView { kind, render_path: Vec<Pt>, snapped_endpoints: Vec<Pt>, will_close }`.

## 4. Swift app structure
- `GoodleApp.swift`: the app entry point, the menu commands, and the Settings scene.
- `SceneStore.swift`: an `@Observable` wrapper around `Engine`. It mirrors `curves` and handles debounced autosave. It also throttles `preview_stroke` calls: about 30 Hz, on a background queue, and stale results are dropped.
- `Viewport.swift`: world↔screen transform, pan and zoom, and nice grid steps.
- `InkCanvasView.swift`: an `NSView` subclass wrapped in `NSViewRepresentable`. It handles:
  - mouse down, drag, and up (with coalesced points);
  - `magnify(with:)` and `scrollWheel`, plus Space+drag;
  - drawing the grid, the live ink, the dashed preview, magnet dots and closing hint, the curves, fills (40% opacity, matching Desmos's default), the highlight, the ghost ink, and the morph animation, all with Core Graphics.
- `MathRenderer.swift`: typesets chip equations.
  - One shared offscreen `WKWebView` loads a tiny local HTML shell. A `WKContentRuleList` blocks every load, so it never touches the network.
  - The CSS uses `font-family: "STIX Two Math"`, with the text colour taken from the current appearance.
  - It renders each curve's `mathml` and snapshots it with `takeSnapshot` into an `NSImage`.
  - Snapshots are cached by (mathml, appearance, backing scale).
- `EquationPanel.swift` and `EquationChip.swift`: the chip list, the typeset image from `MathRenderer`, the tooltip, and the context menu. The chip's accessibility label reads `desmos_text`.
- `KeepAsDrawnPill.swift`, `Toolbar.swift`, `Toast.swift`, `Palette.swift`, `Settings.swift`.

## 4a. GUI design (requires approval)
**Design status: DRAFT — awaiting approval.** Implementation must not start until the user changes this line to `APPROVED (date)`.

### Layout
```
┌──────────────────────────────────────────────────────────────┬──────────────────────┐
│                          │                                   │ Equations        ⌘E  │
│                          │  y                                ├──────────────────────┤
│          ╭───╮           │                                   │ ● ○  (x−2)²+(y−1)²=9 │
│          │   │ ← fit     │                                   │ ● ╱  y = 0.5x + 1    │
│          ╰───╯ ··· ghost │                                   │      {−3 ≤ x ≤ 4}    │
│  ────────────────────────┼─────────────────────────── x      │ ● ∿  Freeform · 6    │
│                          │                                   │      curves          │
│                          │                                   │                      │
│              ╭──────────────────╮                            │                      │
│              │ Copied 3 eqns!   │  ← toast                   │                      │
│              ╰──────────────────╯                            │                      │
│   ╭─────────────────────────────────────────────────────╮    │                      │
│   │ ✏️ 🧽 ✋ │ ↶ ↷ │ 🗑 │  [ Copy for Desmos  ⇧⌘C ]     │    │                      │
│   ╰─────────────────────────────────────────────────────╯    │                      │
└──────────────────────────────────────────────────────────────┴──────────────────────┘
                         canvas (fills window)                     panel ~280 pt
```
- **Window:** minimum 720 × 480 pt. The panel auto-collapses below 900 pt of window width; ⌘E still toggles it.
- **Toolbar:** floating pill, 16 pt above the bottom edge, centred on the canvas (not the window). It uses a `.regularMaterial` background.
- **Toast:** centred 12 pt above the toolbar. It fades in over 150 ms, stays for 1.5 s, then fades out.
- **Chips:** full panel width, 8 pt padding. The typeset equation wraps to at most 3 lines, then truncates with the tooltip holding the full text.

### Visual tokens
| Token | Value |
|---|---|
| Curve colours | Desmos palette (§2), cycling |
| Ink (while drawing) | 3 pt, curve colour, 100% |
| Preview (while drawing) | 2 pt dashed (6/4), curve colour, 60% |
| Fitted curve | 2.5 pt, curve colour |
| Ghost ink (after release) | 2 pt, curve colour, 25% → 0 over 1.5 s |
| Fill | curve colour, 40% |
| Highlight (hover/selected) | fitted width + 3 pt halo, curve colour, 30% |
| Erase hover | curve drawn red (#c74440), 50% |
| Magnet dot | 6 pt circle, system accent colour |
| Grid (light / dark) | minor #e6e6e6 / #2c2c2e, major #c8c8c8 / #48484a, axes #1c1c1e / #e5e5ea |
| Fonts | SF (UI), SF Mono (tooltip Desmos text), STIX Two Math (typeset equations) |
| Corner radii | toolbar 22 pt (pill), chips 10 pt, toast 12 pt, Keep-as-drawn pill 12 pt |

Everything follows the system light and dark appearance.

### States
| State | What's on screen |
|---|---|
| Empty | Grid + axes, the hint "Doodle something ✍️" centred, Copy button disabled |
| Drawing | Solid ink, dashed preview, magnet dots at snapping endpoints, closing hint when auto-close will fire |
| Just released | Fit morphs in, ghost ink fades over 1.5 s, "Keep as drawn" pill near the stroke end for 3 s |
| Hover chip ↔ curve | Curve gets the highlight halo, and the chip gets a tinted background |
| Selected | Same highlight, which persists; the chip is scrolled into view |
| Erase hover | The curve under the eraser is drawn red at 50% |
| Copied | "Copied N equations!" toast |

### Interaction map
Tools, shortcuts, and gestures are as in §2 (toolbar, navigation, erase, tap vs. stroke, settings).

### Accessibility
- VoiceOver labels on every toolbar button and chip. Chips read their `desmos_text`.
- Reduce Motion: no morph, no ghost fade (the ghost disappears on the next stroke instead), and no toast slide.
- Full keyboard access to the toolbar and the panel. Chip context menus open with ⇧F10 / Control-Return.
- Text contrast ≥ 4.5:1 in both appearances.

### Approval checklist
- [ ] Layout (wireframe, window and panel sizes)
- [ ] Overlay behaviour (preview, magnet, closing hint, ghost, Keep as drawn)
- [ ] Chip content (typeset MathML, freeform summary, tooltip)
- [ ] Visual tokens and states

## 5. Repo layout & build
```
Cargo.toml                 # workspace
core/                      # goodle-core (lib + cdylib/staticlib), src/{preprocess,coerce,fit/*,fidelity,classify,math,templates,scene,ffi}.rs
core/tests/                # golden tests on synthetic strokes
uniffi-bindgen/            # tiny bin for Swift binding generation
scripts/build-apple.sh     # cargo build for aarch64+x86_64-apple-darwin → lipo → xcframework → apple/GoodleCore (local SwiftPM pkg)
apple/project.yml          # XcodeGen; app target depends on local package GoodleCore
apple/Goodle/…             # Swift sources + Assets (doodly app icon)
.github/workflows/ci.yml   # ubuntu: cargo fmt/clippy/test · macos-14: build-apple.sh + xcodegen + xcodebuild
SPEC.md, README.md
```
- There are no third-party Swift dependencies and no TeX anywhere. Math uses the system WebKit and fonts.
- The app bundle should stay under 15 MB.

## 6. Verification (on a Mac, at implementation time)
1. **Core tests** (`cargo test`, which also runs here on Linux). Synthetic strokes with ±1.5% jitter must produce:
   - a circle, ellipse (rotated 30°), triangle, and square → the right kind, with parameters within 3%;
   - a sine-ish open stroke → Function (degree ≤ 5);
   - a spiral or scribble → Freeform;
   - a near-closed circle with a 10 pt gap → closed Circle;
   - an overshooting loop → trimmed;
   - a large, wobbly "circle" that passes the 4% RMS test but fails the fidelity gate → not a Circle (it falls back to a faithful kind).
   
   More checks:
   - snapping works (an endpoint within 12 pt ends up exactly on the target);
   - `render_path` sampled from the emitted parameters stays within tolerance of the input;
   - every accepted fit has `fidelity_pt` ≤ `clamp(4%·diag, 8pt, 20pt)`;
   - `preview_stroke` returns the same kind that `add_stroke` then produces, and it doesn't change the scene.
2. **Golden tests:**
   - fixed shapes → exact expected Desmos text;
   - fixed shapes → exact expected MathML;
   - a property test that `to_desmos` and `to_mathml` carry identical numbers for random ASTs.
3. **Manual:** `scripts/build-apple.sh && cd apple && xcodegen && open Goodle.xcodeproj`, then run the app.
   - Draw one of each shape with the trackpad and with Sidecar + Pencil.
   - While drawing, the dashed preview, magnet dots, and closing hint appear. On release, the ghost fades and "Keep as drawn" turns the curve into Freeform.
   - With Wi-Fi off, chips typeset fractions and exponents correctly in both light and dark mode.
   - Press Copy for Desmos, paste into desmos.com, and confirm every expression appears and matches the preview, including fill opacity.
   - **Check specifically that pasting multiple lines creates one expression per line.** If it doesn't, change the fallback so the button copies the expressions one at a time and the chip "Copy this" becomes the main path.
4. Undo, redo, and reinterpret round-trip. Quit and relaunch, and check that autosave restores the scene.

## 7. Out of scope / future
- iPad target: real finger touch and Pencil (same core, SwiftUI shared).
- Embedded live Desmos. This would need an API key and network access, which conflicts with the current no-network decision, so it needs its own decision.
- Rounding coefficients to "nice" numbers.
- Multiple documents.
- Windows or Linux builds.
