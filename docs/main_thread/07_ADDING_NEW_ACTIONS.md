# Adding New Motion/Posture Recognition

## 🎯 Overview

This guide provides a complete, step-by-step process for adding new action detection capabilities to the system. Whether you want to detect a specific gesture, posture, or motion pattern, this document will walk you through the entire process.

## 📋 Prerequisites

Before adding a new action, ensure you understand:
- The 17 COCO keypoints and their indices
- Basic geometry (distance calculation, angles)
- Python programming
- The `detect_custom_actions()` function structure

## 🔍 Step 1: Define the Action

### 1.1 Choose a Clear Name

Pick a descriptive, lowercase name with underscores:

✅ **Good Names**:
- `hand_on_hip`
- `arms_raised`
- `thumbs_up`
- `hand_wave`
- `head_tilt_left`

❌ **Bad Names**:
- `action_11` (not descriptive)
- `HandOnHip` (use snake_case)
- `gesture` (too vague)

### 1.2 Define Behavioral Meaning

Document what the action indicates:

**Example**: `hand_on_hip`
- **Behavioral Meaning**: Confidence, impatience, defiance, or casual stance
- **Cultural Context**: Common in many cultures, can indicate various emotions
- **Use Case**: Body language analysis in interviews, presentations

### 1.3 Identify Visual Characteristics

Describe how the action looks geometrically:

**Example**: `hand_on_hip`
- One or both hands positioned near hip area
- Wrist close to hip keypoint (< 60 pixels)
- Elbow pointing outward
- Usually indicates relaxed or confident posture

## 🎨 Step 2: Sketch the Geometry

### 2.1 Identify Relevant Keypoints

For `hand_on_hip`, we need:
```python
# Keypoints involved:
l_wrist = kp[9]   # Left wrist
r_wrist = kp[10]  # Right wrist
l_hip = kp[11]    # Left hip
r_hip = kp[12]    # Right hip
```

### 2.2 Draw a Diagram

```
    👤 Person (front view)
    
    Left Side         Right Side
    
    L-Shoulder ━━━━━ R-Shoulder
         │               │
         │               │
    L-Elbow         R-Elbow
      ↙                   ↘
  L-Wrist              R-Wrist
      ↓                    ↓
    L-Hip ━━━━━━━━━━━━━ R-Hip
                ↑
        ← Hand touches hip here
```

### 2.3 Define Geometric Relationships

For `hand_on_hip`:
1. **Distance**: Wrist to hip < threshold
2. **Side**: Check both left and right sides
3. **Threshold**: ~50-60 pixels (adjust based on testing)

## 💻 Step 3: Write Detection Code

### 3.1 Basic Template

```python
def detect_custom_actions(kp):
    """
    Detect custom actions from pose keypoints
    
    Parameters:
        kp: numpy array of shape (17, 2) with [x, y] coordinates
        
    Returns:
        list of detected action names
    """
    global prev_left_wrist, prev_right_wrist
    
    # Extract keypoints
    nose = kp[0]
    left_eye, right_eye = kp[1], kp[2]
    left_ear, right_ear = kp[3], kp[4]
    l_shoulder, r_shoulder = kp[5], kp[6]
    l_elbow, r_elbow = kp[7], kp[8]
    l_wrist, r_wrist = kp[9], kp[10]
    l_hip, r_hip = kp[11], kp[12]
    l_knee, r_knee = kp[13], kp[14]
    l_ankle, r_ankle = kp[15], kp[16]
    
    actions = []
    
    # Calculate centers (if needed)
    shoulder_center = (
        (l_shoulder[0] + r_shoulder[0]) / 2,
        (l_shoulder[1] + r_shoulder[1]) / 2
    )
    
    # --- EXISTING ACTIONS ---
    # (10 existing action checks here)
    # ...
    
    # --- ADD YOUR NEW ACTION HERE ---
    # 11. Hand on Hip
    if distance(l_wrist, l_hip) < 60 or distance(r_wrist, r_hip) < 60:
        actions.append("hand_on_hip")
    
    # Update previous positions (for temporal actions)
    prev_left_wrist = l_wrist
    prev_right_wrist = r_wrist
    
    return list(set(actions))  # Remove duplicates
```

### 3.2 Add Your Detection Logic

Location: Add after the existing 10 actions (around line 120 in the function)

```python
    # -----------------------------
    # 11. Hand on Hip
    # -----------------------------
    if distance(l_wrist, l_hip) < 60 or distance(r_wrist, r_hip) < 60:
        actions.append("hand_on_hip")
```

### 3.3 More Complex Example: Arms Raised

```python
    # -----------------------------
    # 11. Arms Raised
    # -----------------------------
    # Both wrists above shoulders
    if (l_wrist[1] < l_shoulder[1] - 50 and 
        r_wrist[1] < r_shoulder[1] - 50):
        actions.append("arms_raised")
```

**Note**: Y increases downward, so `<` means "above"

### 3.4 Temporal Action Example: Hand Wave

```python
# At the top of the function, add global state
global prev_left_wrist, prev_right_wrist, wave_counter_left, wave_counter_right

# Initialize counters (in global scope before function)
wave_counter_left = 0
wave_counter_right = 0

# In the function:
    # -----------------------------
    # 11. Hand Wave
    # -----------------------------
    # Detect side-to-side hand movement
    if prev_left_wrist is not None:
        # Horizontal movement
        horizontal_move = abs(l_wrist[0] - prev_left_wrist[0])
        # Vertical stability
        vertical_move = abs(l_wrist[1] - prev_left_wrist[1])
        
        if horizontal_move > 20 and vertical_move < 10:
            wave_counter_left += 1
            if wave_counter_left > 3:  # 3 consecutive frames
                actions.append("hand_wave")
        else:
            wave_counter_left = max(0, wave_counter_left - 1)
```

## 🔬 Step 4: Test Your Action

### 4.1 Create Test Script

```python
# test_new_action.py
import cv2
import numpy as np
from ultralytics import YOLO

model = YOLO("yolo11m-pose.pt")

def distance(p1, p2):
    return np.linalg.norm(np.array(p1) - np.array(p2))

def detect_custom_actions(kp):
    # ... (paste your updated function)
    pass

# Test with webcam
cap = cv2.VideoCapture(0)

while True:
    ret, frame = cap.read()
    if not ret:
        break
    
    results = model(frame, device="cpu", verbose=False)
    
    for r in results:
        if r.keypoints is None:
            continue
        
        for person in r.keypoints.xy:
            kp = person.cpu().numpy()
            actions = detect_custom_actions(kp)
            
            # Print for debugging
            print(f"Detected: {actions}")
            
            # Draw on frame
            y = 30
            for act in actions:
                cv2.putText(frame, f"ACTION: {act}", (10, y),
                           cv2.FONT_HERSHEY_SIMPLEX, 0.8, (0, 255, 0), 2)
                y += 30
    
    cv2.imshow("Test New Action", frame)
    if cv2.waitKey(1) & 0xFF == ord('q'):
        break

cap.release()
cv2.destroyAllWindows()
```

### 4.2 Test Different Scenarios

Test your new action with:
1. **Correct pose**: Perform the action correctly
2. **Near-miss poses**: Similar but not quite the action
3. **Different angles**: Front view, slight side view
4. **Different distances**: Close to camera, far from camera
5. **Occlusion**: Partially hidden by objects
6. **Multiple people**: Ensure it works with multiple people

### 4.3 Debug Distances

Add debug output to understand actual distances:

```python
def detect_custom_actions(kp, debug=False):
    # ... extract keypoints ...
    
    # Debug distances
    if debug:
        print("\n=== Hand on Hip Debug ===")
        print(f"L-Wrist to L-Hip: {distance(l_wrist, l_hip):.1f} pixels")
        print(f"R-Wrist to R-Hip: {distance(r_wrist, r_hip):.1f} pixels")
        print(f"Threshold: 60 pixels")
    
    # Detection logic
    if distance(l_wrist, l_hip) < 60 or distance(r_wrist, r_hip) < 60:
        if debug:
            print("✓ DETECTED: hand_on_hip")
        actions.append("hand_on_hip")
```

## 🎛️ Step 5: Tune Thresholds

### 5.1 Collect Data

Perform the action multiple times and record distances:

```python
distances = []

# In your test loop
if distance(l_wrist, l_hip) < 100:  # Wide threshold for data collection
    dist = distance(l_wrist, l_hip)
    distances.append(dist)
    print(f"Distance: {dist:.1f}")

# After testing
print(f"\nStatistics:")
print(f"Mean: {np.mean(distances):.1f}")
print(f"Std: {np.std(distances):.1f}")
print(f"Min: {np.min(distances):.1f}")
print(f"Max: {np.max(distances):.1f}")
print(f"Suggested threshold: {np.percentile(distances, 80):.1f}")
```

### 5.2 Choose Threshold

Use the **80th percentile** as your threshold:
- 80% of correct poses will be detected
- Reduces false positives
- Balance between precision and recall

### 5.3 Test False Positives

Test poses that should NOT trigger the action:
- **For hand_on_hip**: Hand near hip but not touching
- Record false positive rate
- Adjust threshold if needed

## 📝 Step 6: Document Your Action

### 6.1 Add to Code Comments

```python
    # -----------------------------
    # 11. Hand on Hip
    # -----------------------------
    # Detects when one or both hands are placed on hips
    # Indicates: Confidence, impatience, or casual stance
    # Threshold: 60 pixels (wrist to hip distance)
    if distance(l_wrist, l_hip) < 60 or distance(r_wrist, r_hip) < 60:
        actions.append("hand_on_hip")
```

### 6.2 Update Action List

Document in the main README or documentation:

```markdown
## Detected Actions (11)

1. **arms_crossed** - Defensive posture
2. **hands_clasped** - Formal posture
3. **chin_rest** - Thinking
4. **lean_forward** - Interest
5. **lean_back** - Relaxation
6. **head_down** - Submission
7. **touch_face** - Self-soothing
8. **touch_nose** - Specific touch
9. **fix_hair** - Grooming
10. **fidget_hands** - Nervousness
11. **hand_on_hip** - Confidence (NEW)
```

## 🎨 Advanced Examples

### Example 1: Angle-Based Detection (Arm Angle)

```python
def calculate_angle(p1, p2, p3):
    """Calculate angle at point p2 formed by p1-p2-p3"""
    v1 = np.array(p1) - np.array(p2)
    v2 = np.array(p3) - np.array(p2)
    
    cos_angle = np.dot(v1, v2) / (np.linalg.norm(v1) * np.linalg.norm(v2))
    angle = np.arccos(np.clip(cos_angle, -1.0, 1.0))
    return np.degrees(angle)

# In detect_custom_actions():
    # -----------------------------
    # 11. Arms Crossed (angle-based)
    # -----------------------------
    left_arm_angle = calculate_angle(l_shoulder, l_elbow, l_wrist)
    right_arm_angle = calculate_angle(r_shoulder, r_elbow, r_wrist)
    
    # Arms bent at ~90 degrees and crossed
    if (80 < left_arm_angle < 100 and 80 < right_arm_angle < 100 and
        distance(l_wrist, r_elbow) < 80 and distance(r_wrist, l_elbow) < 80):
        actions.append("arms_crossed")
```

### Example 2: Multi-Keypoint Pattern (Sitting)

```python
    # -----------------------------
    # 11. Sitting Posture
    # -----------------------------
    # Detect when person is sitting
    # Indicators:
    # - Hips and knees at similar height
    # - Knees bent at ~90 degrees
    # - Ankles below knees
    
    hip_center_y = (l_hip[1] + r_hip[1]) / 2
    knee_center_y = (l_knee[1] + r_knee[1]) / 2
    
    # Hips and knees relatively close (sitting)
    if abs(hip_center_y - knee_center_y) < 100:
        # Ankles below knees
        if l_ankle[1] > l_knee[1] and r_ankle[1] > r_knee[1]:
            actions.append("sitting")
```

### Example 3: Motion Pattern (Clapping)

```python
# Global state
clap_state = {"last_distance": None, "clap_count": 0, "last_clap_time": 0}

    # -----------------------------
    # 11. Clapping
    # -----------------------------
    import time
    
    current_distance = distance(l_wrist, r_wrist)
    current_time = time.time()
    
    if clap_state["last_distance"] is not None:
        # Hands were apart, now close (clap motion)
        if clap_state["last_distance"] > 100 and current_distance < 50:
            # Not too soon after last clap
            if current_time - clap_state["last_clap_time"] > 0.3:
                clap_state["clap_count"] += 1
                clap_state["last_clap_time"] = current_time
                
                if clap_state["clap_count"] >= 2:  # At least 2 claps
                    actions.append("clapping")
    
    clap_state["last_distance"] = current_distance
    
    # Reset if hands stay apart for too long
    if current_distance > 150:
        clap_state["clap_count"] = 0
```

### Example 4: Asymmetric Gesture (Pointing)

```python
    # -----------------------------
    # 11. Pointing
    # -----------------------------
    # Detect when one arm is extended forward
    
    # Check left arm
    left_arm_extended = (
        distance(l_shoulder, l_wrist) > 250 and  # Arm extended
        l_wrist[0] > l_shoulder[0] + 100  # Forward direction
    )
    
    # Check right arm
    right_arm_extended = (
        distance(r_shoulder, r_wrist) > 250 and
        r_wrist[0] < r_shoulder[0] - 100  # Forward (right side)
    )
    
    if left_arm_extended or right_arm_extended:
        actions.append("pointing")
```

## ⚠️ Common Pitfalls

### 1. Coordinate System Confusion

❌ **Wrong**: Thinking Y increases upward
```python
if wrist[1] > shoulder[1]:  # Means wrist is BELOW shoulder
    actions.append("arms_raised")  # WRONG!
```

✅ **Correct**: Y increases downward
```python
if wrist[1] < shoulder[1]:  # Means wrist is ABOVE shoulder
    actions.append("arms_raised")  # CORRECT
```

### 2. Forgetting to Handle Missing Keypoints

❌ **Wrong**: Assuming keypoints are always detected
```python
if distance(l_wrist, nose) < 50:
    actions.append("touch_face")
```

✅ **Better**: Check for valid coordinates
```python
if (l_wrist[0] > 0 and l_wrist[1] > 0 and 
    nose[0] > 0 and nose[1] > 0):
    if distance(l_wrist, nose) < 50:
        actions.append("touch_face")
```

### 3. Not Accounting for Camera Distance

**Problem**: Thresholds work up close but not when far from camera

**Solution**: Normalize by person size
```python
# Calculate person height (shoulder to ankle)
person_height = distance(shoulder_center, 
                        ((l_ankle[0] + r_ankle[0])/2, 
                         (l_ankle[1] + r_ankle[1])/2))

# Normalize threshold
normalized_threshold = 60 * (person_height / 400)  # 400 is reference height

if distance(l_wrist, l_hip) < normalized_threshold:
    actions.append("hand_on_hip")
```

### 4. Temporal Logic Without State Management

❌ **Wrong**: Checking only current frame
```python
if distance(l_wrist, r_wrist) > 25:
    actions.append("hands_moving")  # Triggers every frame!
```

✅ **Correct**: Track state across frames
```python
global prev_wrist_dist, stable_frames

current_dist = distance(l_wrist, r_wrist)
if prev_wrist_dist is not None:
    if abs(current_dist - prev_wrist_dist) > 25:
        stable_frames = 0
    else:
        stable_frames += 1
    
    if stable_frames > 10:  # Stable for 10 frames
        actions.append("hands_still")

prev_wrist_dist = current_dist
```

## ✅ Validation Checklist

Before considering your action complete:

- [ ] Action name is descriptive and follows naming convention
- [ ] Behavioral meaning is documented
- [ ] Code is added to `detect_custom_actions()` function
- [ ] Threshold is tuned based on test data
- [ ] False positive rate is acceptable (< 10%)
- [ ] False negative rate is acceptable (< 20%)
- [ ] Works at different camera distances (1-3 meters)
- [ ] Works with different lighting conditions
- [ ] Works from front and slight side views
- [ ] Handles multiple people correctly
- [ ] Debug output has been added (optional but recommended)
- [ ] Documentation is updated
- [ ] Code comments are clear

## 🚀 Deployment

### Update Both Files

If your action is general-purpose, add it to both:
1. `interview_system.py` (full system)
2. `action_detector3.py` (action-only version)

### Testing in Production

```python
# Add logging for first week
if "your_new_action" in actions:
    import logging
    logging.info(f"New action detected at {time.time()}")
```

### Monitor Performance

```python
# Check if new action slows down system
start = time.time()
actions = detect_custom_actions(kp)
elapsed = time.time() - start

if elapsed > 0.005:  # 5ms threshold
    print(f"Warning: Action detection took {elapsed*1000:.1f}ms")
```

## 📚 Further Reading

- **[05_ACTION_DETECTION_ENGINE.md](05_ACTION_DETECTION_ENGINE.md)** - Action detection architecture
- **[06_CONFIGURING_DETECTIONS.md](06_CONFIGURING_DETECTIONS.md)** - Tuning parameters
- **[../06_ACTION_DETECTION.md](../06_ACTION_DETECTION.md)** - All existing actions explained

---

← [Previous: Configuring Detections](06_CONFIGURING_DETECTIONS.md) | [Back to Main Thread Docs](00_README.md) | [Next: UI Rendering →](08_UI_RENDERING.md)
