# Keypoint Extraction from YOLO Results

## 🎯 Overview

After YOLO inference, we need to extract the 17 pose keypoints from the results object. This document explains the extraction process, data formats, coordinate systems, and how to handle multiple people.

## 📦 YOLO Results Structure

### Results Object

```python
results = model(frame, device="cpu", verbose=False)

# Results is a list (one element per frame, or per batch)
# results[0] corresponds to our single frame

for r in results:
    # r is an ultralytics.engine.results.Results object
    print(type(r))  # <class 'ultralytics.engine.results.Results'>
```

### Available Attributes

```python
for r in results:
    # Bounding boxes (person detection boxes)
    r.boxes          # Detection boxes
    r.boxes.xyxy     # Box coordinates [x1, y1, x2, y2]
    r.boxes.conf     # Confidence scores
    
    # Keypoints (pose estimation)
    r.keypoints      # Pose keypoints (what we need!)
    r.keypoints.xy   # Keypoint (x, y) coordinates
    r.keypoints.xyn  # Normalized coordinates [0-1]
    r.keypoints.conf # Keypoint confidence scores
    
    # Metadata
    r.orig_shape     # Original image shape
    r.path           # Image path (if from file)
```

## 🔑 Extracting Keypoints

### Basic Extraction

```python
results = model(frame, device="cpu", verbose=False)

for r in results:
    # Check if keypoints detected
    if r.keypoints is None:
        print("No people detected")
        continue
    
    # Iterate through each detected person
    for person in r.keypoints.xy:
        # person is a tensor, convert to numpy
        kp = person.cpu().numpy()  # Shape: (17, 2)
        
        # kp now contains 17 keypoints
        # kp[0] = [x, y] for nose
        # kp[1] = [x, y] for left_eye
        # ... and so on
```

### Understanding the Shape

```python
# r.keypoints.xy shape
print(r.keypoints.xy.shape)  # torch.Size([N, 17, 2])
# N = number of people detected
# 17 = number of keypoints per person
# 2 = (x, y) coordinates

# Example: 2 people detected
# Shape: (2, 17, 2)
# Person 0: r.keypoints.xy[0] → (17, 2)
# Person 1: r.keypoints.xy[1] → (17, 2)
```

### Complete Extraction Pattern

```python
def extract_all_keypoints(results):
    """Extract keypoints from all detected people"""
    all_people_keypoints = []
    
    for r in results:
        if r.keypoints is None:
            continue
        
        for person in r.keypoints.xy:
            kp = person.cpu().numpy()  # Convert to NumPy
            all_people_keypoints.append(kp)
    
    return all_people_keypoints

# Usage
results = model(frame, device="cpu", verbose=False)
keypoints_list = extract_all_keypoints(results)

print(f"Detected {len(keypoints_list)} people")

for i, kp in enumerate(keypoints_list):
    print(f"Person {i}: {kp.shape}")  # (17, 2)
```

## 📍 Keypoint Index Reference

### COCO Keypoint Layout

```python
KEYPOINT_NAMES = [
    "nose",           # 0
    "left_eye",       # 1
    "right_eye",      # 2
    "left_ear",       # 3
    "right_ear",      # 4
    "left_shoulder",  # 5
    "right_shoulder", # 6
    "left_elbow",     # 7
    "right_elbow",    # 8
    "left_wrist",     # 9
    "right_wrist",    # 10
    "left_hip",       # 11
    "right_hip",      # 12
    "left_knee",      # 13
    "right_knee",     # 14
    "left_ankle",     # 15
    "right_ankle",    # 16
]

# Access by index
nose = kp[0]           # [x, y]
left_eye = kp[1]       # [x, y]
right_wrist = kp[10]   # [x, y]
```

### Extracting Specific Keypoints

```python
# Method 1: Direct indexing
nose = kp[0]
left_eye, right_eye = kp[1], kp[2]
l_shoulder, r_shoulder = kp[5], kp[6]
l_wrist, r_wrist = kp[9], kp[10]

# Method 2: Unpacking (as in detect_custom_actions)
nose = kp[0]
left_eye, right_eye = kp[1], kp[2]
left_ear, right_ear = kp[3], kp[4]
l_shoulder, r_shoulder = kp[5], kp[6]
l_elbow, r_elbow = kp[7], kp[8]
l_wrist, r_wrist = kp[9], kp[10]
l_hip, r_hip = kp[11], kp[12]
l_knee, r_knee = kp[13], kp[14]
l_ankle, r_ankle = kp[15], kp[16]

# Method 3: Dictionary
keypoint_dict = {
    name: kp[i] for i, name in enumerate(KEYPOINT_NAMES)
}
nose = keypoint_dict["nose"]
left_wrist = keypoint_dict["left_wrist"]
```

## 📏 Coordinate System

### Image Coordinates

```python
# Example keypoint: [320, 240]
x, y = kp[0]  # nose position

# Coordinate system:
# Origin (0, 0) is at TOP-LEFT of image
# X increases to the RIGHT
# Y increases DOWNWARD

# For 1280×720 image:
# Top-left corner: (0, 0)
# Top-right corner: (1280, 0)
# Bottom-left corner: (0, 720)
# Bottom-right corner: (1280, 720)
```

### Visual Representation

```
(0,0) ━━━━━━━━━━━━━━━━━━━━━━ (1280,0)
  ┃                              ┃
  ┃        👤 Person             ┃
  ┃       (640, 360)             ┃
  ┃         nose                 ┃
  ┃                              ┃
(0,720) ━━━━━━━━━━━━━━━━━━━━━ (1280,720)
```

### Important Y-Axis Note

```python
# Common mistake: assuming Y increases upward
if wrist[1] > shoulder[1]:
    # This means wrist is BELOW shoulder, not above!
    pass

if nose[1] < shoulder[1]:
    # This means nose is ABOVE shoulder
    pass
```

## 🎯 Keypoint Confidence Scores

### Extracting Confidence

```python
for r in results:
    if r.keypoints is None:
        continue
    
    for person_idx, person in enumerate(r.keypoints.xy):
        kp = person.cpu().numpy()  # (17, 2)
        
        # Get confidence scores
        conf = r.keypoints.conf[person_idx].cpu().numpy()  # (17,)
        
        # conf[i] is confidence for keypoint i
        # Range: [0.0, 1.0]
        # Higher = more confident detection
        
        for i, name in enumerate(KEYPOINT_NAMES):
            x, y = kp[i]
            confidence = conf[i]
            print(f"{name}: ({x:.0f}, {y:.0f}) conf={confidence:.2f}")
```

### Filtering Low-Confidence Keypoints

```python
CONFIDENCE_THRESHOLD = 0.5

def filter_keypoints(kp, conf, threshold=CONFIDENCE_THRESHOLD):
    """Replace low-confidence keypoints with invalid marker"""
    filtered_kp = kp.copy()
    
    for i in range(len(kp)):
        if conf[i] < threshold:
            filtered_kp[i] = [-1, -1]  # Invalid marker
    
    return filtered_kp

# Usage
kp_filtered = filter_keypoints(kp, conf)

# Check validity before using
if kp_filtered[0][0] >= 0:  # Nose is valid
    nose = kp_filtered[0]
else:
    print("Nose not detected with confidence")
```

## 👥 Handling Multiple People

### Detecting Multiple People

```python
results = model(frame, device="cpu", verbose=False)

for r in results:
    if r.keypoints is None:
        continue
    
    num_people = len(r.keypoints.xy)
    print(f"Detected {num_people} people")
    
    for person_idx, person in enumerate(r.keypoints.xy):
        kp = person.cpu().numpy()
        
        # Process this person's keypoints
        actions = detect_custom_actions(kp)
        print(f"Person {person_idx}: {actions}")
```

### Person Tracking (Advanced)

```python
import numpy as np

previous_people = []

def match_people(current_keypoints, previous_keypoints):
    """Simple person tracking by distance"""
    if not previous_keypoints:
        return list(range(len(current_keypoints)))
    
    assignments = []
    
    for curr_kp in current_keypoints:
        # Calculate distance to each previous person
        distances = []
        for prev_kp in previous_keypoints:
            # Use center of mass for matching
            curr_center = np.mean(curr_kp, axis=0)
            prev_center = np.mean(prev_kp, axis=0)
            dist = np.linalg.norm(curr_center - prev_center)
            distances.append(dist)
        
        # Assign to closest previous person
        if distances:
            assignments.append(np.argmin(distances))
        else:
            assignments.append(-1)  # New person
    
    return assignments

# Usage
current_keypoints = extract_all_keypoints(results)
person_ids = match_people(current_keypoints, previous_people)
previous_people = current_keypoints

for person_id, kp in zip(person_ids, current_keypoints):
    actions = detect_custom_actions(kp)
    print(f"Person {person_id}: {actions}")
```

## 🛠️ Utility Functions

### Visualize Keypoints

```python
def draw_keypoints(frame, kp, color=(0, 255, 0), radius=5):
    """Draw keypoints on frame"""
    for i, (x, y) in enumerate(kp):
        if x >= 0 and y >= 0:  # Valid keypoint
            cv2.circle(frame, (int(x), int(y)), radius, color, -1)
            
            # Optional: draw keypoint index
            cv2.putText(frame, str(i), (int(x), int(y) - 10),
                       cv2.FONT_HERSHEY_SIMPLEX, 0.4, (255, 255, 255), 1)
    
    return frame

# Usage
ret, frame = cap.read()
results = model(frame, device="cpu", verbose=False)

for r in results:
    if r.keypoints is not None:
        for person in r.keypoints.xy:
            kp = person.cpu().numpy()
            frame = draw_keypoints(frame, kp)

cv2.imshow("Keypoints", frame)
```

### Draw Skeleton

```python
# Skeleton connections (which keypoints connect)
SKELETON = [
    (0, 1), (0, 2),      # nose to eyes
    (1, 3), (2, 4),      # eyes to ears
    (0, 5), (0, 6),      # nose to shoulders
    (5, 6),              # shoulders
    (5, 7), (7, 9),      # left arm
    (6, 8), (8, 10),     # right arm
    (5, 11), (6, 12),    # torso
    (11, 12),            # hips
    (11, 13), (13, 15),  # left leg
    (12, 14), (14, 16),  # right leg
]

def draw_skeleton(frame, kp, color=(0, 255, 0), thickness=2):
    """Draw skeleton connections"""
    for i, j in SKELETON:
        pt1 = tuple(map(int, kp[i]))
        pt2 = tuple(map(int, kp[j]))
        
        # Only draw if both keypoints valid
        if pt1[0] >= 0 and pt2[0] >= 0:
            cv2.line(frame, pt1, pt2, color, thickness)
    
    return frame
```

### Calculate Body Dimensions

```python
def calculate_body_metrics(kp):
    """Calculate useful body measurements"""
    metrics = {}
    
    # Shoulder width
    l_shoulder, r_shoulder = kp[5], kp[6]
    metrics["shoulder_width"] = np.linalg.norm(r_shoulder - l_shoulder)
    
    # Torso height
    shoulder_center = (l_shoulder + r_shoulder) / 2
    hip_center = (kp[11] + kp[12]) / 2
    metrics["torso_height"] = np.linalg.norm(hip_center - shoulder_center)
    
    # Total height (shoulder to ankle)
    ankle_center = (kp[15] + kp[16]) / 2
    metrics["total_height"] = np.linalg.norm(ankle_center - shoulder_center)
    
    # Arm span
    l_wrist, r_wrist = kp[9], kp[10]
    metrics["arm_span"] = np.linalg.norm(r_wrist - l_wrist)
    
    return metrics

# Usage
metrics = calculate_body_metrics(kp)
print(f"Shoulder width: {metrics['shoulder_width']:.1f} pixels")
print(f"Torso height: {metrics['torso_height']:.1f} pixels")
```

## 🐛 Troubleshooting

### Issue: No Keypoints Detected

```python
# Check if keypoints exist
if r.keypoints is None:
    print("No people detected")
    
    # Possible causes:
    # - No person in frame
    # - Person too far from camera
    # - Poor lighting
    # - Person occluded
    
    # Solutions:
    # - Lower confidence threshold
    results = model(frame, conf=0.2)  # Default is 0.25
    
    # - Check if boxes detected but no keypoints
    if r.boxes is not None:
        print(f"Detected {len(r.boxes)} people but no keypoints")
```

### Issue: Keypoints Jump Around (Jitter)

```python
# Apply smoothing filter
from collections import deque

keypoint_history = deque(maxlen=5)

def smooth_keypoints(kp, history):
    """Smooth keypoints using moving average"""
    history.append(kp)
    
    if len(history) < 3:
        return kp
    
    # Average of last N frames
    smoothed = np.mean(list(history), axis=0)
    return smoothed

# Usage
kp_smoothed = smooth_keypoints(kp, keypoint_history)
```

### Issue: Wrong Person Assigned

For multi-person scenarios, implement proper tracking (see Person Tracking section above).

## 📚 Further Reading

- **[03_YOLO_MODEL_INFERENCE.md](03_YOLO_MODEL_INFERENCE.md)** - How YOLO produces keypoints
- **[05_ACTION_DETECTION_ENGINE.md](05_ACTION_DETECTION_ENGINE.md)** - Using keypoints for detection
- **[../06_ACTION_DETECTION.md](../06_ACTION_DETECTION.md)** - COCO keypoint format details

---

← [Previous: YOLO Model Inference](03_YOLO_MODEL_INFERENCE.md) | [Back to Main Thread Docs](00_README.md) | [Next: Action Detection Engine →](05_ACTION_DETECTION_ENGINE.md)
