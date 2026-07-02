---
title: "How I Built a System to Spot Building Damage in Satellite Images"
date: 2026-07-02T22:39:52+05:30
draft: true
tags: []
categories: []
description: ""
showToc: true
cover:
  image: ""
  alt: ""
  caption: ""
---

Last year I was watching news coverage of the US strikes on Iranian nuclear sites. The channel put up two satellite images side by side, one from before the strike and one from after, and the anchor pointed at a spot on the second image and said this is where the damage happened. I stared at both pictures for a while and honestly couldn't tell. The buildings looked more or less the same to me. If a trained analyst had to point out the damage frame by frame on live television, I figured there was room for a system that could do it automatically and consistently.

That's where this project started. I wanted something that takes a before image and an after image of the same location and tells you, visually, where the damage is and how severe it is, so that after a disaster the people deciding where to send help aren't stuck squinting at two nearly identical photos. The paper describing the final version, DINOv2-VM-UNet, is co-authored with Harsshita Vontivillu, Iris Soj, and Divya Chirayil, and it was accepted at CVR 2026. I also put a working demo online at geosense.piyushdeshmukh.dev.

## Why this is harder than it looks

Satellite imagery is already used for post-disaster response because it doesn't require sending people into a collapsed or flooded area just to see what's still standing. But turning two images into a reliable damage map runs into a few real problems:

- Damage cues like debris, shifted roofs, or cracks are subtle and easy to miss.
- The same location photographed on different days looks different because of viewing angle, sunlight, shadows, and atmospheric haze, none of which have anything to do with actual damage.
- The images are large, typically 1024×1024 pixels, which is a lot to push through a neural network without running into memory limits.
- Damage looks different depending on the disaster. An earthquake leaves different visual signatures than a flood or a wildfire, so a model trained on one type doesn't automatically generalize to another.
- Damaged buildings typically make up under 2–5 percent of the pixels in an image. A model can score well on raw accuracy just by predicting "no damage" everywhere and still miss almost every damaged building.

That last point turned out to be the most stubborn problem in practice, and I'll get to how we dealt with it.

## The approach: a pretrained vision model paired with a lighter decoder

The system, DINOv2-VM-UNet, has two main pieces: an encoder that reads the images and a decoder that turns what it read into a damage map.

The encoder is DINOv2, a vision transformer trained on a large collection of ordinary photographs, not satellite images, using self-supervised learning. It's good at pulling out semantically meaningful features from an image without needing labels for what it's looking at. We run the pre-disaster and post-disaster patches through this same encoder separately, then subtract the post-disaster feature map from the pre-disaster one. What's left is a change signal: the parts of the scene that shifted structurally rather than just a lighting difference.

Since DINOv2 was never trained on satellite imagery, we didn't fine-tune the whole thing. We used two lightweight techniques, LoRA and bottleneck adapters, which add small trainable pieces into an otherwise frozen network. This lets the model adjust to the new domain without retraining millions of parameters or overfitting on a dataset that isn't huge to begin with.

The decoder is where the state-space part comes in. Instead of standard convolutions or a transformer, the decoder (VM-UNet) uses Vision Mamba blocks, which model the whole image with state-space math instead of attention. The practical upshot is that it scales linearly with image size instead of the quadratic cost you get with full-resolution attention, so it can take in the entire scene at once without the compute cost exploding. It scans the image in both directions, so detection doesn't depend on which way a damage pattern happens to be oriented. Skip connections carry the change features from multiple depths of the encoder into the decoder, so the output keeps both the fine detail of a building edge and the broader context of the surrounding block.

Because the images are big, we don't feed them in whole. They're split into overlapping patches sized to the encoder's native resolution (multiples of 14×14), processed independently, and the overlapping predictions are averaged back together at inference time. That overlap is what keeps the seams between patches from showing up as artifacts in the final map.

## Getting the model to actually notice the damage

The class imbalance problem needed direct intervention. Early training runs hit 94 percent pixel accuracy while getting almost zero IoU, 0.18 percent, on the actual damage classes. In other words, the model was basically predicting "no damage" everywhere and still looking good on paper.

To fix that, we used geographic oversampling, which biases patch sampling toward regions that contain damage while still keeping some undamaged context around them, along with a dynamic weighted loss. A script re-evaluated sampling and loss weights every three epochs, testing combinations where background and no-damage classes were weighted down (0.1×–0.5×), minor damage got moderate weight (20×–50×), and major or destroyed damage got weighted way up (100×–200×), keeping whichever combination did best on minority-class validation F1.

We also trained three models with different random seeds and combined their predictions by majority vote. That ensemble cuts down on variance for ambiguous cases, at the cost of roughly three times the inference time.

## How well it actually works

We tested on xView2 (xBD), the standard benchmark for this task. It contains high-resolution before/after RGB pairs across 19 disaster events, including hurricanes, volcanic eruptions, floods, tsunamis, earthquakes, and wildfires, with pixels labeled as background, no damage, minor damage, major damage, or destroyed.

On the standard xView2 metrics, DINOv2-VM-UNet got a localization F1 of 87.32, a damage classification F1 of 77.10, and an overall weighted F1 of 80.16. That's ahead of CNN-based baselines like Siamese-UNet and the ChangeOS family, and ahead of DamFormer-Tiny. It gets close to, but doesn't beat, MambaBDA-Base, the strongest baseline in our comparison, which scored 81.41 overall F1.

The ablation study gives a more honest picture of where the gains actually came from. A basic U-Net-style baseline using a simple pre/post difference scored 0.26 in overall F1. Swapping in the DINOv2 encoder alone brought that to 0.56, the largest single jump in the study. Adding the VM-UNet decoder brought it to 0.65. Everything after that, the fine-tuning method, multi-scale fusion, oversampling, dynamic weighting, and the ensemble, added smaller amounts each, between 0.01 and 0.04, stacking up to the final 0.80.

## Where it still falls short

DINOv2 was pretrained on natural images, and satellite imagery is a different visual domain. That gap is still there, and the paper calls it out directly as a real limitation we haven't solved. The ensemble that helps accuracy also roughly triples inference time, which matters if you're trying to run this on limited hardware. Training was also capped at 40 epochs because of compute constraints, and the training curves were still trending upward across all three seeds when we stopped, so longer training would likely help further.

## What's next

A few directions worth exploring: quantizing the model so it can run on smaller hardware for edge deployment, combining SAR (radar) imagery with the optical images used here, since radar can see through cloud cover that optical satellites can't, and continued pretraining on remote sensing data specifically to close the domain gap between natural images and satellite imagery.

If you want to try it, the demo is live at [geosense.piyushdeshmukh.dev](https://geosense.piyushdeshmukh.dev/). Feed it a before and after image of the same location and it produces the damage heatmap directly.