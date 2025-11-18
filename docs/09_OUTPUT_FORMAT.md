# Output Data Format

## 📋 Overview

The system generates three JSON files containing structured data about detected actions and transcribed speech. This document explains the format and content of each file.

## 📂 Output Files

| File | Content | Size | Use Case |
|------|---------|------|----------|
| `action_log.json` | All detected actions | Small-Medium | Body language analysis |
| `transcription_log.json` | All transcribed speech | Medium-Large | Speech analysis |
| `combined_log.json` | Merged timeline | Medium-Large | Complete analysis ⭐ |

---

## 1️⃣ action_log.json

### Purpose
Contains every detected body action with precise timestamps.

### Structure
```json
[
  {
    "time": "00:05",
    "timestamp_seconds": 5.42,
    "actions": ["arms_crossed", "lean_back"]
  },
  {
    "time": "00:07",
    "timestamp_seconds": 7.31,
    "actions": ["fidget_hands"]
  }
]
```

### Schema

**Array of Action Entries**

Each entry:
```json
{
  "time": string,              // Format: "MM:SS" (zero-padded)
  "timestamp_seconds": number, // Precise time in seconds (float)
  "actions": string[]          // Array of detected action names
}
```

### Field Descriptions

| Field | Type | Description | Example |
|-------|------|-------------|---------|
| `time` | string | Human-readable timestamp | `"01:23"` |
| `timestamp_seconds` | number | Precise time since start | `83.45` |
| `actions` | array | List of detected actions | `["arms_crossed"]` |

### Possible Action Values

```json
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

### Example File

```json
[
  {
    "time": "00:03",
    "timestamp_seconds": 3.12,
    "actions": ["lean_forward"]
  },
  {
    "time": "00:05",
    "timestamp_seconds": 5.42,
    "actions": ["arms_crossed", "lean_back"]
  },
  {
    "time": "00:05",
    "timestamp_seconds": 5.89,
    "actions": ["arms_crossed"]
  },
  {
    "time": "00:08",
    "timestamp_seconds": 8.23,
    "actions": ["fidget_hands", "touch_face"]
  },
  {
    "time": "00:12",
    "timestamp_seconds": 12.67,
    "actions": ["hands_clasped"]
  }
]
```

### Characteristics

**Frequency**: Variable
- Actions logged only when detected
- Can have multiple entries per second
- Can have gaps (no actions detected)

**Duration**: Not tracked
- Each entry is a point-in-time detection
- To infer duration: Look at consecutive entries

**Multiple Actions**: Common
- One frame can detect multiple actions
- Example: `["arms_crossed", "lean_back"]`

### Analysis Tips

**Count occurrences**:
```python
from collections import Counter
all_actions = []
for entry in data:
    all_actions.extend(entry["actions"])
print(Counter(all_actions))
```

**Find first occurrence**:
```python
for entry in data:
    if "arms_crossed" in entry["actions"]:
        print(f"First arms_crossed at {entry['time']}")
        break
```

**Calculate action duration**:
```python
action = "lean_forward"
times = [e["timestamp_seconds"] for e in data if action in e["actions"]]
if len(times) >= 2:
    duration = max(times) - min(times)
    print(f"{action} lasted {duration:.1f} seconds")
```

---

## 2️⃣ transcription_log.json

### Purpose
Contains all transcribed speech segments with timestamps.

### Structure
```json
[
  {
    "time": "00:03",
    "timestamp_seconds": 3.21,
    "text": "Hello, I am very excited to be here today"
  },
  {
    "time": "00:07",
    "timestamp_seconds": 7.85,
    "text": "I have five years of experience in software development"
  }
]
```

### Schema

**Array of Speech Entries**

Each entry:
```json
{
  "time": string,              // Format: "MM:SS"
  "timestamp_seconds": number, // Approximate time (float)
  "text": string               // Transcribed text
}
```

### Field Descriptions

| Field | Type | Description | Example |
|-------|------|-------------|---------|
| `time` | string | Approximate timestamp | `"00:15"` |
| `timestamp_seconds` | number | Approximate time | `15.34` |
| `text` | string | Transcribed speech | `"Hello world"` |

### Example File

```json
[
  {
    "time": "00:03",
    "timestamp_seconds": 3.21,
    "text": "Hello, I am very excited to be here today"
  },
  {
    "time": "00:07",
    "timestamp_seconds": 7.85,
    "text": "I have five years of experience in software development"
  },
  {
    "time": "00:11",
    "timestamp_seconds": 11.42,
    "text": "My strengths include problem-solving and teamwork"
  },
  {
    "time": "00:16",
    "timestamp_seconds": 16.89,
    "text": "I am very passionate about this opportunity"
  }
]
```

### Characteristics

**Frequency**: ~1 entry per 4 seconds
- System records in 4-second audio chunks
- Actual frequency depends on speech patterns
- Silent periods may be skipped

**Timestamp Accuracy**: ±1-2 seconds
- Approximate due to processing delay
- Shows when transcription completed, not exact speech time

**Text Quality**:
- Generally accurate for clear speech
- May have errors with:
  - Background noise
  - Accents
  - Technical terms
  - Fast speech

### Analysis Tips

**Word count**:
```python
total_words = sum(len(entry["text"].split()) for entry in data)
print(f"Total words spoken: {total_words}")
```

**Speaking rate**:
```python
if data:
    duration = data[-1]["timestamp_seconds"]
    words = sum(len(entry["text"].split()) for entry in data)
    wpm = (words / duration) * 60
    print(f"Speaking rate: {wpm:.1f} words per minute")
```

**Keyword search**:
```python
keyword = "experience"
for entry in data:
    if keyword.lower() in entry["text"].lower():
        print(f"{entry['time']}: {entry['text']}")
```

**Extract all text**:
```python
full_transcript = " ".join(entry["text"] for entry in data)
print(full_transcript)
```

---

## 3️⃣ combined_log.json ⭐

### Purpose
**Most useful file**: Merges actions and speech into a single timeline, grouped by second.

### Structure
```json
[
  {
    "time": "00:05",
    "timestamp_seconds": 5.0,
    "actions": ["arms_crossed", "lean_back"],
    "texts": ["I have five years of experience"]
  },
  {
    "time": "00:08",
    "timestamp_seconds": 8.0,
    "actions": ["fidget_hands"],
    "texts": []
  }
]
```

### Schema

**Array of Combined Entries**

Each entry:
```json
{
  "time": string,              // Format: "MM:SS"
  "timestamp_seconds": number, // Integer seconds
  "actions": string[],         // All actions in this second
  "texts": string[]            // All speech segments in this second
}
```

### Field Descriptions

| Field | Type | Description | Example |
|-------|------|-------------|---------|
| `time` | string | Timestamp | `"01:23"` |
| `timestamp_seconds` | number | Integer second | `83.0` |
| `actions` | array | Unique actions in this second | `["arms_crossed"]` |
| `texts` | array | Speech segments in this second | `["Hello"]` |

### Example File

```json
[
  {
    "time": "00:03",
    "timestamp_seconds": 3.0,
    "actions": ["lean_forward"],
    "texts": ["Hello, I am very excited to be here today"]
  },
  {
    "time": "00:05",
    "timestamp_seconds": 5.0,
    "actions": ["arms_crossed", "lean_back"],
    "texts": ["I have five years of experience"]
  },
  {
    "time": "00:07",
    "timestamp_seconds": 7.0,
    "actions": [],
    "texts": ["in software development"]
  },
  {
    "time": "00:08",
    "timestamp_seconds": 8.0,
    "actions": ["fidget_hands", "touch_face"],
    "texts": []
  },
  {
    "time": "00:11",
    "timestamp_seconds": 11.0,
    "actions": ["hands_clasped"],
    "texts": ["My strengths include problem-solving"]
  }
]
```

### Generation Algorithm

```python
# Pseudocode showing how combined_log is created

combined = {}

# 1. Add all actions, grouped by second
for entry in action_logs:
    sec = int(entry["timestamp_seconds"])
    if sec not in combined:
        combined[sec] = {"actions": [], "texts": []}
    combined[sec]["actions"].extend(entry["actions"])

# 2. Add all speech, grouped by second
for entry in speech_logs:
    sec = int(entry["timestamp_seconds"])
    if sec not in combined:
        combined[sec] = {"actions": [], "texts": []}
    combined[sec]["texts"].append(entry["text"])

# 3. Deduplicate actions, sort by time
for sec in sorted(combined.keys()):
    item = combined[sec]
    item["actions"] = sorted(list(set(item["actions"])))
    # Add time and timestamp fields
```

### Characteristics

**Resolution**: 1 second
- All events within same second are grouped
- `timestamp_seconds` is always an integer

**Empty Fields**: Possible
- `actions` can be empty `[]` (no actions that second)
- `texts` can be empty `[]` (no speech that second)
- Both can be empty (included if other modality had data)

**Action Deduplication**: Yes
- If same action detected multiple times in one second, only listed once
- Actions are sorted alphabetically

**Text Aggregation**: Yes
- Multiple speech segments in same second are kept separate
- Useful for fragmented speech

### Analysis Tips

**Find correlations**:
```python
# When did actions and speech co-occur?
for entry in data:
    if entry["actions"] and entry["texts"]:
        print(f"{entry['time']}: {entry['actions']} + {entry['texts']}")
```

**Timeline view**:
```python
for entry in data:
    actions_str = ", ".join(entry["actions"]) if entry["actions"] else "—"
    texts_str = " ".join(entry["texts"]) if entry["texts"] else "—"
    print(f"{entry['time']} | {actions_str:20} | {texts_str}")
```

**Most active seconds**:
```python
activity = [
    (e["time"], len(e["actions"]) + len(e["texts"]))
    for e in data
]
activity.sort(key=lambda x: x[1], reverse=True)
print("Top 5 most active seconds:")
for time, count in activity[:5]:
    print(f"  {time}: {count} events")
```

---

## 📊 Data Analysis Examples

### Load All Files

```python
import json

# Load all three files
with open("action_log.json") as f:
    action_data = json.load(f)

with open("transcription_log.json") as f:
    speech_data = json.load(f)

with open("combined_log.json") as f:
    combined_data = json.load(f)
```

### Summary Statistics

```python
from collections import Counter

# Action statistics
all_actions = []
for entry in action_data:
    all_actions.extend(entry["actions"])

action_counts = Counter(all_actions)
print("Action Counts:")
for action, count in action_counts.most_common():
    print(f"  {action}: {count}")

# Speech statistics
total_words = sum(len(e["text"].split()) for e in speech_data)
total_duration = speech_data[-1]["timestamp_seconds"] if speech_data else 0

print(f"\nSpeech Statistics:")
print(f"  Total words: {total_words}")
print(f"  Duration: {total_duration:.1f} seconds")
print(f"  Words per minute: {(total_words/total_duration*60):.1f}")
```

### Export to CSV

```python
import csv

with open("combined_log.csv", "w", newline="") as f:
    writer = csv.writer(f)
    writer.writerow(["Time", "Timestamp", "Actions", "Speech"])
    
    for entry in combined_data:
        writer.writerow([
            entry["time"],
            entry["timestamp_seconds"],
            "; ".join(entry["actions"]),
            "; ".join(entry["texts"])
        ])
```

### Visualize Timeline

```python
import matplotlib.pyplot as plt

times = [e["timestamp_seconds"] for e in combined_data]
action_counts = [len(e["actions"]) for e in combined_data]
speech_counts = [len(e["texts"]) for e in combined_data]

plt.figure(figsize=(12, 6))
plt.plot(times, action_counts, label="Actions", marker='o')
plt.plot(times, speech_counts, label="Speech", marker='s')
plt.xlabel("Time (seconds)")
plt.ylabel("Count per second")
plt.title("Activity Timeline")
plt.legend()
plt.grid(True)
plt.show()
```

### Find Patterns

```python
# Find nervous behavior patterns
nervous_actions = ["fidget_hands", "touch_face", "touch_nose", "fix_hair"]

for entry in combined_data:
    nervous_count = sum(1 for a in entry["actions"] if a in nervous_actions)
    if nervous_count >= 2:
        print(f"High nervousness at {entry['time']}:")
        print(f"  Actions: {entry['actions']}")
        print(f"  Speech: {entry['texts']}")
```

---

## 🔍 Data Quality Considerations

### Timestamp Accuracy

**Action Timestamps**: ±0.1 seconds
- Very accurate (frame-level precision)
- Directly tied to video frame time

**Speech Timestamps**: ±1-2 seconds
- Less accurate due to processing delay
- Chunked recording causes batching

**Combined Timestamps**: 1 second resolution
- Grouped by second, losing sub-second precision

### Missing Data

**Why Actions Might Be Missing**:
- Poor lighting
- Person not visible
- Occlusion (body parts hidden)
- Person too far from camera
- Action too subtle

**Why Speech Might Be Missing**:
- Silence
- Background noise too loud
- Microphone issue
- Speech too quiet
- Non-English speech (if language set)

### False Positives

**Actions**:
- Fidget detection: Camera shake or YOLO jitter
- Arms crossed: Holding something across chest
- Touch face: Drinking, eating

**Speech**:
- Background conversations
- TV/radio in background
- Hallucinations (Whisper generating text when none spoken)

### Data Size

**Typical File Sizes** (10-minute session):

| File | Typical Size | Range |
|------|-------------|-------|
| action_log.json | 5-50 KB | 2-100 KB |
| transcription_log.json | 10-30 KB | 5-100 KB |
| combined_log.json | 15-60 KB | 10-150 KB |

**Factors**:
- More actions = larger action_log
- More speech = larger transcription_log
- Longer session = proportionally larger

---

## 💾 Data Storage Best Practices

### Backup Important Sessions

```bash
# Create timestamped backup
timestamp=$(date +%Y%m%d_%H%M%S)
mkdir backup_$timestamp
cp *_log.json backup_$timestamp/
```

### Compress for Archival

```bash
# Compress all JSON files
tar -czf session_data.tar.gz *_log.json

# Or zip
zip session_data.zip *_log.json
```

### Database Storage

For many sessions, consider storing in database:

```python
import sqlite3
import json

conn = sqlite3.connect("sessions.db")
cursor = conn.cursor()

cursor.execute("""
CREATE TABLE IF NOT EXISTS sessions (
    id INTEGER PRIMARY KEY,
    date TEXT,
    participant_id TEXT,
    action_log TEXT,
    speech_log TEXT,
    combined_log TEXT
)
""")

with open("action_log.json") as f:
    action_log = f.read()
# ... (load other files)

cursor.execute("""
INSERT INTO sessions (date, participant_id, action_log, speech_log, combined_log)
VALUES (?, ?, ?, ?, ?)
""", (date, participant_id, action_log, speech_log, combined_log))

conn.commit()
```

---

## 📝 JSON Schema (Formal)

### action_log.json

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "array",
  "items": {
    "type": "object",
    "required": ["time", "timestamp_seconds", "actions"],
    "properties": {
      "time": {
        "type": "string",
        "pattern": "^[0-9]{2}:[0-9]{2}$"
      },
      "timestamp_seconds": {
        "type": "number",
        "minimum": 0
      },
      "actions": {
        "type": "array",
        "items": {
          "type": "string",
          "enum": [
            "arms_crossed", "hands_clasped", "chin_rest",
            "lean_forward", "lean_back", "head_down",
            "touch_face", "touch_nose", "fix_hair", "fidget_hands"
          ]
        }
      }
    }
  }
}
```

### transcription_log.json

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "array",
  "items": {
    "type": "object",
    "required": ["time", "timestamp_seconds", "text"],
    "properties": {
      "time": {
        "type": "string",
        "pattern": "^[0-9]{2}:[0-9]{2}$"
      },
      "timestamp_seconds": {
        "type": "number",
        "minimum": 0
      },
      "text": {
        "type": "string"
      }
    }
  }
}
```

### combined_log.json

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "array",
  "items": {
    "type": "object",
    "required": ["time", "timestamp_seconds", "actions", "texts"],
    "properties": {
      "time": {
        "type": "string",
        "pattern": "^[0-9]{2}:[0-9]{2}$"
      },
      "timestamp_seconds": {
        "type": "number",
        "minimum": 0,
        "multipleOf": 1.0
      },
      "actions": {
        "type": "array",
        "items": {"type": "string"}
      },
      "texts": {
        "type": "array",
        "items": {"type": "string"}
      }
    }
  }
}
```

---

← [Previous: User Guide](08_USER_GUIDE.md) | [Back to Documentation Home](00_README.md) | [Next: Troubleshooting →](10_TROUBLESHOOTING.md)
