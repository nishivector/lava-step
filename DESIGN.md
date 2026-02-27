# Lava Step — Design Document
Round 14

---

## IDENTITY

**Game name:** Lava Step
**Tagline:** You don't cross the lava. You conduct it three beats ahead.
**What is the player:** A Geomancer — a compact, heat-suited human figure no bigger than a thumb against the scale of a volcanic caldera. She moves forward relentlessly. Your job is not to guide her feet — your job is to think for her feet two seconds before she needs them.

**World feel:**
You are inside a volcanic caldera at the moment before a new vent opens — the air presses close and hot, every surface either glowing amber or sweating steam. The lava channels don't roar; they hiss, a deep pressurized sound like the planet breathing through clenched teeth. Black basalt platforms gleam wet between the flows, lit from below by orange sub-surface light that casts shadows you learn to read like a clock.

**Emotional experience:** Conducted urgency

**Reference games:**
- **Gran Turismo** — Every crystal tap carries tactile weight and consequence. A placement that's 400ms late means molten floor under a mid-stride foot. There are no soft mistakes.
- **R-Type** — You are pattern-reading three simultaneous flow-streams and committing to a traversal path before it fully exists. You operate at the edge of prediction, not reaction.
- **Thumper** — The game has a pulse. 94 BPM tectonic rhythm underlies every decision. The best players don't tap to the level; they tap *with* it.

---

## VISUAL SPEC

**Background color:** `#C4612A` — burnt amber volcanic floor. RGB: 196, 97, 42. All channels above 30. ✓

**Primary color:** `#F5D060` — crystal-hot shimmer. Crystal platforms, player highlight ring, bloom source.

**Secondary color:** `#EFF0E8` — steam-white solidified basalt. Solid walkable platforms, UI backgrounds.

**Accent color:** `#FF3A1A` — magma-red danger. Melting platform final 800ms, death flash, boulder warning pulse.

**Bloom:** YES. Strength: 0.8. Threshold: 0.55. Applied to crystal platforms and lava channel edges. NOT to background.

**Vignette:** YES. 25% radius, 0.4 opacity. Color: `#2B1000` (deep lava-shadow brown).

**Camera:** Angled top-down at 40° from vertical. Three.js perspective camera.
- `camera.position.set(0, 12, 10)`
- `camera.lookAt(0, 0, -2)`
- FOV: 52°. Near: 0.1. Far: 100.
- Floor recedes toward top of screen. Perspective foreshortening is intentional — channels near top appear narrower (depth cue for planning).

**Player silhouette:** Upright, small, white-hot-edged
- Capsule geometry 0.3 units wide × 0.8 units tall
- Body `#EFF0E8` with `#F5D060` rim-light outline (additive blend)
- Casts shadow on floor behind her (directional light from above-right)
- Idle bob: ±0.04 units vertical at 1.4s period

---

## SOUND SPEC

**Music:** 94 BPM two-layer track.
- Layer 1 (always): deep tectonic bass drone — sine oscillator at 42Hz, LFO wobble 0.15Hz, amplitude ±20%.
- Layer 2 (builds through play): high glassine arpeggiated melody — A minor pentatonic, octave 5–6, sixteenth-note arpeggios. Sharp attack, decay 0.3s, no sustain.

**Music BPM:** 94

**Music responds to:**
- Crystal placement: each tap triggers next arp note in sequence (melody becomes player-driven)
- Danger zone (no crystal ahead, grace window active): bass LFO amplitude increases to ±60%, subtle distortion
- Boulder on screen: resonant THUD on nearest downbeat; kick on beats 1 and 3 activates
- Death: all layers cut except bass, which pitches down 3 semitones over 1.5s then fades
- Level complete: melody plays full ascending run, bass resolves to root, 2s reverb bloom

**Sound effects (6):**

1. **Crystallize snap** — Sharp transient, obsidian splitting.
   `new Tone.MetalSynth({ frequency: 1200, envelope: { attack: 0.001, decay: 0.05, release: 0.1 }, resonance: 3200, modulationIndex: 32 })` at velocity 0.9. Concurrent `Tone.Synth` at 800Hz, 0.02s decay for body.

2. **Platform melt (hiss-sizzle)** — 0.8s white noise through falling lowpass.
   `new Tone.NoiseSynth({ noise: { type: "white" }, envelope: { attack: 0.05, decay: 0.7, sustain: 0, release: 0.1 } })` through `new Tone.Filter({ frequency: 3000, rolloff: -24 })`. Automate filter 3000→400Hz over 0.8s.

3. **Player footstep** — Glassy tink. `new Tone.Synth({ oscillator: { type: "triangle" }, envelope: { attack: 0.001, decay: 0.12, sustain: 0, release: 0.05 } })` at 880Hz (right) and 660Hz (left), alternating.

4. **Boulder impact/crystal shatter** — Low membranous THUD + high crystal-burst.
   `new Tone.MembraneSynth({ pitchDecay: 0.15, octaves: 6, envelope: { attack: 0.001, decay: 0.4 } })` at 40Hz. Plus 6× rapid MetalSynth hits at 800–1600Hz, 20ms spacing.

5. **Death (lava submersion)** — Wet pressurized sizzle descending.
   `new Tone.NoiseSynth` + `Tone.Distortion(0.4)` + `Tone.Filter` (lowpass, 2000Hz). Automate filter 2000→150Hz over 1.5s. Concurrent `Tone.Synth` at 200Hz pitching to 60Hz over 1.5s.

6. **Level complete shimmer** — Major chord bloom.
   `new Tone.PolySynth(Tone.Synth)` — A5, C#5, E5, A6 simultaneously. `{ attack: 0.01, decay: 0.3, sustain: 0.6, release: 2.0 }`. Through `Tone.Reverb({ decay: 3, wet: 0.7 })`. Duration 3s.

---

## MECHANIC SPEC

**Core loop:** Tap any lava channel to freeze a crystal platform at that point; the platform melts based on sub-surface current speed (readable from its shadow direction/length); cross the platform chain before it dissolves, always tapping 2–3 steps ahead to build a timing multiplier.

**Primary input:**
- `pointerdown on lava channel`: Instantly creates crystal platform at tap position. Formation animation: ice-crystal ripple expands from point over 300ms (walkable from t=0). Melt timer starts immediately.
- `pointerdown on solid basalt`: No effect.
- `pointerup`: No effect.
- `pointermove across multiple lava zones`: Triggers crystals at each contact point — rapid multi-crystal placement by dragging is valid skilled play.
- **Mobile touch controls**: No movement buttons needed — tap anywhere on the lava to place a crystal. The game is tap-only.

**Dead state:**
- Kill condition: Player auto-advances. If player's foot would land in a lava gap with no crystal within ±0.5 world units, a 500ms grace window opens. No crystal in time = death.
- Death animation: Player sinks 0.8 units into lava over 1500ms. Screen flash `#FF3A1A` at 0.4 opacity for 200ms.
- Respawn: 2000ms delay. Player respawns at last checkpoint (every 5 successful lava crossings = new checkpoint).
- Lives: 3 per level (3 crystal icons, top-right). 0 lives = full level restart.

**Key timing values:**

| Value | Amount |
|---|---|
| Beat duration at 94 BPM | 638ms |
| 1-beat-ahead multiplier threshold | 638ms before foot lands → ×2 |
| 2-beat-ahead multiplier threshold | 1276ms before foot lands → ×3 |
| 3-beat-ahead multiplier threshold | 1914ms before foot lands → ×4 |
| Crystal formation animation | 300ms (walkable from t=0) |
| Grace window before death | 500ms |
| Crystal lifetime — slow zone (≤1.5 u/s current) | 4600ms |
| Crystal lifetime — medium zone (2.0–3.5 u/s) | 3700ms |
| Crystal lifetime — fast zone (4.0–5.5 u/s) | 2500ms |
| Eddy zone lifetime multiplier | ×1.4 |
| Crystal melt warning (cracking) | Final 800ms — `#EFF0E8`→`#FF3A1A`, cracks appear |
| Player walk speed Level 1 | 1.2 u/s |
| Player walk speed Level 2 | 1.4 u/s |
| Player walk speed Level 3 | 1.6 u/s |
| Player walk speed Level 4 | 1.8 u/s |
| Player walk speed Level 5 | 2.0 u/s |
| Boulder roll speed Level 3 | 2.5 u/s |
| Boulder roll speed Level 4 | 3.5 u/s |
| Boulder roll speed Level 5 | 4.5 u/s |
| Shadow reveal delay after placement | 200ms |
| Death animation | 1500ms |
| Respawn delay | 2000ms |

**Sub-surface current and shadow system:**
Each lava channel has hidden current vectors per segment (speed 0–5.5 u/s, direction: channel-parallel ±30° at bends). When a crystal is placed, after 200ms it casts a directional shadow on the basalt floor — a soft amber elongated ellipse pointing *opposite* to sub-surface current direction. Shadow length = `current_speed × 0.3` world units. Shadow opacity: 0.7. The shadow reveals current speed and direction but NOT crystal lifetime directly — player infers lifetime from shadow length. Eddy zones (2–4 per screen length) have current ≤0.8 u/s; crystals there last ×1.4. Eddies are ONLY revealed via another crystal's shadow — no surface markings.

**Difficulty curve:**

| Level | Channels | Flow speeds | Player speed | Boulders |
|---|---|---|---|---|
| 1 | 2 | 1.5, 1.5 u/s | 1.2 u/s | None |
| 2 | 3 | 1.5, 3.0, 1.5 u/s | 1.4 u/s | None |
| 3 | 3 | 2.0, 3.5, 2.0 u/s | 1.6 u/s | 1 lane |
| 4 | 4 | 1.5, 4.5, 3.0, 4.5 u/s | 1.8 u/s | 2 lanes |
| 5 | 4 | 2.0, 5.0, 4.5, 3.5 u/s | 2.0 u/s | 2 + speed pulses ±1.0 u/s every 3 beats |

**Win condition:** Player reaches exit arch — `#F5D060` hexagonal gate at far end, pulsing at 94 BPM.

**Lose condition:** 0 lives remaining.

**Score:**
- Cross a channel on crystal: 100 pts base
- Timing multiplier: ×1 (<1 beat), ×2 (1–2 beats), ×3 (2–3 beats), ×4 (3+ beats ahead)
- Eddy placement bonus: +50 pts
- Boulder shatter bonus: +500 pts
- No-death level bonus: +1000 pts
- Time bonus: +10 pts/sec under par (L1=45s, L2=60s, L3=75s, L4=90s, L5=120s)
- Consecutive ×2+ chain: +0.1× multiplier per link, max 3.0×, resets on miss or death

---

## LEVEL DESIGN

### Level 1 — THE FIRST STEP
**What's new:** Everything. Single mechanic. Tutorial that doesn't announce itself.
**Parameters:** 2 lava channels, both 1.5 u/s. Crystal lifetime 4600ms. Player speed 1.2 u/s. 15 units depth. No eddies.
**Layout:** Entry basalt (2u) → Channel A (2u wide) → Basalt bridge (1.5u) → Channel B (2u wide) → Exit arch.
**Special:** On the very first crystal ever placed, time slows to 30% for one beat (638ms) — player sees formation animation clearly. Happens once, never again.
**Duration:** 30–45 seconds.

### Level 2 — READING THE FLOOR
**What's new:** Shadow system activates meaningfully. Middle channel is fast (3.0 u/s). Eddies introduced.
**Parameters:** Channels A/C at 1.5 u/s, Channel B at 3.0 u/s. Eddies: 2 per B channel. Player 1.4 u/s.
**Layout:** 3 channels, basalt strips between. B channel has 2 visible-width eddy pockets.
**Duration:** 45–60 seconds.

### Level 3 — THE BOULDER
**What's new:** Boulder introduced. Crystallizing under it shatters boulder AND clears path (wow moment).
**Parameters:** 3 channels, medium/fast/medium. 1 boulder rolling left-to-right at 2.5 u/s. Boulder radius: 0.6 units. Warning pulse: 500ms amber ring before entry.
**Boulder rule:** If crystal is placed within 0.3u of boulder's path within 200ms of boulder arrival → crystal shatters boulder; +500 pts; passage cleared. Miss → boulder destroys crystal; blocks for 1.5s.
**Duration:** 60–75 seconds.

### Level 4 — MISMATCHED SPEEDS
**What's new:** 4 channels, two at 4.5 u/s (very fast, 2500ms crystal lifetime). 2 boulders on separate lanes.
**Parameters:** Pattern: slow/fast/slow/fast. Two boulders, independent timing. Player 1.8 u/s.
**Challenge:** Fast channels require 3-beat-ahead placement to have any time at all. First time players fail here repeatedly.
**Duration:** 75–90 seconds.

### Level 5 — THE GAUNTLET
**What's new:** Speed pulses (±1.0 u/s oscillation every 3 beats), 2 boulders + timing pulses simultaneously.
**Parameters:** 4 channels, varied. Pulses shift crystal lifetime by ±1000ms dynamically. Player 2.0 u/s.
**Challenge:** Shadows now show a *changing* current — the length varies in real time as pulses hit. Players must read the shadow right before placement, not 2s before.
**Duration:** 90–120 seconds.

---

## THE MOMENT

Level 3: The player encounters the boulder for the first time. They freeze — they've been placing crystals away from it. Then, running out of space, they tap directly under the boulder as it rolls over their crystal. The boulder SHATTERS. Crystal shards, golden bloom burst, +500 pts. The path ahead is clear. The player didn't plan it. They will spend the rest of the game trying to do it on purpose.

---

## EMOTIONAL ARC

**First 30 seconds:** Frantic single-step tapping. Crystals melt before the player's foot lands. The player dies once, learns the grace window exists. Dies again. Then slows down, taps 2 steps ahead. Survives. The pace clicks.

**After 2 minutes:** The player has found the groove — tapping in a sweeping gesture 3 steps ahead, reading the amber floor for shadow direction before committing. The rhythm of tap-tap-tap-cross feels musical. They're inside the 94 BPM pulse.

**Near the win (Level 5):** Four channels. Shadows rippling as speed pulses hit. Two boulders on independent timers. The player is running the whole caldera like a conductor: reading, placing, crossing, shattering. They feel enormous.

---

## THIS GAME'S IDENTITY IN ONE LINE

"This is the game where you learn to read the shadow on the floor that tells you the future."

---

## START SCREEN

**Idle animation (Three.js canvas — game-world specific):**
- The full Level 1 layout is visible: amber volcanic floor, two lava channels glowing orange-red, basalt platforms between them
- A Geomancer figure stands at the entry platform, idling (gentle bob animation)
- Every 5 seconds: she auto-taps one crystal into Channel A (formation ripple plays), walks across it, taps another, crosses, reaches the exit arch — then the scene resets to start position
- Crystal platforms appear and melt on the lava channels during this loop — the world is breathing
- Ambient ember particles rise from lava channel edges: 3–5 particles/sec, orange → fade, rising 0.5u over 2s

**SVG overlay spec (Options A + C):**

**Option A — SVG glow title:**
```
Text: "LAVA STEP"
Font: monospace, bold, 52px
Fill: #F5D060 (crystal gold)
Filter: feGaussianBlur stdDeviation=8, feComposite over (strong volcanic glow)
Letter spacing: 0.15em
Animation: fadeIn + letter-spacing 0.3em → 0.15em over 1.2s ease-out
```

**Option C — SVG corner brackets (volcanic/tactical frame):**
```
Four L-shaped corner brackets, 28px each arm, 2px stroke, #F5D060
stroke-dasharray=40; stroke-dashoffset=40
Animation: drawBracket 0.8s ease-out, staggered: TL 0.0s, TR 0.1s, BL 0.2s, BR 0.3s
Result: brackets draw in around the title, like targeting reticles locking on
```

Below title: instruction text in monospace 12px `#EFF0E8` opacity 0.7:
`TAP THE LAVA TO FREEZE IT`
Below that: "Tap to begin" blinking at 1s, `#F5D060`.
