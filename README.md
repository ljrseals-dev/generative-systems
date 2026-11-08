London Fog Papers & Artifact Harness

A collection of papers, software, and ethical frameworks on constrained generative systems, seeded architectures, and the quiet mathematics of making something that belongs to no one in particular.

---

Status: Three papers complete. Codec and voice pipelines in active development.

Maintained by: Lewis James Seals Jr, Laila Star Seals, Busy Works Beats, and the London Fog Agent (ljrseals-dev)

License: MIT (code); CC-BY-4.0 (papers) — see LICENSE

---

What this is

A collection, not a single paper. The thread running through it: how to make generative systems that are reproducible, auditable, and honest about what they claim.

Paper 01 established constrained Markov synthesis for generative beats. Paper 02 documented Finite Scalar Quantization (FSQ) and a verification protocol for GPU-native codecs. Paper 03 applies this architecture to a high-stakes real-world problem: the International Baccalaureate's assessment data pipeline, proposing an autonomous "Artifact Harness" for local voice synthesis, vocal calibration, and cryptographic provenance.

The papers share four commitments:

1. Determinism as a first-class property. Every claim of reproducibility is backed by a 64-bit seed, a versioned scaffold, and a SHA-256 provenance record.
2. Constraints as the contribution. The interesting thing is never the generator. It is what the generator refuses to do (e.g., FSQ's fixed grid refusing to collapse; consent policies refusing to clone unverified voices).
3. Real code, not pseudocode. Every algorithm ships as deployable Python or JavaScript.
4. Verification stated plainly. What is verified by direct execution is labeled as verified. What is unverified (because it requires GPU hardware, missing weights, or a proprietary plugin) is labeled as unverified, with the test that will verify it specified.

---

Papers & Reports

# Title Status Path
01 Constrained Markov Synthesis in the Browser: A Seeded, Constraint-Filtered Architecture for Combinatorially Vast Beat Production Complete /papers/01-constrained-markov/
02 Constrained Quantization: Finite Scalar Quantization as Constraint Architecture, with a Verification Protocol for GPU-Native Neural Audio Codecs Complete /papers/02-constrained-quantization/
03 Autonomous Voice and Music Generation in Assessment Data Pipelines: A Technical and Ethical Report to the International Baccalaureate (IBO) Complete /papers/03-ib-assessment-pipeline/
04 Full DSP Synthesis for Browser-Native Beat Engines Planned —
05 GPU Verification Results: The Triton Path, Measured Planned (requires GPU) —

---

The Artifact Harness

The IB report introduces the Artifact Harness, a constrained pipeline for autonomous data and audio processing. The architecture is designed for zero-data-leakage (local-first execution) and absolute auditability.

The Pipeline Chain

1. Source (Text / Audio): Text-to-speech via local XTTS-v2 (24 kHz), or a sung/spoken take.
2. Vocal Calibration (VCL): engine/vcl.py applies vocal presets (SING, RAP, neutral) to control noise floor, vowel warp, and prosody lock.
3. Harmonia Codec (Optional): harmonia.py compresses the audio using FSQ (8,5,5,5) with straight-through gradient estimation. Off by default until trained weights exist.
4. Little sQrt Master: ITU-R BS.1770-4 loudness normalisation and true-peak ceiling (-1.0 dBTP).
5. Provenance & Checks: Generates a SHA-256 provenance record, seed receipt, and runs the verification protocol.

See the full plan in ELEVENX_VCL_HARMONIA_PLAN.md and the architectural diagrams in /docs/diagrams/.

Current Implementation Status

· VCL & Hush Master: Built and verified.
· Harmonia / FSQ: Code complete (Paper 02). CPU path verified. Triton GPU path structurally correct but unverified on hardware.
· ElevenX (XTTS-v2): Bridge file exists (sardine_juncture/tts/elevenx_bridge.py), but weights are not bundled. It requires local installation by the user.
· Crypt2.vst3: A Windows binary; not run or verified in this environment.

---

Repository Structure

```text
/
├── papers/
│   ├── 01-constrained-markov/
│   ├── 02-constrained-quantization/
│   └── 03-ib-assessment-pipeline/
├── src/
│   ├── engine/
│   │   ├── markov.js          # order-k transition model (Paper 01)
│   │   ├── constraints.js     # restricted-row resampling (Paper 01)
│   │   ├── seed.js            # splitmix64 → sfc32 (Paper 01)
│   │   ├── vcl.py             # Vocal calibration (SING/RAP) (Paper 03)
│   │   └── vocal_chain.py     # Ordered pipeline runner (Paper 03)
│   ├── audio/
│   │   ├── scheduler.js       # Look-ahead scheduler (Paper 01)
│   │   ├── synth.js           # Reference renderer (Paper 01)
│   │   └── hush.py            # Little sQrt BS.1770-4 mastering (Paper 03)
│   └── scaffolds/             # Genre scaffolds (trap, lofi, house)
├── codec/
│   └── harmonia_gpu.py        # FSQ + SSRR + Triton (Paper 02)
├── sardine_juncture/
│   └── tts/
│       └── elevenx_bridge.py  # Local XTTS-v2 hookup (Paper 03)
├── docs/
│   ├── ELEVENX_VCL_HARMONIA_PLAN.md
│   ├── LONDON_FOG.md          # Silent design choices made explicit
│   ├── VERIFICATION.md        # The 3-part verification protocol
│   └── diagrams/              # SVG flowcharts of the pipelines
├── tests/
│   ├── test_fsq.py
│   ├── test_ssrr.py
│   ├── test_triton_eager.py   # GPU required
│   └── test_no_weights.py     # Enforces no model weights in repo
├── README.md
└── LICENSE
```

---

What is verified and what is not

This section exists because a system about silent failures must not itself be a silent failure.

Verified by direct execution:

· Paper 01's seed derivation, Markov engine, and constraint layer.
· Paper 02's pure-PyTorch FSQ path (index ranges, per-dimension levels).
· Paper 02's SSRR gradient flow (loss carries gradient; extractor params frozen).
· Paper 03's Hush master stage (finite output, true-peak bounded) and VCL calibration (bounded noise floor).

Not verified (and why):

· Triton Kernel & CUDA Graphs (Paper 02): The author's environment has no GPU. The code is structurally correct but requires execution on NVIDIA hardware.
· XTTS-v2 / ElevenX Voice Synthesis: The weights are not bundled (per licensing and repo size constraints). The bridge exists but requires local installation.
· SW2V Extractor for SSRR: The real checkpoint is absent; a log-mel fallback is used instead.
· Audio Quality Claims: Paper 03 explicitly states that the pipeline guarantees structural integrity and provenance, not that the synthetic voice sounds perfectly human.

---

Quickstart

1. Test the Codec Verification Protocol (CPU)

Verifies the FSQ quantizer ranges, SSRR gradient flow, and extraction logic.

```bash
python codec/harmonia_gpu.py
```

2. Test the Vocal Chain (CPU)

Runs a synthetic vowel through the VCL and Hush master to check for finite output and true-peak compliance.

```bash
python src/engine/vocal_chain.py --source synthetic_vowel.wav
```

3. Generate a Beat (Browser/Node)

```javascript
import { deriveVisitorSeed, createLayerRng } from './src/engine/seed.js';
import { buildOrderKModel, generateSequence } from './src/engine/markov.js';

const scaffold = await fetch('./src/scaffolds/trap.json').then(r => r.json());
const seed = await deriveVisitorSeed("user-uuid-123", scaffold.version, '2026-11');
const melodyRng = createLayerRng(seed, 'melody');
// ... (see Paper 01 for full scheduler implementation)
```

---

Contributing

Papers in this collection are by invitation or direct collaboration. Code contributions, scaffold contributions, and reproductions of published results are welcome.

For GPU Reproductions:
If you have access to NVIDIA hardware, we need your help verifying the Triton-vs-eager bit-exactness test (Paper 02 §4.4). Run tests/test_triton_eager.py. A reproduction that confirms the fix will move Paper 02's status from "unverified" to "verified." A reproduction that finds a mismatch will be the most valuable contribution anyone can make.

For Corrections:
A single fabricated citation is grounds for desk rejection and removal from this collection. Open an issue tagged citation-integrity if you find one.

---

License & Attributions

· Code: MIT
· Papers: CC-BY-4.0
· FSQ: The scheme is from NVIDIA's NeMo Codec family. Check their license for redistribution of derived weights.
· XTTS-v2: Coqui Public Model License (CPML). Non-commercial use only. Commercial deployment requires a separate license.
· SSRR & Triton VQ Pattern: From JHCodec (MIT).

---

A note on the collection

A collection of papers is a longer bet than a single paper. It assumes the thread will hold: the most interesting questions in generative systems are not about what they can produce, but about what they can be trusted to produce, and what they refuse.

Paper 01 argued this for constraints on musical output. Paper 02 argued it for constraints on the codec. Paper 03 argues it for constraints on institutional AI deployment—where human rights, job displacement, and student data are at stake.

The fog will settle. The seeds will regenerate the outputs either way. The refusals are the contribution.

— Lewis James Seals Jr, with the London Fog Agent
November 2026
