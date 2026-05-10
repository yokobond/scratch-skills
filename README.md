# scratch-skills

Agent skills for automating the Scratch editor via Playwright. Compatible with any [Agent Skills](https://agentskills.io/)-capable agent (GitHub Copilot, Claude Code, Cursor, Gemini, etc.).

## Skills

### Editor Operations

- **scratch-project-edit** — Injects blocks directly into the Scratch VM by serializing/deserializing the project JSON. Best for programmatically creating complete Scratch programs quickly.
- **scratch-block-dragging** — Builds Scratch programs by dragging and dropping blocks from the palette to the script area using mouse operations. Mimics real user interaction with the Scratch editor UI.
- **scratch-project-file** — Download and upload Scratch project `.sb3` files.
- **scratch-project-inspect** — Read and summarize all sprites, scripts, and variables in the currently loaded project. Works with both new and existing public projects.
- **scratch-sprite-library** — Add sprites and backdrops from the Scratch built-in library.
- **scratch-costume-insert** — Add custom SVG or PNG costumes to sprites programmatically.

> **Coding patterns** (multi-sprite drawing, storytelling messaging) and skill-authoring workflows live in the companion repo [yokobond/scratch-coding-skills](https://github.com/yokobond/scratch-coding-skills), which depends on this one via APM.

## Requirements

These skills delegate all browser automation to the `playwright-cli` skill (provided by the [`@playwright/cli`](https://www.npmjs.com/package/@playwright/cli) npm package). To set it up once on a new machine:

```bash
# 1. Install the CLI binary from npm
npm install -g @playwright/cli

# 2. Register the playwright-cli skill so your agent can discover it
playwright-cli install --skills

# 3. Install the Chromium browser used by Playwright
playwright-cli install-browser
```

After this, agents that load the scratch-skills will be able to drive the Scratch editor through `playwright-cli` commands.

## Installation

### GitHub CLI

```bash
gh skill install yokobond/scratch-skills
```

Install a single skill:

```bash
gh skill install yokobond/scratch-skills scratch-project-edit
```

### APM

```bash
apm install yokobond/scratch-skills
```

### npx

```bash
npx skills add yokobond/scratch-skills
```

## Usage

Ask your agent to create a Scratch program:

```
Scratchで猫が四角形を描くプログラムを作って
```

```
Make a Scratch project where the cat bounces around the screen
```

The skills handle all low-level interaction — finding the VM instance, block manipulation, and project management — so you can focus on describing what you want.

## License

MIT
