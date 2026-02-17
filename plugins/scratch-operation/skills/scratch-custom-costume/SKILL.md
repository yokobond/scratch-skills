---
name: scratch-custom-costume
description: Add custom SVG or PNG costumes to sprites programmatically via the Scratch VM storage API. Enables creating original characters, shapes, or artwork without using the paint editor UI.
---

# Scratch Custom Costume Injection

This skill provides a pattern for **programmatically adding custom costumes (SVG or PNG)** to Scratch sprites. Instead of manually drawing in the paint editor, you create image data and inject it through the VM's storage API.

## The Problem

When building Scratch projects programmatically, the default sprite (cat) costume is often unsuitable. The paint editor requires complex UI interactions (clicks, drags, color picks) that are fragile and slow to automate. There is no built-in block or simple API to add a custom-drawn costume from code.

## The Solution

Use the Scratch VM's **storage API** to create an image asset (SVG or PNG), wrap it in a costume object, and add it to the target sprite. This bypasses the paint editor entirely and works reliably in a single `page.evaluate()` call.

```
1. Create image data (SVG string or PNG binary)
2. Encode it as an asset via vm.runtime.storage.createAsset()
3. Build a costume descriptor object
4. Call vm.addCostume() to register it
5. Switch to the new costume with target.setCostume()
```

## Prerequisites

This skill builds on the **scratch-vm-injection** skill. Ensure:
- The Playwright MCP server is configured
- The Scratch editor is open and `window.vm` is available

## Implementation — SVG Costume

Use `browser_evaluate` or `browser_run_code` to inject an SVG costume:

```javascript
// browser_evaluate function:
async () => {
  const vm = window.vm;
  const target = vm.editingTarget;
  const storage = vm.runtime.storage;

  // Step 1: Define SVG markup
  const svgString = `
<svg xmlns="http://www.w3.org/2000/svg" width="100" height="100" viewBox="-50 -50 100 100">
  <circle cx="0" cy="0" r="40" fill="#4CAF50" stroke="#2E7D32" stroke-width="3"/>
  <circle cx="-12" cy="-10" r="5" fill="white"/>
  <circle cx="12" cy="-10" r="5" fill="white"/>
  <circle cx="-12" cy="-10" r="2.5" fill="#333"/>
  <circle cx="12" cy="-10" r="2.5" fill="#333"/>
  <path d="M -10,10 Q 0,20 10,10" fill="none" stroke="#333" stroke-width="2" stroke-linecap="round"/>
</svg>`;

  // Step 2: Create a storage asset
  const asset = storage.createAsset(
    storage.AssetType.ImageVector,  // SVG type
    storage.DataFormat.SVG,         // SVG format
    new TextEncoder().encode(svgString),  // SVG as UTF-8 bytes
    null,   // auto-generate asset ID (md5)
    true    // generate md5 from content
  );

  // Step 3: Build costume descriptor
  const costume = {
    name: 'my-costume',
    dataFormat: 'svg',
    asset: asset,
    md5: asset.assetId + '.svg',
    assetId: asset.assetId,
    bitmapResolution: 1,
    rotationCenterX: 50,   // center X in pixels (width / 2)
    rotationCenterY: 50    // center Y in pixels (height / 2)
  };

  // Step 4: Add costume and switch to it
  await vm.addCostume(costume.md5, costume);
  const costumeIndex = target.getCostumes().length - 1;
  target.setCostume(costumeIndex);

  return 'Costume added: ' + costume.name;
}
```

## Implementation — PNG Costume

### From a Local File

Use `browser_run_code` to read a local PNG file and inject it. The file is read via Playwright's `fs` access, converted to a base64 data URL, and passed into the page.

```javascript
// browser_run_code code:
async (page) => {
  const fs = require('fs');
  const pngPath = '/absolute/path/to/image.png';
  const pngBase64 = fs.readFileSync(pngPath).toString('base64');

  await page.evaluate(async (b64) => {
    const vm = window.vm;
    const target = vm.editingTarget;
    const storage = vm.runtime.storage;

    // Decode base64 to Uint8Array
    const binaryString = atob(b64);
    const bytes = new Uint8Array(binaryString.length);
    for (let i = 0; i < binaryString.length; i++) {
      bytes[i] = binaryString.charCodeAt(i);
    }

    const asset = storage.createAsset(
      storage.AssetType.ImageBitmap,
      storage.DataFormat.PNG,
      bytes,
      null,
      true
    );

    const costume = {
      name: 'png-costume',
      dataFormat: 'png',
      asset: asset,
      md5: asset.assetId + '.png',
      assetId: asset.assetId,
      bitmapResolution: 2,    // 2 for high-res PNGs (see Key Details)
      rotationCenterX: 0,     // set after measuring image
      rotationCenterY: 0
    };

    // Measure actual image dimensions to set rotation center
    const blob = new Blob([bytes], { type: 'image/png' });
    const url = URL.createObjectURL(blob);
    const img = new Image();
    await new Promise((resolve, reject) => {
      img.onload = resolve;
      img.onerror = reject;
      img.src = url;
    });
    URL.revokeObjectURL(url);

    // For bitmapResolution=2, rotation center is in image pixels (not halved)
    costume.rotationCenterX = Math.floor(img.width / 2);
    costume.rotationCenterY = Math.floor(img.height / 2);

    await vm.addCostume(costume.md5, costume);
    const costumeIndex = target.getCostumes().length - 1;
    target.setCostume(costumeIndex);

    return `PNG costume added: ${img.width}x${img.height}`;
  }, pngBase64);
}
```

### From a Canvas (Programmatic Drawing)

Generate a PNG entirely in the browser using an off-screen canvas:

```javascript
// browser_evaluate function:
async () => {
  const vm = window.vm;
  const target = vm.editingTarget;
  const storage = vm.runtime.storage;

  // Draw on an off-screen canvas
  const width = 200;
  const height = 200;
  const canvas = document.createElement('canvas');
  canvas.width = width;
  canvas.height = height;
  const ctx = canvas.getContext('2d');

  // Example: draw a star
  ctx.fillStyle = '#FFD700';
  ctx.beginPath();
  for (let i = 0; i < 5; i++) {
    const angle = (i * 4 * Math.PI) / 5 - Math.PI / 2;
    const x = 100 + 80 * Math.cos(angle);
    const y = 100 + 80 * Math.sin(angle);
    i === 0 ? ctx.moveTo(x, y) : ctx.lineTo(x, y);
  }
  ctx.closePath();
  ctx.fill();
  ctx.strokeStyle = '#B8860B';
  ctx.lineWidth = 3;
  ctx.stroke();

  // Convert canvas to PNG bytes
  const blob = await new Promise(r => canvas.toBlob(r, 'image/png'));
  const arrayBuffer = await blob.arrayBuffer();
  const bytes = new Uint8Array(arrayBuffer);

  const asset = storage.createAsset(
    storage.AssetType.ImageBitmap,
    storage.DataFormat.PNG,
    bytes,
    null,
    true
  );

  const costume = {
    name: 'star',
    dataFormat: 'png',
    asset: asset,
    md5: asset.assetId + '.png',
    assetId: asset.assetId,
    bitmapResolution: 2,
    rotationCenterX: width / 2,    // 100
    rotationCenterY: height / 2     // 100
  };

  await vm.addCostume(costume.md5, costume);
  const costumeIndex = target.getCostumes().length - 1;
  target.setCostume(costumeIndex);

  return 'Canvas PNG costume added';
}
```

## Common Operations

### Adding a Costume to a Specific Sprite

To add a costume to a sprite other than the currently selected one, find the target first:

```javascript
const target = vm.runtime.getSpriteTargetByName('Sprite2');
// or for the stage:
const stage = vm.runtime.getTargetForStage();

await vm.addCostume(costume.md5, costume, target.id);
//                                        ^^^^^^^^^ target ID
```

### Removing Default Costumes

After adding the custom costume, you can remove the default cat costumes:

```javascript
// Remove costumes by index (remove from end to avoid index shift)
target.deleteCostume(1);  // remove costume2
target.deleteCostume(0);  // remove costume1
```

### Multiple Costumes for Animation

Add multiple costumes to create frame-based animation:

```javascript
const frames = [
  { name: 'walk1', data: svgOrPngData1 },
  { name: 'walk2', data: svgOrPngData2 },
  { name: 'walk3', data: svgOrPngData3 }
];

for (const frame of frames) {
  const isSvg = typeof frame.data === 'string';
  const asset = storage.createAsset(
    isSvg ? storage.AssetType.ImageVector : storage.AssetType.ImageBitmap,
    isSvg ? storage.DataFormat.SVG : storage.DataFormat.PNG,
    isSvg ? new TextEncoder().encode(frame.data) : frame.data,
    null, true
  );
  const ext = isSvg ? 'svg' : 'png';
  await vm.addCostume(asset.assetId + '.' + ext, {
    name: frame.name,
    dataFormat: ext,
    asset: asset,
    md5: asset.assetId + '.' + ext,
    assetId: asset.assetId,
    bitmapResolution: isSvg ? 1 : 2,
    rotationCenterX: 50,
    rotationCenterY: 50
  });
}
```

Then use `looks_nextcostume` or `looks_switchcostumeto` blocks to animate.

## Key Details

### SVG vs PNG: When to Use Which

| | SVG | PNG |
|---|---|---|
| **Best for** | Simple shapes, icons, characters with flat colors | Photos, complex textures, pixel art, gradients |
| **Scaling** | Scales without quality loss | Fixed resolution, may blur when enlarged |
| **Creation** | Write markup as a string | Canvas drawing, file read, or fetch from URL |
| **AssetType** | `ImageVector` | `ImageBitmap` |
| **DataFormat** | `SVG` | `PNG` |
| **bitmapResolution** | `1` | `2` (standard for Scratch HD) |
| **Data encoding** | `new TextEncoder().encode(str)` | `Uint8Array` of raw PNG bytes |

### bitmapResolution

Scratch uses `bitmapResolution` to scale bitmap costumes on the stage:

- **`1`** — Used for SVG. 1 image pixel = 1 stage unit.
- **`2`** — Standard for PNG in Scratch. The image is displayed at **half its pixel dimensions** on stage (2x retina). A 200x200px PNG appears as 100x100 on stage.

Always use `bitmapResolution: 2` for PNG costumes to match Scratch's default behavior.

### Rotation Center

The `rotationCenterX` and `rotationCenterY` values define the point the sprite rotates around and where the sprite's position (0,0) maps to in the image.

**For SVG:**
```javascript
{
  rotationCenterX: 50,   // typically SVG width / 2
  rotationCenterY: 50    // typically SVG height / 2
}
```

**For PNG:**
```javascript
// With bitmapResolution: 2, use image pixel coordinates (not halved)
{
  rotationCenterX: Math.floor(img.width / 2),   // e.g. 100 for a 200px wide image
  rotationCenterY: Math.floor(img.height / 2)
}
```

### SVG Coordinate System

The SVG `viewBox` defines the coordinate space. For Scratch, center-origin viewBoxes work well:

```xml
<svg xmlns="http://www.w3.org/2000/svg"
     width="100" height="100"
     viewBox="-50 -50 100 100">
  <!-- (0,0) is now the center -->
</svg>
```

- `width` / `height`: the pixel dimensions of the costume
- `viewBox`: `"minX minY width height"` — use negative min values to center the origin
- Keep dimensions reasonable (50–200px). Scratch scales costumes to fit the sprite size setting.

### Storage API Parameters

```javascript
storage.createAsset(
  assetType,    // storage.AssetType.ImageVector (SVG) or ImageBitmap (PNG)
  dataFormat,   // storage.DataFormat.SVG, PNG, or JPG
  dataBuffer,   // Uint8Array of file content
  null,         // assetId: null to auto-generate
  true          // generateMd5: true to hash from content
);
```

## When to Use This Pattern

- **Custom characters**: Creating original sprites that don't exist in the Scratch library
- **Programmatic generation**: Generating costumes from data (charts, patterns, procedural art)
- **Multi-costume animation**: Adding multiple frames for smooth animation
- **Replacing default sprite**: Changing the cat to a project-specific character
- **External images**: Loading photos or downloaded images as costumes from local files

## When NOT to Use This Pattern

- **Library sprites**: If a suitable costume exists in the Scratch sprite library, use the **scratch-sprite-library** skill to add it from the UI instead
- **Interactive drawing**: If the user needs to draw or edit the costume themselves through the paint editor
