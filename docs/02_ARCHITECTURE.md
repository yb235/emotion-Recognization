# System Architecture

## 🏗️ Architecture Overview

The system is built on a **multi-threaded, modular architecture** that processes video and audio streams in parallel. This document explains the technical design for developers and advanced users.

## 📐 High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        Main Application                          │
│                    (interview_system.py)                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌────────────────────┐         ┌──────────────────────┐       │
│  │  Main Thread       │         │   STT Thread         │       │
│  │  (Video + Pose)    │         │   (Audio + Speech)   │       │
│  │                    │         │                      │       │
│  │  1. Video Capture  │         │  1. Audio Recording  │       │
│  │  2. YOLO Inference │         │  2. Whisper Inference│       │
│  │  3. Action Logic   │         │  3. Speech Logging   │       │
│  │  4. Action Logging │         │                      │       │
│  │  5. UI Display     │         │                      │       │
│  └────────────────────┘         └──────────────────────┘       │
│           │                                │                    │
│           └────────────┬───────────────────┘                    │
│                        ▼                                        │
│              ┌──────────────────┐                              │
│              │  Shared State    │                              │
│              │  - start_time    │                              │
│              │  - stop_flag     │                              │
│              │  - action_logs   │                              │
│              │  - speech_logs   │                              │
│              │  - subtitle      │                              │
│              └──────────────────┘                              │
│                        │                                        │
│                        ▼                                        │
│              ┌──────────────────┐                              │
│              │  JSON Generator  │                              │
│              │  - action_log    │                              │
│              │  - speech_log    │                              │
│              │  - combined_log  │                              │
│              └──────────────────┘                              │
└─────────────────────────────────────────────────────────────────┘
```

## 🧵 Threading Model

### Why Multi-Threading?

The system uses **parallel processing** because:
1. **Video processing is frame-based** (30+ FPS ideal)
2. **Audio processing is continuous** (real-time recording)
3. **Both are computationally intensive** and would block each other
4. **User experience requires responsiveness** (display updates)

### Thread Structure

#### Main Thread (Video Processing)
**Purpose**: Handle video capture, pose estimation, and UI

**Responsibilities**:
- Open webcam via OpenCV
- Read frames continuously
- Run YOLO pose detection on each frame
- Analyze pose keypoints for actions
- Draw UI elements (action labels, subtitles)
- Handle keyboard input (quit on 'q')
- Generate final JSON logs on exit

**Blocking Operations**: 
- `cv2.VideoCapture.read()` - blocks until frame ready
- `model()` - YOLO inference takes ~50-100ms per frame

#### STT Thread (Audio Processing)
**Purpose**: Handle audio recording and speech transcription

**Responsibilities**:
- Wait for video to start (synchronization)
- Record audio in 4-second chunks
- Transcribe each chunk using Whisper
- Update shared subtitle variable
- Log transcribed text with timestamps

**Blocking Operations**:
- `sd.rec()` + `sd.wait()` - blocks for 4 seconds per chunk
- `model_stt.transcribe()` - Whisper inference takes ~1-3 seconds

### Thread Synchronization

**Shared Variables** (with Python GIL protection):
```python
start_time = None         # Set by main, read by STT
stop_flag = False         # Set by main, read by STT
current_subtitle = ""     # Set by STT, read by main
action_logs = []          # Appended by main
speech_logs = []          # Appended by STT
```

**Synchronization Points**:
1. **Startup**: STT thread waits for `start_time` to be set by main thread
2. **Shutdown**: Main thread sets `stop_flag`, STT thread exits gracefully
3. **Display**: Main thread reads `current_subtitle` for on-screen display

## 🤖 AI Models Architecture

### YOLO11m-Pose Model

**Purpose**: Detect human pose from images

**Architecture**:
- **Backbone**: CSPDarknet (feature extraction)
- **Neck**: PANet (multi-scale feature fusion)
- **Head**: Pose detection head (17 keypoints + bounding box)

**Input**: 
- Image tensor: (1, 3, 640, 640) - RGB format
- Normalized: [0, 1] range

**Output**:
- Bounding boxes: (x, y, w, h) for detected persons
- Keypoints: 17 (x, y, confidence) pairs per person
- Confidence scores: Detection confidence

**17 COCO Keypoints**:
```
0: nose           9: left_wrist      
1: left_eye      10: right_wrist
2: right_eye     11: left_hip
3: left_ear      12: right_hip
4: right_ear     13: left_knee
5: left_shoulder 14: right_knee
6: right_shoulder 15: left_ankle
7: left_elbow    16: right_ankle
8: right_elbow
```

**Inference Device**: CPU (can be changed to CUDA/MPS)

**Performance**: ~50-100ms per frame on CPU

### Faster-Whisper Model

**Purpose**: Convert speech to text

**Architecture**:
- Based on OpenAI Whisper
- Optimized with CTranslate2 (4x faster)
- Uses int8 quantization for speed

**Model Size**: `tiny` (39M parameters)
- Small size for fast inference
- Adequate accuracy for English
- Can be upgraded to `base`, `small`, `medium`, `large`

**Input**:
- Audio: 16kHz mono, float32
- Duration: Variable (system uses 4-second chunks)

**Output**:
- Segments: List of (start, end, text, confidence)
- Language: Auto-detected or specified (system uses "en")

**Inference Device**: CPU only (Whisper is CPU-optimized)

**Performance**: ~1-3 seconds per 4-second audio chunk

## 📦 Data Flow

### Video Processing Pipeline

```
Camera → Frame Capture → YOLO Model → Keypoints
    ↓
Keypoints → detect_custom_actions() → Actions List
    ↓
Actions List → Logging (with timestamp)
    ↓
Actions List → UI Display (on frame)
```

**Step-by-Step**:
1. **Capture**: `cv2.VideoCapture(0)` reads frame from webcam
2. **Inference**: `model(frame)` runs YOLO on frame
3. **Extraction**: `r.keypoints.xy` extracts (x, y) coordinates
4. **Analysis**: `detect_custom_actions(kp)` applies heuristics
5. **Logging**: Append to `action_logs` with timestamp
6. **Display**: Draw actions on frame with `cv2.putText()`

### Audio Processing Pipeline

```
Microphone → Audio Recording → Whisper Model → Text Segments
    ↓
Text Segments → Logging (with timestamp)
    ↓
Text → Subtitle Update (for display)
```

**Step-by-Step**:
1. **Record**: `sd.rec()` records 4 seconds of audio
2. **Wait**: `sd.wait()` blocks until recording complete
3. **Transcribe**: `model_stt.transcribe()` converts to text
4. **Parse**: Iterate through segments, extract text
5. **Log**: Append to `speech_logs` with timestamp
6. **Update**: Set `current_subtitle` for main thread to display

### Combined Log Generation

**Happens on exit** (after pressing 'q'):

```python
# Create dictionary keyed by second
combined = {}

# Add actions
for entry in action_logs:
    sec = int(entry["timestamp_seconds"])
    combined[sec]["actions"].append(...)

# Add speech
for entry in speech_logs:
    sec = int(entry["timestamp_seconds"])
    combined[sec]["texts"].append(...)

# Sort and save
combined_list = sorted(combined.items())
```

**Result**: Single timeline with both actions and speech per second

## 🎛️ Configuration & Settings

### Adjustable Parameters

**Video Processing**:
```python
model = YOLO("yolo11m-pose.pt")  # Model path
device = "cpu"                    # "cpu", "cuda", "mps"
verbose = False                   # Suppress YOLO output
```

**Audio Processing**:
```python
SAMPLE_RATE = 16000    # Hz (Whisper requirement)
CHUNK_SECONDS = 4      # Audio chunk duration
model_size = "tiny"    # "tiny", "base", "small", "medium", "large"
language = "en"        # "en", "zh", None for auto
beam_size = 1          # Whisper beam search (1=greedy, faster)
```

**Action Detection Thresholds**:
```python
# These are distance thresholds in pixels
arms_crossed_threshold = 80      # Wrist-to-opposite-elbow
hands_clasped_threshold = 60     # Wrist-to-wrist
chin_rest_threshold = 70         # Wrist-to-nose
torso_height_lean_forward = 120  # Shoulder-to-hip
torso_height_lean_back = 200     # Shoulder-to-hip
head_down_offset = 40            # Nose below shoulder
touch_face_threshold = 70        # Wrist-to-face
touch_nose_threshold = 40        # Wrist-to-nose (specific)
fix_hair_threshold = 60          # Wrist-to-ear
fidget_movement_threshold = 25   # Frame-to-frame wrist movement
```

## 🔧 Alternative Implementations

### 1. action_detector3.py
**Simplified version** - Pose detection only, no speech

**Differences**:
- Single-threaded (no STT thread)
- No audio processing
- Faster startup (no Whisper model loading)
- Outputs only `action_log.json`

**Use when**: You only need body language analysis

### 2. run_pose_onnx_dml.py
**ONNX-optimized version** - Hardware acceleration

**Differences**:
- Uses ONNX runtime instead of PyTorch
- Supports DirectML (Windows GPU acceleration)
- Lower-level inference code
- No action detection logic (just raw pose visualization)

**Use when**: You need maximum performance on Windows with compatible GPU

### 3. export_pose.py
**Utility script** - Model conversion

**Purpose**: Convert PyTorch `.pt` model to ONNX `.onnx` format

**Use when**: You want to deploy with ONNX runtime

## 💾 Memory & Performance

### Memory Usage

**Main Application** (`interview_system.py`):
- YOLO Model: ~90 MB (yolo11m-pose)
- Whisper Model: ~75 MB (tiny)
- OpenCV: ~50 MB
- Frame Buffers: ~10 MB
- **Total**: ~225 MB RAM

**Action Detector Only** (`action_detector3.py`):
- YOLO Model: ~90 MB
- OpenCV: ~50 MB
- **Total**: ~140 MB RAM

### Performance Benchmarks

**On CPU** (Intel i5 or equivalent):
- Video Processing: ~30 FPS (33ms per frame)
- YOLO Inference: ~80-100ms per frame
- Audio Processing: Real-time (4s chunk / 4s processing)
- Whisper Inference: ~2-3s per 4s chunk

**On GPU** (CUDA-enabled):
- Video Processing: ~60 FPS (16ms per frame)
- YOLO Inference: ~20-30ms per frame
- Audio Processing: Same (Whisper is CPU-bound)

## 🔐 Privacy & Security

### Local Processing
- All inference runs locally
- No network calls (after model download)
- No data transmitted to servers
- Complete data ownership

### Data Storage
- JSON files stored locally
- No encryption by default (user's responsibility)
- No cloud backup (user can implement if needed)

### Model Provenance
- YOLO: Ultralytics open-source (AGPL-3.0)
- Whisper: OpenAI open-source (MIT)
- Both models downloaded via official channels

## 📊 Scalability Considerations

### Single User
Current implementation works perfectly for:
- Individual interview practice
- One-on-one interview recording
- Single-person behavioral analysis

### Multiple People
To handle multiple people:
- YOLO already detects multiple persons
- Loop through `r.keypoints.xy` (already implemented)
- Need to add person tracking (ID assignment)
- Need separate logs per person

### Long Sessions
For sessions > 30 minutes:
- Logs grow linearly with time
- Memory usage for logs is minimal
- Consider chunked file writing for very long sessions

---

← [Previous: System Overview](01_SYSTEM_OVERVIEW.md) | [Back to Documentation Home](00_README.md) | [Next: Workflow →](03_WORKFLOW.md)
