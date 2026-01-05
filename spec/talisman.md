# SPECIFICATION: TALISMAN (v1.1)

**Purpose:** A high-contrast, cryptic visualization for Wiremap patches. Prioritizes vertical density and visual impact over schematic completeness.
**Input:** Valid Wiremap patch.
**Output:** UTF-8 Plaintext Block (Monospaced).

---

## 1. Glyph Vocabulary

### 1.1 Node Radicals

Nodes are represented *only* by radicals. No IDs.

| Class | Radical | Meaning |
| --- | --- | --- |
| **Source** | `⊙` | Oscillator, Noise, Input |
| **Shape** | `⧉` | Filter, Wavefolder, EQ |
| **Time** | `◴` | Clock, Timer |
| **Logic** | `∷` | Sequencer, S&H, Logic |
| **Env** | `Λ` | Envelope, Function Gen |
| **LFO** | `≈` | Cyclic Modulator |
| **Amp** | `▷` | VCA, LPG, Attenuator |
| **FX** | `★` | Distortion, Delay, Reverb |
| **Mix** | `∑` | Mixer, Summer |
| **Out** | `Ω` | Main Output |

### 1.2 State Notches

Appended to the radical in fixed order: **[Variant] [Intensity] [Duration]**. Max one char per category.

* **Variant:** `^` (Sharp/Saw/HP), `_` (Flat/Pulse/LP), `~` (Sine/BP).
* **Intensity:** `!` (High drive/feedback), `"` (High resonance/Q).
* **Duration:** `>` (Long decay/release), `|` (Short/Percussive).
* *Example:* `⧉_"` (Lowpass Filter, High Resonance).
* *Example:* `⊙^` (Saw Oscillator).



### 1.3 Connection Strokes (Strict Separation)

* **Audio (The Spine):** `║` (Vertical), `═` (Horizontal). **Sacred.** No control signals may use double strokes.
* **Control (The Wing):** `│` (Vertical), `─` (Horizontal).
* **Feedback:** `«` (Upward return).
* **Crossings:** `┼` (Control over Control), `╫` (Control over Audio).

### 1.4 Junctions

* **Branch:** `╦` (Audio), `┬` (Control).
* **Merge:** `╩` (Audio), `┴` (Control).
* **Hot Sum:** `⨂` (Audio merge without explicit `∑` node).

---

## 2. Encoding Rules

### 2.1 Depth Markers

Modulation depth is encoded by a marker placed **exactly one character before** the contact point with the target line.

* **Low:** `·` (Middle dot)
* **Med:** `•` (Bullet)
* **High:** `●` (Black circle)
* **Invert:** `⊖` (Circled minus)

*Syntax:* `Source ───●║ Target`

### 2.2 Directionality

* **Arrows are Banned.**
* Direction is implied by the layout:
* **Audio:** Always Top-to-Bottom.
* **Control:** Always Satellite-to-Pillar (Inward).
* **Feedback:** Explicit `«` only.



---

## 3. Layout Rules (The Obelisk)

### 3.1 The Pillar (Audio Spine)

The audio chain forms a **single central column**.

* Contains *only* Radicals and Audio Strokes (`║`).
* Adjacency implies flow: `A` above `B` = `A -> B`.
* Explicit strokes `║` are only used for gaps, splits, or visual rhythm.

### 3.2 Satellites (Control Wings)

Control sources hover in columns to the Left/Right.

* **Global Time:** Top Center/Left.
* **Modulators:** Aligned horizontally with their primary target in the Pillar.
* **Injection:** Control lines `─` enter the Pillar from the side.

### 3.3 Crossing Policy

* Control lines pass *behind* Audio lines.
* If a Control line must cross the Audio Pillar to reach the other side, use `╫`.

---

## 4. Constraints

* **Width:** Fixed 80-100 chars (Optimized for mobile/overlay).
* **Palette:** Monochrome safe. No Emoji.
* **Font:** Monospace required.

---

## 5. Reference Render: "Acid Bass"

**Patch Context:**

* **Time:** Clock -> Seq.
* **Audio:** Osc(Saw) -> Filt(LP/Res) -> Dist(Drive) -> VCA -> Out.
* **Control 1:** Seq(Pitch) -> Osc; Seq(Gate) -> Env.
* **Control 2:** Env -> Filt (High Depth); Env -> VCA (Full Depth).
* **Control 3:** Seq(Vel) -> Dist (Low Depth).

**BLACKSEAL Output:**

```text
      ◴
      │
      ∷
    ┌─┘
    │
    │ ⊙^
    │ ║
    │ ⧉_"
    │ ║
    │ ★!
    │ ║
    │ ▷
    │ ║
    │ Ω
    │
  Λ─┼─●║
  │ │  ║
  └─┼──●║
    │
    └──·║

```

**Reading the Rune:**

1. **Pillar:** Saw Osc `⊙^` feeds LP/Res Filt `⧉_"` feeds Drive FX `★!` feeds VCA `▷` feeds Out `Ω`.
2. **Top:** Clock `◴` feeds Seq `∷`. Seq drops a line down the left.
3. **Left Wing:** Env `Λ` injects High Depth `●` into Filter and Full Depth `●` into VCA.
4. **Wiring:** The Sequencer line `│` drops past the Env, injecting Low Depth `·` into the Distortion `★!` (representing Velocity mapping).
