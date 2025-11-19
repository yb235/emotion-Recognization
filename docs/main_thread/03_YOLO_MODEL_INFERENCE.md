# YOLO Model Inference - Deep Dive

## 🎯 Overview

This document provides an in-depth explanation of how YOLO11 Pose model inference works in the Main Thread, including model architecture, inference pipeline, optimization strategies, and device configuration.

## 🏗️ YOLO11 Pose Architecture

### Model Structure

```
Input Image (640×640×3)
         ↓
┌─────────────────────┐
│   Backbone          │  ← Feature extraction
│   (CSPDarknet)      │     
└─────────────────────┘
         ↓
┌─────────────────────┐
│   Neck              │  ← Multi-scale fusion
│   (PANet)           │
└─────────────────────┘
         ↓
┌─────────────────────┐
│   Head              │  ← Pose detection
│   (Pose Head)       │
└─────────────────────┘
         ↓
Output: Bounding Boxes + 17 Keypoints per Person
```

### Component Details

#### 1. Backbone: CSPDarknet
**Purpose**: Extract hierarchical features from input image

**Architecture**:
- Multiple convolutional layers
- Cross-Stage Partial (CSP) connections
- Reduces redundant gradient information
- Efficient feature reuse

**Output**: Feature maps at multiple scales (P3, P4, P5)

#### 2. Neck: PANet (Path Aggregation Network)
**Purpose**: Fuse multi-scale features for better detection

**Architecture**:
- Bottom-up path augmentation
- Feature pyramid
- Enhances localization accuracy

**Output**: Enriched feature maps

#### 3. Head: Pose Detection Head
**Purpose**: Predict bounding boxes and keypoints

**Output per Detection**:
- Bounding box: `[x, y, width, height]`
- Confidence score: `[0.0 - 1.0]`
- Class: `0` (person)
- 17 Keypoints: `[(x, y, confidence)] × 17`

## 🔄 Inference Pipeline

### Step-by-Step Process

```
Frame (BGR, HxWx3)
         ↓
┌────────────────────────┐
│  1. Preprocessing      │
│  - Resize to 640×640   │
│  - BGR → RGB           │
│  - Normalize [0,1]     │
│  - Add batch dimension │
└────────────────────────┘
         ↓
┌────────────────────────┐
│  2. Model Forward      │
│  - Backbone inference  │
│  - Neck processing     │
│  - Head prediction     │
└────────────────────────┘
         ↓
┌────────────────────────┐
│  3. Postprocessing     │
│  - NMS (suppress dups) │
│  - Coordinate scaling  │
│  - Confidence filter   │
└────────────────────────┘
         ↓
Results Object
```

### 1. Preprocessing (Automatic)

When using Ultralytics YOLO, preprocessing is handled automatically:

```python
results = model(frame, device="cpu", verbose=False)
```

**Behind the Scenes**:
```python
# What YOLO does internally:
# 1. Convert BGR → RGB
img = cv2.cvtColor(frame, cv2.COLOR_BGR2RGB)

# 2. Resize (maintain aspect ratio with letterbox)
img = letterbox(img, new_shape=(640, 640), stride=32)

# 3. Normalize to [0, 1]
img = img.astype(np.float32) / 255.0

# 4. Transpose HWC → CHW
img = img.transpose(2, 0, 1)

# 5. Add batch dimension
img = img[np.newaxis, ...]  # Shape: (1, 3, 640, 640)

# 6. Convert to tensor and move to device
tensor = torch.from_numpy(img).to(device)
```

**Why 640×640?**
- YOLO11 is trained on 640×640 images
- Stride of 32 pixels (model architecture constraint)
- Square input simplifies aspect ratio handling
- Good balance between speed and accuracy

### 2. Model Forward Pass

```python
# High-level API (what we use)
results = model(frame, device="cpu", verbose=False)

# What happens internally
with torch.no_grad():  # Disable gradient computation
    predictions = model.model(tensor)  # Forward pass
```

**Computation Graph**:
```
Input Tensor (1, 3, 640, 640)
    ↓
Conv2d → BatchNorm → SiLU → Conv2d → ... (Backbone)
    ↓
Feature Maps: P3(80×80), P4(40×40), P5(20×20)
    ↓
PANet Fusion
    ↓
Detection Head
    ↓
Raw Predictions (1, 56, 8400)
```

**Output Shape Explanation**:
- **Dimension 0**: Batch size (always 1 in our case)
- **Dimension 1**: Prediction vector (56 values per detection)
  - Index 0-3: Bounding box `[x, y, w, h]`
  - Index 4: Objectness score
  - Index 5: Class score
  - Index 6-55: 17 keypoints × 3 (x, y, confidence) = 51 values
  - Total: 4 + 1 + 1 + 51 = 57 (actual is 56, last confidence may be omitted)
- **Dimension 2**: Anchor candidates (8400 detection proposals)
  - 80×80 grid = 6400 proposals
  - 40×40 grid = 1600 proposals  
  - 20×20 grid = 400 proposals
  - Total: 8400 proposals

### 3. Postprocessing (Automatic)

**Non-Maximum Suppression (NMS)**:
```python
# YOLO automatically applies NMS to remove duplicate detections
# Parameters (internal):
conf_threshold = 0.25  # Minimum confidence to keep detection
iou_threshold = 0.45   # IoU threshold for NMS
max_det = 300          # Maximum detections per image
```

**What NMS Does**:
1. Filter detections by confidence (> 0.25)
2. Sort by confidence (highest first)
3. For each detection:
   - Keep if no overlap with higher-confidence detection
   - Remove if IoU > 0.45 with higher-confidence detection

**Coordinate Scaling**:
```python
# Scale keypoints from 640×640 back to original frame size
# YOLO does this automatically
kp_scaled = keypoints * (original_width / 640, original_height / 640)
```

## 📦 Using the Model

### Basic Usage

```python
from ultralytics import YOLO

# Load model (once at startup)
model = YOLO("yolo11m-pose.pt")

# Run inference (in loop)
results = model(frame, device="cpu", verbose=False)
```

### Extracting Results

```python
for r in results:
    # Bounding boxes (if needed)
    boxes = r.boxes.xyxy.cpu().numpy()  # Shape: (N, 4)
    confidences = r.boxes.conf.cpu().numpy()  # Shape: (N,)
    
    # Keypoints (what we use)
    if r.keypoints is not None:
        for person in r.keypoints.xy:  # xy gives (x, y) coordinates
            kp = person.cpu().numpy()  # Shape: (17, 2)
            
            # Now kp contains 17 keypoints
            # kp[0] = nose (x, y)
            # kp[1] = left_eye (x, y)
            # ...
```

### Keypoint Format

```python
# Each person has 17 keypoints
keypoints = [
    [x0, y0],   # 0: nose
    [x1, y1],   # 1: left_eye
    [x2, y2],   # 2: right_eye
    [x3, y3],   # 3: left_ear
    [x4, y4],   # 4: right_ear
    [x5, y5],   # 5: left_shoulder
    [x6, y6],   # 6: right_shoulder
    [x7, y7],   # 7: left_elbow
    [x8, y8],   # 8: right_elbow
    [x9, y9],   # 9: left_wrist
    [x10, y10], # 10: right_wrist
    [x11, y11], # 11: left_hip
    [x12, y12], # 12: right_hip
    [x13, y13], # 13: left_knee
    [x14, y14], # 14: right_knee
    [x15, y15], # 15: left_ankle
    [x16, y16], # 16: right_ankle
]
```

**Coordinate System**:
- Origin (0, 0) at top-left of image
- X increases to the right
- Y increases downward
- Units: pixels in original frame size

## ⚙️ Device Configuration

### CPU Inference

```python
# Explicit CPU
results = model(frame, device="cpu", verbose=False)

# Pros:
# - Works everywhere
# - No special setup required
# - Consistent behavior

# Cons:
# - Slower (~80-100ms per frame)
# - Limited to ~10 FPS
# - Uses 40-60% of one CPU core
```

**Performance Characteristics**:
- **Intel i5/i7**: 80-100ms per frame
- **Intel i3**: 120-150ms per frame
- **AMD Ryzen 5/7**: 70-90ms per frame

### NVIDIA GPU (CUDA)

```python
# Requires: CUDA toolkit + PyTorch with CUDA support
results = model(frame, device="cuda", verbose=False)

# Or specify GPU index
results = model(frame, device="cuda:0", verbose=False)

# Pros:
# - Much faster (~20-30ms per frame)
# - Achieves ~30+ FPS
# - Frees CPU for other tasks

# Cons:
# - Requires NVIDIA GPU
# - Requires CUDA installation
# - Uses ~500MB VRAM
```

**Performance Characteristics**:
- **RTX 3060**: 20-25ms per frame
- **GTX 1660**: 30-40ms per frame  
- **RTX 4090**: 10-15ms per frame

### Apple Silicon (MPS)

```python
# Requires: macOS 12.3+ with Apple Silicon (M1/M2/M3)
results = model(frame, device="mps", verbose=False)

# Pros:
# - Faster than CPU (~30-50ms per frame)
# - Native on Apple Silicon Macs
# - Good energy efficiency

# Cons:
# - macOS only
# - Some operations fall back to CPU
# - Less mature than CUDA
```

**Performance Characteristics**:
- **M1**: 40-50ms per frame
- **M1 Pro/Max**: 30-40ms per frame
- **M2**: 35-45ms per frame
- **M3**: 25-35ms per frame

### Device Selection Strategy

```python
import torch

def select_device():
    """Automatically select best available device"""
    if torch.cuda.is_available():
        return "cuda"
    elif torch.backends.mps.is_available():
        return "mps"
    else:
        return "cpu"

# Use in code
device = select_device()
results = model(frame, device=device, verbose=False)
```

## 🎛️ Model Selection

### Available YOLO Pose Models

| Model | Size | Speed (CPU) | Speed (GPU) | Accuracy |
|-------|------|-------------|-------------|----------|
| yolov8n-pose | 6.5 MB | 50ms | 10ms | ⭐⭐⭐ |
| yolov8s-pose | 23 MB | 65ms | 15ms | ⭐⭐⭐⭐ |
| yolov8m-pose | 52 MB | 100ms | 25ms | ⭐⭐⭐⭐⭐ |
| yolo11n-pose | 5.8 MB | 45ms | 8ms | ⭐⭐⭐ |
| yolo11s-pose | 20 MB | 60ms | 12ms | ⭐⭐⭐⭐ |
| **yolo11m-pose** | **84 MB** | **80ms** | **20ms** | **⭐⭐⭐⭐⭐** |
| yolo11l-pose | 165 MB | 120ms | 30ms | ⭐⭐⭐⭐⭐⭐ |
| yolo11x-pose | 232 MB | 180ms | 45ms | ⭐⭐⭐⭐⭐⭐ |

**Current Choice**: `yolo11m-pose.pt`
- Good balance of speed and accuracy
- 84 MB model size (manageable)
- 80ms on CPU (acceptable)
- 20ms on GPU (excellent)

### Changing Models

```python
# Faster but less accurate
model = YOLO("yolov8n-pose.pt")

# Current (balanced)
model = YOLO("yolo11m-pose.pt")

# Slower but more accurate
model = YOLO("yolo11l-pose.pt")
```

**When to Use Smaller Models**:
- CPU-only systems
- Real-time requirements (30 FPS needed)
- Resource-constrained environments
- Testing and development

**When to Use Larger Models**:
- GPU available
- Accuracy is critical
- Complex scenes with occlusions
- Multiple people detection

## 🚀 Optimization Strategies

### 1. Reduce Input Resolution

```python
# Before inference, resize frame
frame_small = cv2.resize(frame, (640, 480))
results = model(frame_small, device="cpu", verbose=False)

# Trade-off: Faster inference, lower accuracy
```

### 2. Skip Frames

```python
frame_count = 0
skip_frames = 2  # Process every 3rd frame

while True:
    ret, frame = cap.read()
    frame_count += 1
    
    if frame_count % skip_frames == 0:
        results = model(frame, device="cpu", verbose=False)
        # ... process results ...
    
    # Still display all frames for smooth video
    cv2.imshow("Window", frame)
```

### 3. Use Half-Precision (FP16)

```python
# Only on GPU with tensor cores (RTX 20xx+)
model = YOLO("yolo11m-pose.pt")
model.model.half()  # Convert to FP16

# Faster inference with minimal accuracy loss
results = model(frame, device="cuda", verbose=False)
```

### 4. Batch Processing (If Applicable)

```python
# For video file processing (not real-time)
frames = [frame1, frame2, frame3, frame4]
results_batch = model(frames, device="cuda", verbose=False)

# More efficient GPU utilization
```

### 5. ONNX Runtime

For maximum performance, export to ONNX:

```python
# Export model (once)
model = YOLO("yolo11m-pose.pt")
model.export(format="onnx")

# Use with ONNX Runtime (see run_pose_onnx_dml.py)
import onnxruntime as ort
session = ort.InferenceSession("yolo11m-pose.onnx")
```

**Benefits**:
- 10-20% faster inference
- Better hardware optimization
- Support for DirectML (Windows GPU)
- Lower memory footprint

## 🐛 Troubleshooting

### Issue: "CUDA out of memory"

**Solution**:
```python
# Reduce batch size (if batching)
# Or clear cache
import torch
torch.cuda.empty_cache()

# Or use smaller model
model = YOLO("yolo11n-pose.pt")
```

### Issue: "No keypoints detected"

**Possible Causes**:
1. Person too far from camera
2. Poor lighting
3. Occlusion
4. Person facing away

**Debug**:
```python
results = model(frame, device="cpu", verbose=True)
for r in results:
    print(f"Boxes detected: {len(r.boxes)}")
    print(f"Keypoints: {r.keypoints}")
    if r.keypoints is not None:
        print(f"Keypoint shape: {r.keypoints.xy.shape}")
```

### Issue: "Inference too slow"

**Solutions**:
1. Use GPU: `device="cuda"`
2. Use smaller model: `yolo11n-pose.pt`
3. Skip frames: Process every 2nd or 3rd frame
4. Reduce resolution: Resize input frame
5. Use ONNX Runtime

## 📊 Performance Profiling

### Measure Inference Time

```python
import time

# Warm-up (first inference is slower)
for _ in range(5):
    _ = model(frame, device="cpu", verbose=False)

# Measure
times = []
for _ in range(100):
    start = time.time()
    results = model(frame, device="cpu", verbose=False)
    times.append(time.time() - start)

print(f"Mean: {np.mean(times)*1000:.1f}ms")
print(f"Std: {np.std(times)*1000:.1f}ms")
print(f"Min: {np.min(times)*1000:.1f}ms")
print(f"Max: {np.max(times)*1000:.1f}ms")
```

### Profile with PyTorch Profiler

```python
from torch.profiler import profile, ProfilerActivity

with profile(activities=[ProfilerActivity.CPU, ProfilerActivity.CUDA]) as prof:
    results = model(frame, device="cuda", verbose=False)

print(prof.key_averages().table(sort_by="cuda_time_total", row_limit=10))
```

## 🔗 Advanced Topics

### Custom Confidence Threshold

```python
# YOLO uses conf=0.25 by default
# To change:
results = model(frame, device="cpu", verbose=False, conf=0.5)

# Higher conf = fewer false positives, more false negatives
# Lower conf = more detections, more false positives
```

### IOU Threshold for NMS

```python
results = model(frame, device="cpu", verbose=False, iou=0.6)

# Higher IOU = keep more overlapping detections
# Lower IOU = more aggressive duplicate suppression
```

### Maximum Detections

```python
results = model(frame, device="cpu", verbose=False, max_det=10)

# Limit number of people detected (useful for performance)
```

## 📚 Further Reading

- **[04_KEYPOINT_EXTRACTION.md](04_KEYPOINT_EXTRACTION.md)** - Working with keypoints
- **[05_ACTION_DETECTION_ENGINE.md](05_ACTION_DETECTION_ENGINE.md)** - Using keypoints for action detection
- **[06_CONFIGURING_DETECTIONS.md](06_CONFIGURING_DETECTIONS.md)** - Tuning for your use case

## 🔗 External Resources

- [Ultralytics YOLO Documentation](https://docs.ultralytics.com/)
- [YOLO11 Paper](https://arxiv.org/abs/2301.00808)
- [COCO Keypoints Format](https://cocodataset.org/#keypoints-2020)
- [PyTorch CUDA Documentation](https://pytorch.org/docs/stable/cuda.html)

---

← [Previous: Video Capture Pipeline](02_VIDEO_CAPTURE_PIPELINE.md) | [Back to Main Thread Docs](00_README.md) | [Next: Keypoint Extraction →](04_KEYPOINT_EXTRACTION.md)
