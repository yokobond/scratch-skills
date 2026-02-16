# Scratch Skills for Gemini

This repository provides skills for Gemini to automate Scratch programming. It is organized into two separate extensions:

- **scratch-operation**: Core tools for interacting with the Scratch editor (VM injection, block dragging).
- **scratch-coding**: Advanced coding patterns and templates (multi-sprite coordination, storytelling).

## Installation

You can install one or both extensions depending on your needs.

### 1. Install via Gemini CLI

#### Scratch Operation (Core)
```bash
gemini extensions link plugins/scratch-operation
```

#### Scratch Coding (Patterns)
```bash
gemini extensions link plugins/scratch-coding
```

### 2. Manual Check-out

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

Once loaded, you can ask Gemini to create Scratch programs:

> "Create a Scratch program where the cat walks in a square." (Uses `scratch-operation`)

> "How do I make two sprites talk to each other?" (Uses `scratch-coding`)

## Requirements

- **Playwright MCP Server**: Ensure you have the Playwright MCP server configured in your `gemini-cli` or generic MCP settings.
- **Chromium**: The Playwright browser must be installed.
