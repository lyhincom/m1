# Fast image upload to a draw.io diagram on Google Drive

Measured: **17.6 s, 3 tool calls, no duplicates.**

## What's used

| Tool | Source | Role |
|---|---|---|
| **Playwright MCP** (`@playwright/mcp`, user scope) | MCP server: `cmd /c npx -y @playwright/mcp@latest` | Drives a real Chromium window that can be given local files and can run scripts in the page |
| `Bash` | Claude Code built-in | Copies the image into the folder Playwright is allowed to read |
| `mcp__playwright__browser_drop` | Playwright MCP | Drops the file onto the draw.io canvas; draw.io embeds it as `image=data:…` |
| `mcp__playwright__browser_run_code_unsafe` | Playwright MCP | One script: place the image and wait for the Drive save |
| `mcp__playwright__browser_evaluate` | Playwright MCP | One-time setup after page load: get a handle on draw.io's editor |

Not used (and why):

- **Built-in browser pane (Claude_Browser)**: can't hand local files to the page. draw.io's URL import goes through its server ("File not found"), and the page can't fetch from localhost.
- **drawio MCP** (`mcp__drawio__*`): opens/generates diagrams, doesn't edit a live Google Drive file.
- **+ → Image… → Open** (file picker): Playwright MCP intercepts the chooser, which led to a double insert and a multi-hour hang. Dropping the file avoids the picker.
- **Screenshots**: slow and not needed. State is read straight from the editor.

## One-time setup per browser session

1. Open the file in Playwright: `browser_navigate` to `https://app.diagrams.net/#G<fileId>`.
2. If "Authorization required" (Google Drive) appears, **you** click Authorize and sign in to Google in the Playwright window. It has its own profile, separate from your Chrome. If you can't find the window, Claude brings it to the front with a CDP call (`Browser.setWindowBounds`).
3. Hook the editor (draw.io has no global). In `browser_evaluate`:
   ```js
   () => { for (const n of ['updateActionStates','getCurrentFile']) { const o = EditorUi.prototype[n];
     EditorUi.prototype[n] = function(){ window.__ui = this; return o.apply(this, arguments); }; } }
   ```
   then click once on empty canvas so `window.__ui` gets set. Repeat after any page reload.

## Per image (3 calls)

1. **Bash**: copy the image to `C:\000. Projects\[project folder]\.playwright-mcp\` (Playwright only reads under the repo root).
2. **browser_drop**: target `.geDiagramContainer`, path = the copied file.
3. **browser_run_code_unsafe**: find the new embedded image. It's the last image cell in its parent's children, since boxes no longer share one x (see the left edge rule). Set width 763.5, y = lowest image bottom + 30, and height from its own aspect ratio. Set x using the left edge rule below, then clear the selection. Optionally `waitForFunction(() => !__ui.getCurrentFile().isModified())` and expect "All changes saved in Drive". Skip this wait for max speed: autosave still saves within ~1–2 s, as long as the status doesn't say "Not connected".

Afterwards: delete the copied file.

## Title rule

**Placement:** a new image goes under the named title bar, 30 px below the last image in that section, not just below the lowest image on the page. If more sections follow, shift everything below down by the new image's height + 30.

**How a title bar looks** (copy of the existing "Customer Refund Request Flow" and "title 2" bars):

| Property | Value |
|---|---|
| Shape | plain rectangle, square corners |
| Size | 540 × 60 |
| x | 290 (centered over the page) |
| Fill | `#FFFF66` (yellow) |
| Border | `#000000`, 1 px |
| Text | 12 pt, centered horizontally and vertically, wraps |
| Spacing | ~30 px above (from the previous image) and 30 px below (to the first image) |

Style string:

```
rounded=0;whiteSpace=wrap;html=1;fillColor=#FFFF66;strokeColor=#000000;fontSize=12;align=center;verticalAlign=middle;
```

To add a new title bar: insert a vertex with this style and size, x=290, y = last image bottom + 30, then put that section's first image at title bottom + 30.

## Left edge rule

The **visible picture content** of every image starts at x=40, not the image box. Screenshots carry their own white margin on the left (0–28 px at width 763.5), so equal box x still looks uneven.

For each image, in the same script:
1. Load its `image=data:` source into an `Image`, draw it on a canvas, and scan for the first column that has a non-white pixel (any channel < 235, alpha > 20). That's `leftPx`.
2. `offset = leftPx * geometry.width / naturalWidth`.
3. `geometry.x = 40 - offset`, rounded to 0.1.

Apply it to the new image every time, and to all images if any were added another way.

## Speed rules

- No screenshots unless asked.
- No fixed waits; wait on the condition (image present, file not modified).
- Batch placement and the save check in one script.
- Copy into `.playwright-mcp` first. A temp/scratchpad path gets "outside allowed roots" and costs extra calls.
