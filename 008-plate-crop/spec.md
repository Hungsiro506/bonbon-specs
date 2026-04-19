# 008 — Plate Capture Image Cropping

## User Story

As a garage employee scanning a license plate, I want the captured image to be automatically cropped to the guide box region so that the plate detection backend receives a focused image with less noise, improving accuracy and speed.

## Acceptance Scenarios

### Scenario 1: Normal camera capture with crop
**Given** the camera is active and the guide box is visible
**When** the employee taps "Chup bien so"
**Then** the captured image is cropped to the guide box region (with 10% padding) before being sent to the detection backend

### Scenario 2: Scale factor mapping
**Given** a captured image of 4032x3024 pixels and a camera preview of 390x520 points
**When** the crop region is calculated
**Then** the scale factors correctly map the guide box from preview coordinates to image pixel coordinates

### Scenario 3: Gallery pick bypass
**Given** the user picks an image from the gallery
**When** it is processed
**Then** the full image is sent without cropping (no guide box alignment for gallery images)

### Scenario 4: Boundary clamping
**Given** the crop calculation with padding would exceed image bounds
**When** the crop is applied
**Then** the coordinates are clamped to valid ranges within the image dimensions

### Scenario 5: Missing dimensions fallback
**Given** the camera returns a photo without width/height metadata
**When** the capture is processed
**Then** the image is sent without cropping (resize-only fallback)
