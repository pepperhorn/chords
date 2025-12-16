# Consolidated Implementation Plan: Chord Data System with SVG Generation

## Overview

This plan consolidates three separate planning documents into a unified implementation strategy for a chord data management system with Directus backend and Astro frontend, supporting both string instruments (guitar) and piano.

## Architecture Decision: Hybrid Approach

**Recommended: Client-Side Preview + Server-Side Generation**

- **Client-side**: Interactive preview in editor using React components
- **Server-side**: Automatic SVG generation via Directus Flow when data is saved
- **Rationale**: Best of both worlds - immediate feedback for users, consistent production SVGs

---

## Phase 1: Directus Schema Setup

### 1.1 Extend `chord_data` Collection

Add the following fields to the existing `chord_data` collection:

#### Core Identification Fields

| Field | Type | Required | Default | Purpose |
|-------|------|----------|---------|---------|
| `chord_key` | string | Yes | - | Root note: "C", "D", "E", "F", "G", "A", "B" |
| `chord_suffix` | string | No | "" | Chord quality: "", "m", "7", "m7", "sus2", "sus4", "dim", "aug" |
| `chord_type` | string | Yes | - | Discriminator: "string" or "piano" |
| `difficulty` | string | No | - | "beginner", "intermediate", "advanced" |

#### String Instrument Specific Fields (conditional)

| Field | Type | Required | Conditional | Purpose |
|-------|------|----------|-------------|---------|
| `string_count` | integer | No | `chord_type = "string"` | Number of strings: 3, 4, 6 |
| `position_index` | integer | No | - | Multiple fingerings (0-based) |
| `is_barre` | boolean | No | `chord_type = "string"` | Barre chord flag |
| `capo_fret` | integer | No | `chord_type = "string"` | Capo position (null if none) |

#### Existing Fields (to maintain)

- `id` (UUID, primary key)
- `name` (string, required) - Display name
- `json` (JSON, required) - Chord data structure
- `status` (string) - System field
- `sort` (integer) - System field
- `user_created`, `date_created`, `user_updated`, `date_updated` - System fields
- `private_notes` (text) - Internal notes
- `public_notes` (text) - Public description
- `instruments` (M2M) - Link to instrument collection

### 1.2 JSON Field Structure

#### For String Instruments (`chord_type = "string"`)

```json
{
  "chord_type": "string",
  "instrument": {
    "name": "guitar-beginner",
    "stringCount": 3,
    "strings": ["G", "B", "E"],
    "tuning": "standard-top3"
  },
  "positions": [
    {
      "frets": "010",
      "fingers": "010",
      "barres": [],
      "capo": false
    }
  ]
}
```

#### For Piano (`chord_type = "piano"`)

```json
{
  "chord_type": "piano",
  "keys": [
    {
      "note": "C",
      "octave": 4,
      "pressed": true
    },
    {
      "note": "E",
      "octave": 4,
      "pressed": true
    },
    {
      "note": "G",
      "octave": 4,
      "pressed": true
    }
  ],
  "hand": "right",
  "fingering": {
    "C4": 1,
    "E4": 2,
    "G4": 5
  }
}
```

### 1.3 Schema Design: Unified Collection

**Decision**: Keep unified `chord_data` collection (current approach)

**Rationale**:
- Single source of truth for chord names
- Easy to query all chords regardless of instrument
- `instruments` M2M links to specific instruments
- `chord_type` discriminator handles different structures
- `json` field is flexible enough for both

**Structure**:
- One `chord_data` entry per chord + instrument + position combination
- Example: "C Major" on "Acoustic Guitar" (6-string, position 1) = one entry
- Example: "C Major" on "Piano & Keyboard" = separate entry
- Both share same `chord_key` ("C") and `chord_suffix` ("")

---

## Phase 2: Frontend Editor Implementation

### 2.1 Project Structure

```
view/src/
├── components/
│   └── editor/
│       ├── ChordEditor.tsx          # Main editor component
│       ├── ChordPreview.tsx          # Preview component (client-side)
│       ├── StringChordEditor.tsx     # String instrument editor
│       ├── PianoChordEditor.tsx      # Piano editor
│       └── MetadataForm.tsx          # Chord metadata form
├── lib/
│   ├── directus.ts                  # Directus client setup
│   ├── types.ts                     # TypeScript types
│   └── utils.ts                     # Helper functions
└── pages/
    └── editor/
        └── index.astro              # Editor page
```

### 2.2 Dependencies

Add to `view/package.json`:

```json
{
  "dependencies": {
    "@tombatossals/react-chords": "^1.0.0",
    "piano-chord-charts": "^1.13.1",
    "@directus/sdk": "^17.0.0"
  }
}
```

### 2.3 TypeScript Types

```typescript
// src/lib/types.ts

export type ChordType = 'string' | 'piano';
export type Difficulty = 'beginner' | 'intermediate' | 'advanced';
export type Hand = 'left' | 'right' | 'both';

export interface StringChordData {
  chord_type: 'string';
  instrument: {
    name: string;
    stringCount: number;
    strings: string[];
    tuning: string;
  };
  positions: Array<{
    frets: string;
    fingers: string;
    barres: number[];
    capo: boolean;
  }>;
}

export interface PianoChordData {
  chord_type: 'piano';
  keys: Array<{
    note: string;
    octave: number;
    pressed: boolean;
  }>;
  hand: Hand;
  fingering: Record<string, number>;
}

export interface ChordData {
  id?: string;
  name: string;
  chord_key: string;
  chord_suffix: string;
  chord_type: ChordType;
  difficulty?: Difficulty;
  string_count?: number;
  position_index?: number;
  is_barre?: boolean;
  capo_fret?: number | null;
  json: StringChordData | PianoChordData;
  instruments: string[];
  private_notes?: string;
  public_notes?: string;
}
```

### 2.4 Editor Component (Client-Side Preview)

```typescript
// src/components/editor/ChordEditor.tsx
import { useState } from 'react';
import { ChordPreview } from './ChordPreview';
import { StringChordEditor } from './StringChordEditor';
import { PianoChordEditor } from './PianoChordEditor';
import { MetadataForm } from './MetadataForm';
import type { ChordData } from '@/lib/types';

export function ChordEditor() {
  const [chordData, setChordData] = useState<ChordData>({
    name: '',
    chord_key: 'C',
    chord_suffix: '',
    chord_type: 'string',
    json: {
      chord_type: 'string',
      instrument: {
        name: 'guitar-beginner',
        stringCount: 3,
        strings: ['G', 'B', 'E'],
        tuning: 'standard-top3'
      },
      positions: [{
        frets: '010',
        fingers: '010',
        barres: [],
        capo: false
      }]
    },
    instruments: []
  });
  
  const [isSubmitting, setIsSubmitting] = useState(false);
  
  const handleSubmit = async () => {
    setIsSubmitting(true);
    try {
      // Save to Directus (Flow will generate SVG automatically)
      await directus.items('chord_data').createOne({
        name: generateDisplayName(chordData),
        chord_key: chordData.chord_key,
        chord_suffix: chordData.chord_suffix,
        chord_type: chordData.chord_type,
        difficulty: chordData.difficulty,
        string_count: chordData.string_count,
        position_index: chordData.position_index,
        is_barre: chordData.is_barre,
        capo_fret: chordData.capo_fret,
        json: chordData.json,
        instruments: chordData.instruments,
        private_notes: chordData.private_notes,
        public_notes: chordData.public_notes
      });
      
      // Flow will automatically generate SVG
      // Optionally poll for completion
    } catch (error) {
      console.error('Failed to save chord:', error);
    } finally {
      setIsSubmitting(false);
    }
  };
  
  return (
    <div className="chord-editor">
      <MetadataForm data={chordData} onChange={setChordData} />
      
      {chordData.chord_type === 'string' ? (
        <StringChordEditor data={chordData} onChange={setChordData} />
      ) : (
        <PianoChordEditor data={chordData} onChange={setChordData} />
      )}
      
      <div className="preview-section">
        <ChordPreview data={chordData} />
      </div>
      
      <button onClick={handleSubmit} disabled={isSubmitting}>
        {isSubmitting ? 'Saving...' : 'Save Chord'}
      </button>
    </div>
  );
}
```

### 2.5 Preview Component (Client-Side Only)

```typescript
// src/components/editor/ChordPreview.tsx
import Chord from '@tombatossals/react-chords';
import { generateChordChart } from 'piano-chord-charts';
import type { ChordData } from '@/lib/types';

export function ChordPreview({ data }: { data: ChordData }) {
  if (data.chord_type === 'string') {
    const chordFormat = {
      frets: data.json.positions[0].frets,
      fingers: data.json.positions[0].fingers,
      barres: data.json.positions[0].barres || [],
      capo: data.json.positions[0].capo || false
    };
    
    const instrument = {
      strings: data.json.instrument.stringCount,
      fretsOnChord: 4,
      name: data.json.instrument.name,
      tunings: { standard: data.json.instrument.strings }
    };
    
    return <Chord chord={chordFormat} instrument={instrument} />;
  }
  
  if (data.chord_type === 'piano') {
    const keys = data.json.keys
      .filter(k => k.pressed)
      .map(k => k.note);
    
    return generateChordChart({
      highlightKeys: keys.join(' '),
      size: 5,
      format: 'compact'
    });
  }
  
  return null;
}
```

---

## Phase 3: Directus Flow Setup (Server-Side SVG Generation)

### 3.1 Flow Configuration

**Flow Name**: "Generate Chord SVG"

**Trigger**: Event hook
- Type: `action` (non-blocking)
- Scope: `items.create`, `items.update`
- Collections: `chord_data`

### 3.2 Flow Operations

#### Operation 1: Read Chord Data

```json
{
  "key": "read_chord_data",
  "type": "item-read",
  "collection": "chord_data",
  "query": {
    "fields": ["*", "json"]
  }
}
```

#### Operation 2: Generate SVG (Exec)

```json
{
  "key": "generate_svg",
  "type": "exec",
  "code": "module.exports = async function(data) {\n  const chordData = data.read_chord_data.json;\n  \n  if (chordData.chord_type === 'string') {\n    // String chord SVG generation\n    const { renderToString } = require('react-dom/server');\n    const React = require('react');\n    const Chord = require('@tombatossals/react-chords').default;\n    \n    const chordFormat = {\n      frets: chordData.positions[0].frets,\n      fingers: chordData.positions[0].fingers,\n      barres: chordData.positions[0].barres || [],\n      capo: chordData.positions[0].capo || false\n    };\n    \n    const instrument = {\n      strings: chordData.instrument.stringCount,\n      fretsOnChord: 4,\n      name: chordData.instrument.name,\n      tunings: { standard: chordData.instrument.strings }\n    };\n    \n    const html = renderToString(\n      React.createElement(Chord, { chord: chordFormat, instrument })\n    );\n    \n    const svgMatch = html.match(/<svg[^>]*>[\\s\\S]*?<\\/svg>/);\n    return { svg: svgMatch ? svgMatch[0] : '' };\n  }\n  \n  if (chordData.chord_type === 'piano') {\n    const { generateChordChart } = require('piano-chord-charts');\n    const keys = chordData.keys\n      .filter(k => k.pressed)\n      .map(k => k.note);\n    \n    const svg = generateChordChart({\n      highlightKeys: keys.join(' '),\n      size: 5,\n      format: 'compact'\n    });\n    \n    return { svg };\n  }\n  \n  throw new Error('Unknown chord type');\n}"
}
```

#### Operation 3: Upload SVG File

```json
{
  "key": "upload_svg",
  "type": "request",
  "method": "POST",
  "url": "{{ $env.DIRECTUS_URL }}/files",
  "headers": [
    {
      "header": "Authorization",
      "value": "Bearer {{ $env.DIRECTUS_TOKEN }}"
    },
    {
      "header": "Content-Type",
      "value": "multipart/form-data"
    }
  ],
  "body": "{{ generate_svg.svg }}"
}
```

**Note**: May need to use Directus SDK operation instead of raw request.

#### Operation 4: Update Chord Data with File Reference

```json
{
  "key": "update_chord_data",
  "type": "item-update",
  "collection": "chord_data",
  "key": "{{ read_chord_data.id }}",
  "data": {
    "json": {
      "{{ read_chord_data.json }}": "{{ read_chord_data.json }}",
      "svg_file": {
        "id": "{{ upload_svg.data.id }}",
        "url": "{{ upload_svg.data.url }}",
        "filename": "{{ upload_svg.data.filename_disk }}"
      },
      "generated_at": "{{ $trigger.date_created }}"
    }
  }
}
```

### 3.3 Server Dependencies

Install in Directus container/installation:

```bash
npm install @tombatossals/react-chords react react-dom
npm install piano-chord-charts
```

**Alternative**: If Directus exec operation cannot use external packages, use external API service (see Option 3 below).

---

## Phase 4: Alternative Approaches

### Option A: External API Service (If Flow Exec Limited)

If Directus Flow exec operation cannot use npm packages:

1. **Create external Node.js service**:
   ```typescript
   // chord-svg-service/index.ts
   import express from 'express';
   import { generateChordSVG } from './svg-generator';
   
   const app = express();
   
   app.post('/generate', async (req, res) => {
     const chordData = req.body;
     const svg = await generateChordSVG(chordData);
     res.json({ svg });
   });
   ```

2. **Flow calls external service**:
   ```json
   {
     "key": "generate_svg",
     "type": "request",
     "method": "POST",
     "url": "{{ $env.SVG_GENERATOR_URL }}/generate",
     "body": "{{ read_chord_data.json }}"
   }
   ```

### Option B: Client-Side Generation Only

If server-side generation is not feasible:

- Generate SVG in browser
- Upload SVG file to Directus
- Save chord_data with file reference
- No Flow needed (or Flow only validates/organizes)

**Trade-offs**: Less consistent, but simpler setup.

---

## Phase 5: Implementation Checklist

### Schema Setup
- [ ] Add `chord_key` field to `chord_data`
- [ ] Add `chord_suffix` field to `chord_data`
- [ ] Add `chord_type` field to `chord_data`
- [ ] Add `difficulty` field to `chord_data`
- [ ] Add `string_count` field to `chord_data` (conditional)
- [ ] Add `position_index` field to `chord_data`
- [ ] Add `is_barre` field to `chord_data` (conditional)
- [ ] Add `capo_fret` field to `chord_data` (conditional)
- [ ] Configure field conditions (show string fields only when `chord_type = "string"`)

### Frontend Development
- [ ] Install dependencies (`@tombatossals/react-chords`, `piano-chord-charts`, `@directus/sdk`)
- [ ] Create TypeScript types
- [ ] Set up Directus client
- [ ] Create `ChordEditor` component
- [ ] Create `ChordPreview` component
- [ ] Create `StringChordEditor` component
- [ ] Create `PianoChordEditor` component
- [ ] Create `MetadataForm` component
- [ ] Create editor page route
- [ ] Add styling (Tailwind CSS)

### Directus Flow Setup
- [ ] Create flow "Generate Chord SVG"
- [ ] Configure event trigger
- [ ] Add "read_chord_data" operation
- [ ] Add "generate_svg" exec operation (or external API call)
- [ ] Add "upload_svg" operation
- [ ] Add "update_chord_data" operation
- [ ] Test flow with sample data
- [ ] Install server dependencies (if using exec)

### Testing
- [ ] Test string chord creation
- [ ] Test piano chord creation
- [ ] Test SVG generation for string chords
- [ ] Test SVG generation for piano chords
- [ ] Test file upload to Directus
- [ ] Test chord_data update with file reference
- [ ] Test editor preview
- [ ] Test form validation

---

## Phase 6: Future Enhancements

### Potential Improvements

1. **Auto-generate `name` field**:
   - Generate from `chord_key` + `chord_suffix` + instrument + position
   - Example: "C Major (Acoustic Guitar - Position 1)"

2. **Separate `chord` collection** (if needed):
   - Create `chord` collection for chord names only
   - Link `chord_data` to `chord` via M2O
   - Currently not necessary - `chord_key` + `chord_suffix` is sufficient

3. **Multiple piano positions**:
   - Support multiple octaves/positions like string chords
   - Add `position_index` for piano chords

4. **SVG optimization**:
   - Minify SVG output
   - Remove unnecessary attributes
   - Optimize paths

5. **Batch operations**:
   - Import multiple chords at once
   - Bulk SVG generation

---

## Decision Points Summary

### ✅ Resolved Decisions

1. **Schema**: Unified `chord_data` collection (not separate collections)
2. **Architecture**: Client-side preview + Server-side generation
3. **Chord Types**: Support both "string" and "piano" in same collection
4. **SVG Generation**: Server-side via Directus Flow (with client preview)

### ⚠️ Open Questions

1. **Name generation**: Auto-generate `name` field or manual entry?
2. **Piano positions**: Support multiple positions/octaves for piano?
3. **Flow limitations**: Can Directus exec use npm packages, or need external API?
4. **File organization**: What folder structure for SVG files in Directus?

---

## Next Steps

1. **Start with Phase 1**: Set up Directus schema
2. **Then Phase 2**: Build frontend editor
3. **Then Phase 3**: Configure Directus Flow
4. **Test end-to-end**: Create chord → Generate SVG → Verify file

This consolidated plan provides a clear roadmap for implementing the complete chord data system with SVG generation support.
