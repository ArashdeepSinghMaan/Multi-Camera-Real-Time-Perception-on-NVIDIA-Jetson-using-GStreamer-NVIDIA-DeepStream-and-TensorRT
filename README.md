# edgevision-deepstream# EdgeVision

### Multi-Camera Real-Time Perception on NVIDIA Jetson using GStreamer, NVIDIA DeepStream and TensorRT

EdgeVision is an edge-AI perception pipeline designed for **real-time multi-camera video analytics on NVIDIA Jetson platforms**.

The project integrates **GStreamer, NVIDIA DeepStream, TensorRT and YOLO-based object detection** into a hardware-accelerated video pipeline with multi-object tracking, batching, performance monitoring and optional RTSP output.

The primary objective is to study and optimize the complete path from **camera input to real-time perception output**, with particular focus on latency, throughput, memory movement and GPU utilization.

---

## 1. Motivation

Modern autonomous robots and edge-AI systems need to process multiple camera streams under strict computational and latency constraints.

A typical perception system must handle:

* USB/CSI cameras
* camera capture
* image format conversion
* video decoding
* batching
* neural-network inference
* object tracking
* visualization
* video encoding
* streaming
* synchronization
* GPU/CPU memory transfers

A naïve implementation can spend significant time moving frames between CPU and GPU or repeatedly converting image formats.

This project investigates how a hardware-accelerated multimedia pipeline can be constructed using NVIDIA's edge-AI stack.

---

## 2. Objectives

The project has five primary objectives:

1. Build a real-time camera pipeline using **GStreamer**.
2. Integrate object detection using **NVIDIA DeepStream + TensorRT**.
3. Add **multi-object tracking** to the detection pipeline.
4. Extend the pipeline to support multiple video sources.
5. Measure and analyze **FPS, latency, GPU utilization and pipeline bottlenecks**.

---

## 3. System Architecture

### Single-camera pipeline

```text
              ┌─────────────────┐
              │ USB / CSI Camera │
              └────────┬────────┘
                       │
                       ▼
              ┌─────────────────┐
              │    GStreamer    │
              │ Camera Capture  │
              └────────┬────────┘
                       │
                       ▼
              ┌─────────────────┐
              │  NVStreamMux    │
              │   Batching      │
              └────────┬────────┘
                       │
                       ▼
              ┌─────────────────┐
              │   DeepStream    │
              │   nvinfer       │
              └────────┬────────┘
                       │
                       ▼
              ┌─────────────────┐
              │    TensorRT     │
              │   YOLO Engine   │
              └────────┬────────┘
                       │
                       ▼
              ┌─────────────────┐
              │   NvTracker     │
              │ Multi-Object    │
              │    Tracking     │
              └────────┬────────┘
                       │
                       ▼
              ┌─────────────────┐
              │   OSD / Output  │
              └────────┬────────┘
                       │
                ┌──────┴──────┐
                ▼             ▼
             Display         RTSP
```

---

## 4. Multi-Camera Architecture

The multi-camera version uses DeepStream's stream multiplexer to combine multiple sources into batches before inference.

```text
Camera 0 ──┐
           │
Camera 1 ──┼──► GStreamer Sources
           │
RTSP 0 ────┘
              │
              ▼
       ┌───────────────┐
       │   nvstreammux │
       │               │
       │ Batch Frames  │
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

---

# 5. Technology Stack

| Component        | Technology                                      |
| ---------------- | ----------------------------------------------- |
| Programming      | C++, Python                                     |
| OS               | Ubuntu / JetPack Linux                          |
| Video pipeline   | GStreamer                                       |
| AI framework     | NVIDIA DeepStream                               |
| Inference        | TensorRT                                        |
| Model            | YOLO                                            |
| Computer Vision  | OpenCV                                          |
| Tracking         | DeepStream NvTracker                            |
| Hardware         | NVIDIA Jetson                                   |
| Model format     | ONNX / TensorRT Engine                          |
| Streaming        | RTSP                                            |
| Containerization | Docker                                          |
| Profiling        | DeepStream performance APIs + system monitoring |

---

# 6. Model Pipeline

The perception model follows:

```text
YOLO
  │
  ├── ONNX
  │
  ▼
TensorRT
  │
  ├── FP16
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

The inference engine is kept separate from the video-processing pipeline so that different TensorRT engines can be evaluated without redesigning the complete application.

---

# 7. GStreamer Pipeline

A simplified conceptual pipeline is:

```text
camera
  ↓
source
  ↓
video conversion
  ↓
NVMM buffer
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

The important design principle is to keep frames in GPU-compatible memory wherever possible and avoid unnecessary CPU↔GPU transfers.

---

# 8. Camera Support

The project is designed around three input modes.

## USB Camera

Example:

```bash
/dev/video0
```

The GStreamer source can be configured for formats such as:

```text
YUYV
MJPEG
NV12
```

depending on the camera capabilities.

Inspect the camera using:

```bash
v4l2-ctl --list-formats-ext -d /dev/video0
```

---

## CSI Camera

Jetson CSI cameras can be connected using the appropriate NVIDIA camera source.

The pipeline is designed so that the source component can be replaced without changing the downstream inference and tracking stages.

---

## RTSP Camera

The same perception pipeline can consume an RTSP stream:

```text
RTSP
 ↓
H264/H265 decoder
 ↓
NVMM
 ↓
DeepStream
```

This makes the architecture applicable to distributed camera systems.

---

# 9. Object Detection

The project uses a YOLO detector converted to TensorRT.

Example model flow:

```text
best.pt
   ↓
ONNX
   ↓
TensorRT
   ↓
FP16 engine
   ↓
DeepStream
```

The detector produces:

* bounding box
* class ID
* confidence
* frame/source ID

These are passed to downstream tracking components through DeepStream metadata.

---

# 10. Multi-Object Tracking

Detection alone produces independent bounding boxes for each frame.

Tracking adds temporal identity:

```text
Frame N

Person A ── ID 12
Person B ── ID 15


Frame N+1

Person A ── ID 12
Person B ── ID 15
```

This enables downstream applications such as:

* object counting
* trajectory analysis
* region-of-interest monitoring
* collision analysis
* behavior analysis

The project uses DeepStream's tracking interface so that different tracking backends can be evaluated without changing the rest of the pipeline.

---

# 11. Multi-Camera Processing

For multiple sources:

```text
                 ┌── Camera 0
                 │
                 ├── Camera 1
Sources ─────────┤
                 ├── Camera 2
                 │
                 └── RTSP
                      │
                      ▼
                nvstreammux
                      │
                      ▼
                  nvinfer
                      │
                      ▼
                  nvtracker
```

Each object retains source information:

```text
source_id
frame_id
object_id
class_id
confidence
bbox
```

This allows the system to distinguish objects belonging to different cameras.

---

# 12. Performance Optimization

The main optimization target is:

```text
Maximum throughput
+
Minimum latency
+
Controlled memory usage
+
Stable inference performance
```

Important parameters include:

* input resolution
* batch size
* number of cameras
* inference interval
* TensorRT precision
* tracker configuration
* decoder configuration
* encoder settings
* frame rate

---

# 13. Performance Metrics

The benchmark framework records:

### Throughput

```text
FPS = processed frames / elapsed time
```

### Latency

Time between frame arrival and generated perception output.

### Per-source FPS

```text
Camera 0 → 29.7 FPS
Camera 1 → 29.4 FPS
```

### GPU utilization

Measured using NVIDIA system monitoring tools.

### Memory

Track:

* GPU memory
* system memory
* TensorRT workspace
* buffer usage

---

# 14. Benchmark Matrix

The project uses controlled experiments rather than reporting a single FPS number.

Example benchmark matrix:

| Experiment     | Cameras | Resolution | Precision | Batch | FPS | Latency |
| -------------- | ------: | ---------: | --------- | ----: | --: | ------: |
| Baseline       |       1 |   1280×720 | FP32      |     1 | TBD |     TBD |
| FP16           |       1 |   1280×720 | FP16      |     1 | TBD |     TBD |
| Multi-camera   |       2 |   1280×720 | FP16      |     2 | TBD |     TBD |
| Multi-camera   |       4 |   1280×720 | FP16      |     4 | TBD |     TBD |
| Low-resolution |       4 |    640×384 | FP16      |     4 | TBD |     TBD |

Actual measurements should be collected on the target Jetson device.

---

# 15. Bottleneck Analysis

The project explicitly separates the pipeline into:

```text
Capture
  ↓
Decode
  ↓
Pre-processing
  ↓
Batching
  ↓
Inference
  ↓
Tracking
  ↓
Post-processing
  ↓
Encoding
  ↓
Streaming
```

This allows performance problems to be localized rather than assuming that neural-network inference is always the bottleneck.

For example:

```text
Inference:      12 ms
Pre-processing:  4 ms
Tracking:        3 ms
OSD:             2 ms
Encoding:        6 ms
----------------------
Pipeline:       ~27 ms
```

This distinction is important for real-time systems because optimizing the neural network alone may not improve end-to-end latency.

---

# 16. Failure Analysis

The project also evaluates common real-time perception failures.

### Camera failures

* frame drops
* unsupported formats
* camera disconnects
* exposure changes
* frame-rate instability

### Streaming failures

* RTSP timeout
* decoder errors
* packet loss
* network jitter
* H264/H265 compatibility

### Inference failures

* incorrect TensorRT engine
* unsupported layer
* preprocessing mismatch
* incorrect input dimensions
* confidence threshold problems

### Tracking failures

* ID switches
* missed detections
* occlusion
* fast motion

---

# 17. Debugging Methodology

When a pipeline fails, debugging follows a layered approach.

```text
Camera
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
Verify output encoder
  ↓
Verify RTSP
```

This avoids debugging the complete pipeline simultaneously.

---

# 18. Docker Deployment

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
    └── Application
```

This makes the software environment reproducible across development and deployment systems.

---

# 19. Example Commands

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

Run the single-camera pipeline:

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

---

# 20. Project Development Stages

## Stage 1 — GStreamer Camera Pipeline

* [x] Camera source
* [x] GStreamer pipeline
* [x] Format conversion
* [ ] FPS measurement
* [ ] Camera failure handling

## Stage 2 — DeepStream Inference

* [ ] DeepStream application
* [ ] TensorRT engine
* [ ] YOLO inference
* [ ] Detection metadata
* [ ] Performance measurement

## Stage 3 — Tracking

* [ ] NvTracker
* [ ] Persistent object IDs
* [ ] ID-switch analysis
* [ ] Tracking performance

## Stage 4 — Multi-Camera

* [ ] Multiple camera sources
* [ ] nvstreammux
* [ ] Batched inference
* [ ] Per-source metrics
* [ ] Synchronization analysis

## Stage 5 — Streaming

* [ ] H264/H265 encoding
* [ ] RTSP output
* [ ] Network streaming
* [ ] Latency measurement
* [ ] Stream failure recovery

## Stage 6 — Optimization

* [ ] FP16 TensorRT
* [ ] Batch-size experiments
* [ ] Resolution experiments
* [ ] GPU utilization analysis
* [ ] Memory analysis
* [ ] End-to-end latency analysis

---

# 21. Expected Final Demonstration

The final system should demonstrate:

```text
          Camera 0 ──────┐
                         │
          Camera 1 ──────┤
                         │
          RTSP Stream ───┤
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
                    NvTracker
                         │
                         ▼
                    nvdsosd
                         │
                  ┌──────┴──────┐
                  ▼             ▼
               Display         RTSP
```

The system should report:

```text
Sources:          2
Inference:        TensorRT FP16
Resolution:       1280 × 720
Detection:        YOLO
Tracking:         Enabled

Source 0 FPS:     XX.X
Source 1 FPS:     XX.X
Pipeline FPS:     XX.X
Latency:          XX ms
GPU utilization:  XX %
GPU memory:       XX MB
```

The values above should be replaced with measurements obtained on the actual Jetson hardware.

---

# 22. What This Project Demonstrates

This project demonstrates practical understanding of:

### Computer Vision

* Object detection
* Multi-object tracking
* Image processing
* Camera pipelines
* Real-time perception

### Camera Systems

* USB cameras
* CSI cameras
* Camera formats
* Camera-to-pipeline integration
* Frame-rate handling

### Video Systems

* GStreamer
* H264/H265
* RTSP
* Hardware decoding/encoding
* Pipeline debugging

### NVIDIA Edge AI

* Jetson
* DeepStream
* TensorRT
* CUDA
* GPU memory
* FP16 inference

### Software Engineering

* C++
* Python
* Docker
* Configuration-driven pipelines
* Performance benchmarking
* Failure analysis

---

# 23. Why This Project Matters

The important aspect of EdgeVision is that it treats perception as an **end-to-end systems problem** rather than simply an ML inference problem.

A high-performing neural network does not automatically result in a high-performing perception system.

The actual system performance depends on:

```text
Camera
+
Memory
+
Video Pipeline
+
Pre-processing
+
Inference
+
Tracking
+
Post-processing
+
Encoding
+
Network
```

The project therefore evaluates both **model-level performance and end-to-end pipeline performance**.

---

# 24. Future Work

Potential extensions include:

* Camera synchronization
* Hardware-triggered cameras
* Cross-camera object association
* Camera calibration integration
* Temporal sensor fusion
* LiDAR-camera fusion
* TensorRT INT8 calibration
* CUDA preprocessing kernels
* Zero-copy optimization
* Dynamic source management
* Fault-tolerant stream recovery
* ROS2 integration
* WebRTC output
* Edge-to-cloud telemetry

---

# 25. Resume Description

A concise resume version of the project:

**EdgeVision — Multi-Camera Real-Time Perception**

* Built a GStreamer + NVIDIA DeepStream perception pipeline for multi-camera edge video analytics, integrating TensorRT YOLO inference, NvTracker and hardware-accelerated video processing on NVIDIA Jetson.
* Developed configurable camera/RTSP pipelines and benchmarked end-to-end FPS, latency, GPU utilization and memory across camera counts, resolutions and FP16 inference configurations.
* Debugged capture, decoding, inference, tracking and streaming stages independently to identify pipeline bottlenecks and optimize real-time performance.

---

# 26. Interview Talking Point

The key engineering lesson from this project is:

> **"I learned to treat real-time Computer Vision as a complete streaming system rather than just an inference problem. When FPS or latency drops, I isolate capture, decoding, memory transfer, preprocessing, inference, tracking and encoding individually before optimizing the model."**

This is the central idea behind the project and the reason for using DeepStream and GStreamer rather than implementing the complete pipeline with OpenCV alone.

---

## License

MIT License
