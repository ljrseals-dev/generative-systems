London Fog Papers

A collection of papers on constrained generative systems, seeded architectures, and the quiet mathematics of making something that belongs to no one in particular.

---

Status: Two papers complete. Several in draft or planned.

Maintained by: Lewis James Seals Jr, Laila Star Seals, Busy Works Beats, and the London Fog Agent (ljrseals-dev)

License: MIT (code); CC-BY-4.0 (papers) — see LICENSE

---

What this is

A collection, not a single paper. The thread running through it: how to make generative systems that are reproducible, auditable, and honest about what they claim. Paper 01 is on constrained Markov synthesis in the browser. Paper 02 is on Finite Scalar Quantization as constraint architecture, with a verification protocol for GPU-native neural audio codecs. The next may be on full DSP synthesis, adaptive scaffolds, perceptual uniqueness at scale, or the legal geometry of seed receipts. The collection is named after the weather in which it was written.

The papers share four commitments:

1. Determinism as a first-class property. Every claim of reproducibility is backed by a 64-bit seed and a versioned scaffold, not by a promise.
2. Constraints as the contribution. The interesting thing is never the generator. It is what the generator refuses to do.
3. Real code, not pseudocode. Every algorithm in every paper ships as deployable code. The papers are arguments for software that exists.
4. Verification stated plainly. What is verified by direct execution is labeled as verified. What is not verified — because it requires GPU hardware, or a checkpoint that is not available, or a plugin that could not be run — is labeled as unverified, with the test that will verify it specified. A paper about silent failures that overstates its own verification would be a third silent failure.

---

Papers

# Title Status Path
01 Constrained Markov Synthesis in the Browser: A Seeded, Constraint-Filtered Architecture for Combinatorially Vast Beat Production at Millions-of-Visitors Scale Complete draft /papers/01-constrained-markov/
02 Constrained Quantization: Finite Scalar Quantization as Constraint Architecture, with a Verification Protocol for GPU-Native Neural Audio Codecs Complete draft /papers/02-constrained-quantization/
03 Full DSP Synthesis for Browser-Native Beat Engines Planned (companion to 01) —
04 Adaptive Scaffolds: Preference Learning Without Server-Side Inference Planned —
05 Perceived Uniqueness: A Longitudinal Study of Seed-Receipt Beats Planned (n ≥ 100) —
06 Seed Receipts and Provenance: The Legal Geometry of Generative Uniqueness Planned (with counsel) —
07 GPU Verification Results: The Triton Path, Measured Planned (requires GPU) —

Each paper is self-contained. Read them in any order. If you read one, read Paper 01 first — it establishes the architecture the others assume. Paper 02 assumes Paper 01's event stream as its input.

---

Repository structure

```
/
├── papers/
│   ├── 01-constrained-markov/
│   │   ├── manuscript.md      # The paper
│   │   ├── manuscript.pdf     # Typeset version
│   │   └── code/              # Paper-specific code listings
│   ├── 02-constrained-quantization/
│   │   ├── manuscript.md
│   │   ├── manuscript.pdf
│   │   └── code/              # harmonia_gpu.py, reproduced for reference
│   └── _template/             # Scaffold for new papers
├── src/
│   ├── engine/
│   │   ├── markov.js          # order-k transition model (Paper 01)
│   │   ├── constraints.js     # restricted-row resampling + repair (Paper 01)
│   │   └── seed.js            # splitmix64 → sfc32; layer substreams (Paper 01)
│   ├── audio/
│   │   ├── scheduler.js       # integer-index look-ahead scheduler (Paper 01)
│   │   └── synth.js           # reference renderer (Paper 01)
│   ├── scaffolds/
│   │   ├── trap.json
│   │   ├── lofi.json
│   │   └── house.json
│   └── ui/
│       └── controls.js
├── codec/
│   └── harmonia_gpu.py        # FSQ + SSRR + Triton + graphs (Paper 02)
├── test/
│   ├── markov.test.js
│   ├── seed.test.js
│   ├── constraints.test.js
│   ├── fsq_verify.py          # Paper 02 verification protocol
│   └── verify-receipt.js
├── docs/
│   ├── seed-receipt-format.md
│   ├── scaffold-schema.md
│   ├── reproducibility.md
│   ├── LONDON_FOG.md          # Paper 01 silent choices, made explicit
│   └── VERIFICATION.md        # Paper 02 protocol, in full
├── README.md
├── LICENSE
└── CITATION.cff
```

---

Paper 01 at a glance

The problem. Generative beatmakers fail at scale for one of two reasons: server-side generation cannot sustain millions of unique requests without accruing cost, latency, and state; client-side generation without constraint produces incoherent noise.

The architecture. A 64-bit master seed, derived via splitmix64 and expanded into per-layer sfc32 substreams, guarantees combinatorial uniqueness at visitor scale. A constraint layer — beat-position-dependent matrices, per-genre consonance floors, and a resample-or-repair fallback — prevents the random-walk-into-noise failure that afflicts unconstrained Markov composition. The engine is genre-agnostic; genre specificity lives in downloadable scaffolds under fifty kilobytes compressed.

The result. At four million visitors, the expected number of colliding seed pairs is approximately 4.3 × 10⁻⁷. Perceptual evaluation (n = 30, pilot) found the constrained engine non-inferior to producer-curated loops within δ = 0.5 (TOST; underpowered, framed accordingly). Uniqueness is established analytically rather than empirically.

The contribution. The constraint layer, not the Markov chain. A first-order Markov chain is trivial; the constraint architecture that renders it musically viable without sacrificing uniqueness is not.

Read the full paper: papers/01-constrained-markov/manuscript.md

---

Paper 02 at a glance

The problem. Neural audio codecs fail silently. A residual vector quantizer collapses a codebook without raising an error. A Triton kernel quantizes against the wrong grid without raising an error. A loss term contributes zero gradient for the entire training run without raising an error.

The architecture. Finite Scalar Quantization is presented not merely as a quantizer but as a constraint architecture: the fixed element-wise grid removes codebook collapse by construction. The SSRR loss is analyzed with attention to a specific failure mode — the @torch.no_grad() decorator that silently strips gradient flow — and the correct fix (freeze parameters, not outputs) is derived and verified. A Triton kernel is shown to have silently quantized three of four dimensions against the wrong grid; the per-dimension fix is derived and specified against a GPU test that will verify it.

The verification. A three-part protocol — gradient flow check, per-dimension range check, Triton-versus-eager bit-exactness — catches the failure modes above and is reproducible in any environment. The first two are verified by direct execution on CPU. The third is specified as a GPU-only test, with the reason it cannot be run in the author's environment stated explicitly.

The silent choices, documented. Three design choices Paper 01 left unstated — the semantics of the rhythm matrix state, the onset-to-instrument mapping, and the leap cap on melodic transitions — are documented in docs/LONDON_FOG.md and argued to be constraints in the collection's sense.

The contribution. The verification protocol, and the reframing of FSQ as constraint architecture. Without the protocol, the other contributions are unverified claims.

Read the full paper: papers/02-constrained-quantization/manuscript.md

---

Reproducibility protocol

Every claim of uniqueness in every paper in this collection rests on the following.

A seed receipt is a triple (Paper 01) or a quadruple (Paper 02):

```
{ visitorSeed: 64-bit integer,
  scaffoldVersion: semver string,
  params: { tempo, complexity, ... },
  codecVersion: semver string    # Paper 02 and later
}
```

The receipt determines the beat, bit-for-bit, across browsers and devices. Given the same receipt, the engine produces the same event stream and the codec produces the same audio. Given a different receipt, it produces a different one. This is not a probabilistic claim. It is a cryptographic one.

To verify a beat (from the repository root):

```bash
node test/verify-receipt.js --receipt ./receipts/example.json
```

To regenerate the stimuli from Paper 01's perceptual study:

```bash
node test/regenerate-stimuli.js --paper 01 --out ./stimuli/
```

To run Paper 02's verification protocol:

```bash
python codec/harmonia_gpu.py
```

The output states plainly which parts are verified in the current environment and which are not. On a CPU-only machine, the Triton path is unverified; the paper says so, and the README repeats it.

See docs/reproducibility.md and docs/VERIFICATION.md for the full protocols.

---

Running the engine

Minimal example — generate and play a beat from a seed (Paper 01):

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
  .filter((_, i) => i % 2 === 0);

const ctx = new AudioContext();
const scheduler = new BeatScheduler(ctx, scaffold.tempo ?? 140);
scheduler.start();
```

Minimal example — verify the codec quantizer (Paper 02):

```bash
# CPU-only: verifies the pure-PyTorch path and the verification protocol
python codec/harmonia_gpu.py

# On GPU: additionally verifies the Triton path against eager
# (see paper §4.4 for the expected output)
```

See docs/scaffold-schema.md for the scaffold format, and the individual paper manuscripts for the full architectures.

---

Scaffolds

Scaffolds are data, not code. They are versioned, human-auditable, and under fifty kilobytes compressed. Each one specifies:

· A rhythmic grid (sixteenth-note resolution by default)
· Strong-beat positions and per-position transition matrices
· A melodic order-k model (order 1 for the current papers)
· Per-genre consonance sets (allowed pitch-class intervals)
· A pitch range, tempo, and leap cap

Adding a new genre does not require touching the engine. It requires authoring a scaffold. The collection welcomes scaffold contributions that follow the schema and include test fixtures.

Current scaffolds: trap, lofi, house. Planned: drill, boom-bap, ambient, jungle.

Note on the drum corpus. Paper 01's §4.1 describes a 500-pattern drum corpus that was not available in the archive at the time of writing. The drum data in the current scaffolds is authored, not derived from that corpus. This is documented in docs/LONDON_FOG.md and in the scaffold files themselves. When the corpus is available, the scaffolds will be rebuilt and the measurements re-run. Until then, the scaffolds are reproducible and versioned; they are not claimed to be representative of the corpus.

---

What is verified and what is not

This section exists because a paper about silent failures must not itself be a silent failure.

Verified by direct execution:

· Paper 01's seed derivation, per-layer substreams, and collision analysis (numpy simulation).
· Paper 01's Markov engine, constraint layer, and scheduler-drift fix (unit tests).
· Paper 02's pure-PyTorch FSQ path (per-dimension levels, index ranges, round-trip).
· Paper 02's SSRR gradient flow (loss carries gradient, backward pass produces nonzero gradient, extractor parameters stay frozen).
· Paper 02's per-dimension level arithmetic (bit-exact against a numpy reference).

Not verified:

· Paper 02's Triton kernel on GPU hardware. Structurally correct, numerically derived, unexercised on real hardware.
· Paper 02's CUDA graph capture and replay on GPU hardware. Structurally correct, unexercised.
· Paper 01's end-to-end integration with Paper 02's codec. Specified, not executed.
· The real SW2V checkpoint for SSRR. Paper 02 uses a log-mel fallback; the real checkpoint is not available in the author's environment.
· Crypt2.vst3. The plugin is a Windows binary; the author's environment could not run it. Paper 02's design choices were made without access to the plugin and are stated as the author's choices.

The tests that will verify the unverified items are specified in the papers (Paper 02 §4.4 for the Triton path; Paper 01 §6.2 for the perceptual study; Paper 02 §7.4 for the integration). The collection treats unverified claims as a first-class status, not as a weakness to be hidden.

---

Contributing

Papers in this collection are by invitation or by direct collaboration with the maintainers. Code contributions, scaffold contributions, and reproductions of published results are welcome and encouraged.

For code and scaffolds:

1. Open an issue describing the change.
2. Fork, branch, and submit a pull request against main.
3. Include tests. If you add a scaffold, include a fixture that verifies a known seed produces a known event stream. If you touch the codec, run the verification protocol and paste its output in the PR description.

For paper reproductions:

If you reproduce a result from any paper in this collection — whether by running the verification protocol, regenerating a perceptual stimulus set, or running the Triton-vs-eager test on GPU hardware — please open an issue tagged reproduction. Reproductions are treated as first-class contributions and will be listed in the paper's errata. A reproduction that contradicts a paper's claim is not a failure of the collection; it is the collection working as intended.

For corrections:

A single fabricated or unverifiable citation is grounds for desk rejection at most venues, and grounds for removal from this collection. If you find one, open an issue tagged citation-integrity. We will correct or cut.

For GPU verification specifically:

If you have GPU hardware and would like to run Paper 02's Triton-vs-eager test, please do. The test is in test/fsq_verify.py and its expected output is documented in Paper 02 §4.4. A reproduction that confirms the fix will move Paper 02's status from "unverified on GPU" to "verified on [hardware, driver, CUDA version]." A reproduction that finds a mismatch will be the most valuable contribution anyone can make to this collection.

See CONTRIBUTING.md for the full protocol.

---

Citation

If you use the engine, the codec, or any paper in this collection, please cite the specific paper and the corresponding code:

```bibtex
@article{seals2026constrained,
  title   = {Constrained Markov Synthesis in the Browser: A Seeded,
             Constraint-Filtered Architecture for Combinatorially Vast
             Beat Production at Millions-of-Visitors Scale},
  author  = {Seals, Laila Star and {Busy Works Beats} and {London Fog Agent}},
  journal = {London Fog Papers},
  volume  = {1},
  number  = {1},
  year    = {2026},
  url     = {https://github.com/ljrseals-dev/london-fog-papers}
}

@article{seals2026quantization,
  title   = {Constrained Quantization: Finite Scalar Quantization as
             Constraint Architecture, with a Verification Protocol for
             GPU-Native Neural Audio Codecs},
  author  = {Seals, Lewis James, Jr.},
  journal = {London Fog Papers},
  volume  = {1},
  number  = {2},
  year    = {2026},
  url     = {https://github.com/ljrseals-dev/london-fog-papers}
}

@software{londonfogengine,
  title  = {London Fog Engine},
  author = {{London Fog Agent}},
  year   = {2026},
  url    = {https://github.com/ljrseals-dev/london-fog-engine}
}

@software{harmoniacodec,
  title  = {HarmoniaCodec: GPU-Native Quantizer and SSRR Loss},
  author = {Seals, Lewis James, Jr.},
  year   = {2026},
  url    = {https://github.com/ljrseals-dev/london-fog-papers}
}
```

A CITATION.cff file is included for GitHub's citation widget.

---

License

· Code (/src, /test, /codec, /papers/*/code): MIT. See LICENSE.
· Papers (/papers/*/manuscript.*): CC-BY-4.0. Attribution required; commercial use permitted.
· Scaffolds (/src/scaffolds): MIT, but note that the musical idioms they encode are not owned by anyone. The scaffold data is ours; the style is not.

Third-party attributions:

· FSQ is from the NeMo Codec family (NVIDIA). Check NVIDIA's license terms for redistribution of anything derived from their weights.
· SSRR loss and the Triton VQ pattern are from JHCodec (MIT).
· splitmix64 is from Steele, Lea & Flood (OOPSLA 2014); the reference implementation is public domain.
· sfc32 is from Chris Doty-Humphrey's PractRand; the reference implementation is public domain.

---

A note on the collection

A collection of papers is a longer bet than a single paper. It assumes there will be a second paper, and a third, and that the thread will hold across them. The thread here is simple and, we think, durable: the most interesting questions in generative systems are not about what they can produce, but about what they can be trusted to produce, and what they refuse.

Paper 01 argued this for constraints on musical output. Paper 02 argues it for constraints on the codec — and for the verification protocol that makes the codec's claims checkable rather than merely stated. The papers will hold up or they won't. The seeds will regenerate the beats either way. The verification protocol will run, and the Triton path will be bit-exact or it won't, and the collection will say which.

The fog will settle. The refusals are the contribution.

— Lewis James Seals Jr, with the London Fog Agent (ljrseals-dev)
November 2026
