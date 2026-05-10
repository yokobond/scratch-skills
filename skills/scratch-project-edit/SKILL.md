---
name: scratch-project-edit
description: Automates Scratch programming by injecting blocks directly into the web editor's VM. Use this skill when you need to create or run Scratch programs programmatically.
license: MIT
---

# Scratch VM Injection Skill

This skill programs Scratch (scratch.mit.edu) by directly interacting with the Scratch Virtual Machine (VM) running in the browser via the `playwright-cli` skill.

## Prerequisites

This skill drives the browser via the `playwright-cli` skill. Ensure that skill is installed and a browser session has been opened (e.g. `playwright-cli open --headed https://scratch.mit.edu/projects/editor/`).

> **Visibility**: Always use `--headed` when opening the browser so that the Scratch editor is visible to the user during project creation.

## Workflow

### 1. Open Scratch Editor

Open the editor in a visible Chrome browser:

```bash
playwright-cli open --headed https://scratch.mit.edu/projects/editor/
```

### 2. Connect to VM

The most critical step is finding the Scratch VM instance hidden within the React components. Run the finder via `playwright-cli run-code` and expose it as `window.vm`:

```bash
playwright-cli run-code "$(cat <<'EOF'
async page => await page.evaluate(() => {
  const findVM = () => {
    if (window.vm) return window.vm;
    const root = document.body;
    const queue = [root];
    const visited = new Set();
    while (queue.length > 0) {
      const node = queue.shift();
      if (visited.has(node)) continue;
      visited.add(node);
      const keys = Object.keys(node);
      const reactKey = keys.find(
        key => key.startsWith('__reactInternalInstance$') ||
               key.startsWith('__reactFiber$')
      );
      if (reactKey) {
        let fiber = node[reactKey];
        while (fiber) {
          if (fiber.memoizedProps && fiber.memoizedProps.vm)
            return fiber.memoizedProps.vm;
          fiber = fiber.return;
        }
      }
      if (node.children) {
        for (let i = 0; i < node.children.length; i++) {
          queue.push(node.children[i]);
        }
      }
    }
    return null;
  };

  const vm = findVM();
  if (!vm) throw new Error('Scratch VM not found');
  window.vm = vm;
  return 'VM found and exposed as window.vm';
})
EOF
)"
```

If the VM is not found, the editor may not be fully loaded. Wait a few seconds and retry:

```bash
playwright-cli run-code "async page => await page.waitForTimeout(3000)"
```

### 3. Define the Update Helper

After connecting to the VM, define the reusable `window.updateSprite(name, blocks)` helper. This function handles JSON serialization, block injection, and project reloading.

```bash
playwright-cli run-code "$(cat <<'EOF'
async page => await page.evaluate(() => {
  const ensureVM = () => {
    if (window.vm) return window.vm;
    const root = document.body;
    const queue = [root];
    const visited = new Set();
    while (queue.length > 0) {
      const node = queue.shift();
      if (visited.has(node)) continue;
      visited.add(node);
      const keys = Object.keys(node);
      const reactKey = keys.find(
        key => key.startsWith('__reactInternalInstance$') ||
               key.startsWith('__reactFiber$')
      );
      if (reactKey) {
        let fiber = node[reactKey];
        while (fiber) {
          if (fiber.memoizedProps && fiber.memoizedProps.vm)
            return fiber.memoizedProps.vm;
          fiber = fiber.return;
        }
      }
      if (node.children) {
        for (let i = 0; i < node.children.length; i++) {
          queue.push(node.children[i]);
        }
      }
    }
    return null;
  };

  window.updateSprite = async (spriteName, blocks) => {
    const foundVM = ensureVM();
    if (foundVM && !window.vm) window.vm = foundVM;
    const activeVM = window.vm || foundVM;
    if (!activeVM) throw new Error('Scratch VM not found');

    const projectJSON = JSON.parse(activeVM.toJSON());

    let sprite;
    if (spriteName === '_stage_') {
      sprite = projectJSON.targets.find(t => t.isStage);
    } else if (spriteName) {
      sprite = projectJSON.targets.find(t => t.name === spriteName);
    } else {
      sprite = projectJSON.targets.find(t => !t.isStage);
    }
    if (!sprite)
      throw new Error('Sprite not found: ' + (spriteName || 'default'));

    sprite.blocks = blocks;
    await activeVM.loadProject(JSON.stringify(projectJSON));
    return 'Updated ' + sprite.name;
  };

  return 'window.updateSprite is now defined';
})
EOF
)"
```

### 4. Program Sprites

Once `window.vm` is set and `window.updateSprite` is defined, you can inject blocks.

#### Safely Update Blocks (The "Load Project" Pattern)
**CRITICAL:** Do not use `target.blocks.createBlock` directly for complex scripts, as it often fails to set internal properties (like `BROADCAST_OPTION`) or parent/child relationships correctly.
**Instead:**
1. Serialize the current project (`vm.toJSON()`).
2. Parse the JSON.
3. Modify the `blocks` object of the target sprite in the JSON.
4. Reload the project (`vm.loadProject()`).

This ensures **built-in and CDN-hosted** assets (costumes/sounds) are preserved while code is updated. However, **costumes injected via the storage API** (using the **scratch-costume-insert** skill, in the companion [`yokobond/scratch-coding-skills`](https://github.com/yokobond/scratch-coding-skills) repo) are lost — see Tip #6 below.

Use `playwright-cli run-code` to call `updateSprite` and then run the project:

```bash
playwright-cli run-code "$(cat <<'EOF'
async page => await page.evaluate(async () => {
  await window.updateSprite('Sprite1', {
    'START': {
      opcode: 'event_whenflagclicked',
      next: 'NEXT_BLOCK',
      parent: null,
      topLevel: true,
      x: 100,
      y: 100
    }
    // ... other blocks
  });
  window.vm.greenFlag();
  return 'Program started';
})
EOF
)"
```

For complex block definitions, use the same `playwright-cli run-code` heredoc form — the body is multi-line Playwright code:

```bash
playwright-cli run-code "$(cat <<'EOF'
async (page) => {
  await page.evaluate(async () => {
    const blocks = {
      'flag_clicked': {
        opcode: 'event_whenflagclicked',
        next: 'move_step',
        parent: null,
        inputs: {},
        fields: {},
        shadow: false,
        topLevel: true,
        x: 100,
        y: 100
      },
      'move_step': {
        opcode: 'motion_movesteps',
        next: null,
        parent: 'flag_clicked',
        inputs: { STEPS: [1, [4, '10']] },
        fields: {},
        shadow: false,
        topLevel: false
      }
    };
    await window.updateSprite('Sprite1', blocks);
    window.vm.greenFlag();
  });
}
EOF
)"
```

### 5. Managing Sprites

- **Rename**: `vm.renameSprite(target.id, 'NewName')`
- **Duplicate**: `await vm.duplicateSprite(target.id)` (Async!)
- **Delete**: `vm.deleteSprite(target.id)`

These can be called via `playwright-cli run-code`, e.g.:

```bash
playwright-cli run-code "$(cat <<'EOF'
async page => await page.evaluate(() => {
  const target = window.vm.runtime.targets.find(
    t => t.sprite.name === 'Sprite1'
  );
  window.vm.renameSprite(target.id, 'Player');
  return 'Renamed';
})
EOF
)"
```

## Block JSON Structure

- **opcode**: Internal Scratch opcode (e.g., `motion_movesteps`, `control_forever`).
- **inputs**: Arguments for the block. Keys are input names (e.g., `STEPS`, `SUBSTACK`).
  - Values must be objects: `{ name: "INPUT_NAME", block: "block_id", shadow: "shadow_id_if_any" }`
- **fields**: Dropdowns or values.
  - Values: `{ name: "FIELD_NAME", value: "value" }`
- **shadow**: `true` if it's a value block inside another block (like a number inside move steps).
- **topLevel**: `true` for event blocks (Hats).

## Tips & Best Practices

### 1. Executing Complex Scripts
For large block definitions, `playwright-cli run-code` accepts an `async page => { ... }` function. Use the bash heredoc form (`"$(cat <<'EOF' ... EOF)"`) so multi-line JS is preserved verbatim and shell metacharacters (`$`, backticks) inside the JS are not expanded.

### 2. Wait for VM
The VM might not be available immediately after `playwright-cli goto`. If the VM finder fails, wait a few seconds and retry:

```bash
playwright-cli run-code "async page => await page.waitForTimeout(3000)"
```

### 3. Checking Page State
Use `playwright-cli snapshot` to inspect the current page state and verify the editor has loaded correctly before attempting to connect to the VM.

### 4. Input Types Reference
When defining `inputs`, Scratch uses specific arrays for literal values:
- **Number**: `[1, [4, "10"]]` (The `4` indicates a number type)
- **String**: `[1, [10, "hello"]]`
- **Wait/Duration**: `[1, [5, "0.1"]]` (The `5` is often used for positive numbers/duration)
- **Block Connection**: `[2, "BLOCK_ID"]` (The `2` indicates a connection to another block, e.g., for `SUBSTACK`)

### 5. Custom Costumes and loadProject()

If the project uses costumes injected via `storage.createAsset()` (the **scratch-costume-insert** skill), calling `loadProject()` — including via `updateSprite` — will **destroy those costumes**. Scratch tries to fetch every asset by md5 hash from the remote CDN, which fails for local-only assets, leaving "?" placeholders.

**Fix**: Re-inject the costumes **inside the same `page.evaluate` call** immediately after `loadProject()` completes. Splitting into two separate `page.evaluate` calls risks a render gap where the broken "?" costume briefly appears. Also, always **add the new costume first, then delete old ones** — `deleteCostume` silently fails when only one costume remains.

```bash
playwright-cli run-code "$(cat <<'EOF'
async (page) => {
  const svgData = `<svg ...>...</svg>`;  // keep costume data in Node.js scope

  // Update blocks AND re-inject costumes in a single evaluate (no render gap)
  await page.evaluate(async (svg) => {
    const vm = window.vm;

    // Step 1: update blocks
    const projectJSON = JSON.parse(vm.toJSON());
    const sprite = projectJSON.targets.find(t => !t.isStage);
    sprite.blocks = { /* ... */ };
    await vm.loadProject(JSON.stringify(projectJSON));
    // NOTE: custom costumes are now lost — re-inject immediately below

    // Step 2: re-inject costumes
    const target = vm.editingTarget;
    const storage = vm.runtime.storage;

    // Record old costume count BEFORE adding new ones
    const oldCount = target.getCostumes().length;

    // Add new costume FIRST (so the list always has >1 entry during deletion)
    const asset = storage.createAsset(
      storage.AssetType.ImageVector, storage.DataFormat.SVG,
      new TextEncoder().encode(svg), null, true
    );
    await vm.addCostume(asset.assetId + '.svg', {
      name: 'my-costume', dataFormat: 'svg', asset,
      md5: asset.assetId + '.svg', assetId: asset.assetId,
      bitmapResolution: 1, rotationCenterX: 50, rotationCenterY: 50
    });

    // THEN delete old costumes (cat, "?", or any leftovers)
    for (let i = oldCount - 1; i >= 0; i--) target.deleteCostume(i);

    target.setCostume(0);
  }, svgData);
}
EOF
)"
```

For PNG costumes, convert bytes to base64 before passing to `page.evaluate` (since `Uint8Array` is not JSON-serializable), then decode with `atob()` inside. See the **scratch-costume-insert** skill for the full PNG re-injection pattern.

### 6. Passing Arguments and Escaping Characters
When using `page.evaluate(fn, arg)` within `playwright-cli run-code`, keep these in mind to avoid common errors:
- **Single Argument Limit**: `page.evaluate` only accepts **one** argument after the function. If you need to pass multiple values (like blocks for different sprites), wrap them in a single object:
  ```javascript
  await page.evaluate(async ({rBlocks, tBlocks}) => {
    await window.updateSprite('Rabbit', rBlocks);
    await window.updateSprite('Turtle', tBlocks);
  }, {rBlocks: rabbitBlocks, tBlocks: turtleBlocks});
  ```
- **Avoid Syntax Errors in Strings**: Inside the heredoc, special characters like backslashes (`\`) or literal newlines inside strings still need to be valid JavaScript. Unescaped `\n` or `\t` can break the code, leading to `SyntaxError: Invalid or unexpected token`.
- **Serialization**: All data passed to `page.evaluate` must be JSON-serializable. Functions or complex class instances cannot be passed this way.
- **Quoted heredoc terminator**: Always write `<<'EOF'` (single-quoted) — the quotes prevent the shell from expanding `$`, backticks, and `\` inside the JS body.

### 7. Syncing Editor UI

When injecting blocks using `target.blocks.createBlock` or modifying the block model directly (instead of using `loadProject`), the Scratch editor UI (Blockly workspace) may not reflect the changes immediately. To force a UI update, call:

```javascript
window.vm.emitWorkspaceUpdate();
```

This is particularly useful when developing extensions or using internal APIs where a full project reload is overkill.

**IMPORTANT**: `emitWorkspaceUpdate()` called immediately after `loadProject()` will NOT reliably show blocks. The GUI re-mounts the Blockly component asynchronously after `loadProject()`. Call `emitWorkspaceUpdate()` only after a ~1.5 second delay. See Tip #9 for the full reliable workflow.

### 8. Showing Blocks in the Workspace After loadProject()

After `loadProject()`, the Scratch GUI re-mounts the Blockly workspace component asynchronously. To reliably display blocks in the editor:

**Step 1**: Call `loadProject()` and wait ~1.5 seconds for the component to re-mount:

```bash
playwright-cli run-code "$(cat <<'EOF'
async (page) => {
  await page.evaluate(async () => {
    const vm = window.vm;
    const projectJSON = JSON.parse(vm.toJSON());
    // ... modify projectJSON.targets[spriteIndex].blocks ...
    await vm.loadProject(JSON.stringify(projectJSON));
  });
  await page.waitForTimeout(1500);
}
EOF
)"
```

**Step 2**: Emit the workspace update and scroll into view:

```bash
playwright-cli run-code "$(cat <<'EOF'
async (page) => {
  await page.evaluate(() => { window.vm.emitWorkspaceUpdate(); });
  await page.waitForTimeout(1000);
  await page.evaluate(() => {
    const svgEl = document.querySelector('svg.blocklyWorkspace') ||
                  document.querySelector('[class*="blocklyWorkspace"]')?.closest('svg');
    let el = svgEl ? svgEl.parentElement : null;
    for (let i = 0; i < 20 && el; i++) {
      const keys = Object.keys(el);
      const fk = keys.find(k => k.startsWith('__reactFiber$') || k.startsWith('__reactInternalInstance$'));
      if (fk) {
        let fiber = el[fk];
        while (fiber) {
          if (fiber.stateNode && fiber.stateNode.workspace && fiber.stateNode.workspace.getAllBlocks) {
            const ws = fiber.stateNode.workspace;
            window._scratchWorkspace = ws;
            ws.scrollCenter();
            return ws.getAllBlocks().length + ' blocks visible';
          }
          fiber = fiber.return;
        }
      }
      el = el.parentElement;
    }
  });
}
EOF
)"
```

To zoom out and see all scripts at once:

```bash
playwright-cli run-code "$(cat <<'EOF'
async page => await page.evaluate(() => {
  const ws = window._scratchWorkspace;
  if (!ws) return;
  ws.setScale(0.6);
  const blocks = ws.getTopBlocks(false);
  if (blocks.length > 0) {
    const bounds = blocks.reduce((b, blk) => {
      const xy = blk.getRelativeToSurfaceXY();
      return { minX: Math.min(b.minX, xy.x), minY: Math.min(b.minY, xy.y) };
    }, { minX: Infinity, minY: Infinity });
    ws.scroll(-bounds.minX * 0.6 + 20, -bounds.minY * 0.6 + 20);
  }
})
EOF
)"
```

### 9. Dynamic Menu Shadow Blocks — Workspace Display Limitation

Some shadow blocks are **dynamically registered** based on the project's sprite list and are **NOT pre-registered** in ScratchBlocks. When `emitWorkspaceUpdate()` tries to render a script containing these blocks, the **entire script chain is silently dropped** from the workspace (the VM still runs it correctly).

**Known problematic block** (causes "Invalid block definition" error in ScratchBlocks):
- `sensing_distanceto_menu` — the dropdown for "distance to ▼"

**Workaround**: Replace `sensing_distanceto` with `sensing_touchingobject` for mouse-proximity detection. The touching menu block works fine:

```javascript
// ❌ AVOID — sensing_distanceto_menu is not registered, whole script won't render
'dist_sense': { opcode: 'sensing_distanceto', inputs: { DISTANCETOMENU: [1, 'dist_menu'] }, ... },
'dist_menu':  { opcode: 'sensing_distanceto_menu', fields: { DISTANCETOMENU: ['_mouse_', null] }, shadow: true, ... },

// ✅ USE INSTEAD — sensing_touchingobjectmenu works correctly
'touch':   { opcode: 'sensing_touchingobject', inputs: { TOUCHINGOBJECTMENU: [1, 'touch_m'] }, ... },
'touch_m': { opcode: 'sensing_touchingobjectmenu', fields: { TOUCHINGOBJECTMENU: ['_mouse_', null] }, shadow: true, ... },
```

**Menu shadow blocks confirmed to work** in the workspace:
- `motion_goto_menu` (`_random_`, `_mouse_`)
- `motion_pointtowards_menu` (`_mouse_`)
- `sensing_touchingobjectmenu` (`_mouse_`, `_edge_`)
- `sound_sounds_menu`

**Sound name localization**: Sound names in `sound_sounds_menu` fields depend on the editor's display language. The default cat sprite's meow sound is `"Meow"` in English but `"ニャー"` in Japanese. Always confirm the actual sound name at runtime:
```javascript
const cat = vm.runtime.targets.find(t => !t.isStage);
cat.sprite.sounds.map(s => s.name); // e.g. ["ニャー"]
```

### 10. DANGER — Do Not Call setEditingTarget() After loadProject()

Calling `vm.setEditingTarget()` causes the Scratch GUI to **save the current workspace state back to the VM**. If the workspace is empty or partially rendered at that moment, it **overwrites the injected blocks with the empty state**, destroying your work.

```javascript
// ❌ NEVER DO THIS after loadProject() — destroys injected blocks
vm.setEditingTarget(stageTarget.id);   // saves empty workspace → VM (wipes sprite blocks!)
vm.setEditingTarget(spriteTarget.id);  // reloads from now-empty VM

// ✅ CORRECT — let the GUI handle target selection on its own
await vm.loadProject(JSON.stringify(projectJSON));
// wait, then call emitWorkspaceUpdate() — do NOT touch setEditingTarget
```

### 11. Shadow Blocks (Important)

When defining shadow blocks (e.g., dropdown menus, number inputs) in your `blocks` object, follow these rules to ensure they are correctly rendered and persisted:

1.  **Correct Opcode**: Ensure you use the exact internal opcode (e.g., `note` for music notes, `motion_direction_menu` for direction dropdowns).
2.  **Parent Relationship**: A shadow block MUST have its parent block's ID in its `parent` property.
3.  **Field Format**: Use the array format `[value, null]` for fields within shadow blocks.
4.  **Input Connection**: Connect the shadow in the parent block's `inputs` using the `[1, "shadow_id"]` format (or `[3, "block_id", "shadow_id"]` if there's an overlapping block).

**Correct Example:**
```javascript
const blocks = {
  'move_block': {
    opcode: 'motion_movesteps',
    inputs: { 'STEPS': [1, 'steps_shadow'] },
    parent: 'start_block',
    // ...
  },
  'steps_shadow': {
    opcode: 'math_number',
    fields: { 'NUM': ['10', null] },
    parent: 'move_block',
    shadow: true
  }
};
```
