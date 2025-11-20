# Video Capture Pipeline

## 🎯 Overview

The Video Capture Pipeline is the entry point of the Main Thread, responsible for initializing the camera, capturing frames, and managing the video stream. This document explains the capture process, configuration options, and troubleshooting.

## 🔄 Capture Pipeline

```
Camera Hardware
      ↓
┌──────────────────────┐
│ OpenCV VideoCapture  │ ← cv2.VideoCapture(0)
└──────────────────────┘
      ↓
┌──────────────────────┐
│ Frame Buffer         │ ← Internal buffering
└──────────────────────┘
      ↓
┌──────────────────────┐
│ cap.read()           │ ← Read frame
└──────────────────────┘
      ↓
Frame (NumPy array, BGR format)
```

## 📦 Initialization

### Basic Initialization

```python
import cv2

# Open default camera (device 0)
cap = cv2.VideoCapture(0)

# Check if opened successfully
if not cap.isOpened():
    print("Error: cannot open camera.")
    exit()

# Camera is ready to use
```

**What Happens**:
1. OpenCV connects to camera driver
2. Camera hardware initialized
3. Default settings applied
4. Buffer allocated for frames

**Time**: ~1-2 seconds

### Device Selection

```python
# Default camera (usually built-in webcam)
cap = cv2.VideoCapture(0)

# External USB camera (second camera)
cap = cv2.VideoCapture(1)

# Third camera
cap = cv2.VideoCapture(2)

# By device path (Linux)
cap = cv2.VideoCapture("/dev/video0")

# Network camera (RTSP stream)
cap = cv2.VideoCapture("rtsp://192.168.1.100:554/stream")

# Video file (for testing)
cap = cv2.VideoCapture("interview_recording.mp4")
```

### Camera Properties Configuration

```python
# Must be set BEFORE reading first frame
cap = cv2.VideoCapture(0)

# Resolution
cap.set(cv2.CAP_PROP_FRAME_WIDTH, 1280)
cap.set(cv2.CAP_PROP_FRAME_HEIGHT, 720)

# Frame rate
cap.set(cv2.CAP_PROP_FPS, 30)

# Auto exposure
cap.set(cv2.CAP_PROP_AUTO_EXPOSURE, 0.75)  # 0.75 = auto, 0.25 = manual

# Exposure value (if manual)
cap.set(cv2.CAP_PROP_EXPOSURE, -6)

# Focus
cap.set(cv2.CAP_PROP_AUTOFOCUS, 1)  # 1 = auto, 0 = manual

# Brightness (0-255)
cap.set(cv2.CAP_PROP_BRIGHTNESS, 128)

# Contrast (0-255)
cap.set(cv2.CAP_PROP_CONTRAST, 128)

# Saturation (0-255)
cap.set(cv2.CAP_PROP_SATURATION, 128)

# Buffer size (reduce latency)
cap.set(cv2.CAP_PROP_BUFFERSIZE, 1)
```

**Verify Settings**:
```python
# Read back actual values
actual_width = cap.get(cv2.CAP_PROP_FRAME_WIDTH)
actual_height = cap.get(cv2.CAP_PROP_FRAME_HEIGHT)
actual_fps = cap.get(cv2.CAP_PROP_FPS)

print(f"Resolution: {actual_width}x{actual_height}")
print(f"FPS: {actual_fps}")
```

**Note**: Not all cameras support all properties. Some settings may be ignored.

## 📸 Frame Capture

### Reading a Frame

```python
ret, frame = cap.read()

# ret: Boolean - True if frame captured successfully, False otherwise
# frame: NumPy array (H, W, 3) - BGR image, or None if ret is False
```

**Frame Structure**:
```python
# Shape
print(frame.shape)  # e.g., (720, 1280, 3) for 720p

# Data type
print(frame.dtype)  # uint8 (values 0-255)

# Color format
# frame[:, :, 0] = Blue channel
# frame[:, :, 1] = Green channel
# frame[:, :, 2] = Red channel
```

### Capture Loop Pattern

```python
# Standard pattern used in interview_system.py
while True:
    ret, frame = cap.read()
    if not ret:
        print("Error: cannot read frame.")
        break
    
    # Process frame here
    # ...
    
    # Display
    cv2.imshow("Window", frame)
    
    # Exit on 'q' key
    if cv2.waitKey(1) & 0xFF == ord('q'):
        break

# Cleanup
cap.release()
cv2.destroyAllWindows()
```

### Error Handling

```python
def read_frame_safe(cap, max_retries=3):
    """Read frame with retry logic"""
    for attempt in range(max_retries):
        ret, frame = cap.read()
        if ret:
            return True, frame
        
        print(f"Frame read failed, retry {attempt + 1}/{max_retries}")
        time.sleep(0.1)
    
    return False, None

# Usage
ret, frame = read_frame_safe(cap)
if not ret:
    print("Failed to read frame after retries")
    break
```

## ⏱️ Timing and Synchronization

### First Frame Synchronization

In `interview_system.py`:

```python
start_time = None  # Global variable

# In main loop
ret, frame = cap.read()
if not ret:
    break

# Set start_time on first successful frame
if start_time is None:
    start_time = time.time()
    print("[MAIN] Timing started from first frame.")
```

**Why First Frame?**
- Camera may take 100-200ms to produce first frame
- Ensures both video and audio start from same time reference
- Avoids negative timestamps in logs

### Frame Timestamps

```python
# Calculate elapsed time
ts = time.time() - start_time

# Format for display
mm = int(ts // 60)  # Minutes
ss = int(ts % 60)   # Seconds
ms = int((ts % 1) * 1000)  # Milliseconds

time_str = f"{mm:02d}:{ss:02d}.{ms:03d}"
```

### Frame Rate Measurement

```python
import time

frame_count = 0
start_time = time.time()

while True:
    ret, frame = cap.read()
    if not ret:
        break
    
    frame_count += 1
    
    # Calculate FPS every second
    if frame_count % 30 == 0:
        elapsed = time.time() - start_time
        fps = frame_count / elapsed
        print(f"FPS: {fps:.1f}")
    
    # ... rest of processing ...
```

## 🎛️ Camera Settings Presets

### Preset 1: Low Latency

```python
cap = cv2.VideoCapture(0)
cap.set(cv2.CAP_PROP_FRAME_WIDTH, 640)
cap.set(cv2.CAP_PROP_FRAME_HEIGHT, 480)
cap.set(cv2.CAP_PROP_FPS, 30)
cap.set(cv2.CAP_PROP_BUFFERSIZE, 1)  # Minimal buffer
```

**Use Case**: Real-time interaction, gesture control

### Preset 2: High Quality

```python
cap = cv2.VideoCapture(0)
cap.set(cv2.CAP_PROP_FRAME_WIDTH, 1920)
cap.set(cv2.CAP_PROP_FRAME_HEIGHT, 1080)
cap.set(cv2.CAP_PROP_FPS, 30)
cap.set(cv2.CAP_PROP_AUTO_EXPOSURE, 0.75)
```

**Use Case**: Recording interviews, analysis

### Preset 3: Low Light

```python
cap = cv2.VideoCapture(0)
cap.set(cv2.CAP_PROP_AUTO_EXPOSURE, 0.25)  # Manual
cap.set(cv2.CAP_PROP_EXPOSURE, -2)  # Higher exposure
cap.set(cv2.CAP_PROP_BRIGHTNESS, 160)
cap.set(cv2.CAP_PROP_ISO_SPEED, 800)  # If supported
```

**Use Case**: Dimly lit rooms

### Preset 4: Outdoor/Bright

```python
cap = cv2.VideoCapture(0)
cap.set(cv2.CAP_PROP_AUTO_EXPOSURE, 0.75)
cap.set(cv2.CAP_PROP_EXPOSURE, -8)  # Lower exposure
cap.set(cv2.CAP_PROP_BRIGHTNESS, 100)
```

**Use Case**: Bright environments, sunlight

## 🔍 Frame Processing Considerations

### Color Space Conversion

OpenCV captures in BGR, but YOLO expects RGB:

```python
# OpenCV frame is BGR
ret, frame = cap.read()

# YOLO handles conversion internally
results = model(frame)  # Automatic BGR→RGB

# Manual conversion (if needed)
frame_rgb = cv2.cvtColor(frame, cv2.COLOR_BGR2RGB)
```

### Frame Preprocessing

```python
def preprocess_frame(frame):
    """Optional preprocessing"""
    
    # Resize (if needed for performance)
    frame = cv2.resize(frame, (640, 480))
    
    # Denoise (if camera is noisy)
    frame = cv2.fastNlMeansDenoisingColored(frame, None, 10, 10, 7, 21)
    
    # Adjust brightness/contrast (if needed)
    frame = cv2.convertScaleAbs(frame, alpha=1.2, beta=30)
    
    return frame

# Usage
ret, frame = cap.read()
frame = preprocess_frame(frame)
results = model(frame)
```

**Note**: Preprocessing adds latency. Only use if necessary.

## 📊 Performance Optimization

### Reduce Frame Read Time

```python
# Use threading for frame capture
import threading
from queue import Queue

frame_queue = Queue(maxsize=2)

def capture_frames(cap, queue):
    while not stop_flag:
        ret, frame = cap.read()
        if ret:
            if not queue.full():
                queue.put(frame)

# Start capture thread
cap_thread = threading.Thread(target=capture_frames, args=(cap, frame_queue))
cap_thread.start()

# Main thread reads from queue
while True:
    frame = frame_queue.get()
    # Process frame...
```

**Benefit**: Camera I/O doesn't block processing

### Skip Frames for Speed

```python
frame_count = 0
PROCESS_EVERY_N = 2  # Process every 2nd frame

while True:
    ret, frame = cap.read()
    if not ret:
        break
    
    frame_count += 1
    
    # Only process every Nth frame
    if frame_count % PROCESS_EVERY_N == 0:
        results = model(frame)
        # ... action detection ...
    
    # Still display all frames (smooth video)
    cv2.imshow("Window", frame)
```

## 🐛 Troubleshooting

### Issue: Camera Not Opening

```python
cap = cv2.VideoCapture(0)
if not cap.isOpened():
    # Try different device indices
    for i in range(5):
        cap = cv2.VideoCapture(i)
        if cap.isOpened():
            print(f"Found camera at index {i}")
            break
    else:
        print("No camera found")
        exit()
```

### Issue: Low Frame Rate

**Causes**:
1. Processing too slow (YOLO bottleneck)
2. High resolution camera
3. Poor USB connection
4. Camera settings

**Solutions**:
```python
# Reduce resolution
cap.set(cv2.CAP_PROP_FRAME_WIDTH, 640)
cap.set(cv2.CAP_PROP_FRAME_HEIGHT, 480)

# Use GPU for YOLO
results = model(frame, device="cuda")

# Skip frames
if frame_count % 2 == 0:
    results = model(frame)
```

### Issue: Delayed Video

**Cause**: Frame buffering

**Solution**:
```python
# Reduce buffer size
cap.set(cv2.CAP_PROP_BUFFERSIZE, 1)

# Or flush buffer
for _ in range(5):
    cap.read()  # Discard buffered frames
```

### Issue: Poor Image Quality

```python
# Increase resolution
cap.set(cv2.CAP_PROP_FRAME_WIDTH, 1280)
cap.set(cv2.CAP_PROP_FRAME_HEIGHT, 720)

# Adjust exposure
cap.set(cv2.CAP_PROP_AUTO_EXPOSURE, 0.75)

# Increase sharpness (if supported)
cap.set(cv2.CAP_PROP_SHARPNESS, 200)
```

## 🧪 Testing Camera Configuration

```python
def test_camera_config():
    """Test and display camera configuration"""
    
    cap = cv2.VideoCapture(0)
    
    properties = {
        "Width": cv2.CAP_PROP_FRAME_WIDTH,
        "Height": cv2.CAP_PROP_FRAME_HEIGHT,
        "FPS": cv2.CAP_PROP_FPS,
        "Brightness": cv2.CAP_PROP_BRIGHTNESS,
        "Contrast": cv2.CAP_PROP_CONTRAST,
        "Saturation": cv2.CAP_PROP_SATURATION,
        "Exposure": cv2.CAP_PROP_EXPOSURE,
        "Autofocus": cv2.CAP_PROP_AUTOFOCUS,
    }
    
    print("\n=== Camera Configuration ===")
    for name, prop in properties.items():
        value = cap.get(prop)
        print(f"{name:12s}: {value}")
    
    # Test frame capture
    ret, frame = cap.read()
    if ret:
        print(f"\nFrame shape: {frame.shape}")
        print(f"Frame dtype: {frame.dtype}")
        print(f"Frame size: {frame.nbytes / 1024:.1f} KB")
    
    cap.release()

test_camera_config()
```

## 📚 Further Reading

- **[01_MAIN_THREAD_OVERVIEW.md](01_MAIN_THREAD_OVERVIEW.md)** - Main thread architecture
- **[03_YOLO_MODEL_INFERENCE.md](03_YOLO_MODEL_INFERENCE.md)** - Processing captured frames
- **[06_CONFIGURING_DETECTIONS.md](06_CONFIGURING_DETECTIONS.md)** - Camera-dependent settings

---

← [Previous: Main Thread Overview](01_MAIN_THREAD_OVERVIEW.md) | [Back to Main Thread Docs](00_README.md) | [Next: YOLO Model Inference →](03_YOLO_MODEL_INFERENCE.md)
