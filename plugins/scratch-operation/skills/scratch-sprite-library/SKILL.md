---
name: scratch-sprite-library
description: Adds sprites and backdrops from the Scratch built-in library via the editor UI. Use this skill when you need to add pre-made characters, animals, objects, or backgrounds to a Scratch project.
---

# Scratch Sprite Library Skill

This skill adds sprites and backdrops to a Scratch project by interacting with the built-in library dialogs in the Scratch editor via Playwright MCP tools.

## Prerequisites

This skill requires the **Playwright MCP server** to be installed and configured. Ensure the following is added to your MCP settings (e.g. `~/.claude/claude_desktop_config.json` or project `.mcp.json`):

```json
{
  "mcpServers": {
    "playwright": {
      "command": "npx",
      "args": ["@anthropic-ai/mcp-playwright@latest"]
    }
  }
}
```

If the browser is not installed, run `browser_install` first to download the required Chromium binary.

## When to Use

- When you need specific characters (animals, people, fantasy creatures, etc.)
- When you need themed backdrops (outdoor, indoor, space, etc.)
- When costumes with proper SVG assets and multiple animation frames are needed
- **Prefer this over manual `addSprite()` with JSON** — the old CDN asset URLs (`cdn.assets.scratch.mit.edu/internalapi/asset/...`) no longer work, so programmatic sprite creation with hardcoded asset IDs will fail to load costumes

## Language-Independent Selectors

The Scratch editor supports dynamic language switching. All UI text (button labels, placeholders, category names) changes with the locale. **Never use `aria-label`, placeholder text, or visible text for element selection.**

Instead, use CSS class prefix matching. Scratch uses CSS Modules with the pattern `{component}_{style}_{hash}`. The hash changes per build, but `{component}_{style}` is stable across builds and locales. Always use `[class*="component_style"]` selectors.

### Selector Reference

| Element | Selector |
|---|---|
| Sprite chooser button | `div[class*="sprite-selector_add-button"] button[class*="action-menu_main-button"]` |
| Backdrop chooser button | `div[class*="stage-selector_add-button"] button[class*="action-menu_main-button"]` |
| Library dialog | `div[class*="modal_modal-content"][role="dialog"]` |
| Library search textbox | `input[class*="filter_filter-input"]` |
| Library item buttons | `button[class*="library-item_library-item"]` |
| Library item name span | `span[class*="library-item_library-item-name"]` |
| Category filter buttons | `button[class*="tag-button_tag-button"]` |
| Active category button | `button[class*="tag-button_tag-button"][class*="tag-button_active"]` |
| Library scroll grid | `div[class*="library_library-scroll-grid"]` |
| Modal close/back button | `div[class*="modal_header-item-close"] button` |

## Workflow

### 1. Open Scratch Editor

```
browser_navigate url="https://scratch.mit.edu/projects/editor/"
```

Wait for the editor to fully load.

### 2. Open the Sprite Library Dialog

**CRITICAL:** The sprite chooser button has overlapping sub-buttons that intercept pointer events, causing `browser_click` to time out. You **must** use JavaScript `click()` instead.

```javascript
// browser_run_code code:
async (page) => {
  await page.evaluate(() => {
    const btn = document.querySelector(
      'div[class*="sprite-selector_add-button"] button[class*="action-menu_main-button"]'
    );
    if (btn) btn.click();
  });
  await page.waitForTimeout(2000);
  return 'Opened sprite library';
}
```

### 3. Search for a Sprite (Optional)

Type into the search box to filter the library. **Sprite names are always in English** regardless of the editor locale.

```javascript
// browser_run_code code:
async (page) => {
  const searchBox = page.locator('input[class*="filter_filter-input"]');
  await searchBox.fill('Hare');
  await page.waitForTimeout(1000);
  return 'Searched';
}
```

### 4. Click a Sprite to Add It

Find the library item by matching the name text inside `span[class*="library-item_library-item-name"]`. The dialog closes automatically after selection.

```javascript
// browser_run_code code:
async (page) => {
  await page.evaluate((name) => {
    const spans = document.querySelectorAll('span[class*="library-item_library-item-name"]');
    for (const span of spans) {
      if (span.textContent.trim() === name) {
        span.closest('button[class*="library-item_library-item"]').click();
        return;
      }
    }
    throw new Error('Sprite not found: ' + name);
  }, 'Hare');
  await page.waitForTimeout(1000);
  return 'Sprite added';
}
```

### 5. Rename the Sprite (Optional)

After adding, rename the sprite via the VM for localized or custom names:

```javascript
// browser_evaluate function:
() => {
  const vm = window.vm;
  const target = vm.runtime.targets.find(t => t.sprite.name === 'Hare');
  if (target) vm.renameSprite(target.id, 'うさぎ');
  return 'Renamed';
}
```

### 6. Repeat for Additional Sprites

To add multiple sprites, repeat steps 2–5 for each one. The library dialog must be re-opened each time since it closes after each selection.

## Adding Backdrops

The backdrop library works the same way but uses a different button.

### Open Backdrop Library

```javascript
// browser_run_code code:
async (page) => {
  await page.evaluate(() => {
    const btn = document.querySelector(
      'div[class*="stage-selector_add-button"] button[class*="action-menu_main-button"]'
    );
    if (btn) btn.click();
  });
  await page.waitForTimeout(2000);
  return 'Opened backdrop library';
}
```

### Search and Select a Backdrop

```javascript
// browser_run_code code:
async (page) => {
  const searchBox = page.locator('input[class*="filter_filter-input"]');
  await searchBox.fill('Blue Sky');
  await page.waitForTimeout(1000);

  await page.evaluate((name) => {
    const spans = document.querySelectorAll('span[class*="library-item_library-item-name"]');
    for (const span of spans) {
      if (span.textContent.trim() === name) {
        span.closest('button[class*="library-item_library-item"]').click();
        return;
      }
    }
    throw new Error('Backdrop not found: ' + name);
  }, 'Blue Sky');
  await page.waitForTimeout(1000);
  return 'Backdrop added';
}
```

## Filtering by Category

Category buttons are ordered consistently regardless of locale. Use `:nth-child(n)` to select by position.

| Position | Category (English) |
|----------|-------------------|
| 1 | All |
| 2 | Animals |
| 3 | People |
| 4 | Fantasy |
| 5 | Dance |
| 6 | Music |
| 7 | Sports |
| 8 | Food |
| 9 | Fashion |
| 10 | Letters |

```javascript
// browser_run_code code:
async (page) => {
  // Click "Animals" category (2nd button)
  await page.evaluate(() => {
    const buttons = document.querySelectorAll('button[class*="tag-button_tag-button"]');
    if (buttons.length >= 2) buttons[1].click(); // 0-indexed
  });
  await page.waitForTimeout(1000);
  return 'Filtered by Animals';
}
```

## Deleting the Default Sprite

To remove the default cat sprite (Sprite1) before adding your own:

```javascript
// browser_evaluate function:
() => {
  const vm = window.vm;
  const cat = vm.runtime.targets.find(t => t.sprite.name === 'Sprite1');
  if (cat) vm.deleteSprite(cat.id);
  return 'Default sprite deleted';
}
```

**Note:** This requires the VM to be connected first (see the `scratch-vm-injection` skill for the VM finder code).

## Available Sprites Reference

### Animal Sprites

| Sprite Name | Notes |
|-------------|-------|
| Bat | 2 costumes |
| Bear | 2 costumes |
| Bear-walking | walking animation |
| Beetle | 2 costumes |
| Butterfly 1 | 2 costumes |
| Butterfly 2 | 2 costumes |
| Cat | default sprite, 2 costumes |
| Cat 2 | alternate cat |
| Cat Flying | flying animation |
| Chick | 3 costumes |
| Crab | 2 costumes |
| Dinosaur1–5 | various dinosaurs |
| Dog1 | 2 costumes |
| Dog2 | 3 costumes |
| Dove | 2 costumes |
| Dragon | 2 costumes |
| Dragonfly | 2 costumes |
| Duck | 2 costumes |
| Elephant | 2 costumes |
| Fish | 4 costumes |
| Fox | 2 costumes |
| Frog | 2 costumes |
| Frog 2 | alternate frog |
| Giraffe | 2 costumes |
| Grasshopper | 2 costumes |
| Hare | 4 costumes (hare-a through hare-d) |
| Hedgehog | 2 costumes |
| Hen | 2 costumes |
| Hippo1 | 2 costumes |
| Horse | 2 costumes |
| Jellyfish | 4 costumes |
| Ladybug1 | 2 costumes |
| Ladybug2 | 4 costumes |
| Lion | 2 costumes |
| Llama | 3 costumes |
| Monkey | 3 costumes |
| Mouse1 | 2 costumes |
| Octopus | 5 costumes |
| Owl | 2 costumes |
| Panther | 3 costumes |
| Parrot | 2 costumes |
| Penguin | 3 costumes |
| Penguin 2 | 2 costumes |
| Polar Bear | 3 costumes |
| Pufferfish | 3 costumes |
| Puppy | 3 costumes |
| Rabbit | 5 costumes |
| Reindeer | 2 costumes |
| Rooster | 3 costumes |
| Shark | 2 costumes |
| Shark 2 | 3 costumes |
| Snake | 3 costumes |
| Squirrel | 2 costumes |
| Starfish | 2 costumes |
| Toucan | 2 costumes |
| Unicorn | 2 costumes |
| Unicorn 2 | 3 costumes |
| Unicorn Running | running animation |
| Zebra | 2 costumes |


### Commonly Used Non-Animal Sprites

| Sprite Name | Type |
|-------------|------|
| Arrow1 | object |
| Ball | object |
| Button1–5 | UI elements |
| Green Flag | UI element |
| Heart | object |
| Key | object |
| Lightning | effect |
| Rocketship | vehicle |
| Star | object |
| Sun | nature |
| Tree1 | nature |
| Trees | nature |

## Complete Example: Add Hare and Frog for a Story

```javascript
// browser_run_code code:
async (page) => {
  // Helper: open sprite library via JS click (language-independent)
  const openSpriteLibrary = async () => {
    await page.evaluate(() => {
      const btn = document.querySelector(
        'div[class*="sprite-selector_add-button"] button[class*="action-menu_main-button"]'
      );
      if (btn) btn.click();
    });
    await page.waitForTimeout(2000);
  };

  // Helper: select a sprite by English name from the open library dialog
  const selectSprite = async (name) => {
    await page.evaluate((spriteName) => {
      const spans = document.querySelectorAll('span[class*="library-item_library-item-name"]');
      for (const span of spans) {
        if (span.textContent.trim() === spriteName) {
          span.closest('button[class*="library-item_library-item"]').click();
          return;
        }
      }
      throw new Error('Sprite not found: ' + spriteName);
    }, name);
    await page.waitForTimeout(1000);
  };

  // Delete default cat sprite (requires VM to be connected)
  await page.evaluate(() => {
    const vm = window.vm;
    const cat = vm.runtime.targets.find(t => t.sprite.name === 'Sprite1');
    if (cat) vm.deleteSprite(cat.id);
  });

  // Add Hare
  await openSpriteLibrary();
  await selectSprite('Hare');

  // Add Frog
  await openSpriteLibrary();
  await selectSprite('Frog');

  // Rename sprites
  await page.evaluate(() => {
    const vm = window.vm;
    const hare = vm.runtime.targets.find(t => t.sprite.name === 'Hare');
    if (hare) vm.renameSprite(hare.id, 'うさぎ');
    const frog = vm.runtime.targets.find(t => t.sprite.name === 'Frog');
    if (frog) vm.renameSprite(frog.id, 'かめ');
  });

  return 'Added and renamed Hare → うさぎ, Frog → かめ';
}
```

## Tips & Troubleshooting

### Dialog Not Opening
- The sprite chooser button has nested sub-buttons (upload, surprise, paint, library) that intercept clicks. Always use the JavaScript `click()` pattern with the CSS class selector shown above.
- If the dialog still doesn't open, verify the selector matches by checking `document.querySelector('div[class*="sprite-selector_add-button"] button[class*="action-menu_main-button"]')` returns a non-null element.

### Sprite Not Found in Search
- All sprite names are in **English** regardless of the editor locale. Search with English names (e.g., "Hare" not "うさぎ").
- The search input matches on sprite names. If no results appear, clear the search and browse by category instead.

### Adding Multiple Sprites Quickly
- The library dialog closes after each selection. You must re-open it for each sprite.
- For batch additions, use the `browser_run_code` pattern with helper functions (see complete example above).

### Sprite Position After Adding
- Newly added sprites appear at a default position (often near center or slightly offset).
- Use `vm.runtime.targets.find(t => t.sprite.name === 'Name')` to find the target, then set `.x`, `.y`, `.size`, `.direction` properties, or use motion blocks in your program to position sprites at startup.

### CSS Class Hash Changes
- The hash suffix in CSS class names (e.g., `_KILTP` in `action-menu_main-button_KILTP`) may change when Scratch deploys a new build. The prefix portion (`action-menu_main-button`) is stable. Always use `[class*="prefix"]` matching, never exact class names.
