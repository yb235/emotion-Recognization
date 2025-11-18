# Installation Guide

## 🎯 Overview

This guide walks you through installing the Interview Behavior and Speech Analysis System on your computer. Follow these steps carefully, and you'll be up and running in about 10-15 minutes.

## ✅ Prerequisites

Before you begin, make sure you have:

### Hardware Requirements
- **Computer**: Windows, macOS, or Linux
- **Webcam**: Built-in or USB webcam
- **Microphone**: Built-in or external microphone
- **RAM**: At least 4 GB (8 GB recommended)
- **Storage**: At least 500 MB free space
- **Processor**: Intel i5 or equivalent (better CPU = faster processing)

### Optional for Better Performance
- **GPU**: NVIDIA GPU with CUDA support (for faster pose detection)
- **Dedicated Graphics**: AMD GPU with DirectML support (Windows only)

### Software Requirements
- **Python**: Version 3.10 or higher
- **pip**: Python package installer (comes with Python)
- **Git**: For cloning the repository (optional)

## 📥 Step 1: Install Python

### Check if Python is Already Installed

Open your terminal/command prompt and run:

**Windows (Command Prompt or PowerShell)**:
```bash
python --version
```

**macOS/Linux (Terminal)**:
```bash
python3 --version
```

If you see something like `Python 3.10.x` or higher, you're good! Skip to Step 2.

### Install Python (if needed)

#### Windows
1. Download Python from [python.org/downloads](https://www.python.org/downloads/)
2. Run the installer
3. **Important**: Check "Add Python to PATH" during installation
4. Click "Install Now"

#### macOS
```bash
# Using Homebrew (recommended)
brew install python@3.10

# Or download from python.org
```

#### Linux (Ubuntu/Debian)
```bash
sudo apt update
sudo apt install python3.10 python3-pip python3-venv
```

#### Linux (Fedora/RHEL)
```bash
sudo dnf install python3.10 python3-pip
```

## 📂 Step 2: Get the Code

You have two options:

### Option A: Download ZIP (Easier)
1. Go to [https://github.com/yb235/emotion-Recognization](https://github.com/yb235/emotion-Recognization)
2. Click the green "Code" button
3. Click "Download ZIP"
4. Extract the ZIP file to a folder of your choice

### Option B: Clone with Git (Recommended)
```bash
# Open terminal and navigate to where you want the project
cd ~/Documents  # or any folder you prefer

# Clone the repository
git clone https://github.com/yb235/emotion-Recognization.git

# Enter the directory
cd emotion-Recognization
```

## 🐍 Step 3: Create Virtual Environment

A virtual environment keeps this project's dependencies separate from other Python projects.

### Windows (Command Prompt)
```bash
# Navigate to project folder
cd path\to\emotion-Recognization

# Create virtual environment
python -m venv venv

# Activate it
venv\Scripts\activate
```

### Windows (PowerShell)
```bash
# Navigate to project folder
cd path\to\emotion-Recognization

# Create virtual environment
python -m venv venv

# Activate it
venv\Scripts\Activate.ps1

# If you get an error, run this first:
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
```

### macOS/Linux
```bash
# Navigate to project folder
cd ~/path/to/emotion-Recognization

# Create virtual environment
python3 -m venv venv

# Activate it
source venv/bin/activate
```

**You'll know it worked** when you see `(venv)` at the start of your command prompt.

## 📦 Step 4: Install Dependencies

With your virtual environment activated, install the required packages:

### Essential Packages
```bash
pip install --upgrade pip
pip install ultralytics opencv-python numpy faster-whisper sounddevice
```

**What each package does**:
- `ultralytics`: YOLO pose detection models
- `opencv-python`: Video capture and display
- `numpy`: Numerical operations
- `faster-whisper`: Fast speech recognition
- `sounddevice`: Audio recording

### Installation Time
- **First time**: 5-10 minutes (downloads ~500 MB)
- **Subsequent installs**: 1-2 minutes

### Platform-Specific Notes

#### macOS (Apple Silicon M1/M2/M3)
You may need to install PortAudio for audio support:
```bash
brew install portaudio
pip install sounddevice
```

#### Linux (Additional Dependencies)
```bash
# Ubuntu/Debian
sudo apt-get install portaudio19-dev python3-pyaudio
sudo apt-get install libgl1-mesa-glx  # For OpenCV

# Fedora
sudo dnf install portaudio-devel
sudo dnf install mesa-libGL
```

#### Windows
No additional steps needed! The packages work out of the box.

## 🎮 Step 5: Optional - GPU Support

For **significantly faster** pose detection, install GPU support:

### NVIDIA GPU (CUDA)

**Check if you have NVIDIA GPU**:
```bash
nvidia-smi  # Windows/Linux
```

**Install PyTorch with CUDA**:
```bash
# CUDA 11.8
pip install torch torchvision --index-url https://download.pytorch.org/whl/cu118

# CUDA 12.1
pip install torch torchvision --index-url https://download.pytorch.org/whl/cu121
```

**Verify GPU is available**:
```bash
python -c "import torch; print(torch.cuda.is_available())"
```
Should print `True` if successful.

**Update code to use GPU**: Change `device="cpu"` to `device="cuda"` in the Python files.

### AMD GPU (DirectML - Windows Only)

**Install ONNX Runtime with DirectML**:
```bash
pip install onnxruntime-directml
```

**Use the ONNX script**: Run `run_pose_onnx_dml.py` instead of `interview_system.py`.

### Apple Silicon (MPS - macOS M1/M2/M3)

**PyTorch already supports MPS**, but it's not always faster for this use case.

**To try it**: Change `device="cpu"` to `device="mps"` in the Python files.

## 🔍 Step 6: Verify Installation

Let's make sure everything is installed correctly.

### Quick Test
```bash
# With virtual environment activated
python -c "import cv2, numpy, faster_whisper, sounddevice; from ultralytics import YOLO; print('All packages installed successfully!')"
```

If you see "All packages installed successfully!" - you're ready!

### Test Webcam
```bash
python -c "import cv2; cap = cv2.VideoCapture(0); ret, frame = cap.read(); cap.release(); print('Webcam working!' if ret else 'Webcam not detected')"
```

### Test Microphone
```bash
python -c "import sounddevice as sd; print('Available audio devices:'); print(sd.query_devices())"
```

You should see a list of your audio devices.

## 📋 Step 7: Download Models (Automatic)

The YOLO models are already included in the repository:
- `yolo11m-pose.pt` (84 MB)
- `yolo11m-pose.onnx` (84 MB)
- `yolov8n-pose.pt` (6.5 MB)

**Whisper models** download automatically on first run:
- First run: Downloads model (~75 MB for "tiny")
- Subsequent runs: Uses cached model

Location of cached models:
- **Windows**: `C:\Users\YourName\.cache\huggingface\`
- **macOS**: `~/.cache/huggingface/`
- **Linux**: `~/.cache/huggingface/`

## 🎉 Step 8: First Run

You're ready to run the system!

### Run Main Application (Pose + Speech)
```bash
# Make sure virtual environment is activated
# Make sure you're in the project folder

python interview_system.py
```

**What should happen**:
1. Console shows "Loading Whisper tiny model on CPU..."
2. Webcam light turns on
3. Window opens showing video feed
4. After ~5-10 seconds, system starts logging
5. Press 'q' to quit

### Run Action Detection Only (No Speech)
```bash
python action_detector3.py
```

**Faster startup** since it doesn't load Whisper.

### Run ONNX Version (Windows with GPU)
```bash
python run_pose_onnx_dml.py
```

**Note**: This version doesn't include action detection logic, just raw pose visualization.

## 🐛 Troubleshooting

### Issue: "python is not recognized"
**Solution**: Python not in PATH. Reinstall Python and check "Add to PATH".

### Issue: "No module named 'cv2'"
**Solution**: 
```bash
# Make sure virtual environment is activated
pip install opencv-python
```

### Issue: "Could not open camera"
**Solution**:
- Check if webcam is connected
- Close other apps using webcam (Zoom, Skype, etc.)
- Try different camera index: Change `VideoCapture(0)` to `VideoCapture(1)`

### Issue: "sounddevice" errors on macOS
**Solution**:
```bash
brew install portaudio
pip install --force-reinstall sounddevice
```

### Issue: Slow performance (< 10 FPS)
**Solutions**:
- Use GPU support (see Step 5)
- Use smaller model: Change to `yolov8n-pose.pt` (faster but less accurate)
- Reduce resolution: Add this before YOLO inference:
  ```python
  frame = cv2.resize(frame, (640, 480))
  ```

### Issue: Whisper model download fails
**Solution**: Manual download:
```bash
# Download directly
python -c "from faster_whisper import WhisperModel; WhisperModel('tiny')"
```

### Issue: "Permission denied" on macOS/Linux
**Solution**:
```bash
# If you get permission errors
chmod +x *.py
```

## 📝 Configuration Tips

### Change Whisper Model Size
In `interview_system.py`, line 142:
```python
# Faster, less accurate
model_stt = WhisperModel("tiny", device="cpu")

# More accurate, slower
model_stt = WhisperModel("base", device="cpu")    # +150 MB
model_stt = WhisperModel("small", device="cpu")   # +450 MB
```

### Change Language
In `interview_system.py`, line 169:
```python
# Auto-detect language
language=None

# Specific language
language="en"  # English
language="zh"  # Chinese
language="es"  # Spanish
language="fr"  # French
```

### Adjust Action Detection Sensitivity
Edit threshold values in `detect_custom_actions()` function. See [Action Detection Algorithms](06_ACTION_DETECTION.md) for details.

## ✅ Verification Checklist

Before proceeding to use the system, verify:

- [ ] Python 3.10+ installed
- [ ] Virtual environment created and activated
- [ ] All packages installed without errors
- [ ] Webcam test passed
- [ ] Microphone test passed
- [ ] YOLO models present in project folder
- [ ] Successfully ran `interview_system.py`
- [ ] JSON files generated after pressing 'q'

## 🎓 Next Steps

Now that you have the system installed:

1. **Learn to use it**: Read the [User Guide](08_USER_GUIDE.md)
2. **Understand the outputs**: Check [Output Data Format](09_OUTPUT_FORMAT.md)
3. **Understand the code**: Review [Code Components](05_CODE_COMPONENTS.md)

## 🔧 Updating the System

To update to the latest version:

### With Git
```bash
# Navigate to project folder
cd ~/path/to/emotion-Recognization

# Pull latest changes
git pull origin main

# Update dependencies
pip install --upgrade ultralytics opencv-python faster-whisper
```

### Without Git
1. Download the latest ZIP from GitHub
2. Extract and replace old files
3. Keep your virtual environment
4. Run `pip install --upgrade` for dependencies

## 🗑️ Uninstalling

To remove the system:

```bash
# Deactivate virtual environment
deactivate

# Delete project folder
rm -rf ~/path/to/emotion-Recognization  # macOS/Linux
# or
rmdir /s /q "path\to\emotion-Recognization"  # Windows
```

Models cached by Whisper will remain in `~/.cache/` - delete manually if desired.

---

← [Previous: Workflow](03_WORKFLOW.md) | [Back to Documentation Home](00_README.md) | [Next: Code Components →](05_CODE_COMPONENTS.md)
