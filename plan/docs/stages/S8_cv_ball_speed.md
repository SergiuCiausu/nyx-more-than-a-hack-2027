# S8 (optional) — Ball speed from video only

## Goal
Measure ball speed purely from the video with computer vision, and validate it against the Metrica ball speed. Independent of S1–S7 except for the homography.

## Pipeline
1. **Decode** frames (decord or PyAV; ffmpeg NVDEC on GPU). Skip `camera_cut` frames.
2. **Detect** the ball per frame:
   - YOLO (Ultralytics) fine-tuned on ball crops, input ≥ 1280 px; or
   - TrackNet-style heatmap model on 3 consecutive frames (better for small/blurred balls).
   - SAHI tiling on wide shots, or crop around the predicted position once tracking.
3. **Track:** Kalman filter with distance gate (`filterpy`) or Norfair; reject false positives (heads, socks, penalty spot); interpolate gaps ≤ 5 frames, leave longer gaps NaN.
4. **Project** pixel (u, v) → pitch metres with the per-frame homography (inverse of pitch→image, as verified in S2).
5. **Smooth then differentiate:** Savitzky–Golay or RTS smoother per continuous run; speed = distance per frame × 25 × 3.6 km/h.

```python
xy[t] = cv2.perspectiveTransform(uv[t].reshape(1,1,2), H_img2pitch[t])[0,0]
xs = savgol_filter(xy[:,0], 9, 2); ys = savgol_filter(xy[:,1], 9, 2)
v = np.hypot(np.diff(xs), np.diff(ys)) * 25 * 3.6
```

## Known difficulties
- Pixel noise: 1 px ≈ 10–20 cm far side → never differentiate raw positions.
- Ball in the air: ground-plane homography inflates position/speed. Either flag aerial segments and report ground-plane speed, or calibrate the full camera (solvePnP / decompose H, SoccerNet calibration tools) and fit a ballistic parabola with gravity to recover 3D speed.
- Motion blur on shots: streak length/direction can estimate speed where detection fails.
- Occlusion: interpolate up to ~0.2 s only.

## Stack
Ultralytics YOLO or TrackNet (PyTorch), SAHI, filterpy/Norfair, OpenCV, scipy, decord/PyAV, CVAT for labelling (YOLO export), SoccerNet + Roboflow football datasets plus frames from this broadcast, TensorRT/ONNX Runtime FP16 for inference. NVIDIA GPU required.

## Validation
Compare CV speed with Metrica `B[:,2]`: median absolute error overall and separately for ground passes, aerial balls (S4 `aerial_flag`), and shots. Detection recall per frame vs Metrica ball presence.

## Acceptance
Report `reports/S8_ball_speed.md` with error tables and example plots of both speed curves.
