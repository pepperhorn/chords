# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Development Commands

All frontend commands run from the `view/` directory (uses **pnpm**):

```bash
cd view
pnpm install       # Install dependencies
pnpm dev           # Dev server at http://localhost:4321
pnpm build         # Production build
pnpm preview       # Preview production build
```

Python utility scripts run from the repo root:

```bash
python import_chords_db.py   # Import chord data from chords-db → Directus
python analyze_json.py       # Analyze chord JSON structure
python count_chords.py       # Count total chords in database
python fix_instruments.py    # Fix instrument field inconsistencies
pip install -r requirements.txt  # Install Python deps (requests, python-dotenv)
```

## Architecture Overview

This is a **chord data management system** for musicians. It has three main parts:

### 1. Frontend — `view/` (Astro 5 + React 19)
A git submodule. Two pages:
- `/` — Landing page
- `/editor` — Main chord editor

The editor is a React SPA embedded in Astro. It has two modes controlled by `chord_type`:
- **String instruments** (guitar, ukulele, mandolin): uses `@techies23/react-chords` for preview
- **Piano**: uses `piano-chord-charts` for preview

Key source files:
- `view/src/components/editor/ChordEditor.tsx` — Main editor, save logic
- `view/src/lib/types.ts` — All TypeScript interfaces
- `view/src/lib/directus.ts` — Directus SDK client setup
- `view/src/lib/generateSVG.ts` — Client-side SVG generation and upload to Directus
- `view/src/lib/utils.ts` — Helpers (e.g., `generateDisplayName`)

Path alias `@` → `./src` is configured in both `tsconfig.json` and `astro.config.mjs`.

`piano-chord-charts` is SSR-externalized in `astro.config.mjs` to avoid bundling issues.

### 2. Backend — Directus (headless CMS)
URL: `https://admin.creativeranges.org`

Collections:
- **`chord_data`** — Main collection. Each record is one chord voicing for one instrument. Key fields: `chord_key`, `chord_suffix`, `chord_type` (`"string"` or `"piano"`), `json` (flexible chord data), `svg_file` (UUID reference to generated SVG).
- **`chords`** — High-level chord concepts (C Major, D minor, etc.). M2M linked to `chord_data`.
- **`instruments`** — Reference list of instruments.

A Directus **Flow** ("Generate Chord SVG") triggers on chord create/update: reads chord data → generates SVG → uploads to Directus files → updates the chord record's `svg_file` field.

### 3. Data Source — `chords-db/` (git submodule)
The `tombatossals/chords-db` library. Pre-built JSON files at `chords-db/lib/guitar.json`, `ukulele.json`, `piano.json`. The Python import scripts read these and push data to Directus.

## Environment Variables

```env
PUBLIC_DIRECTUS_URL=https://admin.creativeranges.org
PUBLIC_DIRECTUS_TOKEN=<token>
```

The `PUBLIC_` prefix makes these available in the browser (Astro convention). Set in `view/.env.local` for local dev (see `view/ENV.example`).

## JSON Data Shapes

**String instrument** `json` field:
```json
{
  "chord_type": "string",
  "instrument": { "name": "guitar-beginner", "stringCount": 3, "strings": ["G","B","E"], "tuning": "standard-top3" },
  "positions": [{ "frets": "010", "fingers": "010", "barres": [], "capo": false }]
}
```

**Piano** `json` field:
```json
{
  "chord_type": "piano",
  "keys": [{ "note": "C", "octave": 4, "pressed": true }],
  "hand": "right",
  "fingering": { "C4": 1 }
}
```

## Key Conventions

- The app uses a **hybrid preview model**: real-time client-side preview during editing, then server-side SVG generation via Directus Flow for persistent assets.
- `chord_type` is the discriminator field that controls which editor/preview component renders and which JSON shape is used.
- Both `view/` and `chords-db/` are git submodules — changes inside them need to be committed separately.
