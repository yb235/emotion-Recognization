# Main Thread (Video Processing) Module Documentation

## 📚 Overview

This folder contains in-depth technical documentation for the **Main Thread (Video Processing)** module, which is the core component responsible for real-time video capture, pose detection, action recognition, and UI rendering in the Interview Behavior Analysis System.

## 📖 Documentation Structure

### Core Documentation

1. **[01_MAIN_THREAD_OVERVIEW.md](01_MAIN_THREAD_OVERVIEW.md)**
   - High-level architecture of the Main Thread
   - Responsibilities and scope
   - Integration with other system components
   - Performance characteristics

2. **[02_VIDEO_CAPTURE_PIPELINE.md](02_VIDEO_CAPTURE_PIPELINE.md)**
   - Webcam initialization and configuration
   - Frame capture process
   - Timing and synchronization
   - Error handling

3. **[03_YOLO_MODEL_INFERENCE.md](03_YOLO_MODEL_INFERENCE.md)**
   - YOLO11 Pose model architecture
   - Inference pipeline in detail
   - Input preprocessing and output postprocessing
   - Performance optimization
   - Device selection (CPU/GPU)

4. **[04_KEYPOINT_EXTRACTION.md](04_KEYPOINT_EXTRACTION.md)**
   - COCO keypoint format
   - Extracting keypoints from YOLO results
   - Coordinate system and transformations
   - Multi-person handling

5. **[05_ACTION_DETECTION_ENGINE.md](05_ACTION_DETECTION_ENGINE.md)**
   - Action detection algorithm architecture
   - The detect_custom_actions() function in depth
   - Geometric calculations and heuristics
   - State management for temporal actions

6. **[06_CONFIGURING_DETECTIONS.md](06_CONFIGURING_DETECTIONS.md)**
   - Adjusting detection thresholds
   - Tuning for different environments
   - Camera distance and resolution considerations
   - Performance vs accuracy trade-offs

7. **[07_ADDING_NEW_ACTIONS.md](07_ADDING_NEW_ACTIONS.md)**
   - Step-by-step guide to adding new action detection
   - Best practices for defining actions
   - Testing and validation strategies
   - Common pitfalls and solutions

8. **[08_UI_RENDERING.md](08_UI_RENDERING.md)**
   - Frame annotation and overlay
   - Drawing detected actions
   - Displaying subtitles
   - UI performance considerations

9. **[09_THREADING_AND_SYNC.md](09_THREADING_AND_SYNC.md)**
   - Main Thread vs STT Thread coordination
   - Shared state management
   - Synchronization points
   - Thread-safety considerations

10. **[10_LOGGING_AND_OUTPUT.md](10_LOGGING_AND_OUTPUT.md)**
    - Action logging mechanism
    - Timestamp generation
    - Log structure and format
    - Final JSON generation

## 🎯 Who Should Read This?

### Developers
- Understanding the video processing pipeline
- Adding new action detection features
- Optimizing performance
- Debugging detection issues

### Researchers
- Understanding action detection algorithms
- Modifying detection logic
- Collecting training data
- Evaluating system performance

### System Integrators
- Integrating with other systems
- Understanding data flow
- Modifying input/output formats
- Scaling considerations

## 🔗 Related Documentation

This Main Thread documentation complements the existing system documentation:

- **[../02_ARCHITECTURE.md](../02_ARCHITECTURE.md)** - Overall system architecture
- **[../03_WORKFLOW.md](../03_WORKFLOW.md)** - Complete execution flow
- **[../05_CODE_COMPONENTS.md](../05_CODE_COMPONENTS.md)** - File-by-file code reference
- **[../06_ACTION_DETECTION.md](../06_ACTION_DETECTION.md)** - Action detection algorithms

## 🚀 Quick Start

If you're new to the Main Thread module, we recommend reading in this order:

1. Start with **01_MAIN_THREAD_OVERVIEW.md** for the big picture
2. Read **03_YOLO_MODEL_INFERENCE.md** to understand pose detection
3. Study **05_ACTION_DETECTION_ENGINE.md** for action recognition
4. Follow **07_ADDING_NEW_ACTIONS.md** to customize the system

## 💡 Key Concepts

### Main Thread Responsibilities
- Video frame capture (30 FPS)
- Pose estimation with YOLO
- Action detection from keypoints
- UI rendering and display
- Event logging
- User input handling

### Performance Characteristics
- **Throughput**: 30 frames per second (typical)
- **Latency**: 50-100ms per frame (CPU)
- **Memory**: ~140 MB (action_detector3.py)
- **CPU Usage**: 40-60% (single core)

### Threading Model
The Main Thread runs synchronously, processing:
```
Frame Capture → YOLO Inference → Action Detection → UI Rendering → Display
     ↓              ↓                  ↓                 ↓            ↓
   10ms          80ms               1ms              5ms          1ms
```

## 📝 Code Examples

All documentation includes practical code examples. Key files referenced:

- `interview_system.py` - Full system with speech
- `action_detector3.py` - Simplified action detection only
- `run_pose_onnx_dml.py` - ONNX-optimized inference

## 🔧 Configuration Reference

### Key Parameters Covered
- YOLO model selection
- Device configuration (CPU/GPU)
- Detection thresholds (all 10 actions)
- Video source configuration
- Frame rate and resolution
- UI styling

## 📊 Performance Optimization

Topics covered across the documentation:
- Device selection strategies
- Model size trade-offs
- Batch processing (when applicable)
- Threshold tuning for speed
- UI rendering optimization

## 🐛 Debugging Guide

Common issues addressed:
- Low detection accuracy
- Performance bottlenecks
- Threading issues
- False positives/negatives
- Memory leaks

## 🤝 Contributing

When adding new actions or modifying the Main Thread:
1. Read **07_ADDING_NEW_ACTIONS.md**
2. Follow existing code patterns
3. Test thoroughly with edge cases
4. Update relevant documentation
5. Add code examples

## 📧 Support

For questions about the Main Thread module:
- Check the troubleshooting sections in each document
- Review existing action detection logic
- Consult the main [../10_TROUBLESHOOTING.md](../10_TROUBLESHOOTING.md)

---

**Last Updated**: 2025-11-19  
**Module Version**: 1.0  
**Compatibility**: Python 3.10+, YOLO11, OpenCV 4.x
