# Heuristic Score — Figma Plugin

UX heuristic scoring tool based on methodology **m1-ai-optional v1.0**. Scores selected Figma frames against 55 questions using hybrid auto-scoring: deterministic checks via the Figma Plugin API (contrast, tap targets, typography, layout) plus optional AI vision pre-fill (OpenAI / Anthropic / Gemini).

## Run locally

```bash
npm install
npm run build    # or npm run watch
```

Import `manifest.json` in Figma → Plugins → Development → Import plugin from manifest.

## Context

See [`CONTEXT-FOR-NEW-CHAT.md`](CONTEXT-FOR-NEW-CHAT.md) for full architecture and current state.

Methodology docs: [`docs/methodology/`](docs/methodology/)
