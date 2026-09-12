# Step-by-Step Guide: Building the Heuristic Score Figma Plugin

## Prerequisites

Before you begin, install these on your machine:

| Tool | Why You Need It | Download |
|------|----------------|----------|
| **Figma Desktop App** | Plugin dev only works on desktop, not browser | [figma.com/downloads](https://www.figma.com/downloads/) |
| **Node.js (v18+)** | Runs the TypeScript compiler and package manager | [nodejs.org](https://nodejs.org/) |
| **VS Code (or Cursor)** | Code editor for writing plugin code | Already have it |

Verify Node.js is installed by running in your terminal:
```bash
node --version
npm --version
```

---

## Part 1: Create the Plugin Scaffold in Figma

### Step 1: Create a new plugin

1. Open the **Figma Desktop App** and log in.
2. Open any existing design file (or create a new one).
3. Go to the top menu: **Plugins → Development → New plugin...**
4. In the modal that appears:
   - Select **"Figma design"** (not FigJam or Slides)
   - Give your plugin a name: `Heuristic Screen Score`
5. On the next screen, select the **"Custom UI"** template (this gives you a UI panel — you need this for the questionnaire interface).
6. Choose a save location on your computer (e.g., `~/Desktop/ai-work-2026/figma-plugins/heuristic-score/`).
7. Click **Save** — Figma creates a folder with starter files.

### Step 2: Understand what Figma generated

Open the saved folder in VS Code/Cursor. You'll see:

```
heuristic-score/
├── manifest.json      ← Plugin metadata (name, permissions, entry points)
├── code.ts            ← Main thread code (accesses Figma API — nodes, selection, canvas)
├── code.js            ← Compiled output (auto-generated, don't edit)
├── ui.html            ← UI thread (HTML/CSS/JS — the panel the user interacts with)
├── package.json       ← Node dependencies
└── tsconfig.json      ← TypeScript configuration
```

**Two contexts — this is critical to understand:**

| Context | File | Can Access | Cannot Access |
|---------|------|-----------|---------------|
| **Main thread** | `code.ts` | Figma API (nodes, selection, create frames, export) | Browser APIs (DOM, fetch, localStorage) |
| **UI thread** | `ui.html` | Browser APIs (DOM, HTML rendering, fetch) | Figma API directly |

They communicate via `postMessage`:
- UI → Main: `parent.postMessage({ pluginMessage: data }, '*')`
- Main → UI: `figma.ui.postMessage(data)`

---

## Part 2: Set Up the Development Environment

### Step 3: Install dependencies

Open a terminal in your plugin folder and run:

```bash
npm install
```

This installs `@figma/plugin-typings` (TypeScript type definitions for the Figma Plugin API).

### Step 4: Set up TypeScript compilation

Your `tsconfig.json` should already be configured. To compile:

```bash
# One-time build
npx tsc

# Watch mode (auto-recompiles on save) — use this during development
npx tsc --watch
```

Alternatively, in VS Code: **Terminal → Run Build Task → npm: watch**

### Step 5: Run the plugin for the first time

1. Go back to the Figma Desktop App.
2. Open a design file with some frames.
3. Go to **Plugins → Development → Heuristic Screen Score**
4. The plugin UI panel should appear (showing the default template content).

**Hot reloading:** After you edit code and it recompiles, close and re-open the plugin in Figma to see changes. You can also enable "Use developer VM" in Figma's plugin dev settings for faster iteration.

---

## Part 3: Plugin Architecture for Your Use Case

### Step 6: Plan the plugin flow

```
┌─────────────────────────────────────────────────────────┐
│                    USER FLOW                              │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  1. User selects frame(s) on canvas                      │
│  2. User opens plugin                                    │
│  3. Plugin detects selection → validates frames          │
│     - Filters out child frames of selected parents       │
│  4. Plugin shows CONFIGURATION screen (dropdowns)        │
│  5. User fills config → clicks "Start Scoring"           │
│  6. Plugin shows CHECKLIST (55 questions, pillar by      │
│     pillar, with conditional visibility)                  │
│  7. User answers all questions                           │
│  8. Plugin computes score → shows RESULTS screen         │
│  9. User can:                                            │
│     a. "Export as Image" → renders scorecard frame       │
│        next to the evaluated frame on canvas             │
│     b. "Get Feedback" → shows bulleted feedback with     │
│        recommendations                                   │
│     c. "Score Next Frame" → loops to step 4 for next    │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

### Step 7: Update manifest.json

Replace the contents of `manifest.json` with:

```json
{
  "name": "Heuristic Screen Score",
  "id": "heuristic-screen-score-m1",
  "api": "1.0.0",
  "main": "code.js",
  "capabilities": [],
  "enableProposedApi": false,
  "editorType": ["figma"],
  "ui": "ui.html",
  "networkAccess": {
    "allowedDomains": ["none"]
  }
}
```

Notes:
- `"ui": "ui.html"` — tells Figma where your UI file lives.
- `"editorType": ["figma"]` — plugin works in Figma Design only.
- No network access needed (all logic is local).

---

## Part 4: Build the Main Thread (code.ts)

### Step 8: Handle frame selection and parent-child filtering

Replace `code.ts` with the following structure:

```typescript
// code.ts — Main thread (Figma API access)

figma.showUI(__html__, { width: 480, height: 640, themeColors: true });

// ─── SELECTION HANDLING ───────────────────────────────────────────────

interface FrameInfo {
  id: string;
  name: string;
  width: number;
  height: number;
}

function getSelectedFrames(): FrameInfo[] {
  const selection = figma.currentPage.selection;

  // Only keep FrameNodes and ComponentNodes (top-level screens)
  const frames = selection.filter(
    (node): node is FrameNode | ComponentNode =>
      node.type === "FRAME" || node.type === "COMPONENT"
  );

  if (frames.length === 0) return [];

  // Filter out child frames: if a frame's ancestor is also in the selection, skip it
  const selectedIds = new Set(frames.map((f) => f.id));

  const topLevelFrames = frames.filter((frame) => {
    let parent = frame.parent;
    while (parent && parent.type !== "PAGE") {
      if (selectedIds.has(parent.id)) {
        return false; // This frame is a child of another selected frame
      }
      parent = parent.parent;
    }
    return true;
  });

  return topLevelFrames.map((f) => ({
    id: f.id,
    name: f.name,
    width: f.width,
    height: f.height,
  }));
}

// Send initial selection to UI
figma.ui.postMessage({
  type: "selection",
  frames: getSelectedFrames(),
});

// Listen for selection changes
figma.on("selectionchange", () => {
  figma.ui.postMessage({
    type: "selection",
    frames: getSelectedFrames(),
  });
});

// ─── MESSAGE HANDLER (from UI) ───────────────────────────────────────

figma.ui.onmessage = async (msg) => {
  if (msg.type === "export-scorecard") {
    await exportScorecard(msg.frameId, msg.scorecardData);
  }

  if (msg.type === "close") {
    figma.closePlugin();
  }
};

// ─── EXPORT SCORECARD AS IMAGE NEXT TO FRAME ─────────────────────────

async function exportScorecard(frameId: string, data: any) {
  const targetNode = figma.getNodeById(frameId) as FrameNode;
  if (!targetNode) return;

  // Load fonts before creating text
  await figma.loadFontAsync({ family: "Inter", style: "Regular" });
  await figma.loadFontAsync({ family: "Inter", style: "Medium" });
  await figma.loadFontAsync({ family: "Inter", style: "Bold" });

  // Create the scorecard frame
  const card = figma.createFrame();
  card.name = `Score: ${data.screenName} — ${data.grade}`;
  card.resize(400, 1); // Width fixed, height auto
  card.layoutMode = "VERTICAL";
  card.primaryAxisSizingMode = "AUTO";
  card.counterAxisSizingMode = "FIXED";
  card.paddingTop = 32;
  card.paddingBottom = 32;
  card.paddingLeft = 24;
  card.paddingRight = 24;
  card.itemSpacing = 16;
  card.cornerRadius = 12;
  card.fills = [{ type: "SOLID", color: { r: 1, g: 1, b: 1 } }];
  card.strokes = [{ type: "SOLID", color: { r: 0.9, g: 0.9, b: 0.9 } }];
  card.strokeWeight = 1;

  // Position next to the target frame (to the right, with 80px gap)
  card.x = targetNode.x + targetNode.width + 80;
  card.y = targetNode.y;

  // ─── HEADER ───
  const header = createText("HEURISTIC SCORE", 11, "Medium", { r: 0.4, g: 0.4, b: 0.4 });
  card.appendChild(header);

  const screenName = createText(data.screenName, 20, "Bold", { r: 0.1, g: 0.1, b: 0.1 });
  card.appendChild(screenName);

  // ─── SCORE ───
  const scoreText = createText(
    `${data.compositeScore.toFixed(2)} / 5.00  •  Grade: ${data.grade}`,
    16,
    "Bold",
    getGradeColor(data.grade)
  );
  card.appendChild(scoreText);

  // ─── METADATA ───
  const metaText = createText(
    `Vertical: ${data.vertical}  |  Device: ${data.device}  |  Type: ${data.screenType}`,
    11,
    "Regular",
    { r: 0.5, g: 0.5, b: 0.5 }
  );
  card.appendChild(metaText);

  // ─── TIMESTAMP ───
  const timestamp = createText(
    `Evaluated: ${data.timestamp}`,
    11,
    "Regular",
    { r: 0.5, g: 0.5, b: 0.5 }
  );
  card.appendChild(timestamp);

  const evaluator = createText(
    `By: ${data.evaluatorName}  |  Methodology: m1-ai-optional v1.0`,
    11,
    "Regular",
    { r: 0.5, g: 0.5, b: 0.5 }
  );
  card.appendChild(evaluator);

  // ─── DIVIDER ───
  const divider = figma.createFrame();
  divider.resize(352, 1);
  divider.fills = [{ type: "SOLID", color: { r: 0.9, g: 0.9, b: 0.9 } }];
  card.appendChild(divider);
  divider.layoutSizingHorizontal = "FILL";

  // ─── PILLAR BREAKDOWN ───
  const breakdownTitle = createText("PILLAR BREAKDOWN", 11, "Medium", { r: 0.4, g: 0.4, b: 0.4 });
  card.appendChild(breakdownTitle);

  for (const pillar of data.pillarScores) {
    const pillarRow = createText(
      `${pillar.id}  ${pillar.name.padEnd(16)}  ${pillar.grade}/5  ${pillar.label}`,
      12,
      "Regular",
      getPillarColor(pillar.grade)
    );
    card.appendChild(pillarRow);
  }

  // ─── ISSUES (if any) ───
  if (data.issues && data.issues.length > 0) {
    const divider2 = figma.createFrame();
    divider2.resize(352, 1);
    divider2.fills = [{ type: "SOLID", color: { r: 0.9, g: 0.9, b: 0.9 } }];
    card.appendChild(divider2);
    divider2.layoutSizingHorizontal = "FILL";

    const issuesTitle = createText(
      `ISSUES (${data.issues.length})`,
      11,
      "Medium",
      { r: 0.4, g: 0.4, b: 0.4 }
    );
    card.appendChild(issuesTitle);

    for (const issue of data.issues.slice(0, 10)) {
      const issueText = createText(
        `• [${issue.answer}] ${issue.questionId}: ${issue.questionText}`,
        11,
        "Regular",
        issue.answer === "No"
          ? { r: 0.94, g: 0.27, b: 0.27 }
          : { r: 0.98, g: 0.7, b: 0.09 }
      );
      card.appendChild(issueText);
    }

    if (data.issues.length > 10) {
      const moreText = createText(
        `... and ${data.issues.length - 10} more issues`,
        11,
        "Regular",
        { r: 0.5, g: 0.5, b: 0.5 }
      );
      card.appendChild(moreText);
    }
  }

  // Select the card and zoom to it
  figma.currentPage.selection = [card];
  figma.viewport.scrollAndZoomIntoView([targetNode, card]);

  figma.ui.postMessage({ type: "export-complete" });
}

// ─── HELPER FUNCTIONS ────────────────────────────────────────────────

function createText(
  content: string,
  fontSize: number,
  style: "Regular" | "Medium" | "Bold",
  color: { r: number; g: number; b: number }
): TextNode {
  const text = figma.createText();
  text.fontName = { family: "Inter", style };
  text.fontSize = fontSize;
  text.characters = content;
  text.fills = [{ type: "SOLID", color }];
  return text;
}

function getGradeColor(grade: string): { r: number; g: number; b: number } {
  switch (grade) {
    case "A+": return { r: 0.13, g: 0.77, b: 0.37 };
    case "A":  return { r: 0.52, g: 0.8, b: 0.09 };
    case "B":  return { r: 0.92, g: 0.7, b: 0.03 };
    case "C":  return { r: 0.98, g: 0.45, b: 0.09 };
    case "D":  return { r: 0.94, g: 0.27, b: 0.27 };
    default:   return { r: 0.6, g: 0.11, b: 0.11 };
  }
}

function getPillarColor(grade: number): { r: number; g: number; b: number } {
  if (grade >= 5) return { r: 0.13, g: 0.77, b: 0.37 };
  if (grade >= 4) return { r: 0.52, g: 0.8, b: 0.09 };
  if (grade >= 3) return { r: 0.6, g: 0.6, b: 0.03 };
  if (grade >= 2) return { r: 0.98, g: 0.45, b: 0.09 };
  if (grade >= 1) return { r: 0.94, g: 0.27, b: 0.27 };
  return { r: 0.6, g: 0.11, b: 0.11 };
}
```

---

## Part 5: Build the UI (ui.html)

### Step 9: Create the plugin interface

The UI is a single HTML file with embedded CSS and JavaScript. It handles three screens: Configuration, Checklist, and Results.

Replace `ui.html` with the following structure (this is a skeleton — you'll flesh out each section):

```html
<!DOCTYPE html>
<html>
<head>
  <style>
    /* ─── BASE STYLES ─── */
    * { box-sizing: border-box; margin: 0; padding: 0; }
    body {
      font-family: Inter, -apple-system, sans-serif;
      font-size: 13px;
      color: #333;
      padding: 16px;
      background: var(--figma-color-bg, #fff);
    }
    h1 { font-size: 16px; font-weight: 700; margin-bottom: 12px; }
    h2 { font-size: 14px; font-weight: 600; margin-bottom: 8px; color: #555; }
    h3 { font-size: 12px; font-weight: 600; margin: 16px 0 8px; color: #666; text-transform: uppercase; letter-spacing: 0.5px; }

    /* ─── FORM ELEMENTS ─── */
    label { display: block; font-size: 11px; font-weight: 500; margin-bottom: 4px; color: #666; }
    select, input[type="text"] {
      width: 100%;
      padding: 8px 12px;
      border: 1px solid #ddd;
      border-radius: 6px;
      font-size: 13px;
      margin-bottom: 12px;
      background: #fafafa;
    }
    select:focus, input:focus { outline: none; border-color: #0D99FF; }

    .btn {
      padding: 10px 20px;
      border: none;
      border-radius: 8px;
      font-size: 13px;
      font-weight: 600;
      cursor: pointer;
      transition: background 0.2s;
    }
    .btn-primary { background: #0D99FF; color: white; }
    .btn-primary:hover { background: #0B85E0; }
    .btn-secondary { background: #f0f0f0; color: #333; }
    .btn-secondary:hover { background: #e0e0e0; }

    /* ─── SCREENS ─── */
    .screen { display: none; }
    .screen.active { display: block; }

    /* ─── QUESTION CARD ─── */
    .question-card {
      border: 1px solid #eee;
      border-radius: 8px;
      padding: 12px;
      margin-bottom: 12px;
      background: #fafafa;
    }
    .question-card .q-text { font-size: 12px; font-weight: 500; margin-bottom: 8px; }
    .question-card .q-guidance { font-size: 11px; color: #888; margin-bottom: 8px; }
    .question-card .q-options { display: flex; gap: 8px; }
    .question-card .q-options button {
      padding: 6px 12px;
      border: 1px solid #ddd;
      border-radius: 6px;
      background: white;
      font-size: 11px;
      cursor: pointer;
    }
    .question-card .q-options button.selected { border-color: #0D99FF; background: #E8F4FD; color: #0D99FF; font-weight: 600; }
    .question-card .q-options button.selected-no { border-color: #EF4444; background: #FEF2F2; color: #EF4444; }
    .question-card .q-options button.selected-partial { border-color: #EAB308; background: #FEFCE8; color: #92400E; }
    .question-card .note-field { width: 100%; margin-top: 8px; padding: 6px 10px; border: 1px solid #ddd; border-radius: 6px; font-size: 11px; display: none; }
    .question-card .note-field.visible { display: block; }

    /* ─── RESULTS ─── */
    .score-display {
      text-align: center;
      padding: 24px;
      background: #f8f9fa;
      border-radius: 12px;
      margin-bottom: 16px;
    }
    .score-display .score-number { font-size: 36px; font-weight: 800; }
    .score-display .score-grade { font-size: 20px; font-weight: 600; margin-top: 4px; }
    .pillar-bar {
      display: flex;
      align-items: center;
      margin-bottom: 6px;
      font-size: 11px;
    }
    .pillar-bar .pillar-name { width: 100px; }
    .pillar-bar .bar-container { flex: 1; height: 8px; background: #eee; border-radius: 4px; margin: 0 8px; }
    .pillar-bar .bar-fill { height: 100%; border-radius: 4px; }
    .pillar-bar .pillar-grade { width: 60px; text-align: right; }

    /* ─── FEEDBACK SECTION ─── */
    .feedback-item {
      border-left: 3px solid #EF4444;
      padding: 8px 12px;
      margin-bottom: 8px;
      background: #fefefe;
      border-radius: 0 6px 6px 0;
    }
    .feedback-item.partial { border-left-color: #EAB308; }
    .feedback-item .fb-question { font-size: 12px; font-weight: 500; }
    .feedback-item .fb-recommendation { font-size: 11px; color: #555; margin-top: 4px; }

    /* ─── UTILITY ─── */
    .flex { display: flex; gap: 8px; }
    .mt-16 { margin-top: 16px; }
    .mb-8 { margin-bottom: 8px; }
    .text-center { text-align: center; }
    .text-muted { color: #888; font-size: 11px; }
    .frame-badge { display: inline-block; padding: 4px 8px; background: #E8F4FD; color: #0D99FF; border-radius: 4px; font-size: 11px; font-weight: 500; margin: 2px; }
    .progress-bar { width: 100%; height: 4px; background: #eee; border-radius: 2px; margin: 12px 0; }
    .progress-bar .fill { height: 100%; background: #0D99FF; border-radius: 2px; transition: width 0.3s; }
  </style>
</head>
<body>

  <!-- ═══════ SCREEN 1: CONFIGURATION ═══════ -->
  <div id="screen-config" class="screen active">
    <h1>Heuristic Screen Score</h1>
    <p class="text-muted mb-8">m1-ai-optional v1.0 • 55 questions across 10 pillars</p>

    <div id="selected-frames-info">
      <h3>Selected Frames</h3>
      <div id="frames-list"><span class="text-muted">Select one or more frames on the canvas</span></div>
    </div>

    <h3>Configuration</h3>

    <label>Screen Name *</label>
    <input type="text" id="config-screen-name" placeholder="e.g., Loan Eligibility Check">

    <label>Product Vertical *</label>
    <select id="config-vertical">
      <option value="General">General</option>
      <option value="Lending">Lending</option>
      <option value="Investing">Investing</option>
      <option value="Shopping">Shopping (EMI Card)</option>
      <option value="Trading">Trading</option>
      <option value="Insurance">Insurance</option>
      <option value="Payments">Payments</option>
    </select>

    <label>Screen Type *</label>
    <select id="config-screen-type">
      <option value="">— Select —</option>
      <option value="Landing">Landing</option>
      <option value="Form">Form</option>
      <option value="Dashboard">Dashboard</option>
      <option value="Confirmation">Confirmation</option>
      <option value="Error">Error</option>
      <option value="Listing">Listing</option>
      <option value="Detail">Detail</option>
      <option value="Onboarding">Onboarding</option>
      <option value="Settings">Settings</option>
      <option value="Empty State">Empty State</option>
    </select>

    <label>Device *</label>
    <select id="config-device">
      <option value="Mobile">Mobile</option>
      <option value="Desktop">Desktop</option>
      <option value="Responsive">Responsive</option>
    </select>

    <label>Flow Position *</label>
    <select id="config-flow-position">
      <option value="Standalone">Standalone</option>
      <option value="Entry">Entry</option>
      <option value="Mid-flow">Mid-flow</option>
      <option value="Terminal">Terminal</option>
    </select>

    <label>User Login State</label>
    <select id="config-login-state">
      <option value="Post-login">Post-login</option>
      <option value="Pre-login">Pre-login</option>
    </select>

    <label>Evaluator Name *</label>
    <input type="text" id="config-evaluator" placeholder="Your name">

    <div class="mt-16">
      <button class="btn btn-primary" id="btn-start-scoring" disabled>Start Scoring</button>
    </div>
  </div>

  <!-- ═══════ SCREEN 2: CHECKLIST ═══════ -->
  <div id="screen-checklist" class="screen">
    <div style="display: flex; justify-content: space-between; align-items: center;">
      <h2 id="current-pillar-name">Pillar 1: Clarity</h2>
      <span class="text-muted" id="progress-text">0 / 55</span>
    </div>
    <div class="progress-bar"><div class="fill" id="progress-fill" style="width: 0%"></div></div>
    <div id="questions-container"></div>
    <div class="mt-16 flex">
      <button class="btn btn-secondary" id="btn-prev-pillar">← Previous</button>
      <button class="btn btn-primary" id="btn-next-pillar">Next Pillar →</button>
    </div>
  </div>

  <!-- ═══════ SCREEN 3: RESULTS ═══════ -->
  <div id="screen-results" class="screen">
    <h1>Score Results</h1>
    <div class="score-display">
      <div class="score-number" id="result-score">4.12</div>
      <div class="score-grade" id="result-grade">Grade: A</div>
      <div class="text-muted" id="result-timestamp"></div>
    </div>

    <h3>Pillar Breakdown</h3>
    <div id="pillar-breakdown"></div>

    <h3>Issues Found</h3>
    <div id="issues-list"></div>

    <div class="mt-16 flex" style="flex-wrap: wrap; gap: 8px;">
      <button class="btn btn-primary" id="btn-export">Export as Image</button>
      <button class="btn btn-secondary" id="btn-feedback">Get Feedback</button>
      <button class="btn btn-secondary" id="btn-score-next">Score Next Frame</button>
    </div>
  </div>

  <!-- ═══════ SCREEN 4: FEEDBACK ═══════ -->
  <div id="screen-feedback" class="screen">
    <div style="display: flex; justify-content: space-between; align-items: center;">
      <h1>Directed Feedback</h1>
      <button class="btn btn-secondary" id="btn-back-results">← Back to Score</button>
    </div>
    <p class="text-muted mb-8">Actionable recommendations for each issue found</p>
    <div id="feedback-container"></div>
    <div class="mt-16">
      <button class="btn btn-primary" id="btn-export-feedback">Export Feedback as Image</button>
    </div>
  </div>

  <!-- ═══════ JAVASCRIPT ═══════ -->
  <script>
    // ─── DATA: ALL 55 QUESTIONS ─────────────────────────────────────
    // (You will paste the full question bank here — see Step 10)
    const QUESTIONS = []; // Populated in Step 10

    // ─── STATE ──────────────────────────────────────────────────────
    let selectedFrames = [];
    let currentFrameIndex = 0;
    let config = {};
    let answers = {}; // { "1.1": "Yes", "1.2": "Partial", ... }
    let notes = {};   // { "1.2": "CTA says Submit", ... }
    let currentPillarIndex = 0;

    // ─── COMMUNICATION WITH MAIN THREAD ─────────────────────────────
    window.onmessage = (event) => {
      const msg = event.data.pluginMessage;
      if (!msg) return;

      if (msg.type === "selection") {
        selectedFrames = msg.frames;
        renderFramesList();
        updateStartButton();
      }

      if (msg.type === "export-complete") {
        alert("Scorecard exported to canvas!");
      }
    };

    function postToPlugin(data) {
      parent.postMessage({ pluginMessage: data }, "*");
    }

    // ─── SCREEN NAVIGATION ──────────────────────────────────────────
    function showScreen(screenId) {
      document.querySelectorAll(".screen").forEach((s) => s.classList.remove("active"));
      document.getElementById(screenId).classList.add("active");
    }

    // ─── CONFIG SCREEN LOGIC ────────────────────────────────────────
    function renderFramesList() {
      const container = document.getElementById("frames-list");
      if (selectedFrames.length === 0) {
        container.innerHTML = '<span class="text-muted">Select one or more frames on the canvas</span>';
      } else {
        container.innerHTML = selectedFrames
          .map((f) => `<span class="frame-badge">${f.name}</span>`)
          .join(" ");
      }
    }

    function updateStartButton() {
      const btn = document.getElementById("btn-start-scoring");
      const hasFrames = selectedFrames.length > 0;
      const hasName = document.getElementById("config-screen-name").value.trim() !== "";
      const hasType = document.getElementById("config-screen-type").value !== "";
      const hasEvaluator = document.getElementById("config-evaluator").value.trim() !== "";
      btn.disabled = !(hasFrames && hasName && hasType && hasEvaluator);
    }

    // Add input listeners for validation
    ["config-screen-name", "config-screen-type", "config-evaluator"].forEach((id) => {
      document.getElementById(id).addEventListener("input", updateStartButton);
      document.getElementById(id).addEventListener("change", updateStartButton);
    });

    // Start scoring button
    document.getElementById("btn-start-scoring").addEventListener("click", () => {
      config = {
        screenName: document.getElementById("config-screen-name").value.trim(),
        vertical: document.getElementById("config-vertical").value,
        screenType: document.getElementById("config-screen-type").value,
        device: document.getElementById("config-device").value,
        flowPosition: document.getElementById("config-flow-position").value,
        loginState: document.getElementById("config-login-state").value,
        evaluator: document.getElementById("config-evaluator").value.trim(),
      };
      answers = {};
      notes = {};
      currentPillarIndex = 0;
      renderChecklist();
      showScreen("screen-checklist");
    });

    // ─── CHECKLIST LOGIC ────────────────────────────────────────────
    // (Rendering, answering, navigation — see Step 11)

    // ─── SCORING LOGIC ──────────────────────────────────────────────
    // (Computation — see Step 12)

    // ─── RESULTS LOGIC ──────────────────────────────────────────────
    // (Display, export, feedback — see Step 13)

    // ─── EXPORT BUTTON ──────────────────────────────────────────────
    document.getElementById("btn-export").addEventListener("click", () => {
      const scorecardData = computeFullResults();
      postToPlugin({
        type: "export-scorecard",
        frameId: selectedFrames[currentFrameIndex].id,
        scorecardData: scorecardData,
      });
    });

    // ─── FEEDBACK BUTTON ────────────────────────────────────────────
    document.getElementById("btn-feedback").addEventListener("click", () => {
      renderFeedback();
      showScreen("screen-feedback");
    });

    document.getElementById("btn-back-results").addEventListener("click", () => {
      showScreen("screen-results");
    });
  </script>

</body>
</html>
```

---

## Part 6: Implement the Question Data and Scoring Logic

### Step 10: Define the question bank

In your `ui.html` `<script>` section, replace the empty `QUESTIONS` array with the full 55-question bank. Each question object looks like this:

```javascript
const QUESTIONS = [
  {
    id: "1.1",
    pillar: 1,
    pillarName: "Clarity & Comprehension",
    text: "Can a first-time user tell what this screen is for within 3 seconds?",
    guidance: "Cover the body text with your hand. Read only the heading and CTA. Is the purpose obvious?",
    // applicability: which screen types make this N/A (empty = always applicable)
    autoNA: {
      screenTypes: [],   // never auto-N/A for this question
      device: [],        // never auto-N/A for device
    },
  },
  {
    id: "1.6",
    pillar: 1,
    pillarName: "Clarity & Comprehension",
    text: "Is detailed/secondary info hidden behind a tap?",
    guidance: "Check if T&C, fee breakdowns, or additional explanations are tucked away.",
    autoNA: {
      screenTypes: ["Confirmation", "Error", "Empty State"],
      device: [],
    },
  },
  // ... all 55 questions following the same structure
  // Use the Applicability Matrix (Section 6 of m1-ai-optional.md) to set autoNA
];
```

You need to define all 55 questions. The `autoNA` field encodes the logic from Section 6 of your methodology document. Questions not in the applicability matrix have empty arrays (always shown).

### Step 11: Implement checklist rendering

Add this function to your `<script>`:

```javascript
const PILLAR_NAMES = [
  "Clarity & Comprehension",
  "Navigation & Wayfinding",
  "Task Efficiency",
  "Visual Hierarchy & Layout",
  "Feedback & System Status",
  "Error Handling & Recovery",
  "Accessibility & Inclusion",
  "Emotional Design & Trust",
  "Content Quality & Microcopy",
  "Ethical Integrity",
];

function getApplicableQuestions() {
  return QUESTIONS.map((q) => {
    const isAutoNA =
      q.autoNA.screenTypes.includes(config.screenType) ||
      (q.autoNA.device.includes("Desktop") && config.device === "Desktop");
    return { ...q, isAutoNA };
  });
}

function renderChecklist() {
  const questions = getApplicableQuestions();
  const pillarQuestions = questions.filter((q) => q.pillar === currentPillarIndex + 1);
  const container = document.getElementById("questions-container");
  const pillarName = PILLAR_NAMES[currentPillarIndex];

  document.getElementById("current-pillar-name").textContent =
    `Pillar ${currentPillarIndex + 1}: ${pillarName}`;

  // Update progress
  const answered = Object.keys(answers).length;
  const total = questions.filter((q) => !q.isAutoNA).length;
  document.getElementById("progress-text").textContent = `${answered} / ${total}`;
  document.getElementById("progress-fill").style.width = `${(answered / total) * 100}%`;

  container.innerHTML = pillarQuestions
    .map((q) => {
      if (q.isAutoNA) {
        answers[q.id] = "N/A"; // auto-set
        return `<div class="question-card" style="opacity: 0.5;">
          <div class="q-text">${q.id}. ${q.text}</div>
          <div class="q-guidance">Auto-set to N/A based on configuration</div>
        </div>`;
      }

      const currentAnswer = answers[q.id] || "";
      return `<div class="question-card">
        <div class="q-text">${q.id}. ${q.text}</div>
        <div class="q-guidance">${q.guidance}</div>
        <div class="q-options">
          ${["Yes", "Partial", "No", "N/A"]
            .map((opt) => {
              let cls = "";
              if (currentAnswer === opt) {
                if (opt === "No") cls = "selected-no";
                else if (opt === "Partial") cls = "selected-partial";
                else cls = "selected";
              }
              return `<button class="${cls}" onclick="setAnswer('${q.id}', '${opt}', this)">${opt}</button>`;
            })
            .join("")}
        </div>
        <input type="text" class="note-field ${currentAnswer === "Partial" || currentAnswer === "No" ? "visible" : ""}"
          placeholder="Optional note (why Partial/No?)"
          value="${notes[q.id] || ""}"
          oninput="setNote('${q.id}', this.value)"
          id="note-${q.id}">
      </div>`;
    })
    .join("");
}

function setAnswer(questionId, answer, btn) {
  answers[questionId] = answer;
  renderChecklist(); // re-render to update styles and progress
}

function setNote(questionId, value) {
  notes[questionId] = value;
}

// Navigation
document.getElementById("btn-next-pillar").addEventListener("click", () => {
  if (currentPillarIndex < 9) {
    currentPillarIndex++;
    renderChecklist();
    document.getElementById("screen-checklist").scrollTop = 0;
  } else {
    // All pillars done — show results
    const results = computeFullResults();
    renderResults(results);
    showScreen("screen-results");
  }
});

document.getElementById("btn-prev-pillar").addEventListener("click", () => {
  if (currentPillarIndex > 0) {
    currentPillarIndex--;
    renderChecklist();
  }
});
```

### Step 12: Implement the scoring math

```javascript
const VERTICAL_WEIGHTS = {
  General:   [10, 10, 10, 10, 10, 10, 10, 10, 10, 10],
  Lending:   [12, 8, 12, 8, 12, 12, 10, 10, 8, 8],
  Investing: [12, 8, 8, 10, 8, 8, 10, 14, 12, 10],
  Shopping:  [10, 12, 12, 12, 8, 8, 10, 8, 10, 10],
  Trading:   [8, 8, 18, 12, 14, 10, 6, 6, 8, 10],
  Insurance: [14, 8, 8, 8, 10, 10, 10, 14, 12, 6],
  Payments:  [8, 10, 18, 8, 14, 10, 10, 6, 8, 8],
};

function computeFullResults() {
  const questions = getApplicableQuestions();
  const pillarScores = [];
  const issues = [];
  let activePillars = [];

  for (let p = 1; p <= 10; p++) {
    const pillarQs = questions.filter((q) => q.pillar === p);
    const applicableQs = pillarQs.filter((q) => answers[q.id] !== "N/A");

    if (applicableQs.length === 0) {
      pillarScores.push({ id: `P${p}`, name: PILLAR_NAMES[p - 1], grade: null, label: "N/A", percentage: null });
      continue;
    }

    let rawPoints = 0;
    applicableQs.forEach((q) => {
      const ans = answers[q.id];
      if (ans === "Yes") rawPoints += 2;
      else if (ans === "Partial") rawPoints += 1;
      // "No" = 0

      if (ans === "No" || ans === "Partial") {
        issues.push({
          questionId: q.id,
          questionText: q.text,
          pillar: p,
          pillarName: PILLAR_NAMES[p - 1],
          answer: ans,
          note: notes[q.id] || "",
          guidance: q.guidance,
        });
      }
    });

    const maxPoints = applicableQs.length * 2;
    const percentage = (rawPoints / maxPoints) * 100;
    const grade = percentageToGrade(percentage);
    const label = gradeToLabel(grade);

    pillarScores.push({ id: `P${p}`, name: PILLAR_NAMES[p - 1], grade, label, percentage });
    activePillars.push({ pillarIndex: p - 1, grade });
  }

  // Compute composite with weights
  const weights = VERTICAL_WEIGHTS[config.vertical] || VERTICAL_WEIGHTS.General;
  let totalWeight = 0;
  let weightedSum = 0;

  activePillars.forEach(({ pillarIndex, grade }) => {
    totalWeight += weights[pillarIndex];
    weightedSum += grade * weights[pillarIndex];
  });

  const compositeScore = totalWeight > 0 ? weightedSum / totalWeight : 0;
  const overallGrade = compositeToGrade(compositeScore);

  // Sort issues by pillar weight (highest first)
  issues.sort((a, b) => weights[b.pillar - 1] - weights[a.pillar - 1]);

  const now = new Date();
  const timestamp = now.toLocaleString("en-IN", {
    day: "2-digit", month: "short", year: "numeric",
    hour: "2-digit", minute: "2-digit", hour12: true,
  });

  return {
    screenName: config.screenName,
    vertical: config.vertical,
    device: config.device,
    screenType: config.screenType,
    evaluatorName: config.evaluator,
    timestamp,
    compositeScore,
    grade: overallGrade,
    pillarScores,
    issues,
  };
}

function percentageToGrade(pct) {
  if (pct >= 85) return 5;
  if (pct >= 68) return 4;
  if (pct >= 51) return 3;
  if (pct >= 34) return 2;
  if (pct >= 17) return 1;
  return 0;
}

function gradeToLabel(grade) {
  const labels = ["Critical Failure", "Poor", "Below Average", "Adequate", "Good", "Exemplary"];
  return labels[grade];
}

function compositeToGrade(score) {
  if (score >= 4.5) return "A+";
  if (score >= 4.0) return "A";
  if (score >= 3.5) return "B";
  if (score >= 3.0) return "C";
  if (score >= 2.0) return "D";
  return "F";
}
```

### Step 13: Implement results rendering and feedback

```javascript
function renderResults(results) {
  document.getElementById("result-score").textContent = results.compositeScore.toFixed(2);
  document.getElementById("result-grade").textContent = `Grade: ${results.grade}`;
  document.getElementById("result-timestamp").textContent = results.timestamp;

  // Pillar breakdown bars
  const breakdown = document.getElementById("pillar-breakdown");
  breakdown.innerHTML = results.pillarScores
    .map((p) => {
      if (p.grade === null) return `<div class="pillar-bar"><span class="pillar-name">${p.id} ${p.name.split(" ")[0]}</span><span class="text-muted">N/A</span></div>`;
      const widthPct = (p.grade / 5) * 100;
      const color = getPillarBarColor(p.grade);
      return `<div class="pillar-bar">
        <span class="pillar-name">${p.id}</span>
        <div class="bar-container"><div class="bar-fill" style="width:${widthPct}%; background:${color}"></div></div>
        <span class="pillar-grade">${p.grade}/5 ${p.label}</span>
      </div>`;
    })
    .join("");

  // Issues list
  const issuesList = document.getElementById("issues-list");
  if (results.issues.length === 0) {
    issuesList.innerHTML = '<p class="text-muted">No issues found — excellent!</p>';
  } else {
    issuesList.innerHTML = results.issues
      .map((i) => `<div class="feedback-item ${i.answer === "Partial" ? "partial" : ""}">
        <div class="fb-question">[${i.answer}] ${i.questionId}: ${i.questionText}</div>
        ${i.note ? `<div class="fb-recommendation">Note: ${i.note}</div>` : ""}
      </div>`)
      .join("");
  }
}

function getPillarBarColor(grade) {
  const colors = ["#991B1B", "#EF4444", "#F97316", "#EAB308", "#84CC16", "#22C55E"];
  return colors[grade];
}

// ─── FEEDBACK WITH RECOMMENDATIONS ──────────────────────────────────

const RECOMMENDATIONS = {
  "1.1": "Add a clear, benefit-driven headline. Remove competing visual elements. Ensure the screen's purpose is obvious from the heading + CTA alone.",
  "1.2": "Make the primary CTA the largest, most colourful interactive element. Reduce visual weight of banners/secondary links.",
  "1.3": "Replace generic labels ('Submit', 'Next') with specific action verbs ('Check Eligibility', 'Pay ₹1,200').",
  "1.4": "Replace or explain jargon terms. Add tooltips or inline definitions for domain-specific language.",
  "1.5": "Apply progressive disclosure — move secondary details behind 'View more' or an expand control.",
  "1.6": "Tuck T&C, fee breakdowns, and long explanations behind expandable sections or 'View details' links.",
  "1.7": "Add text labels below or beside ambiguous icons. Only home, search, back, and close can be unlabelled.",
  "2.1": "Add a screen title, breadcrumb, highlighted nav tab, or step indicator to show location.",
  "2.2": "Add a visible back arrow, close button (X), or 'Cancel' link.",
  "2.3": "Add a clear forward CTA or 'What's next?' guidance.",
  "2.4": "Maintain consistent navigation bar, header style, and tab positions across screens.",
  "2.5": "Add search or filter controls for lists with more than 7 items.",
  "2.6": "Add enough context (title, summary) so the screen makes sense without prior navigation.",
  "3.1": "Remove unnecessary fields. Auto-fill what's possible. Combine related steps.",
  "3.2": "Use appropriate input types: numeric keyboards for numbers, date pickers for dates, dropdowns for short lists.",
  "3.3": "Pre-fill name, phone, email, and other known data for logged-in users.",
  "3.4": "Increase all tappable areas to minimum 44×44 pt (iOS) or 48×48 dp (Android).",
  "3.5": "Move the primary CTA to the bottom 40% of the screen for easy thumb reach.",
  "3.7": "Add quick-actions, recent items, or saved preferences for returning users.",
  "3.8": "Remove forced cross-sell interstitials, mandatory rating prompts, or full-screen promotions from the task flow.",
  "4.1": "Increase size/contrast of the primary message and CTA. Reduce prominence of secondary elements.",
  "4.2": "Restructure layout to follow natural F-pattern (text-heavy) or Z-pattern (minimal) scanning flow.",
  "4.3": "Group related information together using proximity, cards, or section dividers.",
  "4.4": "Adjust spacing: add breathing room between sections, reduce gaps that waste viewport space.",
  "4.5": "Establish clear type scale: at least 3 distinct levels (heading → subheading → body).",
  "4.6": "Audit colour usage: ensure consistent meaning (green=success, red=error) throughout.",
  "4.7": "Cut content at the fold line or add scroll indicators to hint at below-fold content.",
  "5.2": "Add a step indicator (e.g., 'Step 2 of 4') or progress bar for multi-step flows.",
  "5.4": "Add a clear success message with next-step guidance ('View your application', 'Go to Dashboard').",
  "5.7": "Remove promotional banners from task flows. Keep only notifications relevant to the current action.",
  "6.1": "Move error messages inline — directly below or beside the field with the error.",
  "6.2": "Rewrite errors to explain what went wrong AND how to fix it (e.g., 'Phone number must be 10 digits').",
  "6.3": "Add a confirmation dialog before irreversible actions (delete, submit, pay).",
  "6.4": "Allow users to edit previous selections. Add 'Back' or 'Edit' options.",
  "6.7": "Design empty states with explanation + action (e.g., 'No transactions yet. Make your first payment →').",
  "6.8": "After errors, offer clear next steps: 'Retry', 'Go back', 'Contact support'.",
  "7.1": "Increase text contrast to minimum 4.5:1. Avoid light grey on white or coloured text on coloured backgrounds.",
  "7.2": "Add icons, patterns, or text labels alongside colour to convey meaning.",
  "7.4": "Enlarge all interactive elements to minimum 44×44 pt.",
  "7.6": "Simplify language to Class 8 reading level. Replace jargon with plain terms.",
  "8.1": "Fix placeholder text, broken images, misaligned elements, inconsistent padding.",
  "8.2": "Match the screen's tone to its context (reassuring for payments, supportive for errors).",
  "8.3": "Add security badges, partner logos, ratings, or helpline numbers where personal/financial data is collected.",
  "8.4": "Align with brand fonts, colours, and icon style. Ensure visual consistency.",
  "8.5": "Add anxiety-reducing elements: cancellation policy, helpline number, security lock icon.",
  "8.7": "Simplify the visual design. Reduce cognitive load through progressive disclosure.",
  "9.1": "Make headings and bold text self-sufficient — they should convey the key message without reading body text.",
  "9.2": "Replace or explain all jargon (NAV, exit load, co-pay, etc.).",
  "9.3": "Replace generic CTA labels with specific action verbs.",
  "9.4": "Rewrite error messages to be human, empathetic, and action-oriented.",
  "9.5": "Design empty states with clear explanation + call to action.",
  "9.6": "Keep legal/regulatory text present but visually subordinate. Use collapsible sections if lengthy.",
  "9.7": "Use ₹ symbol, lakhs/crores, DD/MM/YYYY, and Indian phone formatting.",
  "10.1": "Uncheck all opt-in checkboxes by default.",
  "10.2": "Show all costs, fees, and charges before the commitment step.",
  "10.3": "Neutralise comparison displays. Avoid oversized 'Recommended' badges that manipulate choice.",
  "10.4": "Make 'Decline' and 'Accept' buttons equally visible and similar in size.",
  "10.5": "Use neutral decline language: 'No thanks', 'Skip', 'Maybe later'. Avoid guilt-tripping.",
  "10.6": "Remove pre-selected add-ons. Let users actively opt in.",
  "10.7": "Remove artificial urgency messaging, or ensure all countdown timers/stock warnings are genuine.",
  "10.8": "Remove fields not relevant to the current task. Collect only what's needed now.",
};

function renderFeedback() {
  const results = computeFullResults();
  const container = document.getElementById("feedback-container");

  if (results.issues.length === 0) {
    container.innerHTML = '<p class="text-muted">No issues to provide feedback on. Great work!</p>';
    return;
  }

  container.innerHTML = results.issues
    .map((issue) => {
      const recommendation = RECOMMENDATIONS[issue.questionId] || "Review this criterion and address the gap.";
      return `<div class="feedback-item ${issue.answer === "Partial" ? "partial" : ""}">
        <div class="fb-question"><strong>[${issue.answer}]</strong> ${issue.questionId}: ${issue.questionText}</div>
        ${issue.note ? `<div style="font-size:11px; color:#666; margin-top:4px;">Your note: "${issue.note}"</div>` : ""}
        <div class="fb-recommendation">
          <strong>Recommendation:</strong> ${recommendation}
        </div>
      </div>`;
    })
    .join("");
}
```

---

## Part 7: Handle Multiple Frame Scoring

### Step 14: Multi-frame queue logic

Add this to your UI script to handle scoring multiple frames in sequence:

```javascript
document.getElementById("btn-score-next").addEventListener("click", () => {
  if (currentFrameIndex < selectedFrames.length - 1) {
    currentFrameIndex++;
    // Pre-fill the screen name with the frame name
    document.getElementById("config-screen-name").value = selectedFrames[currentFrameIndex].name;
    showScreen("screen-config");
    updateStartButton();
  } else {
    alert("All selected frames have been scored!");
  }
});
```

The parent-child filtering already happens in `code.ts` (Step 8) — if a user selects a page layout frame and also a card inside it, the card is automatically excluded.

---

## Part 8: Export Scorecard as Image on Canvas

### Step 15: How the export works

The export flow is:

1. User clicks "Export as Image" in the UI.
2. UI sends `{ type: "export-scorecard", frameId, scorecardData }` to the main thread.
3. Main thread (`code.ts`) creates a new auto-layout frame on the canvas with text nodes showing the score, breakdown, timestamp, and issues.
4. The frame is positioned to the right of the evaluated frame (80px gap).
5. The canvas scrolls to show both frames.

This is already implemented in Step 8's `code.ts`. The scorecard is a live, editable Figma frame (not a flat image), which is actually more useful — the team can inspect, copy, or move it.

If you want a flat PNG instead, add this after creating the card:

```typescript
// Optional: flatten to image
const bytes = await card.exportAsync({ format: "PNG", constraint: { type: "SCALE", value: 2 } });
const image = figma.createImage(bytes);
const imageFrame = figma.createFrame();
imageFrame.resize(card.width, card.height);
imageFrame.fills = [{ imageHash: image.hash, scaleMode: "FILL", scalingFactor: 1, type: "IMAGE" }];
imageFrame.x = card.x;
imageFrame.y = card.y;
card.remove(); // remove the editable version, keep the flat image
```

---

## Part 9: Add Date/Time Stamp

### Step 16: Timestamp implementation

Already handled in Step 12's `computeFullResults()`:

```javascript
const now = new Date();
const timestamp = now.toLocaleString("en-IN", {
  day: "2-digit", month: "short", year: "numeric",
  hour: "2-digit", minute: "2-digit", hour12: true,
});
```

This produces output like: `01 Jun 2026, 11:30 AM`

The timestamp is:
- Shown in the results screen UI
- Included in the exported scorecard frame on canvas
- Part of the scorecardData sent to the main thread

---

## Part 10: Build, Test, and Iterate

### Step 17: Build and run

```bash
# In your plugin folder
npx tsc --watch
```

Then in Figma:
1. Select a frame on your canvas.
2. Go to **Plugins → Development → Heuristic Screen Score**
3. The plugin panel opens with your selected frame(s) listed.
4. Fill in configuration → answer questions → see results.

### Step 18: Debug tips

- **Console logs:** In the main thread (`code.ts`), use `console.log()` — view output in Figma's developer console (**Plugins → Development → Show/Hide console**).
- **UI console:** Right-click the plugin panel → "Inspect Element" to open browser DevTools for the UI iframe.
- **Common error:** "Cannot read property 'characters' of undefined" → You forgot to `await figma.loadFontAsync()` before creating text nodes.
- **Common error:** "Plugin closed unexpectedly" → Unhandled exception in `code.ts`. Check the console.

### Step 19: Test the parent-child filter

1. Create a frame called "Screen A" with child frames inside it (cards, etc.).
2. Select both "Screen A" AND one of its child frames.
3. Run the plugin — only "Screen A" should appear in the frames list.

### Step 20: Polish and publish (when ready)

1. Add a plugin icon (128×128 PNG) to your folder.
2. Update `manifest.json` to include the icon path.
3. Test thoroughly with different screen types and configurations.
4. To publish: **Plugins → Development → Manage plugins in development → Publish**

---

## Summary: File Structure (Final)

```
heuristic-score/
├── manifest.json          ← Plugin config
├── code.ts                ← Main thread (frame selection, canvas operations, export)
├── code.js                ← Auto-compiled output
├── ui.html                ← Full UI (config, checklist, results, feedback)
├── package.json           ← Dependencies
├── tsconfig.json          ← TypeScript config
└── icon.png               ← Plugin icon (128×128, add later)
```

---

## Key Figma Plugin API Methods Used

| Method | Purpose |
|--------|---------|
| `figma.currentPage.selection` | Get selected nodes |
| `figma.on("selectionchange", cb)` | React to selection changes |
| `figma.showUI(__html__, opts)` | Show the UI panel |
| `figma.ui.postMessage(data)` | Send data to UI |
| `figma.ui.onmessage = (msg) => {}` | Receive data from UI |
| `figma.createFrame()` | Create a frame on canvas |
| `figma.createText()` | Create a text node |
| `figma.loadFontAsync(fontName)` | Load font before text operations |
| `node.exportAsync(settings)` | Export node as PNG/SVG |
| `figma.createImage(bytes)` | Create image from exported bytes |
| `figma.viewport.scrollAndZoomIntoView(nodes)` | Pan camera to nodes |
| `figma.getNodeById(id)` | Get node by ID |
| `figma.closePlugin()` | Close the plugin |

---

## Next Steps (After Basic Version Works)

1. **Persist evaluator name** using `figma.clientStorage.setAsync()` so it's remembered.
2. **Add keyboard shortcuts** for answering questions (Y/P/N/A keys).
3. **Add pillar-level skip**: if a reviewer is confident about a pillar, let them mark all as Yes at once.
4. **Batch export**: when scoring multiple frames, export all scorecards at once.
5. **History**: store past scores in client storage to track improvement over time.
