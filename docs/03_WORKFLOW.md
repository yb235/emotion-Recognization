# Workflow & Execution Flow

## 🔄 Complete System Workflow

This document explains the step-by-step execution flow of the system from startup to shutdown.

## 📋 High-Level Workflow

```
Start → Initialize Models → Start Threads → Process Streams → Exit → Generate Logs → End
```

## 🚀 Detailed Execution Flow

### Phase 1: Initialization (Startup)

#### Step 1.1: Import Dependencies
```python
import cv2                      # Video capture and display
import numpy as np              # Numerical operations
import time                     # Timing and timestamps
import json                     # JSON file operations
import threading                # Multi-threading
import sounddevice as sd        # Audio recording
from faster_whisper import WhisperModel  # Speech recognition
from ultralytics import YOLO    # Pose detection
```

#### Step 1.2: Load YOLO Model
```python
model = YOLO("yolo11m-pose.pt")
```

**What happens**:
- Loads the YOLO11m-pose PyTorch model from disk
- Validates model architecture
- Prepares model for inference
- **Time**: ~2-3 seconds
- **Memory**: ~90 MB

#### Step 1.3: Initialize Global Variables
```python
prev_left_wrist = None
prev_right_wrist = None
action_logs = []
speech_logs = []
start_time = None
stop_flag = False
current_subtitle = ""
```

**Purpose**: Set up shared state for threads

### Phase 2: Thread Startup

#### Step 2.1: Start STT Thread
```python
stt_thread = threading.Thread(target=stt_worker, daemon=True)
stt_thread.start()
```

**What happens**:
- Creates background thread for audio processing
- Thread starts but immediately waits (see Step 2.2)
- Marked as daemon (exits when main thread exits)

#### Step 2.2: STT Thread Initialization
```python
# Inside stt_worker()
model_stt = WhisperModel("tiny", device="cpu")
```

**What happens**:
- Downloads Whisper model (if not cached) - **~75 MB**
- Loads model into memory
- **Time**: ~5-10 seconds first run, ~1-2 seconds subsequent
- Thread then waits for `start_time` to be set

#### Step 2.3: Open Webcam
```python
cap = cv2.VideoCapture(0)
```

**What happens**:
- Opens default webcam (device 0)
- Initializes video stream
- **Time**: ~1-2 seconds
- May show permission dialog on first run

### Phase 3: Main Processing Loop

#### Step 3.1: Read First Frame
```python
ret, frame = cap.read()
if start_time is None:
    start_time = time.time()
```

**Critical moment**: 
- First successful frame sets `start_time`
- This signals STT thread to start recording
- Both threads now synchronized to same timeline

#### Step 3.2: Video Processing (Main Thread Loop)

**For each frame** (~30 FPS):

```
┌─────────────────────────────────────────┐
│ 1. Read Frame from Webcam               │
│    ↓                                     │
│ 2. Calculate Elapsed Time               │
│    ↓                                     │
│ 3. Run YOLO Pose Detection              │
│    ↓                                     │
│ 4. Extract Keypoints                    │
│    ↓                                     │
│ 5. Detect Custom Actions                │
│    ↓                                     │
│ 6. Log Actions (if any detected)        │
│    ↓                                     │
│ 7. Draw UI (actions + subtitles)        │
│    ↓                                     │
│ 8. Display Frame                        │
│    ↓                                     │
│ 9. Check for 'q' key press              │
│    ↓                                     │
│ 10. Repeat or Exit                      │
└─────────────────────────────────────────┘
```

**Detailed Steps**:

**3.2.1 - Read Frame**:
```python
ret, frame = cap.read()
```
- Captures single frame from webcam
- `ret`: True if successful, False if error
- `frame`: NumPy array, shape (height, width, 3) BGR format
- **Time**: ~5-10ms (depends on camera)

**3.2.2 - Calculate Time**:
```python
ts = time.time() - start_time
mm = int(ts // 60)
ss = int(ts % 60)
```
- `ts`: Seconds elapsed since first frame
- `mm`: Minutes component
- `ss`: Seconds component
- Used for timestamps in logs

**3.2.3 - YOLO Inference**:
```python
results = model(frame, device="cpu", verbose=False)
```
- Runs pose estimation on frame
- Detects all people in frame
- Returns keypoints for each person
- **Time**: ~50-100ms on CPU

**3.2.4 - Extract Keypoints**:
```python
for r in results:
    for person in r.keypoints.xy:
        kp = person.cpu().numpy()  # Shape: (17, 2)
```
- Converts GPU tensors to NumPy arrays (if on GPU)
- `kp[i]` = [x, y] for i-th keypoint
- Coordinates in pixel space

**3.2.5 - Detect Actions**:
```python
actions = detect_custom_actions(kp)
```
- Analyzes keypoint positions
- Calculates distances between keypoints
- Applies heuristic rules
- Returns list of detected actions
- Details in [Action Detection Algorithms](06_ACTION_DETECTION.md)

**3.2.6 - Log Actions**:
```python
if frame_actions:
    action_logs.append({
        "time": f"{mm:02d}:{ss:02d}",
        "timestamp_seconds": round(ts, 2),
        "actions": list(set(frame_actions))
    })
```
- Only logs if actions detected
- Removes duplicate actions in same frame
- Appends to global `action_logs` list

**3.2.7 - Draw UI**:
```python
# Draw actions
for act in set(frame_actions):
    cv2.putText(frame, f"ACTION: {act}", (10, y), ...)
    y += 30

# Draw subtitle
cv2.putText(frame, current_subtitle, (10, h - 20), ...)
```
- Green text for actions (top of frame)
- Yellow text for speech (bottom of frame)
- Updates in real-time

**3.2.8 - Display**:
```python
cv2.imshow("Interview Action + Speech Monitor", frame)
```
- Shows annotated frame in window
- Window title identifies the application

**3.2.9 - Check Exit**:
```python
if cv2.waitKey(1) & 0xFF == ord('q'):
    break
```
- Waits 1ms for key press
- Exits loop if 'q' pressed
- **Why 1ms**: Keeps UI responsive at ~30 FPS

#### Step 3.3: Audio Processing (STT Thread Loop)

**Runs in parallel**, independent cycle:

```
┌─────────────────────────────────────────┐
│ 1. Wait for start_time to be set        │
│    ↓                                     │
│ 2. Record 4 seconds of audio            │
│    ↓                                     │
│ 3. Check stop_flag                       │
│    ↓                                     │
│ 4. Flatten audio to mono                │
│    ↓                                     │
│ 5. Transcribe with Whisper              │
│    ↓                                     │
│ 6. Parse segments                        │
│    ↓                                     │
│ 7. Log text with timestamp              │
│    ↓                                     │
│ 8. Update current_subtitle              │
│    ↓                                     │
│ 9. Repeat or Exit                        │
└─────────────────────────────────────────┘
```

**Detailed Steps**:

**3.3.1 - Wait for Start**:
```python
while start_time is None and not stop_flag:
    time.sleep(0.1)
```
- Polls every 100ms
- Waits for main thread to set `start_time`
- Also checks `stop_flag` in case of early exit

**3.3.2 - Record Audio**:
```python
audio = sd.rec(int(4 * 16000), samplerate=16000, 
               channels=1, dtype="float32")
sd.wait()
```
- Records 4 seconds at 16kHz
- Mono channel (1 channel)
- Float32 format (Whisper requirement)
- **Blocks for 4 seconds**

**3.3.3 - Check Stop**:
```python
if stop_flag:
    break
```
- Allows graceful exit
- Main thread sets `stop_flag` when user presses 'q'

**3.3.4 - Flatten Audio**:
```python
audio_mono = audio.flatten()
```
- Converts (N, 1) array to (N,) array
- Whisper expects 1D array

**3.3.5 - Transcribe**:
```python
segments, _ = model_stt.transcribe(
    audio_mono, beam_size=1, language="en"
)
```
- Runs Whisper inference
- `beam_size=1`: Fast, greedy decoding
- `language="en"`: English (skip auto-detect)
- **Time**: ~1-3 seconds per 4-second chunk

**3.3.6 - Parse Segments**:
```python
for seg in segments:
    text = seg.text.strip()
    if not text:
        continue
```
- Whisper returns multiple segments per chunk
- Each segment has start/end times and text
- Skip empty segments

**3.3.7 - Log Speech**:
```python
ts = time.time() - start_time
speech_logs.append({
    "time": f"{mm:02d}:{ss:02d}",
    "timestamp_seconds": round(ts, 2),
    "text": text
})
```
- Uses current time (approximate)
- Note: Slight delay from actual speech time (~1-2 seconds)

**3.3.8 - Update Subtitle**:
```python
current_subtitle = text
```
- Main thread reads this for display
- Shows latest recognized text

### Phase 4: Shutdown & Log Generation

#### Step 4.1: Cleanup Resources
```python
cap.release()
cv2.destroyAllWindows()
stop_flag = True
time.sleep(0.5)
```

**What happens**:
- Release webcam
- Close all OpenCV windows
- Signal STT thread to exit
- Wait for thread to finish

#### Step 4.2: Save Action Log
```python
with open("action_log.json", "w", encoding="utf-8") as f:
    json.dump(action_logs, f, ensure_ascii=False, indent=2)
```

**Output format**:
```json
[
  {
    "time": "00:05",
    "timestamp_seconds": 5.42,
    "actions": ["arms_crossed", "lean_back"]
  },
  ...
]
```

#### Step 4.3: Save Transcription Log
```python
with open("transcription_log.json", "w", encoding="utf-8") as f:
    json.dump(speech_logs, f, ensure_ascii=False, indent=2)
```

**Output format**:
```json
[
  {
    "time": "00:03",
    "timestamp_seconds": 3.21,
    "text": "Hello, I am excited to be here"
  },
  ...
]
```

#### Step 4.4: Generate Combined Log

**Algorithm**:
```python
combined = {}

# Merge actions by second
for entry in action_logs:
    sec = int(entry["timestamp_seconds"])
    if sec not in combined:
        combined[sec] = {
            "time": entry["time"],
            "timestamp_seconds": float(sec),
            "actions": [],
            "texts": []
        }
    combined[sec]["actions"].extend(entry["actions"])

# Merge speech by second
for entry in speech_logs:
    sec = int(entry["timestamp_seconds"])
    if sec not in combined:
        combined[sec] = { ... }
    combined[sec]["texts"].append(entry["text"])

# Deduplicate and sort
for sec in sorted(combined.keys()):
    item = combined[sec]
    item["actions"] = sorted(list(set(item["actions"])))
    combined_list.append(item)
```

**Output format**:
```json
[
  {
    "time": "00:05",
    "timestamp_seconds": 5.0,
    "actions": ["arms_crossed", "lean_back"],
    "texts": ["I have five years of experience"]
  },
  ...
]
```

#### Step 4.5: Save Combined Log
```python
with open("combined_log.json", "w", encoding="utf-8") as f:
    json.dump(combined_list, f, ensure_ascii=False, indent=2)

print("Saved action_log.json, transcription_log.json, combined_log.json")
```

## ⏱️ Timeline Example

**Real-world execution timeline**:

```
Time    | Main Thread                | STT Thread
--------|----------------------------|---------------------------
00:00   | Load YOLO model            | Thread created (waiting)
00:02   | Open webcam                | Load Whisper model
00:04   | Read first frame           | Still waiting...
00:04   | Set start_time             | Start recording! (4s)
00:04   | YOLO inference + display   | Recording...
00:05   | YOLO inference + display   | Recording...
00:06   | YOLO inference + display   | Recording...
00:07   | YOLO inference + display   | Recording...
00:08   | Detect "arms_crossed"      | Transcribe chunk 1
00:08   | Log action, display        | Log "Hello I am..."
00:08   | YOLO inference + display   | Record next chunk (4s)
00:09   | YOLO inference + display   | Recording...
...     | ...                        | ...
00:30   | User presses 'q'           | Recording...
00:30   | Set stop_flag = True       | Finish current chunk
00:30   | Release resources          | Transcribe final chunk
00:31   | Wait for STT thread        | Exit thread
00:31   | Generate logs              | (Thread stopped)
00:31   | Save JSON files            | 
00:31   | Exit program               |
```

## 🔁 Simplified Action Detection Workflow

For each frame:

```
Keypoints → Calculate Centers → Measure Distances → Apply Rules → Return Actions
```

**Example: Detecting "Arms Crossed"**

```python
# Input: 17 keypoints (x, y)
kp = [[x0, y0], [x1, y1], ..., [x16, y16]]

# Extract relevant keypoints
l_elbow = kp[7]   # [x7, y7]
r_elbow = kp[8]   # [x8, y8]
l_wrist = kp[9]   # [x9, y9]
r_wrist = kp[10]  # [x10, y10]

# Calculate distances
dist_l_wrist_to_r_elbow = distance(l_wrist, r_elbow)
dist_r_wrist_to_l_elbow = distance(r_wrist, l_elbow)

# Apply rule
if dist_l_wrist_to_r_elbow < 80 and dist_r_wrist_to_l_elbow < 80:
    actions.append("arms_crossed")
```

See [Action Detection Algorithms](06_ACTION_DETECTION.md) for all 10 actions.

## 🎯 Key Design Decisions

### Why Multi-Threading?
- **Alternative**: Single-threaded would process sequentially
- **Problem**: Video would pause during audio transcription (1-3s)
- **Solution**: Parallel threads allow both to run independently

### Why 4-Second Audio Chunks?
- **Too Short** (1-2s): Incomplete sentences, poor transcription
- **Too Long** (10+ s): Delayed feedback, memory intensive
- **4 seconds**: Balance between accuracy and responsiveness

### Why CPU for YOLO?
- **GPU**: Faster but requires CUDA/MPS setup
- **CPU**: Slower but works everywhere
- **Trade-off**: 30 FPS on CPU is sufficient for this use case

### Why Greedy Decoding (beam_size=1)?
- **Beam Search**: More accurate but slower
- **Greedy**: Faster but slightly less accurate
- **Trade-off**: Speed over perfect accuracy for real-time use

## 🐛 Error Handling

### Camera Not Available
```python
if not cap.isOpened():
    print("Error: cannot open camera.")
    stop_flag = True
    return
```

### Frame Read Failure
```python
ret, frame = cap.read()
if not ret:
    print("Error: cannot read frame.")
    break
```

### Silent Audio Handling
```python
text = seg.text.strip()
if not text:
    continue  # Skip empty transcriptions
```

## 🔍 Debugging Tips

**To see what's happening**:

1. **Enable YOLO verbose**:
```python
results = model(frame, device="cpu", verbose=True)
```

2. **Print detected actions**:
```python
if actions:
    print(f"[{mm:02d}:{ss:02d}] Actions: {actions}")
```

3. **Check thread status**:
```python
print(f"[STT] Thread alive: {stt_thread.is_alive()}")
```

4. **Monitor timing**:
```python
start = time.time()
results = model(frame, device="cpu", verbose=False)
print(f"YOLO inference: {time.time() - start:.3f}s")
```

---

← [Previous: Architecture](02_ARCHITECTURE.md) | [Back to Documentation Home](00_README.md) | [Next: Installation Guide →](04_INSTALLATION.md)
