# Universal Screen Content (USC)

USC is a design specification for creating full-screen content that displays correctly across any aspect ratio and orientation — portrait or landscape — without cropping, distortion, or loss of critical information.

## The Problem

Modern screens come in a staggering variety of shapes: ultrawide monitors, square displays, 16:9 TVs, tablets, phones held upright or sideways, and everything in between. A single image or video rarely survives all of them intact. Platform-specific safe zone guidelines address device cutouts; responsive layout conventions address apps and web pages. Neither provides a portable standard for fixed-dimension content that must survive any screen shape and orientation, on any platform, without modification.

<!-- Example: side-by-side of the same image cropped on a phone vs. ultrawide monitor -->

## The Solution

USC defines a **square canvas** and a set of nested **safe areas** — one per supported aspect ratio. Critical content placed inside the safe area is guaranteed to remain visible regardless of how the screen crops the canvas.

<!-- Example: annotated image showing safe vs. decorative zones -->

## How It Works

For a square canvas of side length `S` displayed on a screen with aspect ratio `W:H`, the safe area is a centered square with side length `S × (H/W)`, where `H` is the shorter dimension. This is the largest region that stays fully visible in both portrait and landscape orientations.

| Aspect Ratio | Safe Area Width | Common Screens |
|---|---|---|
| 16:9 | ~56% of canvas | TVs, monitors, laptops |
| 4:3 | 75% of canvas | Tablets, older displays |
| 2:1 | 50% of canvas | Many smartphones |
| 21:9 | ~43% of canvas | Ultrawide monitors, modern iPhones |

A piece of content is described as **USC-compliant at a specific level** — e.g., **USC-16:9** means all critical content is safe on any screen with an aspect ratio of 16:9 or less extreme (16:9, 8:5, 4:3, 1:1, etc.) in either orientation. Note that a wider aspect ratio means a *smaller* safe area: USC-16:9 requires tighter composition than USC-4:3, but is compatible with more screen shapes.

<!-- Example: the same USC-16:9 image shown on a phone, tablet, and ultrawide -->

## Quick Start

**Designers** — Start with a square canvas. Draw a centered safe area guide for your target aspect ratio. Keep all critical content inside it; use the edges for decoration or bleed only.

**Developers** — Display USC assets centered in their container with `object-fit: contain`. Never auto-crop or auto-pan. If a crop is required, crop from center.

**Content creators** — Compose within a square frame. Keep subjects and essential text within the safe area for your target ratio.

> [!NOTE]
> The specification is not concerned with decorative spacing or padding. You should add extra padding to the specification safe areas as needed for your design purposes.

## Learn More

- [Specification](SPEC.md) — formal definitions, safe area math, compliance levels, and authoring guidance
