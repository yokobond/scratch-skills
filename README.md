# scratch-skills

A Claude Code plugin that automates Scratch programming via Playwright.

## Skills

### scratch-vm-injection

Injects blocks directly into the Scratch VM by serializing/deserializing the project JSON. Best for programmatically creating complete Scratch programs quickly.

### scratch-block-dragging

Builds Scratch programs by dragging and dropping blocks from the palette to the script area using mouse operations. Mimics real user interaction with the Scratch editor UI.

## Requirements

- [Playwright plugin](https://github.com/anthropics/claude-plugins-official/tree/main/plugins/playwright) installed in Claude Code

## Installation

### From marketplace

First, add the marketplace:

```
/plugins add-marketplace yokobond/scratch-skills
```

Then install the plugin:

```
/install-plugin scratch-vm-injection
```

### From URL

```
/install-plugin https://github.com/yokobond/scratch-skills.git
```

## Usage

Ask Claude to create a Scratch program:

```
Scratchで猫が四角形を描くプログラムを作って
```

```
Make a Scratch project where the cat bounces around the screen
```

The plugin handles all the low-level interaction — finding the VM instance, block manipulation, and project management — so you can focus on describing what you want.

## License

MIT
