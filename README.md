# People_Traffic_Heatmap
# People-Traffic Heatmap (Detection + Time-Bucketed Analytics)

A computer-vision tool that watches a video feed (local file or RTSP camera),
detects people with YOLO, and turns their foot traffic into a browsable
heatmap. Pick any date and 15-minute time window in the desktop app and see
where people spent the most time in that window, rendered over the actual
scene.

> This is a standalone portfolio rebuild of a system I originally built for
> production use. It has been simplified and re-architected from scratch for
> public release — it contains no proprietary code, data, or business logic.

## Features

- **Person detection** on video files or live RTSP streams, via a
  YOLO (Ultralytics) model.
- **Spatial + temporal accumulation**: every detection is binned into a grid
  cell (configurable resolution, default 128×72) and a 15-minute time bucket,
  stored per calendar day.
- **Any time-range query**: aggregate any contiguous set of 15-minute buckets
  into a single heatmap — last 15 minutes, a whole shift, a full day.
- **Desktop GUI** (PyQt5, dark/neon theme):
  - Live video panel with detection boxes + running heatmap overlay.
  - A draggable 24-hour timeline for picking the time range to visualize,
    shaded by how much traffic was recorded in each bucket.
  - One-click PNG export of the generated heatmap.
- **RTSP-resilient**: automatically reconnects on stream drops.
- **Multi-camera ready**: an optional camera ID keeps each source's data in
  its own folder.

## How it works

The whole pipeline is intentionally kept in two files:

| File | Responsibility |
|---|---|
| `heatmap_engine.py` | Config, video capture + YOLO inference, the day/grid/time-bucket data store (`.npy` cubes on disk), and image rendering (normalize → resize → blur → colormap → blend). Framework-agnostic, no GUI dependency. |
| `gui_app.py` | PyQt5 desktop app: source panel, the custom timeline widget, and a background `QThread` that runs the live processing loop without freezing the UI. |

Detection points are stored as counts in a 3D array per day:
`(96 time-buckets, grid_height, grid_width)`. Querying a time range is just
summing a slice of that array along the time axis, then colorizing the
result — which is why both the live overlay and the historical query use the
exact same rendering code path.

The time resolution is fixed at **15 minutes** in this build to keep the data
format simple and predictable for a demo; the engine itself is easy to point
at a different bucket size if you fork it.

## Installation

```bash
git clone https://github.com/<your-username>/people-traffic-heatmap.git
cd people-traffic-heatmap
pip install -r requirements.txt
```

YOLO weights (e.g. `yolov8n.pt`) are downloaded automatically by
`ultralytics` on first run. A CUDA-capable GPU is used automatically if
available; otherwise it falls back to CPU.

## Usage

### Desktop app (recommended)

```bash
python gui_app.py
```

1. **Source panel** — choose a video file (Browse…) or paste an RTSP URL,
   set the frame size to match your source, then **Start Live Processing**.
   The left panel shows the live feed with detections and a running heatmap
   overlay; data is saved to disk periodically and on Stop.
2. **Timeline panel** — pick a date, click **Load Day** to shade the
   24-hour timeline by recorded traffic, drag across the buckets you want,
   then **Generate Heatmap**. Use **Save Image** to export the result as a
   PNG.

### Headless quick test

```bash
python heatmap_engine.py path/to/video.mp4
```

Runs the detection + accumulation loop in a plain OpenCV window (press `q`
to stop) — useful for a quick sanity check without launching the GUI.

## Configuration

All tunables live in the `Config` dataclass at the top of
`heatmap_engine.py`:

| Field | Meaning | Default |
|---|---|---|
| `grid_w`, `grid_h` | Detection grid resolution | 128 × 72 |
| `frame_w`, `frame_h` | Must match your video's resolution | 1920 × 1080 |
| `model_path` | YOLO weights file | `yolov8n.pt` |
| `conf_threshold` | Minimum detection confidence | 0.2 |
| `target_class` | COCO class ID to track | 0 (person) |
| `infer_every_n_frames` | Run inference every Nth frame (perf knob) | 10 |
| `point_mode` | `"center"` or `"bottom_center"` of the box | `center` |
| `save_every_seconds` | How often to flush data to disk | 900 (synced to the 15-min bucket) |

## Project structure

```
people-traffic-heatmap/
├── heatmap_engine.py   # detection, storage cube, rendering — no GUI deps
├── gui_app.py          # PyQt5 dark/neon desktop app
├── requirements.txt
├── LICENSE
└── README.md
```

At runtime the app also creates (git-ignored):

```
heatmap_data/<camera_id>/
├── background.jpg      # first frame of the source, used as the overlay base
└── YYYY-MM-DD.npy       # one detection-count cube per day
```

## Notes

- No sample video is bundled — point the app at any video with people
  walking through frame, or an RTSP camera.
- This project is for portfolio/demonstration purposes.

## License

MIT — see [LICENSE](LICENSE).
