# Code Components - Detailed File Explanation

## 📂 Repository Structure

```
emotion-Recognization/
├── interview_system.py       # Main application - Full system
├── action_detector3.py       # Standalone action detector
├── run_pose_onnx_dml.py     # ONNX-optimized inference
├── export_pose.py           # Model export utility
├── yolo11m-pose.pt          # YOLO11 PyTorch model (84 MB)
├── yolo11m-pose.onnx        # YOLO11 ONNX model (84 MB)
├── yolov8n-pose.pt          # YOLO8 lightweight model (6.5 MB)
├── Readme.md                # Basic project README
└── docs/                    # Complete documentation
```

## 1️⃣ interview_system.py

**Purpose**: Main application combining pose detection and speech recognition

**Lines of Code**: 325 lines

**Key Features**:
- Multi-threaded processing (video + audio)
- Real-time pose detection with YOLO
- Real-time speech transcription with Whisper
- Synchronized logging
- On-screen visualization

### Code Structure

```python
# Lines 1-8: Imports
import cv2, numpy, time, json, threading
import sounddevice as sd
from faster_whisper import WhisperModel
from ultralytics import YOLO

# Lines 10-14: Model Initialization
model = YOLO("yolo11m-pose.pt")

# Lines 16-31: Global Variables
prev_left_wrist = None       # For fidget detection
prev_right_wrist = None
action_logs = []             # Action timeline
speech_logs = []             # Speech timeline
start_time = None            # Synchronization timestamp
stop_flag = False            # Thread coordination
current_subtitle = ""        # Latest speech text

# Lines 34-124: Action Detection Function
def detect_custom_actions(kp):
    """
    Analyzes 17 pose keypoints to detect 10 body actions
    
    Parameters:
        kp: numpy array of shape (17, 2) with [x, y] coordinates
        
    Returns:
        list of detected action names (e.g., ["arms_crossed", "lean_forward"])
    """
    # Extract keypoint positions
    nose = kp[0]
    left_eye, right_eye = kp[1], kp[2]
    # ... (17 keypoints total)
    
    # Calculate geometric features
    shoulder_center = (l_shoulder + r_shoulder) / 2
    hip_center = (l_hip + r_hip) / 2
    torso_height = distance(shoulder_center, hip_center)
    
    # Apply heuristic rules
    if distance(l_wrist, r_elbow) < 80 and distance(r_wrist, l_elbow) < 80:
        actions.append("arms_crossed")
    # ... (10 detection rules total)
    
    return list(set(actions))  # Remove duplicates

# Lines 130-192: Speech-to-Text Worker Thread
def stt_worker():
    """
    Background thread that continuously:
    1. Records audio from microphone
    2. Transcribes with Whisper
    3. Logs recognized text
    """
    # Initialize Whisper model
    model_stt = WhisperModel("tiny", device="cpu")
    
    # Wait for video to start
    while start_time is None and not stop_flag:
        time.sleep(0.1)
    
    # Main loop
    while not stop_flag:
        # Record 4 seconds
        audio = sd.rec(int(4 * 16000), samplerate=16000, channels=1)
        sd.wait()
        
        # Transcribe
        segments, _ = model_stt.transcribe(audio.flatten(), language="en")
        
        # Log results
        for seg in segments:
            speech_logs.append({
                "time": f"{mm:02d}:{ss:02d}",
                "timestamp_seconds": round(ts, 2),
                "text": seg.text.strip()
            })

# Lines 198-321: Main Video Processing
def main():
    """
    Main function that:
    1. Starts STT thread
    2. Opens webcam
    3. Processes video frames
    4. Generates logs on exit
    """
    # Start audio processing thread
    stt_thread = threading.Thread(target=stt_worker, daemon=True)
    stt_thread.start()
    
    # Open webcam
    cap = cv2.VideoCapture(0)
    
    # Main loop
    while True:
        ret, frame = cap.read()
        
        # Set timing on first frame
        if start_time is None:
            start_time = time.time()
        
        # Calculate elapsed time
        ts = time.time() - start_time
        
        # Run YOLO pose detection
        results = model(frame, device="cpu", verbose=False)
        
        # Analyze detected poses
        for r in results:
            for person in r.keypoints.xy:
                kp = person.cpu().numpy()
                actions = detect_custom_actions(kp)
                
                # Log actions
                if actions:
                    action_logs.append({
                        "time": f"{mm:02d}:{ss:02d}",
                        "timestamp_seconds": round(ts, 2),
                        "actions": actions
                    })
        
        # Draw UI
        for act in actions:
            cv2.putText(frame, f"ACTION: {act}", ...)
        cv2.putText(frame, current_subtitle, ...)
        
        # Display
        cv2.imshow("Interview Action + Speech Monitor", frame)
        
        # Check exit
        if cv2.waitKey(1) & 0xFF == ord('q'):
            break
    
    # Cleanup
    cap.release()
    cv2.destroyAllWindows()
    stop_flag = True
    
    # Generate JSON logs
    save_action_log()
    save_speech_log()
    save_combined_log()

# Lines 323-325: Entry Point
if __name__ == "__main__":
    main()
```

### Key Functions

#### `distance(p1, p2)`
```python
def distance(p1, p2):
    return np.linalg.norm(np.array(p1) - np.array(p2))
```
- **Purpose**: Calculate Euclidean distance between two points
- **Parameters**: Two points as tuples/lists `[x, y]`
- **Returns**: Float distance in pixels
- **Usage**: Used in all action detection logic

#### `detect_custom_actions(kp)`
- **Purpose**: Main action detection logic
- **Input**: 17 keypoints array (shape: 17×2)
- **Output**: List of action strings
- **Complexity**: O(1) - Fixed number of checks
- **Details**: See [Action Detection Algorithms](06_ACTION_DETECTION.md)

#### `stt_worker()`
- **Purpose**: Background audio processing
- **Thread**: Runs in separate thread
- **Blocks**: Yes (4 seconds per audio chunk)
- **Synchronization**: Uses `start_time` and `stop_flag`

#### `main()`
- **Purpose**: Main control flow
- **Thread**: Primary thread
- **Responsibilities**: Video, UI, orchestration
- **Exit**: On 'q' key press

### Important Variables

#### Global State
```python
start_time = None      # float: Unix timestamp of first frame
stop_flag = False      # bool: Signals threads to stop
action_logs = []       # list: All detected actions
speech_logs = []       # list: All transcribed text
current_subtitle = ""  # str: Latest speech (for display)
```

#### Detection State
```python
prev_left_wrist = None   # array: Last frame's left wrist position
prev_right_wrist = None  # array: Last frame's right wrist position
```
Used to detect rapid hand movements (fidgeting).

### Configuration Constants

```python
SAMPLE_RATE = 16000      # Whisper requirement
CHUNK_SECONDS = 4        # Audio chunk duration
model_size = "tiny"      # Whisper model: tiny/base/small/medium/large
language = "en"          # Speech language
beam_size = 1            # Whisper decoding: 1=greedy (fast)
```

### Outputs

Three JSON files generated on exit:

1. **action_log.json**: All detected actions with timestamps
2. **transcription_log.json**: All speech segments with timestamps
3. **combined_log.json**: Merged timeline (actions + speech per second)

---

## 2️⃣ action_detector3.py

**Purpose**: Simplified version - pose detection only (no speech)

**Lines of Code**: 207 lines

**Key Features**:
- Single-threaded (simpler)
- Faster startup (no Whisper loading)
- Same action detection logic
- Outputs only `action_log.json`

### Differences from interview_system.py

| Feature | interview_system.py | action_detector3.py |
|---------|-------------------|-------------------|
| Speech Recognition | ✅ Yes | ❌ No |
| Multi-threading | ✅ Yes | ❌ No |
| Startup Time | ~10 seconds | ~3 seconds |
| Memory Usage | ~225 MB | ~140 MB |
| Outputs | 3 JSON files | 1 JSON file |
| Microphone | Required | Not required |

### Code Structure

```python
# Lines 1-5: Imports (no sounddevice, no faster_whisper, no threading)
import cv2, numpy, time, json
from ultralytics import YOLO

# Lines 9: Model Initialization
model = YOLO("yolo11m-pose.pt")

# Lines 12-14: Utility Function
def distance(p1, p2):
    return np.linalg.norm(np.array(p1) - np.array(p2))

# Lines 17-18: Detection State
prev_left_wrist = None
prev_right_wrist = None

# Lines 21-137: Action Detection Function
def detect_custom_actions(kp):
    # Identical to interview_system.py
    # See that file for details

# Lines 144-206: Main Loop (no function wrapper)
cap = cv2.VideoCapture(0)
start_time = None
event_logs = []

while True:
    ret, frame = cap.read()
    
    if start_time is None:
        start_time = time.time()
    
    current_time = time.time() - start_time
    results = model(frame, device="cpu", verbose=False)
    
    for r in results:
        for person in r.keypoints.xy:
            kp = person.cpu().numpy()
            actions = detect_custom_actions(kp)
            
            # Draw actions
            for act in actions:
                cv2.putText(frame, f"ACTION: {act}", ...)
            
            # Log actions
            if actions:
                event_logs.append({
                    "time": f"{mm:02d}:{ss:02d}",
                    "timestamp_seconds": round(current_time, 2),
                    "actions": actions
                })
    
    cv2.imshow("Interview Action Detector - 10 Actions", frame)
    
    if cv2.waitKey(1) & 0xFF == ord('q'):
        break

# Save logs
cap.release()
cv2.destroyAllWindows()

with open("action_log.json", "w", encoding="utf-8") as f:
    json.dump(event_logs, f, indent=4)
```

### When to Use This File

✅ **Use action_detector3.py when**:
- You only need body language analysis
- You don't have a microphone
- You want faster startup
- You want simpler code to understand/modify

❌ **Use interview_system.py when**:
- You need both body language and speech
- You want complete interview analysis
- You need synchronized action + speech data

---

## 3️⃣ run_pose_onnx_dml.py

**Purpose**: Hardware-accelerated pose detection using ONNX Runtime + DirectML

**Lines of Code**: 69 lines

**Key Features**:
- ONNX runtime instead of PyTorch
- DirectML support (Windows GPU acceleration)
- Lower-level inference code
- No action detection logic (just visualization)

### Code Structure

```python
# Lines 1-3: Imports
import cv2, numpy as np
import onnxruntime as ort

# Lines 8-12: Setup ONNX Runtime with DirectML
providers = [
    ("DmlExecutionProvider", {"device_id": 0}),  # Use GPU
    "CPUExecutionProvider"                        # Fallback to CPU
]
session = ort.InferenceSession("yolo11m-pose.onnx", providers=providers)
input_name = session.get_inputs()[0].name

# Lines 14-15: Verify Provider
print("Using providers:", session.get_providers())

# Lines 20-65: Main Loop
cap = cv2.VideoCapture(0)

while True:
    ret, frame = cap.read()
    
    # Preprocess
    img = cv2.resize(frame, (640, 640))
    img_input = img[:, :, ::-1] / 255.0           # BGR → RGB, normalize
    img_input = img_input.transpose(2, 0, 1)      # HWC → CHW
    img_input = np.expand_dims(img_input, axis=0) # Add batch dimension
    
    # ONNX Inference
    outputs = session.run(None, {input_name: img_input})[0]
    outputs = outputs[0]  # Shape: (56, 8400)
    
    # Parse YOLO11 Pose Output
    # Format: [bbox(4), score(1), class(1), keypoints(50)]
    scores = outputs[4]
    mask = scores > 0.4  # Filter low-confidence detections
    filtered = outputs[:, mask].T  # Shape: (N, 56)
    
    # Visualize
    for det in filtered:
        # Bounding box
        x, y, w, h = det[:4].astype(int)
        cv2.rectangle(frame, (x, y), (x+w, y+h), (0,255,0), 2)
        
        # Keypoints (25 points for YOLO11, but we use 17 COCO standard)
        kpts = det[6:].reshape(25, 2)
        for (kx, ky) in kpts.astype(int):
            cv2.circle(frame, (kx, ky), 3, (0,0,255), -1)
    
    cv2.imshow("ONNX Pose (DirectML)", frame)
    if cv2.waitKey(1) & 0xFF == ord('q'):
        break

cap.release()
cv2.destroyAllWindows()
```

### Key Differences from Other Files

| Aspect | interview_system.py | run_pose_onnx_dml.py |
|--------|-------------------|-------------------|
| Runtime | PyTorch | ONNX Runtime |
| Acceleration | CPU/CUDA/MPS | DirectML (Windows GPU) |
| Model Format | .pt (PyTorch) | .onnx |
| Action Detection | ✅ Yes | ❌ No |
| Speech Recognition | ✅ Yes | ❌ No |
| Preprocessing | Handled by YOLO | Manual (BGR→RGB, normalize, transpose) |
| Output Parsing | Handled by YOLO | Manual |
| Use Case | Complete analysis | Performance testing |

### DirectML Provider

**What is DirectML?**
- Microsoft's hardware acceleration API for Windows
- Works with AMD, NVIDIA, and Intel GPUs
- Part of Windows 10/11
- No CUDA installation needed

**Performance Comparison**:
- CPU: ~100ms per frame
- DirectML (GPU): ~20-30ms per frame
- CUDA (NVIDIA): ~15-25ms per frame

**Limitations**:
- Windows only
- No action detection implemented
- Requires manual preprocessing
- More complex code

### ONNX Model Format

**YOLO11 Pose ONNX Output**:
```
Shape: (1, 56, 8400)

For each of 8400 detection candidates:
- Index 0-3: Bounding box [x, y, w, h]
- Index 4: Confidence score
- Index 5: Class ID (always 0 for "person")
- Index 6-55: 25 keypoints × 2 (x, y) + confidence (not used in this code)
```

**Note**: Standard COCO pose has 17 keypoints, but YOLO11 outputs 25. This code visualizes all 25 but doesn't interpret them.

### When to Use This File

✅ **Use run_pose_onnx_dml.py when**:
- You're on Windows with GPU
- You want maximum performance
- You're testing ONNX deployment
- You don't need action detection (just raw pose)

❌ **Don't use when**:
- You need action detection logic
- You need speech recognition
- You're on macOS or Linux
- You prefer simpler code

---

## 4️⃣ export_pose.py

**Purpose**: Convert YOLO PyTorch model to ONNX format

**Lines of Code**: 5 lines

**Code**:
```python
from ultralytics import YOLO

model = YOLO("yolo11m-pose.pt")
model.export(format="onnx")
print("Exported ONNX model")
```

### What It Does

1. **Loads** PyTorch model (`yolo11m-pose.pt`)
2. **Exports** to ONNX format (`yolo11m-pose.onnx`)
3. **Validates** exported model works

### When to Use

✅ **Run this when**:
- You downloaded a new PyTorch model
- You want to use ONNX runtime
- You want to deploy to production
- You want cross-platform compatibility

### Export Options

You can export to other formats:
```python
# ONNX (cross-platform)
model.export(format="onnx")

# TensorRT (NVIDIA GPUs)
model.export(format="engine")

# CoreML (Apple devices)
model.export(format="coreml")

# TFLite (mobile devices)
model.export(format="tflite")

# OpenVINO (Intel hardware)
model.export(format="openvino")
```

### Export Parameters

```python
# Advanced export options
model.export(
    format="onnx",
    imgsz=640,           # Input image size
    half=False,          # FP16 quantization
    dynamic=False,       # Dynamic input shapes
    simplify=True,       # Simplify ONNX graph
    opset=12            # ONNX opset version
)
```

### Output

Creates `yolo11m-pose.onnx` (~84 MB) in the same directory.

---

## 📊 File Comparison Matrix

| Feature | interview_system.py | action_detector3.py | run_pose_onnx_dml.py | export_pose.py |
|---------|-------------------|-------------------|-------------------|--------------|
| **Purpose** | Full analysis | Actions only | Performance test | Model conversion |
| **Lines of Code** | 325 | 207 | 69 | 5 |
| **Runtime** | PyTorch | PyTorch | ONNX | PyTorch |
| **Threads** | Multi (2) | Single | Single | N/A |
| **Pose Detection** | ✅ | ✅ | ✅ | N/A |
| **Action Detection** | ✅ | ✅ | ❌ | N/A |
| **Speech Recognition** | ✅ | ❌ | ❌ | N/A |
| **GPU Support** | CUDA/MPS | CUDA/MPS | DirectML | N/A |
| **Microphone** | Required | Not required | Not required | Not required |
| **Startup Time** | ~10s | ~3s | ~2s | N/A |
| **Memory** | ~225 MB | ~140 MB | ~120 MB | N/A |
| **JSON Outputs** | 3 files | 1 file | None | None |
| **Complexity** | High | Medium | Medium | Low |
| **Best For** | Complete analysis | Quick testing | Performance | Deployment prep |

---

## 🔍 Code Reuse

Several code blocks are shared across files:

### Shared: Action Detection Logic
Files: `interview_system.py`, `action_detector3.py`

The `detect_custom_actions(kp)` function is **identical** in both files. This is intentional for:
- Code simplicity (no imports between files)
- Independent execution (each file runs standalone)
- Easy modification (change one without affecting other)

### Shared: Distance Calculation
Files: `interview_system.py`, `action_detector3.py`

```python
def distance(p1, p2):
    return np.linalg.norm(np.array(p1) - np.array(p2))
```

Also duplicated for same reasons.

### Not Shared: ONNX Preprocessing
File: `run_pose_onnx_dml.py`

ONNX runtime requires manual preprocessing:
```python
img_input = img[:, :, ::-1] / 255.0          # BGR → RGB, normalize
img_input = img_input.transpose(2, 0, 1)     # HWC → CHW  
img_input = np.expand_dims(img_input, axis=0) # Add batch
```

PyTorch YOLO handles this automatically.

---

## 💡 Customization Tips

### Add New Action Detection

1. Open `interview_system.py` or `action_detector3.py`
2. Find `detect_custom_actions()` function
3. Add new detection logic:
```python
# 11. Hand on chest
if distance(l_wrist, shoulder_center) < 50:
    actions.append("hand_on_chest")
```

### Change Video Source

```python
# Default webcam
cap = cv2.VideoCapture(0)

# External USB camera
cap = cv2.VideoCapture(1)

# Video file
cap = cv2.VideoCapture("interview.mp4")

# RTSP stream
cap = cv2.VideoCapture("rtsp://...")
```

### Change YOLO Model

```python
# Faster, less accurate
model = YOLO("yolov8n-pose.pt")

# Current (balanced)
model = YOLO("yolo11m-pose.pt")

# Slower, more accurate
model = YOLO("yolo11l-pose.pt")
```

### Enable GPU

```python
# In any file with YOLO
results = model(frame, device="cuda")  # NVIDIA
results = model(frame, device="mps")   # Apple Silicon
```

---

← [Previous: Installation Guide](04_INSTALLATION.md) | [Back to Documentation Home](00_README.md) | [Next: Action Detection Algorithms →](06_ACTION_DETECTION.md)
