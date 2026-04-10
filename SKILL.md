---
name: showroom-to-figma
description: Use when the user provides a prototype (Showroom URL, GitHub URL, or local file path) and wants native Figma screens built — triggers on "build in figma", "showroom to figma", "convert prototype to figma", "prototype to figma", or any HTML prototype reference with Figma as the target
argument-hint: "<source> [--size 1440x900]"
allowed-tools: [WebFetch, Read, Glob, Grep, Bash, mcp__figma-api__use_figma, mcp__figma-api__generate_figma_design, mcp__figma-api__search_design_system, mcp__figma-api__get_screenshot, mcp__figma-api__get_metadata, mcp__figma-api__whoami, mcp__figma-api__create_new_file]
---

# /showroom-to-figma

Build native Figma screens from an HTML prototype using Copilot Web components, Copilot Foundations tokens, and the official Figma MCP. No hardcoded keys — everything discovered dynamically.

## CRITICAL RULES — Read Before Anything Else

1. **Component-first, always.** NEVER hand-build a UI element that exists as a Copilot Web component. Every button MUST be a `button` component instance. Every text input MUST be a `text-short` component instance. Every avatar MUST be an `Avatar` component instance. Every loading indicator MUST be a `Spinner` component instance. If you write `figma.createFrame()` for something a library component covers, you are doing it wrong.

2. **Frame size is sacred.** The default frame size is **1440x900**. This is the Figma frame size — NOT the prototype's internal viewport. The prototype might use 960px or 375px internally, but you scale the layout to fill the Figma frame. Only change frame size if the user explicitly passes `--size`.

3. **Do NOT call `setCurrentPageAsync`.** It does not persist across separate `use_figma` calls and will cause frames to be lost. Instead, work on the default page (Page 1 / 0:1) and rename it. Or create a new page and reference it by ID in subsequent calls using `figma.root.children.find(p => p.id === "...")`.

4. **One `use_figma` call per screen.** Each screen gets its own call. Import tokens and components at the top of every call — they don't persist across calls.

5. **NEVER create spacer frames.** No `figma.createFrame()` with `name: "spacer"`. All spacing MUST be achieved with `itemSpacing` bound to Copilot Foundations spacing tokens via `frame.setBoundVariable("itemSpacing", spacingVar)`, or with `paddingTop`/`paddingBottom`/etc. bound to spacing tokens. If you need different spacing between specific children, group them in a wrapper frame with its own `itemSpacing`.

6. **ALL text MUST use Copilot Foundations text styles.** Never create raw text with manual `fontSize`/`fontName`. Every text node must have a `textStyleId` from Copilot Foundations. In Phase 3, discover all text styles (heading, body, label sizes). In Phase 6, apply them: `t.textStyleId = style.id`. The only exception is text inside component instances (buttons, inputs) — those inherit their own styles.

7. **ALL spacing MUST use Copilot Foundations spacing tokens.** Never use hardcoded pixel values for `itemSpacing`, `paddingTop`, `paddingBottom`, `paddingLeft`, `paddingRight`, or gaps. Always bind to a spacing variable: `frame.setBoundVariable("paddingTop", space400)`.

## Arguments

Parse `$ARGUMENTS`:
- First positional arg: prototype source (required) — one of:
  - **Showroom URL** — `https://showroom.mgt.dpty.io/rgavan/my-proto.html`
  - **GitHub URL** — `https://github.com/DeputyApp/repo/blob/main/proto.html`
  - **Local file path** — `~/projects/proto.html` or `/absolute/path/to/proto.html`
- `--size WxH`: Frame size (default: **1440x900** — always use this unless overridden)

## Phase 1: Setup Check

Test the official Figma MCP with a valid file key. Use `mcp__figma-api__whoami` to confirm authentication, then search for a component:
```
mcp__figma-api__search_design_system({ query: "button", fileKey: "<valid-file-key>", includeComponents: true })
```

If whoami fails:
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
- Text inputs (email, password, username fields) — note the label text above each input and placeholder text inside
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

### Components — MANDATORY searches

For EVERY interactive UI element type found in Phase 2, search Copilot Web. You must find a component match or explicitly mark it as "hand-build."

**Required searches (run ALL of these):**
- `query: "button"` — for all buttons. Filter to `libraryName: "Copilot Web"`.
- `query: "text-short"` — for text input fields. Filter to `libraryName: "Copilot Web"`.
- `query: "text-long"` — for textareas. Filter to `libraryName: "Copilot Web"`.
- `query: "avatar"` — for avatars/identity circles. Filter to `libraryName: "Copilot Web"`.
- `query: "spinner loading"` — for loading indicators. Filter to `libraryName: "Copilot Web"`.
- `query: "banner alert notification"` — for toasts/banners. Filter to `libraryName: "Copilot Web"`.
- `query: "divider"` — for dividers. Filter to `libraryName: "Copilot Web"`.

### Component Inspection — MANDATORY

After finding component keys, import EACH component set and inspect its variants AND property definitions:

```javascript
const set = await figma.importComponentSetByKeyAsync(key);
// Get variant names
const variants = set.children.map(c => ({ name: c.name, key: c.key }));
// Get property definitions (on the SET, not individual variants)
const defs = set.componentPropertyDefinitions;
// Returns: { "Label#10072:106": { type: "TEXT", defaultValue: "Button" }, ... }
```

Record the full property key strings (e.g., `"Label#10072:106"`, `"Placeholder#12271:18"`) — you need these exact strings for `setProperties()` in Phase 6.

### Color Variables
Search Copilot Foundations for semantic color tokens:
```
query: "text color default secondary accent inverse success"
query: "surface color default accent success"
query: "border color default accent success"
```
Filter to `libraryName: "Copilot Foundations"`, `variableType: "COLOR"`. Match prototype hex values to semantic tokens.

### Text Styles — MANDATORY

You MUST discover and use Copilot Foundations text styles for ALL text. Run multiple searches to find all styles:

```
query: "heading"     — for h1-h6, titles
query: "body"        — for paragraphs, descriptions
query: "label"       — for form labels, captions
query: "link"        — for link text
query: "semibold"    — for emphasized body text
```

Filter each to `libraryName: "Copilot Foundations"`, `styleType: "TEXT"`.

After discovering styles, import them via `use_figma` to get the full style objects:
```javascript
const style = await figma.importStyleByKeyAsync(styleKey);
// style.id — use this for textStyleId
// style.fontName — use for loadFontAsync
// style.fontSize — for reference
```

Create a mapping from prototype font sizes to Foundation styles:
- 24-28px bold → heading/xl or heading/lg
- 20px semibold → heading/md
- 15px regular → body/md/regular
- 14px regular → body/sm/regular
- 13px medium → label/sm or body/sm/semibold
- 12px regular → body/xs/regular
- Link text → body/md/link or body/sm/link

When creating text in Phase 6, ALWAYS apply the style:
```javascript
const t = figma.createText();
await figma.loadFontAsync(headingLgStyle.fontName);
t.textStyleId = headingLgStyle.id;
t.characters = "My Heading";
```

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

Show the user a plan that includes an explicit **component mapping**:

```
Found N screens:
1. [Screen name] — [element summary]
2. ...

Component mapping:
- Buttons → Copilot Web `button` (key: xxx) — Primary for CTAs, Default for secondary, Text for links
- Text inputs → Copilot Web `text-short` (key: xxx) — with labels and placeholders
- Avatars → Copilot Web `Avatar` (key: xxx) — Text type for initials
- Spinners → Copilot Web `Spinner` (key: xxx)
- [anything not matched] → hand-built with Foundations tokens

Color variables: [count] mapped
Text styles: [count] mapped
Spacing tokens: [count] mapped

Frame size: [WxH] (default 1440x900)
Target: [will ask next]

Proceed?
```

Wait for confirmation. User can adjust screen names, remove screens, change size.

## Phase 5: Select Figma Target

1. Call `mcp__figma-api__whoami` to get available plans.
2. Ask: new file or existing file?
   - **New file:** Use `mcp__figma-api__create_new_file` with the appropriate planKey.
   - **Existing file:** Ask for Figma URL, extract fileKey.
3. Record the `fileKey`.
4. **Page setup:** Rename the default page (0:1) via `use_figma`:
```javascript
figma.currentPage.name = pageName;
```
Do NOT use `setCurrentPageAsync` or `figma.createPage()` — frames created after page switching are lost.

## Phase 6: Build Screens

One `use_figma` call per screen. **IMPORTANT:** Every call must re-import tokens and components at the top — they don't persist between calls.

### Step 1: Import components and tokens (every call)

```javascript
// Components — import at the top of EVERY use_figma call
const buttonSet = await figma.importComponentSetByKeyAsync(BUTTON_KEY);
const textShortSet = await figma.importComponentSetByKeyAsync(TEXT_SHORT_KEY);
const avatarSet = await figma.importComponentSetByKeyAsync(AVATAR_KEY);
const spinnerComp = await figma.importComponentByKeyAsync(SPINNER_KEY);

// Tokens — import at the top of EVERY use_figma call
const textColor = await figma.variables.importVariableByKeyAsync(TEXT_COLOR_KEY);
// ... etc

// Text styles — import ALL discovered styles from Copilot Foundations
const headingXlStyle = await figma.importStyleByKeyAsync(HEADING_XL_KEY);
const headingLgStyle = await figma.importStyleByKeyAsync(HEADING_LG_KEY);
const headingMdStyle = await figma.importStyleByKeyAsync(HEADING_MD_KEY);
const bodyMdRegularStyle = await figma.importStyleByKeyAsync(BODY_MD_REGULAR_KEY);
const bodySmRegularStyle = await figma.importStyleByKeyAsync(BODY_SM_REGULAR_KEY);
const labelSmStyle = await figma.importStyleByKeyAsync(LABEL_SM_KEY);
// ... import every text style you discovered in Phase 3

// Fonts — load from the styles themselves, plus SF Pro for Copilot components
for (const style of [headingXlStyle, headingLgStyle, headingMdStyle, bodyMdRegularStyle, bodySmRegularStyle, labelSmStyle]) {
  await figma.loadFontAsync(style.fontName);
}
await figma.loadFontAsync({ family: "SF Pro", style: "Medium" });
await figma.loadFontAsync({ family: "SF Pro", style: "Regular" });
await figma.loadFontAsync({ family: "SF Pro", style: "Semibold" });
```

### Step 2: Component helper functions

Define these helpers BEFORE building any screen content:

```javascript
// Variant finder
function getVariant(set, props) {
  return set.children.find(c =>
    Object.entries(props).every(([k, v]) => c.name.includes(`${k}=${v}`))
  );
}

// Button — ALWAYS use this for any clickable action
function makeButton(parent, label, type = "Primary") {
  const v = getVariant(buttonSet, { Type: type, State: "Default", Usage: "Default" });
  const inst = v.createInstance();
  parent.appendChild(inst);
  inst.setProperties({ "Label#XXXX": label }); // Use actual property key from Phase 3
  // Do NOT set layoutSizingHorizontal = "FILL" unless the button should be full-width
  return inst;
}

// Text input — ALWAYS use this for any input field
// The text-short component includes a built-in label (Header) — USE IT
function makeInput(parent, { label, placeholder, filled = false } = {}) {
  const content = filled ? "Filled" : "Empty";
  const v = getVariant(textShortSet, { State: "Default", Content: content });
  const inst = v.createInstance();
  parent.appendChild(inst);
  inst.layoutSizingHorizontal = "FILL";
  // Set placeholder and label via properties discovered in Phase 3
  // Use setProperties with the exact keys you discovered
  return inst;
}

// Avatar
function makeAvatar(parent, initials, size = "Medium") {
  const v = getVariant(avatarSet, { Size: size, Type: "Text", Colour: "Default" });
  if (!v) return null; // Some size+colour combos may not exist
  const inst = v.createInstance();
  parent.appendChild(inst);
  inst.setProperties({ "✏️ Label#XXXX": initials }); // Use actual property key
  return inst;
}

// Spinner
function makeSpinner(parent) {
  const inst = spinnerComp.createInstance();
  parent.appendChild(inst);
  return inst;
}
```

### Step 3: Layout helpers (for non-component parts only)

These are ONLY for structural layout frames, NOT for interactive UI elements:

```javascript
// Color binding
function bindFill(node, variable) {
  node.fills = [figma.variables.setBoundVariableForPaint(
    { type: "SOLID", color: { r: 0.5, g: 0.5, b: 0.5 } },
    "color", variable
  )];
}
function bindStroke(node, variable) {
  node.strokes = [figma.variables.setBoundVariableForPaint(
    { type: "SOLID", color: { r: 0.5, g: 0.5, b: 0.5 } },
    "color", variable
  )];
}

// Spacing — ALWAYS use tokens, NEVER hardcoded values
frame.setBoundVariable("itemSpacing", space300);    // gap between children
frame.setBoundVariable("paddingTop", space600);     // padding
frame.setBoundVariable("paddingBottom", space600);
frame.setBoundVariable("paddingLeft", space400);
frame.setBoundVariable("paddingRight", space400);

// Border radius — ALWAYS use tokens
frame.setBoundVariable("topLeftRadius", borderRadius200);
frame.setBoundVariable("topRightRadius", borderRadius200);
frame.setBoundVariable("bottomLeftRadius", borderRadius200);
frame.setBoundVariable("bottomRightRadius", borderRadius200);

// Text — ALWAYS use text styles from Copilot Foundations
const t = figma.createText();
await figma.loadFontAsync(bodyMdRegularStyle.fontName);
t.textStyleId = bodyMdRegularStyle.id;  // REQUIRED — never skip this
t.characters = "Body text";
bindFill(t, textColor);

// Child sizing — MUST be AFTER appendChild
parent.appendChild(child);
child.layoutSizingHorizontal = "FILL";  // fails if set before appendChild
child.layoutSizingVertical = "HUG";
```

**BANNED PATTERNS — never do these:**
```javascript
// ❌ NEVER create spacer frames
const spacer = figma.createFrame(); spacer.resize(1, 16); // WRONG

// ❌ NEVER use hardcoded spacing
frame.itemSpacing = 16; // WRONG — use setBoundVariable

// ❌ NEVER use hardcoded padding
frame.paddingTop = 40; // WRONG — use setBoundVariable

// ❌ NEVER create text without a text style
t.fontSize = 14; t.fontName = { family: "Inter", style: "Regular" }; // WRONG — use textStyleId
```

**Instead, control spacing by grouping content into auto-layout frames with different `itemSpacing` tokens.** For example, if a form has 8px between label and input but 16px between input groups, use nested frames:
```javascript
const formGroup = figma.createFrame();
formGroup.layoutMode = "VERTICAL";
formGroup.setBoundVariable("itemSpacing", space100); // 8px between label and input
// ... add label and input

const formContainer = figma.createFrame();
formContainer.layoutMode = "VERTICAL";
formContainer.setBoundVariable("itemSpacing", space300); // 16px between groups
// ... add multiple formGroups
```

### Step 4: Build each screen

Root frame pattern:
```javascript
const frame = figma.createFrame();
frame.name = screenName;
frame.resize(width, height); // Use the frame size from arguments (default 1440x900)
frame.layoutMode = "VERTICAL";
frame.primaryAxisSizingMode = "FIXED";
frame.counterAxisSizingMode = "FIXED";
frame.clipsContent = true;
```

**Key sizing rules:**
- Root frames: always use the `--size` value (default 1440x900)
- Buttons: use `layoutSizingHorizontal = "HUG"` by default. Only use `"FILL"` if the button is explicitly full-width in the prototype (e.g., a login form submit button)
- Text inputs (`text-short`): use `layoutSizingHorizontal = "FILL"` — inputs should fill their container
- Layout frames: set sizing based on prototype layout
- Content containers: constrain `maxWidth` by setting a fixed width on the container, not by stretching children

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

Component text overrides often do not render visually on the first pass (the property value is set correctly but the text node rendering lags). Run a second pass per screen to force text updates:

```javascript
// Load SF Pro — Copilot components use it, not Inter
await figma.loadFontAsync({ family: "SF Pro", style: "Medium" });
await figma.loadFontAsync({ family: "SF Pro", style: "Regular" });
await figma.loadFontAsync({ family: "SF Pro", style: "Semibold" });

// Buttons: setProperties + direct .characters on nested text node
const btn = frame.findOne(n => n.name === "Button Name" && n.type === "INSTANCE");
btn.setProperties({ "Label#...": "desired text" });
const textNode = btn.findOne(n => n.type === "TEXT" && n.name === "label");
if (textNode) {
  await figma.loadFontAsync(textNode.fontName); // Will be SF Pro, not Inter
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

// Avatars: setProperties + direct text node
const avatar = frame.findOne(n => n.type === "INSTANCE" && n.componentProperties?.["✏️ Label#..."]);
avatar.setProperties({ "✏️ Label#...": "MB" });
const avText = avatar.findOne(n => n.type === "TEXT");
if (avText) {
  await figma.loadFontAsync(avText.fontName);
  avText.characters = "MB";
}
```

To discover the property keys for any component instance:
```javascript
const props = instance.componentProperties;
// Returns: { "Label#10072:106": { type: "TEXT", value: "Button" }, ... }
```

## Phase 7b: Component Verification

After overrides, verify every screen:

```javascript
const frame = figma.currentPage.findOne(n => n.name === screenName);
const instances = frame.findAll(n => n.type === "INSTANCE");
const buttons = instances.filter(n => n.componentProperties?.["Label#..."]);
const inputs = instances.filter(n => /* match text-short */);
const avatars = instances.filter(n => n.componentProperties?.["✏️ Label#..."]);

// Verify counts match expected
// Verify no hand-built frames exist where components should be
const handBuiltInputs = frame.findAll(n =>
  n.type === "FRAME" && n.name?.includes("Input") && !n.componentProperties
);
if (handBuiltInputs.length > 0) {
  // ERROR: found hand-built inputs — replace with text-short instances
}
```

## Phase 8: Verify & Report

1. `mcp__figma-api__get_screenshot` for each screen.
2. Present screenshots with Figma links:
```
Built N screens on page "Page Name":
1. Screen Name — [screenshot]
2. ...
```
3. Note: The screenshot API may show stale text on component overrides. The properties ARE set correctly — verify by querying `componentProperties` values. Labels render correctly when you open the file in Figma Desktop.
4. Ask if anything needs adjustment.

## What This Skill Does NOT Do

- No prototyping/interaction wiring
- No navigation components (user adds manually)
- No external image downloads (placeholder rectangles for hosted images)
- No modification of existing Figma content (creates new frames on a new page only)
- No `mcp__figma__*` tools (WebSocket community plugin) — only official `mcp__figma-api__*`
