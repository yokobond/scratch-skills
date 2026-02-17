---
name: scratch-custom-costume
description: Add custom SVG costumes to sprites programmatically via the Scratch VM storage API. Enables creating original characters, shapes, or artwork without using the paint editor UI.
---

# Scratch Custom Costume Injection

This skill provides a pattern for **programmatically adding custom SVG costumes** to Scratch sprites. Instead of manually drawing in the paint editor, you create SVG markup and inject it through the VM's storage API.

## The Problem

When building Scratch projects programmatically, the default sprite (cat) costume is often unsuitable. The paint editor requires complex UI interactions (clicks, drags, color picks) that are fragile and slow to automate. There is no built-in block or simple API to add a custom-drawn costume from code.

## The Solution

Use the Scratch VM's **storage API** to create an SVG asset, wrap it in a costume object, and add it to the target sprite. This bypasses the paint editor entirely and works reliably in a single `page.evaluate()` call.

```
1. Create SVG markup as a string
2. Encode it as an asset via vm.runtime.storage.createAsset()
3. Build a costume descriptor object
4. Call vm.addCostume() to register it
5. Switch to the new costume with target.setCostume()
```

## Prerequisites

This skill builds on the **scratch-vm-injection** skill. Ensure:
- The Playwright MCP server is configured
- The Scratch editor is open and `window.vm` is available

## Implementation

### Full Example

Use `browser_evaluate` or `browser_run_code` to inject a costume:

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

## Key Details

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

### Rotation Center

The `rotationCenterX` and `rotationCenterY` values define the point the sprite rotates around and where coordinates (0,0) maps to on the stage.

```javascript
{
  rotationCenterX: 50,   // typically width / 2
  rotationCenterY: 50    // typically height / 2
}
```

For a `viewBox="-50 -50 100 100"` with `width="100" height="100"`, set both to `50` so the SVG origin `(0,0)` aligns with the sprite's position on stage.

### Storage API Parameters

```javascript
storage.createAsset(
  storage.AssetType.ImageVector,  // type: ImageVector for SVG, ImageBitmap for PNG
  storage.DataFormat.SVG,         // format: SVG, PNG, or JPG
  dataBuffer,                     // Uint8Array of file content
  null,                           // assetId: null to auto-generate
  true                            // generateMd5: true to hash from content
);
```

### Multiple Costumes for Animation

Add multiple costumes to create frame-based animation:

```javascript
const frames = [
  { name: 'walk1', svg: '<svg>...</svg>' },
  { name: 'walk2', svg: '<svg>...</svg>' },
  { name: 'walk3', svg: '<svg>...</svg>' }
];

for (const frame of frames) {
  const asset = storage.createAsset(
    storage.AssetType.ImageVector,
    storage.DataFormat.SVG,
    new TextEncoder().encode(frame.svg),
    null, true
  );
  await vm.addCostume(asset.assetId + '.svg', {
    name: frame.name,
    dataFormat: 'svg',
    asset: asset,
    md5: asset.assetId + '.svg',
    assetId: asset.assetId,
    bitmapResolution: 1,
    rotationCenterX: 50,
    rotationCenterY: 50
  });
}
```

Then use `looks_nextcostume` or `looks_switchcostumeto` blocks to animate.

## When to Use This Pattern

- **Custom characters**: Creating original sprites that don't exist in the Scratch library
- **Programmatic generation**: Generating costumes from data (charts, patterns, procedural art)
- **Multi-costume animation**: Adding multiple frames for smooth animation
- **Replacing default sprite**: Changing the cat to a project-specific character

## When NOT to Use This Pattern

- **Library sprites**: If a suitable costume exists in the Scratch sprite library, use the **scratch-sprite-library** skill to add it from the UI instead
- **Bitmap images**: For photographic or complex raster images, use `AssetType.ImageBitmap` with `DataFormat.PNG` and a base64-decoded PNG buffer instead of SVG
- **Interactive drawing**: If the user needs to draw or edit the costume themselves through the paint editor
