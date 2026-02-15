---
name: scratch-storytelling-messaging
description: Pattern for storytelling with sequential dialog (message chaining) and simultaneous speech (parallel broadcast). Combines backdrop switching with broadcast messages to create multi-scene conversations between characters.
---

# Scratch Storytelling with Message Chaining

This skill provides a pattern for Scratch projects where **characters have conversations across multiple scenes**. It uses broadcast messages to implement both sequential dialog (one character speaks after another) and parallel speech (multiple characters speak at the same time).

## Core Concepts

### Sequential Processing

Characters take turns speaking by **chaining broadcast messages**. Each character waits for a specific message, speaks using a timed say block, then sends the next message to trigger the other character's line.

```
Sprite A: [receive "scene1"] -> [say "..." for 2 secs] -> [broadcast "reply"]
Sprite B: [receive "reply"]  -> [say "..." for 2 secs] -> [broadcast "next"]
```

The `say ... for N secs` block is crucial: it **blocks execution** for N seconds before proceeding to the next block. This ensures the previous speech bubble disappears before the next character's line appears.

### Parallel Processing

Characters speak simultaneously when **multiple sprites receive the same broadcast message**. Each sprite has its own `when I receive` hat block for the same message, and their scripts run in parallel.

```
Sprite A: [receive "finale"] -> [say "Hooray!"]
Sprite B: [receive "finale"] -> [say "Oh no!"]
```

The two sprites start speaking at nearly the same time. The exact order is unspecified and cannot be controlled, but they appear simultaneous to the viewer.

### Scene Switching

The **Stage** controls backdrop changes and acts as the bridge between scenes. It listens for a message indicating the current scene's dialog is complete, switches the backdrop, and broadcasts a new message to start the next scene.

```
Stage: [flag clicked] -> [switch backdrop to "scene1"] -> [broadcast "scene1"]
Stage: [receive "scene1done"] -> [switch backdrop to "scene2"] -> [broadcast "scene2"]
```

## Architecture

A typical storytelling project has this message flow:

```
[Green Flag]
    |
    v
Stage: switch to backdrop1, broadcast "scene1"
    |
    v
Sprite A receives "scene1": say line (2s), broadcast "lineA_done"
    |
    v
Sprite B receives "lineA_done": say reply (2s), broadcast "scene1done"
    |
    v
Stage receives "scene1done": wait, switch to backdrop2, broadcast "scene2"
    |
    v
Sprite A receives "scene2": say line  }  parallel
Sprite B receives "scene2": say line  }  (simultaneous)
```

## Prerequisites

This skill builds on the **scratch-vm-injection** skill. Ensure:
- The Playwright MCP server is configured
- The Scratch editor is open and `window.vm` is available

## Implementation

### Broadcast Message Registration

All broadcast messages must be registered in the Stage's `broadcasts` object. Each entry maps an internal ID to a display name.

```javascript
stage.broadcasts = {
  'b_scene1':     'scene1',
  'b_lineA_done': 'lineA_done',
  'b_scene1done': 'scene1done',
  'b_scene2':     'scene2'
};
```

### Stage Blocks

The Stage has two scripts: one for initialization and one for each scene transition.

```javascript
stage.blocks = {
  // Script 1: Green flag -> set initial backdrop -> broadcast scene1
  'flag': {
    opcode: 'event_whenflagclicked',
    next: 'switch_bg1',
    parent: null,
    inputs: {}, fields: {},
    shadow: false, topLevel: true,
    x: 50, y: 50
  },
  'switch_bg1': {
    opcode: 'looks_switchbackdropto',
    next: 'bcast_scene1',
    parent: 'flag',
    inputs: { BACKDROP: [1, 'bg1_menu'] },
    fields: {},
    shadow: false, topLevel: false
  },
  'bg1_menu': {
    opcode: 'looks_backdrops',
    next: null,
    parent: 'switch_bg1',
    inputs: {},
    fields: { BACKDROP: ['BackdropName1', null] },
    shadow: true, topLevel: false
  },
  'bcast_scene1': {
    opcode: 'event_broadcast',
    next: null,
    parent: 'switch_bg1',
    inputs: { BROADCAST_INPUT: [1, [11, 'scene1', 'b_scene1']] },
    fields: {},
    shadow: false, topLevel: false
  },

  // Script 2: Receive scene1done -> wait -> switch backdrop -> broadcast scene2
  'recv_scene1done': {
    opcode: 'event_whenbroadcastreceived',
    next: 'wait_transition',
    parent: null,
    inputs: {},
    fields: { BROADCAST_OPTION: ['scene1done', 'b_scene1done'] },
    shadow: false, topLevel: true,
    x: 50, y: 350
  },
  'wait_transition': {
    opcode: 'control_wait',
    next: 'switch_bg2',
    parent: 'recv_scene1done',
    inputs: { DURATION: [1, [5, '1']] },
    fields: {},
    shadow: false, topLevel: false
  },
  'switch_bg2': {
    opcode: 'looks_switchbackdropto',
    next: 'bcast_scene2',
    parent: 'wait_transition',
    inputs: { BACKDROP: [1, 'bg2_menu'] },
    fields: {},
    shadow: false, topLevel: false
  },
  'bg2_menu': {
    opcode: 'looks_backdrops',
    next: null,
    parent: 'switch_bg2',
    inputs: {},
    fields: { BACKDROP: ['BackdropName2', null] },
    shadow: true, topLevel: false
  },
  'bcast_scene2': {
    opcode: 'event_broadcast',
    next: null,
    parent: 'switch_bg2',
    inputs: { BROADCAST_INPUT: [1, [11, 'scene2', 'b_scene2']] },
    fields: {},
    shadow: false, topLevel: false
  }
};
```

### Sprite Blocks — Sequential Dialog

Each sprite in a sequential dialog chain receives a message, speaks with a timed say block, then broadcasts the next message.

```javascript
sprite.blocks = {
  // Sequential: receive message -> say line -> broadcast next message
  'recv_scene1': {
    opcode: 'event_whenbroadcastreceived',
    next: 'say_line',
    parent: null,
    inputs: {},
    fields: { BROADCAST_OPTION: ['scene1', 'b_scene1'] },
    shadow: false, topLevel: true,
    x: 50, y: 50
  },
  'say_line': {
    opcode: 'looks_sayforsecs',
    next: 'bcast_next',
    parent: 'recv_scene1',
    inputs: {
      MESSAGE: [1, [10, 'Dialog text here']],
      SECS: [1, [4, '2']]
    },
    fields: {},
    shadow: false, topLevel: false
  },
  'bcast_next': {
    opcode: 'event_broadcast',
    next: null,
    parent: 'say_line',
    inputs: { BROADCAST_INPUT: [1, [11, 'lineA_done', 'b_lineA_done']] },
    fields: {},
    shadow: false, topLevel: false
  }
};
```

### Sprite Blocks — Parallel Speech

For simultaneous speech, multiple sprites each have a `when I receive` block for the **same** broadcast message. No further chaining is needed — both scripts run in parallel.

```javascript
// Sprite A — parallel speech
spriteA.blocks = {
  'recv_scene2': {
    opcode: 'event_whenbroadcastreceived',
    next: 'say_parallel',
    parent: null,
    inputs: {},
    fields: { BROADCAST_OPTION: ['scene2', 'b_scene2'] },
    shadow: false, topLevel: true,
    x: 50, y: 300
  },
  'say_parallel': {
    opcode: 'looks_sayforsecs',
    next: null,
    parent: 'recv_scene2',
    inputs: {
      MESSAGE: [1, [10, 'Sprite A reaction']],
      SECS: [1, [4, '2']]
    },
    fields: {},
    shadow: false, topLevel: false
  }
};

// Sprite B — same broadcast, different line
spriteB.blocks = {
  'recv_scene2': {
    opcode: 'event_whenbroadcastreceived',
    next: 'say_parallel',
    parent: null,
    inputs: {},
    fields: { BROADCAST_OPTION: ['scene2', 'b_scene2'] },
    shadow: false, topLevel: true,
    x: 50, y: 300
  },
  'say_parallel': {
    opcode: 'looks_sayforsecs',
    next: null,
    parent: 'recv_scene2',
    inputs: {
      MESSAGE: [1, [10, 'Sprite B reaction']],
      SECS: [1, [4, '2']]
    },
    fields: {},
    shadow: false, topLevel: false
  }
};
```

## Key Block Details

### Timed Say (looks_sayforsecs)

Shows a speech bubble for a fixed duration, then clears it. **Blocks execution** until the duration elapses — this is essential for sequential dialog timing.

```javascript
{
  opcode: 'looks_sayforsecs',
  inputs: {
    MESSAGE: [1, [10, 'Text to display']],   // [10] = string literal
    SECS: [1, [4, '2']]                      // [4] = number literal
  }
}
```

### Persistent Say (looks_say)

Shows a speech bubble that stays until explicitly cleared. Use this for the final line in a scene if the bubble should remain visible.

```javascript
{
  opcode: 'looks_say',
  inputs: {
    MESSAGE: [1, [10, 'Text to display']]
  }
}
```

### Backdrop Switch (looks_switchbackdropto)

Requires a shadow menu block (`looks_backdrops`) specifying the backdrop name.

```javascript
{
  opcode: 'looks_switchbackdropto',
  inputs: { BACKDROP: [1, 'menu_block_id'] }
}
// Shadow menu:
{
  opcode: 'looks_backdrops',
  fields: { BACKDROP: ['BackdropName', null] },
  shadow: true
}
```

### Sprite Positioning (motion_gotoxy)

Set character positions at the start of each scene to place them appropriately for the backdrop.

```javascript
{
  opcode: 'motion_gotoxy',
  inputs: {
    X: [1, [4, '-80']],
    Y: [1, [4, '-30']]
  }
}
```

## When to Use This Pattern

Apply this pattern when a Scratch project needs:

1. **Two or more characters having a conversation** — dialog must appear in order, one line at a time
2. **Scene changes** — the backdrop switches between conversation segments
3. **Simultaneous reactions** — multiple characters respond to the same event at the same time

## When NOT to Use This Pattern

- **Single character narration**: Use a simple `[flag clicked] -> [say ... for N secs]` chain
- **User-driven dialog**: If the user clicks to advance dialog, use `when this sprite clicked` events instead of timed broadcasts
- **Continuous animation**: If characters move and speak simultaneously throughout, consider `broadcast and wait` with `forever` loops instead

## Extending the Pattern

### Adding More Scenes

Add more `when I receive` / `broadcast` pairs to the Stage for each scene transition. The message chain can be extended indefinitely:

```
scene1 -> lineA -> lineB -> scene1done -> scene2 -> lineC -> scene2done -> scene3 -> ...
```

### Mixing Sequential and Parallel Within a Scene

Within a single scene, you can have sequential dialog followed by a parallel moment:

```
[receive "scene1"] -> [say line1 for 2s] -> [broadcast "reply"]
[receive "reply"]  -> [say line2 for 2s] -> [broadcast "react"]
[receive "react"]  -> [say "wow!" for 2s]    // Sprite A  } parallel
[receive "react"]  -> [say "yay!" for 2s]    // Sprite B  }
```

### Using "broadcast and wait"

Replace `event_broadcast` with `event_broadcastandwait` when the sending sprite needs to perform actions after the receiving sprite finishes. This is useful when one character should react after another completes a multi-step sequence.

```javascript
{
  opcode: 'event_broadcastandwait',
  inputs: { BROADCAST_INPUT: [1, [11, 'messageName', 'message_id']] }
}
```
