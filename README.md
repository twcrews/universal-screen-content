# Universal Screen Content (USC)

Universal Screen Content (USC) is a design specification for creating full-screen content that displays correctly across a wide range of aspect ratios and orientations — portrait and landscape alike — without cropping, distortion, or loss of critical information.

## Overview

Modern screens come in a staggering variety of shapes: ultrawide monitors, square displays, standard 16:9 TVs, tablets, phones held upright or sideways, and everything in between. Designing content that works well across all of these is a non-trivial challenge.

USC addresses this by defining a set of rules and safe areas that, when respected, guarantee content remains usable and visually coherent on any target screen.

## Definition

> **Universal Screen Content** is any screen content which can be properly viewed in either portrait or landscape orientation, up to a given maximum aspect ratio.

"Properly viewed" means:
- All user-critical content is fully visible and unobstructed
- The content is not distorted, stretched, or cropped in a way that impairs comprehension or aesthetics
- The content is legible and functional at any supported orientation

## Content Types

USC distinguishes between two broad categories of screen content:

### 1. Dynamically Resizable Content

Content that can reflow, scale, or recompose itself to fill the available viewport — for example, a web page, a native app UI, or a vector graphic with responsive layout rules.

For dynamically resizable content, USC compliance requires that the layout logic respects the safe area definitions for each target aspect ratio (see [Safe Areas](#safe-areas) below).

### 2. Non-Resizable Content

Content with fixed dimensions that cannot reflow — for example, a raster image or a pre-rendered video.

For non-resizable content, USC compliance requires:

1. **The content must be square (1:1 aspect ratio).** A square is the only shape that, when centered, fits within both a landscape and a portrait viewport of any aspect ratio without requiring different crops.
2. **All user-critical content must exist within a centered "safe square"** whose size is determined by the maximum supported aspect ratio. Content outside the safe square may be partially or fully cropped on certain screens and must therefore be decorative only.

## Safe Areas

When designing non-resizable USC content, the **safe area** is the largest centered square that remains fully visible on a screen with a given aspect ratio — in both portrait and landscape orientations.

Because the canvas is square (1:1), rotating the viewport between portrait and landscape is equivalent to toggling between a tall and a wide crop of the same square canvas. The safe area at a given aspect ratio `W:H` (where `W ≥ H`) is the region of the square canvas that is visible when the canvas is letterboxed or pillarboxed into a `W:H` rectangle.

For a square canvas of side length `S` displayed on a screen with aspect ratio `W:H`:

- **Landscape**: the canvas is pillarboxed; the visible height is `S`, the visible width is `S × (H/W)` centered. The safe area width is `S × (H/W)`.
- **Portrait** (i.e., `H:W`): the canvas is letterboxed; the visible width is `S`, the visible height is `S × (H/W)`. The safe area height is `S × (H/W)`.

The intersection of these two constraints is a centered square with side length `S × (H/W)`, where `H` is the shorter dimension of the target aspect ratio. This is the USC safe area for that aspect ratio.

### Safe Area Guide

The diagram below illustrates the nested safe areas for common aspect ratios, centered on a square canvas. Each colored region represents the safe area boundary for a different aspect ratio — content within that boundary is guaranteed to be visible on screens up to that aspect ratio in either orientation.

![USC Safe Area Guide](Guide.svg)

Safe areas shown (from innermost to outermost):

| Aspect Ratio | Landscape Safe Width | Common Uses |
|---|---|---|
| 32:9 | ~28% of canvas | Super-ultrawide displays |
| 21:9 | ~43% of canvas | Ultrawide displays; modern iPhones (approx.) |
| 2:1 | 50% of canvas | Many phones (range from here to 21:9) |
| 16:9 | ~56% of canvas | Most modern TVs and monitors |
| 8:5 | 62.5% of canvas | MacBooks, high-end laptops |
| 3:2 | ~67% of canvas | Older phones |
| 4:3 | 75% of canvas | iPads/tablets, old/industrial monitors, old TVs |
| 5:4 | 80% of canvas | Foldable phones, old/industrial monitors |
| 6:5 | ~83% of canvas | Foldable phones |
| 1:1 | 100% of canvas | Rare, specialized displays |

> Note: Portrait equivalents are the reciprocal of each ratio (e.g., 9:16, 9:21, 2:3, etc.) and are symmetric — the same safe area applies.

## Choosing a Maximum Aspect Ratio

The maximum aspect ratio defines the outermost safe area and therefore how much of the canvas edges can be used for decorative content. Choosing a tighter maximum (closer to 1:1) wastes less canvas but limits ambition; choosing a wider maximum (e.g., 21:9) allows richer edge content but means that content will be cropped on ultra-wide screens.

**Recommended defaults:**

- **General-purpose content** (unknown audience): target **16:9** as the maximum. This covers the vast majority of TVs, monitors, and laptops, while keeping safe area content reasonably large.
- **Social/mobile-first content**: target **2:1** or **21:9**, since phones are the primary viewing device.
- **Broadcast/presentation**: target **16:9**.
- **Print-to-screen or square-format platforms**: target **4:3** or **1:1**.

## Applying USC in Practice

### For Designers

1. Start with a square canvas in your design tool.
2. Draw a centered safe area rectangle for your maximum target aspect ratio (see the guide above for dimensions).
3. Place all user-critical content — logos, text, focal subjects, UI controls — within the safe area.
4. Use the region outside the safe area for background fills, decorative elements, or bleed.

### For Developers

- When displaying USC content, center the square asset within the viewport and use `object-fit: contain` (or equivalent) to prevent stretching.
- Do not crop or pan the asset automatically; the safe area convention assumes centered display.
- If the platform requires a specific aspect ratio crop, crop from the center.

### For Content Creators (Video/Photo)

- Shoot and compose within a square frame where possible.
- Keep the main subject and all essential on-screen text within the safe area for your target maximum aspect ratio.
- Use guides or overlays during production that match the safe area dimensions shown above.

## USC Compliance Levels

A piece of content may be described as USC-compliant at a specific maximum aspect ratio. For example:

- **USC-1:1** — Critical content is safe on square displays and wider.
- **USC-16:9** — Critical content is safe on 16:9 displays and wider. *(Recommended minimum)*
- **USC-32:9** — Critical content is safe on super-ultrawide displays and wider.

Higher numbers (wider ratios) indicate more aggressive cropping tolerance; the safe area is smaller and the content must be more tightly composed.
