---
name: scratch-operation-project-file
description: Downloads (saves) and uploads (loads) Scratch project .sb3 files. Use this skill when you need to save a Scratch project to a local file or load a previously saved .sb3 file into the editor.
license: MIT
---

# Scratch Project File Skill

This skill handles saving Scratch projects as `.sb3` files to the local filesystem and loading `.sb3` files back into the Scratch editor.

## Prerequisites

This skill drives the browser via the `playwright-cli` skill. Ensure that skill is installed and a browser session has been opened (e.g. `playwright-cli open --headed https://scratch.mit.edu/projects/editor/`).

> **Visibility**: Always use `--headed` when opening the browser so that the Scratch editor is visible to the user during project creation.

## When to Use

- When the user asks to save/download/export a Scratch project
- When the user asks to load/upload/import a `.sb3` file into the editor
- When you need to persist a project before making destructive changes
- When transferring a project between sessions

## Downloading (Saving) a Project

Use `playwright-cli run-code` to trigger `vm.saveProjectSb3()`, create a download link, and capture the downloaded file via Playwright's download event.

### Step 1: Ensure VM is Connected

The VM must be available as `window.vm`. If not, find it first (see `scratch-operation-code-injection` skill).

### Step 2: Download the .sb3 File

```bash
playwright-cli run-code "$(cat <<'EOF'
async (page) => {
  // Set up download handler BEFORE triggering download
  const downloadPromise = page.waitForEvent('download', { timeout: 10000 });

  // Generate .sb3 blob and trigger browser download
  await page.evaluate(async (filename) => {
    const blob = await window.vm.saveProjectSb3();
    const url = URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.href = url;
    a.download = filename;
    document.body.appendChild(a);
    a.click();
    document.body.removeChild(a);
    URL.revokeObjectURL(url);
  }, 'my-project.sb3');

  // Capture the download and save to local path
  const download = await downloadPromise;
  await download.saveAs('/absolute/path/to/my-project.sb3');
  return 'Saved to /absolute/path/to/my-project.sb3';
}
EOF
)"
```

**Key points:**
- `page.waitForEvent('download')` MUST be called BEFORE triggering the download (before `page.evaluate`). Otherwise the event is missed.
- The filename passed to `a.download` is just a hint for the browser; the actual save path is determined by `download.saveAs()`.
- Always use an absolute path for `download.saveAs()`.

### Complete Download Example

```bash
playwright-cli run-code "$(cat <<'EOF'
async (page) => {
  const downloadPromise = page.waitForEvent('download', { timeout: 10000 });

  await page.evaluate(async () => {
    const vm = window.vm;
    if (!vm) throw new Error('VM not connected. Run VM finder first.');
    const blob = await vm.saveProjectSb3();
    const url = URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.href = url;
    a.download = 'project.sb3';
    document.body.appendChild(a);
    a.click();
    document.body.removeChild(a);
    URL.revokeObjectURL(url);
  });

  const download = await downloadPromise;
  const savePath = '/Users/username/projects/project.sb3';
  await download.saveAs(savePath);
  return 'Project saved to: ' + savePath;
}
EOF
)"
```

## Uploading (Loading) a Project
To load a `.sb3` file into the Scratch editor, you can directly load the project via the VM. This bypasses the UI file chooser and loads the project programmatically.
For a purely programmatic approach that bypasses the UI menu, read the file as base64 and load it directly through the VM. This requires the file content to be passed into the browser context.

**Note:** This method works but does NOT update certain UI states (like the project title).

```bash
playwright-cli run-code "$(cat <<'EOF'
async (page) => {
  // Read the .sb3 file and convert to base64
  const fs = require('fs');  // Only available in the Node.js (run-code) context
  const fileBuffer = fs.readFileSync('/absolute/path/to/project.sb3');
  const base64 = fileBuffer.toString('base64');

  await page.evaluate(async (b64) => {
    // Decode base64 to ArrayBuffer
    const binary = atob(b64);
    const bytes = new Uint8Array(binary.length);
    for (let i = 0; i < binary.length; i++) {
      bytes[i] = binary.charCodeAt(i);
    }

    // Load into VM
    await window.vm.loadProject(bytes.buffer);
  }, base64);

  await page.waitForTimeout(2000);
  return 'Project loaded via VM';
}
EOF
)"
```

**IMPORTANT:** `require('fs')` is available in the Node.js context that `playwright-cli run-code` provides, but NOT inside `page.evaluate()` (which runs in the browser). The file must be read in the outer (Node) layer and the data passed to `page.evaluate` as a JSON-serializable argument.

## Handling File Chooser Dialogs

### Avoiding Stale Dialogs

- For downloads, always use the `downloadPromise` pattern (set up listener before triggering). This avoids stray file chooser dialogs in the first place.
- If a dialog is somehow left open, use `playwright-cli dialog-dismiss` to close it, or close the browser tab and re-open with `playwright-cli goto`.

## Tips & Troubleshooting

### Download Fails with "require is not defined"
- `require('fs')` is NOT available inside `page.evaluate()` (browser context).
- Use the download event pattern shown above instead of trying to write files from the browser.

### File Chooser Dialog Not Appearing
- The "Load from your computer" menu item internally clicks a hidden `<input type="file">`. If the menu is not found, verify the editor is fully loaded.
- Use `playwright-cli snapshot` to check the current page state.

### Project Not Fully Loading After Upload
- Wait at least 2-3 seconds after loading for assets (costumes, sounds) to initialize.
- After loading, re-find the VM reference as `loadProject` may invalidate previous references:

  ```bash
  playwright-cli run-code "$(cat <<'EOF'
  async page => await page.evaluate(() => {
    const el = document.querySelector('canvas');
    const key = Object.keys(el).find(k =>
      k.startsWith('__reactFiber') || k.startsWith('__reactInternalInstance')
    );
    let fiber = el[key];
    while (fiber) {
      if (fiber.memoizedProps?.vm) { window.vm = fiber.memoizedProps.vm; break; }
      fiber = fiber.return;
    }
    return window.vm ? 'VM reconnected' : 'VM not found';
  })
  EOF
  )"
  ```

### Verifying Project Contents After Load

```bash
playwright-cli run-code "$(cat <<'EOF'
async page => await page.evaluate(() => {
  const vm = window.vm;
  const sprites = vm.runtime.targets.map(t => t.sprite.name);
  const stage = vm.runtime.targets.find(t => t.isStage);
  const backdrops = stage.sprite.costumes.map(c => c.name);
  return { sprites, backdrops };
})
EOF
)"
```
