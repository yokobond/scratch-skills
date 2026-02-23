---
name: scratch-skill-creator
description: Build a Scratch program from a user prompt using vibe coding, then distill the novel techniques into a new reusable SKILL.md. Use this skill when you want to grow the scratch-operation or scratch-coding plugin with a new pattern discovered during live coding.
---

# Scratch Skill Creator

This skill guides Claude through a **vibe coding session** that results in both a working Scratch program and a new, tested skill file.

## Phase 1 — Build the Program

### 1.1 Understand the Prompt

Read the user's description carefully. Identify:
- The **visual or interactive goal** (what should appear on screen / what should happen)
- The **Scratch mechanisms** likely needed (motion, pen, sound, broadcast, variables, clones, etc.)
- Any hints about the **coding style** (simple/complex, single sprite/multi-sprite, custom graphics, etc.)

### 1.2 Open the Scratch Editor

Use the `scratch-vm-injection` skill to open the editor and connect to the VM:

```
browser_navigate url="https://scratch.mit.edu/projects/editor/"
```

Wait for the editor to load (`browser_wait_for time=3`), then expose `window.vm` and define `window.updateSprite`.

### 1.3 Implement Iteratively

Build the program in short cycles:

1. **Plan** the block structure for one sprite or scene at a time.
2. **Inject** blocks via `window.updateSprite` or `browser_run_code`.
3. **Run** with `window.vm.greenFlag()`.
4. **Observe** the result using `browser_take_screenshot` or `browser_snapshot`.
5. **Adjust** until the behavior matches the intent.

> **Vibe coding rule**: Don't plan the entire program upfront. Code one piece, see how it feels, then continue. Follow the energy of what's working.

### 1.4 Note Techniques as You Go

While coding, maintain a running **Technique Log** (just in your working memory / scratchpad):

```
Technique Log:
- [opcode X] — what it does, non-obvious inputs
- [pattern Y] — why this ordering matters
- [gotcha Z] — what broke and how you fixed it
```

Anything that wasn't already obvious from existing skills is worth logging.

### 1.5 Save the Project to a File

Once the program is working correctly, **save the project as an `.sb3` file** using the `scratch-project-file` skill. This preserves the working implementation as a reference artifact alongside the new skill.

Save path convention:

```
plugins/<target-plugin>/skills/<skill-name>/example.sb3
```

Use the download pattern from `scratch-project-file`:

```javascript
// browser_run_code code:
async (page) => {
  const downloadPromise = page.waitForEvent('download', { timeout: 10000 });
  await page.evaluate(async () => {
    const blob = await window.vm.saveProjectSb3();
    const url = URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.href = url;
    a.download = 'example.sb3';
    document.body.appendChild(a);
    a.click();
    document.body.removeChild(a);
    URL.revokeObjectURL(url);
  });
  const download = await downloadPromise;
  await download.saveAs('/absolute/path/to/plugins/<target-plugin>/skills/<skill-name>/example.sb3');
  return 'Project saved';
}
```

> **Why save here?** The `.sb3` file serves as ground truth for the skill. If the SKILL.md documentation is ever unclear, the tester or a future developer can load this file to see the exact working implementation.

---

## Phase 2 — Distill the Skill

Once the program works, extract the reusable knowledge into a new skill file.

### 2.1 Choose a Skill Name and Target Plugin

- **`scratch-operation`** — for new editor operations (new Playwright interactions, new VM API usage, new asset handling)
- **`scratch-coding`** — for new Scratch coding patterns (new block combinations, new architecture patterns, new game/animation techniques)

Skill name format: `scratch-<noun>` (e.g., `scratch-clone-animation`, `scratch-variable-hud`, `scratch-audio-reactive`).

### 2.2 Create the Skill Directory

```
plugins/<target-plugin>/skills/<skill-name>/SKILL.md
```

### 2.3 Write the SKILL.md

Use this template:

```markdown
---
name: <skill-name>
description: <one sentence — what pattern this is and when to use it>
---

# <Human-readable title>

<One paragraph: the problem this pattern solves and why the naive approach fails>

## The Solution

<Concise explanation of the approach>

## Prerequisites

<Which existing skills this builds on>

## Implementation

### <Step or sub-pattern name>

<Explanation>

\`\`\`javascript
// Concrete code example
\`\`\`

## Tips & Best Practices

### 1. <Tip title>
<Detail>

### 2. <Tip title>
<Detail>

## When to Use This Pattern

- <Condition 1>
- <Condition 2>

## When NOT to Use This Pattern

- <Counter-condition 1>
```

**Quality checklist before saving:**
- [ ] The `description` front-matter is one sentence, specific enough to trigger the skill in context
- [ ] Every non-obvious block opcode or input format is shown with a concrete example
- [ ] Gotchas discovered during coding are documented as Tips
- [ ] "When to Use / When NOT to Use" sections are honest and specific
- [ ] No copy-paste from existing skills — only genuinely new information

### 2.4 Update the Plugin Index

**Add to the plugin's top-level `SKILL.md`** (the index file, e.g. `plugins/scratch-coding/SKILL.md`):

```markdown
- [New Skill Name](./skills/<skill-name>/SKILL.md): <same one-sentence description>
```

This is the only index update required. The plugin's `gemini-extension.json` and `.claude-plugin/plugin.json` reference the top-level `SKILL.md` as the context entry point — no `skills` array needs to be maintained in those files.

---

## Phase 3 — Hand Off to Skill Tester

After saving the new SKILL.md, invoke the **scratch-skill-tester** skill:

> "Now test the new skill `<skill-name>` in plugin `<target-plugin>`."

The tester will spawn an autonomous agent and report results. If the agent finds gaps or bugs in the SKILL.md, return to Phase 2 and fix them, then re-test.

---

## Example Session Transcript

```
User: "Make a Scratch program where clones bounce around and leave colored trails."

Phase 1:
  - Open editor, connect VM
  - Implement: Stage creates 5 clones of Sprite1
  - Each clone picks a random color, moves with pen down
  - Clones bounce off edges using motion_ifonedgebounce
  - Discover: pen color must be set AFTER pen down to take effect per-clone
  - Discover: clone init needs a brief wait before drawing or all clones start at origin

Phase 2:
  - Skill name: scratch-clone-pen-trail
  - Target plugin: scratch-coding
  - Document: clone initialization order, pen-color-per-clone pattern, edge bounce with pen

Phase 3:
  - Invoke scratch-skill-tester
  - Agent tries to build "fireflies that glow and fade" using the new skill
  - Agent reports: "pen_color input format not shown in SKILL.md"
  - Fix: add pen color input example to Tips section
  - Re-test: pass
```
