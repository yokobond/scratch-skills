---
name: scratch-project-inspect
description: Reads and summarizes the code of the Scratch project currently loaded in the editor. Use this skill when you need to inspect what sprites, scripts, and blocks exist in the project before making changes.
license: MIT
---

# Scratch Project Reader

This skill reads the project JSON from the Scratch VM and extracts a human-readable summary of every sprite's scripts.

## Prerequisites

This skill drives the browser via the `playwright-cli` skill. Ensure that skill is installed and the Scratch editor is open in the browser session (see `scratch-project-edit` skill for opening it).

---

## Workflow

### Step 1 — Connect to the VM

If `window.vm` is not yet set, find and expose it:

```bash
playwright-cli run-code "$(cat <<'EOF'
async page => await page.evaluate(() => {
  if (window.vm) return 'VM already connected';
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
        if (fiber.memoizedProps && fiber.memoizedProps.vm) {
          window.vm = fiber.memoizedProps.vm;
          return 'VM found and exposed as window.vm';
        }
        fiber = fiber.return;
      }
    }
    if (node.children) {
      for (let i = 0; i < node.children.length; i++) {
        queue.push(node.children[i]);
      }
    }
  }
  return 'ERROR: VM not found — editor may not be loaded yet';
})
EOF
)"
```

If the VM is not found, wait a few seconds and retry:

```bash
playwright-cli run-code "async page => await page.waitForTimeout(3000)"
```

---

### Step 2 — Read and Summarize the Project

Run the following code to extract sprites and scripts. It returns a structured text summary.

```bash
playwright-cli run-code "$(cat <<'EOF'
async page => await page.evaluate(() => {
  if (!window.vm) return 'ERROR: window.vm is not set. Run Step 1 first.';

  const project = JSON.parse(window.vm.toJSON());

  // Build a block chain starting from a top-level block id
  function buildChain(blockId, blocks, depth) {
    if (!blockId || depth > 100) return [];
    const block = blocks[blockId];
    if (!block) return [];

    const lines = [];

    // Format inputs (only show non-shadow references)
    const inputParts = [];
    for (const [key, val] of Object.entries(block.inputs || {})) {
      const [, ref] = val;
      if (ref && typeof ref === 'string' && blocks[ref] && !blocks[ref].shadow) {
        // Inline reporter block
        const inner = blocks[ref];
        const innerField = Object.values(inner.fields || {})[0];
        inputParts.push(`${key}: [${inner.opcode}${innerField ? ' ' + innerField[0] : ''}]`);
      } else if (ref && typeof ref === 'object') {
        // Primitive literal
        inputParts.push(`${key}: ${JSON.stringify(ref[1])}`);
      }
    }

    // Format fields
    const fieldParts = Object.entries(block.fields || {}).map(
      ([k, v]) => `${k}: ${v[0]}`
    );

    const allArgs = [...inputParts, ...fieldParts].join(', ');
    const indent = '  '.repeat(depth);
    lines.push(`${indent}${block.opcode}${allArgs ? '  (' + allArgs + ')' : ''}`);

    // Recurse into sub-stacks (e.g. if/repeat bodies)
    for (const [key, val] of Object.entries(block.inputs || {})) {
      if (key === 'SUBSTACK' || key === 'SUBSTACK2') {
        const [, substackId] = val;
        if (substackId && typeof substackId === 'string') {
          lines.push(`${indent}  [${key}]:`);
          lines.push(...buildChain(substackId, blocks, depth + 2));
        }
      }
    }

    // Follow next block in sequence
    lines.push(...buildChain(block.next, blocks, depth));
    return lines;
  }

  const output = [];

  for (const target of project.targets) {
    const label = target.isStage ? '=== Stage ===' : `=== Sprite: ${target.name} ===`;
    output.push(label);

    const blocks = target.blocks;
    const topLevelIds = Object.keys(blocks).filter(id => blocks[id].topLevel);

    if (topLevelIds.length === 0) {
      output.push('  (no scripts)');
    }

    for (const startId of topLevelIds) {
      output.push('');
      output.push(...buildChain(startId, blocks, 1));
    }

    output.push('');
  }

  return output.join('\n');
})
EOF
)"
```

The return value is a plain-text summary like:

```
=== Stage ===
  (no scripts)

=== Sprite: Sprite1 ===

  event_whenflagclicked
  motion_movesteps  (STEPS: 10)
  control_repeat  (TIMES: 10)
    [SUBSTACK]:
      motion_turnright  (DEGREES: 15)
```

---

### Step 3 — Interpret the Output

Read the summary to understand:

| Term | Meaning |
|------|---------|
| `event_whenflagclicked` | Script starts when green flag is clicked |
| `motion_movesteps` | Move N steps |
| `control_repeat` | Repeat loop with SUBSTACK body |
| `looks_sayforsecs` | Say bubble for N seconds |
| `data_setvariableto` | Set variable |
| `broadcast_broadcast` | Send a broadcast message |

Use the [Scratch block opcode reference](https://en.scratch-wiki.info/wiki/Scratch_File_Format#Blocks) for the full opcode list when you encounter an unfamiliar opcode.

---

### Step 4 — Get Full Raw JSON (Optional)

If you need the complete unabridged project data for a specific sprite, use:

```bash
playwright-cli run-code "$(cat <<'EOF'
async page => await page.evaluate((spriteName) => {
  if (!window.vm) return 'ERROR: window.vm not set';
  const project = JSON.parse(window.vm.toJSON());
  const target = spriteName
    ? project.targets.find(t => t.name === spriteName)
    : project.targets.find(t => !t.isStage);
  if (!target) return 'Sprite not found: ' + spriteName;
  return JSON.stringify(target.blocks, null, 2);
}, 'Sprite1')
EOF
)"
```

Pass the sprite name as the second argument to `page.evaluate`. This is useful when you need exact block IDs, input structures, or field values to modify later.

---

### Step 5 — List Variables and Lists

```bash
playwright-cli run-code "$(cat <<'EOF'
async page => await page.evaluate(() => {
  if (!window.vm) return 'ERROR: window.vm not set';
  const project = JSON.parse(window.vm.toJSON());
  const output = [];
  for (const target of project.targets) {
    const label = target.isStage ? 'Stage' : `Sprite: ${target.name}`;
    const vars = Object.values(target.variables || {});
    const lists = Object.values(target.lists || {});
    if (vars.length || lists.length) {
      output.push(label);
      for (const [name, value] of vars) output.push(`  var  ${name} = ${JSON.stringify(value)}`);
      for (const [name, items] of lists) output.push(`  list ${name} = [${items.join(', ')}]`);
    }
  }
  return output.length ? output.join('\n') : '(no variables or lists found)';
})
EOF
)"
```

---

## Tips

### 1. Run Step 2 before reading any script
Always check the summary first. It tells you which sprite names and opcodes exist, preventing wasted injection attempts.

### 2. Combine with `scratch-project-edit` when editing
After reading the project to understand its structure, use `window.updateSprite` from the `scratch-project-edit` skill to make changes. Use the raw block IDs from Step 4 as anchors.

### 3. Shadow blocks are skipped intentionally
Shadow blocks (e.g., default numeric literal inputs) are omitted from the summary to reduce noise. Use Step 4 (raw JSON) if you need them.
