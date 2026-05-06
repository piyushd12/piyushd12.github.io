---
title: "Real-Time Sign Language Translation with MediaPipe and LSTM"
date: 2026-05-06
tags: ["deep-learning", "computer-vision", "lstm", "python", "mediapipe"]
categories: ["AI & ML"]
description: "Building a real-time pipeline that translates hand gestures into text using MediaPipe hand tracking and LSTM sequence networks — including a custom dataset collection tool."
showToc: true
draft: false
---

Most sign language recognition research uses pre-recorded video datasets, trains a model on them, and calls it done. The problem is that sign languages vary enormously — not just between countries (ASL vs BSL vs ISL) but even regionally within a country. A model trained on one dataset often fails completely on another.

My goal was different: build a system that *anyone* can train on *their own* sign language gestures in minutes, without writing a single line of code.

## System Overview

The pipeline has three components:

1. **Data Collection Tool** — records and labels hand gesture sequences via webcam
2. **LSTM Classifier** — trained on the collected sequences to recognise gestures
3. **Real-Time Inference Engine** — runs the model live on webcam input and outputs text

## Step 1: Hand Tracking with MediaPipe

[MediaPipe Hands](https://developers.google.com/mediapipe/solutions/vision/hand_landmarker) gives us 21 3D landmarks per hand in real time — fingertips, knuckles, wrist — at 30+ FPS on CPU. This is the foundation of the whole system.

```python
import cv2
import mediapipe as mp
import numpy as np

mp_hands = mp.solutions.hands
mp_draw  = mp.solutions.drawing_utils

def extract_landmarks(frame) -> np.ndarray | None:
    """Returns a flattened array of 21×3 landmarks, or None if no hand found."""
    with mp_hands.Hands(
        static_image_mode=False,
        max_num_hands=1,
        min_detection_confidence=0.7
    ) as hands:
        rgb = cv2.cvtColor(frame, cv2.COLOR_BGR2RGB)
        result = hands.process(rgb)
        
        if not result.multi_hand_landmarks:
            return None
        
        landmarks = result.multi_hand_landmarks[0]
        coords = np.array([[lm.x, lm.y, lm.z] for lm in landmarks.landmark])
        return coords.flatten()   # shape: (63,)
```

Using normalised landmark coordinates instead of raw pixel positions makes the model **invariant to hand size and camera distance** — a huge advantage over pixel-based approaches.

## Step 2: Custom Dataset Collection Tool

The key innovation is a simple interactive tool that lets users define their own gesture vocabulary:

```python
import os, time

def collect_gesture_data(label: str, num_sequences=30, seq_length=30):
    """
    Records `num_sequences` gesture sequences of length `seq_length` frames
    and saves them as numpy arrays under data/<label>/.
    """
    save_path = os.path.join("data", label)
    os.makedirs(save_path, exist_ok=True)
    
    cap = cv2.VideoCapture(0)
    print(f"Recording '{label}' — press SPACE to start each sequence")
    
    for seq_idx in range(num_sequences):
        # Wait for user to press space
        while True:
            ret, frame = cap.read()
            cv2.putText(frame, f"Ready? [{seq_idx+1}/{num_sequences}] — SPACE to record",
                        (10, 30), cv2.FONT_HERSHEY_SIMPLEX, 0.7, (0,255,0), 2)
            cv2.imshow("Collect", frame)
            if cv2.waitKey(1) == ord(' '):
                break
        
        sequence = []
        for frame_idx in range(seq_length):
            ret, frame = cap.read()
            landmarks = extract_landmarks(frame)
            sequence.append(landmarks if landmarks is not None else np.zeros(63))
        
        np.save(os.path.join(save_path, f"{seq_idx}.npy"), np.array(sequence))
    
    cap.release()
    cv2.destroyAllWindows()

# Example: collect data for three gestures
for gesture in ["hello", "thank_you", "yes"]:
    collect_gesture_data(gesture)
```

Each sequence is 30 frames (~1 second) of 63-dimensional landmark vectors. 30 sequences per gesture is usually enough to get good accuracy.

## Step 3: LSTM Classifier

Sign language is inherently temporal — the *motion* of a gesture matters as much as its shape. LSTMs are well-suited for this:

```python
import tensorflow as tf
from tensorflow.keras import layers, models

def build_model(num_classes: int, seq_length=30, feature_dim=63):
    model = models.Sequential([
        layers.Input(shape=(seq_length, feature_dim)),
        layers.LSTM(64, return_sequences=True),
        layers.LSTM(128, return_sequences=True),
        layers.LSTM(64),
        layers.Dense(64, activation='relu'),
        layers.Dropout(0.4),
        layers.Dense(num_classes, activation='softmax')
    ])
    model.compile(
        optimizer='adam',
        loss='categorical_crossentropy',
        metrics=['accuracy']
    )
    return model
```

Training on 30 sequences × 3 gestures takes under 60 seconds on CPU and typically reaches >95% validation accuracy.

## Step 4: Real-Time Inference

```python
import collections

def run_inference(model, label_map: dict, seq_length=30, threshold=0.85):
    cap = cv2.VideoCapture(0)
    buffer = collections.deque(maxlen=seq_length)
    current_text = ""
    
    with mp_hands.Hands(max_num_hands=1) as hands:
        while cap.isOpened():
            ret, frame = cap.read()
            
            landmarks = extract_landmarks(frame)
            buffer.append(landmarks if landmarks is not None else np.zeros(63))
            
            if len(buffer) == seq_length:
                sequence = np.expand_dims(np.array(buffer), axis=0)
                prediction = model.predict(sequence, verbose=0)[0]
                confidence = np.max(prediction)
                
                if confidence > threshold:
                    label = label_map[np.argmax(prediction)]
                    if label != current_text:
                        current_text = label
            
            cv2.putText(frame, current_text, (10, 60),
                        cv2.FONT_HERSHEY_SIMPLEX, 1.5, (0,255,100), 3)
            cv2.imshow("Sign Language Translator", frame)
            
            if cv2.waitKey(1) == ord('q'):
                break
    
    cap.release()
```

The sliding window buffer always holds the last `seq_length` frames, so inference runs continuously — no need to segment gestures manually.

## Results

On a 5-gesture custom vocabulary, the system achieves:
- **>95% accuracy** on the validation set
- **~25 FPS** inference on CPU (no GPU required)
- **~2 second startup** from launch to live inference

## Why This Approach Generalises

Because the system learns from *your* gestures in *your* environment:

- Works for **regional sign languages** not covered by any existing dataset
- Adapts to **individual signing style** — everyone's gestures differ slightly
- **No internet required** — everything runs locally
- Extendable to **two-hand gestures** by doubling the landmark vector (126 dims)

## What I Would Do Differently

Using a Transformer-based sequence model (like a small temporal Transformer) instead of LSTM would likely improve accuracy on longer, more complex gestures. The data collection pipeline is also a bottleneck — 30 sequences takes about 5 minutes; active learning could cut that significantly.

The full code and dataset collection scripts are available on my GitHub.
