# scratch-skills

Agent skills for automating the Scratch editor via Playwright. Compatible with any [Agent Skills](https://agentskills.io/)-capable agent (GitHub Copilot, Claude Code, Cursor, Gemini, etc.).

## Skills

### Editor Operations

- **scratch-operation-code-injection** — Injects blocks directly into the Scratch VM by serializing/deserializing the project JSON. Best for programmatically creating complete Scratch programs quickly.
- **scratch-operation-block-dragging** — Builds Scratch programs by dragging and dropping blocks from the palette to the script area using mouse operations. Mimics real user interaction with the Scratch editor UI.
- **scratch-operation-project-file** — Download and upload Scratch project `.sb3` files.
- **scratch-operation-project-reader** — Read and summarize all sprites, scripts, and variables in the currently loaded project. Works with both new and existing public projects.

### Coding Patterns

- **scratch-coding-sprite-library** — Add sprites and backdrops from the Scratch built-in library.
- **scratch-coding-custom-costume** — Add custom SVG or PNG costumes to sprites programmatically.
- **scratch-coding-storytelling-messaging** — Patterns for sequential dialog and messages.
- **scratch-coding-multi-sprite-drawing** — Coordinate multiple sprites drawing simultaneously.

## Requirements

- Playwright MCP server configured in your agent (e.g. `@playwright/mcp`)
- Chromium installed for Playwright

## Installation

### GitHub CLI

```bash
gh skill install yokobond/scratch-skills
```

Install a single skill:

```bash
gh skill install yokobond/scratch-skills scratch-operation-code-injection
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
