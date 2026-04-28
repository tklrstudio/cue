# Cue

Intelligent clip editor — connected media library and NLE timeline.

Your media is already ingested, analysed, tagged, and understood before you open the editor. No import dialogs. No project setup. No file management. Just editing.

## What is Cue?

Cue is a web-based NLE (non-linear editor) where the library is always pre-populated by connected systems. MAP pushes F1 moments. Chassis pushes podcast recordings. Everything arrives with analysis metadata already attached — race context, transcripts, quality scores, tags — so the Cut and Edit screens are purely about creative decisions.

Think DaVinci Resolve's Cut + Edit metaphor, without the file management.

## Tech Stack

- **Python** + **Starlette** — lightweight async web framework
- **PostgreSQL** (tklrdata on db-vps) — timeline and asset reference storage
- **Bouncer** — shared auth across TKLR apps (JWT, MFA)
- **Vanilla HTML/CSS/JS** + **hls.js** — no build step, HLS-native playback

## Integration Ecosystem

| Consumer | What it provides | Status |
|----------|-----------------|--------|
| [MAP](https://github.com/tklrstudio/motorsport-platform) | F1 moments, clips, HLS manifests, race context and analysis | Planned |
| [Chassis](https://github.com/tklrstudio/chassis) | Podcast episode recordings, HLS manifests, transcripts | Planned |

## Repo Structure

```
cue/
├── CLAUDE.md                    # Claude Code entry point → GLOBAL + LOCAL
├── LOCAL_AI_INSTRUCTIONS.md     # Workspace-specific AI context
├── README.md
├── .gitignore
├── _context/
│   └── CHARTER.md               # Project definition, boundaries, invariants
├── temporal/
│   ├── decisions/               # Architectural decisions
│   ├── discussions/             # Design conversations
│   ├── changes/                 # Change explainers
│   └── WORK_IN_PROGRESS.md
├── _guides/                     # User-facing walkthrough guides
└── .living-systems/             # Constitutional framework (submodule)
```

## How to Work Here

Start with `_context/CHARTER.md` — project boundaries, architectural invariants, and integration contracts.

This project operates within the [living-systems](https://github.com/tklrstudio/living-systems) constitutional framework.

## License

MIT
