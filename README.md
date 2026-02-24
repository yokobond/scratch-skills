# scratch-skills

A plugin/extension that automates Scratch programming via Playwright.
Supports both **Claude Code** and **Gemini**.

## Plugins

### scratch-operation

Low-level skills for direct interaction with the Scratch editor.

- **scratch-vm-injection** — Injects blocks directly into the Scratch VM by serializing/deserializing the project JSON. Best for programmatically creating complete Scratch programs quickly.
- **scratch-block-dragging** — Builds Scratch programs by dragging and dropping blocks from the palette to the script area using mouse operations. Mimics real user interaction with the Scratch editor UI.
- **scratch-sprite-library** — Add sprites and backdrops from the Scratch built-in library.
- **scratch-custom-costume** — Add custom SVG or PNG costumes to sprites programmatically.
- **scratch-project-file** — Download and upload Scratch project `.sb3` files.

### scratch-coding

Advanced coding patterns and templates for Scratch projects.

- **scratch-multi-sprite-drawing** — Coordinate multiple sprites drawing simultaneously.
- **scratch-storytelling-messaging** — Patterns for sequential dialog and messages.

## Meta-Skills

### scratch-skill-creator

Guides a vibe coding session that produces both a working Scratch program and a new reusable `SKILL.md`. Use this to grow the `scratch-operation` or `scratch-coding` plugin with patterns discovered during live coding.

### scratch-skill-tester

Spawns an autonomous sub-agent that implements a test Scratch program using only a newly written skill, then reports whether the skill's documentation is accurate, complete, and sufficient to produce working code. Run this immediately after `scratch-skill-creator` produces a new skill.

## Requirements

### For Claude Code
- [Playwright plugin](https://github.com/anthropics/claude-plugins-official/tree/main/plugins/playwright) installed in Claude Code

### For Gemini
- **Playwright MCP Server**: Ensure you have the Playwright MCP server configured in your `gemini-cli` or generic MCP settings.
- **Chromium**: The Playwright browser must be installed.

## Installation

### For Claude Code

#### From marketplace

First, add the marketplace:

```
/plugins add-marketplace yokobond/scratch-skills
```

Then install the desired plugin:

```
/install-plugin scratch-operation
```

```
/install-plugin scratch-coding
```

#### From URL

```
/install-plugin https://github.com/yokobond/scratch-skills.git
```

### For Gemini

You can install one or both extensions depending on your needs.

#### 1. Install via Gemini CLI

**Scratch Operation (Core)**
```bash
gemini extensions link plugins/scratch-operation
```

**Scratch Coding (Patterns)**
```bash
gemini extensions link plugins/scratch-coding
```

#### 2. Manual Check-out

1. Clone this repository:
   ```bash
   git clone https://github.com/yokobond/scratch-skills.git
   ```

2. Add as a context/skill in your Gemini session:
   ```bash
   gemini add-context ./scratch-skills/plugins/scratch-operation
   gemini add-context ./scratch-skills/plugins/scratch-coding
   ```

## Usage

Ask Claude or Gemini to create a Scratch program:

```
Scratchで猫が四角形を描くプログラムを作って
```

```
Make a Scratch project where the cat bounces around the screen
```

> "Create a Scratch program where the cat walks in a square." (Uses `scratch-operation`)
> "How do I make two sprites talk to each other?" (Uses `scratch-coding`)

The plugin handles all the low-level interaction — finding the VM instance, block manipulation, and project management — so you can focus on describing what you want.

## License

MIT
