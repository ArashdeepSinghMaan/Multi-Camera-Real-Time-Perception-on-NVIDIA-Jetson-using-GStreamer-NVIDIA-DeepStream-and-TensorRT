# EdgeVision — Implementation Guide

> **Implementation README for building, validating, profiling and benchmarking EdgeVision on NVIDIA Jetson AGX Orin.**

EdgeVision is implemented as a **shared hardware-accelerated perception core** with two deployment variants:

```text
                         EdgeVision
                             │
              ┌──────────────┴──────────────┐
              │                             │
              ▼                             ▼
      Standalone Pipeline             ROS2 Integration
              │                             │
              └──────────────┬──────────────┘
                             ▼
                 GStreamer + DeepStream
                             │
                       TensorRT / YOLO
                             │
                          Tracker
                             │
                  Detection / Track Metadata
```

The implementation priority is:

1. **Make one camera work reliably.**
2. **Make TensorRT inference work inside DeepStream.**
3. **Make tracking work.**
4. **Add multiple concurrent streams.**
5. **Measure performance and bottlenecks.**
6. **Add RTSP/video-stream emulation.**
7. **Add ROS2 as a metadata integration layer.**
8. **Benchmark standalone vs ROS2-integrated operation.**

The implementation should not move to the next stage until the current stage has a repeatable validation test.

---

# 1. Implementation Philosophy

The project is not being implemented as two independent applications.

Instead:

```text
                 Shared Core
                     │
        ┌────────────┴────────────┐
        │                         │
   Standalone                 ROS2 Adapter
        │                         │
        ▼                         ▼
 Display / RTSP          ROS2 detections/tracks
```

The **video processing path remains common**.

This is important because it lets us later compare:

```text
Standalone
vs
Standalone + ROS2
```

using approximately the same perception workload.

The core design principle is:

> **Keep high-bandwidth image/video processing inside GStreamer + DeepStream and expose low-bandwidth perception metadata to ROS2.**

---

# 2. Target Hardware

Primary deployment target:

```text
NVIDIA Jetson AGX Orin
```

Primary physical camera:

```text
USB camera
```

Additional streams:

```text
Recorded video
Public video dataset
RTSP-emulated streams
```

The project does **not** require multiple physical cameras.

For example:

```text
USB Camera ───────────────┐
                          │
Video 1 ──────────────────┤
                          │
Video 2 ──────────────────┤
                          ▼
                      nvstreammux
```

The README and benchmark reports must clearly distinguish:

```text
Physical camera
```

from:

```text
Simulated / emulated camera stream
```

---

# 3. Implementation Stages

```text
Stage 0  → Environment validation
Stage 1  → USB camera + GStreamer
Stage 2  → DeepStream pipeline
Stage 3  → TensorRT YOLO
Stage 4  → Tracking
Stage 5  → Multi-stream input
Stage 6  → Video / RTSP emulation
Stage 7  → Profiling + benchmarking
Stage 8  → ROS2 integration
Stage 9  → Standalone vs ROS2 comparison
Stage 10 → Optimization
```

Each stage has:

- implementation task
- validation command/test
- expected result
- completion criterion

---

# 4. Repository Structure

The target repository structure is:

```text
EdgeVision/
│
├── README.md
├── IMPLEMENTATION.md
├── LICENSE
│
├── apps/
│   ├── standalone/
│   │   └── edgevision_app.cpp
│   │
│   └── ros2/
│       └── edgevision_ros2_node.cpp
│
├── src/
│   ├── sources/
│   │   ├── usb_source.cpp
│   │   ├── video_source.cpp
│   │   └── rtsp_source.cpp
│   │
│   ├── pipeline/
│   │   ├── pipeline.cpp
│   │   ├── stream_manager.cpp
│   │   └── pipeline_config.cpp
│   │
│   ├── inference/
│   │   ├── detector.cpp
│   │   └── metadata.cpp
│   │
│   ├── tracking/
│   │   └── tracker_config.cpp
│   │
│   ├── profiling/
│   │   ├── metrics.cpp
│   │   └── latency.cpp
│   │
│   └── output/
│       ├── display.cpp
│       └── rtsp_output.cpp
│
├── ros2/
│   ├── edgevision_msgs/
│   ├── edgevision_ros2/
│   └── launch/
│
├── configs/
│   ├── sources/
│   ├── inference/
│   ├── tracker/
│   └── pipeline/
│
├── models/
│   ├── onnx/
│   └── engines/
│
├── scripts/
│   ├── check_environment.sh
│   ├── check_camera.sh
│   ├── run_usb.sh
│   ├── run_video.sh
│   ├── run_rtsp.sh
│   ├── run_multi_stream.sh
│   └── benchmark.sh
│
├── datasets/
│   └── videos/
│
├── benchmarks/
│   ├── raw/
│   ├── processed/
│   └── reports/
│
├── docker/
│   ├── Dockerfile
│   └── docker-compose.yml
│
└── docs/
    ├── architecture.md
    ├── profiling.md
    ├── ros2_integration.md
    └── troubleshooting.md
```

The exact source-file organization can evolve during implementation. The important separation is:

```text
source
pipeline
inference
tracking
profiling
output
ROS2
```

---

# 5. Stage 0 — Validate the Jetson Environment

Before writing application code, verify the hardware/software stack.

## 5.1 GPU

```bash
nvidia-smi
```

On Jetson, also inspect:

```bash
tegrastats
```

Expected:

- NVIDIA GPU is available.
- Jetson memory statistics are accessible.
- No obvious driver/runtime failure exists.

---

## 5.2 GStreamer

```bash
gst-launch-1.0 --version
```

Check important plugins:

```bash
gst-inspect-1.0 nvstreammux
gst-inspect-1.0 nvinfer
gst-inspect-1.0 nvtracker
gst-inspect-1.0 nvdsosd
```

If a plugin is missing, stop here and fix the environment before continuing.

---

## 5.3 DeepStream

Verify that the installed DeepStream environment is accessible.

For example:

```bash
deepstream-app --version
```

Also inspect the available DeepStream plugins:

```bash
gst-inspect-1.0 | grep -E "nvstreammux|nvinfer|nvtracker|nvdsosd"
```

Do not hard-code a DeepStream version into the implementation until the actual Jetson environment has been checked.

---

## 5.4 TensorRT

Check the installed TensorRT tooling/libraries.

Useful checks include:

```bash
which trtexec
```

and:

```bash
trtexec --help
```

The exact command availability depends on the installed JetPack/TensorRT environment.

---

## 5.5 Camera Tools

Install/verify V4L2 utilities:

```bash
v4l2-ctl --version
```

List devices:

```bash
v4l2-ctl --list-devices
```

---

## 5.6 Environment Validation Script

Create:

```text
scripts/check_environment.sh
```

It should report:

```text
[OK] NVIDIA / Jetson
[OK] GStreamer
[OK] DeepStream
[OK] TensorRT
[OK] V4L2
[OK] USB camera
```

The script should return a non-zero exit code if a required component is unavailable.

---

# 6. Stage 1 — USB Camera Pipeline

The first real implementation target is:

```text
USB Camera
     ↓
GStreamer
     ↓
Display
```

Do not introduce DeepStream yet.

The purpose is to prove:

- camera access
- supported format
- resolution
- frame rate
- GStreamer capture
- basic stability

---

# 7. Inspect the USB Camera

Run:

```bash
v4l2-ctl --list-devices
```

Then:

```bash
v4l2-ctl --list-formats-ext -d /dev/video0
```

Record:

```text
device
formats
resolution
supported FPS
```

Example configuration to investigate:

```text
1280 × 720
30 FPS
MJPEG
```

The actual camera capabilities must be used instead of assuming these values.

---

# 8. First GStreamer Test

Start with the simplest possible pipeline.

For example:

```bash
gst-launch-1.0 \
    v4l2src device=/dev/video0 \
    ! videoconvert \
    ! autovideosink
```

If this works, move toward the camera's native format.

For example, if the camera exposes MJPEG:

```bash
gst-launch-1.0 \
    v4l2src device=/dev/video0 \
    ! image/jpeg,width=1280,height=720,framerate=30/1 \
    ! jpegdec \
    ! videoconvert \
    ! autovideosink
```

The exact caps must be adapted to the camera discovered in Stage 1.

---

# 9. Stage 1 Validation Gate

Stage 1 is complete only when:

- [ ] USB camera is detected.
- [ ] Supported format is known.
- [ ] GStreamer can capture continuously.
- [ ] Target resolution is verified.
- [ ] Target FPS is verified.
- [ ] No continuous frame-drop/error messages occur.
- [ ] Camera disconnect/reconnect behavior is understood.

Record the result in:

```text
benchmarks/raw/stage1_camera.txt
```

---

# 10. Stage 2 — Build the DeepStream Pipeline

Once camera capture is reliable, introduce DeepStream.

Target:

```text
USB Camera
     ↓
GStreamer
     ↓
NVMM
     ↓
DeepStream
     ↓
Display
```

At this stage:

**No custom YOLO integration is required yet.**

The goal is to prove that the DeepStream application can receive and process the stream.

---

# 11. DeepStream Application Architecture

The application should eventually contain:

```text
main()
  │
  ├── load configuration
  │
  ├── initialize GStreamer
  │
  ├── create source
  │
  ├── create streammux
  │
  ├── create inference
  │
  ├── create tracker
  │
  ├── create OSD/output
  │
  ├── link elements
  │
  ├── attach callbacks/probes
  │
  └── run main loop
```

Keep configuration separate from application logic.

---

# 12. Configuration-Driven Pipeline

Do not hard-code everything inside C++.

Example:

```text
configs/
├── sources/
│   └── usb.yaml
│
├── inference/
│   └── yolovX_fp16.yaml
│
├── tracker/
│   └── default.yaml
│
└── pipeline/
    └── single_camera.yaml
```

Configuration should eventually control:

```text
source
resolution
FPS
batch size
inference interval
TensorRT engine
precision
tracker
display
RTSP output
```

This is essential for benchmarking.

---

# 13. Stage 3 — TensorRT YOLO

After the DeepStream pipeline works, integrate the YOLO TensorRT engine.

Target:

```text
USB Camera
     ↓
GStreamer
     ↓
nvstreammux
     ↓
nvinfer
     ↓
TensorRT YOLO
     ↓
Detection Metadata
```

---

# 14. Model Preparation

The intended model flow is:

```text
YOLO .pt
   ↓
ONNX
   ↓
TensorRT
   ↓
FP16 Engine
   ↓
DeepStream nvinfer
```

The model engine should be stored separately:

```text
models/
├── onnx/
│   └── model.onnx
│
└── engines/
    └── model_fp16.engine
```

Do not commit large generated engines to Git unless the repository strategy explicitly requires it.

---

# 15. Validate TensorRT Independently

Before debugging TensorRT through DeepStream, validate the engine independently.

Use the available TensorRT tooling, for example:

```bash
trtexec --loadEngine=models/engines/model_fp16.engine
```

Record:

```text
engine
precision
input shape
inference latency
throughput
workspace
```

This gives us a baseline before adding video-processing overhead.

---

# 16. DeepStream Detection Metadata

Once inference works, verify that detections are actually reaching DeepStream metadata.

Required fields:

```text
source_id
frame_id
class_id
confidence
bbox
```

Create a metadata probe/debug output before adding tracking.

Example conceptual flow:

```text
Frame
  ↓
nvinfer
  ↓
metadata probe
  ↓
print detection count
```

The first target is not a polished visualization.

The first target is:

```text
Frame 100 → 4 detections
Frame 101 → 5 detections
Frame 102 → 5 detections
```

---

# 17. Stage 3 Validation Gate

Do not proceed until:

- [ ] TensorRT engine loads independently.
- [ ] DeepStream loads the engine.
- [ ] Frames reach nvinfer.
- [ ] Detections are produced.
- [ ] Detection metadata contains source/frame information.
- [ ] Bounding boxes are correct.
- [ ] Confidence values are reasonable.
- [ ] Inference errors are absent.

---

# 18. Stage 4 — Add Tracking

Add:

```text
nvinfer
   ↓
nvtracker
```

Target:

```text
Person → ID 12
Person → ID 15
```

Verify persistence over consecutive frames.

---

# 19. Tracking Validation

Create a test sequence containing:

- stationary object
- moving object
- object entering/leaving frame
- partial occlusion

Record:

```text
detection count
track count
ID changes
lost tracks
```

The objective is not initially to optimize tracking accuracy. The first objective is to verify that the tracker is correctly integrated into the pipeline.

---

# 20. Stage 5 — Multi-Stream Architecture

Only after one stream is stable should we add multiple streams.

Target:

```text
Source 0 ──┐
           │
Source 1 ──┼──→ nvstreammux
           │
Source 2 ──┤
           │
Source 3 ──┘
                ↓
             nvinfer
                ↓
             tracker
```

The first multi-stream test should use prerecorded video rather than immediately attempting several live camera devices.

---

# 21. Why Video Files First?

A video file provides:

- deterministic input
- repeatable experiments
- no camera hardware dependency
- easier FPS comparison
- easier regression testing

For example:

```text
video0.mp4
video1.mp4
video2.mp4
video3.mp4
```

can emulate four camera sources.

---

# 22. Multi-Stream Source Abstraction

The source layer should eventually support:

```text
USB
Video
RTSP
```

through a common interface.

Conceptually:

```text
SourceManager
    │
    ├── USBSource
    ├── VideoSource
    └── RTSPSource
```

Downstream:

```text
SourceManager
      ↓
GStreamer
      ↓
nvstreammux
      ↓
DeepStream
```

The inference/tracking stages should not need to know whether the source is physical or simulated.

---

# 23. Stream Scaling Experiments

Run controlled tests:

```text
1 stream
2 streams
4 streams
8 streams
```

For each configuration record:

```text
source count
input resolution
input FPS
batch size
pipeline FPS
per-source FPS
latency
GPU utilization
CPU utilization
GPU memory
system memory
frame drops
```

---

# 24. Stage 6 — RTSP Emulation

Once multiple local video streams work, optionally expose prerecorded streams as RTSP sources.

Architecture:

```text
video0.mp4 ──┐
             │
video1.mp4 ──┼── RTSP server
             │
video2.mp4 ──┘
                  ↓
             EdgeVision
```

This lets us test the network-camera path without buying IP cameras.

The benchmark must distinguish:

```text
Local video source
```

from:

```text
RTSP source
```

because network transport introduces additional variables.

---

# 25. Stage 7 — Profiling

The project is fundamentally a performance-analysis project.

We need measurements at several levels.

## System level

Use Jetson system monitoring such as:

```bash
tegrastats
```

Capture:

```text
GPU utilization
CPU utilization
memory
temperature
power
```

as available on the target system.

---

# 26. Application-Level Metrics

The EdgeVision application should record:

```text
source_id
frame_id
capture timestamp
processing timestamp
detection timestamp
tracking timestamp
output timestamp
```

From these we can derive latency.

---

# 27. Per-Stage Timing

The implementation should eventually expose:

```text
capture
decode
preprocess
batch
inference
tracking
postprocess
ROS2 publish
encode
output
```

Example report:

```text
Capture:       3.1 ms
Decode:        2.4 ms
Preprocess:    1.8 ms
Batch:         0.7 ms
Inference:    12.2 ms
Tracking:      2.9 ms
Postprocess:   1.1 ms
Output:        4.2 ms
--------------------------------
End-to-end:   28.4 ms
```

These are example values only.

---

# 28. Avoid Misleading FPS Measurements

Do not report:

```text
TensorRT = 60 FPS
```

as the final system performance if the complete pipeline only produces:

```text
25 FPS
```

Always distinguish:

```text
Model throughput
```

from:

```text
Pipeline throughput
```

and:

```text
Per-source throughput
```

---

# 29. Stage 8 — ROS2 Integration

ROS2 is intentionally implemented **after the standalone pipeline is stable**.

Target:

```text
GStreamer
   ↓
DeepStream
   ↓
TensorRT
   ↓
Tracker
   ↓
Metadata Adapter
   ↓
ROS2
```

Do not redesign the perception core when adding ROS2.

---

# 30. ROS2 Package Structure

Create:

```text
ros2/
├── edgevision_msgs/
│   ├── msg/
│   │   ├── Detection.msg
│   │   ├── DetectionArray.msg
│   │   ├── Track.msg
│   │   └── TrackArray.msg
│   │
│   └── package.xml
│
├── edgevision_ros2/
│   ├── src/
│   │   └── edgevision_bridge.cpp
│   ├── launch/
│   └── package.xml
│
└── launch/
    └── edgevision.launch.py
```

The message design should remain lightweight.

---

# 31. ROS2 Metadata Design

A detection should conceptually contain:

```text
timestamp
source_id
frame_id
class_id
confidence
bbox
```

A track should conceptually contain:

```text
timestamp
source_id
frame_id
object_id
class_id
confidence
bbox
```

Do not add large image payloads to these messages unless a later experiment specifically requires them.

---

# 32. ROS2 Topic Architecture

Initial interface:

```text
/edgevision/detections
/edgevision/tracks
```

Optional:

```text
/edgevision/camera_info
/edgevision/status
/edgevision/performance
```

The first implementation should keep the interface small.

---

# 33. ROS2 Integration Boundary

The intended architecture is:

```text
              HIGH BANDWIDTH
                   │
Camera ─→ GStreamer ─→ DeepStream ─→ TensorRT
                                      │
                                      ▼
                               Detection Metadata
                                      │
              LOW BANDWIDTH            │
                                      ▼
                                    ROS2
                                      │
                                      ▼
                              Nav2 / Planning
```

This boundary is one of the key architectural decisions of the project.

---

# 34. Stage 9 — Standalone vs ROS2 Benchmark

Once ROS2 integration works, run the same workload in two modes.

### Mode A

```text
Standalone
```

### Mode B

```text
Standalone + ROS2 metadata publishing
```

Use identical:

```text
camera/video
resolution
model
precision
batch size
stream count
tracker
```

Compare:

```text
FPS
latency
CPU
GPU
memory
frame drops
```

---

# 35. ROS2 Benchmark Question

The experiment should answer:

> **What measurable overhead does ROS2 metadata integration introduce when the high-bandwidth video pipeline remains outside ROS2?**

This is a measured engineering question.

Do not assume the result before collecting data.

---

# 36. Stage 10 — Optimization

Only after the complete pipeline is measurable should optimization begin.

Optimization areas:

### Input

- reduce unnecessary resolution
- control source FPS
- select efficient formats

### Memory

- minimize CPU↔GPU copies
- maintain GPU-compatible buffers
- avoid unnecessary conversions

### Batching

- tune batch size
- measure batch latency
- measure GPU utilization

### TensorRT

- FP16
- later INT8
- engine optimization

### Preprocessing

- hardware/GPU preprocessing
- CUDA kernels where justified

### Tracking

- tracker configuration
- processing interval

### Output

- OSD cost
- encoding cost
- streaming cost

### ROS2

- message rate
- queue depth
- executor behavior
- callback frequency

---

# 37. Benchmark Matrix

The first complete benchmark matrix should be:

| Test | Streams | Resolution | Precision | Batch | Source | ROS2 | FPS | Latency | GPU | CPU | GPU Mem | Drops |
|---|---:|---|---|---:|---|---|---:|---:|---:|---:|---:|---:|
| B01 | 1 | 1280×720 | FP32 | 1 | USB | No | TBD | TBD | TBD | TBD | TBD | TBD |
| B02 | 1 | 1280×720 | FP16 | 1 | USB | No | TBD | TBD | TBD | TBD | TBD | TBD |
| B03 | 2 | 1280×720 | FP16 | 2 | Video | No | TBD | TBD | TBD | TBD | TBD | TBD |
| B04 | 4 | 1280×720 | FP16 | 4 | Video | No | TBD | TBD | TBD | TBD | TBD | TBD |
| B05 | 4 | 640×384 | FP16 | 4 | Video | No | TBD | TBD | TBD | TBD | TBD | TBD |
| B06 | 4 | 1280×720 | FP16 | 4 | RTSP | No | TBD | TBD | TBD | TBD | TBD | TBD |
| B07 | 4 | 1280×720 | FP16 | 4 | Video | Yes | TBD | TBD | TBD | TBD | TBD | TBD |

Additional experiments can be added after the baseline is stable.

---

# 38. Benchmark Rules

For every benchmark:

1. Use the same model.
2. Use the same source sequence where possible.
3. Warm up the pipeline before measurement.
4. Record the measurement duration.
5. Record source FPS.
6. Record stream count.
7. Record batch size.
8. Record resolution.
9. Record precision.
10. Record GPU/CPU/memory statistics.
11. Record frame drops.
12. Store raw results before calculating summaries.

Avoid reporting only the best run.

---

# 39. Benchmark Data Layout

Store raw measurements:

```text
benchmarks/
├── raw/
│   ├── B01/
│   ├── B02/
│   ├── B03/
│   └── ...
│
├── processed/
│   ├── benchmark_summary.csv
│   └── latency_summary.csv
│
└── reports/
    ├── stream_scaling.md
    ├── resolution_scaling.md
    ├── precision_comparison.md
    └── ros2_overhead.md
```

Raw data should remain available so that conclusions can be reproduced.

---

# 40. Performance Dashboard

Eventually generate a summary such as:

```text
Streams     FPS/source     Latency     GPU     Memory
-----------------------------------------------------
1           XX.X           XX ms       XX %    XX MB
2           XX.X           XX ms       XX %    XX MB
4           XX.X           XX ms       XX %    XX MB
8           XX.X           XX ms       XX %    XX MB
```

Useful plots:

- stream count vs total FPS
- stream count vs per-source FPS
- stream count vs latency
- stream count vs GPU utilization
- stream count vs memory
- resolution vs latency
- FP32 vs FP16
- standalone vs ROS2 overhead

---

# 41. Debugging Order

When something fails, debug from the bottom of the stack upward.

```text
1. Hardware
      ↓
2. Camera / video source
      ↓
3. GStreamer
      ↓
4. Decoder
      ↓
5. NVMM / GPU memory
      ↓
6. DeepStream
      ↓
7. TensorRT
      ↓
8. Detection metadata
      ↓
9. Tracker
      ↓
10. Output
      ↓
11. ROS2
```

Do not debug:

```text
camera + DeepStream + TensorRT + tracker + ROS2
```

simultaneously.

---

# 42. Troubleshooting Checklist

## Camera does not appear

```bash
v4l2-ctl --list-devices
```

Then:

```bash
ls -l /dev/video*
```

Check permissions and physical connection.

---

## Camera appears but GStreamer fails

Inspect:

```bash
v4l2-ctl --list-formats-ext -d /dev/video0
```

Then match GStreamer caps to the actual camera.

---

## DeepStream cannot start

Check:

```bash
gst-inspect-1.0 nvstreammux
gst-inspect-1.0 nvinfer
gst-inspect-1.0 nvtracker
```

---

## TensorRT engine fails

Validate the engine independently before debugging it through DeepStream.

---

## No detections

Check in this order:

```text
Input resolution
      ↓
Model input shape
      ↓
Preprocessing
      ↓
TensorRT engine
      ↓
nvinfer configuration
      ↓
Detection metadata
```

---

## FPS is low

Do not immediately blame inference.

Check:

```text
Capture
Decode
Preprocess
Memory movement
Batching
Inference
Tracking
OSD
Encoding
ROS2
```

---

## Multi-stream FPS collapses

Check:

```text
batch size
buffer queues
GPU utilization
memory bandwidth
CPU utilization
decoder load
frame drops
```

---

## ROS2 slows the pipeline

First determine whether the slowdown comes from:

```text
message creation
serialization
copying
publication rate
executor
callback processing
```

The camera frames should not be routed through ROS2 merely to solve the integration problem.

---

# 43. Implementation Milestones

## Milestone 1 — Camera

```text
USB → GStreamer → Display
```

Status:

```text
[ ]
```

---

## Milestone 2 — DeepStream

```text
USB → GStreamer → DeepStream → Display
```

Status:

```text
[ ]
```

---

## Milestone 3 — Detection

```text
USB → DeepStream → TensorRT YOLO → Detection
```

Status:

```text
[ ]
```

---

## Milestone 4 — Tracking

```text
Detection → NvTracker → Persistent IDs
```

Status:

```text
[ ]
```

---

## Milestone 5 — Multi-Stream

```text
Video 0 ─┐
Video 1 ─┼→ nvstreammux → inference → tracker
Video 2 ─┤
Video 3 ─┘
```

Status:

```text
[ ]
```

---

## Milestone 6 — Profiling

```text
FPS
Latency
GPU
CPU
Memory
Drops
```

Status:

```text
[ ]
```

---

## Milestone 7 — ROS2

```text
Detection / Tracking
        ↓
      ROS2
        ↓
   Robot Stack
```

Status:

```text
[ ]
```

---

## Milestone 8 — Benchmark

```text
Standalone
     vs
ROS2
```

Status:

```text
[ ]
```

---

# 44. Definition of Done

The project is considered functionally complete when all of the following work.

## Core

- [ ] USB camera input
- [ ] Video-file input
- [ ] RTSP input
- [ ] GStreamer source abstraction
- [ ] DeepStream pipeline
- [ ] TensorRT YOLO
- [ ] Detection metadata
- [ ] NvTracker
- [ ] Multi-stream batching
- [ ] Display output
- [ ] RTSP output

## Performance

- [ ] Per-source FPS
- [ ] Pipeline FPS
- [ ] End-to-end latency
- [ ] GPU utilization
- [ ] CPU utilization
- [ ] GPU memory
- [ ] System memory
- [ ] Frame-drop measurement
- [ ] Stream-scaling benchmark
- [ ] Resolution benchmark
- [ ] Precision benchmark

## ROS2

- [ ] Detection message
- [ ] Tracking message
- [ ] Source/frame metadata
- [ ] Timestamp handling
- [ ] ROS2 launch
- [ ] ROS2 performance measurement
- [ ] Standalone vs ROS2 comparison

## Engineering

- [ ] Configuration-driven pipeline
- [ ] Reproducible benchmark scripts
- [ ] Docker environment
- [ ] Troubleshooting documentation
- [ ] Benchmark data stored
- [ ] Final architecture documented

---

# 45. First Implementation Session

The first session should **not** attempt to build the complete project.

Start with:

```text
Step 1
Check Jetson environment

        ↓

Step 2
Identify USB camera

        ↓

Step 3
Inspect camera formats

        ↓

Step 4
Run minimal GStreamer pipeline

        ↓

Step 5
Measure stable FPS

        ↓

Step 6
Create repository skeleton

        ↓

Step 7
Commit Stage 1
```

The first successful result should simply be:

```text
USB Camera
     ↓
GStreamer
     ↓
Display
```

Once that is stable, we move to DeepStream.

---

# 46. Initial Git Workflow

Create the repository:

```bash
mkdir -p EdgeVision
cd EdgeVision

git init
```

Create the initial structure:

```bash
mkdir -p \
    apps/standalone \
    apps/ros2 \
    src/sources \
    src/pipeline \
    src/inference \
    src/tracking \
    src/profiling \
    src/output \
    ros2 \
    configs/sources \
    configs/inference \
    configs/tracker \
    configs/pipeline \
    models/onnx \
    models/engines \
    scripts \
    datasets/videos \
    benchmarks/raw \
    benchmarks/processed \
    benchmarks/reports \
    docker \
    docs
```

Create:

```text
README.md
IMPLEMENTATION.md
.gitignore
LICENSE
```

Then commit the skeleton:

```bash
git add .
git commit -m "Initialize EdgeVision implementation structure"
```

---

# 47. Recommended Commit Sequence

Keep implementation history understandable.

Suggested commits:

```text
01  Initialize repository structure
02  Add USB camera GStreamer pipeline
03  Add camera diagnostics
04  Add DeepStream application skeleton
05  Integrate TensorRT YOLO
06  Add detection metadata
07  Add NvTracker
08  Add video-file source
09  Add multi-stream batching
10  Add profiling metrics
11  Add RTSP input/output
12  Add benchmark framework
13  Add ROS2 messages
14  Add ROS2 bridge
15  Add ROS2 benchmark
16  Add Docker deployment
17  Add optimization experiments
```

This also makes the project easier to explain during interviews.

---

# 48. Implementation Principle

At every stage, ask:

```text
Does it work?
      ↓
Can I measure it?
      ↓
Can I reproduce it?
      ↓
Can I isolate its bottleneck?
      ↓
Can I scale it?
```

Only after these questions are answered should the next layer be added.

---

# 49. Final Implementation Architecture

The intended completed architecture is:

```text
                         ┌─────────────────────┐
                         │   Physical USB      │
                         │      Camera         │
                         └──────────┬──────────┘
                                    │
                    ┌───────────────┼────────────────┐
                    │               │                │
                    ▼               ▼                ▼
                Video Files       RTSP           Future CSI
                    │               │                │
                    └───────────────┴────────────────┘
                                    │
                                    ▼
                              GStreamer
                                    │
                              NVMM / GPU
                                    │
                                    ▼
                              nvstreammux
                                    │
                                    ▼
                                nvinfer
                                    │
                                    ▼
                           TensorRT YOLO
                                    │
                                    ▼
                                Tracker
                                    │
                           Detection Metadata
                                    │
                     ┌──────────────┴──────────────┐
                     │                             │
                     ▼                             ▼
                Standalone                    ROS2 Adapter
                     │                             │
                Display / RTSP             /detections
                                                   │
                                                   ▼
                                                 /tracks
                                                   │
                                                   ▼
                                             Nav2 / Robot
```

The central implementation objective is:

> **Build the fastest, most measurable and source-agnostic perception core first; integrate it with ROS2 only after its standalone behavior is understood.**

---

# 50. Current Status

```text
Project architecture       [DONE]
Implementation plan         [DONE]

Environment validation      [ ]
USB camera pipeline         [ ]
DeepStream pipeline         [ ]
TensorRT YOLO               [ ]
Tracking                    [ ]
Multi-stream                [ ]
Video emulation             [ ]
RTSP                        [ ]
Profiling                   [ ]
Benchmarking                [ ]
ROS2 integration            [ ]
ROS2 benchmark              [ ]
Optimization                [ ]
Docker                      [ ]
```

**Next implementation target: Stage 0 → Stage 1**

```text
Jetson AGX Orin
      ↓
Environment validation
      ↓
USB camera discovery
      ↓
Camera capability inspection
      ↓
Minimal GStreamer pipeline
      ↓
Stable camera FPS measurement
```
