---
name: scratch-vm-injection
description: Automates Scratch programming by injecting blocks directly into the web editor's VM. Use this skill when you need to create or run Scratch programs programmatically.
---

# Scratch VM Injection Skill

This skill programs Scratch (scratch.mit.edu) by directly interacting with the Scratch Virtual Machine (VM) running in the browser via the Playwright MCP tools.

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

## Workflow

### 1. Open Scratch Editor

Navigate to the editor using `browser_navigate`:
```
browser_navigate url="https://scratch.mit.edu/projects/editor/"
```

### 2. Connect to VM

The most critical step is finding the Scratch VM instance hidden within the React components. Use `browser_evaluate` to find it and expose it as `window.vm`.

```javascript
// browser_evaluate function:
() => {
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
}
```

If the VM is not found, the editor may not be fully loaded. Wait a few seconds with `browser_wait_for time=3` and retry.

### 3. Define the Update Helper

After connecting to the VM, define the reusable `window.updateSprite(name, blocks)` helper. This function handles JSON serialization, block injection, and project reloading.

```javascript
// browser_evaluate function:
() => {
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
}
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

This ensures **built-in and CDN-hosted** assets (costumes/sounds) are preserved while code is updated. However, **costumes injected via the storage API** (using the **scratch-custom-costume** skill) are lost — see Tip #6 below.

Use `browser_evaluate` to call `updateSprite` and then run the project:

```javascript
// browser_evaluate function:
async () => {
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
}
```

For complex block definitions, use `browser_run_code` which supports multi-line Playwright code:

```javascript
// browser_run_code code:
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
```

### 5. Managing Sprites

- **Rename**: `vm.renameSprite(target.id, 'NewName')`
- **Duplicate**: `await vm.duplicateSprite(target.id)` (Async!)
- **Delete**: `vm.deleteSprite(target.id)`

These can be called via `browser_evaluate`, e.g.:
```javascript
// browser_evaluate function:
() => {
  const target = window.vm.runtime.targets.find(
    t => t.sprite.name === 'Sprite1'
  );
  window.vm.renameSprite(target.id, 'Player');
  return 'Renamed';
}
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
For large block definitions that are hard to fit in a single `browser_evaluate` call, use `browser_run_code` which accepts multi-line Playwright code. The code parameter takes an `async (page) => { ... }` function.

### 2. Wait for VM
The VM might not be available immediately after `browser_navigate`. If the VM finder fails, use `browser_wait_for time=3` and retry.

### 3. Checking Page State
Use `browser_snapshot` to inspect the current page state and verify the editor has loaded correctly before attempting to connect to the VM.

### 4. Input Types Reference
When defining `inputs`, Scratch uses specific arrays for literal values:
- **Number**: `[1, [4, "10"]]` (The `4` indicates a number type)
- **String**: `[1, [10, "hello"]]`
- **Wait/Duration**: `[1, [5, "0.1"]]` (The `5` is often used for positive numbers/duration)
- **Block Connection**: `[2, "BLOCK_ID"]` (The `2` indicates a connection to another block, e.g., for `SUBSTACK`)

### 5. Custom Costumes and loadProject()

If the project uses costumes injected via `storage.createAsset()` (the **scratch-custom-costume** skill), calling `loadProject()` — including via `updateSprite` — will **destroy those costumes**. Scratch tries to fetch every asset by md5 hash from the remote CDN, which fails for local-only assets, leaving "?" placeholders.

**Fix**: Re-inject the costumes **inside the same `page.evaluate` call** immediately after `loadProject()` completes. Splitting into two separate `page.evaluate` calls risks a render gap where the broken "?" costume briefly appears. Also, always **add the new costume first, then delete old ones** — `deleteCostume` silently fails when only one costume remains.

```javascript
// browser_run_code code:
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
```

For PNG costumes, convert bytes to base64 before passing to `page.evaluate` (since `Uint8Array` is not JSON-serializable), then decode with `atob()` inside. See the **scratch-custom-costume** skill for the full PNG re-injection pattern.

### 6. Passing Arguments and Escaping Characters
When using `page.evaluate(fn, arg)` within `browser_run_code`, keep these in mind to avoid common errors:
- **Single Argument Limit**: `page.evaluate` only accepts **one** argument after the function. If you need to pass multiple values (like blocks for different sprites), wrap them in a single object:
  ```javascript
  await page.evaluate(async ({rBlocks, tBlocks}) => {
    await window.updateSprite('Rabbit', rBlocks);
    await window.updateSprite('Turtle', tBlocks);
  }, {rBlocks: rabbitBlocks, tBlocks: turtleBlocks});
  ```
- **Avoid Syntax Errors in Strings**: Be extremely careful with special characters like backslashes (`\`) or literal newlines inside strings when using `browser_run_code`. Unescaped `\n` or `\t` can break the code being sent to the browser, leading to `SyntaxError: Invalid or unexpected token`.
- **Serialization**: All data passed to `page.evaluate` must be JSON-serializable. Functions or complex class instances cannot be passed this way.

### 7. Syncing Editor UI

When injecting blocks using `target.blocks.createBlock` or modifying the block model directly (instead of using `loadProject`), the Scratch editor UI (Blockly workspace) may not reflect the changes immediately. To force a UI update, call:

```javascript
window.vm.emitWorkspaceUpdate();
```

This is particularly useful when developing extensions or using internal APIs where a full project reload is overkill.

### 8. Shadow Blocks (Important)

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
