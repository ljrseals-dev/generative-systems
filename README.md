London Fog Papers

A collection of papers on constrained generative systems, seeded architectures, and the quiet mathematics of making something that belongs to no one in particular.

---

Status: In formation. One paper complete, several in draft or planned.

Maintained by: Laila Star Seals, Busy Works Beats, and the London Fog Agent (ljrseals-dev)

License: MIT (code); CC-BY-4.0 (papers) — see LICENSE

---

What this is

A collection, not a single paper. The thread running through it: how to make generative systems that are reproducible, auditable, and honest about what they claim. The first paper is on constrained Markov synthesis in the browser. The next may be on full DSP synthesis, adaptive scaffolds, perceptual uniqueness at scale, or the legal geometry of seed receipts. The collection is named after the weather in which it was written.

The papers share three commitments:

1. Determinism as a first-class property. Every claim of reproducibility is backed by a 64-bit seed and a versioned scaffold, not by a promise.
2. Constraints as the contribution. The interesting thing is never the generator. It is what the generator refuses to do.
3. Real code, not pseudocode. Every algorithm in every paper ships as deployable JavaScript. The papers are arguments for software that exists.

---

Papers

# Title Status Path
01 Constrained Markov Synthesis in the Browser: A Seeded, Constraint-Filtered Architecture for Combinatorially Vast Beat Production at Millions-of-Visitors Scale Complete draft /papers/01-constrained-markov/
02 Full DSP Synthesis for Browser-Native Beat Engines Planned (companion) —
03 Adaptive Scaffolds: Preference Learning Without Server-Side Inference Planned —
04 Perceived Uniqueness: A Longitudinal Study of Seed-Receipt Beats Planned (n ≥ 100) —
05 Seed Receipts and Provenance: The Legal Geometry of Generative Uniqueness Planned (with counsel) —
06 Progressive Enhancement for Generative Audio: Browser Compatibility as Design Constraint Planned —

Each paper is self-contained. Read them in any order. If you read one, read Paper 01 first — it establishes the architecture the others assume.

---

Repository structure

```
/
├── papers/
│   ├── 01-constrained-markov/
│   │   ├── manuscript.md      # The paper
│   │   ├── manuscript.pdf     # Typeset version
│   │   └── code/              # Paper-specific code listings
│   └── _template/             # Scaffold for new papers
├── src/
│   ├── engine/
│   │   ├── markov.js          # order-k transition model
│   │   ├── constraints.js     # restricted-row resampling + repair
│   │   └── seed.js            # splitmix64 → sfc32; layer substreams
│   ├── audio/
│   │   ├── scheduler.js       # integer-index look-ahead scheduler
│   │   └── synth.js           # reference renderer
│   ├── scaffolds/
│   │   ├── trap.json
│   │   ├── lofi.json
│   │   └── house.json
│   └── ui/
│       └── controls.js
├── test/
│   ├── markov.test.js
│   ├── seed.test.js
│   └── constraints.test.js
├── docs/
│   ├── seed-receipt-format.md
│   ├── scaffold-schema.md
│   └── reproducibility.md
├── README.md
├── LICENSE
└── CITATION.cff
```

---

Reproducibility protocol

This is the part that matters most, and the part most likely to be skipped. Every claim of uniqueness in every paper in this collection rests on the following:

A seed receipt is a triple:

```
{ visitorSeed: 64-bit integer,
  scaffoldVersion: semver string,
  params: { tempo, complexity, ... } }
```

The receipt determines the beat, bit-for-bit, across browsers and devices. Given the same receipt, the engine produces the same event stream. Given a different receipt, it produces a different one. This is not a probabilistic claim. It is a cryptographic one.

To verify a beat (from the repository root):

```bash
npm install
node test/verify-receipt.js --receipt ./receipts/example.json
```

This regenerates the event stream from the receipt and compares it against the stored hash. If they match, the beat is authentic. If they don't, either the receipt is wrong or the scaffold version has changed. Either way, you know.

To regenerate the stimuli from Paper 01's perceptual study:

```bash
node test/regenerate-stimuli.js --paper 01 --out ./stimuli/
```

The seeds and scaffold versions used in the study are archived in papers/01-constrained-markov/receipts/. This is a genuine strength of the architecture and a genuine obligation of the collection: if a reader doubts a perceptual result, they can regenerate the exact audio and listen for themselves.

See docs/reproducibility.md for the full protocol.

---

Running the engine

Minimal example — generate and play a beat from a seed:

```javascript
import { deriveVisitorSeed, createLayerRng } from './src/engine/seed.js';
import { buildOrderKModel, generateSequence } from './src/engine/markov.js';
import { sampleConsonant } from './src/engine/constraints.js';
import { BeatScheduler } from './src/audio/scheduler.js';
import { playKick, playSnare, playHat, playBass } from './src/audio/synth.js';

const scaffold = await fetch('./src/scaffolds/trap.json').then(r => r.json());
const visitorId = localStorage.getItem('visitor-id') ?? crypto.randomUUID();
localStorage.setItem('visitor-id', visitorId);

const seed = await deriveVisitorSeed(visitorId, scaffold.version, '2026-11');
const drumsRng = createLayerRng(seed, 'drums');
const melodyRng = createLayerRng(seed, 'melody');

const melodyModel = buildOrderKModel(scaffold.melody.trainingSequence, 1);
const melody = generateSequence(melodyModel, [60], 16, melodyRng)
  .filter((_, i) => i % 2 === 0); // half-notes for demonstration

const ctx = new AudioContext();
const scheduler = new BeatScheduler(ctx, scaffold.tempo ?? 140);
scheduler.start();
```

See docs/scaffold-schema.md for the scaffold format, and papers/01-constrained-markov/manuscript.md for the full architecture.

---

Scaffolds

Scaffolds are data, not code. They are versioned, human-auditable, and under fifty kilobytes compressed. Each one specifies:

· A rhythmic grid (sixteenth-note resolution by default)
· Strong-beat positions and per-position transition matrices
· A melodic order-k model (order 1 for the current papers)
· Per-genre consonance sets (allowed pitch-class intervals)
· A pitch range and tempo

Adding a new genre does not require touching the engine. It requires authoring a scaffold. The collection welcomes scaffold contributions that follow the schema and include test fixtures.

Current scaffolds: trap, lofi, house. Planned: drill, boom-bap, ambient, jungle.

---

Contributing

Papers in this collection are by invitation or by direct collaboration with the maintainers. Code contributions, scaffold contributions, and reproductions of published results are welcome and encouraged.

For code and scaffolds:

1. Open an issue describing the change.
2. Fork, branch, and submit a pull request against main.
3. Include tests. If you add a scaffold, include a fixture that verifies a known seed produces a known event stream.

For paper reproductions:

If you reproduce a result from any paper in this collection, please open an issue tagged reproduction. Reproductions — whether they confirm or contradict — are treated as first-class contributions and will be listed in the paper's errata.

For corrections:

A single fabricated or unverifiable citation is grounds for desk rejection at most venues, and grounds for removal from this collection. If you find one, open an issue tagged citation-integrity. We will correct or cut.

See CONTRIBUTING.md for the full protocol.

---

Citation

If you use the engine or any paper in this collection, please cite the specific paper and the engine:

```bibtex
@article{seals2026constrained,
  title   = {Constrained Markov Synthesis in the Browser: A Seeded,
             Constraint-Filtered Architecture for Combinatorially Vast
             Beat Production at Millions-of-Visitors Scale},
  author  = {Seals, Laila Star and Busy Works Beats and {London Fog Agent}},
  journal = {London Fog Papers},
  volume  = {1},
  number  = {1},
  year    = {2026},
  url     = {https://github.com/ljrseals-dev/london-fog-papers}
}

@software{londonfogengine,
  title  = {London Fog Engine},
  author = {{London Fog Agent}},
  year   = {2026},
  url    = {https://github.com/ljrseals-dev/london-fog-engine}
}
```

A CITATION.cff file is included for GitHub's citation widget.

---

License

· Code (/src, /test, /papers/*/code): MIT. See LICENSE.
· Papers (/papers/*/manuscript.*): CC-BY-4.0. Attribution required; commercial use permitted.
· Scaffolds (/src/scaffolds): MIT, but note that the musical idioms they encode are not owned by anyone. The scaffold data is ours; the style is not.

---

A note on the collection

A collection of papers is a longer bet than a single paper. It assumes there will be a second paper, and a third, and that the thread will hold across them. The thread here is simple and, we think, durable: the most interesting questions in generative systems are not about what they can produce, but about what they can be trusted to produce, and what they refuse.

The fog will settle. The papers will either hold up or they won't. The seeds will regenerate the beats either way.

— London Fog Agent, ljrseals-dev
November 2026
