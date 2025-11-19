# Logging and Output Generation

## 🎯 Overview

The logging system records detected actions throughout the session and generates structured JSON output files on exit. This document explains the logging mechanism, output formats, and post-processing options.

## 📝 Logging Mechanism

### Action Logging

```python
# Global log storage
action_logs = []  # List of action events

# During processing (Main Thread)
if frame_actions:
    action_logs.append({
        "time": f"{mm:02d}:{ss:02d}",
        "timestamp_seconds": round(ts, 2),
        "actions": list(set(frame_actions))
    })
```

**When Logging Occurs**:
- Every frame where actions are detected
- Actions deduplicated within same frame
- Timestamp relative to `start_time`

**Log Entry Structure**:
```python
{
    "time": "00:15",              # Human-readable time (MM:SS)
    "timestamp_seconds": 15.42,   # Precise timestamp (seconds)
    "actions": [                  # List of detected actions
        "arms_crossed",
        "lean_forward"
    ]
}
```

### Speech Logging

```python
# Global log storage
speech_logs = []  # List of speech events

# During transcription (STT Thread)
speech_logs.append({
    "time": f"{mm:02d}:{ss:02d}",
    "timestamp_seconds": round(ts, 2),
    "text": text
})
```

**Log Entry Structure**:
```python
{
    "time": "00:18",
    "timestamp_seconds": 18.23,
    "text": "I have five years of experience in this field"
}
```

## 📦 Output Files

### File 1: action_log.json

**Purpose**: Record all detected actions with timestamps

**Generation**:
```python
with open("action_log.json", "w", encoding="utf-8") as f:
    json.dump(action_logs, f, ensure_ascii=False, indent=2)
```

**Example Output**:
```json
[
  {
    "time": "00:05",
    "timestamp_seconds": 5.42,
    "actions": ["arms_crossed", "lean_back"]
  },
  {
    "time": "00:08",
    "timestamp_seconds": 8.17,
    "actions": ["lean_forward"]
  },
  {
    "time": "00:12",
    "timestamp_seconds": 12.33,
    "actions": ["touch_face"]
  }
]
```

**Size**: ~100-500 KB for 30-minute session

### File 2: transcription_log.json

**Purpose**: Record all transcribed speech with timestamps

**Generation**:
```python
with open("transcription_log.json", "w", encoding="utf-8") as f:
    json.dump(speech_logs, f, ensure_ascii=False, indent=2)
```

**Example Output**:
```json
[
  {
    "time": "00:03",
    "timestamp_seconds": 3.21,
    "text": "Hello, thank you for this opportunity"
  },
  {
    "time": "00:08",
    "timestamp_seconds": 8.45,
    "text": "I have been working in software development for five years"
  },
  {
    "time": "00:15",
    "timestamp_seconds": 15.67,
    "text": "My experience includes Python and machine learning"
  }
]
```

**Size**: ~50-200 KB for 30-minute session

### File 3: combined_log.json

**Purpose**: Merge actions and speech into unified timeline

**Generation Algorithm**:
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
        combined[sec] = {
            "time": entry["time"],
            "timestamp_seconds": float(sec),
            "actions": [],
            "texts": []
        }
    combined[sec]["texts"].append(entry["text"])

# Deduplicate actions and sort
combined_list = []
for sec in sorted(combined.keys()):
    item = combined[sec]
    item["actions"] = sorted(list(set(item["actions"])))
    combined_list.append(item)

# Save
with open("combined_log.json", "w", encoding="utf-8") as f:
    json.dump(combined_list, f, ensure_ascii=False, indent=2)
```

**Example Output**:
```json
[
  {
    "time": "00:05",
    "timestamp_seconds": 5.0,
    "actions": ["arms_crossed", "lean_back"],
    "texts": []
  },
  {
    "time": "00:08",
    "timestamp_seconds": 8.0,
    "actions": ["lean_forward"],
    "texts": ["I have been working in software development"]
  },
  {
    "time": "00:12",
    "timestamp_seconds": 12.0,
    "actions": ["touch_face"],
    "texts": []
  }
]
```

**Size**: ~150-700 KB for 30-minute session

## ⏱️ Timestamp Management

### Start Time Initialization

```python
start_time = None

# In main loop, first frame
if start_time is None:
    start_time = time.time()
```

**Why First Frame?**
- Camera takes time to initialize
- Ensures both threads synchronized
- Avoids negative timestamps

### Calculating Timestamps

```python
# Current timestamp (seconds since start)
ts = time.time() - start_time

# Format as minutes:seconds
mm = int(ts // 60)
ss = int(ts % 60)
time_str = f"{mm:02d}:{ss:02d}"

# Store both formats
log_entry = {
    "time": time_str,                # Human-readable
    "timestamp_seconds": round(ts, 2)  # Precise
}
```

### Timestamp Precision

```python
# Round to 2 decimal places (10ms precision)
timestamp_seconds = round(ts, 2)  # e.g., 15.42

# Why 2 decimals?
# - 10ms precision sufficient for human actions
# - Reduces file size
# - Matches typical frame rate (30 FPS = 33ms)
```

## 📊 Log Analysis Examples

### Count Action Frequency

```python
import json
from collections import Counter

# Load action log
with open("action_log.json", "r") as f:
    action_logs = json.load(f)

# Count all actions
action_counts = Counter()
for entry in action_logs:
    for action in entry["actions"]:
        action_counts[action] += 1

# Print sorted
print("Action Frequency:")
for action, count in action_counts.most_common():
    print(f"  {action}: {count}")
```

**Output**:
```
Action Frequency:
  fidget_hands: 127
  touch_face: 45
  lean_forward: 32
  arms_crossed: 18
  head_down: 12
```

### Calculate Action Duration

```python
def calculate_action_durations(action_logs):
    """Calculate total duration of each action"""
    action_durations = {}
    
    # Group consecutive action occurrences
    for i, entry in enumerate(action_logs):
        for action in entry["actions"]:
            if action not in action_durations:
                action_durations[action] = []
            
            # Start new duration or extend existing
            ts = entry["timestamp_seconds"]
            if (not action_durations[action] or 
                ts - action_durations[action][-1]["end"] > 2.0):
                # New occurrence (gap > 2 seconds)
                action_durations[action].append({
                    "start": ts,
                    "end": ts
                })
            else:
                # Extend existing occurrence
                action_durations[action][-1]["end"] = ts
    
    # Calculate total durations
    totals = {}
    for action, occurrences in action_durations.items():
        total = sum(occ["end"] - occ["start"] for occ in occurrences)
        totals[action] = {
            "total_seconds": total,
            "occurrences": len(occurrences),
            "avg_duration": total / len(occurrences) if occurrences else 0
        }
    
    return totals

# Usage
durations = calculate_action_durations(action_logs)
for action, stats in durations.items():
    print(f"{action}:")
    print(f"  Total: {stats['total_seconds']:.1f}s")
    print(f"  Occurrences: {stats['occurrences']}")
    print(f"  Avg duration: {stats['avg_duration']:.1f}s")
```

### Timeline Visualization

```python
import matplotlib.pyplot as plt
import numpy as np

def plot_action_timeline(action_logs):
    """Create timeline plot of actions"""
    # Get all unique actions
    all_actions = set()
    for entry in action_logs:
        all_actions.update(entry["actions"])
    
    actions = sorted(list(all_actions))
    
    # Create plot
    fig, ax = plt.subplots(figsize=(15, len(actions)))
    
    # Plot each action
    for i, action in enumerate(actions):
        timestamps = []
        for entry in action_logs:
            if action in entry["actions"]:
                timestamps.append(entry["timestamp_seconds"])
        
        if timestamps:
            ax.scatter(timestamps, [i] * len(timestamps), 
                      s=50, alpha=0.6)
    
    ax.set_yticks(range(len(actions)))
    ax.set_yticklabels(actions)
    ax.set_xlabel("Time (seconds)")
    ax.set_title("Action Timeline")
    ax.grid(True, alpha=0.3)
    
    plt.tight_layout()
    plt.savefig("action_timeline.png")
    plt.close()

# Usage
plot_action_timeline(action_logs)
```

### Sentiment Analysis on Speech

```python
def analyze_speech_sentiment(speech_logs):
    """Basic sentiment analysis on transcribed speech"""
    # Simple word lists (expand as needed)
    positive_words = ["good", "great", "excellent", "happy", "confident", 
                     "excited", "passionate", "love"]
    negative_words = ["bad", "poor", "difficult", "worried", "nervous",
                     "scared", "hate", "problem"]
    
    sentiment_scores = []
    
    for entry in speech_logs:
        text = entry["text"].lower()
        words = text.split()
        
        pos_count = sum(1 for word in words if word in positive_words)
        neg_count = sum(1 for word in words if word in negative_words)
        
        # Simple sentiment score: positive - negative
        score = pos_count - neg_count
        
        sentiment_scores.append({
            "time": entry["time"],
            "text": entry["text"],
            "sentiment": score
        })
    
    return sentiment_scores

# Usage
with open("transcription_log.json", "r") as f:
    speech_logs = json.load(f)

sentiments = analyze_speech_sentiment(speech_logs)
avg_sentiment = np.mean([s["sentiment"] for s in sentiments])
print(f"Average sentiment: {avg_sentiment:.2f}")
```

## 🔄 Alternative Output Formats

### CSV Export

```python
import csv

def export_to_csv(action_logs, filename="action_log.csv"):
    """Export action logs to CSV format"""
    with open(filename, "w", newline="", encoding="utf-8") as f:
        writer = csv.writer(f)
        
        # Header
        writer.writerow(["Time", "Timestamp_Seconds", "Actions"])
        
        # Data
        for entry in action_logs:
            writer.writerow([
                entry["time"],
                entry["timestamp_seconds"],
                ", ".join(entry["actions"])
            ])

# Usage
export_to_csv(action_logs)
```

### Database Storage

```python
import sqlite3

def store_in_database(action_logs, speech_logs, db_name="interview_data.db"):
    """Store logs in SQLite database"""
    conn = sqlite3.connect(db_name)
    cursor = conn.cursor()
    
    # Create tables
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS actions (
            id INTEGER PRIMARY KEY,
            timestamp_seconds REAL,
            time TEXT,
            action TEXT
        )
    """)
    
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS speech (
            id INTEGER PRIMARY KEY,
            timestamp_seconds REAL,
            time TEXT,
            text TEXT
        )
    """)
    
    # Insert action data
    for entry in action_logs:
        for action in entry["actions"]:
            cursor.execute(
                "INSERT INTO actions (timestamp_seconds, time, action) VALUES (?, ?, ?)",
                (entry["timestamp_seconds"], entry["time"], action)
            )
    
    # Insert speech data
    for entry in speech_logs:
        cursor.execute(
            "INSERT INTO speech (timestamp_seconds, time, text) VALUES (?, ?, ?)",
            (entry["timestamp_seconds"], entry["time"], entry["text"])
        )
    
    conn.commit()
    conn.close()

# Usage
store_in_database(action_logs, speech_logs)
```

### HTML Report

```python
def generate_html_report(action_logs, speech_logs, output_file="report.html"):
    """Generate HTML report with visualizations"""
    html = """
    <!DOCTYPE html>
    <html>
    <head>
        <title>Interview Analysis Report</title>
        <style>
            body { font-family: Arial, sans-serif; margin: 20px; }
            .event { margin: 10px 0; padding: 10px; background: #f0f0f0; }
            .action { color: green; font-weight: bold; }
            .speech { color: blue; font-style: italic; }
        </style>
    </head>
    <body>
        <h1>Interview Analysis Report</h1>
    """
    
    # Merge and sort events
    events = []
    for entry in action_logs:
        events.append({
            "type": "action",
            "timestamp": entry["timestamp_seconds"],
            "time": entry["time"],
            "content": ", ".join(entry["actions"])
        })
    
    for entry in speech_logs:
        events.append({
            "type": "speech",
            "timestamp": entry["timestamp_seconds"],
            "time": entry["time"],
            "content": entry["text"]
        })
    
    events.sort(key=lambda x: x["timestamp"])
    
    # Add events to HTML
    for event in events:
        html += f"""
        <div class="event">
            <strong>[{event['time']}]</strong>
            <span class="{event['type']}">{event['content']}</span>
        </div>
        """
    
    html += """
    </body>
    </html>
    """
    
    with open(output_file, "w", encoding="utf-8") as f:
        f.write(html)

# Usage
generate_html_report(action_logs, speech_logs)
```

## 🧹 Cleanup and Rotation

### Log Rotation

```python
import os
from datetime import datetime

def save_logs_with_rotation(action_logs, speech_logs):
    """Save logs with timestamp in filename"""
    timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
    
    # Create logs directory if doesn't exist
    os.makedirs("logs", exist_ok=True)
    
    # Save with timestamp
    with open(f"logs/action_log_{timestamp}.json", "w") as f:
        json.dump(action_logs, f, indent=2)
    
    with open(f"logs/transcription_log_{timestamp}.json", "w") as f:
        json.dump(speech_logs, f, indent=2)
    
    print(f"Logs saved with timestamp: {timestamp}")
```

### Archive Old Logs

```python
import shutil
import zipfile

def archive_old_logs(days_old=7):
    """Archive logs older than specified days"""
    import time
    
    now = time.time()
    archive_name = f"logs_archive_{datetime.now().strftime('%Y%m%d')}.zip"
    
    with zipfile.ZipFile(archive_name, "w") as zipf:
        for filename in os.listdir("logs"):
            filepath = os.path.join("logs", filename)
            
            # Check age
            if os.path.getmtime(filepath) < now - (days_old * 86400):
                zipf.write(filepath)
                os.remove(filepath)  # Remove after archiving
    
    print(f"Archived old logs to {archive_name}")
```

## 📚 Further Reading

- **[01_MAIN_THREAD_OVERVIEW.md](01_MAIN_THREAD_OVERVIEW.md)** - Overall architecture
- **[09_THREADING_AND_SYNC.md](09_THREADING_AND_SYNC.md)** - Thread coordination
- **[../09_OUTPUT_FORMAT.md](../09_OUTPUT_FORMAT.md)** - Detailed output format specs

---

← [Previous: Threading and Sync](09_THREADING_AND_SYNC.md) | [Back to Main Thread Docs](00_README.md)
