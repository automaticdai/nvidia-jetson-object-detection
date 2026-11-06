# Orin Hardware Run Checklist

Closes Task 6 Steps 1–4 of `2026-09-23-trt-detection-demo.md` on the Jetson AGX Orin, and fills the Orin rows of the README benchmark table.

Run everything from the repo root on the Orin. Commands prefixed with `docker/run.sh` run in the pinned Ultralytics image (8.4.191). That image has no GUI and no GStreamer, so inside it use `--headless` and a USB camera or video file. CSI and preview windows are covered in section 6 (native install).

## 0. Setup

- [ ] `git pull` so the Orin has the latest `main`
- [ ] Record the board: `cat /etc/nv_tegra_release` (expect R36 / JetPack 6) and `sudo nvpmodel -q`
- [ ] Max performance: `sudo nvpmodel -m 0 && sudo jetson_clocks`
- [ ] Container starts and versions print:
  ```bash
  docker/run.sh python3 -c "import ultralytics, cv2; print(ultralytics.__version__, cv2.__version__)"
  ```
  Expect `8.4.191`.
- [ ] Put a test video (~720p, ≥30 s) at `models/demo/in.mp4`
- [ ] USB camera attached? `ls /dev/video*`

## 1. Build engines (plan Step 1)

- [ ] FP16: `docker/run.sh python3 src/export.py --model yolo11n --precision fp16` → `models/yolo11n_fp16_640.engine`
- [ ] INT8: `docker/run.sh python3 src/export.py --model yolo11n --precision int8 --calib-data coco128.yaml` → `models/yolo11n_int8_640.engine`
- [ ] Rerun the FP16 command: prints `... already exists; use --force to rebuild it.` and skips the build
- [ ] Optional: INT8 without `--calib-data` prints the coco8 warning and still builds (use `--force`)

If INT8 fails on an unexpected argument, check whether this Ultralytics version takes `int8` or `quantize` and fix `export_kwargs` in `src/export.py`.

## 2. Run each source (plan Step 2)

Let each run go for ~10 s, then Ctrl+C. No tracebacks anywhere.

- [ ] USB, person only: `docker/run.sh python3 src/detect.py --engine models/yolo11n_fp16_640.engine --source usb --classes person --headless`. Prints one FPS + stage-timing line per second.
- [ ] INT8 on video: `docker/run.sh python3 src/detect.py --engine models/yolo11n_int8_640.engine --source models/demo/in.mp4 --headless`
- [ ] Unknown class: `... --classes dog,unicorn` lists the valid names and exits 1
- [ ] Missing engine: `--engine models/missing_fp16_640.engine` prints the export command and exits 1
- [ ] Browser view: add `--stream 8080` to the USB run and open `http://<orin-ip>:8080/`

## 3. Save a video (plan Step 3)

- [ ] `docker/run.sh python3 src/detect.py --engine models/yolo11n_fp16_640.engine --source usb --save models/demo/out.mp4 --headless`, Ctrl+C after ~10 s
- [ ] Frame count > 0:
  ```bash
  docker/run.sh python3 -c "import cv2; c=cv2.VideoCapture('models/demo/out.mp4'); print(int(c.get(cv2.CAP_PROP_FRAME_COUNT)), 'frames')"
  ```
- [ ] Copy it off the board and check it plays

## 4. Measure FPS (plan Step 4)

- [ ] Benchmark both engines on the test video:
  ```bash
  docker/run.sh python3 src/benchmark.py --engines models/yolo11n_fp16_640.engine models/yolo11n_int8_640.engine \
      --source models/demo/in.mp4 --frames 500 --csv models/demo/orin.csv
  ```
- [ ] Check the printed power mode is MAXN and the results look sane (INT8 should be faster than FP16)
- [ ] Live check: USB `--headless` for ~30 s per engine; note steady FPS, with and without `--threaded`

| Engine | Benchmark FPS | Infer (ms) | Live USB FPS | Live `--threaded` FPS |
|---|---|---|---|---|
| yolo11n FP16 640 | | | | |
| yolo11n INT8 640 | | | | |

## 5. Optional: later features on the Orin

Not part of the original plan; worth a quick check while the board is set up.

- [ ] Tracking: `... --source usb --headless --track` (first run needs network for `lap`)
- [ ] Segmentation: export `yolo11n-seg`, run it with `--save`, check the masks in the video
- [ ] Pose: export `yolo11n-pose`, run it with `--save`, check the skeletons

## 6. Optional: CSI camera and windows (native install)

Follow README → *Native install (without Docker)*, then without `docker/run.sh`:

- [ ] `python3 src/camera_test.py`: CSI preview at ~30 FPS
- [ ] `python3 src/detect.py --engine models/yolo11n_fp16_640.engine`: window with boxes and the timing overlay

Engines are tied to the TensorRT version. If the native TensorRT differs from the container's, rebuild with `src/export.py --force`.

## 7. Wrap up

- [ ] Fill the Orin rows of the README table (FPS, infer ms) and note the power mode and JetPack version
- [ ] Tick Task 6 Steps 1–4 in `2026-09-23-trt-detection-demo.md` and update its status note
- [ ] Note any surprises (INT8 argument, calibration, CSI) in the README Troubleshooting section
- [ ] Commit; `models/` is git-ignored, so no engines or videos get committed
