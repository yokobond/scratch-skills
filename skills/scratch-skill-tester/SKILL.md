---
name: scratch-skill-tester
description: Spawn an autonomous sub-agent that implements a test Scratch program using only a newly written skill, then reports whether the skill's documentation is accurate, complete, and sufficient to produce working code.
---

# Scratch Skill Tester

This skill validates a newly created SKILL.md by having a **fresh, isolated agent** attempt to build a Scratch program using only that skill. Because the agent starts without any memory of the original coding session, it will expose missing documentation, wrong code examples, and hidden assumptions.

## When to Use This Skill

Run this skill immediately after `scratch-skill-creator` produces a new SKILL.md. Also run it after editing a SKILL.md in response to prior test failures.

## Inputs Required

Before invoking, confirm you have:

1. **Skill path** — relative path to the new skill file, e.g.:
   `plugins/scratch-coding/skills/scratch-clone-pen-trail/SKILL.md`
2. **Test program description** — a short, different program that exercises the skill's core pattern. It must be:
   - **Different** from the program built during skill creation (avoids the agent copying from memory)
   - **Focused** — it should specifically require the pattern the skill teaches
   - **Achievable in ~5 minutes** of agent work

---

## How to Spawn the Test Agent

Spawn a sub-agent using the tool available in your environment:
- **VS Code Copilot**: use the `runSubagent` tool
- **Claude Desktop / Claude.ai**: use the `Task` tool with `subagent_type: "general-purpose"`
- **Other**: use whatever sub-agent / parallel agent invocation is supported

Construct the prompt as follows:

```
You are a Scratch programming agent. Your job is to build a working Scratch program
using ONLY the instructions in the skill file provided below. Do not rely on any
knowledge beyond what the skill file contains plus the general scratch-vm-injection
and scratch-operation skills you already know.

## Skill to Test

<paste the full contents of the new SKILL.md here>

## Program to Build

<describe the test program here — be concrete about what should appear on screen
and what the user should observe when they click the green flag>

## Save Path

./tmp/<skill-name>/test.sb3

## Your Task

1. Open the Scratch editor at https://scratch.mit.edu/projects/editor/
2. Connect to the VM using the standard approach
3. Implement the program using ONLY the pattern described in the skill above
4. Run the program and take a screenshot; save the screenshot to `./tmp/<skill-name>/screenshot-build.png`
5. Run `mkdir -p ./tmp/<skill-name>` to create the directory, then save the project as an .sb3 file to the save path above using the scratch-project-file download pattern
6. **Reload test**: Load the saved `.sb3` file back into the Scratch editor using the scratch-project-file load pattern, then click the green flag and confirm the program runs correctly. Take a screenshot and save it to `./tmp/<skill-name>/screenshot-reload.png`
7. Report the result in this format:

### Test Report

**Status**: PASS / PARTIAL / FAIL

**Build screenshot**: [attached or described]

**Reload screenshot**: [attached or described — confirms the saved .sb3 runs correctly after loading]

**Saved project**: <path to saved .sb3 file, or "not saved" with reason>

**Reload test**: PASS / FAIL — <brief description of what happened when the project was loaded and re-run>

**What worked**: <list>

**What failed or was unclear**: <list — be specific about which section of the
SKILL.md was missing, wrong, or ambiguous>

**Suggested SKILL.md fixes**: <concrete edits — quote the problematic line and
propose a replacement>
```

### Save Path Convention

The `./tmp` folder is used **exclusively for temporary test artifacts produced by the sub-agent** — Scratch project files (`.sb3`) and screenshots taken during a test run. It is **not** used for skill files.

| Artifact | Location |
|---|---|
| Sub-agent test project (`.sb3`) | `./tmp/<skill-name>/test.sb3` |
| Sub-agent build screenshot | `./tmp/<skill-name>/screenshot-build.png` |
| Sub-agent reload screenshot | `./tmp/<skill-name>/screenshot-reload.png` |
| New skill file | `plugins/<target-plugin>/skills/<skill-name>/SKILL.md` |
| Creator example project | `plugins/<target-plugin>/skills/<skill-name>/example.sb3` |

Before saving, create the tmp directory:

```bash
mkdir -p ./tmp/<skill-name>
```

If multiple test rounds are needed, use numbered filenames: `./tmp/<skill-name>/test-1.sb3`, `./tmp/<skill-name>/test-2.sb3`, etc.

> **Note**: `./tmp` is gitignored and must never be committed. Skill files under `plugins/` are what get committed.

---

## Evaluating the Test Report

### PASS

The agent successfully built and ran the program. The SKILL.md is ready.

Next steps:
- Confirm the test `.sb3` was saved to `./tmp/<skill-name>/test.sb3`.
- Confirm the new skill file exists at `plugins/<target-plugin>/skills/<skill-name>/SKILL.md`.
- Commit the files under `plugins/` (SKILL.md, example.sb3, marketplace.json update). Do **not** commit anything under `./tmp`.
- Optionally run one more test with a third, more complex program to increase confidence.

### PARTIAL

The agent built something but it didn't fully work, or the agent had to guess at undocumented details.

Next steps:
1. Read the "What failed" and "Suggested fixes" sections carefully.
2. Edit the SKILL.md to fill the gaps (add missing opcodes, fix input format examples, clarify ordering).
3. Re-run scratch-skill-tester with the **same test program** until it passes.

### FAIL

The agent could not build the program at all, or produced something completely wrong.

Common causes:
- A critical code example is missing from the SKILL.md
- An opcode name or input key is wrong
- A required prerequisite step is not mentioned
- The "When to Use" section misdescribes the pattern

Next steps:
1. Return to scratch-skill-creator Phase 2 and substantially revise the SKILL.md.
2. Add a complete, working code example for the core pattern.
3. Re-run scratch-skill-tester.

---

## Debug Loop

```
[scratch-skill-tester] → FAIL/PARTIAL
       │
       ▼
Read agent's "Suggested SKILL.md fixes"
       │
       ▼
Edit SKILL.md (add examples, fix opcodes, clarify tips)
       │
       ▼
[scratch-skill-tester] again (same test program)
       │
       ▼ (repeat until PASS)
```

After **two consecutive PASS results** with different test programs, the skill is considered stable.

---

## Example Agent Prompt (Filled In)

```
You are a Scratch programming agent. Build a working Scratch program using ONLY the
instructions in the skill file below.

## Skill to Test

---
name: scratch-clone-pen-trail
description: Pattern for creating multiple clones that each draw colored pen trails
while bouncing off edges, using correct clone initialization order.
---

# Scratch Clone Pen Trail

... (full SKILL.md contents) ...

## Program to Build

Create a Scratch program where 8 clones of a small circle sprite each start at a
random position on the stage, move in random directions, and draw a pen trail in
their own unique color. When a clone hits an edge it should bounce. The result
should look like 8 colored squiggly lines drawn simultaneously over about 5 seconds.

## Save Path

./tmp/scratch-clone-pen-trail/test.sb3

## Your Task

1. Open https://scratch.mit.edu/projects/editor/
2. Connect to VM
3. Implement using the clone-pen-trail pattern
4. Click green flag, wait 5 seconds
5. Run `mkdir -p ./tmp/scratch-clone-pen-trail`, take a screenshot and save it to `./tmp/scratch-clone-pen-trail/screenshot-build.png`
6. Save the project as .sb3 to the save path above
7. Load the saved .sb3 back into the editor, click green flag again, take a screenshot and save it to `./tmp/scratch-clone-pen-trail/screenshot-reload.png`
8. Report: PASS / PARTIAL / FAIL with details including the reload test result
```

---

## Tips

### Keep the Test Program Different

If the test program is identical to the skill creator's example, the agent may reproduce it from general knowledge rather than truly relying on the SKILL.md. Use the same **technique** but a different **theme**.

### Test the Hardest Part First

Focus the test program on the most complex or unusual aspect of the skill — the part most likely to be underdocumented.

### One Test Agent per Run

Do not reuse a test agent across multiple test rounds. Each round should use a fresh agent invocation so it has no memory of previous attempts.

### Verify Both Screenshots

After the agent reports PASS, check both screenshots:
- `screenshot-build.png` — confirms the program worked immediately after coding
- `screenshot-reload.png` — confirms the saved `.sb3` is a complete, self-contained project that runs correctly after loading

Agents sometimes report success optimistically. If the reload screenshot looks different from the build screenshot (e.g. blank stage, missing sprites), the save/load cycle has a bug and the test should be marked PARTIAL or FAIL.
,