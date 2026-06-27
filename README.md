# Iso Room Editor

An isometric room editor compiled to SWF and run in the browser via [Ruffle](https://ruffle.rs/) (Rust + WebAssembly Flash Player emulator).

## Live demo

Open `index.html` through any static HTTP server — for example:

```
python -m http.server 8000
```

then visit <http://localhost:8000/>.

When hosted via GitHub Pages, the site is reachable at the repo's Pages URL directly.

## Controls

- **Left-click**: place / paint tiles
- **Right-drag** *(or hold space + drag)*: pan the map
- **Mouse wheel**: zoom

## Layout

```
index.html         Ruffle harness
bin/
  NewProject.swf   compiled editor
  manifest.json    asset catalog (auto-generated)
  assets/          floor + wall sprites
fods/              isometric asset library (walls, props, decor)
ruffle/            self-hosted Ruffle build (JS + WASM)
```