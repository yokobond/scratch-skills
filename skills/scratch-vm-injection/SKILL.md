---
name: scratch-vm-injection
description: Automates Scratch programming by injecting blocks directly into the web editor's VM. Use this skill when you need to create or run Scratch programs programmatically.
---

# Scratch VM Injection Skill

This skill programs Scratch (scratch.mit.edu) by directly interacting with the Scratch Virtual Machine (VM) running in the browser via the Playwright MCP tools.

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

This ensures assets (costumes/sounds) are preserved while code is updated perfectly.

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
