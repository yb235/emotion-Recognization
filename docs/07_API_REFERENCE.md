# API Reference

## 📚 Function Reference

This document provides detailed API documentation for all functions in the system.

---

## Core Functions

### `distance(p1, p2)`

Calculate Euclidean distance between two points.

**Module**: All files (`interview_system.py`, `action_detector3.py`)

**Signature**:
```python
def distance(p1, p2) -> float
```

**Parameters**:
| Name | Type | Description |
|------|------|-------------|
| `p1` | `tuple` or `list` or `np.ndarray` | First point as `[x, y]` |
| `p2` | `tuple` or `list` or `np.ndarray` | Second point as `[x, y]` |

**Returns**:
- **Type**: `float`
- **Description**: Euclidean distance in pixels

**Example**:
```python
point_a = [100, 150]
point_b = [200, 250]
dist = distance(point_a, point_b)  # Returns 141.42
```

**Formula**:
```
distance = √((x2 - x1)² + (y2 - y1)²)
```

**Complexity**: O(1)

**Notes**:
- Handles any array-like input (list, tuple, numpy array)
- Returns float even if inputs are integers
- Used by all action detection logic

---

### `detect_custom_actions(kp)`

Detect 10 body actions from pose keypoints.

**Module**: `interview_system.py`, `action_detector3.py`

**Signature**:
```python
def detect_custom_actions(kp: np.ndarray) -> List[str]
```

**Parameters**:
| Name | Type | Description |
|------|------|-------------|
| `kp` | `np.ndarray` | Array of 17 keypoints, shape `(17, 2)`, dtype `float32` or `float64` |

**Keypoint Format**:
```python
kp = np.array([
    [nose_x, nose_y],          # 0
    [left_eye_x, left_eye_y],  # 1
    [right_eye_x, right_eye_y],# 2
    # ... (17 keypoints total)
])
```

**Returns**:
- **Type**: `List[str]`
- **Description**: List of detected action names (can be empty)

**Possible Return Values**:
```python
[
    "arms_crossed",
    "hands_clasped",
    "chin_rest",
    "lean_forward",
    "lean_back",
    "head_down",
    "touch_face",
    "touch_nose",
    "fix_hair",
    "fidget_hands"
]
```

**Example**:
```python
# Get keypoints from YOLO
results = model(frame)
kp = results[0].keypoints.xy[0].cpu().numpy()

# Detect actions
actions = detect_custom_actions(kp)
print(actions)  # e.g., ["arms_crossed", "lean_back"]
```

**Side Effects**:
- Modifies global variables: `prev_left_wrist`, `prev_right_wrist`
- Used for fidget detection across frames

**Complexity**: O(1) - constant time (10 checks)

**Thread Safety**: ⚠️ **Not thread-safe** due to global variables

**Notes**:
- Returns unique actions (duplicates removed)
- Can return empty list if no actions detected
- See [Action Detection Algorithms](06_ACTION_DETECTION.md) for details

---

## Threading Functions

### `stt_worker()`

Background thread for speech-to-text processing.

**Module**: `interview_system.py` only

**Signature**:
```python
def stt_worker() -> None
```

**Parameters**: None

**Returns**: None

**Behavior**:
1. Loads Whisper model (`tiny`, CPU)
2. Waits for `start_time` to be set by main thread
3. Continuously records 4-second audio chunks
4. Transcribes each chunk with Whisper
5. Logs results to `speech_logs` list
6. Updates `current_subtitle` for display
7. Exits when `stop_flag` is set

**Global Variables Used**:
| Variable | Type | Access | Description |
|----------|------|--------|-------------|
| `start_time` | `float` | Read | Timing synchronization |
| `stop_flag` | `bool` | Read | Exit signal |
| `speech_logs` | `list` | Write | Append transcriptions |
| `current_subtitle` | `str` | Write | Latest text |

**Thread Configuration**:
```python
stt_thread = threading.Thread(
    target=stt_worker,
    daemon=True  # Exits when main thread exits
)
```

**Blocking Operations**:
- `sd.rec()` + `sd.wait()`: Blocks for 4 seconds
- `model_stt.transcribe()`: Blocks for 1-3 seconds

**Error Handling**:
- Silently skips empty transcriptions
- Continues on transcription errors

**Example Usage**:
```python
# In main()
stt_thread = threading.Thread(target=stt_worker, daemon=True)
stt_thread.start()

# ... video processing ...

# On exit
stop_flag = True  # Signal thread to stop
time.sleep(0.5)   # Wait for thread to finish
```

**Configuration Constants**:
```python
SAMPLE_RATE = 16000      # Hz (Whisper requirement)
CHUNK_SECONDS = 4        # Audio duration per chunk
model_size = "tiny"      # Whisper model
language = "en"          # Speech language
beam_size = 1            # Decoding strategy
```

**Performance**:
- Model loading: ~5-10 seconds (first run)
- Per-chunk processing: ~5-7 seconds total (4s record + 1-3s transcribe)

---

### `main()`

Main control flow for video processing and coordination.

**Module**: `interview_system.py` only

**Signature**:
```python
def main() -> None
```

**Parameters**: None

**Returns**: None

**Behavior**:
1. Starts STT thread
2. Opens webcam
3. Main loop:
   - Read frame
   - Set `start_time` on first frame
   - Run YOLO inference
   - Detect actions
   - Log actions
   - Draw UI
   - Display frame
   - Check for 'q' key
4. On exit:
   - Release camera
   - Close windows
   - Stop STT thread
   - Generate 3 JSON log files

**Global Variables Used**:
| Variable | Type | Access | Description |
|----------|------|--------|-------------|
| `start_time` | `float` | Read/Write | Set on first frame |
| `stop_flag` | `bool` | Write | Signal threads to stop |
| `action_logs` | `list` | Read/Write | Action timeline |
| `speech_logs` | `list` | Read | Speech timeline |
| `current_subtitle` | `str` | Read | Display on frame |

**Exit Conditions**:
- User presses 'q' key
- Camera read failure
- Window close event

**Output Files**:
1. `action_log.json`: All detected actions
2. `transcription_log.json`: All speech
3. `combined_log.json`: Merged timeline

**Example Usage**:
```python
if __name__ == "__main__":
    main()
```

**Error Handling**:
```python
if not cap.isOpened():
    print("Error: cannot open camera.")
    stop_flag = True
    return

if not ret:
    print("Error: cannot read frame.")
    break
```

---

## YOLO Integration

### Model Initialization

```python
from ultralytics import YOLO

model = YOLO("yolo11m-pose.pt")
```

**Parameters**:
- Model path (string): `.pt` file for PyTorch, `.onnx` for ONNX

**Model Options**:
- `yolov8n-pose.pt`: Fastest, least accurate (6.5 MB)
- `yolo11m-pose.pt`: Balanced (84 MB) ✅ **Default**
- `yolo11l-pose.pt`: Slower, more accurate (168 MB)

### Running Inference

```python
results = model(frame, device="cpu", verbose=False)
```

**Parameters**:
| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `frame` | `np.ndarray` | Required | Input image (BGR format) |
| `device` | `str` | `"cpu"` | `"cpu"`, `"cuda"`, `"mps"` |
| `verbose` | `bool` | `True` | Print inference info |
| `conf` | `float` | `0.25` | Confidence threshold |
| `iou` | `float` | `0.45` | IoU threshold for NMS |

**Returns**:
- **Type**: `List[Results]`
- **Description**: List of result objects (usually length 1)

### Extracting Keypoints

```python
for r in results:
    if r.keypoints is None:
        continue
    
    for person in r.keypoints.xy:
        kp = person.cpu().numpy()  # Shape: (17, 2)
        # ... use kp for action detection
```

**Keypoint Format**:
```python
kp.shape  # (17, 2)
kp[0]     # [nose_x, nose_y]
kp[5]     # [left_shoulder_x, left_shoulder_y]
```

**Attributes**:
- `r.keypoints.xy`: Keypoints in pixel coordinates, shape `(N, 17, 2)`
- `r.keypoints.conf`: Keypoint confidences, shape `(N, 17)`
- `r.boxes`: Bounding boxes (if needed)

---

## Whisper Integration

### Model Initialization

```python
from faster_whisper import WhisperModel

model_stt = WhisperModel("tiny", device="cpu")
```

**Parameters**:
| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `model_size` | `str` | Required | `"tiny"`, `"base"`, `"small"`, `"medium"`, `"large"` |
| `device` | `str` | `"cpu"` | `"cpu"` or `"cuda"` |
| `compute_type` | `str` | Auto | `"int8"`, `"float16"`, `"float32"` |

**Model Sizes**:
| Size | Parameters | Download | Speed | Accuracy |
|------|-----------|----------|-------|----------|
| tiny | 39M | ~75 MB | Fast ✅ | Good |
| base | 74M | ~150 MB | Medium | Better |
| small | 244M | ~450 MB | Slow | Good |
| medium | 769M | ~1.5 GB | Very Slow | Very Good |
| large | 1550M | ~3 GB | Slowest | Best |

### Transcribing Audio

```python
segments, info = model_stt.transcribe(
    audio,
    beam_size=1,
    language="en"
)
```

**Parameters**:
| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `audio` | `np.ndarray` | Required | Audio array (16kHz, mono, float32) |
| `beam_size` | `int` | `5` | Beam search width (1=greedy, faster) |
| `language` | `str` | `None` | Language code or None for auto-detect |
| `temperature` | `float` | `0.0` | Sampling temperature (0=deterministic) |

**Returns**:
- `segments`: Iterable of `Segment` objects
- `info`: `TranscriptionInfo` object with metadata

**Segment Object**:
```python
for seg in segments:
    seg.start    # float: Start time in seconds
    seg.end      # float: End time in seconds
    seg.text     # str: Transcribed text
    seg.avg_logprob  # float: Confidence score
```

### Audio Recording

```python
import sounddevice as sd

audio = sd.rec(
    int(duration * samplerate),
    samplerate=samplerate,
    channels=channels,
    dtype=dtype
)
sd.wait()  # Block until recording finishes
```

**Parameters**:
| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `frames` | `int` | Required | Number of frames to record |
| `samplerate` | `int` | Default device | Sample rate in Hz (use 16000 for Whisper) |
| `channels` | `int` | Default device | Number of channels (use 1 for mono) |
| `dtype` | `str` | `"float32"` | Data type (`"float32"` for Whisper) |

**Returns**:
- **Type**: `np.ndarray`
- **Shape**: `(frames, channels)` or `(frames,)` if mono
- **Description**: Recorded audio data

---

## Data Structures

### Action Log Entry

```python
{
    "time": "00:15",                    # MM:SS format
    "timestamp_seconds": 15.42,         # Float seconds
    "actions": ["arms_crossed", "lean_back"]  # List of action names
}
```

**Type**: `Dict[str, Union[str, float, List[str]]]`

### Speech Log Entry

```python
{
    "time": "00:12",                    # MM:SS format
    "timestamp_seconds": 12.34,         # Float seconds
    "text": "I have five years of experience"  # Transcribed text
}
```

**Type**: `Dict[str, Union[str, float]]`

### Combined Log Entry

```python
{
    "time": "00:15",                    # MM:SS format
    "timestamp_seconds": 15.0,          # Float seconds (rounded to second)
    "actions": ["arms_crossed"],        # All actions in this second
    "texts": ["I am very interested"]   # All speech in this second
}
```

**Type**: `Dict[str, Union[str, float, List[str]]]`

---

## Constants and Configuration

### Global Constants

```python
# interview_system.py
SAMPLE_RATE = 16000      # Audio sample rate (Hz)
CHUNK_SECONDS = 4        # Audio chunk duration (seconds)
```

### Detection Thresholds

```python
# In detect_custom_actions()
ARMS_CROSSED_THRESHOLD = 80       # pixels
HANDS_CLASPED_THRESHOLD = 60      # pixels
CHIN_REST_THRESHOLD = 70          # pixels
LEAN_FORWARD_THRESHOLD = 120      # pixels
LEAN_BACK_THRESHOLD = 200         # pixels
HEAD_DOWN_OFFSET = 40             # pixels
TOUCH_FACE_THRESHOLD = 70         # pixels
TOUCH_NOSE_THRESHOLD = 40         # pixels
FIX_HAIR_THRESHOLD = 60           # pixels
FIDGET_MOVEMENT_THRESHOLD = 25    # pixels per frame
```

### Model Paths

```python
YOLO_MODEL_PATH = "yolo11m-pose.pt"
YOLO_ONNX_PATH = "yolo11m-pose.onnx"
WHISPER_MODEL_SIZE = "tiny"
```

---

## Helper Functions

### Time Formatting

```python
# Convert seconds to MM:SS format
ts = 65.5  # 65.5 seconds
mm = int(ts // 60)  # 1 minute
ss = int(ts % 60)   # 5 seconds
time_str = f"{mm:02d}:{ss:02d}"  # "01:05"
```

### Center Point Calculation

```python
# Calculate center between two points
center = (
    (point1[0] + point2[0]) / 2,
    (point1[1] + point2[1]) / 2
)
```

---

## OpenCV Functions Used

### Video Capture

```python
cap = cv2.VideoCapture(source)
```
- `source=0`: Default webcam
- `source=1`: Second camera
- `source="video.mp4"`: Video file

### Read Frame

```python
ret, frame = cap.read()
```
- `ret`: `bool` - Success status
- `frame`: `np.ndarray` - Image (H, W, 3) BGR format

### Display Frame

```python
cv2.imshow(window_name, frame)
```
- `window_name`: `str` - Window title
- `frame`: `np.ndarray` - Image to display

### Draw Text

```python
cv2.putText(
    img=frame,
    text="ACTION: arms_crossed",
    org=(10, 30),              # Bottom-left corner of text
    fontFace=cv2.FONT_HERSHEY_SIMPLEX,
    fontScale=0.8,
    color=(0, 255, 0),         # Green (BGR format)
    thickness=2
)
```

### Wait for Key

```python
key = cv2.waitKey(1) & 0xFF
if key == ord('q'):
    break
```
- `1`: Wait 1ms (keeps display responsive)
- `0`: Wait indefinitely
- `ord('q')`: ASCII code for 'q' key

---

## JSON File I/O

### Save JSON

```python
import json

with open("action_log.json", "w", encoding="utf-8") as f:
    json.dump(
        action_logs,
        f,
        ensure_ascii=False,  # Allow Unicode characters
        indent=2             # Pretty-print with 2-space indent
    )
```

### Load JSON

```python
with open("action_log.json", "r", encoding="utf-8") as f:
    data = json.load(f)
```

---

## Error Handling Patterns

### Camera Error

```python
cap = cv2.VideoCapture(0)
if not cap.isOpened():
    print("Error: cannot open camera.")
    return
```

### Frame Read Error

```python
ret, frame = cap.read()
if not ret:
    print("Error: cannot read frame.")
    break
```

### Empty Transcription

```python
text = seg.text.strip()
if not text:
    continue  # Skip empty segments
```

---

## Performance Tips

### GPU Acceleration

```python
# Use CUDA for YOLO
results = model(frame, device="cuda")

# Use GPU for Whisper
model_stt = WhisperModel("tiny", device="cuda")
```

### Reduce Resolution

```python
# Resize frame before YOLO
frame_small = cv2.resize(frame, (640, 480))
results = model(frame_small, device="cpu")
```

### Smaller Models

```python
# Use lightweight YOLO
model = YOLO("yolov8n-pose.pt")  # Faster, less accurate

# Use smaller Whisper
model_stt = WhisperModel("tiny", device="cpu")  # Already smallest
```

---

← [Previous: Action Detection](06_ACTION_DETECTION.md) | [Back to Documentation Home](00_README.md) | [Next: User Guide →](08_USER_GUIDE.md)
