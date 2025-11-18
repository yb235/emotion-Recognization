# System Overview

## 🎯 What Is This System?

The **Interview Behavior and Speech Analysis System** is an AI-powered tool that monitors and records both **body language** and **speech** during interviews or conversations. It helps analyze human behavior by capturing:

1. **Body Actions** - 10 different types of gestures and postures
2. **Spoken Words** - Everything said during the session, transcribed in real-time
3. **Timing** - When each action or word occurred

Think of it as a smart assistant that watches and listens during an interview, taking detailed notes about both what someone says and how they behave.

## 🎭 Why Was This Built?

This system addresses several real-world needs:

### 1. Interview Analysis
- **For Recruiters**: Understand candidate body language and communication patterns
- **For Coaches**: Provide feedback on interview performance
- **For Researchers**: Study behavioral patterns during conversations

### 2. Self-Improvement
- **Practice Interviews**: Record yourself and review your body language
- **Communication Training**: Identify nervous habits or distracting gestures
- **Public Speaking**: Improve your presentation skills

### 3. Behavioral Research
- **Psychology Studies**: Analyze correlations between body language and speech
- **Sentiment Analysis**: Combine verbal and non-verbal cues
- **Data Collection**: Gather structured data for ML/AI training

## 🧩 Key Components

### Component 1: Body Language Detection (Computer Vision)

**What it does**: Watches the video feed and identifies body movements

**How it works**:
- Uses a webcam to capture video in real-time
- Uses YOLO (a computer vision AI model) to detect human pose
- Analyzes 17 body keypoints (joints) to identify specific actions
- Displays detected actions on screen

**10 Detected Actions**:
1. **Arms Crossed** - Defensive or closed posture
2. **Hands Clasped** - Nervous or formal gesture
3. **Chin Rest** - Thinking or boredom
4. **Lean Forward** - Interest and engagement
5. **Lean Back** - Relaxation or disinterest
6. **Head Down** - Submission or lack of confidence
7. **Touch Face** - Self-soothing or nervousness
8. **Touch Nose** - Specific anxious gesture
9. **Fix Hair** - Self-grooming, nervousness
10. **Fidget Hands** - Nervousness, restlessness

### Component 2: Speech Transcription (Audio Processing)

**What it does**: Listens to the microphone and converts speech to text

**How it works**:
- Records audio from your microphone in 4-second chunks
- Uses Whisper AI (speech recognition model) to transcribe
- Runs locally on your computer (no internet needed)
- Displays recognized text on screen in real-time

### Component 3: Data Logging System

**What it does**: Records everything with timestamps

**Outputs**:
- `action_log.json` - All detected body actions with timestamps
- `transcription_log.json` - All spoken words with timestamps
- `combined_log.json` - Both actions and speech merged by time

## 🔄 How It All Works Together

```
┌─────────────┐          ┌──────────────────┐
│   Webcam    │──────►  │  YOLO Pose Model │
└─────────────┘          │  (Body Actions)  │
                         └─────────┬────────┘
                                   │
                                   ▼
                         ┌─────────────────┐
                         │  Action Logger  │
                         │  (JSON Output)  │
                         └────────┬────────┘
                                  │
┌─────────────┐                  │          ┌─────────────────┐
│ Microphone  │──────►           ▼          │  On-Screen      │
└─────────────┘     │   ┌─────────────────┐ │  Display        │
                    │   │  Combined Log   │◄┤  - Actions      │
                    │   │  Generator      │ │  - Subtitles    │
                    │   └─────────────────┘ └─────────────────┘
                    │            ▲
                    ▼            │
         ┌──────────────────┐   │
         │ Whisper Model    │───┘
         │ (Speech to Text) │
         └──────────────────┘
                    │
                    ▼
         ┌──────────────────┐
         │ Speech Logger    │
         │ (JSON Output)    │
         └──────────────────┘
```

## 🌟 Key Features Explained

### Real-Time Processing
The system processes video and audio **simultaneously** using multi-threading:
- One thread handles video frames and pose detection
- Another thread handles audio recording and transcription
- Both run in parallel for efficient performance

### Synchronized Timestamps
Everything is logged with precise timestamps:
- Actions are recorded when they occur in the video
- Speech is recorded when it's detected in the audio
- Combined log merges both by second for easy analysis

### Privacy-First Design
All processing happens **locally** on your computer:
- No data sent to external servers
- No internet connection required (after installing models)
- Complete control over your data

### Visual Feedback
The system shows what it's detecting in real-time:
- Green text shows detected actions (e.g., "ACTION: arms_crossed")
- Yellow text at bottom shows transcribed speech
- Window title: "Interview Action + Speech Monitor"

## 📊 Use Case Example

**Scenario**: You're practicing for a job interview

1. **Start the system**: Run `python interview_system.py`
2. **Conduct mock interview**: Answer practice questions
3. **System observes**: 
   - Records that you crossed your arms at 00:15
   - Transcribes: "I have five years of experience"
   - Records that you leaned forward at 00:18
   - Transcribes: "I'm very passionate about this role"
4. **Stop recording**: Press 'q' when done
5. **Review data**: Open JSON files to see:
   - When you showed confident postures
   - When you showed nervous gestures
   - What you said and how you said it

## 🎓 Technical Level

This system is:
- **User-Friendly**: No coding required to use
- **Developer-Friendly**: Clean Python code, easy to modify
- **Research-Ready**: Structured data output for analysis

## 🚀 Different Modes Available

The repository includes multiple ways to use the system:

### 1. Full System (`interview_system.py`)
- Body language + Speech recognition
- **Best for**: Complete interview analysis
- **Requires**: Webcam + Microphone

### 2. Action Detection Only (`action_detector3.py`)
- Body language detection only
- **Best for**: Quick gesture analysis
- **Requires**: Webcam only

### 3. ONNX Accelerated (`run_pose_onnx_dml.py`)
- Optimized pose detection with DirectML
- **Best for**: Windows systems with GPU
- **Requires**: Webcam + ONNX runtime

## 💡 Who Should Use This?

- **Job Seekers**: Practice and improve interview skills
- **Recruiters**: Analyze candidate behavior patterns
- **Researchers**: Collect behavioral data for studies
- **Coaches**: Provide data-driven feedback
- **Developers**: Build upon the system or integrate into other tools

## 🔜 Next Steps

Now that you understand what the system does, proceed to:
- [Architecture](02_ARCHITECTURE.md) - Learn how it's built
- [Installation Guide](04_INSTALLATION.md) - Set up the system
- [User Guide](08_USER_GUIDE.md) - Start using it

---

← [Back to Documentation Home](00_README.md) | [Next: Architecture →](02_ARCHITECTURE.md)
