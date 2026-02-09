# scratch-programming

A Claude Code plugin that automates Scratch programming by injecting blocks directly into the web editor's VM via Playwright.

## What it does

This plugin enables Claude to programmatically create and run Scratch programs by:

1. Opening the Scratch editor (scratch.mit.edu)
2. Connecting to the Scratch VM through React internals
3. Injecting block JSON directly into sprites
4. Running the project with `vm.greenFlag()`

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
/install-plugin scratch-programming
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

The plugin handles all the low-level VM interaction — finding the VM instance, serializing/deserializing the project JSON, and injecting blocks — so you can focus on describing what you want.

## License

MIT
