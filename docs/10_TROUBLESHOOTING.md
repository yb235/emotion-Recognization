# Troubleshooting Guide

## 🆘 Common Issues and Solutions

This guide covers common problems you might encounter and how to solve them.

---

## 🎥 Camera Issues

### Issue: "Error: cannot open camera"

**Symptoms**:
- Console shows: "Error: cannot open camera."
- No webcam window opens
- Program exits immediately

**Causes & Solutions**:

1. **Another app is using the camera**
   ```bash
   # Windows: Close these apps
   - Zoom, Skype, Teams, Discord
   - Other Python scripts
   - Browser tabs with camera access
   
   # macOS: Check in Activity Monitor
   # Search for apps using "Camera"
   
   # Linux: Check processes
   lsof /dev/video0
   ```

2. **Camera not connected**
   - Check USB connection (for external webcam)
   - Check System Preferences/Settings → Camera permissions
   - Try different USB port

3. **Wrong camera index**
   ```python
   # Try different camera indices
   cap = cv2.VideoCapture(1)  # Second camera
   cap = cv2.VideoCapture(2)  # Third camera
   ```

4. **Driver issues** (Windows)
   - Update webcam drivers
   - Device Manager → Cameras → Update Driver

5. **Permission denied** (macOS)
   - System Preferences → Security & Privacy → Camera
   - Enable Python/Terminal

**Test camera independently**:
```bash
# Quick camera test
python -c "import cv2; cap = cv2.VideoCapture(0); ret, frame = cap.read(); cap.release(); print('Camera OK' if ret else 'Camera FAIL')"
```

---

### Issue: Camera works but shows black screen

**Symptoms**:
- Window opens
- Shows black/gray screen
- No error messages

**Solutions**:

1. **Camera lens covered**
   - Check if lens cap is on
   - Check if physically blocked

2. **Very dark room**
   - Turn on lights
   - Adjust camera exposure (automatic usually)

3. **Camera initialization delay**
   - Add delay after opening camera:
   ```python
   cap = cv2.VideoCapture(0)
   time.sleep(2)  # Wait for camera to initialize
   ```

4. **Wrong video format**
   ```python
   # Try setting format explicitly
   cap.set(cv2.CAP_PROP_FRAME_WIDTH, 640)
   cap.set(cv2.CAP_PROP_FRAME_HEIGHT, 480)
   ```

---

### Issue: Low frame rate / Choppy video

**Symptoms**:
- Video updates slowly (< 10 FPS)
- Laggy display
- Delayed response

**Solutions**:

1. **Use GPU acceleration**
   ```python
   results = model(frame, device="cuda")  # NVIDIA
   results = model(frame, device="mps")   # Apple Silicon
   ```

2. **Use smaller YOLO model**
   ```python
   model = YOLO("yolov8n-pose.pt")  # Faster, less accurate
   ```

3. **Reduce resolution**
   ```python
   frame = cv2.resize(frame, (640, 480))
   results = model(frame, device="cpu")
   ```

4. **Close other programs**
   - Close browser tabs
   - Close other CPU-intensive apps
   - Check Task Manager/Activity Monitor

5. **Disable speech recognition** (use action_detector3.py)
   ```bash
   python action_detector3.py  # No STT, faster
   ```

---

## 🎤 Microphone Issues

### Issue: Speech not being transcribed

**Symptoms**:
- No yellow text at bottom of screen
- Empty transcription_log.json
- Console shows STT thread started but no output

**Solutions**:

1. **Check microphone permissions**
   - **macOS**: System Preferences → Security & Privacy → Microphone
   - **Windows**: Settings → Privacy → Microphone
   - Enable for Python/Terminal

2. **Wrong microphone selected**
   ```python
   # List available devices
   import sounddevice as sd
   print(sd.query_devices())
   
   # Set default device
   sd.default.device = 1  # Use device index from list
   ```

3. **Microphone muted**
   - Check system volume settings
   - Check microphone hardware mute switch
   - Increase microphone volume

4. **Speaking too quietly**
   - Speak louder
   - Move closer to microphone
   - Test with: "Ahem, testing one two three"

5. **Background noise too loud**
   - Move to quieter location
   - Use directional microphone
   - Close windows/doors

6. **Language mismatch**
   ```python
   # If speaking non-English, change language
   segments, _ = model_stt.transcribe(
       audio_mono,
       language=None  # Auto-detect
   )
   ```

**Test microphone independently**:
```python
import sounddevice as sd
import numpy as np

# Record 3 seconds
print("Recording...")
audio = sd.rec(int(3 * 16000), samplerate=16000, channels=1)
sd.wait()
print("Done. Max amplitude:", np.max(np.abs(audio)))
# Should be > 0.01 if microphone is working
```

---

### Issue: Whisper model not downloading

**Symptoms**:
- Hangs at "Loading Whisper tiny model..."
- Connection errors
- Timeout errors

**Solutions**:

1. **Network issues**
   - Check internet connection
   - Disable VPN temporarily
   - Try different network

2. **Download manually**
   ```bash
   # Download model ahead of time
   python -c "from faster_whisper import WhisperModel; WhisperModel('tiny')"
   ```

3. **Disk space**
   - Check available disk space (need ~500 MB)
   - Clean up temporary files

4. **Use different model location**
   ```python
   # Specify cache directory
   model_stt = WhisperModel(
       "tiny",
       device="cpu",
       download_root="/path/to/cache"
   )
   ```

---

## 🤖 Model & AI Issues

### Issue: Actions not being detected

**Symptoms**:
- No green text showing actions
- Empty action_log.json
- You're clearly performing actions but system doesn't detect

**Solutions**:

1. **Poor lighting**
   - Turn on more lights
   - Ensure even lighting (no harsh shadows)
   - Face light source

2. **Wrong distance from camera**
   - **Too far**: Move closer (1-2 meters ideal)
   - **Too close**: Move back
   - Ensure full upper body visible

3. **Not facing camera**
   - Face camera directly (±30° okay)
   - Side/back views won't work well

4. **Actions not clear enough**
   - Perform actions more deliberately
   - Hold pose for 1-2 seconds
   - Make movements more pronounced

5. **Thresholds too strict**
   - Edit detection thresholds in code
   - See [Action Detection Algorithms](06_ACTION_DETECTION.md)
   - Example: Increase threshold values

6. **YOLO not detecting person**
   ```python
   # Add debug output
   results = model(frame, device="cpu", verbose=True)
   for r in results:
       print(f"Detected {len(r.keypoints.xy)} person(s)")
   ```

**Test action detection**:
- Cross arms very clearly and hold for 3 seconds
- Lean very far forward
- Put hand directly on nose
- If still no detection, thresholds may need adjustment

---

### Issue: Too many false positive actions

**Symptoms**:
- Actions detected when not performing them
- Screen filled with action labels
- Unrealistic number of actions in logs

**Solutions**:

1. **Increase detection thresholds**
   ```python
   # Make stricter (in detect_custom_actions)
   # Example: arms_crossed
   if (distance(l_wrist, r_elbow) < 60 and  # Was 80
       distance(r_wrist, l_elbow) < 60):    # Was 80
       actions.append("arms_crossed")
   ```

2. **Reduce fidget sensitivity**
   ```python
   # Increase movement threshold
   if distance(prev_left_wrist, l_wrist) > 40:  # Was 25
       fidget_detected = True
   ```

3. **Filter by duration**
   ```python
   # Only log if action persists
   if len(recent_actions) > 3:  # Detected in 3+ consecutive frames
       action_logs.append(...)
   ```

4. **Smooth keypoints**
   - YOLO detection jitter causes false positives
   - Implement Kalman filter or moving average

---

### Issue: Speech transcription is inaccurate

**Symptoms**:
- Wrong words transcribed
- Nonsense text
- Hallucinations (text when nobody spoke)

**Solutions**:

1. **Use larger Whisper model**
   ```python
   # More accurate but slower
   model_stt = WhisperModel("base", device="cpu")   # Better
   model_stt = WhisperModel("small", device="cpu")  # Best
   ```

2. **Reduce background noise**
   - Use quiet room
   - Close windows/doors
   - Turn off fans, AC

3. **Speak more clearly**
   - Enunciate words
   - Speak at moderate pace
   - Avoid mumbling

4. **Set correct language**
   ```python
   segments, _ = model_stt.transcribe(
       audio_mono,
       language="en"  # Explicitly set language
   )
   ```

5. **Check for hallucinations**
   - Whisper sometimes generates text in silence
   - Filter by confidence:
   ```python
   for seg in segments:
       if seg.avg_logprob > -1.0:  # Higher confidence
           # Use this segment
   ```

---

## 💻 Installation & Dependency Issues

### Issue: "No module named 'cv2'"

**Solution**:
```bash
pip install opencv-python
```

If still doesn't work:
```bash
pip uninstall opencv-python opencv-python-headless
pip install opencv-python
```

---

### Issue: "No module named 'ultralytics'"

**Solution**:
```bash
pip install ultralytics
```

---

### Issue: "No module named 'faster_whisper'"

**Solution**:
```bash
pip install faster-whisper
```

---

### Issue: sounddevice errors on macOS

**Symptoms**:
- Import errors
- "PortAudio not found"
- Crashes on audio recording

**Solution**:
```bash
# Install PortAudio first
brew install portaudio

# Reinstall sounddevice
pip uninstall sounddevice
pip install sounddevice
```

---

### Issue: Virtual environment not activating

**Windows**:
```bash
# If PowerShell gives execution policy error
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser

# Then try again
venv\Scripts\Activate.ps1
```

**macOS/Linux**:
```bash
# Ensure you're in project directory
cd ~/path/to/emotion-Recognization

# Activate
source venv/bin/activate
```

---

### Issue: "python: command not found"

**Solution**:
- Python not installed or not in PATH
- Try `python3` instead of `python`
- Reinstall Python and check "Add to PATH"

---

## 📁 File & Permission Issues

### Issue: JSON files not created

**Symptoms**:
- Program exits normally but no JSON files
- "Permission denied" errors

**Solutions**:

1. **Check write permissions**
   ```bash
   # Check current directory permissions
   ls -la  # macOS/Linux
   dir     # Windows
   
   # Change permissions if needed (macOS/Linux)
   chmod 755 .
   ```

2. **Change output directory**
   ```python
   # Write to home directory instead
   import os
   output_dir = os.path.expanduser("~/interview_outputs")
   os.makedirs(output_dir, exist_ok=True)
   
   with open(os.path.join(output_dir, "action_log.json"), "w") as f:
       json.dump(action_logs, f, indent=2)
   ```

3. **Check disk space**
   ```bash
   df -h  # macOS/Linux
   ```

4. **Ensure proper exit**
   - Must press 'q' to exit properly
   - Ctrl+C force quit won't save files

---

### Issue: Can't find YOLO model file

**Symptoms**:
- "FileNotFoundError: yolo11m-pose.pt"
- "No such file or directory"

**Solutions**:

1. **Check current directory**
   ```bash
   ls yolo11m-pose.pt  # Should exist
   ```

2. **Download model**
   ```python
   from ultralytics import YOLO
   model = YOLO("yolo11m-pose.pt")  # Auto-downloads if missing
   ```

3. **Use absolute path**
   ```python
   model = YOLO("/full/path/to/yolo11m-pose.pt")
   ```

---

## ⚡ Performance Issues

### Issue: Program crashes / Out of memory

**Symptoms**:
- "Killed" message
- Program suddenly exits
- System freezes

**Solutions**:

1. **Close other programs**
   - Free up RAM
   - Check Task Manager/Activity Monitor

2. **Use smaller model**
   ```python
   model = YOLO("yolov8n-pose.pt")  # Uses less memory
   model_stt = WhisperModel("tiny", device="cpu")  # Already smallest
   ```

3. **Reduce batch size** (if processing multiple frames)
   - Process frames one at a time (already default)

4. **Monitor memory usage**
   ```bash
   # macOS/Linux
   top -p $(pgrep -f interview_system)
   
   # Windows: Task Manager
   ```

5. **Increase system swap/virtual memory**
   - Depends on OS

---

### Issue: GPU not being used

**Symptoms**:
- Specified `device="cuda"` but still slow
- nvidia-smi shows 0% GPU usage

**Solutions**:

1. **Check CUDA installation**
   ```python
   import torch
   print(torch.cuda.is_available())  # Should print True
   print(torch.cuda.get_device_name(0))  # Shows GPU name
   ```

2. **Install CUDA-enabled PyTorch**
   ```bash
   # CUDA 11.8
   pip install torch torchvision --index-url https://download.pytorch.org/whl/cu118
   ```

3. **Check GPU is CUDA-compatible**
   - Only NVIDIA GPUs support CUDA
   - AMD GPUs: Use DirectML (run_pose_onnx_dml.py)
   - Intel GPUs: Not supported, use CPU

4. **Update GPU drivers**
   - Download from NVIDIA website
   - Restart after installation

---

## 🐛 Unexpected Behavior

### Issue: Program hangs / Freezes

**Symptoms**:
- No response to keyboard
- Window frozen
- CPU at 100%

**Solutions**:

1. **Wait for model loading** (first time)
   - YOLO: ~2-3 seconds
   - Whisper: ~5-10 seconds
   - Be patient on first run

2. **Check thread deadlock** (rare)
   - Force quit: Ctrl+C
   - Restart program

3. **Check infinite loop** (if you modified code)
   - Add print statements for debugging

4. **Reduce audio chunk size** (if hanging in STT)
   ```python
   CHUNK_SECONDS = 2  # Was 4, try smaller
   ```

---

### Issue: Window doesn't appear

**Symptoms**:
- Program running but no window
- Console output looks normal

**Solutions**:

1. **Check if window is hidden**
   - Alt+Tab / Cmd+Tab to find window
   - Check all virtual desktops

2. **Display issues** (headless system)
   - Can't run on server without display
   - Use X11 forwarding or VNC

3. **macOS permission**
   - System Preferences → Security → Screen Recording
   - Enable Terminal/Python

4. **Force window to foreground**
   ```python
   cv2.namedWindow("Interview Action + Speech Monitor", cv2.WINDOW_NORMAL)
   cv2.setWindowProperty("Interview Action + Speech Monitor", 
                         cv2.WND_PROP_TOPMOST, 1)
   ```

---

### Issue: Keyboard input not working

**Symptoms**:
- Pressing 'q' doesn't quit
- No response to keyboard

**Solutions**:

1. **Click on video window first**
   - Window must have focus
   - Click on it, then press 'q'

2. **Try different key**
   - Some keyboards may have issues
   - Modify code to use Esc:
   ```python
   if cv2.waitKey(1) & 0xFF == 27:  # Esc key
       break
   ```

3. **Force quit if stuck**
   - Ctrl+C in terminal
   - Kill process from Task Manager

---

## 📊 Data Quality Issues

### Issue: Timestamps don't match reality

**Symptoms**:
- Timestamps seem off by seconds
- Events not aligned properly

**Explanation**:
- **Actions**: Very accurate (±0.1s)
- **Speech**: Less accurate (±1-2s) due to processing delay

**Solutions**:

1. **Use action timestamps** as ground truth
   - More reliable than speech timestamps

2. **Adjust for processing delay**
   ```python
   # Subtract ~2 seconds from speech timestamps
   adjusted_time = entry["timestamp_seconds"] - 2.0
   ```

3. **Use combined_log.json**
   - Grouped by second, reduces precision issues

---

### Issue: Actions logged multiple times

**Symptoms**:
- Same action appears many times per second
- action_log.json is huge

**Explanation**:
- System logs every frame (30+ per second)
- Same action detected in consecutive frames

**Solutions**:

1. **Use combined_log.json**
   - Already deduplicates per second

2. **Filter duplicates**
   ```python
   # Remove consecutive duplicates
   filtered = []
   prev_actions = None
   for entry in action_logs:
       if entry["actions"] != prev_actions:
           filtered.append(entry)
           prev_actions = entry["actions"]
   ```

3. **Implement debouncing** (modify code)
   ```python
   # Only log if action changes
   if actions != prev_frame_actions:
       action_logs.append(...)
   ```

---

## 🔧 Advanced Troubleshooting

### Enable Debug Mode

Add debug output to see what's happening:

```python
# At top of file
DEBUG = True

# In detect_custom_actions
if DEBUG:
    print(f"Detected {len(kp)} keypoints")
    print(f"Actions: {actions}")

# In main loop
if DEBUG:
    print(f"Frame {frame_count}, FPS: {fps:.1f}")
```

---

### Check Component Individually

**Test YOLO only**:
```python
from ultralytics import YOLO
import cv2

model = YOLO("yolo11m-pose.pt")
cap = cv2.VideoCapture(0)

while True:
    ret, frame = cap.read()
    results = model(frame, device="cpu", verbose=True)
    print(f"Detected {len(results[0].keypoints.xy)} person(s)")
    cv2.imshow("Test", frame)
    if cv2.waitKey(1) & 0xFF == ord('q'):
        break

cap.release()
cv2.destroyAllWindows()
```

**Test Whisper only**:
```python
import sounddevice as sd
from faster_whisper import WhisperModel

model = WhisperModel("tiny", device="cpu")

audio = sd.rec(int(4 * 16000), samplerate=16000, channels=1)
sd.wait()

segments, _ = model.transcribe(audio.flatten(), language="en")
for seg in segments:
    print(seg.text)
```

---

### Collect Debug Information

When reporting issues:

```bash
# Python version
python --version

# Package versions
pip list | grep -E "ultralytics|opencv|whisper|sounddevice|torch"

# System info
uname -a  # macOS/Linux
systeminfo  # Windows

# GPU info
nvidia-smi  # NVIDIA GPU
```

---

## 📞 Getting More Help

### Documentation

- [System Overview](01_SYSTEM_OVERVIEW.md) - Understand what system does
- [Installation Guide](04_INSTALLATION.md) - Setup instructions
- [User Guide](08_USER_GUIDE.md) - Usage instructions
- [API Reference](07_API_REFERENCE.md) - Technical details

### Online Resources

- [Ultralytics YOLO Docs](https://docs.ultralytics.com/)
- [Faster-Whisper GitHub](https://github.com/guillaumekln/faster-whisper)
- [OpenCV Python Tutorials](https://docs.opencv.org/4.x/d6/d00/tutorial_py_root.html)

### Community

- GitHub Issues for this project
- Stack Overflow (tag: python, opencv, yolo)

---

## ✅ Verification Checklist

Before asking for help, verify:

- [ ] Python 3.10+ installed
- [ ] Virtual environment activated
- [ ] All packages installed (pip list)
- [ ] YOLO model files present
- [ ] Webcam works in other apps
- [ ] Microphone works in other apps
- [ ] Tried examples from documentation
- [ ] Checked error messages carefully
- [ ] Tried solutions in this guide

---

← [Previous: Output Format](09_OUTPUT_FORMAT.md) | [Back to Documentation Home](00_README.md)
