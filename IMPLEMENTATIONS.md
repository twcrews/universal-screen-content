# Universal Screen Content — Implementation Examples

This document provides detailed examples of three common USC implementation patterns. Each example includes server-side logic, client-side code, and notes on key design decisions.

- [1. USC-Compliant Upload Endpoint with Safe Area Preview](#1-usc-compliant-upload-endpoint-with-safe-area-preview)
  - [Overview](#overview)
  - [Server: Image Validation (Node.js / Express)](#server-image-validation-nodejs--express)
  - [Client: Upload Form with Safe Area Preview (React)](#client-upload-form-with-safe-area-preview-react)
- [2. USC-Compliant CMS with Compliance-Level Tagging](#2-usc-compliant-cms-with-compliance-level-tagging)
  - [Overview](#overview-1)
  - [Database Schema (PostgreSQL)](#database-schema-postgresql)
  - [Asset Delivery Endpoint (Node.js / Express)](#asset-delivery-endpoint-nodejs--express)
  - [CMS Admin: Tagging UI (React)](#cms-admin-tagging-ui-react)
- [3. Design App Tooling Extension — USC Safe Area Overlay](#3-design-app-tooling-extension--usc-safe-area-overlay)
  - [Overview](#overview-2)
  - [Architecture Notes](#architecture-notes)
  - [Plugin Worker (main.ts)](#plugin-worker-maints)
  - [Plugin UI Panel (ui.html)](#plugin-ui-panel-uihtml)
  - [Overlay Math Reference](#overlay-math-reference)

## 1. USC-Compliant Upload Endpoint with Safe Area Preview

### Overview

An upload endpoint that enforces the square canvas requirement for non-resizable USC content, paired with a client-side preview component that renders a configurable safe area overlay so creators can verify their critical content placement before submitting.

### Server: Image Validation (Node.js / Express)

```ts
// upload.ts
import express from "express";
import multer from "multer";
import sharp from "sharp";

const upload = multer({ storage: multer.memoryStorage() });
const router = express.Router();

// Aspect ratio W:H → safe area fraction H/W
const USC_COMPLIANCE_LEVELS: Record<string, number> = {
  "1:1":   1.0000,
  "6:5":   0.8333,
  "5:4":   0.8000,
  "4:3":   0.7500,
  "3:2":   0.6667,
  "8:5":   0.6250,
  "16:9":  0.5625,
  "2:1":   0.5000,
  "21:9":  0.4286,
  "32:9":  0.2813,
};

router.post("/upload", upload.single("image"), async (req, res) => {
  if (!req.file) {
    return res.status(400).json({ error: "No file provided." });
  }

  const meta = await sharp(req.file.buffer).metadata();
  const { width, height } = meta;

  if (!width || !height) {
    return res.status(422).json({ error: "Could not read image dimensions." });
  }

  // USC requires square canvas for non-resizable content
  if (width !== height) {
    return res.status(422).json({
      error: "USC violation: image must be square (1:1 aspect ratio).",
      detail: `Received ${width}×${height}. Crop or pad to ${Math.max(width, height)}×${Math.max(width, height)} before uploading.`,
    });
  }

  // Optional: tag with a declared compliance level from the upload form
  const declaredLevel = req.body["usc-level"] as string | undefined;
  if (declaredLevel && !(declaredLevel in USC_COMPLIANCE_LEVELS)) {
    return res.status(422).json({
      error: `Unknown USC compliance level: "${declaredLevel}".`,
      supported: Object.keys(USC_COMPLIANCE_LEVELS),
    });
  }

  const safeAreaFraction = declaredLevel
    ? USC_COMPLIANCE_LEVELS[declaredLevel]
    : null;

  // Persist the file and metadata (implementation-specific)
  const fileId = await saveFile(req.file.buffer, {
    width,
    uscLevel: declaredLevel ?? null,
    safeAreaFraction,
  });

  return res.status(201).json({
    id: fileId,
    width,
    height,
    uscLevel: declaredLevel ?? null,
    safeAreaFraction,
  });
});
```

**Key decisions:**
- Validation uses pixel-exact equality (`width !== height`). For uploads that may have rounding from prior transforms, allow a ±1px tolerance if needed.
- `sharp` reads metadata without fully decoding the image, so this is fast even for large files.
- The declared `usc-level` is stored as metadata; it is the uploader's assertion that critical content fits within that safe area. The server does not analyze pixel content.

### Client: Upload Form with Safe Area Preview (React)

The preview renders the image in a square container and draws a centered overlay rectangle representing the safe area for the selected compliance level.

```tsx
// UploadPreview.tsx
import { useRef, useEffect, useState } from "react";

const USC_LEVELS = [
  { label: "1:1  (100%)",  value: "1:1",  fraction: 1.0000 },
  { label: "5:4  (80%)",   value: "5:4",  fraction: 0.8000 },
  { label: "4:3  (75%)",   value: "4:3",  fraction: 0.7500 },
  { label: "16:9 (56%)",   value: "16:9", fraction: 0.5625 },
  { label: "2:1  (50%)",   value: "2:1",  fraction: 0.5000 },
  { label: "21:9 (43%)",   value: "21:9", fraction: 0.4286 },
];

export function UploadPreview() {
  const canvasRef = useRef<HTMLCanvasElement>(null);
  const [image, setImage] = useState<HTMLImageElement | null>(null);
  const [level, setLevel] = useState(USC_LEVELS[3]); // default: 16:9
  const [error, setError] = useState<string | null>(null);

  function handleFile(e: React.ChangeEvent<HTMLInputElement>) {
    const file = e.target.files?.[0];
    if (!file) return;
    const url = URL.createObjectURL(file);
    const img = new Image();
    img.onload = () => {
      if (img.naturalWidth !== img.naturalHeight) {
        setError(
          `Image is ${img.naturalWidth}×${img.naturalHeight}. USC requires a square canvas.`
        );
        setImage(null);
      } else {
        setError(null);
        setImage(img);
      }
      URL.revokeObjectURL(url);
    };
    img.src = url;
  }

  useEffect(() => {
    const canvas = canvasRef.current;
    if (!canvas || !image) return;

    const SIZE = canvas.width; // canvas is square
    const ctx = canvas.getContext("2d")!;

    // Draw the image
    ctx.clearRect(0, 0, SIZE, SIZE);
    ctx.drawImage(image, 0, 0, SIZE, SIZE);

    // Draw the unsafe (cropped) region — dimmed overlay outside the safe area
    const safeSize = SIZE * level.fraction;
    const offset = (SIZE - safeSize) / 2;

    ctx.fillStyle = "rgba(0, 0, 0, 0.45)";
    // Top strip
    ctx.fillRect(0, 0, SIZE, offset);
    // Bottom strip
    ctx.fillRect(0, offset + safeSize, SIZE, offset);
    // Left strip
    ctx.fillRect(0, offset, offset, safeSize);
    // Right strip
    ctx.fillRect(offset + safeSize, offset, offset, safeSize);

    // Safe area border
    ctx.strokeStyle = "rgba(255, 220, 0, 0.9)";
    ctx.lineWidth = 2;
    ctx.setLineDash([6, 3]);
    ctx.strokeRect(offset + 1, offset + 1, safeSize - 2, safeSize - 2);

    // Label
    ctx.setLineDash([]);
    ctx.fillStyle = "rgba(255, 220, 0, 0.9)";
    ctx.font = "bold 13px sans-serif";
    ctx.fillText(`Safe area — USC-${level.value}`, offset + 6, offset + 18);
  }, [image, level]);

  async function handleSubmit() {
    if (!image) return;
    // Re-fetch the file from the input and POST to the upload endpoint
    const input = document.querySelector<HTMLInputElement>("#file-input")!;
    const file = input.files![0];
    const form = new FormData();
    form.append("image", file);
    form.append("usc-level", level.value);

    const res = await fetch("/upload", { method: "POST", body: form });
    const data = await res.json();
    if (!res.ok) alert(`Upload failed: ${data.error}`);
    else alert(`Uploaded successfully. File ID: ${data.id}`);
  }

  return (
    <div style={{ display: "flex", flexDirection: "column", gap: 12, maxWidth: 520 }}>
      <input id="file-input" type="file" accept="image/*" onChange={handleFile} />

      {error && <p style={{ color: "red" }}>{error}</p>}

      <label>
        Minimum USC compliance:{" "}
        <select value={level.value} onChange={(e) =>
          setLevel(USC_LEVELS.find((l) => l.value === e.target.value)!)
        }>
          {USC_LEVELS.map((l) => (
            <option key={l.value} value={l.value}>{l.label}</option>
          ))}
        </select>
      </label>

      {/* Square canvas — CSS enforces display size, canvas resolution is fixed */}
      <canvas
        ref={canvasRef}
        width={480}
        height={480}
        style={{ width: 480, height: 480, border: "1px solid #ccc", display: image ? "block" : "none" }}
      />

      <button disabled={!image} onClick={handleSubmit}>
        Upload with USC-{level.value} tag
      </button>
    </div>
  );
}
```

**Overlay logic:**
- The safe area side = `canvasPixelSize × (H/W)`, where `H/W` is the compliance level fraction.
- The four surrounding strips are dimmed with a semi-transparent fill so the unsafe edge area is visually distinguished from the protected safe area.
- The dashed yellow border marks the exact safe area boundary.

## 2. USC-Compliant CMS with Compliance-Level Tagging

### Overview

A content management system that records each asset's USC compliance level and exposes it through a delivery API. Consumers can request a minimum compliance level via a URL query parameter; the API either serves the asset (if it meets the requirement) or returns a compliant fallback.

The URL parameter format uses hyphens instead of colons because colons are not valid in unencoded URL query values:

```
https://cdn.example.com/images/hero.png?usc-min=16-9
```

### Database Schema (PostgreSQL)

```sql
CREATE TABLE assets (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  slug        TEXT NOT NULL UNIQUE,
  filename    TEXT NOT NULL,
  width_px    INTEGER NOT NULL,
  height_px   INTEGER NOT NULL,
  -- Stored as "W:H", e.g. "16:9", "4:3", "1:1"
  -- NULL means not tagged / unknown compliance
  usc_level   TEXT,
  -- Derived and stored for fast numeric comparison:
  -- safe_area_fraction = H / W for the declared usc_level
  safe_area_fraction NUMERIC(6, 4),
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Index for compliance-level queries
CREATE INDEX idx_assets_safe_area ON assets (safe_area_fraction);
```

### Asset Delivery Endpoint (Node.js / Express)

```ts
// assetRouter.ts
import express from "express";
import path from "path";
import { db } from "./db"; // your database client

const router = express.Router();

// Parse "16-9" → 9/16 = 0.5625
function parseUscParam(param: string): number | null {
  const parts = param.split("-").map(Number);
  if (parts.length !== 2 || parts.some(isNaN) || parts[1] === 0) return null;
  const [w, h] = parts;
  // Normalize so that w >= h (portrait and landscape are equivalent)
  return Math.min(w, h) / Math.max(w, h);
}

router.get("/images/:slug", async (req, res) => {
  const uscMinParam = req.query["usc-min"] as string | undefined;

  // Resolve the asset
  const asset = await db.query(
    `SELECT * FROM assets WHERE slug = $1 LIMIT 1`,
    [req.params.slug]
  );

  if (!asset.rows.length) {
    return res.status(404).json({ error: "Asset not found." });
  }

  const row = asset.rows[0];

  // If no compliance filter requested, serve directly
  if (!uscMinParam) {
    return serveFile(res, row.filename);
  }

  const requestedFraction = parseUscParam(uscMinParam);
  if (requestedFraction === null) {
    return res.status(400).json({
      error: `Invalid usc-min value: "${uscMinParam}". Expected format: W-H (e.g. 16-9).`,
    });
  }

  // Check asset compliance
  // A larger fraction means a larger safe area → safer for narrow screens.
  // An asset tagged USC-16:9 (fraction ≈ 0.5625) satisfies a request for
  // usc-min=4-3 (fraction = 0.75) only if asset.fraction >= requested.fraction.
  if (row.safe_area_fraction === null) {
    // Untagged assets cannot be verified — return 409 with metadata
    return res.status(409).json({
      error: "Asset has no USC compliance tag and cannot be verified.",
      asset: { id: row.id, slug: row.slug },
    });
  }

  if (row.safe_area_fraction < requestedFraction) {
    // Asset does not meet the required compliance level
    return res.status(422).json({
      error: "Asset does not meet the requested USC compliance level.",
      asset: {
        slug: row.slug,
        uscLevel: row.usc_level,
        safeAreaFraction: row.safe_area_fraction,
      },
      requested: {
        uscMin: uscMinParam,
        safeAreaFraction: requestedFraction,
      },
    });
  }

  // Asset meets or exceeds the requirement — serve it
  // Optionally include compliance metadata in response headers
  res.setHeader("X-USC-Level", row.usc_level);
  res.setHeader("X-USC-Safe-Area-Fraction", String(row.safe_area_fraction));
  return serveFile(res, row.filename);
});

function serveFile(res: express.Response, filename: string) {
  const filePath = path.join(process.env.ASSETS_DIR!, filename);
  return res.sendFile(filePath);
}

export default router;
```

**Compliance comparison logic:**

| Asset tagged | Requested min | Result |
|---|---|---|
| USC-4:3 (75%) | `usc-min=16-9` (56%) | Served — asset is more conservative |
| USC-16:9 (56%) | `usc-min=4-3` (75%) | Rejected — asset's safe area is too narrow |
| USC-16:9 (56%) | `usc-min=16-9` (56%) | Served — exact match |
| (untagged) | `usc-min=16-9` | 409 — compliance unknown |

> A higher fraction = a larger safe area = safer for more screen shapes. Requesting a minimum means "I need at least this much of the canvas to be safe."

### CMS Admin: Tagging UI (React)

```tsx
// AssetTagEditor.tsx
const USC_LEVELS = [
  { label: "1:1  — 100% safe area",  value: "1:1",  fraction: 1.0000 },
  { label: "6:5  — 83% safe area",   value: "6:5",  fraction: 0.8333 },
  { label: "5:4  — 80% safe area",   value: "5:4",  fraction: 0.8000 },
  { label: "4:3  — 75% safe area",   value: "4:3",  fraction: 0.7500 },
  { label: "3:2  — 67% safe area",   value: "3:2",  fraction: 0.6667 },
  { label: "8:5  — 63% safe area",   value: "8:5",  fraction: 0.6250 },
  { label: "16:9 — 56% safe area",   value: "16:9", fraction: 0.5625 },
  { label: "2:1  — 50% safe area",   value: "2:1",  fraction: 0.5000 },
  { label: "21:9 — 43% safe area",   value: "21:9", fraction: 0.4286 },
  { label: "32:9 — 28% safe area",   value: "32:9", fraction: 0.2813 },
];

interface Asset {
  id: string;
  slug: string;
  uscLevel: string | null;
}

export function AssetTagEditor({ asset }: { asset: Asset }) {
  const [selected, setSelected] = useState(asset.uscLevel ?? "");
  const [saved, setSaved] = useState(false);

  async function save() {
    await fetch(`/admin/assets/${asset.id}`, {
      method: "PATCH",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ uscLevel: selected || null }),
    });
    setSaved(true);
    setTimeout(() => setSaved(false), 2000);
  }

  return (
    <div>
      <label htmlFor="usc-tag">USC compliance level:</label>
      <select
        id="usc-tag"
        value={selected}
        onChange={(e) => setSelected(e.target.value)}
      >
        <option value="">(untagged)</option>
        {USC_LEVELS.map((l) => (
          <option key={l.value} value={l.value}>{l.label}</option>
        ))}
      </select>
      <button onClick={save}>Save</button>
      {saved && <span> Saved.</span>}

      {selected && (
        <p>
          Consumers may request this asset with{" "}
          <code>?usc-min={selected.replace(":", "-")}</code> or any looser
          ratio. Requests for a tighter ratio will be rejected.
        </p>
      )}
    </div>
  );
}
```

## 3. Design App Tooling Extension — USC Safe Area Overlay

### Overview

A plugin for design applications (illustrated here as a Figma plugin) that lets the designer declare a minimum USC compliance level and overlays the canvas with a grid showing the safe area boundary. The overlay updates live as the level changes and can display nested safe areas for multiple levels simultaneously.

### Architecture Notes

Design app plugins typically:
- Communicate with a sandboxed plugin worker via `postMessage`
- Use the host app's API (e.g., `figma.*`) to read frame dimensions and create/update overlay nodes
- Render a separate UI panel (an `<iframe>`) for controls

The example below is structured as a Figma plugin but the overlay math applies to any canvas-based tool.

### Plugin Worker (main.ts)

```ts
// main.ts — runs in Figma's plugin sandbox

figma.showUI(__html__, { width: 280, height: 360, title: "USC Safe Area" });

// Safe area fractions keyed by "W:H"
const USC_FRACTIONS: Record<string, number> = {
  "1:1":   1.0000,
  "6:5":   0.8333,
  "5:4":   0.8000,
  "4:3":   0.7500,
  "3:2":   0.6667,
  "8:5":   0.6250,
  "16:9":  0.5625,
  "2:1":   0.5000,
  "21:9":  0.4286,
  "32:9":  0.2813,
};

// Colors per level for nested display (innermost = tightest safe area)
const LEVEL_COLORS: Record<string, { r: number; g: number; b: number }> = {
  "32:9":  { r: 0.90, g: 0.20, b: 0.20 }, // red
  "21:9":  { r: 0.95, g: 0.55, b: 0.10 }, // orange
  "2:1":   { r: 0.95, g: 0.85, b: 0.10 }, // yellow
  "16:9":  { r: 0.20, g: 0.80, b: 0.30 }, // green
  "8:5":   { r: 0.20, g: 0.70, b: 0.90 }, // cyan
  "3:2":   { r: 0.30, g: 0.40, b: 0.95 }, // blue
  "4:3":   { r: 0.60, g: 0.30, b: 0.95 }, // purple
  "5:4":   { r: 0.90, g: 0.30, b: 0.80 }, // pink
  "6:5":   { r: 0.80, g: 0.80, b: 0.80 }, // light grey
  "1:1":   { r: 0.50, g: 0.50, b: 0.50 }, // grey
};

const USC_GUIDE_GROUP_NAME = "USC Safe Area Guide";
const OPACITY = 0.7;

figma.ui.onmessage = async (msg: {
  type: "apply" | "remove";
  minLevel: string;
  showNested: boolean;
}) => {
  const frame = figma.currentPage.selection[0];

  if (!frame || frame.type !== "FRAME") {
    figma.ui.postMessage({ type: "error", text: "Select a frame first." });
    return;
  }

  if (msg.type === "remove") {
    removeGuide(frame);
    return;
  }

  const { width, height } = frame;
  if (Math.abs(width - height) > 1) {
    figma.ui.postMessage({
      type: "error",
      text: `Frame is ${Math.round(width)}×${Math.round(height)}px. USC requires a square canvas.`,
    });
    return;
  }

  const size = width;
  removeGuide(frame); // clear any existing guide

  const levelsToShow = msg.showNested
    ? levelsAtOrTighter(msg.minLevel)
    : [msg.minLevel];

  const group = figma.group(
    levelsToShow.flatMap((level) => createOverlayForLevel(frame, size, level)),
    frame
  );
  group.name = USC_GUIDE_GROUP_NAME;
  group.locked = true;

  figma.ui.postMessage({ type: "success", text: `Guide applied (USC-${msg.minLevel}).` });
};

function removeGuide(frame: FrameNode) {
  frame.children
    .filter((n) => n.name === USC_GUIDE_GROUP_NAME)
    .forEach((n) => n.remove());
}

// Return all levels from widest (least safe) to the requested min level
function levelsAtOrTighter(minLevel: string): string[] {
  const order = ["32:9","21:9","2:1","16:9","8:5","3:2","4:3","5:4","6:5","1:1"];
  const idx = order.indexOf(minLevel);
  return idx === -1 ? [minLevel] : order.slice(0, idx + 1);
}

function createOverlayForLevel(
  frame: FrameNode,
  size: number,
  level: string
): RectangleNode[] {
  const fraction = USC_FRACTIONS[level] ?? 1;
  const safeSize = size * fraction;
  const offset = (size - safeSize) / 2;
  const color = LEVEL_COLORS[level] ?? { r: 1, g: 1, b: 0 };

  const nodes: RectangleNode[] = [];

  // Four dimming strips (unsafe regions)
  const strips: Array<[number, number, number, number]> = [
    [0,          0,          size,   offset],   // top
    [0,          offset + safeSize, size,   offset],   // bottom
    [0,          offset,     offset, safeSize], // left
    [offset + safeSize, offset, offset, safeSize], // right
  ];

  for (const [x, y, w, h] of strips) {
    const rect = figma.createRectangle();
    rect.x = x;
    rect.y = y;
    rect.resize(w, h);
    rect.fills = [{ type: "SOLID", color, opacity: 0.15 }];
    rect.strokes = [];
    rect.name = `USC-${level} unsafe region`;
    frame.appendChild(rect);
    nodes.push(rect);
  }

  // Safe area border
  const border = figma.createRectangle();
  border.x = offset;
  border.y = offset;
  border.resize(safeSize, safeSize);
  border.fills = [];
  border.strokes = [{ type: "SOLID", color, opacity: OPACITY }];
  border.strokeWeight = 1.5;
  border.dashPattern = [8, 4];
  border.name = `USC-${level} safe area border`;
  frame.appendChild(border);
  nodes.push(border);

  return nodes;
}
```

### Plugin UI Panel (ui.html)

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8" />
  <style>
    body { font-family: sans-serif; font-size: 13px; margin: 0; padding: 12px; }
    label { display: block; margin-bottom: 6px; }
    select, button { width: 100%; margin-bottom: 10px; padding: 6px; font-size: 13px; }
    .info { font-size: 11px; color: #666; margin-bottom: 10px; line-height: 1.5; }
    .status { margin-top: 8px; font-size: 12px; }
    .error { color: #c0392b; }
    .success { color: #27ae60; }
    .safe-area-bar {
      height: 8px;
      background: linear-gradient(to right, #e74c3c, #f39c12, #f1c40f, #2ecc71);
      border-radius: 4px;
      margin-bottom: 4px;
    }
    .safe-area-marker {
      height: 8px;
      width: 2px;
      background: #333;
      margin-top: -8px;
      position: relative;
    }
  </style>
</head>
<body>
  <h3 style="margin-top:0">USC Safe Area Guide</h3>

  <label>
    Minimum compliance level:
    <select id="level">
      <option value="32:9">USC-32:9 — Super-ultrawide (28%)</option>
      <option value="21:9">USC-21:9 — Ultrawide / modern phones (43%)</option>
      <option value="2:1">USC-2:1 — Most phones (50%)</option>
      <option value="16:9" selected>USC-16:9 — Most TVs &amp; monitors (56%) ★</option>
      <option value="8:5">USC-8:5 — MacBooks / laptops (63%)</option>
      <option value="3:2">USC-3:2 — Older phones (67%)</option>
      <option value="4:3">USC-4:3 — Tablets / older screens (75%)</option>
      <option value="5:4">USC-5:4 — Foldables (80%)</option>
      <option value="6:5">USC-6:5 — Foldables, wide (83%)</option>
      <option value="1:1">USC-1:1 — Square only (100%)</option>
    </select>
  </label>

  <label>
    <input type="checkbox" id="nested" checked />
    Show all levels from widest to selected (nested guide)
  </label>

  <p class="info" id="description"></p>

  <button id="apply">Apply to selected frame</button>
  <button id="remove" style="background:#f8f8f8">Remove guide</button>

  <div class="status" id="status"></div>

  <script>
    const FRACTIONS = {
      "1:1":   1.0000, "6:5":  0.8333, "5:4":  0.8000, "4:3":  0.7500,
      "3:2":   0.6667, "8:5":  0.6250, "16:9": 0.5625, "2:1":  0.5000,
      "21:9":  0.4286, "32:9": 0.2813,
    };

    const levelSelect = document.getElementById("level");
    const descEl = document.getElementById("description");

    function updateDesc() {
      const level = levelSelect.value;
      const frac = FRACTIONS[level];
      const pct = (frac * 100).toFixed(1);
      const safeEdgePct = ((1 - frac) / 2 * 100).toFixed(1);
      descEl.textContent =
        `Safe area: ${pct}% of canvas width/height. ` +
        `Critical content must stay ${safeEdgePct}% from each edge. ` +
        `Decorative content may extend to the canvas edge.`;
    }

    levelSelect.addEventListener("change", updateDesc);
    updateDesc();

    document.getElementById("apply").onclick = () => {
      parent.postMessage({
        pluginMessage: {
          type: "apply",
          minLevel: levelSelect.value,
          showNested: document.getElementById("nested").checked,
        }
      }, "*");
    };

    document.getElementById("remove").onclick = () => {
      parent.postMessage({ pluginMessage: { type: "remove" } }, "*");
    };

    window.onmessage = (e) => {
      const msg = e.data.pluginMessage;
      if (!msg) return;
      const statusEl = document.getElementById("status");
      statusEl.textContent = msg.text;
      statusEl.className = "status " + msg.type;
    };
  </script>
</body>
</html>
```

### Overlay Math Reference

For any level `W:H` applied to a square canvas of side `S`:

| Property | Formula | Example (USC-16:9, S=1000px) |
|---|---|---|
| Safe area side | `S × (H/W)` | `1000 × (9/16)` = 562.5 px |
| Offset from each edge | `S × (1 − H/W) / 2` | `1000 × (7/32)` = 218.75 px |
| Unsafe edge strip width | `S × (1 − H/W) / 2` | 218.75 px |
| Fraction of canvas that is safe | `(H/W)²` | `(9/16)²` ≈ 31.6% of total area |

> Note: the last row — the fraction of total *area* that is safe — is `(H/W)²` because both dimensions are scaled by `H/W`. This is useful for understanding how much of the canvas is available for critical content at a given level.

The plugin's `levelsAtOrTighter()` function lists compliance levels from widest (smallest safe area, e.g. `32:9`) inward to the selected minimum. The nested overlay lets designers immediately see which level each region corresponds to, matching the nested-ring style of the USC Safe Area Guide diagram in the spec.
