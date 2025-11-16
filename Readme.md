🎯 Interview Behavior and Speech Analysis System

This repository provides an integrated real-time system for analyzing interview behavior based on computer vision and speech recognition. It combines YOLO-based pose estimation with Whisper-based speech transcription, allowing synchronized logging of body gestures and spoken content during interviews. Outputs are recorded in JSON for further psychological, behavioral, or NLP-based analysis.

🔍 Features

🧍 Body Action Detection using YOLO Pose (10 gestures such as arm crossing, chin rest, lean forward, touch face, etc.)

🎤 Real-Time Speech Transcription using Faster-Whisper (local, privacy-safe)

🧠 Parallel Processing of video and audio streams

🕒 Synchronized Timeline Logs across both modalities

📄 Structured JSON outputs:

action_log.json: detected behaviors + timestamps

transcription_log.json: spoken text + timestamps

combined_log.json: merged action + speech events per second

📂 Repository Structure
📁 interview-analysis-system
│
├── interview_system.py      # Main script: action + speech tracking
├── yolo11m-pose.pt          # YOLO pose model (place in root folder)
│
├── action_log.json          # Generated output example
├── transcription_log.json   # Generated output example
├── combined_log.json        # Generated output example
│
├── README.md                # (This file)
└── requirements.txt         # Library dependencies

⚙️ Environment Setup
🔸 1. Requirements

Windows / macOS / Linux

Python 3.10+

Webcam + Microphone

CPU (works best) or GPU if using CUDA

🔸 2. Clone the repository
git clone https://github.com/your-username/interview-analysis-system.git
cd interview-analysis-system

🔸 3. Create a virtual environment
python -m venv venv
source venv/bin/activate     # macOS/Linux
venv\Scripts\activate        # Windows

🔸 4. Install dependencies
pip install -r requirements.txt

Or install manually
pip install ultralytics opencv-python numpy faster-whisper sounddevice


Note: If using CPU-only, install PyTorch CPU wheels:

pip install torch==2.x+cpu torchvision==0.x+cpu --index-url https://download.pytorch.org/whl/cpu

▶️ Usage

Make sure yolo11m-pose.pt is in the same directory.

Run the script:

python interview_system.py


A webcam window will open. Press q on the keyboard to exit.

On exit, the JSON logs will be saved in the project directory.

🧪 Outputs

Example of combined_log.json:

[
  {
    "time": "00:01",
    "timestamp_seconds": 1.02,
    "actions": ["lean_forward", "touch_face"],
    "texts": ["I am very passionate about this role"]
  },
  {
    "time": "00:04",
    "timestamp_seconds": 4.13,
    "actions": ["arms_crossed"],
    "texts": ["I have two years of experience"]
  }
]

🧠 System Overview (Block Diagram)
Camera ─→ YOLO Pose ─→ Action Log ─┐
                                   ├─→ Combined JSON Log
Mic ─────→ Whisper STT ─→ Speech Log ┘

📌 Notes

Speech transcription may have a slight delay (~1–3s) due to chunked inference.

Frame-based action timestamps are near real-time.

All processing is done locally to ensure privacy.

🤝 Acknowledgements

Ultralytics
 for YOLO models

Faster-Whisper
 for optimized speech-to-text

OpenAI Whisper
 for original ASR architecture

WebRTC VAD
 (optional)