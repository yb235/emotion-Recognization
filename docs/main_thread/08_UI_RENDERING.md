# UI Rendering and Display

## 🎯 Overview

The UI rendering component draws detected actions and speech subtitles on video frames and displays them to the user. This document explains the rendering process, customization options, and performance considerations.

## 🎨 Rendering Pipeline

```
Frame from Camera
      ↓
┌─────────────────────┐
│ Draw Action Labels  │ ← cv2.putText()
└─────────────────────┘
      ↓
┌─────────────────────┐
│ Draw Subtitles      │ ← cv2.putText()
└─────────────────────┘
      ↓
┌─────────────────────┐
│ Draw Bounding Boxes │ ← cv2.rectangle() (optional)
└─────────────────────┘
      ↓
┌─────────────────────┐
│ Draw Keypoints      │ ← cv2.circle() (optional)
└─────────────────────┘
      ↓
┌─────────────────────┐
│ Display Window      │ ← cv2.imshow()
└─────────────────────┘
```

## 📝 Drawing Action Labels

### Basic Implementation

From `interview_system.py`:

```python
# Draw actions on frame (top of frame)
y = 30
for act in set(frame_actions):
    cv2.putText(
        frame,                          # Image to draw on
        f"ACTION: {act}",               # Text to display
        (10, y),                        # Position (x, y)
        cv2.FONT_HERSHEY_SIMPLEX,      # Font
        0.8,                            # Font scale
        (0, 255, 0),                   # Color (B, G, R) - Green
        2                               # Thickness
    )
    y += 30  # Move down for next action
```

### Rendering Options

#### Font Choices

```python
# Available OpenCV fonts
cv2.FONT_HERSHEY_SIMPLEX       # Simple sans-serif (default)
cv2.FONT_HERSHEY_PLAIN         # Small simple font
cv2.FONT_HERSHEY_DUPLEX        # Sans-serif with more complex lines
cv2.FONT_HERSHEY_COMPLEX       # Serif font
cv2.FONT_HERSHEY_TRIPLEX       # More complex serif
cv2.FONT_HERSHEY_SCRIPT_SIMPLEX  # Handwriting style
cv2.FONT_HERSHEY_SCRIPT_COMPLEX  # More complex handwriting
```

#### Text Properties

```python
cv2.putText(
    frame,
    text="ACTION: arms_crossed",
    org=(10, 30),                    # Bottom-left corner of text
    fontFace=cv2.FONT_HERSHEY_SIMPLEX,
    fontScale=0.8,                   # Scale: 0.5 (small) to 2.0 (large)
    color=(0, 255, 0),               # BGR format
    thickness=2,                     # Pixel thickness
    lineType=cv2.LINE_AA             # Anti-aliased (smoother)
)
```

### Improved Action Display

```python
def draw_actions_enhanced(frame, actions):
    """Enhanced action display with background"""
    y_offset = 30
    
    for act in actions:
        # Prepare text
        text = f"ACTION: {act.replace('_', ' ').upper()}"
        
        # Get text size for background
        (text_width, text_height), baseline = cv2.getTextSize(
            text,
            cv2.FONT_HERSHEY_SIMPLEX,
            0.8,
            2
        )
        
        # Draw background rectangle
        cv2.rectangle(
            frame,
            (5, y_offset - text_height - 5),
            (15 + text_width, y_offset + baseline + 5),
            (0, 0, 0),        # Black background
            -1                # Filled
        )
        
        # Draw semi-transparent overlay
        overlay = frame.copy()
        cv2.rectangle(
            overlay,
            (5, y_offset - text_height - 5),
            (15 + text_width, y_offset + baseline + 5),
            (0, 100, 0),      # Dark green
            -1
        )
        cv2.addWeighted(overlay, 0.3, frame, 0.7, 0, frame)
        
        # Draw text
        cv2.putText(
            frame,
            text,
            (10, y_offset),
            cv2.FONT_HERSHEY_SIMPLEX,
            0.8,
            (0, 255, 0),      # Bright green
            2,
            cv2.LINE_AA
        )
        
        y_offset += 35
    
    return frame
```

## 💬 Drawing Subtitles

### Basic Implementation

From `interview_system.py`:

```python
# Draw current subtitle (bottom of frame)
if current_subtitle:
    h, w, _ = frame.shape
    cv2.putText(
        frame,
        current_subtitle,
        (10, h - 20),                   # Bottom-left corner
        cv2.FONT_HERSHEY_SIMPLEX,
        0.7,
        (0, 255, 255),                  # Yellow (B=0, G=255, R=255)
        2
    )
```

### Word Wrapping

```python
def draw_subtitle_wrapped(frame, text, max_width=60):
    """Draw subtitle with word wrapping"""
    import textwrap
    
    # Wrap text to max width
    wrapped = textwrap.wrap(text, width=max_width)
    
    h, w, _ = frame.shape
    y_start = h - 20 - (len(wrapped) * 30)  # Start higher for multiple lines
    
    for i, line in enumerate(wrapped):
        y = y_start + (i * 30)
        
        # Background for readability
        (text_width, text_height), _ = cv2.getTextSize(
            line,
            cv2.FONT_HERSHEY_SIMPLEX,
            0.7,
            2
        )
        
        cv2.rectangle(
            frame,
            (5, y - text_height - 5),
            (15 + text_width, y + 5),
            (0, 0, 0),
            -1
        )
        
        # Draw text
        cv2.putText(
            frame,
            line,
            (10, y),
            cv2.FONT_HERSHEY_SIMPLEX,
            0.7,
            (0, 255, 255),
            2,
            cv2.LINE_AA
        )
    
    return frame
```

### Subtitle Fading Effect

```python
import time

subtitle_data = {"text": "", "timestamp": 0}

def draw_fading_subtitle(frame, new_text=None, fade_duration=3.0):
    """Draw subtitle with fade-out effect"""
    global subtitle_data
    
    if new_text:
        subtitle_data["text"] = new_text
        subtitle_data["timestamp"] = time.time()
    
    if not subtitle_data["text"]:
        return frame
    
    # Calculate alpha based on time since display
    elapsed = time.time() - subtitle_data["timestamp"]
    
    if elapsed > fade_duration:
        subtitle_data["text"] = ""
        return frame
    
    # Fade alpha from 1.0 to 0.0
    alpha = 1.0 - (elapsed / fade_duration)
    
    # Draw with alpha
    overlay = frame.copy()
    h, w, _ = overlay.shape
    
    cv2.putText(
        overlay,
        subtitle_data["text"],
        (10, h - 20),
        cv2.FONT_HERSHEY_SIMPLEX,
        0.7,
        (0, 255, 255),
        2,
        cv2.LINE_AA
    )
    
    cv2.addWeighted(overlay, alpha, frame, 1 - alpha, 0, frame)
    
    return frame
```

## 📦 Optional Visualizations

### Draw Bounding Boxes

```python
def draw_bounding_boxes(frame, results):
    """Draw person detection boxes"""
    for r in results:
        if r.boxes is None:
            continue
        
        boxes = r.boxes.xyxy.cpu().numpy()
        confidences = r.boxes.conf.cpu().numpy()
        
        for box, conf in zip(boxes, confidences):
            x1, y1, x2, y2 = map(int, box)
            
            # Draw rectangle
            cv2.rectangle(
                frame,
                (x1, y1),
                (x2, y2),
                (255, 0, 0),  # Blue
                2
            )
            
            # Draw confidence
            cv2.putText(
                frame,
                f"{conf:.2f}",
                (x1, y1 - 10),
                cv2.FONT_HERSHEY_SIMPLEX,
                0.5,
                (255, 0, 0),
                2
            )
    
    return frame
```

### Draw Keypoints

```python
def draw_keypoints(frame, results):
    """Draw pose keypoints"""
    for r in results:
        if r.keypoints is None:
            continue
        
        for person in r.keypoints.xy:
            kp = person.cpu().numpy()
            
            for i, (x, y) in enumerate(kp):
                if x > 0 and y > 0:
                    cv2.circle(
                        frame,
                        (int(x), int(y)),
                        5,
                        (0, 0, 255),  # Red
                        -1
                    )
    
    return frame
```

### Draw Skeleton

```python
SKELETON_CONNECTIONS = [
    (5, 6),   # shoulders
    (5, 7), (7, 9),   # left arm
    (6, 8), (8, 10),  # right arm
    (5, 11), (6, 12),  # torso
    (11, 12),  # hips
    (11, 13), (13, 15),  # left leg
    (12, 14), (14, 16),  # right leg
]

def draw_skeleton(frame, results):
    """Draw skeleton connections"""
    for r in results:
        if r.keypoints is None:
            continue
        
        for person in r.keypoints.xy:
            kp = person.cpu().numpy()
            
            for i, j in SKELETON_CONNECTIONS:
                pt1 = tuple(map(int, kp[i]))
                pt2 = tuple(map(int, kp[j]))
                
                if pt1[0] > 0 and pt2[0] > 0:
                    cv2.line(
                        frame,
                        pt1,
                        pt2,
                        (0, 255, 0),
                        2
                    )
    
    return frame
```

## 🖼️ Complete Rendering Function

```python
def render_complete_ui(frame, actions, subtitle, results=None, 
                      show_boxes=False, show_keypoints=False):
    """Complete UI rendering with all options"""
    
    # 1. Draw bounding boxes (if enabled)
    if show_boxes and results:
        frame = draw_bounding_boxes(frame, results)
    
    # 2. Draw keypoints and skeleton (if enabled)
    if show_keypoints and results:
        frame = draw_skeleton(frame, results)
        frame = draw_keypoints(frame, results)
    
    # 3. Draw action labels
    frame = draw_actions_enhanced(frame, actions)
    
    # 4. Draw subtitle
    if subtitle:
        frame = draw_subtitle_wrapped(frame, subtitle)
    
    # 5. Add timestamp
    timestamp = time.strftime("%Y-%m-%d %H:%M:%S")
    cv2.putText(
        frame,
        timestamp,
        (frame.shape[1] - 220, 25),
        cv2.FONT_HERSHEY_SIMPLEX,
        0.5,
        (255, 255, 255),
        1
    )
    
    # 6. Add FPS counter
    if hasattr(render_complete_ui, 'fps'):
        cv2.putText(
            frame,
            f"FPS: {render_complete_ui.fps:.1f}",
            (frame.shape[1] - 100, 50),
            cv2.FONT_HERSHEY_SIMPLEX,
            0.5,
            (255, 255, 255),
            1
        )
    
    return frame
```

## 🪟 Window Display

### Basic Display

```python
cv2.imshow("Interview Action + Speech Monitor", frame)

# Wait 1ms and check for key press
if cv2.waitKey(1) & 0xFF == ord('q'):
    break
```

### Window Configuration

```python
# Create named window
cv2.namedWindow("Interview Monitor", cv2.WINDOW_NORMAL)

# Set window size
cv2.resizeWindow("Interview Monitor", 1280, 720)

# Make window resizable
cv2.namedWindow("Interview Monitor", cv2.WINDOW_AUTOSIZE)

# Fullscreen
cv2.namedWindow("Interview Monitor", cv2.WINDOW_NORMAL)
cv2.setWindowProperty("Interview Monitor", cv2.WND_PROP_FULLSCREEN, 
                     cv2.WINDOW_FULLSCREEN)

# Always on top (platform dependent)
cv2.setWindowProperty("Interview Monitor", cv2.WND_PROP_TOPMOST, 1)
```

## ⚡ Performance Optimization

### Reduce Drawing Operations

```python
# Only redraw when actions change
prev_actions = set()

while True:
    ret, frame = cap.read()
    results = model(frame)
    current_actions = set(detect_actions())
    
    # Only redraw if actions changed
    if current_actions != prev_actions:
        frame = draw_actions_enhanced(frame, current_actions)
        prev_actions = current_actions
    
    cv2.imshow("Window", frame)
```

### Use Lookup Tables for Colors

```python
# Pre-compute colors
ACTION_COLORS = {
    "arms_crossed": (0, 0, 255),      # Red
    "hands_clasped": (0, 255, 0),     # Green
    "lean_forward": (255, 0, 0),      # Blue
    "lean_back": (255, 255, 0),       # Cyan
    # ... etc
}

# Use in drawing
color = ACTION_COLORS.get(action, (255, 255, 255))
cv2.putText(frame, action, (10, 30), font, 0.8, color, 2)
```

### Measure Rendering Time

```python
import time

start = time.time()
frame = render_complete_ui(frame, actions, subtitle)
render_time = (time.time() - start) * 1000

print(f"Rendering: {render_time:.1f}ms")
```

## 🎨 Customization Examples

### Dark Theme

```python
def apply_dark_theme(frame):
    """Apply dark overlay for better text visibility"""
    overlay = np.zeros_like(frame)
    cv2.addWeighted(frame, 0.7, overlay, 0.3, 0, frame)
    return frame
```

### Color-Coded Actions

```python
ACTION_SEVERITY = {
    "arms_crossed": "warning",
    "fidget_hands": "warning",
    "lean_back": "neutral",
    "lean_forward": "positive",
    # ...
}

SEVERITY_COLORS = {
    "positive": (0, 255, 0),   # Green
    "neutral": (255, 255, 0),  # Yellow
    "warning": (0, 165, 255),  # Orange
    "negative": (0, 0, 255),   # Red
}

def draw_colored_actions(frame, actions):
    y = 30
    for act in actions:
        severity = ACTION_SEVERITY.get(act, "neutral")
        color = SEVERITY_COLORS[severity]
        
        cv2.putText(frame, f"{act}", (10, y), 
                   cv2.FONT_HERSHEY_SIMPLEX, 0.8, color, 2)
        y += 30
    
    return frame
```

## 📚 Further Reading

- **[01_MAIN_THREAD_OVERVIEW.md](01_MAIN_THREAD_OVERVIEW.md)** - Main thread architecture
- **[05_ACTION_DETECTION_ENGINE.md](05_ACTION_DETECTION_ENGINE.md)** - What to display
- **[09_THREADING_AND_SYNC.md](09_THREADING_AND_SYNC.md)** - Subtitle synchronization

---

← [Previous: Adding New Actions](07_ADDING_NEW_ACTIONS.md) | [Back to Main Thread Docs](00_README.md) | [Next: Threading and Sync →](09_THREADING_AND_SYNC.md)
