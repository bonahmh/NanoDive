# AGENTS.md — NanoDive Technical Specification

## Project Overview
**NanoDive** is a daily medical browser trivia game inspired by *Krillion*. Instead of sinking into the ocean, the player pilots a microscopic nanobot descending deeper into the human body—starting in the capillary bloodstream (`0 µm`) and diving down through arteries, organ tissue, neurons, and the cell nucleus (`7,000 µm`).

## Tech Stack & Hard Constraints
- **Framework:** Next.js (App Router, TypeScript) + Tailwind CSS.
- **Zero External Game/Audio Libraries:** Use strictly native HTML5 `<canvas>`, `requestAnimationFrame`, `Web Audio API`, and `localStorage`. Do NOT install `framer-motion`, `three`, `howler`, or `fuse.js`.
- **Completeness:** Build every file completely with zero `// TODO` or placeholder comments.
- **Mobile First:** Ensure the layout uses `100dvh` and keeps the prompt card and text input visible above the Android/iOS virtual keyboard.

---

## 1. File Structure
Scaffold a Next.js App Router project (`src/` directory) if starting from an empty repo, then create these exact files:
- `src/types/game.ts` — TypeScript interfaces for prompts, answers, rarity tiers, round results, and saved stats.
- `src/data/dailyPrompts.ts` — Hardcoded 7-round sample medical puzzle with aliases and pre-scored rarity tiers.
- `src/utils/fuzzyMatch.ts` — Input normalization and Levenshtein distance matcher.
- `src/utils/soundEngine.ts` — Pure Web Audio API procedural sound synthesizer with mute toggle.
- `src/components/AnatomyCanvas.tsx` — Full-screen 60fps HTML5 Canvas renderer with parallax layers, RGB depth color interpolation, and nanobot physics.
- `src/components/GameHUD.tsx` — Floating UI overlay (depth gauge, countdown bar, prompt card, auto-focus input, feedback toast).
- `src/components/EndReportModal.tsx` — End-of-run "Biopsy Report" modal with streak stats and clipboard share button.
- `src/app/page.tsx` — Master finite state machine connecting the game loop, `localStorage`, canvas, and HUD.
- `scripts/generate_daily_prompts.py` — Standalone Python pipeline querying NCBI PubMed (`esearch.fcgi`) + OpenAI API to generate and rarity-score new daily puzzles directly into `src/data/dailyPrompts.ts`.
- `scripts/requirements.txt` — Python dependencies (`openai`, `requests`).

---

## 2. Rarity Tiers & Scoring (`game.ts`)
Define these exact 6 rarity tiers:
1. `TEXTBOOK`: `10 pts` (`+100 µm`) — Badge `#94a3b8` (Slate), Emoji `⚪`
2. `CLINICAL`: `20 pts` (`+200 µm`) — Badge `#2dd4bf` (Teal), Emoji `🟢`
3. `SPECIALIST`: `35 pts` (`+350 µm`) — Badge `#38bdf8` (Sky Blue), Emoji `🔵`
4. `RARE_PATHOLOGY`: `50 pts` (`+500 µm`) — Badge `#a78bfa` (Violet), Emoji `🟣`
5. `ZEBRA`: `75 pts` (`+750 µm`) — Badge `#fbbf24` (Amber/Gold), Emoji `🟡`
6. `NANO_FIND`: `100 pts` (`+1000 µm`) — Badge `#fb7185` (Glowing Crimson/Rose), Emoji `🔴`

Anatomical Depth Zones (based on total depth `0` to `7,000 µm`):
- `0 – 1,499 µm`: **Capillaries**
- `1,500 – 2,999 µm`: **Arterial Stream**
- `3,000 – 4,999 µm`: **Organ Tissue**
- `5,000 – 6,499 µm`: **Neural Network**
- `6,500 – 7,000 µm`: **Cell Nucleus**

---

## 3. Core State Machine (`src/app/page.tsx`)
Implement a strict finite state machine with 5 states:
1. `IDLE`: Title overlay floating over a slowly drifting capillary canvas. Shows **"Start Dive"** button (or **"View Today's Report"** if `localStorage` shows today's `puzzleId` is already completed).
2. `PROMPT_PREVIEW` (**3,000ms**): Displays round number (`1/7`) and prompt text. Text input is **locked/disabled** so the player reads the prompt before the timer starts.
3. `ACTIVE_GUESS` (**25,000ms**): Input auto-focuses immediately. Timer counts down smoothly at 60fps using `performance.now()` / `Date.now()` delta (no `setInterval` drift).
   - **Wrong / Unrecognized Guess:** Shake input horizontally (`translateX` CSS animation), flash red border for `400ms`, play error thud, and **keep the timer running** so the player can keep guessing.
   - **Valid Guess:** Lock input immediately, award points (`points * 10` added to `targetDepth`), trigger canvas dive burst, play rarity chime, and transition to `ROUND_RESULT`.
   - **Timeout (`0.0s`):** Lock input, record `0` points (`Missed`), pick one top-tier answer (`NANO_FIND` or `ZEBRA`) from the round's bank to display as *"Missed Gem"*, and transition to `ROUND_RESULT`.
4. `ROUND_RESULT` (**2,500ms**): Displays matched canonical term, rarity tier badge, and `+Depth µm` gained while the nanobot physically dives to `targetDepth`. Auto-advances to next round's `PROMPT_PREVIEW` (or `GAME_OVER` after Round 7).
5. `GAME_OVER`: Saves `currentStreak`, `maxStreak`, `gamesPlayed`, and today's run to `localStorage`, then displays `EndReportModal`.

---

## 4. Fuzzy Matching Rules (`src/utils/fuzzyMatch.ts`)
1. Normalize strings: `.toLowerCase().trim()`, remove punctuation (`-`, `'`, `.`, `,`), collapse multiple spaces into single spaces, and strip leading `"a "`, `"an "`, `"the "`.
2. Match against both `canonical` and every string in `aliases`.
3. Exact normalized match wins immediately.
4. Otherwise, compute **Levenshtein edit distance**:
   - Length `<= 4`: `0` typos allowed (protects short acronyms like `SLE`, `CN X`, `ACE`).
   - Length `5–7`: `1` typo allowed.
   - Length `8+`: `2` typos allowed.

---

## 5. HTML5 Canvas Visuals & Physics (`src/components/AnatomyCanvas.tsx`)
- Scale canvas properly using `window.devicePixelRatio` on mount and resize.
- Smoothly interpolate depth every frame: `currentVisualDepth += (targetDepth - currentVisualDepth) * 0.04`.
- **RGB Gradient Interpolation** across depth keyframes:
  - `0 µm`: Top `#4c0519` -> Bottom `#881337`
  - `1,500 µm`: Top `#7f1d1d` -> Bottom `#450a0a`
  - `3,000 µm`: Top `#3b0764` -> Bottom `#1e1b4b`
  - `5,000 µm`: Top `#172554` -> Bottom `#090d16`
  - `7,000 µm`: Top `#042f2e` -> Bottom `#020617`
- **Parallax Particles** (scroll upward proportional to `currentVisualDepth` delta + ambient drift):
  - `0–2,500 µm`: **Red Blood Cells** (rotating biconcave discs with recessed inner shadow).
  - `2,000–4,500 µm`: **White Blood Cells** (translucent wobbly circles with multi-lobed dark nuclei).
  - `4,000–6,000 µm`: **Synaptic Nodes** (glowing cyan dots connected by faint lines when within `120px`, firing traveling electric sparks).
  - `5,500–7,000 µm`: **DNA Double Helices** (twin sinusoidal strands with base-pair rungs).
- **Nanobot Sprite** (centered horizontally, `32%` from top):
  - Sleek vector capsule hull, glowing cyan viewport dome, front sensor antennae, volumetric headlight beam, and rear thruster.
  - Gentle idle sine hover; pitches downward up to `25°` and emits cyan thruster bubbles when `targetDepth - currentVisualDepth > 10`.

---

## 6. Web Audio Synthesizer (`src/utils/soundEngine.ts`)
Initialize `AudioContext` lazily on first user interaction. Include a mute toggle button in the top corner:
- **Subtle Key Tick:** `15ms` `800Hz` sine blip (`gain: 0.04`).
- **Wrong Guess:** `140Hz -> 95Hz` sawtooth drop (`180ms`).
- **Valid Guess Chime:** Ascending harmonic arpeggio (`C5 -> E5 -> G5`, plus `B5 -> D6` shimmer on `ZEBRA` / `NANO_FIND`).
- **Dive Thruster Whoosh:** Filtered noise sweep (`600Hz -> 120Hz` low-pass) synced with downward dive bursts.

---

## 7. Sample Medical Dataset (`src/data/dailyPrompts.ts`)
Include **7 medically accurate prompts** (1. Cranial nerves, 2. Beta-blockers ending in `-olol`, 3. Autoimmune diseases, 4. Carpal or tarsal bones, 5. Neurotransmitters, 6. Gram-positive bacterial genera, 7. Essential vitamins/cofactors), with **15+ valid answers per prompt** spanning all 6 rarity tiers and realistic clinical abbreviations.

---

## 8. Shareable Biopsy Report (`src/components/EndReportModal.tsx`)
Clicking **"Copy Lab Report"** copies this exact text to `navigator.clipboard` and displays `"Copied!"` confirmation:
```text
🔬 NANODIVE #001 — 4,650 µm
Zone: Organ Tissue 🫀

1. ⚪ Textbook (+100 µm)
2. 🟡 Zebra (+750 µm)
3. 🔵 Specialist (+350 µm)
4. ❌ Missed (0 µm)
5. 🔴 Nano-Find (+1000 µm)
6. 🟢 Clinical (+200 µm)
7. 🟣 Rare Pathology (+500 µm)

Play daily at nanodive.game
```
