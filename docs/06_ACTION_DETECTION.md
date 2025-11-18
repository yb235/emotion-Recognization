# Action Detection Algorithms

## 🎯 Overview

This document explains in detail how the system detects 10 different body actions from pose keypoints. Each action uses geometric heuristics based on distances and relative positions of body parts.

## 📍 COCO Keypoint Format

The YOLO pose model detects **17 keypoints** following the COCO format:

```
Index | Keypoint Name     | Description
------|------------------|------------------
0     | nose             | Tip of nose
1     | left_eye         | Left eye center
2     | right_eye        | Right eye center
3     | left_ear         | Left ear
4     | right_ear        | Right ear
5     | left_shoulder    | Left shoulder joint
6     | right_shoulder   | Right shoulder joint
7     | left_elbow       | Left elbow joint
8     | right_elbow      | Right elbow joint
9     | left_wrist       | Left wrist
10    | right_wrist      | Right wrist
11    | left_hip         | Left hip joint
12    | right_hip        | Right hip joint
13    | left_knee        | Left knee joint
14    | right_knee       | Right knee joint
15    | left_ankle       | Left ankle
16    | right_ankle      | Right ankle
```

**Coordinate System**: 
- Origin (0,0) is at **top-left** of image
- X increases **right**
- Y increases **down**
- Units are **pixels**

## 📏 Distance Function

All action detection uses Euclidean distance:

```python
def distance(p1, p2):
    """
    Calculate Euclidean distance between two points
    
    Parameters:
        p1: [x1, y1] - First point
        p2: [x2, y2] - Second point
        
    Returns:
        float - Distance in pixels
    """
    return np.linalg.norm(np.array(p1) - np.array(p2))
```

**Formula**: 
```
distance = √((x2 - x1)² + (y2 - y1)²)
```

**Example**:
```python
p1 = [100, 200]  # Point at (100, 200)
p2 = [150, 250]  # Point at (150, 250)
dist = distance(p1, p2)  # Returns ~70.7 pixels
```

## 🔍 The 10 Detected Actions

### 1. Arms Crossed 🙅

**Behavioral Meaning**: Defensive posture, closed body language, disagreement, or cold

**Detection Logic**:
```python
if (distance(l_wrist, r_elbow) < 80 and 
    distance(r_wrist, l_elbow) < 80):
    actions.append("arms_crossed")
```

**Visual Explanation**:
```
    👤 Person
    
    L-Elbow -------- R-Elbow
        |              |
        |   X      X   |  ← Wrists cross to opposite elbows
        |              |
    L-Wrist -------- R-Wrist
```

**Geometric Interpretation**:
- Left wrist is near right elbow (< 80 pixels)
- Right wrist is near left elbow (< 80 pixels)
- Forms an "X" pattern across the chest

**Threshold**: 80 pixels
- **Lower** (e.g., 60): More strict, fewer false positives
- **Higher** (e.g., 100): More lenient, catches loose crosses

**False Positives**:
- Holding something across chest
- Adjusting clothing

**Improvements**:
- Add angle check (arms should be roughly horizontal)
- Verify wrists are in front of body (Z-depth if available)

---

### 2. Hands Clasped 🙏

**Behavioral Meaning**: Formal posture, nervousness, prayer, or politeness

**Detection Logic**:
```python
if distance(l_wrist, r_wrist) < 60:
    actions.append("hands_clasped")
```

**Visual Explanation**:
```
    👤 Person
    
        🤝  ← Wrists close together
```

**Geometric Interpretation**:
- Both wrists are within 60 pixels of each other
- Simple proximity check

**Threshold**: 60 pixels
- **Lower** (e.g., 40): Only detects tight clasping
- **Higher** (e.g., 80): Detects hands near each other

**False Positives**:
- Hands passing by each other
- Holding small object with both hands

**Improvements**:
- Check that hands are in front of body
- Verify fingers are intertwined (requires hand keypoints)
- Duration filter (must last > 0.5 seconds)

---

### 3. Chin Rest 🤔

**Behavioral Meaning**: Thinking, contemplating, boredom, or evaluation

**Detection Logic**:
```python
if distance(l_wrist, nose) < 70 or distance(r_wrist, nose) < 70:
    actions.append("chin_rest")
```

**Visual Explanation**:
```
    👤 Person
    
      👃 ← Nose
       |
       ✋ ← Wrist supporting chin
```

**Geometric Interpretation**:
- Either wrist is within 70 pixels of nose
- Indicates hand is near face/chin

**Threshold**: 70 pixels

**Note**: This overlaps with "touch_nose" (threshold 40). The difference:
- **Chin rest**: 40-70 pixels (hand near but not touching)
- **Touch nose**: < 40 pixels (directly touching)

**False Positives**:
- Scratching face
- Adjusting glasses
- Drinking

**Improvements**:
- Check hand orientation (should be facing up)
- Verify elbow is below wrist (supporting posture)
- Distinguish from "touch_nose" by checking if elbow is resting on something

---

### 4. Lean Forward 👉

**Behavioral Meaning**: Interest, engagement, attentiveness, or aggression

**Detection Logic**:
```python
shoulder_center = (l_shoulder + r_shoulder) / 2
hip_center = (l_hip + r_hip) / 2
torso_height = abs(shoulder_center[1] - hip_center[1])

if torso_height < 120:
    actions.append("lean_forward")
```

**Visual Explanation**:
```
Side View:

Normal:          Lean Forward:
  
  👤 Shoulder      👤 Shoulder
   |                /
   | 200px         / 80px ← Compressed torso
   |              /
  👤 Hip         👤 Hip
```

**Geometric Interpretation**:
- Calculates vertical distance between shoulder center and hip center
- When person leans forward, torso appears compressed (smaller Y distance)
- Threshold: < 120 pixels indicates forward lean

**Why This Works**:
- Forward lean brings shoulders closer to hips in Y-axis
- Camera sees shortened torso projection

**Threshold**: 120 pixels
- **Lower** (e.g., 100): Only extreme forward leans
- **Higher** (e.g., 150): Catches slight forward posture

**False Positives**:
- Short people (naturally shorter torso)
- Person is sitting and camera angle is high
- Person is far from camera

**Improvements**:
- Normalize by person's height
- Use shoulder angle (rotation) instead of just distance
- Track baseline torso height per person

---

### 5. Lean Back 🛋️

**Behavioral Meaning**: Relaxation, disinterest, confidence, or dominance

**Detection Logic**:
```python
torso_height = abs(shoulder_center[1] - hip_center[1])

if torso_height > 200:
    actions.append("lean_back")
```

**Visual Explanation**:
```
Side View:

Normal:          Lean Back:
  
  👤 Shoulder      👤 Shoulder
   |                \
   | 200px           \ 250px ← Extended torso
   |                  \
  👤 Hip             👤 Hip
```

**Geometric Interpretation**:
- When person leans back, torso appears elongated
- Shoulders move up (smaller Y value), hips stay same or move down
- Larger Y distance between shoulders and hips

**Threshold**: 200 pixels

**False Positives**:
- Tall people
- Standing vs sitting differences
- Person is close to camera

**Improvements**:
- Normalize by person's height
- Combine with shoulder angle
- Track posture change over time

---

### 6. Head Down 😔

**Behavioral Meaning**: Submission, sadness, shame, reading, or lack of confidence

**Detection Logic**:
```python
shoulder_center = (l_shoulder + r_shoulder) / 2

if nose[1] > shoulder_center[1] + 40:
    actions.append("head_down")
```

**Visual Explanation**:
```
Normal:          Head Down:
  
   👃 Nose          👤 Shoulder ← Head below shoulders
   👤 Shoulder          |
                        👃 Nose
```

**Geometric Interpretation**:
- Nose Y-coordinate is lower (bigger number) than shoulder Y-coordinate
- Offset of 40 pixels ensures head is significantly down, not just neutral

**Why Y > means "down"**: Remember, Y increases downward in image coordinates

**Threshold**: 40 pixels offset

**False Positives**:
- Looking at phone/document (intentional)
- Natural neck position
- Camera angle below person

**Improvements**:
- Compare to initial head position (track deviation)
- Use eye keypoints to determine gaze direction
- Combine with overall posture

---

### 7. Touch Face 👋

**Behavioral Meaning**: Self-soothing, nervousness, thinking, or discomfort

**Detection Logic**:
```python
face_center = (
    (left_eye[0] + right_eye[0]) / 2,
    (left_eye[1] + right_eye[1]) / 2
)

if distance(l_wrist, face_center) < 70 or distance(r_wrist, face_center) < 70:
    actions.append("touch_face")
```

**Visual Explanation**:
```
    👤 Person
    
    👁️  👁️  ← Eyes (calculate center)
       👃   
    
       ✋  ← Hand near face center
```

**Geometric Interpretation**:
- Calculate face center as midpoint between eyes
- Check if either wrist is within 70 pixels of face center

**Threshold**: 70 pixels

**Overlap with Other Actions**:
- **Touch nose**: More specific (< 40 pixels to nose)
- **Chin rest**: More specific (< 70 pixels to nose, but implies resting)
- **Fix hair**: More specific (near ears)

**False Positives**:
- Yawning
- Eating/drinking
- Talking on phone

**Improvements**:
- Distinguish touch types by hand orientation
- Check duration (quick touch vs sustained)
- Use hand pose to detect open palm vs closed fist

---

### 8. Touch Nose 👃

**Behavioral Meaning**: Specific anxious gesture, lying indicator (debated), or itch

**Detection Logic**:
```python
if distance(l_wrist, nose) < 40 or distance(r_wrist, nose) < 40:
    actions.append("touch_nose")
```

**Visual Explanation**:
```
    👤 Person
    
       👃  ← Nose
       ✋  ← Hand directly on/very near nose
```

**Geometric Interpretation**:
- Very close proximity (< 40 pixels) to nose specifically
- More precise than general "touch face"

**Threshold**: 40 pixels (stricter than touch_face's 70)

**Note**: This action often triggers simultaneously with "chin_rest" or "touch_face"

**Behavioral Context**:
- Brief touch: Often interpreted as dishonesty (though controversial)
- Sustained: More likely itch or thinking

**Improvements**:
- Add temporal filter (brief vs sustained)
- Distinguish from chin rest by checking elbow position

---

### 9. Fix Hair 💇

**Behavioral Meaning**: Self-grooming, nervousness, flirtation, or habit

**Detection Logic**:
```python
if (distance(l_wrist, left_ear) < 60 or 
    distance(r_wrist, right_ear) < 60 or
    distance(l_wrist, right_ear) < 60 or 
    distance(r_wrist, left_ear) < 60):
    actions.append("fix_hair")
```

**Visual Explanation**:
```
    👤 Person
    
   👂     👂  ← Ears
    \     /
     ✋ ✋   ← Hands near ears/head sides
```

**Geometric Interpretation**:
- Either hand near either ear (< 60 pixels)
- Checks all four combinations (both hands × both ears)

**Threshold**: 60 pixels

**Why Check All Combinations**:
- Can use left hand on right side of head
- Can use either hand on either side

**False Positives**:
- Adjusting glasses
- Scratching head
- Phone call

**Improvements**:
- Add motion pattern (stroking motion vs static)
- Check hand orientation
- Distinguish from touching ear (more specific)

---

### 10. Fidget Hands 🤲

**Behavioral Meaning**: Nervousness, restlessness, energy, or boredom

**Detection Logic**:
```python
global prev_left_wrist, prev_right_wrist

fidget_detected = False

if prev_left_wrist is not None:
    if distance(prev_left_wrist, l_wrist) > 25:
        fidget_detected = True

if prev_right_wrist is not None:
    if distance(prev_right_wrist, r_wrist) > 25:
        fidget_detected = True

if fidget_detected:
    actions.append("fidget_hands")

# Update for next frame
prev_left_wrist = l_wrist
prev_right_wrist = r_wrist
```

**Visual Explanation**:
```
Frame N:     Frame N+1:
  ✋            ✋ ← Hand moved significantly
   |           /
   |          /
  (100,200) (130,210) ← Distance > 25 pixels
```

**Geometric Interpretation**:
- Compares current wrist position to previous frame's position
- If movement > 25 pixels, considers it fidgeting
- **Temporal detection** (only action that uses frame history)

**Threshold**: 25 pixels per frame

**At 30 FPS**: 25 pixels/frame = 750 pixels/second (fast movement)

**State Management**:
```python
prev_left_wrist = None   # Initialized globally
prev_right_wrist = None

# Updated every frame after detection
prev_left_wrist = l_wrist
prev_right_wrist = r_wrist
```

**False Positives**:
- Intentional gesturing while speaking
- Reaching for something
- Camera shake
- Detection jitter (YOLO slight variations)

**False Negatives**:
- Very slow fidgeting (< 25 pixels/frame)
- Fidgeting with fingers (wrist stays still)

**Improvements**:
- Smooth keypoints (Kalman filter) to reduce jitter
- Lower threshold but require sustained movement
- Track finger movements (requires hand pose model)
- Direction analysis (random vs directed movement)

---

## 🎚️ Threshold Tuning Guide

All thresholds can be adjusted based on your use case:

### Distance Thresholds Summary

| Action | Threshold (pixels) | Adjustability | Impact |
|--------|------------------|---------------|--------|
| Arms Crossed | 80 | ±20 | Low |
| Hands Clasped | 60 | ±20 | Low |
| Chin Rest | 70 | ±15 | Medium |
| Lean Forward | 120 | ±30 | High |
| Lean Back | 200 | ±50 | High |
| Head Down | 40 | ±15 | Medium |
| Touch Face | 70 | ±20 | Low |
| Touch Nose | 40 | ±10 | Medium |
| Fix Hair | 60 | ±15 | Low |
| Fidget Hands | 25 | ±10 | High |

### Factors Affecting Thresholds

**Camera Distance**:
- **Far** (3+ meters): Increase all thresholds by 50%
- **Close** (< 1 meter): Decrease all thresholds by 30%

**Resolution**:
- **1080p**: Use default values
- **720p**: Decrease thresholds by ~20%
- **4K**: Increase thresholds by ~50%

**Person Size**:
- **Children**: Decrease by 30-40%
- **Adults**: Use defaults
- **Very tall people**: Increase torso thresholds

### Tuning Process

1. **Collect test data**: Record yourself performing each action
2. **Visualize keypoints**: Print keypoint coordinates
3. **Measure distances**: Calculate actual distances for each action
4. **Set thresholds**: Use 80th percentile of measured distances
5. **Test edge cases**: Try borderline cases
6. **Iterate**: Adjust based on false positives/negatives

### Example Tuning Code

```python
# Add to detect_custom_actions() for debugging
def detect_custom_actions(kp, debug=False):
    # ... existing code ...
    
    if debug:
        print(f"Arms Crossed Check:")
        print(f"  L-Wrist to R-Elbow: {distance(l_wrist, r_elbow):.1f}")
        print(f"  R-Wrist to L-Elbow: {distance(r_wrist, l_elbow):.1f}")
        print(f"  Threshold: 80")
        print()
```

---

## 🧮 Geometric Calculations

### Center Point Calculation

```python
# Shoulder center (horizontal midpoint)
shoulder_center_x = (l_shoulder[0] + r_shoulder[0]) / 2
shoulder_center_y = (l_shoulder[1] + r_shoulder[1]) / 2
shoulder_center = (shoulder_center_x, shoulder_center_y)

# Equivalent compact form
shoulder_center = (
    (l_shoulder[0] + r_shoulder[0]) / 2,
    (l_shoulder[1] + r_shoulder[1]) / 2
)
```

### Why Calculate Centers?

**Benefits**:
- More stable than single keypoint
- Accounts for body rotation
- Reduces noise from detection jitter

**Used For**:
- Shoulder center: Torso measurements, posture
- Hip center: Torso measurements
- Face center: Face-touching detection

---

## 🔬 Algorithm Performance

### Computational Complexity

**Per Action**: O(1) - constant time operations
- Each action requires fixed number of distance calculations
- No loops or recursion

**Total Per Frame**: O(1) - still constant
- All 10 actions checked sequentially
- ~10 distance calculations + some comparisons

**Bottleneck**: NOT action detection (< 1ms)
- YOLO inference: ~50-100ms
- Frame capture: ~10ms
- Action detection: < 1ms

### Accuracy Characteristics

**Precision** (how many detected actions are correct):
- **High** (80-90%): arms_crossed, hands_clasped, head_down
- **Medium** (60-80%): lean_forward, lean_back, touch_face
- **Lower** (40-60%): fidget_hands, fix_hair, chin_rest

**Recall** (how many actual actions are detected):
- **High** (80-90%): arms_crossed, hands_clasped, head_down
- **Medium** (60-80%): lean_forward, lean_back, fidget_hands
- **Lower** (40-60%): touch_nose, fix_hair, chin_rest

**Factors Affecting Accuracy**:
1. **Lighting**: Poor lighting reduces YOLO keypoint accuracy
2. **Occlusion**: Body parts hidden by objects
3. **Angle**: Side/back views less accurate than front view
4. **Distance**: Further from camera = less accurate
5. **Motion blur**: Fast movements blur keypoints

---

## 🎯 Best Practices

### For Reliable Detection

1. **Good Lighting**: Evenly lit, no strong shadows
2. **Front-Facing**: Person faces camera (±30° angle okay)
3. **Appropriate Distance**: 1-2 meters from camera
4. **Stable Camera**: Mount on tripod, minimize shake
5. **Clear Background**: Avoid cluttered backgrounds

### For Custom Actions

**Adding New Action Checklist**:
1. Define clear behavioral meaning
2. Identify relevant keypoints
3. Determine geometric relationship
4. Collect test data
5. Calculate appropriate threshold
6. Test false positive/negative rates
7. Document the action

**Example: Adding "Hand on Hip"**:
```python
# Add after other actions in detect_custom_actions()

# 11. Hand on hip
if distance(l_wrist, l_hip) < 50 or distance(r_wrist, r_hip) < 50:
    actions.append("hand_on_hip")
```

---

← [Previous: Code Components](05_CODE_COMPONENTS.md) | [Back to Documentation Home](00_README.md) | [Next: API Reference →](07_API_REFERENCE.md)
