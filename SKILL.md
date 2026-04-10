---
name: showroom-to-figma
description: Use when the user provides a prototype (Showroom URL, GitHub URL, or local file path) and wants native Figma screens built — triggers on "build in figma", "showroom to figma", "convert prototype to figma", "prototype to figma", or any HTML prototype reference with Figma as the target
argument-hint: "<source> [--size 1440x900]"
allowed-tools: [WebFetch, Read, Glob, Grep, Bash, mcp__figma-api__use_figma, mcp__figma-api__generate_figma_design, mcp__figma-api__search_design_system, mcp__figma-api__get_screenshot, mcp__figma-api__get_metadata]
---

# /showroom-to-figma

Build native Figma screens from an HTML prototype using Copilot Web components, Copilot Foundations tokens, and the official Figma MCP. No hardcoded keys — everything discovered dynamically.

## Arguments

Parse `$ARGUMENTS`:
- First positional arg: prototype source (required) — one of:
  - **Showroom URL** — `https://showroom.mgt.dpty.io/rgavan/my-proto.html`
  - **GitHub URL** — `https://github.com/DeputyApp/repo/blob/main/proto.html`
  - **Local file path** — `~/projects/proto.html` or `/absolute/path/to/proto.html`
- `--size WxH`: Frame size (default: 1440x900)

## Phase 1: Setup Check

Test the official Figma MCP:
```
mcp__figma-api__search_design_system({ query: "button", fileKey: "any", includeComponents: true })
```

If it fails:
> "You need the official Figma MCP. In Claude Code, run `/mcp` and connect the Figma integration. Authenticate with your Figma account. Then try again."

Stop until MCP responds.

## Phase 2: Resolve & Analyze Prototype

Resolve the source to local HTML content. Try in order:

### Local file path
If the argument is a file path (starts with `/`, `~/`, or `./`):
```bash
# Expand ~ and read directly
cat ~/path/to/proto.html
```

### GitHub URL
If the argument contains `github.com`:
```bash
# Extract org/repo/branch/path from URL
# https://github.com/DeputyApp/repo/blob/main/path/to/proto.html
gh api repos/DeputyApp/repo/contents/path/to/proto.html?ref=main --jq '.content' | base64 -d
```
If the file is too large for the API (>1MB), clone and read:
```bash
gh repo clone DeputyApp/repo /tmp/proto-repo -- --depth 1
cat /tmp/proto-repo/path/to/proto.html
```

### Showroom URL
If the argument contains `showroom.mgt.dpty.io`:
1. Try `WebFetch` first — works if user is not behind VPN proxy.
2. If WebFetch returns Pritunl/proxy HTML instead of the prototype, search for a local copy:
   - `~/.superpowers/brainstorm/*/content/*.html` — match by filename
   - Ask the user to download it: `! curl -o ~/proto.html "URL"`
3. If no local copy found, ask the user to provide a GitHub URL or local path instead.

### Once HTML is resolved, parse to identify:

**Screens:** Look for `.screen` classes, `id="screen*"` patterns, `data-screen` attributes, or major structural sections. Record each screen's purpose from comments, headings, or content.

**UI elements per screen — categorize:**
- Headings (h1-h6, large bold text)
- Body text (paragraphs, descriptions)
- Buttons (primary CTAs, secondary, SSO/social with icons)
- Text inputs (email, password, username fields)
- Links (inline, standalone)
- Dividers (hr, "or" separators)
- Identity cards (avatar + name + email)
- Spinners/loading indicators
- Success states (checkmarks, confirmation messages)
- Toast/banner messages
- Cards/tiles
- Inline SVGs (extract the SVG source for each icon)

**CSS values:** Extract hex colors, font stacks, font sizes/weights, spacing values (padding, margin, gap), border-radius values, gradients.

## Phase 3: Design System Discovery

Search Copilot Web and Copilot Foundations for matches. **Do not hardcode keys.**

### Components
For each UI pattern found, search `mcp__figma-api__search_design_system`:
- `query: "button"` — for buttons
- `query: "input text field"` — for text inputs
- `query: "spinner loading"` — for spinners
- `query: "toast banner alert"` — for notifications
- `query: "avatar"` — for avatars
- etc.

Filter results to `libraryName: "Copilot Web"`. Record component keys.

Import each component set via `use_figma` and inspect variants:
```javascript
const set = await figma.importComponentSetByKeyAsync(key);
const variants = set.children.map(c => ({ name: c.name, key: c.key }));
```

### Color Variables
Search Copilot Foundations for semantic color tokens:
```
query: "text color default secondary accent inverse success"
query: "surface color default accent success"
query: "border color default accent success"
```
Filter to `libraryName: "Copilot Foundations"`, `variableType: "COLOR"`. Match prototype hex values to semantic tokens.

### Text Styles
Search Copilot Foundations:
```
query: "heading xl lg md sm"
query: "body md sm regular semibold link"
```
Filter to `libraryName: "Copilot Foundations"`, `styleType: "TEXT"`. Match prototype font-size + weight combos.

### Spacing Tokens
Search Copilot Foundations:
```
query: "spacing space gap padding"
```
Filter to `variableType: "FLOAT"`, `variableCollectionName: "Spacing"`. Match prototype pixel values to nearest token.

### Border Radius Tokens
Search Copilot Foundations:
```
query: "radius corner border-radius"
```
Filter to `variableType: "FLOAT"`. Match prototype border-radius values.

## Phase 4: Present Plan

Show the user:
```
Found N screens:
1. [Screen name] — [element summary]
2. ...

Copilot Web components: [list with names]
Color variables: [count] mapped
Text styles: [count] mapped
Spacing tokens: [count] mapped

Target: [will ask next]
Frame size: [WxH]

Proceed?
```

Wait for confirmation. User can adjust screen names, remove screens, change size.

## Phase 5: Select Figma Target

1. Call `mcp__figma-api__generate_figma_design` with no `outputMode` to get recent files list.
2. User picks a file. Record `fileKey`.
3. Ask for page name (default: derive from prototype title).
4. Create the page via `use_figma`:
```javascript
const page = figma.createPage();
page.name = pageName;
await figma.setCurrentPageAsync(page);
```

## Phase 6: Build Screens

One `use_figma` call per screen. Follow these patterns exactly:

### Root Frame
```javascript
const frame = figma.createFrame();
frame.name = screenName;
frame.resize(width, height);
frame.layoutMode = "VERTICAL"; // or HORIZONTAL for split layouts
frame.primaryAxisSizingMode = "FIXED";
frame.counterAxisSizingMode = "FIXED";
frame.clipsContent = true;
```

### Color Variable Binding
```javascript
function bindFill(node, variable) {
  node.fills = [figma.variables.setBoundVariableForPaint(
    { type: "SOLID", color: { r: 0.5, g: 0.5, b: 0.5 } },
    "color", variable
  )];
}
// For strokes:
node.strokes = [figma.variables.setBoundVariableForPaint(
  { type: "SOLID", color: { r: 0.5, g: 0.5, b: 0.5 } },
  "color", variable
)];
```

### Text with Library Style
```javascript
async function createText(text, style, colorVar) {
  const t = figma.createText();
  await figma.loadFontAsync(style.fontName);
  t.fontName = style.fontName;
  t.fontSize = style.fontSize;
  t.textStyleId = style.id;
  t.characters = text;
  if (colorVar) bindFill(t, colorVar);
  return t;
}
```

### Spacing Token Binding
```javascript
frame.setBoundVariable("itemSpacing", spacingVar);
frame.setBoundVariable("paddingTop", spacingVar);
frame.setBoundVariable("paddingBottom", spacingVar);
frame.setBoundVariable("paddingLeft", spacingVar);
frame.setBoundVariable("paddingRight", spacingVar);
```

### Border Radius Token Binding
```javascript
frame.setBoundVariable("topLeftRadius", radiusVar);
frame.setBoundVariable("topRightRadius", radiusVar);
frame.setBoundVariable("bottomLeftRadius", radiusVar);
frame.setBoundVariable("bottomRightRadius", radiusVar);
```

### Child Sizing — MUST be AFTER appendChild
```javascript
parent.appendChild(child);
child.layoutSizingHorizontal = "FILL";  // fails if set before appendChild
child.layoutSizingVertical = "HUG";
```

### Component Instances
```javascript
function getVariant(set, props) {
  return set.children.find(c =>
    Object.entries(props).every(([k, v]) => c.name.includes(`${k}=${v}`))
  );
}
const variant = getVariant(buttonSet, { Type: "Primary", State: "Default" });
const instance = variant.createInstance();
parent.appendChild(instance);
instance.layoutSizingHorizontal = "FILL";
```

### SVG Icons
```javascript
const svgNode = figma.createNodeFromSvg(svgString);
svgNode.resize(18, 18);
parent.insertChild(0, svgNode);
```

### Valid counterAxisAlignItems values
`"MIN"`, `"MAX"`, `"CENTER"`, `"BASELINE"` — NOT "STRETCH"

## Phase 7: Fix Component Overrides

**CRITICAL: This MUST be a separate `use_figma` call from Phase 6.**

Component text overrides do not stick on the first pass. Run a second pass per screen:

```javascript
// Buttons: setProperties + direct .characters
const btn = frame.findOne(n => n.name === "Button Name" && n.type === "INSTANCE");
// First: set via component properties
btn.setProperties({ "Label#...": "desired text" });
// Second: also set directly on nested text node
const textNode = btn.findOne(n => n.type === "TEXT" && n.name === "label");
if (textNode) {
  await figma.loadFontAsync(textNode.fontName);
  textNode.characters = "desired text";
}

// Text Fields: set properties + nested label instance
field.setProperties({
  "Placeholder#...": "placeholder text",
  "Support text#...": false,
  "Prefix#...": false,
  "Suffix#...": false,
});
// Find nested input header label
const header = field.findOne(n => n.name === "Header");
const labelInst = header?.findOne(n => n.name === "Label" && n.type === "INSTANCE");
if (labelInst) {
  const props = labelInst.componentProperties;
  const textProp = Object.keys(props).find(k => props[k].type === "TEXT");
  if (textProp) labelInst.setProperties({ [textProp]: "Label Text" });
}
```

To discover the property keys for any component instance:
```javascript
const props = instance.componentProperties;
// Returns: { "Label#10072:106": { type: "TEXT", value: "Button" }, ... }
```

## Phase 8: Verify & Report

1. `mcp__figma-api__get_screenshot` for each screen.
2. Present screenshots with Figma links:
```
Built N screens on page "Page Name":
1. Screen Name — [screenshot]
2. ...
```
3. Ask if anything needs adjustment.

## What This Skill Does NOT Do

- No prototyping/interaction wiring
- No navigation components (user adds manually)
- No external image downloads (placeholder rectangles for hosted images)
- No modification of existing Figma content (creates new frames on a new page only)
- No `mcp__figma__*` tools (WebSocket community plugin) — only official `mcp__figma-api__*`
