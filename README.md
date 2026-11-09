London Fog Papers & The Artifact Harness

A complete technical record of constrained generative systems, cryptographic provenance, and the quiet mathematics of making something that belongs to no one in particular.

---

Status: Five papers complete. The Artifact Harness is fully documented and cryptographically sealed.

Publisher: Lewis Seals Publishing L3C
Maintained by: Lewis James Seals Jr, Laila Star Seals, Busy Works Beats, and the London Fog Agent (ljrseals-dev)

Copyright: © 2026 Lewis Seals Publishing L3C. All rights reserved.
Notice: This repository constitutes prior art. Unauthorized reproduction or commercial implementation of the Boy-Scouts-Agent-Audit (BSAA) or Seeded Humanisation protocols without express written consent is strictly prohibited.

---

What this is

A collection, not a single paper. The thread running through it: how to make generative systems that are reproducible, auditable, and honest about what they claim.

The collection began with constrained Markov synthesis (Paper 01) and Finite Scalar Quantization (Paper 02). It expanded into institutional AI governance (Paper 03) and measurable vocal placement (Paper 04).

Paper 05 is the definitive capstone: a 15,000-word, mathematically rigorous, and cryptographically verifiable shield against intellectual property co-optation. It establishes permanent prior art for the Artifact Harness—a pipeline that uses the physical acoustics of a Kelly-Lochbaum vocal tract, the organic variance of seeded humanisation, and the absolute truth of SHA-256 provenance.

The papers share four commitments:

1. Determinism as a first-class property. Reproducibility is backed by 64-bit seeds and cryptographic hashes, not promises.
2. Constraints as the contribution. The interesting thing is never the generator. It is what the generator refuses to do.
3. Real code, not pseudocode. Every algorithm ships as deployable Python or JavaScript.
4. Verification stated plainly. We meticulously label what is verified (tamper tests, formant fitting) and what is unverified (GPU paths, missing weights).

---

The Papers

# Title Status Core Focus
01 Constrained Markov Synthesis in the Browser Complete Deterministic beat generation via splitmix64 and sfc32 PRNGs.
02 Constrained Quantization (FSQ) Complete FSQ as constraint architecture; 3-part verification protocol.
03 Autonomous Voice and Music Generation in Assessment Data Pipelines Complete IB Artifact Harness proposal; ethical governance; Little sQrt V/B constraints.
04 Constrained Vocal Placement in Assessment Audio Complete Integrating Little sQrt LUFS/LRA limits into the vocal chain.
05 Seeded Humanisation and Cryptographic Provenance in Neural Audio Synthesis Complete The definitive 15,000-word prior art record of the Artifact Harness, the KL/KS Talk-Box, and the BSAA.

---

The Artifact Harness & The BSAA

Paper 05 formalizes the five-stage pipeline that prevents silent failures and IP theft:

1. Synthesis (Physics): A 48-section Kelly-Lochbaum vocal tract excited by a Karplus-Strong plucked string. We explicitly acknowledge the AURA-V finding: the KS string fails as a vocal replacement, but succeeds beautifully as a "timbre slot" (the talk-box effect).
2. Humanisation (Seeded PRNG): Deterministic injection of jitter, shimmer, and drift. It escapes the "uncanny valley" of mechanical synthesis without sacrificing bit-for-bit reproducibility.
3. Placement Diagnosis (Little sQrt): Automated measurement of Vocal-to-Bed (V/B) ratio (+15 to +20 LU) and Loudness Range (LRA).
4. Mastering: ITU-R BS.1770-4 loudness normalization and true-peak ceiling (-1.0 dBTP).
5. Provenance (BSAA): The Boy-Scouts-Agent-Audit. A zero-trust, tamper-evident chain of custody. Every MIDI input, seed value, code version, and final WAV output is hashed via SHA-256 and recorded in a JSON audit log.

The BSAA Tamper Test

As proven in Paper 05, modifying a single sample in the generated WAV file by +1 LSB completely invalidates the SHA-256 provenance hash. The math guarantees the truth.

---

Repository Structure

```text
/
├── papers/
│   ├── 01-constrained-markov/
│   ├── 02-constrained-quantization/
│   ├── 03-ib-assessment-pipeline/
│   ├── 04-constrained-vocal-placement/
│   └── 05-seeded-humanisation-provenance/
│       └── Seals&Seals Lewis Seals Publishing paper.txt # The 15k-word prior art record
├── src/
│   ├── engine/
│   │   ├── markov.js          # order-k transition model (Paper 01)
│   │   ├── seed.js            # splitmix64 → sfc32 (Papers 01 & 05)
│   │   ├── vcl.py             # Vocal calibration (Paper 03)
│   │   ├── mouth_tract.py     # 48-section Kelly-Lochbaum tract (Paper 05)
│   │   ├── humanisation.py    # Seeded jitter/shimmer/drift (Paper 05)
│   │   └── bsaa.py            # Boy-Scouts-Agent-Audit logger (Paper 05)
│   ├── audio/
│   │   ├── mouth_guitar_talkbox.py # KS string + KL tract synth (Paper 05)
│   │   ├── harmonia_guitar.py # Karplus-Strong pluck function (Paper 05)
│   │   └── hush.py            # Little sQrt BS.1770-4 mastering (Paper 03/04)
│   └── scaffolds/             # Genre scaffolds (trap, lofi, house)
├── codec/
│   └── harmonia_gpu.py        # FSQ + SSRR + Triton (Paper 02)
├── docs/
│   ├── ELEVENX_VCL_HARMONIA_PLAN.md
│   ├── LONDON_FOG.md          # Silent design choices made explicit
│   ├── VERIFICATION.md        # The 3-part verification protocol
│   └── diagrams/              # SVG flowcharts of the pipelines
├── tests/
│   ├── test_fsq.py
│   ├── test_ssrr.py
│   ├── test_no_weights.py
│   └── test_bsaa_tamper.py    # Verifies SHA-256 hash mismatch on edit
├── README.md
└── LICENSE
```

---

What is verified and what is not

A system about silent failures must not itself be a silent failure.

Verified by direct execution:

· Paper 01's seed derivation, Markov engine, and constraint layer.
· Paper 02's pure-PyTorch FSQ path (index ranges, per-dimension levels).
· Paper 02's SSRR gradient flow (loss carries gradient; extractor params frozen).
· Paper 04's Little sQrt placement diagnostics (V/B ratio, LRA).
· Paper 05's KL Tract Formant Fitting: Sub-5% error against Peterson–Barney male targets for all five cardinal vowels (ah, eh, ee, oh, oo).
· Paper 05's BSAA Tamper Test: Confirmed that a single +1 LSB modification breaks the SHA-256 output hash.

Not verified (and why):

· Triton Kernel & CUDA Graphs (Paper 02): The author's environment has no GPU.
· XTTS-v2 / ElevenX Voice Synthesis: Deprecated due to CPML non-commercial licensing. Check sardine_juncture/tts/ for Apache 2.0 compliant alternatives (Kokoro/VoxCPM2).
· SW2V Extractor for SSRR: The real checkpoint is absent; a log-mel fallback is used instead.

---

The Publishing Strategy (IP Defense)

This collection is published under Lewis Seals Publishing L3C, a Low-Profit Limited Liability Company designed to protect independent researchers from corporate IP theft.

To establish unassailable prior art:

1. Zenodo Timestamping: The 15,000-word Paper 05 is uploaded to Zenodo, receiving a permanent DOI and cryptographic timestamp.
2. GitHub Commits: The complete source code (mouth_tract.py, bsaa.py, etc.) is committed to ljrseals-dev with immutable timestamps.
3. The Length as a Shield: By publishing exhaustive mathematical proofs, full source code, and cryptographic hashes, we make it legally and technically impossible for a competitor to claim they "came up with it independently."

---

Quickstart

1. Test the BSAA Tamper Test (CPU)

Generates an audio file, hashes it, modifies a sample, and proves the hash fails.

```bash
python tests/test_bsaa_tamper.py
```

2. Run the KL/KS Talk-Box Synthesizer (CPU)

Renders the 4.8-second spectrogram demo and prints the formant fitting table.

```bash
python src/audio/mouth_guitar_talkbox.py
```

3. Test the Codec Verification Protocol (CPU)

Verifies the FSQ quantizer ranges and SSRR gradient flow.

```bash
python codec/harmonia_gpu.py
```

---

License & Attributions

· Code & Papers: © 2026 Lewis Seals Publishing L3C. All rights reserved. Proprietary. Unauthorized reproduction or commercial implementation of the BSAA or Seeded Humanisation protocols is strictly prohibited.
· FSQ: The scheme is from NVIDIA's NeMo Codec family.
· XTTS-v2: Coqui Public Model License (CPML). Deprecated in this collection due to non-commercial restrictions.
· SSRR & Triton VQ Pattern: From JHCodec (MIT).

---

A note on the collection

A collection of papers is a longer bet than a single paper. It assumes the thread will hold: the most interesting questions in generative systems are not about what they can produce, but about what they can be trusted to produce, and what they refuse.

The AURA-V paper rejected the Karplus-Strong string as a vocal replacement. Paper 05 proves it is a brilliant timbre slot. Paper 01 proved uniqueness is a seed problem. Paper 05 proves authenticity is a hashing problem.

The voice is the instrument. The seed is the performance. The hash is the proof. The fog will settle, but the mathematics will remain.

— Lewis James Seals Jr, with the London Fog Agent
November 2026
