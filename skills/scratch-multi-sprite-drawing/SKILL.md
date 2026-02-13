---
name: scratch-multi-sprite-drawing
description: Pattern for coordinating pen drawing across multiple sprites. Uses a stage-based broadcast to ensure "pen clear" runs once before all sprites begin drawing, preventing partial erasure.
---

# Scratch Multi-Sprite Drawing Coordination

This skill provides a pattern for Scratch projects where **multiple sprites draw with the pen extension simultaneously**. It solves the common problem where each sprite's "pen clear" block erases other sprites' drawings.

## The Problem

When multiple sprites each have their own "green flag clicked -> pen clear -> draw" script, the execution order is unpredictable. One sprite may start drawing before another sprite's "pen clear" runs, erasing the first sprite's lines.

Example of the broken pattern:

```
Sprite1: [flag clicked] -> [pen clear] -> [pen down] -> [repeat 3 ...]
Sprite2: [flag clicked] -> [pen clear] -> [pen down] -> [repeat 4 ...]
```

If Sprite1 draws first and Sprite2's "pen clear" runs after, Sprite1's drawing is partially or fully erased.

## The Solution

Move "pen clear" to the **Stage** and use a **broadcast message** to synchronize drawing start:

1. **Stage** handles the green flag, clears the pen, then broadcasts a "draw" message
2. **Each sprite** listens for the "draw" broadcast and begins drawing

This guarantees "pen clear" executes exactly once, before any sprite starts drawing.

```
Stage:   [flag clicked] -> [pen clear] -> [broadcast "draw"]
Sprite1: [when I receive "draw"] -> [pen down] -> [repeat 3 ...]
Sprite2: [when I receive "draw"] -> [pen down] -> [repeat 4 ...]
```

## Prerequisites

This skill builds on the **scratch-vm-injection** skill. Ensure:
- The Playwright MCP server is configured
- The Scratch editor is open and `window.vm` and `window.updateSprite` are available

## Implementation

### Block Definitions

Use `browser_run_code` to inject all targets at once:

```javascript
// browser_run_code code:
async (page) => {
  await page.evaluate(async () => {
    // Ensure pen extension is loaded
    if (!window.vm.extensionManager.isExtensionLoaded('pen')) {
      await window.vm.extensionManager.loadExtensionURL('pen');
    }

    const projectJSON = JSON.parse(window.vm.toJSON());

    // --- Stage: flag -> pen clear -> broadcast "draw" ---
    const stage = projectJSON.targets.find(t => t.isStage);

    // Register the broadcast message
    stage.broadcasts = stage.broadcasts || {};
    stage.broadcasts['draw_id'] = 'draw';

    stage.blocks = {
      'flag_clicked': {
        opcode: 'event_whenflagclicked',
        next: 'pen_erase',
        parent: null,
        inputs: {},
        fields: {},
        shadow: false,
        topLevel: true,
        x: 100,
        y: 100
      },
      'pen_erase': {
        opcode: 'pen_clear',
        next: 'broadcast',
        parent: 'flag_clicked',
        inputs: {},
        fields: {},
        shadow: false,
        topLevel: false
      },
      'broadcast': {
        opcode: 'event_broadcast',
        next: null,
        parent: 'pen_erase',
        inputs: {
          BROADCAST_INPUT: [1, [11, 'draw', 'draw_id']]
        },
        fields: {},
        shadow: false,
        topLevel: false
      }
    };

    // --- Sprite blocks: receive "draw" -> draw shape ---
    // Each sprite uses event_whenbroadcastreceived instead of event_whenflagclicked.
    // Example for a sprite that draws a triangle:
    const sprite1 = projectJSON.targets.find(t => t.name === 'Sprite1');
    sprite1.blocks = {
      'recv_draw': {
        opcode: 'event_whenbroadcastreceived',
        next: 'pen_down',   // first drawing block
        parent: null,
        inputs: {},
        fields: {
          BROADCAST_OPTION: ['draw', 'draw_id']
        },
        shadow: false,
        topLevel: true,
        x: 100,
        y: 100
      },
      // ... drawing blocks follow here
    };

    await window.vm.loadProject(JSON.stringify(projectJSON));
    window.vm.greenFlag();
  });
}
```

### Key Block Details

#### Broadcast message registration

Broadcast messages must be registered in the Stage's `broadcasts` object before use:

```javascript
stage.broadcasts['draw_id'] = 'draw';
//                 ^ ID         ^ display name
```

#### Sending a broadcast (Stage)

```javascript
{
  opcode: 'event_broadcast',
  inputs: {
    BROADCAST_INPUT: [1, [11, 'draw', 'draw_id']]
    //                    [11 = broadcast type, name, ID]
  }
}
```

#### Receiving a broadcast (Sprites)

```javascript
{
  opcode: 'event_whenbroadcastreceived',
  fields: {
    BROADCAST_OPTION: ['draw', 'draw_id']
    //                 [name,   ID]
  }
}
```

## When to Use This Pattern

Apply this pattern whenever a Scratch project meets **all** of these conditions:

1. **Multiple sprites** use the pen extension to draw
2. The canvas should be **cleared at the start** of execution (green flag)
3. No sprite's drawing should be erased by another sprite's "pen clear"

## When NOT to Use This Pattern

- **Single sprite drawing**: Just use `[flag clicked] -> [pen clear] -> ...` directly
- **No pen clear needed**: If the project intentionally layers drawings across runs
- **Sequential drawing**: If sprites should draw one after another, use `[broadcast "draw" and wait]` instead of `[broadcast "draw"]` combined with separate broadcast messages per sprite
