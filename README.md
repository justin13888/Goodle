# Goodle ✍️

Doodle on a graph, and get equations back.

Goodle is a small native macOS app. Every stroke you draw snaps to a clean equation, like a line, circle, ellipse, polygon, or y=f(x). Anything that doesn't match a shape becomes a smooth parametric curve. Closed shapes are filled, and strokes that almost close are closed for you. Press **Copy for Desmos** and paste the result into [desmos.com](https://www.desmos.com/calculator).

- **Core:** Rust (`goodle-core`), which turns strokes into equations and is unit-tested on any OS
- **App:** Swift + SwiftUI (macOS 14+), bridged to the core with UniFFI

See [SPEC.md](SPEC.md) for the full product and technical spec.

> Status: spec only. Implementation hasn't started yet.
