---
name: scratch-block-dragging
description: Builds Scratch programs by dragging and dropping blocks from the palette to the script area using Playwright mouse operations. Use this skill when you need to visually assemble Scratch code through the editor UI rather than injecting via the VM.
---

# Scratch Block Dragging Skill

This skill programs Scratch (scratch.mit.edu) by visually dragging blocks from the block palette to the script area using Playwright MCP mouse operations. Unlike the VM injection approach, this method mimics real user interaction with the Scratch editor UI.

## Block Geometry & Connection Points

All values below are in **workspace units**. Multiply by `workspace.scale` to get screen pixels.

### Key Constants

| Constant | Value | Meaning |
|----------|-------|---------|
| SNAP_RADIUS | 48 | Max distance (workspace units) for snap detection |
| NOTCH_START_PADDING | 12 | Notch X-offset from block's left edge |
| NOTCH_HEIGHT | 8 | Height of the notch/tab |
| MIN_BLOCK_Y | 48 | Standard block height |
| CORNER_RADIUS | 4 | Block corner rounding |
| SUBSTACK_OFFSET_X | ~24 | C-block inner bay X-offset |

### Connection Point Locations (screen pixels)

Given a block's bounding box `(left, top, width, height)` and `scale = workspace.scale`:

| Connection | Position | Used For |
|-----------|----------|----------|
| **Previous** (top of block) | `(left + 12*scale, top)` | Where this block receives a connection from above |
| **Next** (bottom of block) | `(left + 12*scale, top + height)` | Where the next block connects below |
| **Substack** (C-block inner) | `(left + 24*scale, top + 48*scale)` | Where blocks snap inside a C-shape (forever/if) |

**To connect block B below block A:** drop block B so that B's **previous** connection aligns with A's **next** connection. In practice, drop B at approximately `(A.left, A.top + A.height)`.

**To insert block B inside a C-block:** drop block B at the C-block's **substack** point: `(C.left + 24*scale, C.top + 48*scale)`.

## Prerequisites

This skill requires the **Playwright MCP server** to be installed and configured. Ensure the following is added to your MCP settings (e.g. `~/.claude/claude_desktop_config.json` or project `.mcp.json`):

```json
{
  "mcpServers": {
    "playwright": {
      "command": "npx",
      "args": ["@anthropic-ai/mcp-playwright@latest"]
    }
  }
}
```

If the browser is not installed, run `browser_install` first to download the required Chromium binary.

## When to Use

- When you want to visually demonstrate how to build a Scratch program
- When VM injection is not available or not desired
- When the user explicitly asks for drag-and-drop block assembly

## Workflow

### 1. Open Scratch Editor

```
browser_navigate url="https://scratch.mit.edu/projects/editor/"
```

Wait for the editor to fully load. Take a screenshot to confirm the editor is ready:

```
browser_take_screenshot
```

### 2. Detect Workspace Scale

Before dragging blocks, detect the workspace scale factor so connection-point calculations use correct screen coordinates.

```javascript
async (page) => {
  const scale = await page.evaluate(() => {
    // Method 1: Blockly API
    try {
      const ws = Blockly.getMainWorkspace();
      if (ws && ws.scale) {
        window.workspaceScale = ws.scale;
        return ws.scale;
      }
    } catch(e) {}
    // Method 2: Parse SVG transform
    try {
      const canvas = document.querySelector('.blocklyBlockCanvas');
      const transform = canvas?.getAttribute('transform') || '';
      const match = transform.match(/scale\(([\d.]+)/);
      if (match) {
        window.workspaceScale = parseFloat(match[1]);
        return window.workspaceScale;
      }
    } catch(e) {}
    // Fallback: default Scratch editor scale
    window.workspaceScale = 0.675;
    return 0.675;
  });
  return `Workspace scale: ${scale}`;
}
```

Store `window.workspaceScale` for use in subsequent connection-point calculations.

### 3. Language-Independent Element Selection

The Scratch editor supports dynamic language switching. All UI text (block labels, category names, button labels) changes with the locale. **Never use `text=...` locators, `aria-label`, or visible text for element selection.**

Instead, use **Blockly API** for block palette operations and **CSS class prefix matching** for GUI controls.

#### GUI Element Selectors

| Element | Selector |
|---|---|
| Green flag button | `img[class*="green-flag_green-flag"]` |
| Stop button | `img[class*="stop-all_stop-all"]` |
| Category item (by ID) | `.scratchCategoryId-<id>` (e.g., `.scratchCategoryId-motion`) |
| Selected category | `.scratchCategoryMenuItem.categorySelected` |
| Category menu container | `.scratchCategoryMenu` |

#### Block Category IDs

| Category ID | Description | Color |
|---|---|---|
| `motion` | Movement, rotation, coordinates | blue |
| `looks` | Costumes, speech, size, effects | purple |
| `sound` | Play sounds, volume | magenta |
| `events` | Flag clicked, key pressed, messages | yellow |
| `control` | Forever, if, wait, repeat, clones | orange |
| `sensing` | Touching, mouse position, ask, timer | cyan |
| `operators` | Math, random, string ops, logic | green |
| `data` | Variables, lists | orange-red |

#### Block Location via Blockly API

Always use block opcodes (e.g., `motion_movesteps`, `control_forever`) to identify blocks. These are language-independent.

- **Palette (flyout) blocks:** `ws.getFlyout().getWorkspace().getAllBlocks().find(b => b.type === '<opcode>')`
- **Script area blocks:** `ws.getAllBlocks().find(b => b.type === '<opcode>')`
- **Bounding box:** `block.getSvgRoot().getBoundingClientRect()`

### 4. Select a Block Category

Use the Blockly Toolbox API to select a category by its internal ID. This scrolls the flyout to show that category's blocks.

```javascript
async (page) => {
  await page.evaluate(() => {
    const ws = Blockly.getMainWorkspace();
    ws.getToolbox().setSelectedCategoryById('events');
  });
  return 'Selected Events category';
}
```

**Fallback** — click the category DOM element directly:

```javascript
async (page) => {
  await page.evaluate(() => {
    const cat = document.querySelector('.scratchCategoryId-events');
    if (cat) cat.click();
  });
  return 'Clicked Events category';
}
```

### 5. Locate the Block Position

Use `browser_run_code` with the Blockly API to find the exact bounding box of a block by its opcode:

```javascript
async (page) => {
  const box = await page.evaluate(() => {
    const ws = Blockly.getMainWorkspace();
    const flyout = ws.getFlyout().getWorkspace();
    const block = flyout.getAllBlocks().find(b => b.type === '<opcode>');
    if (!block) throw new Error('Block not found in flyout: <opcode>');
    const rect = block.getSvgRoot().getBoundingClientRect();
    return { x: rect.x, y: rect.y, width: rect.width, height: rect.height };
  });
  return JSON.stringify(box);
}
```

For example, to find the "point towards" block:
```javascript
async (page) => {
  const box = await page.evaluate(() => {
    const ws = Blockly.getMainWorkspace();
    const flyout = ws.getFlyout().getWorkspace();
    const block = flyout.getAllBlocks().find(b => b.type === 'motion_pointtowards');
    const rect = block.getSvgRoot().getBoundingClientRect();
    return { x: rect.x, y: rect.y, width: rect.width, height: rect.height };
  });
  return JSON.stringify(box);
}
```

**Important:** Use `ws.getAllBlocks()` for blocks already in the script area, and `ws.getFlyout().getWorkspace().getAllBlocks()` for blocks in the palette.

### 6. Drag a Block to the Script Area

Use `browser_run_code` to perform a mouse drag operation. The drag must be done in small incremental steps for Scratch to properly register it.

#### Precision Connection-Point Targeting (Recommended)

To actually snap blocks together, calculate the drop position from the target block's connection points. This is the **primary method** — approximate positioning rarely results in real connections.

**CRITICAL: Grab blocks near their top-left corner.** Scratch preserves the offset between the grab point and the block's origin during drag. If you grab from the block's center, the block's previous connection (top-left) will be offset from where you drop. Grabbing near the top-left (~15px in from each edge) ensures the previous connection aligns with the drop point.

**Use Blockly API for reliable positioning.** Text locators can match multiple elements. Use `Blockly.getMainWorkspace().getAllBlocks()` for script area blocks and `ws.getFlyout().getWorkspace().getAllBlocks()` for palette blocks, then `block.getSvgRoot().getBoundingClientRect()` for exact screen positions.

**Pattern: Connect block B below block A**

```javascript
async (page) => {
  // 1. Get source block position from flyout (palette) via Blockly API
  const src = await page.evaluate(() => {
    const ws = Blockly.getMainWorkspace();
    const flyout = ws.getFlyout().getWorkspace();
    const block = flyout.getAllBlocks().find(b => b.type === '<source_block_type>');
    const rect = block.getSvgRoot().getBoundingClientRect();
    return { x: rect.x, y: rect.y, w: rect.width, h: rect.height };
  });
  // Grab near top-left corner (aligns previous connection with drop point)
  const startX = src.x + 15;
  const startY = src.y + 15;

  // 2. Get target block position (already in script area) via Blockly API
  const tgt = await page.evaluate(() => {
    const ws = Blockly.getMainWorkspace();
    const block = ws.getAllBlocks().find(b => b.type === '<target_block_type>');
    const rect = block.getSvgRoot().getBoundingClientRect();
    return { x: rect.x, y: rect.y, w: rect.width, h: rect.height, bottom: rect.bottom };
  });

  // 3. Calculate drop point: target block's bottom edge + small nudge
  const endX = tgt.x;
  const endY = tgt.bottom + 4;  // +4px nudge for notch alignment

  // 4. Perform the drag
  await page.mouse.move(startX, startY);
  await page.waitForTimeout(300);
  await page.mouse.down();
  await page.waitForTimeout(200);
  for (let i = 1; i <= 25; i++) {
    const x = startX + (endX - startX) * i / 25;
    const y = startY + (endY - startY) * i / 25;
    await page.mouse.move(x, y);
    await page.waitForTimeout(40);
  }
  await page.waitForTimeout(500);
  await page.mouse.up();
  await page.waitForTimeout(500);
  return `Dragged to (${endX}, ${endY})`;
}
```

**Pattern: Insert block inside a C-block (forever/if)**

```javascript
async (page) => {
  // 1. Get source block from flyout via Blockly API
  const src = await page.evaluate(() => {
    const ws = Blockly.getMainWorkspace();
    const flyout = ws.getFlyout().getWorkspace();
    const block = flyout.getAllBlocks().find(b => b.type === '<source_block_type>');
    const rect = block.getSvgRoot().getBoundingClientRect();
    return { x: rect.x, y: rect.y, w: rect.width, h: rect.height };
  });
  // Grab near top-left corner
  const startX = src.x + 15;
  const startY = src.y + 15;

  // 2. Get C-block's SUBSTACK connection screen position via SVG coordinate transform
  const substack = await page.evaluate(() => {
    const ws = Blockly.getMainWorkspace();
    const cBlock = ws.getAllBlocks().find(b => b.type === '<c_block_type>');
    const input = cBlock.getInput('SUBSTACK');
    const conn = input.connection;
    const svg = ws.getParentSvg();
    const pt = svg.createSVGPoint();
    pt.x = conn.x_;
    pt.y = conn.y_;
    const canvas = cBlock.getSvgRoot().closest('.blocklyBlockCanvas');
    const screenPt = pt.matrixTransform(canvas.getScreenCTM());
    return { x: screenPt.x, y: screenPt.y };
  });

  const endX = substack.x;
  const endY = substack.y;

  await page.mouse.move(startX, startY);
  await page.waitForTimeout(300);
  await page.mouse.down();
  await page.waitForTimeout(200);
  for (let i = 1; i <= 25; i++) {
    const x = startX + (endX - startX) * i / 25;
    const y = startY + (endY - startY) * i / 25;
    await page.mouse.move(x, y);
    await page.waitForTimeout(40);
  }
  await page.waitForTimeout(500);
  await page.mouse.up();
  await page.waitForTimeout(500);
  return `Dragged into C-block at (${endX}, ${endY})`;
}
```

#### Approximate Visual Positioning (Fallback)

Use this only for the **first hat block** (no connection target exists yet) or when precision targeting fails.

```javascript
async (page) => {
  const startX = <block_center_x>;
  const startY = <block_center_y>;
  const endX = <target_x>;
  const endY = <target_y>;

  await page.mouse.move(startX, startY);
  await page.waitForTimeout(300);
  await page.mouse.down();
  await page.waitForTimeout(200);
  for (let i = 1; i <= 25; i++) {
    const x = startX + (endX - startX) * i / 25;
    const y = startY + (endY - startY) * i / 25;
    await page.mouse.move(x, y);
    await page.waitForTimeout(40);
  }
  await page.waitForTimeout(300);
  await page.mouse.up();
  await page.waitForTimeout(500);
  return 'Block dragged';
}
```

**Critical drag parameters (both methods):**
- **25 steps** with **40ms intervals** for smooth drag that Scratch recognizes
- **300ms pause** before `mousedown` and before `mouseup` for Scratch to register start/end
- **200ms pause** after `mousedown` before moving

### 7. Script Area Drop Targets

The script area occupies the center-right portion of the editor (approximately x: 350-680, y: 100-800 in default layout).

**Drop position strategy:**
- **First block (hat block):** Approximate — drop at ~(500, 250) to leave room below
- **Connect below a block:** Precision target — get target block's bounding box via Blockly API, grab source near top-left, drop at `(target.x, target.bottom + 4)`
- **Inside a C-shaped block (forever/if):** Precision target — get SUBSTACK connection screen position via SVG `getScreenCTM()`, grab source near top-left, drop at exact substack point

Scratch snaps blocks when dropped within SNAP_RADIUS (48 workspace units ≈ 32px at default scale) of a valid connection point. A **white highlight line** appears when snap is available — if you see it during drag, the connection will succeed.

### 8. Verify After Each Drag

Always take a screenshot after each drag to verify the block was placed correctly:

```
browser_take_screenshot
```

If a wrong block was dragged, undo with:
```
browser_press_key key="Control+z"
```

### 9. Run the Program

Click the green flag button to run the program:

```javascript
async (page) => {
  await page.evaluate(() => {
    const flag = document.querySelector('img[class*="green-flag_green-flag"]');
    if (flag) flag.click();
  });
  return 'Clicked green flag';
}
```

## Complete Example: Cat Follows Mouse

Build a program where the cat walks toward the mouse pointer:

```
event_whenflagclicked
  control_forever
    motion_pointtowards (towards: _mouse_)
    motion_movesteps (steps: 10)
    looks_nextcostume
```

### Step-by-step:

1. **Open editor**, detect workspace scale, and take screenshot to see the layout.

2. **Select Events category:**
   ```javascript
   async (page) => {
     await page.evaluate(() => {
       Blockly.getMainWorkspace().getToolbox().setSelectedCategoryById('events');
     });
     return 'Selected Events category';
   }
   ```

3. **Locate `event_whenflagclicked`** block in the flyout:
   ```javascript
   async (page) => {
     const box = await page.evaluate(() => {
       const ws = Blockly.getMainWorkspace();
       const flyout = ws.getFlyout().getWorkspace();
       const block = flyout.getAllBlocks().find(b => b.type === 'event_whenflagclicked');
       const rect = block.getSvgRoot().getBoundingClientRect();
       return { x: rect.x, y: rect.y, width: rect.width, height: rect.height };
     });
     return JSON.stringify(box);
   }
   ```

4. **Drag flag block to script area** at ~(500, 250) — approximate positioning (first block, no target to snap to).

5. **Select Control category**, locate `control_forever` block. **Precision target** the flag block's bottom edge:
   - Get flag block via `ws.getAllBlocks().find(b => b.type === 'event_whenflagclicked')`
   - Drop at `(flagRect.x, flagRect.bottom + 4)`

6. **Select Motion category**, locate `motion_pointtowards` block. **Precision target** the forever block's substack:
   - Get forever block via `ws.getAllBlocks().find(b => b.type === 'control_forever')`
   - Use SUBSTACK connection point via SVG `getScreenCTM()` to insert inside the C-block

7. **Locate `motion_movesteps`** block in flyout. **Precision target** the `motion_pointtowards` block's bottom edge:
   - Get the point-towards block via `ws.getAllBlocks().find(b => b.type === 'motion_pointtowards')`
   - Drop at `(pointRect.x, pointRect.bottom + 4)`

8. **Select Looks category**, locate `looks_nextcostume` block. **Precision target** the `motion_movesteps` block's bottom edge:
   - Get the move block via `ws.getAllBlocks().find(b => b.type === 'motion_movesteps')`
   - Drop at `(moveRect.x, moveRect.bottom + 4)`

9. **Verify via VM** — check that all blocks are connected (no `topLevel: true` except the flag block).

10. **Take screenshot** to verify the completed program visually.

11. **Click the green flag** via `img[class*="green-flag_green-flag"]`.

## Verifying Block Connections via VM (Safety Net)

**CRITICAL:** Even with precision targeting, always verify connections. Visually adjacent blocks may NOT be actually connected — Scratch's snap detection is strict. Precision targeting should handle most cases, but always confirm via the VM and use the fix below if needed.

Use `browser_run_code` to check the block structure through the Scratch VM:

```javascript
async (page) => {
  const result = await page.evaluate(() => {
    // Find the Scratch VM
    const root = document.body;
    const queue = [root];
    const visited = new Set();
    let vm = null;
    while (queue.length > 0) {
      const node = queue.shift();
      if (visited.has(node)) continue;
      visited.add(node);
      const keys = Object.keys(node);
      const reactKey = keys.find(
        key => key.startsWith('__reactInternalInstance$') || key.startsWith('__reactFiber$')
      );
      if (reactKey) {
        let fiber = node[reactKey];
        while (fiber) {
          if (fiber.memoizedProps && fiber.memoizedProps.vm) {
            vm = fiber.memoizedProps.vm;
            break;
          }
          fiber = fiber.return;
        }
      }
      if (node.children) {
        for (let i = 0; i < node.children.length; i++) queue.push(node.children[i]);
      }
      if (vm) break;
    }
    if (!vm) return 'VM not found';
    window.vm = vm;
    const target = vm.runtime.targets.find(t => !t.isStage);
    const blocks = target.blocks._blocks;
    const summary = {};
    for (const [id, block] of Object.entries(blocks)) {
      summary[id] = {
        opcode: block.opcode,
        next: block.next,
        parent: block.parent,
        topLevel: block.topLevel
      };
    }
    return JSON.stringify(summary, null, 2);
  });
  return result;
}
```

**What to check:**
- The `event_whenflagclicked` block should have `next` pointing to the next block (e.g., `control_forever`)
- `control_forever` should have `parent` pointing to the flag block, NOT `null`
- Blocks inside a C-shape (forever/if) should appear in the `SUBSTACK` input of the parent
- If any block has `topLevel: true` and `parent: null` unexpectedly, it is **disconnected**

### Fixing Disconnected Blocks via VM (Safety Net)

If blocks are not connected after precision-targeted dragging, fix them programmatically using the VM's loadProject pattern:

```javascript
async (page) => {
  await page.evaluate(async () => {
    const vm = window.vm;
    const projectJSON = JSON.parse(vm.toJSON());
    const sprite = projectJSON.targets.find(t => !t.isStage);

    sprite.blocks = {
      'flag_clicked': {
        opcode: 'event_whenflagclicked',
        next: 'forever', parent: null,
        inputs: {}, fields: {},
        shadow: false, topLevel: true, x: 100, y: 100
      },
      'forever': {
        opcode: 'control_forever',
        next: null, parent: 'flag_clicked',
        inputs: { SUBSTACK: [2, 'first_inner_block'] },
        fields: {}, shadow: false, topLevel: false
      },
      // ... inner blocks with proper parent/next chains
    };

    await vm.loadProject(JSON.stringify(projectJSON));
  });
  return 'Blocks reconnected';
}
```

## Tips & Troubleshooting

### Wrong Block Grabbed
The palette shows all blocks from multiple categories in a scrollable list. If you grab the wrong block, immediately undo with `Control+z` and retry. Always use `flyout.getAllBlocks().find(b => b.type === '<opcode>')` and `getSvgRoot().getBoundingClientRect()` for exact positions rather than guessing.

### Block Not Connecting (Most Common Issue)
**Use precision connection-point targeting** (see Step 6) as your primary strategy. Approximate pixel coordinates almost never result in actual connections.

**Most critical rule: Grab blocks near their top-left corner** (`block.x + 15, block.y + 15`). Scratch preserves the mouse-to-block offset during drag. Grabbing from the center offsets the block's previous connection ~50px from the drop point, causing snap detection to fail — especially for C-block substack insertion.

If precision targeting still doesn't snap:
- **Verify you're grabbing near top-left** — this is the #1 cause of failed connections
- **Use Blockly API** (`getSvgRoot().getBoundingClientRect()`) instead of text locators for block positions
- **Verify workspace scale** — re-run the scale detection step; an incorrect scale throws off all calculations
- **Check browser zoom is 100%** — browser zoom multiplies all coordinates and breaks alignment
- **Try adjusting the Y offset** — instead of `+4`, try `+2` or `+6` for the nudge value
- **Always verify via VM** after placing blocks, and use the VM fix as a safety net

### Scale Detection Failed
If `window.workspaceScale` returns `undefined` or the Blockly API isn't available:
- Re-run the detection snippet after the editor is fully loaded
- Try the SVG transform fallback method
- Use the default value `0.675` (standard Scratch editor scale)

### White Line Appears But Block Doesn't Snap
If you see the white highlight line during drag but the block doesn't connect on drop:
- Ensure `mouseup` happens **while the white line is visible** — add a longer pause (500ms) before releasing
- The highlight may disappear if you overshoot; try holding the final position longer before releasing
- Slow down the final few drag steps (increase interval from 40ms to 80ms for the last 5 steps)

### Palette Scrolling
Clicking a category button scrolls the palette to that category. After clicking, take a screenshot or locate elements again as positions may have changed.

### Multiple Blocks of Same Type
Some blocks share visual elements (e.g., number fields). Always use opcodes to identify blocks. For instance, use `motion_movesteps` to find the move block rather than searching by visible text.

### Coordinate Discovery Pattern
Always follow this pattern for reliable block placement:
1. Select the category via `setSelectedCategoryById('<id>')`
2. Use `flyout.getAllBlocks().find(b => b.type === '<opcode>').getSvgRoot().getBoundingClientRect()` to find the exact position
3. **Calculate the drop point** from the target block's connection points (precision targeting)
4. Drag from the found position to the calculated target
5. Take a screenshot to verify
6. **Check VM block structure** to confirm connections are real
