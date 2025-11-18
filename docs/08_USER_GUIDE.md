# User Guide

## 🎯 Getting Started

This guide will walk you through using the Interview Behavior and Speech Analysis System from start to finish.

## ✅ Before You Start

Make sure you've completed the [Installation Guide](04_INSTALLATION.md):
- [ ] Python 3.10+ installed
- [ ] Virtual environment created and activated
- [ ] All dependencies installed
- [ ] Webcam and microphone working
- [ ] YOLO models present

---

## 🚀 Quick Start - 3 Minute Tutorial

### Step 1: Activate Virtual Environment

**Windows (Command Prompt)**:
```bash
cd path\to\emotion-Recognization
venv\Scripts\activate
```

**Windows (PowerShell)**:
```bash
cd path\to\emotion-Recognization
venv\Scripts\Activate.ps1
```

**macOS/Linux**:
```bash
cd ~/path/to/emotion-Recognization
source venv/bin/activate
```

You should see `(venv)` at the start of your prompt.

### Step 2: Run the System

```bash
python interview_system.py
```

**What happens**:
1. Console prints: "Loading Whisper tiny model on CPU..." (~5-10 seconds)
2. Console prints: "[MAIN] Timing started from first frame."
3. Webcam window opens showing live video
4. System starts detecting actions and speech
5. Green text shows detected actions (e.g., "ACTION: arms_crossed")
6. Yellow text shows transcribed speech at bottom

### Step 3: Test It Out

Try these actions in front of the camera:
- Cross your arms → Should detect "arms_crossed"
- Clasp your hands → Should detect "hands_clasped"
- Lean forward → Should detect "lean_forward"
- Say something → Should transcribe and show at bottom

### Step 4: Exit

Press **'q'** on your keyboard (focus on video window).

**What happens**:
- Webcam window closes
- Console prints: "Saved action_log.json, transcription_log.json, combined_log.json"
- Three JSON files created in project folder

### Step 5: View Results

Open the JSON files:
```bash
# macOS
open action_log.json

# Linux
xdg-open action_log.json

# Windows
start action_log.json
```

**You did it!** 🎉 You've successfully run the system.

---

## 📊 Understanding the Output

### Three Output Files

1. **action_log.json** - All detected body actions
2. **transcription_log.json** - All transcribed speech
3. **combined_log.json** - Merged timeline ⭐ **Most useful**

### Viewing JSON Files

**Best Tools**:
- [JSON Viewer Online](http://jsonviewer.stack.hu/)
- VS Code (built-in JSON support)
- Any text editor

**Quick View**:
```bash
# Pretty-print in terminal
python -m json.tool action_log.json

# Or
cat action_log.json | python -m json.tool
```

---

## 🎬 Usage Scenarios

### Scenario 1: Practice Interview

**Goal**: Record yourself answering practice questions

**Steps**:
1. Set up camera at eye level, 1-2 meters away
2. Prepare interview questions
3. Run the system: `python interview_system.py`
4. Answer questions naturally
5. Press 'q' when done
6. Review combined_log.json for:
   - Times you showed confidence (lean forward, open posture)
   - Times you showed nervousness (arms crossed, fidgeting, touch face)
   - What you said at different times

**Tips**:
- Practice 2-3 times
- Compare logs to see improvement
- Focus on reducing nervous gestures

---

### Scenario 2: Analyze Real Interview

**Goal**: Record actual interview for later review

**Steps**:
1. Set up camera (with permission!)
2. Run system before interview starts
3. Conduct interview normally
4. Press 'q' after interview ends
5. Review data with interviewer/coach

**Tips**:
- Always get consent from all parties
- Test setup before actual interview
- Have backup recording (video file)

---

### Scenario 3: Behavioral Research

**Goal**: Collect data for research study

**Steps**:
1. Design study protocol
2. Get IRB approval (if required)
3. Collect data from participants:
   ```bash
   python interview_system.py
   # After each participant, rename files:
   mv action_log.json participant_001_action.json
   mv transcription_log.json participant_001_speech.json
   mv combined_log.json participant_001_combined.json
   ```
4. Analyze aggregated data

**Tips**:
- Standardize environment (same room, lighting)
- Keep camera position consistent
- Record participant ID and conditions

---

### Scenario 4: Quick Action Detection

**Goal**: Just monitor body language, no speech

**Use**: `action_detector3.py` instead

**Steps**:
```bash
python action_detector3.py
```

**Benefits**:
- Faster startup (~3 seconds)
- No microphone needed
- Less resource intensive
- Simpler output (one JSON file)

---

### Scenario 5: Maximum Performance

**Goal**: Fastest pose detection (Windows + GPU)

**Use**: `run_pose_onnx_dml.py`

**Steps**:
```bash
pip install onnxruntime-directml
python run_pose_onnx_dml.py
```

**Note**: This version doesn't log actions, just visualizes pose.

---

## ⚙️ Configuration Options

### Change Whisper Model Size

**Trade-off**: Accuracy vs Speed

**Edit** `interview_system.py`, line 142:

```python
# Current (fast, good accuracy)
model_stt = WhisperModel("tiny", device="cpu")

# Better accuracy, slower
model_stt = WhisperModel("base", device="cpu")

# Best accuracy, very slow
model_stt = WhisperModel("small", device="cpu")
```

**Model Comparison**:
| Model | Size | Speed | Accuracy | Best For |
|-------|------|-------|----------|----------|
| tiny | 39M | ⚡⚡⚡ | ⭐⭐⭐ | Real-time |
| base | 74M | ⚡⚡ | ⭐⭐⭐⭐ | Balanced |
| small | 244M | ⚡ | ⭐⭐⭐⭐⭐ | High accuracy |

---

### Change Language

**Edit** `interview_system.py`, line 169:

```python
# English (default)
language="en"

# Spanish
language="es"

# Chinese
language="zh"

# French
language="fr"

# Auto-detect (slower)
language=None
```

**Supported Languages**: 
99 languages! See [Whisper documentation](https://github.com/openai/whisper#available-models-and-languages).

---

### Adjust Detection Sensitivity

**Edit** thresholds in `detect_custom_actions()` function:

```python
# Line 64-66: Arms crossed
# More strict (fewer detections)
if (distance(l_wrist, r_elbow) < 60 and ...):

# More lenient (more detections)
if (distance(l_wrist, r_elbow) < 100 and ...):
```

See [Action Detection Algorithms](06_ACTION_DETECTION.md) for all thresholds.

---

### Use GPU Acceleration

**For YOLO (all files)**:

**NVIDIA GPU**:
```python
# Change device="cpu" to device="cuda"
results = model(frame, device="cuda", verbose=False)
```

**Apple Silicon**:
```python
# Change device="cpu" to device="mps"
results = model(frame, device="mps", verbose=False)
```

**For Whisper** (interview_system.py):
```python
# Only works with NVIDIA GPU
model_stt = WhisperModel("tiny", device="cuda")
```

---

### Change Camera Source

**Edit** line 205 (interview_system.py) or 144 (action_detector3.py):

```python
# Default webcam
cap = cv2.VideoCapture(0)

# Second camera
cap = cv2.VideoCapture(1)

# Video file (for testing)
cap = cv2.VideoCapture("test_video.mp4")

# RTSP stream
cap = cv2.VideoCapture("rtsp://camera-ip/stream")
```

---

## 🎨 On-Screen Display

### Understanding the Video Window

```
┌─────────────────────────────────────────────┐
│ Interview Action + Speech Monitor           │ ← Window title
├─────────────────────────────────────────────┤
│ ACTION: arms_crossed                        │ ← Detected actions
│ ACTION: lean_forward                        │    (green text)
│                                             │
│                                             │
│         [Your webcam feed]                  │
│                                             │
│                                             │
│ I am very excited about this opportunity    │ ← Latest speech
└─────────────────────────────────────────────┘    (yellow text)
```

### Color Coding

- **Green text (top)**: Detected body actions
- **Yellow text (bottom)**: Transcribed speech
- **Red circles**: Keypoints (if you modify code to show them)

---

## 📁 Managing Output Files

### Organizing Multiple Sessions

**Create session folders**:
```bash
mkdir session_001
python interview_system.py
# After exit, move files
mv *_log.json session_001/

mkdir session_002
python interview_system.py
mv *_log.json session_002/
```

**Or rename with timestamps**:
```bash
python interview_system.py
# After exit
timestamp=$(date +%Y%m%d_%H%M%S)
mv action_log.json action_log_$timestamp.json
mv transcription_log.json transcription_log_$timestamp.json
mv combined_log.json combined_log_$timestamp.json
```

### Backup Important Sessions

```bash
# Create backup folder
mkdir backups

# Copy files
cp session_001/*.json backups/
```

---

## 📊 Analyzing the Data

### Quick Statistics

**Count actions**:
```python
import json

with open("action_log.json") as f:
    data = json.load(f)

# Count each action type
from collections import Counter
all_actions = []
for entry in data:
    all_actions.extend(entry["actions"])

counts = Counter(all_actions)
print(counts)
# Output: {'arms_crossed': 12, 'fidget_hands': 8, 'lean_forward': 5, ...}
```

**Count words spoken**:
```python
with open("transcription_log.json") as f:
    data = json.load(f)

total_words = sum(len(entry["text"].split()) for entry in data)
print(f"Total words: {total_words}")
```

### Visualization

**Timeline chart** (requires matplotlib):
```python
import json
import matplotlib.pyplot as plt

with open("combined_log.json") as f:
    data = json.load(f)

times = [entry["timestamp_seconds"] for entry in data]
action_counts = [len(entry["actions"]) for entry in data]

plt.plot(times, action_counts)
plt.xlabel("Time (seconds)")
plt.ylabel("Number of actions")
plt.title("Actions over time")
plt.show()
```

---

## 🎯 Best Practices

### Camera Setup

✅ **Good**:
- Eye level with subject
- 1-2 meters distance
- Well-lit room (evenly lit)
- Clear background
- Stable mount (tripod)

❌ **Avoid**:
- Below chin angle (distorts pose)
- Too close (< 0.5m)
- Backlit (subject in shadow)
- Cluttered background
- Handheld (shaky)

### Lighting

✅ **Good**:
- Front lighting (light in front of person)
- Soft, diffused light
- Consistent brightness
- Daylight or LED

❌ **Avoid**:
- Backlighting (window behind person)
- Harsh shadows
- Flashing/flickering lights
- Very dim rooms

### Environment

✅ **Good**:
- Quiet room (for speech recognition)
- Consistent temperature
- Minimal distractions
- Professional background

❌ **Avoid**:
- Loud background noise
- People walking by
- TV/music playing
- Moving objects in frame

### Session Duration

- **Short** (2-5 min): Quick practice
- **Medium** (5-15 min): Typical interview
- **Long** (15-30 min): In-depth session
- **Very Long** (30+ min): Consider breaks

**Note**: Longer sessions = larger JSON files

---

## 🔧 Advanced Usage

### Batch Processing

Process multiple video files:
```python
import cv2
import json
from ultralytics import YOLO

model = YOLO("yolo11m-pose.pt")

video_files = ["interview1.mp4", "interview2.mp4", "interview3.mp4"]

for video_file in video_files:
    cap = cv2.VideoCapture(video_file)
    action_logs = []
    
    while True:
        ret, frame = cap.read()
        if not ret:
            break
        
        results = model(frame, device="cpu")
        # ... (action detection logic)
    
    cap.release()
    
    # Save with video name
    output_name = video_file.replace(".mp4", "_actions.json")
    with open(output_name, "w") as f:
        json.dump(action_logs, f, indent=2)
```

### Custom Action Detection

Add your own action:
```python
# In detect_custom_actions() function

# Add after existing actions

# 11. Hand raised (asking question)
if (l_wrist[1] < l_shoulder[1] - 50 or 
    r_wrist[1] < r_shoulder[1] - 50):
    actions.append("hand_raised")
```

### Export to CSV

Convert JSON to CSV for Excel/analysis:
```python
import json
import csv

with open("combined_log.json") as f:
    data = json.load(f)

with open("combined_log.csv", "w", newline="") as f:
    writer = csv.writer(f)
    writer.writerow(["Time", "Timestamp", "Actions", "Speech"])
    
    for entry in data:
        writer.writerow([
            entry["time"],
            entry["timestamp_seconds"],
            "; ".join(entry["actions"]),
            "; ".join(entry["texts"])
        ])

print("Exported to combined_log.csv")
```

---

## 🆘 Common Issues

### "Camera not opening"

**Solutions**:
1. Close other apps using camera (Zoom, Skype, etc.)
2. Check camera permissions (System Settings)
3. Try different camera index: `VideoCapture(1)`
4. Restart computer

### "Slow performance / Low FPS"

**Solutions**:
1. Use GPU acceleration (see Configuration Options)
2. Use smaller model: `yolov8n-pose.pt`
3. Reduce resolution:
   ```python
   frame = cv2.resize(frame, (640, 480))
   ```
4. Close other programs

### "Speech not being transcribed"

**Solutions**:
1. Check microphone permissions
2. Speak clearly and close to microphone
3. Increase volume
4. Check selected audio device:
   ```python
   import sounddevice as sd
   print(sd.query_devices())
   ```
5. Use larger Whisper model: `base` instead of `tiny`

### "Actions not being detected"

**Solutions**:
1. Ensure good lighting
2. Face camera directly
3. Perform actions clearly and deliberately
4. Lower detection thresholds (see Configuration Options)
5. Stand closer to camera

### "JSON files not created"

**Solutions**:
1. Check write permissions in folder
2. Make sure you pressed 'q' to exit properly
3. Check console for error messages
4. Ensure script completed (didn't crash)

---

## 💡 Tips & Tricks

### Test Before Important Session

Always do a test run:
```bash
python interview_system.py
# Test for 30 seconds
# Press 'q' and verify JSON files created
```

### Save Console Output

Capture console messages:
```bash
# Save to file
python interview_system.py 2>&1 | tee session_log.txt

# Or
python interview_system.py > output.log 2>&1
```

### Monitor Resource Usage

**Check CPU/RAM**:
```bash
# Linux/macOS
top -p $(pgrep -f interview_system)

# Windows Task Manager
# Look for python.exe process
```

### Keyboard Shortcuts

- **'q'**: Quit application
- **Alt+Tab** / **Cmd+Tab**: Switch windows without stopping
- **Ctrl+C**: Force quit (emergency)

---

← [Previous: API Reference](07_API_REFERENCE.md) | [Back to Documentation Home](00_README.md) | [Next: Output Format →](09_OUTPUT_FORMAT.md)
