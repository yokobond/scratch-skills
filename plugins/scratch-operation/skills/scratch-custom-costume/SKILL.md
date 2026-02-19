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

**⚠️ `deleteCostume` cannot delete the last remaining costume.** If only one costume is left in the list, `deleteCostume(0)` silently fails. Always ensure at least 2 costumes exist before deleting. The safest approach is to add new costumes first, then delete the old ones (see "Correct Pattern" below).

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

## Costume List Management for Animation

### The Core Rule

`looks_nextcostume` cycles through **every** costume in the sprite's list in order. If any unwanted costume remains in the list — the default cat, a "?" placeholder, a dummy entry, or a leftover frame — it will appear during animation.

**The fix is always the same: add the new frames first, then delete all the old costumes.**

### Correct Pattern

Perform `loadProject` and costume injection inside a **single `page.evaluate` call** so the costume list is clean before any rendering occurs.

**Key rule: add new costumes first, then delete old ones.** `deleteCostume` silently fails when only one costume remains in the list. Adding new costumes first guarantees the list always has more than one entry during deletion.

```javascript
// browser_run_code code:
async (page) => {
  const svg1 = `<svg ...>frame 1</svg>`;
  const svg2 = `<svg ...>frame 2</svg>`;

  await page.evaluate(async (frames) => {
    const vm = window.vm;

    // Step 1: inject blocks via loadProject
    const projectJSON = JSON.parse(vm.toJSON());
    const sprite = projectJSON.targets.find(t => !t.isStage);
    sprite.blocks = { /* looks_nextcostume loop ... */ };
    await vm.loadProject(JSON.stringify(projectJSON));

    // Step 2: immediately replace the costume list
    const target = vm.editingTarget;
    const storage = vm.runtime.storage;

    // Record how many OLD costumes exist before adding new ones
    const oldCount = target.getCostumes().length;

    // Add new animation frames FIRST (appended to the end of the list)
    for (const { svg, name } of frames) {
      const asset = storage.createAsset(
        storage.AssetType.ImageVector, storage.DataFormat.SVG,
        new TextEncoder().encode(svg), null, true
      );
      await vm.addCostume(asset.assetId + '.svg', {
        name, dataFormat: 'svg', asset,
        md5: asset.assetId + '.svg', assetId: asset.assetId,
        bitmapResolution: 1, rotationCenterX: 50, rotationCenterY: 50
      });
    }

    // THEN delete the old costumes from the front (new ones are safe at the end)
    for (let i = oldCount - 1; i >= 0; i--) target.deleteCostume(i);

    target.setCostume(0);
  }, [{ svg: svg1, name: 'frame1' }, { svg: svg2, name: 'frame2' }]);

  window.vm.greenFlag();
}
```

### Anti-Patterns

**❌ Putting a dummy/placeholder costume in the project JSON before `loadProject`:**
```javascript
// Wrong — "blank" stays in the costume list and appears during animation
sprite.costumes = [{ name: 'blank', assetId: '00000000000000000000000000000000', ... }];
```

**❌ Two separate `page.evaluate` calls (loadProject then costume injection):**
```javascript
// Risky — there may be a render gap between the two calls
await page.evaluate(async () => { await vm.loadProject(...); });
await page.evaluate(async () => { /* re-inject costumes */ });  // cat/? may flash here
```

**❌ Forgetting to delete default costumes:**
```javascript
// Wrong — cat costumes remain; cycle becomes: frame1 → frame2 → cat1 → cat2 → ...
await vm.addCostume(...);  // added frames, but never deleted the cat
```

**❌ Checking the count before deciding whether to delete:**
After `loadProject` the costume list may contain original cat costumes, previously injected custom costumes, or broken "?" entries from failed CDN fetches. Always delete **all** costumes unconditionally before adding your frames.

**❌ Deleting all costumes before adding new ones (delete loop leaves last costume):**
```javascript
// Wrong — deleteCostume silently fails when only 1 costume remains
const count = target.getCostumes().length;
for (let i = count - 1; i >= 0; i--) target.deleteCostume(i);  // last one survives!
await vm.addCostume(...frame1...);
await vm.addCostume(...frame2...);
// Result: [costume1(cat), frame1, frame2] — cat is still there
```
The fix: **add new costumes first, then delete old ones** (see Correct Pattern above).

## Using Custom Costumes with vm.loadProject()

### The Problem

`vm.loadProject()` (used by the **scratch-vm-injection** skill's `updateSprite` helper) **destroys all locally-injected costumes**. When the project is reloaded, Scratch tries to fetch every asset by its md5 hash from the remote CDN (`api.scratch.mit.edu`). Locally created assets have never been uploaded there, so the fetch fails and the costume appears as a "?" placeholder.

This affects both SVG and PNG costumes equally.

### The Solution: Re-inject After loadProject (in One `page.evaluate`)

Save your costume data before calling `loadProject`, then re-inject the costumes **in the same `page.evaluate` call** immediately after. Splitting into two separate `page.evaluate` calls risks a render gap where the old (cat/"?") costume briefly appears.

**For SVG** — data is a plain string, passed as an argument to `page.evaluate`:

```javascript
// browser_run_code code:
async (page) => {
  const svg1 = `<svg ...>...</svg>`;  // frame 1
  const svg2 = `<svg ...>...</svg>`;  // frame 2

  // Inject blocks AND re-inject costumes in one evaluate (no render gap)
  await page.evaluate(async (svgs) => {
    const vm = window.vm;
    const projectJSON = JSON.parse(vm.toJSON());
    const sprite = projectJSON.targets.find(t => !t.isStage);
    sprite.currentCostume = 0;
    sprite.blocks = { /* ... your blocks ... */ };
    await vm.loadProject(JSON.stringify(projectJSON));
    // NOTE: custom costumes are now broken/missing — fix immediately below

    const target = vm.editingTarget;
    const storage = vm.runtime.storage;

    // Record old costume count BEFORE adding new ones
    const oldCount = target.getCostumes().length;

    // Add new costumes FIRST (so the list always has >1 entry during deletion)
    for (const { svg, name } of svgs) {
      const asset = storage.createAsset(
        storage.AssetType.ImageVector,
        storage.DataFormat.SVG,
        new TextEncoder().encode(svg),
        null, true
      );
      await vm.addCostume(asset.assetId + '.svg', {
        name, dataFormat: 'svg', asset,
        md5: asset.assetId + '.svg', assetId: asset.assetId,
        bitmapResolution: 1, rotationCenterX: 50, rotationCenterY: 50
      });
    }

    // THEN delete old costumes (cat, "?", or any leftovers) from the front
    for (let i = oldCount - 1; i >= 0; i--) target.deleteCostume(i);

    target.setCostume(0);
  }, [{ svg: svg1, name: 'frame1' }, { svg: svg2, name: 'frame2' }]);
}
```

**For PNG** — `Uint8Array` is not JSON-serializable, so convert to base64 before passing to `page.evaluate`, then decode inside:

```javascript
// browser_run_code code:
async (page) => {
  const fs = require('fs');

  // Read PNG files and encode as base64 (JSON-serializable)
  const png1b64 = fs.readFileSync('/path/to/frame1.png').toString('base64');
  const png2b64 = fs.readFileSync('/path/to/frame2.png').toString('base64');

  // Step 1: Inject blocks via loadProject
  await page.evaluate(async () => {
    const vm = window.vm;
    const projectJSON = JSON.parse(vm.toJSON());
    const sprite = projectJSON.targets.find(t => !t.isStage);
    sprite.currentCostume = 0;
    sprite.blocks = { /* ... your blocks ... */ };
    await vm.loadProject(JSON.stringify(projectJSON));
  });

  // Step 2: Re-inject PNG costumes after loadProject
  await page.evaluate(async (pngs) => {
    const vm = window.vm;
    const target = vm.editingTarget;
    const storage = vm.runtime.storage;

    // Helper: decode base64 string to Uint8Array
    const b64ToBytes = (b64) => {
      const bin = atob(b64);
      const bytes = new Uint8Array(bin.length);
      for (let i = 0; i < bin.length; i++) bytes[i] = bin.charCodeAt(i);
      return bytes;
    };

    // Record old costume count
    const oldCount = target.getCostumes().length;

    for (const { b64, name } of pngs) {
      const bytes = b64ToBytes(b64);

      // Measure image dimensions for accurate rotation center
      const blob = new Blob([bytes], { type: 'image/png' });
      const url = URL.createObjectURL(blob);
      const img = new Image();
      await new Promise((res, rej) => { img.onload = res; img.onerror = rej; img.src = url; });
      URL.revokeObjectURL(url);

      const asset = storage.createAsset(
        storage.AssetType.ImageBitmap,
        storage.DataFormat.PNG,
        bytes,
        null, true
      );
      await vm.addCostume(asset.assetId + '.png', {
        name, dataFormat: 'png', asset,
        md5: asset.assetId + '.png', assetId: asset.assetId,
        bitmapResolution: 2,
        rotationCenterX: Math.floor(img.width / 2),
        rotationCenterY: Math.floor(img.height / 2)
      });
    }

    // Remove old broken placeholder costumes
    for (let i = oldCount - 1; i >= 0; i--) target.deleteCostume(i);
    
    target.setCostume(0);
  }, [{ b64: png1b64, name: 'frame1' }, { b64: png2b64, name: 'frame2' }]);
}
```

### Why This Works

`storage.createAsset()` stores the asset data in the VM's in-memory storage. This survives as long as the page is alive. After `loadProject` completes, calling `addCostume` again re-registers the asset in the new runtime's storage, making it available for rendering. This works identically for SVG and PNG.

### Key Difference When Passing PNG Data to page.evaluate

`Uint8Array` is not JSON-serializable and **cannot** be passed directly as an argument to `page.evaluate`. Always convert PNG bytes to a base64 string first (`Buffer.toString('base64')` in Node.js, or `btoa()` in the browser), then decode back to `Uint8Array` inside the evaluate callback using `atob()`.

### Alternative: Inject Blocks Without loadProject

If you only need to add a few blocks and the sprite already has the right costumes, consider manipulating `target.blocks` directly or using the Blockly workspace API instead of `loadProject`. This avoids the asset loss entirely — though it is more complex for large scripts.

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
