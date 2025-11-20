# Main Thread (Video Processing) - Overview

## 🎯 Purpose

The **Main Thread** is the primary execution thread responsible for all video processing operations in the Interview Behavior Analysis System. It handles real-time video capture, pose estimation, action detection, and user interface rendering.

## 🏗️ Architecture Position

```
┌─────────────────────────────────────────────────────────┐
│                    Application                          │
├─────────────────────────────────────────────────────────┤
│                                                           │
│  ┌────────────────────────┐   ┌──────────────────────┐ │
│  │    MAIN THREAD         │   │    STT THREAD        │ │
│  │  (Video Processing)    │   │  (Audio Processing)  │ │
│  │  ← YOU ARE HERE        │   │                      │ │
│  └────────────────────────┘   └──────────────────────┘ │
│           │                            │                │
│           └──────── Shared State ──────┘                │
└─────────────────────────────────────────────────────────┘
```

## 📦 Core Responsibilities

### 1. Video Capture
- Initialize and manage webcam connection
- Continuously read frames at 30 FPS
- Handle camera errors and disconnections
- Manage timing and frame synchronization

### 2. Pose Estimation
- Run YOLO11 Pose model inference on each frame
- Extract 17 COCO keypoints per detected person
- Handle multiple people in frame
- Manage inference device (CPU/GPU/MPS)

### 3. Action Detection
- Analyze pose keypoints to detect body language
- Apply geometric heuristics for 10 action types
- Track temporal changes (e.g., fidgeting)
- Deduplicate simultaneous actions

### 4. User Interface
- Draw action labels on video frames
- Display speech subtitles from STT thread
- Render bounding boxes (optional)
- Show keypoints (optional)

### 5. Event Logging
- Record detected actions with timestamps
- Maintain synchronized timeline
- Generate JSON logs on exit

### 6. User Input
- Monitor keyboard input
- Handle exit command ('q' key)
- Trigger graceful shutdown

## 🔄 Processing Pipeline

```
┌─────────────────────────────────────────────────────────┐
│                   Main Thread Loop                      │
└─────────────────────────────────────────────────────────┘
                         │
                         ↓
         ┌───────────────────────────┐
         │  1. Capture Frame         │ ← OpenCV VideoCapture
         └───────────────────────────┘
                         │
                         ↓
         ┌───────────────────────────┐
         │  2. Calculate Timestamp   │ ← time.time() - start_time
         └───────────────────────────┘
                         │
                         ↓
         ┌───────────────────────────┐
         │  3. YOLO Inference        │ ← model(frame)
         └───────────────────────────┘
                         │
                         ↓
         ┌───────────────────────────┐
         │  4. Extract Keypoints     │ ← results.keypoints.xy
         └───────────────────────────┘
                         │
                         ↓
         ┌───────────────────────────┐
         │  5. Detect Actions        │ ← detect_custom_actions()
         └───────────────────────────┘
                         │
                         ↓
         ┌───────────────────────────┐
         │  6. Log Actions           │ ← action_logs.append()
         └───────────────────────────┘
                         │
                         ↓
         ┌───────────────────────────┐
         │  7. Draw UI Elements      │ ← cv2.putText()
         └───────────────────────────┘
                         │
                         ↓
         ┌───────────────────────────┐
         │  8. Display Frame         │ ← cv2.imshow()
         └───────────────────────────┘
                         │
                         ↓
         ┌───────────────────────────┐
         │  9. Check Exit (q key)    │ ← cv2.waitKey()
         └───────────────────────────┘
                         │
                    Loop or Exit
```

## ⏱️ Timing Breakdown

Typical timing for one frame cycle (CPU):

| Step | Operation | Time | Percentage |
|------|-----------|------|------------|
| 1 | Frame Capture | 10 ms | 10% |
| 2 | Timestamp Calc | <1 ms | <1% |
| 3 | YOLO Inference | 80 ms | 80% |
| 4 | Keypoint Extract | 1 ms | 1% |
| 5 | Action Detection | 1 ms | 1% |
| 6 | Logging | <1 ms | <1% |
| 7 | UI Drawing | 5 ms | 5% |
| 8 | Display | 1 ms | 1% |
| 9 | Input Check | 1 ms | 1% |
| **Total** | **Per Frame** | **~100 ms** | **100%** |

**Resulting Frame Rate**: ~10 FPS (on CPU)

**Note**: On GPU, YOLO inference drops to ~20ms, achieving ~30 FPS

## 🧵 Threading Context

### Main Thread Characteristics

**Type**: Primary application thread (not daemon)

**Blocking Operations**:
- `cv2.VideoCapture.read()` - Blocks until frame ready (~10ms)
- `model(frame)` - Blocks during inference (~80ms CPU)
- `cv2.imshow()` - Blocks briefly (~1ms)
- `cv2.waitKey(1)` - Blocks for 1ms checking input

**Non-Blocking Operations**:
- Timestamp calculations
- Keypoint extraction
- Action detection logic
- Logging to list

### Synchronization with STT Thread

The Main Thread coordinates with the STT (Speech-to-Text) Thread through shared variables:

```python
# Main Thread WRITES, STT Thread READS
start_time = None          # Set on first frame
stop_flag = False          # Set when user exits
action_logs = []           # Appended during processing

# STT Thread WRITES, Main Thread READS  
current_subtitle = ""      # Latest speech text
speech_logs = []           # Appended by STT
```

**Synchronization Points**:

1. **Startup**: Main thread sets `start_time` → STT thread starts recording
2. **Runtime**: Main thread reads `current_subtitle` → Displays on frame
3. **Shutdown**: Main thread sets `stop_flag` → STT thread exits

**Thread Safety**: Python GIL (Global Interpreter Lock) provides automatic safety for simple variable access

## 📊 Performance Characteristics

### Resource Usage

**CPU**: 
- Single-threaded execution
- 40-60% of one core (typical)
- Spikes to 100% during YOLO inference

**Memory**:
- YOLO Model: ~90 MB (yolo11m-pose)
- Frame Buffers: ~10 MB (1080p)
- OpenCV: ~50 MB
- Logs: <1 MB (grows linearly with time)
- **Total**: ~150 MB

**GPU** (if enabled):
- VRAM: ~500 MB (includes model and buffers)
- GPU Utilization: 20-30% (intermittent)

### Bottlenecks

**Primary Bottleneck**: YOLO Inference (80% of cycle time)

**Solutions**:
- Use GPU (`device="cuda"` or `device="mps"`)
- Use smaller model (`yolov8n-pose.pt`)
- Use ONNX Runtime (`run_pose_onnx_dml.py`)
- Reduce frame rate (process every Nth frame)

**Secondary Bottleneck**: Frame Capture (10% of cycle time)

**Solutions**:
- Use higher-performance camera
- Reduce resolution (640x480 vs 1920x1080)
- Use video file instead of live camera

## 🔌 System Integration

### Inputs

| Input | Source | Type | Frequency |
|-------|--------|------|-----------|
| Video frames | Webcam | Image (BGR) | 30 FPS |
| Speech text | STT Thread | String | ~1 per 4s |
| User input | Keyboard | Key press | Event-driven |

### Outputs

| Output | Destination | Type | Frequency |
|--------|-------------|------|-----------|
| Annotated video | Display window | Image | 30 FPS |
| Action logs | Memory → JSON | List of dicts | Per detection |
| Console output | Terminal | Text | On events |

### State Management

**Global State Variables**:
```python
model = YOLO("yolo11m-pose.pt")   # Loaded once at startup
prev_left_wrist = None             # Updated every frame
prev_right_wrist = None            # Updated every frame
action_logs = []                   # Grows throughout execution
start_time = None                  # Set once on first frame
stop_flag = False                  # Set once on exit
current_subtitle = ""              # Updated by STT thread
```

## 📁 Code Location

The Main Thread logic is implemented in two files:

### 1. interview_system.py (Lines 198-321)
```python
def main():
    """Full implementation with STT thread"""
    # Start STT thread
    stt_thread = threading.Thread(target=stt_worker, daemon=True)
    stt_thread.start()
    
    # Open webcam
    cap = cv2.VideoCapture(0)
    
    # Main processing loop
    while True:
        # ... video processing logic ...
        pass
    
    # Cleanup and log generation
    # ...
```

### 2. action_detector3.py (Lines 144-206)
```python
# Simplified version (no main() function wrapper)
cap = cv2.VideoCapture(0)

while True:
    # ... video processing logic ...
    pass

# Cleanup
cap.release()
cv2.destroyAllWindows()
```

**Key Difference**: `interview_system.py` includes STT thread coordination, while `action_detector3.py` is single-threaded.

## 🎛️ Configuration

### Model Selection
```python
# Current (balanced)
model = YOLO("yolo11m-pose.pt")

# Faster (less accurate)
model = YOLO("yolov8n-pose.pt")

# Slower (more accurate)
model = YOLO("yolo11l-pose.pt")
```

### Device Selection
```python
# CPU (default, works everywhere)
results = model(frame, device="cpu")

# NVIDIA GPU (requires CUDA)
results = model(frame, device="cuda")

# Apple Silicon GPU (requires macOS)
results = model(frame, device="mps")
```

### Video Source
```python
# Default webcam
cap = cv2.VideoCapture(0)

# External USB camera
cap = cv2.VideoCapture(1)

# Video file (for testing)
cap = cv2.VideoCapture("interview.mp4")

# Network stream
cap = cv2.VideoCapture("rtsp://example.com/stream")
```

## 🔄 Lifecycle

### 1. Initialization Phase
```python
# Load YOLO model (~2-3 seconds)
model = YOLO("yolo11m-pose.pt")

# Initialize global variables
prev_left_wrist = None
action_logs = []
start_time = None

# Open webcam (~1-2 seconds)
cap = cv2.VideoCapture(0)
```

### 2. Processing Phase
```python
while True:
    # Read, process, display (repeated at ~10-30 FPS)
    ret, frame = cap.read()
    # ... processing ...
    cv2.imshow("Window", frame)
    
    # Check for exit
    if cv2.waitKey(1) & 0xFF == ord('q'):
        break
```

### 3. Cleanup Phase
```python
# Release resources
cap.release()
cv2.destroyAllWindows()

# Stop other threads
stop_flag = True
time.sleep(0.5)

# Generate and save logs
# ... JSON file generation ...
```

## 🎯 Design Decisions

### Why Main Thread for Video Processing?

**Reasoning**:
- Video processing requires UI updates (must be main thread)
- OpenCV `imshow()` requires main thread context
- Keyboard input via `waitKey()` requires main thread
- Python's tkinter/Qt integration requires main thread

**Alternatives Considered**:
- ❌ Process video in background thread: UI updates would fail
- ❌ Use multiprocessing: Higher overhead, complex state sharing
- ✅ Main thread video + background thread audio: Best balance

### Why Synchronous Processing?

**Reasoning**:
- Simple, sequential logic (easy to understand)
- No race conditions on frame data
- Deterministic execution order
- Easier debugging

**Alternatives Considered**:
- ❌ Async/await: Unnecessary complexity for this use case
- ❌ Frame queue with worker threads: Complicates synchronization
- ✅ Synchronous loop: Optimal for real-time processing

### Why 1ms waitKey()?

**Reasoning**:
```python
if cv2.waitKey(1) & 0xFF == ord('q'):
    break
```
- `waitKey(1)` processes UI events and checks input
- 1ms is minimal delay (non-blocking feel)
- Allows ~1000 FPS theoretical max (not actual limit)
- Essential for OpenCV window to remain responsive

**What happens if waitKey(0)**:
- Window freezes (waits indefinitely for key press)
- No video playback

## 📈 Scalability

### Single Person
**Current Implementation**: Optimized for one person
- All actions detected and logged
- Performance: ~10-30 FPS

### Multiple People
**Current Implementation**: Partial support
- YOLO detects all people
- Actions detected for all people
- **Limitation**: No person tracking (actions not attributed to specific person)

**To Add Person Tracking**:
1. Implement person ID assignment (e.g., by bounding box position)
2. Maintain separate action logs per person
3. Update logging structure to include person ID

### Long Sessions
**Current Implementation**: Suitable for sessions up to 1 hour
- Memory grows linearly with logged events
- JSON generation happens on exit (may take a few seconds for large logs)

**For Very Long Sessions**:
- Implement periodic log flushing
- Use chunked JSON writing
- Add log rotation

## 🔍 Debugging

### Enable YOLO Verbose Output
```python
results = model(frame, device="cpu", verbose=True)
```
Shows detection confidence and timing information.

### Print Frame Timing
```python
import time
start = time.time()
results = model(frame, device="cpu", verbose=False)
print(f"YOLO inference: {(time.time() - start)*1000:.1f}ms")
```

### Visualize Keypoints
```python
for r in results:
    for person in r.keypoints.xy:
        kp = person.cpu().numpy()
        for i, (x, y) in enumerate(kp):
            cv2.circle(frame, (int(x), int(y)), 5, (0, 255, 0), -1)
            cv2.putText(frame, str(i), (int(x), int(y)), 
                       cv2.FONT_HERSHEY_SIMPLEX, 0.5, (255, 255, 255), 1)
```

### Check Action Detection
```python
actions = detect_custom_actions(kp)
if actions:
    print(f"[{time.time()-start_time:.1f}s] Detected: {actions}")
```

## 📚 Further Reading

- **[02_VIDEO_CAPTURE_PIPELINE.md](02_VIDEO_CAPTURE_PIPELINE.md)** - Video capture details
- **[03_YOLO_MODEL_INFERENCE.md](03_YOLO_MODEL_INFERENCE.md)** - YOLO inference deep dive
- **[05_ACTION_DETECTION_ENGINE.md](05_ACTION_DETECTION_ENGINE.md)** - Action detection algorithms
- **[09_THREADING_AND_SYNC.md](09_THREADING_AND_SYNC.md)** - Thread coordination details

---

← [Back to Main Thread Docs](00_README.md) | [Next: Video Capture Pipeline →](02_VIDEO_CAPTURE_PIPELINE.md)
