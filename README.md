# Figma Plugins

Local workspace folder for Figma plugins. Each plugin lives in its own subfolder (`manifest.json`, `code.ts`, `ui.html`, etc.).

**GitHub:** one repository per plugin — not a parent repo for this folder. Repo names use the **`fp-`** prefix (e.g. `fp-heuristic-score`).

See [`../PROJECTS.md`](../PROJECTS.md) for the full workspace registry.

---

## Plugin registry

Update this table whenever you **start work on a new plugin**.

| Folder | GitHub repo | Figma name | Status | Started | Description |
|--------|-------------|------------|--------|---------|-------------|
| [`heuristic-score/`](heuristic-score/) | `fp-heuristic-score` | Heuristic score - BFL design | active | 2026-05 | UX heuristic scoring tool (methodology m1-ai-optional v1.0). Hybrid auto-scoring via Figma API + optional AI vision (OpenAI / Anthropic / Gemini). |

---

## Plugins

### heuristic-score

**Path:** `figma-plugins/heuristic-score/`  
**GitHub:** `shravankohli/fp-heuristic-score`  
**Status:** active  
**Context file:** [`heuristic-score/CONTEXT-FOR-NEW-CHAT.md`](heuristic-score/CONTEXT-FOR-NEW-CHAT.md)  
**Cursor rule:** `.cursor/rules/figma-plugin-design-engineer.mdc`

Scores selected frames against 55 heuristic questions. Deterministic checks (contrast, tap targets, typography, layout) run locally; subjective questions can be pre-filled by AI. User reviews, overrides, then exports a scorecard to the canvas.

**Run locally:**
```bash
cd figma-plugins/heuristic-score
npm install
npm run build    # or npm run watch
```
Import `manifest.json` in Figma → Plugins → Development → Import plugin from manifest.

**Methodology docs:** `heuristic-score/docs/methodology/`

---

## Adding a new plugin

When you start a new plugin, do all of the following:

1. **Create folder** — `figma-plugins/<kebab-case-name>/`
2. **Scaffold** — use Figma's plugin template or copy structure from an existing plugin
3. **Add context** — create `CONTEXT-FOR-NEW-CHAT.md` (or `CONTEXT.md`) in the plugin folder
4. **Create GitHub repo** — `fp-<kebab-case-name>` (e.g. folder `my-plugin` → repo `fp-my-plugin`)
5. **Update this file** — add a row to the registry table and a section under **Plugins**
6. **Update workspace registry** — add an entry in [`../PROJECTS.md`](../PROJECTS.md)

### Registry row template

```markdown
| [`<folder>/`](<folder>/) | `fp-<folder>` | <Figma manifest name> | active | YYYY-MM | <one-line description> |
```

### Plugin section template

```markdown
### <folder-name>

**Path:** `figma-plugins/<folder>/`
**GitHub:** `shravankohli/fp-<folder>` (planned)
**Status:** active | paused | done
**Context file:** [`<folder>/CONTEXT-FOR-NEW-CHAT.md`](<folder>/CONTEXT-FOR-NEW-CHAT.md)

<2–3 sentence description>

**Run locally:**
\`\`\`bash
cd figma-plugins/<folder>
npm install
npm run build
\`\`\`
```

---

## Conventions

| Item | Convention |
|------|------------|
| Local folder | `figma-plugins/<kebab-case>/` |
| GitHub repo | `fp-<kebab-case>` under `shravankohli/` |
| Figma display name | `manifest.json` → `name` (human-readable, no prefix required) |
| Source | `code.ts` → compiles to `code.js` |
| UI | `ui.html` (iframe) |
| Git | Init inside each plugin folder; `figma-plugins/` itself is not a repo |
| Built artifacts | Gitignore `code.js`, `node_modules/` per plugin |

---

*Last updated: 2026-06-20*
