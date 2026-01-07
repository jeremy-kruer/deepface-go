# Project Blueprint: DeepFace-Go

## 1. Executive Summary
**Goal:** Create a production-grade, open-source Go library for facial recognition that mirrors the usability of Python's `DeepFace`.
**Philosophy:** "Batteries included, but performance first."
**Core deliverables:**
1.  A reusable **Go Module** (`pkg/`) that can be imported into any application.
2.  A rich **TUI CLI** (`cmd/cli`) using Cobra, Bubble Tea, and Lip Gloss.
3.  A high-performance **REST API** (`cmd/server`) using Chi.

**Standards:** The project will adhere to strict TDD (Test-Driven Development), achieve high test coverage (>80%), and include benchmarks for all critical paths.

## 2. Technology Stack

### Core Library
| Component | Technology | Rationale |
| :--- | :--- | :--- |
| **Inference** | **ONNX Runtime** (`yalue/onnxruntime_go`) | Standardized, cross-platform inference without heavy Python bindings. |
| **Face Detection** | `UltraFace` (ONNX Model) | The standard lightweight detector used with ArcFace. |
| **Face Recognition** | `InsightFace/ArcFace` (ONNX Model) | The Gold Standard |
| **Face Recognition (Option 2)** | `Google FaceNet` (ONNX Model) | Option 2 |
| **Face Recognition (Option 3)** | `VGG-Face` (ONNX Model) | Option 3 |
| **Face Recognition (Option 4)** | `Facebook DeepFace` (ONNX Model) | Option 4 |
| **Image Ops** | **bimg** (libvips) | Streaming architecture for low-memory, high-speed cropping/rotation. |
| **Testing** | Standard `testing` + `testify` | Robust unit testing and mocking. |

### Interfaces
| Interface | Technology | Rationale |
| :--- | :--- | :--- |
| **CLI Framework** | **Cobra** | Standard for Go CLI commands and flags. |
| **TUI / UX** | **Bubble Tea** + **Lip Gloss** | Interactive terminal UI and styling (mimicking modern dev tools). |
| **HTTP Router** | **Chi** | Lightweight, idiomatic, and middleware-friendly router. |

## 3. Architecture & mimicry
The library API should feel familiar to users of Python's `DeepFace`. We will map standard DeepFace functions to Go interfaces where possible.

### Core Functions (Go Interface)
* `Verify(img1, img2 string) (VerificationResult, error)`
* `Find(imgPath string, dbPath string) ([]FaceResult, error)`
* `Represent(img string) ([]float32, error)`
* `Stream()` (Future implementation for webcam)

### The "Smart" Pre-processing Strategy (bimg + ONNX)
To maintain DeepFace parity without `dlib`/`opencv`:
1.  **Detector:** Use **UltraFace** (ONNX) to find the bounding box and 5 landmarks (eyes, nose, mouth).
2.  **Alignment Logic:**
    * Calculate angle between eyes.
    * If angle > 10°, use `bimg.Rotate` to correct orientation.
    * *Constraint:* We forgo complex affine warping (shearing) in favor of speed, as ArcFace is robust enough to handle simple rotation.
3.  **Normalization:** Standardize pixel values (e.g., `(x-127.5)/128`) before tensor conversion.

## 4. Project Directory Structure
Standard Go layout (following `golang-standards/project-layout`):

```text
deepface-go/
├── cmd/
│   ├── cli/
│   │   └── main.go           # Entry point for the Cobra/BubbleTea CLI
│   └── server/
│       └── main.go           # Entry point for the Chi REST API
├── pkg/
│   ├── deepface/             # The public library code
│   │   ├── deepface.go       # High-level API (Verify, Find, Represent)
│   │   ├── detection.go      # UltraFace logic
│   │   ├── recognition.go    # ArcFace logic
│   │   └── preprocessing.go  # bimg logic (crop, rotate)
│   └── types/                # Shared structs (Face, Embedding, Config)
├── internal/
│   └── onnx/                 # Internal wrappers for model loading (Singleton pattern)
├── models/
│   ├── arcface_r100.onnx     # Pre-trained models
│   └── ultraface_slim.onnx
├── tests/                    # Integration and Performance tests
├── go.mod
└── README.md
```

## 5. Development Methodology: TDD & Benchmarks
### Phase 1: The Core Library (TDD Approach)
* **Write the Test:** Create `pkg/deepface/preprocessing_test.go`. Define a test case: "Given an image rotated 45°, expect output image to have eyes horizontally aligned within 5° tolerance."
* **Implement:** Write the `bimg` logic to satisfy the test.
* **Benchmark:** Add `BenchmarkPreprocessing` to ensure latency stays under 10ms.

### Phase 2: The CLI (Bubble Tea)
* **Implement** `deepface verify <img1> <img2>`.
* Use **Bubble Tea** to show a loading spinner while the ONNX model warms up.
* Use **Lip Gloss** to render a green "MATCH" or red "MISMATCH" card in the terminal.

### Phase 3: The API (Chi)
* **Implement** `POST /verify` and `POST /represent`.
* **Middleware:** Add request logging and panic recovery.
* **Output:** JSON structure exactly matching Python DeepFace's dictionary output.
