---
name: scratch-block-dragging
description: Builds Scratch programs by dragging and dropping blocks from the palette to the script area using Playwright mouse operations. Use this skill when you need to visually assemble Scratch code through the editor UI rather than injecting via the VM.
---

# Scratch Block Dragging Skill

This skill programs Scratch (scratch.mit.edu) by visually dragging blocks from the block palette to the script area using Playwright MCP mouse operations. Unlike the VM injection approach, this method mimics real user interaction with the Scratch editor UI.

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

### 2. Select a Block Category

Click on a category button in the left sidebar to display the blocks you need. Use `browser_click` with the category's ref from the page snapshot.

Categories available in the sidebar:
- **動き** (Motion) - blue: movement, rotation, coordinates
- **見た目** (Looks) - purple: costumes, speech, size, effects
- **音** (Sound) - magenta: play sounds, volume
- **イベント** (Events) - yellow: flag clicked, key pressed, messages
- **制御** (Control) - orange: forever, if, wait, repeat, clones
- **調べる** (Sensing) - cyan: touching, mouse position, ask, timer
- **演算** (Operators) - green: math, random, string ops, logic
- **変数** (Variables) - orange-red: variables, lists

Example:
```
browser_click ref="<category_ref>" element="イベントカテゴリ"
```

### 3. Locate the Block Position

Use `browser_run_code` to find the exact bounding box of a block by searching for its text label:

```javascript
async (page) => {
  const el = await page.locator('text=<block_text>').first();
  const box = await el.boundingBox();
  return JSON.stringify(box);
}
```

For example, to find "マウスのポインターへ向ける":
```javascript
async (page) => {
  const el = await page.locator('text=マウスのポインター').first();
  const box = await el.boundingBox();
  return JSON.stringify(box);
}
```

**Important:** The text locator returns the first match. If the same text appears in multiple places (palette and script area), use `.nth(0)` for the palette copy.

### 4. Drag a Block to the Script Area

Use `browser_run_code` to perform a mouse drag operation. The drag must be done in small incremental steps for Scratch to properly register it.

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

**Critical parameters:**
- **25 steps** with **40ms intervals** for smooth drag that Scratch recognizes
- **300ms pause** before `mousedown` and before `mouseup` for Scratch to register start/end
- **200ms pause** after `mousedown` before moving

### 5. Script Area Drop Targets

The script area occupies the center-right portion of the editor (approximately x: 350-680, y: 100-800 in default layout).

**Drop position guidelines:**
- **First block (hat block):** Drop at approximately (500, 300) for a good starting position
- **Connect below a block:** Drop at the same x as the existing block, y + 35 below its bottom edge
- **Inside a C-shaped block (forever/if):** Drop at the block's x + 20, between the top and bottom of the C-shape

Scratch automatically snaps blocks together when they are dropped close to a valid connection point. A white highlight line appears when a snap connection is available.

### 6. Verify After Each Drag

Always take a screenshot after each drag to verify the block was placed correctly:

```
browser_take_screenshot
```

If a wrong block was dragged, undo with:
```
browser_press_key key="Control+z"
```

### 7. Run the Program

Click the green flag button to run the program:
```
browser_click ref="<green_flag_ref>" element="実行ボタン"
```

## Complete Example: Cat Follows Mouse

Build a program where the cat walks toward the mouse pointer:

```
🏴が押されたとき
ずっと
  マウスのポインターへ向ける
  10歩動かす
  次のコスチュームにする
```

### Step-by-step:

1. **Open editor** and take screenshot to see the layout.

2. **Click "イベント" category** to show event blocks.

3. **Locate "が押されたとき"** (flag clicked) block:
   ```javascript
   async (page) => {
     const el = await page.locator('text=が押されたとき').first();
     const box = await el.boundingBox();
     return JSON.stringify(box);
   }
   ```

4. **Drag it to the script area** at (500, 300).

5. **Click "制御" category**, locate "ずっと" block, and drag it to (530, 345) to connect below the flag block.

6. **Click "動き" category**, locate "マウスのポインター" text in the "へ向ける" block, and drag it to (520, 370) inside the forever loop.

7. **Locate "歩動かす"** block and drag it to (520, 405) below the previous block inside the loop.

8. **Click "見た目" category**, locate "次のコスチュームにする" and drag it to (520, 440) below the move block.

9. **Take screenshot** to verify the completed program.

10. **Click the green flag** to run.

## Verifying Block Connections via VM

**CRITICAL:** Visually adjacent blocks may NOT be actually connected. Scratch's snap detection is strict — blocks that appear close together on screen can still be independent. Always verify connections after assembling blocks.

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

### Fixing Disconnected Blocks via VM

If blocks are not connected after dragging, fix them programmatically using the VM's loadProject pattern:

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
The palette shows all blocks from multiple categories in a scrollable list. If you grab the wrong block, immediately undo with `Control+z` and retry with more precise coordinates. Always use `page.locator('text=...')` to get exact positions rather than guessing.

### Block Not Connecting (Most Common Issue)
Blocks often drop without snapping even when they look close. Key strategies:
- **Place the first hat block high** (e.g., y=200) to leave room below for connections
- **Drop the second block with the same x** as the first block and **y just below** (roughly +35px from the first block's bottom)
- **Use `boundingBox()` on blocks already in the script area** (search by text using `.nth(1)` for the second match if the same text appears in the palette) to find the precise bottom edge
- **Always verify via VM** after placing blocks (see above section)

### Palette Scrolling
Clicking a category button scrolls the palette to that category. After clicking, take a screenshot or locate elements again as positions may have changed.

### Multiple Blocks With Same Text
Some text appears in multiple blocks (e.g., numbers). Use the surrounding unique text to identify the correct block. For instance, search for "歩動かす" instead of "10" to find the move block.

### Coordinate Discovery Pattern
Always follow this pattern for reliable block placement:
1. Click the category
2. Use `locator('text=...').boundingBox()` to find the exact position
3. Drag from the found position to the target
4. Take a screenshot to verify
5. **Check VM block structure** to confirm connections are real
