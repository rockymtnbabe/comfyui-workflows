# Qwen Edit 2511 + ControlNet (v1.1)

## Version
Workflow Version: v1.1  
Last Updated: 2025-01-02

## Purpose
This workflow is designed for Qwen Image Edit with optional ControlNet guidance.
It supports:
- Style / clothing edits using Image2
- Pose / structure guidance using ControlNet
- Running Qwen edits without ControlNet when not needed

## When to Use
- Changing clothing or style on a base image
- Pose transfer with OpenPose or Depth
- Iterative editing without reloading models

## Image Inputs
- Image1: Base image to edit
- Image2: Optional reference image
  - Style mode: clothing / aesthetic reference
  - Pose mode: pose or depth map input
- Image3: Additional image for style or subject transfer as needed

## ControlNet Behavior
- ControlNet is additive
- Workflow functions with ControlNet bypassed
- If Image2 is bypassed, ControlNet must also be bypassed

## Known Quirks
- Qwen `image2` input is required if the ControlNet group is enabled
- Bypassing Image2 with ControlNet enabled will cause missing-input errors