# Threading and Synchronization

## 🎯 Overview

This document explains how the Main Thread (Video Processing) coordinates with the STT Thread (Audio Processing) using shared state, synchronization points, and thread-safe communication patterns.

## 🧵 Thread Architecture

```
┌────────────────────────────────────────────────────────┐
│                   Main Process                         │
├────────────────────────────────────────────────────────┤
│                                                         │
│  ┌─────────────────────┐     ┌────────────────────┐  │
│  │   Main Thread       │     │   STT Thread       │  │
│  │   (Primary)         │     │   (Daemon)         │  │
│  │                     │     │                    │  │
│  │  - Video capture    │     │  - Audio capture   │  │
│  │  - YOLO inference   │     │  - Whisper STT     │  │
│  │  - Action detection │     │  - Speech logging  │  │
│  │  - UI rendering     │     │                    │  │
│  │  - User input       │     │                    │  │
│  │  - Orchestration    │     │                    │  │
│  └─────────────────────┘     └────────────────────┘  │
│           │                           │               │
│           └───────────┬───────────────┘               │
│                       ↓                               │
│           ┌────────────────────┐                     │
│           │   Shared State     │                     │
│           │  (Global Variables)│                     │
│           │                    │                     │
│           │  - start_time      │                     │
│           │  - stop_flag       │                     │
│           │  - action_logs     │                     │
│           │  - speech_logs     │                     │
│           │  - current_subtitle│                     │
│           └────────────────────┘                     │
└────────────────────────────────────────────────────────┘
```

## 📊 Shared State Variables

### Overview Table

| Variable | Type | Writer | Reader | Purpose |
|----------|------|--------|--------|---------|
| `start_time` | float | Main | STT | Synchronize timing |
| `stop_flag` | bool | Main | STT | Signal shutdown |
| `action_logs` | list | Main | Main | Store detected actions |
| `speech_logs` | list | STT | Main | Store transcribed speech |
| `current_subtitle` | str | STT | Main | Display latest speech |
| `prev_left_wrist` | array | Main | Main | Track wrist movement |
| `prev_right_wrist` | array | Main | Main | Track wrist movement |

### Variable Details

#### 1. start_time

```python
# Global declaration
start_time = None  # Initially None

# Set by Main Thread (once)
if start_time is None:
    start_time = time.time()
    print("[MAIN] Timing started from first frame.")

# Read by STT Thread (polling)
while start_time is None and not stop_flag:
    time.sleep(0.1)  # Wait for main thread to start
```

**Purpose**: 
- Provides common time reference for both threads
- Ensures both threads use same timestamp baseline
- STT thread waits for this before starting recording

**Type**: `float` (Unix timestamp) or `None`

**Thread Safety**: Safe due to Python GIL + single writer

#### 2. stop_flag

```python
# Global declaration
stop_flag = False

# Set by Main Thread (on exit)
if cv2.waitKey(1) & 0xFF == ord('q'):
    break  # Exit main loop
# After loop
stop_flag = True

# Checked by STT Thread (in loops)
while not stop_flag:
    # Record audio
    audio = sd.rec(...)
    sd.wait()
    
    if stop_flag:  # Check again after blocking operation
        break
```

**Purpose**:
- Signals STT thread to exit gracefully
- Allows clean shutdown without force-killing thread

**Type**: `bool`

**Thread Safety**: Safe (simple boolean, GIL protection)

#### 3. action_logs

```python
# Global declaration
action_logs = []

# Appended by Main Thread (frequently)
if frame_actions:
    action_logs.append({
        "time": f"{mm:02d}:{ss:02d}",
        "timestamp_seconds": round(ts, 2),
        "actions": list(set(frame_actions))
    })

# Read by Main Thread (on exit)
with open("action_log.json", "w") as f:
    json.dump(action_logs, f, indent=2)
```

**Purpose**: Accumulate all detected actions over time

**Type**: `list` of `dict`

**Thread Safety**: 
- Only Main Thread accesses
- No inter-thread conflict

#### 4. speech_logs

```python
# Global declaration
speech_logs = []

# Appended by STT Thread (every ~4 seconds)
speech_logs.append({
    "time": f"{mm:02d}:{ss:02d}",
    "timestamp_seconds": round(ts, 2),
    "text": text
})

# Read by Main Thread (on exit)
with open("transcription_log.json", "w") as f:
    json.dump(speech_logs, f, indent=2)
```

**Purpose**: Accumulate all transcribed speech over time

**Type**: `list` of `dict`

**Thread Safety**: 
- STT writes, Main reads (at end)
- No concurrent access during runtime

#### 5. current_subtitle

```python
# Global declaration
current_subtitle = ""

# Set by STT Thread (every ~4 seconds)
for seg in segments:
    text = seg.text.strip()
    current_subtitle = text  # Update
    print(f"[STT {mm:02d}:{ss:02d}] {text}")

# Read by Main Thread (every frame)
if current_subtitle:
    cv2.putText(frame, current_subtitle, (10, h - 20), ...)
```

**Purpose**: Share latest speech text for on-screen display

**Type**: `str`

**Thread Safety**:
- STT writes occasionally (~4s intervals)
- Main reads frequently (every frame)
- String assignment is atomic in Python
- Race condition impact is minimal (worst case: display old text for one frame)

## 🔄 Synchronization Points

### Point 1: Startup Synchronization

**Sequence**:
```
Time  | Main Thread                    | STT Thread
------|--------------------------------|---------------------------
t0    | Load YOLO model                | Created (waiting)
t1    | Open webcam                    | Load Whisper model
t2    | Read first frame               | While start_time is None:
t3    | Set start_time = time.time()   |     sleep(0.1)
t4    | Start processing               | Start recording (synchronized!)
```

**Code**:

Main Thread:
```python
# First frame
ret, frame = cap.read()
if start_time is None:
    start_time = time.time()  # ← Sets synchronization point
```

STT Thread:
```python
# Wait for main thread
while start_time is None and not stop_flag:
    time.sleep(0.1)  # ← Polling

# Now synchronized, start recording
print("[STT] Start listening...")
```

**Why This Works**:
- STT thread polls every 100ms (low overhead)
- Main thread sets `start_time` on first successful frame
- Both threads now use same time reference

### Point 2: Runtime Communication

**Continuous, non-blocking**:

```
Main Thread (30 FPS)          STT Thread (~0.25 Hz)
      │                              │
      │◄─────── read ────────────────│ current_subtitle
      │                              │
      │◄─────── read ────────────────│ current_subtitle
      │                              │
      │◄─────── read ────────────────│ current_subtitle
      │                              │
      │                              ├─ Transcribe
      │                              │
      │                              ├─ Update ──────►│ current_subtitle
      │◄─────── read ────────────────│ current_subtitle
      │                              │
```

**Characteristics**:
- No blocking between threads
- Main thread displays whatever is in `current_subtitle`
- STT updates `current_subtitle` when new text available
- No locks needed (simple string assignment)

### Point 3: Shutdown Synchronization

**Sequence**:
```
Time  | Main Thread                    | STT Thread
------|--------------------------------|---------------------------
t0    | User presses 'q'               | Recording...
t1    | Break from main loop           | Recording...
t2    | cap.release()                  | Recording...
t3    | cv2.destroyAllWindows()        | Recording...
t4    | Set stop_flag = True           | Recording...
t5    | time.sleep(0.5)                | Check stop_flag → True
t6    | Continue to log generation     | Exit gracefully
t7    | Generate JSON logs             | (Thread terminated)
t8    | Exit program                   |
```

**Code**:

Main Thread:
```python
# Exit loop
cap.release()
cv2.destroyAllWindows()

# Signal STT thread
stop_flag = True

# Wait for graceful exit
time.sleep(0.5)

# Proceed with log generation
save_logs()
```

STT Thread:
```python
while not stop_flag:
    # Record audio (4 seconds, blocking)
    audio = sd.rec(...)
    sd.wait()
    
    if stop_flag:  # Check after blocking
        break
    
    # Transcribe...

print("[STT] Stopped.")  # Graceful exit
```

**Why 0.5 Second Wait?**
- STT thread may be in 4-second recording
- Gives time to complete current chunk
- Prevents abrupt termination
- Better than force-killing thread

## 🔒 Thread Safety Mechanisms

### Python GIL (Global Interpreter Lock)

**What is GIL?**
- Python's built-in thread safety mechanism
- Only one thread executes Python bytecode at a time
- Automatically protects simple operations

**What GIL Protects** (our case):
- Reading/writing `start_time` (float)
- Reading/writing `stop_flag` (bool)
- Reading/writing `current_subtitle` (string)
- Appending to lists (`action_logs`, `speech_logs`)

**What GIL Doesn't Guarantee**:
- Atomic compound operations
- Order of operations across threads
- Progress guarantees (thread may be suspended)

### No Locks Needed!

Our design avoids the need for explicit locks:

```python
# ✅ SAFE: Simple variable assignment
current_subtitle = "new text"

# ✅ SAFE: List append (atomic operation)
action_logs.append({"time": "00:01", ...})

# ✅ SAFE: Boolean check
if stop_flag:
    break

# ❌ UNSAFE (but we don't do this):
# action_logs = sorted(action_logs)  # Modifying while other thread reads
```

**Why No Locks?**
1. **Single Writer per Variable**: Each variable written by one thread only
2. **Simple Operations**: Only assignments and appends
3. **No Concurrent Modifications**: Lists built during runtime, read only at end
4. **Loose Coupling**: Threads don't depend on immediate consistency

### Memory Barriers and Visibility

**Python's Approach**:
- GIL acts as implicit memory barrier
- Changes visible to other threads when GIL released
- Our blocking I/O operations (camera, audio) release GIL frequently

**Practical Impact**:
- `current_subtitle` updates visible within milliseconds
- `stop_flag` seen by STT thread within 100ms (polling interval)
- No stale data issues in practice

## 🎭 Daemon Thread Behavior

### STT Thread as Daemon

```python
stt_thread = threading.Thread(target=stt_worker, daemon=True)
stt_thread.start()
```

**What daemon=True Means**:
- Thread automatically terminates when main thread exits
- No need to explicitly join()
- Prevents program hanging if thread doesn't exit

**Trade-offs**:
- ✅ Simpler code (no explicit thread management)
- ✅ Prevents zombie processes
- ❌ Thread may be killed mid-operation (we handle with stop_flag)

### Graceful Shutdown Pattern

Despite daemon=True, we implement graceful shutdown:

```python
# Main thread
stop_flag = True  # Signal intent to exit
time.sleep(0.5)   # Give time to respond
# Then exit (daemon thread auto-terminates if still running)
```

**Why Both?**
- `stop_flag` allows clean exit (preferred)
- `daemon=True` guarantees exit if clean exit fails
- Defense in depth

## 🔧 Alternative Implementations

### Option 1: Queue-Based Communication

```python
from queue import Queue

# Shared queues
speech_queue = Queue()

# STT Thread (producer)
def stt_worker():
    while not stop_flag:
        # Transcribe
        text = transcribe(audio)
        speech_queue.put(text)  # Thread-safe

# Main Thread (consumer)
while True:
    # Check queue
    try:
        text = speech_queue.get_nowait()
        current_subtitle = text
    except queue.Empty:
        pass
    
    # Process frame...
```

**Pros**: More explicit, better for complex communication
**Cons**: Overkill for our simple use case

### Option 2: Threading Locks

```python
import threading

subtitle_lock = threading.Lock()

# STT Thread
with subtitle_lock:
    current_subtitle = text

# Main Thread
with subtitle_lock:
    text = current_subtitle
```

**Pros**: Explicit thread safety
**Cons**: Unnecessary overhead, potential for deadlocks

### Option 3: Multiprocessing

```python
from multiprocessing import Process, Manager

manager = Manager()
shared_state = manager.dict()

# Separate processes
video_process = Process(target=video_worker, args=(shared_state,))
audio_process = Process(target=audio_worker, args=(shared_state,))
```

**Pros**: True parallelism (no GIL)
**Cons**: 
- Higher memory usage (separate Python interpreters)
- More complex state sharing
- Slower communication between processes

**Why We Don't Use It**:
- GIL not a bottleneck (I/O bound, not CPU bound)
- Threading simpler and sufficient
- Camera/audio libraries work better with threads

## 📊 Performance Impact

### Thread Overhead

**Context Switching**:
- Main thread: Runs continuously (30+ times per second)
- STT thread: Mostly blocked in I/O (4-second chunks)
- Context switches: ~10 per second (low)

**Memory Overhead**:
- Thread stack: ~8 MB per thread
- Total for 2 threads: ~16 MB
- Negligible compared to model memory (~200 MB)

**CPU Overhead**:
- Thread creation: ~1 ms (one-time)
- Context switching: ~1 μs per switch
- Total overhead: < 0.1% of CPU time

### Blocking Operations

**Main Thread**:
```python
ret, frame = cap.read()        # ~10 ms (I/O)
results = model(frame, ...)    # ~80 ms (CPU/GPU)
cv2.imshow(...)                # ~1 ms (I/O)
cv2.waitKey(1)                 # 1 ms (I/O)
```

**STT Thread**:
```python
sd.rec(...)                    # 0 ms (non-blocking)
sd.wait()                      # 4000 ms (I/O)
model_stt.transcribe(...)      # ~2000 ms (CPU)
```

**GIL Released During**:
- Camera I/O (`cv2.VideoCapture.read()`)
- Audio I/O (`sd.rec()`, `sd.wait()`)
- NumPy operations
- YOLO inference (PyTorch releases GIL)
- Whisper inference

**Result**: Threads run truly in parallel during I/O operations

## 🐛 Common Threading Issues (and How We Avoid Them)

### Issue 1: Race Conditions

**Problem**: Two threads modify same data simultaneously

**How We Avoid**:
- Single writer per variable
- No concurrent modifications
- GIL protection for simple operations

### Issue 2: Deadlocks

**Problem**: Threads wait for each other indefinitely

**How We Avoid**:
- No locks used!
- No circular dependencies
- Simple producer-consumer pattern

### Issue 3: Thread Not Exiting

**Problem**: Thread continues running after main exits

**How We Avoid**:
- `daemon=True` (automatic termination)
- `stop_flag` (graceful exit signal)
- Timeout on wait (0.5 seconds)

### Issue 4: Data Corruption

**Problem**: Partially written data read by other thread

**How We Avoid**:
- Atomic operations only (assign, append)
- Complete objects before sharing
- No mid-modification reads

## 📚 Further Reading

- **[01_MAIN_THREAD_OVERVIEW.md](01_MAIN_THREAD_OVERVIEW.md)** - Main thread architecture
- **[../02_ARCHITECTURE.md](../02_ARCHITECTURE.md)** - Overall system threading model
- **[../03_WORKFLOW.md](../03_WORKFLOW.md)** - Execution flow with threading

## 🔗 External Resources

- [Python Threading Documentation](https://docs.python.org/3/library/threading.html)
- [Python GIL Explained](https://realpython.com/python-gil/)
- [Thread Synchronization Primitives](https://docs.python.org/3/library/threading.html#synchronization-primitives)

---

← [Previous: UI Rendering](08_UI_RENDERING.md) | [Back to Main Thread Docs](00_README.md) | [Next: Logging and Output →](10_LOGGING_AND_OUTPUT.md)
