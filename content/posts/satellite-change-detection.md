---
title: "Building a Satellite Image Change Detection System with DINOv2 and VM-UNet"
date: 2026-05-06
tags: ["deep-learning", "computer-vision", "remote-sensing", "pytorch"]
categories: ["AI & ML"]
description: "How I built a change detection pipeline to identify damaged buildings in post-disaster satellite imagery using DINOv2 feature extraction and VM-UNet segmentation."
showToc: true
draft: false
---

Satellite imagery gives us a unique bird's-eye view of the world — and when disaster strikes, that view becomes critical. Emergency responders need to know *exactly* where buildings have been damaged and how severely, often within hours of an event. Manually scanning thousands of square kilometres of imagery is not feasible. This is the problem I set out to solve.

## The Problem: Change Detection at Scale

Change detection in remote sensing means comparing two images of the same area taken at different times and identifying what changed. In a post-disaster context, that means comparing **pre-event** and **post-event** satellite images and classifying each building pixel as: no damage, minor damage, major damage, or destroyed.

The challenge is that satellite images are high-resolution, the changes can be subtle (a partially collapsed roof looks similar from above), and the model needs to understand *both* images simultaneously.

## Dataset: xBD

I trained and evaluated on the [xBD dataset](https://xview2.org/), the largest publicly available dataset for building damage assessment. It contains:

- 22,068 image pairs (pre/post disaster)
- Six disaster types: wildfire, flood, earthquake, hurricane, tsunami, volcanic eruption
- Pixel-level damage labels in four categories

## Architecture

The pipeline has two main stages:

### Stage 1 — Feature Extraction with DINOv2

[DINOv2](https://github.com/facebookresearch/dinov2) is a self-supervised Vision Transformer trained by Meta on a massive curated dataset. I used it as a **frozen feature extractor** — no fine-tuning. Given a pre/post image pair, DINOv2 produces rich spatial feature maps that capture texture, structure, and context far better than a CNN trained from scratch.

```python
import torch
from dinov2.models.vision_transformer import vit_large

model = vit_large(patch_size=14, img_size=518)
model.load_state_dict(torch.load("dinov2_vitl14.pth"))
model.eval()

with torch.no_grad():
    pre_features  = model.forward_features(pre_image)["x_norm_patchtokens"]
    post_features = model.forward_features(post_image)["x_norm_patchtokens"]

# Concatenate along channel dim for the segmentation head
change_features = torch.cat([pre_features, post_features], dim=-1)
```

### Stage 2 — Segmentation with VM-UNet

VM-UNet is a UNet-style architecture with **Visual Mamba** blocks replacing the standard convolutional encoder. Mamba's selective state-space mechanism lets the model capture long-range spatial dependencies efficiently — important when damage patterns span large regions.

The concatenated DINOv2 features feed into VM-UNet's encoder, and the decoder upsamples back to the original resolution, outputting a per-pixel damage class.

## Visualising Results: Colour-Coded Heatmaps

Raw segmentation masks aren't very interpretable for non-technical users. I post-processed the output into **multi-colour heatmaps**:

```python
import cv2
import numpy as np

DAMAGE_PALETTE = {
    0: (200, 200, 200),   # No damage — grey
    1: (255, 255, 0),     # Minor damage — yellow
    2: (255, 140, 0),     # Major damage — orange
    3: (220, 20, 60),     # Destroyed — red
}

def mask_to_heatmap(mask: np.ndarray) -> np.ndarray:
    h, w = mask.shape
    heatmap = np.zeros((h, w, 3), dtype=np.uint8)
    for cls, color in DAMAGE_PALETTE.items():
        heatmap[mask == cls] = color
    return cv2.cvtColor(heatmap, cv2.COLOR_RGB2BGR)
```

Overlaying this on the post-disaster image gives responders an immediately actionable visual.

## Results

The model achieves competitive F1 scores across damage categories on the xBD test set, with the strongest performance on the "destroyed" class — which is also the most safety-critical for emergency response.

## What I Learned

- **Frozen foundation models are powerful** — DINOv2 without any fine-tuning produced better features than a ResNet50 trained specifically on remote sensing data.
- **State-space models scale well** — VM-UNet handled the large spatial context without the quadratic memory cost of full self-attention.
- **Visualisation matters as much as accuracy** — a model that outputs numbers nobody can read has limited real-world impact.

## What's Next

The natural next step is integrating this into a web interface where emergency agencies can upload image pairs and get annotated damage maps instantly. The paper built on this work — *"Hybrid Foundation-State Space Networks for Building Damage Assessment in Remote Sensing"* — was selected for an oral presentation at CVR-2026.
