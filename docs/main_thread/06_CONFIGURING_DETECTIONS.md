# Configuring Motion Recognition Settings

## 🎯 Overview

This document explains how to configure and tune the motion recognition system for different environments, use cases, and accuracy requirements. Learn how to adjust thresholds, optimize for your camera setup, and balance performance vs accuracy.

## ⚙️ Configuration Categories

### 1. Detection Thresholds
### 2. Camera and Environment Settings  
### 3. Performance Optimization
### 4. Model Selection
### 5. Device Configuration

---

## 1️⃣ Detection Thresholds

### Overview

Each action has distance or measurement thresholds that determine when it's detected. These can be adjusted based on your specific needs.

### All Action Thresholds

```python
# In detect_custom_actions() function

# 1. Arms Crossed
ARMS_CROSSED_THRESHOLD = 80  # pixels

# 2. Hands Clasped
HANDS_CLASPED_THRESHOLD = 60  # pixels

# 3. Chin Rest
CHIN_REST_THRESHOLD = 70  # pixels

# 4. Lean Forward
LEAN_FORWARD_THRESHOLD = 120  # pixels (torso height)

# 5. Lean Back
LEAN_BACK_THRESHOLD = 200  # pixels (torso height)

# 6. Head Down
HEAD_DOWN_OFFSET = 40  # pixels (nose below shoulder)

# 7. Touch Face
TOUCH_FACE_THRESHOLD = 70  # pixels

# 8. Touch Nose
TOUCH_NOSE_THRESHOLD = 40  # pixels

# 9. Fix Hair
FIX_HAIR_THRESHOLD = 60  # pixels

# 10. Fidget Hands
FIDGET_MOVEMENT_THRESHOLD = 25  # pixels per frame
```

### How to Modify Thresholds

#### Method 1: Direct Edit (Simple)

Edit the values directly in `interview_system.py` or `action_detector3.py`:

```python
# Find the action detection logic
def detect_custom_actions(kp):
    # ...
    
    # Change this:
    if distance(l_wrist, r_elbow) < 80 and distance(r_wrist, l_elbow) < 80:
        actions.append("arms_crossed")
    
    # To this (more lenient):
    if distance(l_wrist, r_elbow) < 100 and distance(r_wrist, l_elbow) < 100:
        actions.append("arms_crossed")
```

#### Method 2: Configuration Dictionary (Advanced)

Add at the top of the file:

```python
# Configuration dictionary
ACTION_THRESHOLDS = {
    "arms_crossed": 80,
    "hands_clasped": 60,
    "chin_rest": 70,
    "lean_forward": 120,
    "lean_back": 200,
    "head_down": 40,
    "touch_face": 70,
    "touch_nose": 40,
    "fix_hair": 60,
    "fidget_hands": 25,
}

# In detect_custom_actions():
def detect_custom_actions(kp, thresholds=ACTION_THRESHOLDS):
    # Use thresholds["arms_crossed"] instead of hardcoded 80
    if (distance(l_wrist, r_elbow) < thresholds["arms_crossed"] and
        distance(r_wrist, l_elbow) < thresholds["arms_crossed"]):
        actions.append("arms_crossed")
```

#### Method 3: External Config File (Production)

Create `config.json`:
```json
{
  "action_thresholds": {
    "arms_crossed": 80,
    "hands_clasped": 60,
    "chin_rest": 70,
    "lean_forward": 120,
    "lean_back": 200,
    "head_down": 40,
    "touch_face": 70,
    "touch_nose": 40,
    "fix_hair": 60,
    "fidget_hands": 25
  },
  "camera": {
    "device_id": 0,
    "resolution": [1920, 1080]
  },
  "model": {
    "path": "yolo11m-pose.pt",
    "device": "cpu",
    "confidence": 0.25
  }
}
```

Load in code:
```python
import json

with open("config.json", "r") as f:
    config = json.load(f)

thresholds = config["action_thresholds"]

# Use in detect_custom_actions()
def detect_custom_actions(kp, thresholds=thresholds):
    # ... detection logic using thresholds dict ...
```

### Threshold Adjustment Guidelines

#### Making Detection More Sensitive (More Detections)
**Increase thresholds** by 20-30%:
```python
"arms_crossed": 100,  # Was 80
"hands_clasped": 80,  # Was 60
```

**When to use**:
- Camera is far from subject (3+ meters)
- High resolution camera (4K)
- Need to catch subtle movements
- False negatives are more costly than false positives

#### Making Detection More Strict (Fewer False Positives)
**Decrease thresholds** by 20-30%:
```python
"arms_crossed": 60,   # Was 80
"hands_clasped": 40,  # Was 60
```

**When to use**:
- Camera is close to subject (< 1 meter)
- Low resolution camera (720p)
- Need high precision
- False positives are problematic

### Camera Distance Adjustment Table

| Distance | Multiplier | Example (arms_crossed) |
|----------|-----------|------------------------|
| 0.5m - 1m | 0.7x | 56 pixels |
| 1m - 2m | 1.0x | 80 pixels (default) |
| 2m - 3m | 1.3x | 104 pixels |
| 3m - 4m | 1.6x | 128 pixels |
| 4m+ | 2.0x | 160 pixels |

**Implementation**:
```python
CAMERA_DISTANCE_METERS = 2.5  # Adjust based on your setup
DISTANCE_MULTIPLIER = CAMERA_DISTANCE_METERS / 2.0  # 2m is baseline

# Apply to all thresholds
adjusted_threshold = int(80 * DISTANCE_MULTIPLIER)
```

### Resolution Scaling

| Resolution | Multiplier | Example (arms_crossed) |
|-----------|-----------|------------------------|
| 640×480 | 0.6x | 48 pixels |
| 1280×720 | 0.8x | 64 pixels |
| 1920×1080 | 1.0x | 80 pixels (default) |
| 2560×1440 | 1.3x | 104 pixels |
| 3840×2160 | 1.5x | 120 pixels |

---

## 2️⃣ Camera and Environment Settings

### Camera Device Selection

```python
# Default camera
cap = cv2.VideoCapture(0)

# Specific camera by index
cap = cv2.VideoCapture(1)  # Second camera

# By name (Linux)
cap = cv2.VideoCapture("/dev/video0")

# Network camera
cap = cv2.VideoCapture("rtsp://192.168.1.100:554/stream")
```

### Camera Resolution

```python
# Set before reading frames
cap = cv2.VideoCapture(0)

# Lower resolution (faster, less accurate)
cap.set(cv2.CAP_PROP_FRAME_WIDTH, 640)
cap.set(cv2.CAP_PROP_FRAME_HEIGHT, 480)

# Default (balanced)
cap.set(cv2.CAP_PROP_FRAME_WIDTH, 1280)
cap.set(cv2.CAP_PROP_FRAME_HEIGHT, 720)

# High resolution (slower, more accurate)
cap.set(cv2.CAP_PROP_FRAME_WIDTH, 1920)
cap.set(cv2.CAP_PROP_FRAME_HEIGHT, 1080)
```

### Frame Rate

```python
# Set target FPS
cap.set(cv2.CAP_PROP_FPS, 30)

# Verify actual FPS
actual_fps = cap.get(cv2.CAP_PROP_FPS)
print(f"Camera FPS: {actual_fps}")
```

### Camera Settings (Advanced)

```python
# Exposure
cap.set(cv2.CAP_PROP_EXPOSURE, -6)  # Auto exposure

# Brightness
cap.set(cv2.CAP_PROP_BRIGHTNESS, 128)  # 0-255

# Contrast
cap.set(cv2.CAP_PROP_CONTRAST, 128)  # 0-255

# Saturation
cap.set(cv2.CAP_PROP_SATURATION, 128)  # 0-255

# Focus
cap.set(cv2.CAP_PROP_AUTOFOCUS, 1)  # 1=auto, 0=manual
```

### Environment Considerations

#### Lighting Conditions

**Good Lighting** (Recommended):
- Even illumination on subject
- No strong backlight
- Natural or soft white light
- Minimal shadows

**Poor Lighting** (Adjust):
```python
# Increase exposure for dark environments
cap.set(cv2.CAP_PROP_EXPOSURE, -4)

# Increase brightness
cap.set(cv2.CAP_PROP_BRIGHTNESS, 150)

# Consider: Use smaller YOLO model (faster, compensates for detection difficulty)
model = YOLO("yolo11n-pose.pt")
```

#### Background

**Optimal**:
- Plain, contrasting background
- No busy patterns
- Minimal objects

**Problematic backgrounds**:
```python
# If background has people/objects, increase confidence threshold
results = model(frame, device="cpu", conf=0.4)  # Default is 0.25
```

---

## 3️⃣ Performance Optimization

### Speed vs Accuracy Trade-offs

#### Option 1: Model Selection

```python
# Fastest (5ms GPU, 45ms CPU)
model = YOLO("yolo11n-pose.pt")

# Balanced (20ms GPU, 80ms CPU) ← Default
model = YOLO("yolo11m-pose.pt")

# Most Accurate (45ms GPU, 180ms CPU)
model = YOLO("yolo11x-pose.pt")
```

#### Option 2: Skip Frames

```python
frame_count = 0
PROCESS_EVERY_N_FRAMES = 2  # Process every 2nd frame

while True:
    ret, frame = cap.read()
    frame_count += 1
    
    if frame_count % PROCESS_EVERY_N_FRAMES == 0:
        results = model(frame, device="cpu", verbose=False)
        # ... process results ...
    
    # Still display all frames (smooth video)
    cv2.imshow("Window", frame)
```

**Performance Gain**: 2x speedup with minimal accuracy loss

#### Option 3: Lower Input Resolution

```python
# Resize frame before inference
frame_small = cv2.resize(frame, (640, 480))
results = model(frame_small, device="cpu", verbose=False)

# Scale keypoints back to original size
# (handled automatically by YOLO)
```

**Performance Gain**: 30-50% speedup

#### Option 4: Disable Verbose Logging

```python
# Always use verbose=False in production
results = model(frame, device="cpu", verbose=False)
```

### Device-Specific Optimization

#### CPU Optimization

```python
import cv2
import os

# Use all CPU cores for OpenCV
cv2.setNumThreads(os.cpu_count())

# Use CPU-optimized YOLO
model = YOLO("yolo11n-pose.pt")  # Lighter model
results = model(frame, device="cpu", verbose=False)
```

#### GPU Optimization (CUDA)

```python
# Use GPU
results = model(frame, device="cuda", verbose=False)

# Enable half-precision (FP16) on RTX GPUs
model.model.half()
results = model(frame, device="cuda", verbose=False)

# Batch processing (if not real-time)
frames = [frame1, frame2, frame3, frame4]
results = model(frames, device="cuda", verbose=False)
```

#### Apple Silicon Optimization

```python
# Use MPS (Metal Performance Shaders)
results = model(frame, device="mps", verbose=False)

# MPS may be slower than CPU for small models on M1
# Test both:
import time

# Test CPU
start = time.time()
results = model(frame, device="cpu", verbose=False)
cpu_time = time.time() - start

# Test MPS  
start = time.time()
results = model(frame, device="mps", verbose=False)
mps_time = time.time() - start

print(f"CPU: {cpu_time*1000:.1f}ms, MPS: {mps_time*1000:.1f}ms")
device = "cpu" if cpu_time < mps_time else "mps"
```

---

## 4️⃣ Model Configuration

### YOLO Inference Parameters

```python
results = model(
    frame,
    device="cpu",           # Device: "cpu", "cuda", "mps"
    verbose=False,          # Print detection info
    conf=0.25,             # Confidence threshold (0.0-1.0)
    iou=0.45,              # IoU threshold for NMS (0.0-1.0)
    max_det=10,            # Max detections per image
    imgsz=640,             # Input image size
    half=False,            # Use FP16 (GPU only)
    augment=False,         # Test-time augmentation
)
```

### Parameter Explanations

#### conf (Confidence Threshold)

Controls minimum confidence to keep a detection.

```python
# More detections, more false positives
results = model(frame, conf=0.1)

# Default (balanced)
results = model(frame, conf=0.25)

# Fewer detections, fewer false positives
results = model(frame, conf=0.5)
```

**Recommended Values**:
- **0.1-0.2**: Crowded scenes, far from camera
- **0.25-0.35**: Normal use (default)
- **0.4-0.6**: Close-up, good lighting
- **0.7+**: Very strict, high precision needed

#### iou (Intersection over Union for NMS)

Controls duplicate suppression.

```python
# Keep more overlapping detections
results = model(frame, iou=0.6)

# Default (balanced)
results = model(frame, iou=0.45)

# Aggressive duplicate removal
results = model(frame, iou=0.3)
```

**Recommended Values**:
- **0.3-0.4**: Multiple overlapping people
- **0.45**: Normal use (default)
- **0.5-0.6**: Single person, avoid suppression

#### max_det (Maximum Detections)

Limits number of people detected.

```python
# Single person (fastest)
results = model(frame, max_det=1)

# Small group (default)
results = model(frame, max_det=10)

# Large group
results = model(frame, max_det=50)
```

---

## 5️⃣ Complete Configuration Examples

### Configuration 1: High Performance (Speed Priority)

```python
# Use case: Real-time dashboard, 30 FPS required

# Lightest model
model = YOLO("yolo11n-pose.pt")

# GPU if available
device = "cuda" if torch.cuda.is_available() else "cpu"

# Lower resolution
cap = cv2.VideoCapture(0)
cap.set(cv2.CAP_PROP_FRAME_WIDTH, 640)
cap.set(cv2.CAP_PROP_FRAME_HEIGHT, 480)

# Relaxed thresholds (less computation)
ACTION_THRESHOLDS = {k: v * 1.2 for k, v in ACTION_THRESHOLDS.items()}

# Inference
results = model(frame, device=device, conf=0.35, max_det=5, verbose=False)
```

**Expected Performance**: 30+ FPS on GPU, 20+ FPS on decent CPU

### Configuration 2: High Accuracy (Quality Priority)

```python
# Use case: Research, recorded video analysis

# Best model
model = YOLO("yolo11l-pose.pt")

# GPU recommended
device = "cuda"

# High resolution
cap = cv2.VideoCapture(0)
cap.set(cv2.CAP_PROP_FRAME_WIDTH, 1920)
cap.set(cv2.CAP_PROP_FRAME_HEIGHT, 1080)

# Strict thresholds
ACTION_THRESHOLDS = {k: v * 0.8 for k, v in ACTION_THRESHOLDS.items()}

# Inference
results = model(frame, device=device, conf=0.3, iou=0.5, verbose=False)
```

**Expected Performance**: 15-20 FPS on GPU

### Configuration 3: Balanced (Default)

```python
# Use case: General interview analysis

# Balanced model
model = YOLO("yolo11m-pose.pt")

# Auto device
device = select_device()  # Function defined earlier

# Standard resolution
cap = cv2.VideoCapture(0)
cap.set(cv2.CAP_PROP_FRAME_WIDTH, 1280)
cap.set(cv2.CAP_PROP_FRAME_HEIGHT, 720)

# Default thresholds
# (use as-is)

# Inference
results = model(frame, device=device, conf=0.25, verbose=False)
```

**Expected Performance**: 10-15 FPS on CPU, 30 FPS on GPU

### Configuration 4: Multi-Person

```python
# Use case: Group interviews, multiple subjects

# Balanced model
model = YOLO("yolo11m-pose.pt")

# GPU strongly recommended
device = "cuda"

# Higher max detections
results = model(frame, device=device, conf=0.3, max_det=20, verbose=False)

# Separate logs per person (requires person tracking)
# See Advanced Topics section
```

---

## 🧪 Testing Your Configuration

### Benchmark Script

```python
import time
import numpy as np

def benchmark_configuration(model, device, num_frames=100):
    """Test configuration performance"""
    
    cap = cv2.VideoCapture(0)
    times = []
    
    for i in range(num_frames):
        ret, frame = cap.read()
        if not ret:
            break
        
        start = time.time()
        results = model(frame, device=device, verbose=False)
        elapsed = time.time() - start
        times.append(elapsed)
    
    cap.release()
    
    print(f"\n=== Benchmark Results ===")
    print(f"Frames processed: {len(times)}")
    print(f"Mean time: {np.mean(times)*1000:.1f}ms")
    print(f"Std deviation: {np.std(times)*1000:.1f}ms")
    print(f"Min time: {np.min(times)*1000:.1f}ms")
    print(f"Max time: {np.max(times)*1000:.1f}ms")
    print(f"Average FPS: {1/np.mean(times):.1f}")
    print(f"95th percentile: {np.percentile(times, 95)*1000:.1f}ms")

# Run benchmark
benchmark_configuration(model, "cpu")
```

### A/B Testing Thresholds

```python
def test_threshold(threshold_value, test_duration=30):
    """Test a threshold value for specified duration"""
    
    cap = cv2.VideoCapture(0)
    start_time = time.time()
    
    detection_count = 0
    frame_count = 0
    
    while time.time() - start_time < test_duration:
        ret, frame = cap.read()
        if not ret:
            break
        
        frame_count += 1
        results = model(frame, device="cpu", verbose=False)
        
        for r in results:
            if r.keypoints is None:
                continue
            
            for person in r.keypoints.xy:
                kp = person.cpu().numpy()
                # Test with modified threshold
                if distance(kp[9], kp[10]) < threshold_value:
                    detection_count += 1
    
    cap.release()
    
    detection_rate = detection_count / frame_count if frame_count > 0 else 0
    print(f"Threshold {threshold_value}: {detection_rate*100:.1f}% detection rate")
    
    return detection_rate

# Test multiple thresholds
for thresh in [40, 60, 80, 100]:
    test_threshold(thresh, test_duration=10)
```

---

## 📊 Configuration Profiles

### Profile Summary Table

| Profile | Model | Device | FPS | Accuracy | Use Case |
|---------|-------|--------|-----|----------|----------|
| **Speed** | yolo11n | GPU | 30+ | ⭐⭐⭐ | Real-time dashboard |
| **Balanced** | yolo11m | CPU | 10-15 | ⭐⭐⭐⭐ | General use |
| **Quality** | yolo11l | GPU | 15-20 | ⭐⭐⭐⭐⭐ | Research |
| **Budget** | yolo11n | CPU | 15-20 | ⭐⭐⭐ | No GPU available |
| **Multi-Person** | yolo11m | GPU | 25-30 | ⭐⭐⭐⭐ | Group analysis |

---

## 📚 Further Reading

- **[03_YOLO_MODEL_INFERENCE.md](03_YOLO_MODEL_INFERENCE.md)** - Deep dive into YOLO
- **[07_ADDING_NEW_ACTIONS.md](07_ADDING_NEW_ACTIONS.md)** - Adding custom actions
- **[../06_ACTION_DETECTION.md](../06_ACTION_DETECTION.md)** - All actions explained

---

← [Previous: Action Detection Engine](05_ACTION_DETECTION_ENGINE.md) | [Back to Main Thread Docs](00_README.md) | [Next: Adding New Actions →](07_ADDING_NEW_ACTIONS.md)
