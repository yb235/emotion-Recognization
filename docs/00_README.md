# Emotion Recognition System - Complete Documentation

Welcome to the comprehensive documentation for the Interview Behavior and Speech Analysis System! This documentation is designed to help first-time users understand everything about the project.

## 📚 Documentation Structure

This documentation is organized into the following sections:

### Core Documentation
1. **[System Overview](01_SYSTEM_OVERVIEW.md)** - High-level understanding of what the system does
2. **[Architecture](02_ARCHITECTURE.md)** - Technical architecture and system design
3. **[Workflow](03_WORKFLOW.md)** - Step-by-step explanation of how the system works
4. **[Installation Guide](04_INSTALLATION.md)** - Complete setup instructions

### Code Documentation
5. **[Main Components](05_CODE_COMPONENTS.md)** - Detailed explanation of each Python file
6. **[Action Detection Algorithms](06_ACTION_DETECTION.md)** - How body gestures are detected
7. **[API Reference](07_API_REFERENCE.md)** - Functions and their parameters

### User Guides
8. **[User Guide](08_USER_GUIDE.md)** - How to use the system effectively
9. **[Output Data Format](09_OUTPUT_FORMAT.md)** - Understanding the JSON output files
10. **[Troubleshooting](10_TROUBLESHOOTING.md)** - Common issues and solutions

## 🎯 Quick Start for First-Time Users

If you're new to this project, we recommend reading the documentation in this order:

1. Start with **System Overview** to understand what the system does
2. Read **Architecture** to understand how it's built
3. Follow the **Installation Guide** to set up your environment
4. Review the **User Guide** to run the system
5. Check **Code Components** when you want to understand or modify the code

## 🔍 What This System Does

This is an **AI-powered interview analysis system** that:
- **Monitors body language** during interviews using computer vision (10 different gestures)
- **Transcribes speech** in real-time using AI speech recognition
- **Logs everything** in structured JSON format with timestamps
- **Works locally** - no data sent to external servers (privacy-safe)

## 💡 Key Features

- ✅ Real-time body pose estimation with YOLO
- ✅ Real-time speech transcription with Whisper
- ✅ 10 distinct body action detections
- ✅ Synchronized video and audio processing
- ✅ Structured JSON outputs for analysis
- ✅ On-screen visualization of detected actions
- ✅ Multiple deployment options (PyTorch, ONNX)

## 🛠️ Technology Stack

- **Computer Vision**: Ultralytics YOLO11m-pose
- **Speech Recognition**: Faster-Whisper (CPU-optimized)
- **Video Processing**: OpenCV
- **Audio Processing**: sounddevice
- **Programming Language**: Python 3.10+
- **Hardware Acceleration**: Optional (ONNX + DirectML)

## 📦 Repository Files

```
emotion-Recognization/
├── interview_system.py       # Main application (pose + speech)
├── action_detector3.py       # Standalone pose detection only
├── run_pose_onnx_dml.py     # ONNX-based pose detection (DirectML)
├── export_pose.py           # Model export utility (PyTorch → ONNX)
├── yolo11m-pose.pt          # YOLO11 PyTorch model
├── yolo11m-pose.onnx        # YOLO11 ONNX model
├── yolov8n-pose.pt          # YOLO8 lightweight model
├── Readme.md                # Basic project README
└── docs/                    # Complete documentation (you are here!)
```

## 🎓 Learning Path

### For Users (No Coding Experience)
1. System Overview
2. Installation Guide
3. User Guide
4. Output Data Format
5. Troubleshooting

### For Developers (Want to Understand Code)
1. System Overview
2. Architecture
3. Workflow
4. Code Components
5. Action Detection Algorithms
6. API Reference

### For Contributors (Want to Modify/Extend)
1. All of the above
2. Deep dive into specific code files in **Code Components**
3. Study **Action Detection Algorithms** to add new gestures
4. Review **API Reference** for integration points

## 🤝 Getting Help

- **Setup Issues**: Check [Installation Guide](04_INSTALLATION.md) and [Troubleshooting](10_TROUBLESHOOTING.md)
- **Usage Questions**: Refer to [User Guide](08_USER_GUIDE.md)
- **Understanding Code**: Review [Code Components](05_CODE_COMPONENTS.md) and [API Reference](07_API_REFERENCE.md)
- **Output Data**: See [Output Data Format](09_OUTPUT_FORMAT.md)

## 📝 Note

This documentation assumes you're a first-time user with basic computer skills. Technical terms are explained where they first appear, and code examples include detailed comments.

---

**Ready to begin?** Start with the [System Overview](01_SYSTEM_OVERVIEW.md) →
