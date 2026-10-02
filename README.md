# EdgeVision — Multi-Camera Real-Time Perception on NVIDIA Jetson

**GStreamer + NVIDIA DeepStream + TensorRT + YOLO + Optional ROS2 Integration**

EdgeVision is an edge-AI perception system designed to study how **multiple concurrent camera/video streams can be processed efficiently on NVIDIA Jetson platforms**.

The project treats perception as an **end-to-end streaming and systems problem**, rather than only a neural-network inference problem. It focuses on camera ingestion, video transport, GPU-compatible memory, batching, TensorRT inference, tracking, output streaming, profiling and multi-stream scaling.

The project is intentionally organized around a **shared perception core with two deployment variants**:

1. **Standalone EdgeVision** — raw GStreamer + DeepStream + TensorRT pipeline for maximum control over video-system performance.
2. **EdgeVision + ROS2** — the same optimized perception core exposed to a robotics stack through ROS2 metadata interfaces.

This separation allows the project to answer two different engineering questions:

> **How efficiently can a Jetson process multiple video streams?**

and

> **How can that optimized perception pipeline be integrated into a ROS2 robot without unnecessarily moving video frames through the middleware?**

---

## 1. Motivation

Modern autonomous robots, ADAS platforms and edge-AI systems often need to process multiple camera streams under strict computational, memory and latency constraints.

A practical perception system must handle:

- USB / CSI / RTSP camera sources
- camera capture
- image format conversion
- video decoding
- GPU-compatible memory
- multi-stream batching
- neural-network inference
- object tracking
- visualization
- video encoding
- streaming
- synchronization
- CPU/GPU memory movement
- downstream robotics interfaces

A naïve implementation can spend significant time moving frames between CPU and GPU, repeatedly converting formats, or passing large image messages through middleware.

EdgeVision investigates how an NVIDIA Jetson can be used as a **multi-stream edge perception computer**, with particular attention to:

- throughput
- end-to-end latency
- GPU utilization
- memory usage
- frame drops
- batch-size effects
- camera-count scaling
- ROS2 integration overhead

---

# 2. Project Goals

The project has two closely related goals.

### Goal A — Edge Video Systems

Build and benchmark a hardware-accelerated multi-camera perception pipeline using:

**GStreamer → DeepStream → TensorRT → Tracker → Output**

The emphasis is on efficient stream handling, batching, memory movement and end-to-end performance.

### Goal B — Robotics Integration

Expose the perception results to ROS2 while keeping the **high-bandwidth video processing path outside ROS2**.

The intended architecture is:

```text
Camera / Video
      ↓
   GStreamer
      ↓
 DeepStream
      ↓
 TensorRT YOLO
      ↓
   Tracker
      ↓
Detection / Tracking Metadata
      ↓
     ROS2
      ↓
Nav2 / Planning / Robotics Stack
```

ROS2 therefore acts primarily as an **integration and communication layer for perception results**, rather than as the main video transport mechanism.

---

# 3. System Architecture

EdgeVision consists of a shared perception core and two deployment variants.

```text
                         EdgeVision
                             │
              ┌──────────────┴──────────────┐
              │                             │
              ▼                             ▼
      Standalone Version              ROS2 Version
              │                             │
      GStreamer / DeepStream         GStreamer / DeepStream
              │                             │
              └──────────────┬──────────────┘
                             ▼
                       TensorRT / YOLO
                             │
                           Tracker
                             │
              ┌──────────────┴──────────────┐
              │                             │
          Display / RTSP              ROS2 Metadata
                                            │
                                            ▼
                                  Nav2 / Robot Stack
```

The perception implementation should remain common between the two variants as much as possible. This makes performance comparisons meaningful and avoids maintaining two unrelated pipelines.

---

# 4. Standalone EdgeVision

The standalone version is the **core systems-performance implementation**.

```text
USB / CSI / RTSP / Video
          │
          ▼
      GStreamer
          │
          ▼
     NVMM / GPU Memory
          │
          ▼
      nvstreammux
          │
          ▼
       nvinfer
          │
          ▼
   TensorRT YOLO Engine
          │
          ▼
      NvTracker
          │
          ▼
       nvdsosd
          │
       ┌──┴──────┐
       ▼         ▼
    Display     RTSP
```

The standalone pipeline provides direct control over:

- source handling
- decoding
- memory movement
- batching
- inference
- tracking
- encoding
- streaming
- profiling

This is the primary environment for the **multi-camera performance experiments**.

---

# 5. ROS2-Integrated EdgeVision

The ROS2 version uses the same perception core but adds a robotics-facing interface.

```text
USB / RTSP / Video
          │
          ▼
      GStreamer
          │
          ▼
     DeepStream
          │
          ▼
 TensorRT YOLO + Tracker
          │
          ▼
 Detection / Tracking Metadata
          │
          ▼
      ROS2 Adapter
          │
     ┌────┴──────────────┐
     ▼                   ▼
 /detections           /tracks
     │                   │
     └─────────┬─────────┘
               ▼
        Robotics Stack
        Nav2 / Planning
```

### Design principle

The project does **not** require camera frames to be continuously republished through ROS2.

Instead:

```text
High-bandwidth path:
Camera → GStreamer → DeepStream → TensorRT

Low-bandwidth robotics interface:
Detection / Tracking Metadata → ROS2
```

This keeps the GPU/video pipeline focused on perception while allowing the resulting information to be consumed by robotic software.

---

# 6. Why Two Versions?

The two versions serve different purposes while sharing the same core.

| Capability | Standalone | ROS2 Integrated |
|---|---:|---:|
| GStreamer | ✓ | ✓ |
| DeepStream | ✓ | ✓ |
| TensorRT | ✓ | ✓ |
| YOLO | ✓ | ✓ |
| Multi-object tracking | ✓ | ✓ |
| Multi-camera batching | ✓ | ✓ |
| GPU/memory profiling | ✓ | ✓ |
| RTSP output | ✓ | ✓ |
| ROS2 interface | — | ✓ |
| Nav2 integration | — | ✓ |
| Robotics metadata | — | ✓ |

This structure allows the project to demonstrate both:

**Edge-AI systems engineering**

and

**Robotics software integration**

without duplicating the complete perception pipeline.

---

# 7. Multi-Camera Architecture

The central engineering problem is processing multiple independent streams efficiently on a resource-constrained edge device.

```text
 Camera 0 ───────┐
                 │
 Camera 1 ───────┤
                 │
 Camera 2 ───────┤
                 │
 RTSP / Video ───┘
                 │
                 ▼
          ┌───────────────┐
          │  GStreamer    │
          │    Sources    │
          └───────┬───────┘
                  │
                  ▼
          ┌───────────────┐
          │  nvstreammux  │
          │    Batching   │
          └───────┬───────┘
                  │
                  ▼
          ┌───────────────┐
          │    nvinfer    │
          │   TensorRT    │
          └───────┬───────┘
                  │
                  ▼
          ┌───────────────┐
          │   nvtracker   │
          └───────┬───────┘
                  │
                  ▼
          ┌───────────────┐
          │    nvdsosd    │
          └───────┬───────┘
                  │
             ┌────┴─────┐
             ▼          ▼
          Display      RTSP
```

The important point is that multiple sources are brought into a common processing pipeline before inference.

Each object should retain source-specific metadata:

```text
source_id
frame_id
object_id
class_id
confidence
bbox
```

This makes it possible to measure and analyze each stream independently.

---

# 8. Camera and Stream Sources

The architecture is designed to be **source-agnostic**.

## 8.1 USB Camera

The primary physical hardware source for development is a USB camera.

Example device:

```bash
/dev/video0
```

Inspect available formats:

```bash
v4l2-ctl --list-formats-ext -d /dev/video0
```

Possible formats include:

```text
YUYV
MJPEG
NV12
```

depending on the camera.

The USB camera provides the real-hardware validation path.

---

## 8.2 CSI Camera

Jetson CSI cameras can be connected using the appropriate NVIDIA camera source.

The important architectural property is that the **source component can change without redesigning the downstream perception stages**.

```text
CSI Source
    ↓
GStreamer
    ↓
DeepStream
    ↓
TensorRT
    ↓
Tracker
```

If a CSI camera is not physically available during development, the project should not claim physical CSI validation.

---

## 8.3 RTSP Camera

The same pipeline can consume RTSP streams:

```text
RTSP
 ↓
H264 / H265 Decoder
 ↓
NVMM
 ↓
DeepStream
```

RTSP represents a distributed/network-camera deployment scenario.

---

## 8.4 Video Dataset as a Camera Emulator

Because the development hardware may contain only one physical USB camera, additional streams can be generated from **public video datasets or recorded video sequences**.

For example:

```text
USB Camera ────────────────┐
                           │
Video 1 ── GStreamer ──────┤
                           │
Video 2 ── GStreamer ──────┤
                           │
Video 3 ── GStreamer ──────┤
                           ▼
                       nvstreammux
```

This allows the system to evaluate:

- 1 stream
- 2 streams
- 4 streams
- 8 streams

without requiring the physical purchase of multiple cameras.

If video is exposed through an RTSP server, it can also be used to emulate network cameras.

**Important:** simulated streams should be explicitly identified as simulated/emulated sources in benchmark results.

---

# 9. Model Pipeline

The perception model follows:

```text
YOLO
  │
  ▼
ONNX
  │
  ▼
TensorRT
  │
  ├── FP32
  └── FP16
  │
  ▼
DeepStream nvinfer
  │
  ▼
Detection Metadata
  │
  ▼
NvTracker
```

The TensorRT engine remains separate from the video-processing pipeline so that different model engines and precision configurations can be evaluated without redesigning the complete application.

---

# 10. GStreamer Pipeline

A simplified conceptual pipeline is:

```text
camera / video / RTSP
        ↓
source
        ↓
decode / format conversion
        ↓
NVMM / GPU-compatible memory
        ↓
nvstreammux
        ↓
nvinfer
        ↓
nvtracker
        ↓
nvdsosd
        ↓
encoder
        ↓
RTSP / display
```

The key design principle is:

> **Keep frames in GPU-compatible memory wherever possible and avoid unnecessary CPU↔GPU transfers.**

This becomes increasingly important as the number of concurrent streams increases.

---

# 11. Object Detection

The project uses a YOLO detector converted to TensorRT.

Example:

```text
best.pt
   ↓
ONNX
   ↓
TensorRT
   ↓
FP16 Engine
   ↓
DeepStream
```

The detector produces metadata including:

- bounding box
- class ID
- confidence
- frame ID
- source ID

This metadata is passed to downstream tracking components through the DeepStream pipeline.

---

# 12. Multi-Object Tracking

Detection produces independent detections for individual frames.

Tracking adds temporal identity:

```text
Frame N

Person A ── ID 12
Person B ── ID 15


Frame N+1

Person A ── ID 12
Person B ── ID 15
```

This enables:

- object counting
- trajectory analysis
- region-of-interest monitoring
- collision analysis
- behavior analysis

The project uses DeepStream's tracking interface so that different tracking backends can be evaluated without changing the complete perception pipeline.

---

# 13. ROS2 Interface

The ROS2 version should expose **perception metadata**, rather than unnecessarily transporting the complete video stream through ROS2.

A conceptual interface is:

```text
DeepStream
    │
    ├── Detection Metadata
    │
    └── Tracking Metadata
            │
            ▼
       ROS2 Adapter
            │
     ┌──────┼───────────┐
     ▼      ▼           ▼
 /detections /tracks  /camera_info
     │      │           │
     └──────┴───────────┘
              │
              ▼
        Robotics Stack
```

Potential ROS2 information includes:

```text
source_id
frame_id
timestamp
object_id
class_id
confidence
bounding_box
```

The exact message definitions should be kept lightweight and focused on downstream robotics requirements.

### ROS2 is an integration layer

The project intentionally separates:

```text
Perception performance
        from
Robotics communication
```

This makes it possible to benchmark the raw pipeline first and then measure any additional overhead introduced by ROS2 integration.

---

# 14. Performance Optimization

The main optimization target is:

```text
Maximum throughput
        +
Minimum latency
        +
Controlled memory usage
        +
Stable inference performance
        +
Predictable multi-stream scaling
```

Important parameters include:

- input resolution
- batch size
- number of streams
- inference interval
- TensorRT precision
- tracker configuration
- decoder configuration
- encoder settings
- source frame rate
- stream synchronization
- buffer configuration

---

# 15. Performance Metrics

The benchmark framework records:

### Throughput

```text
FPS = processed frames / elapsed time
```

### End-to-End Latency

Time between frame arrival and generated perception output.

### Per-Source FPS

```text
Camera 0 → 29.7 FPS
Camera 1 → 29.4 FPS
Camera 2 → 28.9 FPS
```

### GPU Utilization

Measured using NVIDIA system monitoring tools.

### CPU Utilization

Useful for identifying host-side bottlenecks such as:

- decoding
- preprocessing
- application logic
- ROS2 communication

### Memory

Track:

- GPU memory
- system memory
- TensorRT workspace
- buffer usage

### Frame Drops

Measure whether increasing stream count causes:

- dropped frames
- unstable FPS
- queue buildup
- latency growth

### ROS2 Overhead

For the ROS2 variant, compare the standalone and ROS2 configurations for:

- CPU utilization
- latency
- throughput
- message rate
- memory usage

---

# 16. Benchmark Strategy

The project uses **controlled experiments rather than reporting a single FPS number**.

The target deployment device is the NVIDIA Jetson AGX Orin.

## 16.1 Stream Scaling

```text
1 stream
   ↓
2 streams
   ↓
4 streams
   ↓
8 streams
```

The additional streams can be generated from prerecorded/public video sequences when physical cameras are unavailable.

## 16.2 Resolution Scaling

Example:

```text
640 × 384
1280 × 720
1920 × 1080
```

## 16.3 Precision Scaling

```text
FP32
FP16
```

INT8 can be considered as a later optimization stage.

## 16.4 Batch Scaling

```text
Batch 1
Batch 2
Batch 4
Batch 8
```

The batch size should be evaluated together with the number of active streams rather than assumed to be optimal.

---

# 17. Benchmark Matrix

Example benchmark matrix:

| Experiment | Streams | Resolution | Precision | Batch | FPS | Latency | GPU | Memory |
|---|---:|---:|---|---:|---:|---:|---:|---:|
| Baseline | 1 | 1280×720 | FP32 | 1 | TBD | TBD | TBD | TBD |
| FP16 | 1 | 1280×720 | FP16 | 1 | TBD | TBD | TBD | TBD |
| Multi-stream | 2 | 1280×720 | FP16 | 2 | TBD | TBD | TBD | TBD |
| Multi-stream | 4 | 1280×720 | FP16 | 4 | TBD | TBD | TBD | TBD |
| Low-resolution | 4 | 640×384 | FP16 | 4 | TBD | TBD | TBD | TBD |
| Scaling | 8 | 640×384 | FP16 | 8 | TBD | TBD | TBD | TBD |

Actual measurements should be collected on the target Jetson hardware.

---

# 18. Standalone vs ROS2 Benchmark

One of the project's useful experiments is to compare the same perception workload in two configurations.

```text
                    Same perception workload
                              │
                 ┌────────────┴────────────┐
                 │                         │
            Standalone                 ROS2 Integrated
                 │                         │
                 └────────────┬────────────┘
                              ▼
                       Jetson AGX Orin
                              │
                 ┌────────────┼────────────┐
                 ▼            ▼            ▼
                FPS         Latency      Memory
                              │
                              ▼
                       Compare overhead
```

The comparison should be based on measured data rather than assumptions.

The objective is not to declare one architecture universally better. The objective is to understand the **performance/integration trade-off** between a standalone edge pipeline and a ROS2-connected robotics system.

---

# 19. Bottleneck Analysis

The project explicitly separates the pipeline into:

```text
Capture
  ↓
Decode
  ↓
Pre-processing
  ↓
Memory movement
  ↓
Batching
  ↓
Inference
  ↓
Tracking
  ↓
Post-processing
  ↓
ROS2 interface (optional)
  ↓
Encoding
  ↓
Streaming
```

This allows performance problems to be localized instead of assuming that neural-network inference is always the bottleneck.

For example:

```text
Inference:       12 ms
Pre-processing:   4 ms
Tracking:         3 ms
OSD:              2 ms
Encoding:         6 ms
----------------------
Pipeline:        ~27 ms
```

The actual benchmark should replace these illustrative values with measured values.

---

# 20. Memory and Data-Movement Analysis

A major focus of EdgeVision is understanding where data moves.

The preferred path is:

```text
Camera
  ↓
GStreamer
  ↓
NVMM / GPU-compatible buffer
  ↓
DeepStream
  ↓
TensorRT
```

The project should identify unnecessary transitions such as:

```text
GPU
 ↓
CPU
 ↓
GPU
```

and quantify their effect where possible.

This becomes especially important when increasing the number of streams because memory bandwidth and buffer pressure can become limiting factors even when inference itself remains fast.

---

# 21. Failure Analysis

The project also evaluates common real-time perception failures.

### Camera failures

- frame drops
- unsupported formats
- camera disconnects
- exposure changes
- frame-rate instability

### Streaming failures

- RTSP timeout
- decoder errors
- packet loss
- network jitter
- H264/H265 compatibility

### Inference failures

- incorrect TensorRT engine
- unsupported layer
- preprocessing mismatch
- incorrect input dimensions
- confidence threshold problems

### Tracking failures

- ID switches
- missed detections
- occlusion
- fast motion

### ROS2 integration failures

- message publication failures
- timestamp mismatch
- excessive message rate
- queue buildup
- executor/threading bottlenecks
- unnecessary image-copy overhead

---

# 22. Debugging Methodology

When a pipeline fails, debugging follows a layered approach.

```text
Camera / Video
      ↓
Verify raw stream
      ↓
Verify GStreamer
      ↓
Verify decoder
      ↓
Verify NVMM
      ↓
Verify DeepStream
      ↓
Verify TensorRT
      ↓
Verify detection metadata
      ↓
Verify tracker
      ↓
Verify ROS2 adapter (if enabled)
      ↓
Verify output encoder
      ↓
Verify RTSP
```

This prevents the complete system from being debugged simultaneously.

---

# 23. Docker Deployment

The project includes a containerized deployment option.

Conceptually:

```text
Host Jetson
    │
    ├── NVIDIA Driver
    │
    ▼
NVIDIA Container Runtime
    │
    ▼
EdgeVision Container
    │
    ├── GStreamer
    ├── DeepStream
    ├── TensorRT
    ├── OpenCV
    ├── ROS2 (ROS variant)
    └── Application
```

The standalone and ROS2 variants can share the same base container structure where practical.

Containerization makes the software environment reproducible across development and deployment systems.

---

# 24. Example Commands

Check GStreamer:

```bash
gst-launch-1.0 --version
```

Check DeepStream:

```bash
deepstream-app --version
```

Check camera:

```bash
v4l2-ctl --list-devices
```

Run the standalone single-camera pipeline:

```bash
./scripts/run_camera.sh
```

Run an RTSP pipeline:

```bash
./scripts/run_rtsp.sh
```

Run the benchmark:

```bash
./scripts/benchmark.sh
```

Run the ROS2 variant:

```bash
ros2 launch edgevision_ros2 edgevision.launch.py
```

The exact launch and executable names should be updated to match the implementation.

---

# 25. Project Development Stages

## Stage 1 — GStreamer Camera Pipeline

- [x] Camera source
- [x] GStreamer pipeline
- [x] Format conversion
- [ ] FPS measurement
- [ ] Camera failure handling

## Stage 2 — DeepStream Inference

- [ ] DeepStream application
- [ ] TensorRT engine
- [ ] YOLO inference
- [ ] Detection metadata
- [ ] Performance measurement

## Stage 3 — Tracking

- [ ] NvTracker
- [ ] Persistent object IDs
- [ ] ID-switch analysis
- [ ] Tracking performance

## Stage 4 — Multi-Stream Processing

- [ ] Multiple video sources
- [ ] nvstreammux
- [ ] Batched inference
- [ ] Per-source metrics
- [ ] Stream synchronization analysis
- [ ] Dataset/video stream emulation
- [ ] 1 → 2 → 4 → 8 stream scaling

## Stage 5 — Streaming

- [ ] H264/H265 encoding
- [ ] RTSP output
- [ ] Network streaming
- [ ] Latency measurement
- [ ] Stream failure recovery

## Stage 6 — Performance Optimization

- [ ] FP16 TensorRT
- [ ] Batch-size experiments
- [ ] Resolution experiments
- [ ] GPU utilization analysis
- [ ] CPU utilization analysis
- [ ] Memory analysis
- [ ] End-to-end latency analysis
- [ ] Frame-drop analysis

## Stage 7 — ROS2 Integration

- [ ] ROS2 adapter
- [ ] Detection messages
- [ ] Tracking messages
- [ ] Camera/source metadata
- [ ] Timestamp handling
- [ ] Nav2/robotics integration
- [ ] ROS2 overhead benchmark

## Stage 8 — Advanced Optimization

- [ ] TensorRT INT8 calibration
- [ ] CUDA preprocessing
- [ ] Zero-copy optimization
- [ ] Dynamic source management
- [ ] Fault-tolerant stream recovery
- [ ] Multi-camera synchronization

---

# 26. Expected Final Demonstration

## Standalone

```text
USB Camera ───────────┐
                      │
Video 1 ──────────────┤
                      │
Video 2 ──────────────┤
                      ▼
                  GStreamer
                      │
                      ▼
                 DeepStream
                      │
                      ▼
                TensorRT YOLO
                      │
                      ▼
                   Tracker
                      │
                      ▼
                   nvdsosd
                      │
                 ┌────┴────┐
                 ▼         ▼
              Display     RTSP
```

The system should report:

```text
Sources:          4
Inference:        TensorRT FP16
Resolution:       1280 × 720
Detection:        YOLO
Tracking:         Enabled

Source 0 FPS:     XX.X
Source 1 FPS:     XX.X
Source 2 FPS:     XX.X
Source 3 FPS:     XX.X
Pipeline FPS:     XX.X
Latency:          XX ms
GPU utilization:  XX %
GPU memory:       XX MB
Frame drops:      XX
```

## ROS2 Variant

```text
USB / Video Sources
        │
        ▼
    GStreamer
        │
        ▼
    DeepStream
        │
        ▼
 TensorRT YOLO
        │
        ▼
      Tracker
        │
        ▼
   ROS2 Adapter
        │
   ┌────┴─────┐
   ▼          ▼
Detections   Tracks
   │          │
   └────┬─────┘
        ▼
   Nav2 / Robot
```

The ROS2 demonstration should report both perception performance and ROS2 communication characteristics.

---

# 27. What This Project Demonstrates

## Computer Vision

- Object detection
- Multi-object tracking
- Image processing
- Camera pipelines
- Real-time perception

## Camera Systems

- USB cameras
- CSI camera architecture
- RTSP streams
- Camera formats
- Camera-to-pipeline integration
- Frame-rate handling

## Video Systems

- GStreamer
- H264/H265
- RTSP
- Hardware decoding/encoding
- Pipeline debugging
- Multi-stream batching

## NVIDIA Edge AI

- Jetson
- DeepStream
- TensorRT
- CUDA
- GPU memory
- FP16 inference
- Hardware-accelerated video processing

## Robotics

- ROS2
- Perception metadata interfaces
- Timestamp/source handling
- Nav2 integration
- Robotics middleware

## Systems Engineering

- C++
- Python
- Docker
- Configuration-driven pipelines
- Performance benchmarking
- Memory/data-movement analysis
- Failure analysis
- Bottleneck isolation

---

# 28. Why This Project Matters

The important aspect of EdgeVision is that it treats perception as an **end-to-end systems problem** rather than simply an ML inference problem.

A high-performing neural network does not automatically result in a high-performing perception system.

Actual system performance depends on:

```text
Camera
+
Memory
+
Video Pipeline
+
Pre-processing
+
Batching
+
Inference
+
Tracking
+
Post-processing
+
ROS2 interface (optional)
+
Encoding
+
Network
```

The project therefore evaluates both:

**Model-level performance**

and

**End-to-end pipeline performance.**

The ROS2 variant additionally studies how an optimized edge perception system can be connected to a robotics stack while keeping the high-bandwidth video path efficient.

---

# 29. Future Work

Potential extensions include:

- camera synchronization
- hardware-triggered cameras
- cross-camera object association
- camera calibration integration
- temporal sensor fusion
- LiDAR-camera fusion
- TensorRT INT8 calibration
- CUDA preprocessing kernels
- zero-copy optimization
- dynamic source management
- fault-tolerant stream recovery
- ROS2/Nav2 integration
- WebRTC output
- edge-to-cloud telemetry
- distributed multi-Jetson perception
- multi-camera calibration and association

---

# 30. Resume Description

### EdgeVision — Multi-Camera Real-Time Perception

- Built a **GStreamer + NVIDIA DeepStream** perception pipeline for multi-stream edge video analytics on NVIDIA Jetson, integrating TensorRT YOLO inference, NvTracker and hardware-accelerated video processing.
- Designed a **source-agnostic multi-camera architecture** supporting USB, RTSP and video-stream emulation, with configurable batching, resolution and TensorRT precision for Jetson AGX Orin benchmarking.
- Developed a **ROS2 integration layer** that publishes detection/tracking metadata to robotics software while keeping high-bandwidth video processing in the GStreamer/DeepStream pipeline.
- Benchmarked end-to-end FPS, latency, GPU utilization, memory usage and frame drops across stream counts, resolutions, batch sizes and FP16 configurations.

---

# 31. Interview Talking Point

The central engineering lesson from this project is:

> **"I learned to treat real-time computer vision as a complete streaming system rather than just an inference problem. I separate camera ingestion, decoding, memory movement, batching, inference, tracking and output so that I can identify the actual bottleneck. I then integrate the optimized perception pipeline with ROS2 at the metadata level instead of unnecessarily moving high-bandwidth video through the robotics middleware."**

A key design decision is the separation between:

```text
Edge Perception Core
        │
        ├── Standalone deployment
        │
        └── ROS2 integration
```

This makes it possible to evaluate both **raw edge performance** and **robotics-system integration** using the same underlying perception pipeline.

---

# 32. License

MIT License
