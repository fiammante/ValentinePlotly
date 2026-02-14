# 💗 Beating 3D Heart — Plotly + Jupyter Lab

[![Python](https://img.shields.io/badge/python-3.9%2B-blue.svg)](https://www.python.org/)
[![Plotly](https://img.shields.io/badge/plotly-5.x-informational)](https://plotly.com/python/)
[![NumPy](https://img.shields.io/badge/numpy-2.x-yellow)](https://numpy.org/)
[![License: MIT](https://img.shields.io/badge/license-MIT-green.svg)](LICENSE)

Interactive 3D animation of the implicit heart surface, rendered with **Plotly** inside a single **Jupyter Lab cell**. Beats with a cardiac-realistic systole/diastole envelope. Optionally exports to **MP4** via kaleido + ffmpeg.

$$\left(x^2 + \tfrac{9}{4}y^2 + z^2 - 1\right)^3 - x^2 z^3 - \tfrac{9}{200}y^2 z^3 = 0$$

---

## Features

- **Zero widget dependencies** — uses native Plotly `frames` + `updatemenus` (Play / Pause buttons + scrubber). No `ipywidgets`, no `FigureWidget`, no kernel-browser model sync issues.
- **Cardiac-realistic beat** — asymmetric systole (fast, 18% of cycle) / diastole (slow elastic fill-back with overshoot, 82% of cycle) at configurable BPM.
- **Correct Plotly 5 coloring** — uses `intensity` + `colorscale` (crimson → red → pink). Does **not** use `facecolor`, which is silently ignored by Plotly's WebGL renderer.
- **MP4 export** — set `SAVE_MP4 = True` to render all frames via kaleido and assemble with ffmpeg. The video is embedded inline in the notebook on completion.
- **Single-cell design** — everything in one `.py` file / notebook cell. All parameters in a clearly marked config block at the top.

---

## Preview

> *Run the cell to see the interactive animation. Press ▶ Play to start the heartbeat.*
```
(x²+⁹⁄₄y²+z²−1)³ − x²z³ − ⁹⁄₂₀₀y²z³ = 0
          ♥  beating @ 72 BPM
```

---

## Requirements

| Package | Purpose | Install |
|---|---|---|
| `numpy >= 2.0` | Array math | `pip install numpy` |
| `scikit-image` | Marching cubes | `pip install scikit-image` |
| `plotly >= 5.0` | 3D rendering + animation | `pip install plotly` |
| `kaleido` | Static PNG export (MP4 only) | `pip install kaleido` |
| `ffmpeg` | MP4 assembly (MP4 only) | see below |

**ffmpeg installation:**
```bash
# macOS
brew install ffmpeg

# Ubuntu / Debian
sudo apt install ffmpeg

# conda
conda install -c conda-forge ffmpeg
```

---

## Quick Start
```bash
pip install numpy scikit-image plotly kaleido
```

Then in Jupyter Lab, paste the contents of `heart_jupyter.py` into a cell and run it.

The animation auto-plays. Use the **▶ Play** / **⏸ Pause** buttons and the scrubber slider to control playback.

---

## Configuration

All parameters are in the config block at the top of the file:
```python
OUT_DIR    = "./heart_output"   # MP4 output folder
BPM        = 72                 # heart rate (30–180 typical range)
BEATS      = 3                  # number of beats to animate
FPS        = 24                 # frames per second
MESH_RES   = 80                 # marching-cubes grid (80=fast, 110=smooth)
WIDGET_PX  = 650                # interactive widget size in pixels
SAVE_MP4   = False              # set True to export MP4
MP4_WIDTH  = 700                # MP4 frame width
MP4_HEIGHT = 700                # MP4 frame height
```

| Parameter | Increase effect | Decrease effect |
|---|---|---|
| `BPM` | Faster beat | Slower beat |
| `BEATS` | Longer animation, larger notebook | Shorter |
| `MESH_RES` | Smoother surface, slower build | Faster build, blockier |
| `FPS` | Smoother playback, more frames | Faster build |

---

## How It Works

### 1. Implicit surface → mesh

The heart surface is defined by the implicit equation above. The mesh is extracted using **marching cubes** (`skimage.measure.marching_cubes`) on an `N³` voxel grid, producing a triangle mesh of vertices and face indices.

### 2. Beat envelope

Each frame's scale factor `s(t)` is computed from the cardiac phase:
```
Systole  (phase 0.0 → 0.18):  s = 1.0 − 0.14 · sin(phase/0.18 · π)
                                    fast squeeze:  1.0 → 0.86 → 1.0

Diastole (phase 0.18 → 1.0):  s = 1.0 + exp(−4u) · sin(2πu·0.6) · 0.04
                                    elastic fill-back with tiny overshoot
```

All vertex coordinates are simply multiplied by `s` each frame — no re-meshing needed.

### 3. Coloring

Per-vertex **z-height** drives a `[0, 1]` intensity array mapped through a crimson→red→pink `colorscale`. At systole peak, intensity shifts upward by `0.25 · flush`, sliding all vertices toward the brighter end of the scale (the colour flush visible in real cardiac tissue under pressure).
```python
RED_COLORSCALE = [
    [0.00, "#5a0000"],   # dark crimson
    [0.25, "#990000"],
    [0.55, "#cc0000"],   # vivid red
    [0.80, "#ee1111"],
    [1.00, "#ff6060"],   # bright highlight
]
```

> **Why `intensity` and not `facecolor`?**  
> `facecolor` is silently ignored by Plotly's WebGL renderer (used in Jupyter) in many Plotly 5.x versions, resulting in a black mesh. `intensity` + `colorscale` is the correct and stable API.

### 4. Animation

All `n_frames` Plotly `Frame` objects are built upfront and attached to the figure. The native `updatemenus` Play/Pause buttons and `sliders` scrubber are driven entirely by Plotly.js in the browser — no Python kernel communication after initial render.

### 5. MP4 export

When `SAVE_MP4 = True`, each frame is rendered to a PNG via `fig.write_image()` (kaleido), then ffmpeg assembles the PNG sequence into an H.264 MP4. Temporary frame files are deleted after encoding. The completed video is embedded in the notebook via `IPython.display.Video`.

---

## Known Compatibility Notes

| Issue | Cause | Fix applied |
|---|---|---|
| Black mesh | `facecolor=` ignored by WebGL | Replaced with `intensity` + `colorscale` |
| `titlefont` ValueError | Removed in Plotly 5 | Use `title=dict(font=...)` |
| `ndarray.ptp()` AttributeError | Removed in NumPy 2.0 | Use `.max() - .min()` |
| "model not found" widget error | `ipywidgets`/`FigureWidget` version mismatch | Replaced with native Plotly frames |

---

## File Structure
```
heart_jupyter.py    # single-cell Jupyter script (copy into a notebook cell)
heart_output/       # created on first MP4 export
└── heart_beat.mp4
```

---

## License

MIT
