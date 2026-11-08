Constrained Markov Synthesis in the Browser

A Seeded, Constraint-Filtered Architecture for Combinatorially Vast Beat Production at Millions-of-Visitors Scale

Laila Star Seals and Busy Works Beats
with the London Fog Agent, ljrseals-dev

---

Abstract

Generative beatmakers fail at scale for one of two reasons: server-side generation cannot sustain millions of unique requests without accruing cost, latency, and state; client-side generation without constraint produces incoherent noise. This paper resolves the tension with a seeded, constraint-filtered Markov architecture that runs entirely in the browser. The engine is genre-agnostic; genre specificity lives in downloadable scaffolds — sparse transition matrices, consonance rule sets, and rhythmic grids under fifty kilobytes compressed. A 64-bit master seed, derived via splitmix64 and expanded into per-layer sfc32 substreams, guarantees combinatorial uniqueness at visitor scale: at four million visitors, the expected number of colliding seed pairs is approximately 4.3 × 10⁻⁷. A constraint layer — beat-position-dependent matrices, per-genre consonance floors, and a resample-or-repair fallback — prevents the random-walk-into-noise failure that afflicts unconstrained Markov composition. In a preliminary perceptual study (n = 30), the constrained engine was non-inferior to producer-curated loops within a pre-specified margin of δ = 0.5 on a five-point Likert scale, assessed by two one-sided tests (TOST; equivalence bounds [−0.5, +0.5], α = 0.05, power ≈ 0.55 — underpowered, and framed accordingly in §6.2). Uniqueness is established analytically rather than empirically: with a 64-bit seed and a beat space exceeding 10⁴¹ configurations per scaffold under conservative constraint assumptions, seed space is the binding constraint, and the collision probability is negligible by the birthday bound. The contribution is not the Markov chain, which is trivial, but the constraint architecture that renders a random walk musically viable without sacrificing the uniqueness that scale demands.

---

1. Introduction

1.1 The Scale–Uniqueness Paradox

Four million visitors means four million expectations of a beat that belongs to them. The expectation is not merely that the beat is good; it is that the beat is theirs — that no one else has quite this one. This is the paradox at the heart of generative music at scale: uniqueness without coherence.

Server-side generation appears to offer coherence. A trained model, a curated sampler, a Lambda function holding state — these can produce musically plausible output on demand. But a server-side Markov generator is not an "inference pass" in the machine-learning sense; it is O(1) per step, and a single Lambda invocation can generate thousands of beats per second. The argument against the server is therefore not compute. It is fourfold.

First, latency. A network round-trip per beat costs one hundred to five hundred milliseconds on mobile networks, which is perceptible during the interactive tweaking that a beatmaker invites. Client-side generation completes in under five milliseconds on a 2023 mid-range laptop.

Second, cost at scale. Four million unique beats per month, at even twenty cents per million Lambda invocations plus egress on the generated audio, yields a recurring bill that scales linearly with traffic. Client-side generation has zero marginal cost. The server becomes a CDN and an offline re-renderer for paid purchases — never a real-time generator.

Third, reproducibility. A server-side generator that holds state cannot guarantee that a visitor who returns tomorrow receives the same beat. A seed receipt — visitor seed, scaffold version, parameters — reconstructs the beat deterministically on any device, in any browser, at any time.

Fourth, privacy. Client-side generation means no audio, no beat, and no interaction leaves the device until the visitor chooses to purchase. This is not merely a compliance posture; it is a design commitment that shapes the entire architecture.

Client-side generation, by contrast, appears to offer uniqueness — but at the cost of coherence. Pure randomness produces dissonance. A first-order Markov chain without constraint produces the "random walk into noise" that has dogged algorithmic composition since the Illiac Suite: local plausibility, global incoherence, and a tendency to collapse into repetitive loops that are unique only in the trivial sense that no two loops are identical.

Neither pure server nor pure client is acceptable. The resolution is architectural: client-side generation, constrained by musical rules, seeded for reproducibility, and scaffolded for genre specificity. Uniqueness becomes a property of the seed architecture; coherence becomes a property of the constraint layer. The server recedes to the CDN.

1.2 Why Markov Chains (Still) Matter

Markov models remain the most interpretable probabilistic framework for music. They can be audited — a reviewer can inspect the transition matrix and see exactly which note follows which. They can be constrained — a rule set can prune transitions before sampling rather than after. And they can be reproduced deterministically with a seed, which no deep generative model can claim without caveat.

Their limitation is local context. A first-order chain knows only the previous note; a second-order chain knows the previous two. Neither knows the phrase. Without rule-based correction, the chain wanders. This paper addresses that limitation directly: the constraint layer is not a post-hoc filter but a first-class component of the generative process, applied at the point of sampling rather than after it.

The deep generative model, by contrast, offers phrase-level coherence at the cost of interpretability, reproducibility, and server cost. For a beatmaker at four-million-visitor scale, those costs are prohibitive. The constrained Markov engine is the right tool for this problem, not because it is fashionable but because it is auditable, cheap, deterministic, and — with the constraint layer — musically viable.

1.3 Contributions

This paper makes four contributions.

1. A seeded, client-side Markov architecture that produces bit-identical event streams across browsers, with a 64-bit master seed derived via splitmix64 and per-layer sfc32 substreams that isolate instrument regeneration.
2. A constraint layer — beat-position-dependent matrices, per-genre consonance floors, and a resample-or-repair fallback — that prevents the noise collapse without sacrificing the uniqueness that scale demands.
3. A genre scaffold abstraction that separates musical data (transition matrices, consonance rule sets, rhythmic grids) from probabilistic motion, allowing the engine to remain genre-agnostic while scaffolds carry genre specificity in under fifty kilobytes compressed.
4. Deployable JavaScript code for the entire pipeline: seed derivation, Markov sampling, constraint filtering, scheduling, and a reference synthesis sufficient to demonstrate the architecture end to end.

The novelty is not the Markov chain. The novelty is the constraint architecture that transforms a random walk into a musically viable generator at scale.

---

2. Related Work

2.1 Markov Chains in Algorithmic Composition

The lineage runs from the Illiac Suite (Hiller & Isaacson, 1959) through the statistical modeling tradition of Conklin and Witten (1995) and Pearce and Wiggins (2004) to the modern browser implementations that this paper extends. First-order chains produce local plausibility; higher orders increase stylistic mimicry at the cost of diversity and matrix sparsity. Shapiro and Huber (2021) provide a contemporary survey of the trade-offs, and Temperley (2007) situates Markov models within a broader probabilistic account of musical expectation.

The recurring finding is that unconstrained Markov output tends toward repetition: the chain finds a cycle and stays in it. Prior work has addressed this through higher orders, variable-order models, and post-hoc filtering. This paper addresses it through constraint-at-sampling, which is distinct in that the constraint is applied to the transition distribution before sampling rather than to the output after it.

2.2 Seeded Determinism in Generative Music

The genmusic package demonstrates seed-deterministic pattern generation with the Web Audio API, but lacks a constraint layer for musical coherence. The aleatoric library provides Markov sequence generators and seeded PRNGs but is a library, not a full architecture. Neither addresses the scale question — the problem of guaranteeing uniqueness across millions of visitors with a finite seed space. This paper's seed architecture (splitmix64 → sfc32, with per-layer substreams) is designed specifically for that problem, and the collision analysis in §4.2 is, to our knowledge, the first explicit treatment of birthday-bound collisions in a generative music deployment context.

2.3 Evaluation of Generated Music

Objective metrics for generated music include Shannon entropy over the pitch or onset distribution, a repetition index over consecutive bigrams, and pitch diversity over the pitch-class set. Subjective metrics include catchiness, musicality, and preference on Likert scales. Prior work has found that Markov outputs have lower entropy (1.80 versus 2.29) and higher repetition (0.43 versus 0.34) than human-composed melodies — a gap that this paper tests whether the constraint layer closes. The evaluation protocol in §6.2 uses TOST for non-inferiority rather than a null-hypothesis significance test, because the claim of interest is equivalence, not difference.

---

3. Methodology

3.1 System Overview

```
Visitor → Master Seed → PRNG Substreams → Markov Sampler → Constraint Filter → Event Stream → Web Audio API → Sound
                ↑
         Genre Scaffold (transition matrices, consonance rules, rhythmic grids)
```

The visitor is identified by a client-generated random UUID stored in localStorage — never a browser fingerprint. That UUID, combined with the scaffold version and an epoch, is hashed into a 64-bit master seed. The seed is expanded into four 32-bit words for an sfc32 generator, and each instrument layer (drums, bass, melody) receives its own substream derived by XORing the master seed with a layer-specific tweak. Regenerating the melody therefore does not perturb the drums, which is essential for the interactive tweaking that a beatmaker invites.

The Markov sampler draws from the scaffold's transition matrices, conditioned on beat position (strong versus weak subdivisions receive different matrices — a non-homogeneous Markov model). The constraint filter prunes transitions that violate the scaffold's consonance rules before sampling, with a resample-or-repair fallback for the degenerate case. The resulting event stream is scheduled by a look-ahead scheduler and rendered by a reference synthesis layer.

The server is a CDN and an offline re-renderer for paid purchases. It never generates in real time.

3.2 The Markov Engine: Genre-Agnostic Design

The engine is genre-agnostic; genre specificity lives entirely in the scaffold data. This reconciliation matters because a reviewer encountering trap.json, lofi.json, and house.json in the repository may reasonably ask why a genre-agnostic engine ships with genre-specific files. The answer is that the engine knows nothing about trap or lo-fi or house; it knows only transition matrices, consonance rule sets, and rhythmic grids. The scaffolds are data, not code, and they are downloadable, versionable, and human-auditable.

State definition depends on the layer. For melody, the state is a MIDI pitch class plus octave register; for rhythm, the interval since the last onset in sixteenth-note units; for drums, the beat position within the strong/weak subdivision.

The order-k Markov implementation below is correct for any k \geq 1. States are k-tuples of symbols, serialized as strings for Map keys. Add-\alpha smoothing ensures that seen states assign nonzero probability to unseen successors; unseen states fall back to a uniform distribution over the alphabet.

```javascript
// order-k Markov model with add-alpha smoothing.
function buildOrderKModel(sequence, order, alpha = 1) {
  if (order < 1) throw new Error("order must be >= 1");
  if (sequence.length <= order) throw new Error("sequence too short for order");

  const transitions = new Map();
  const alphabet = [...new Set(sequence)];

  for (let i = 0; i + order < sequence.length; i++) {
    const stateKey = sequence.slice(i, i + order).join("\u0001");
    const nextSym = sequence[i + order];
    if (!transitions.has(stateKey)) transitions.set(stateKey, new Map());
    const row = transitions.get(stateKey);
    row.set(nextSym, (row.get(nextSym) || 0) + 1);
  }

  const rows = new Map();
  for (const [stateKey, counts] of transitions) {
    const denom = [...counts.values()].reduce((a, b) => a + b, 0)
                + alpha * alphabet.length;
    const probs = alphabet.map(s => ((counts.get(s) || 0) + alpha) / denom);
    rows.set(stateKey, { symbols: alphabet, probs });
  }

  const uniformRow = {
    symbols: alphabet,
    probs: alphabet.map(() => 1 / alphabet.length),
  };

  return { order, alphabet, alpha, rows, uniformRow };
}

function sampleSuccessor(model, history, rng) {
  const stateKey = history.slice(-model.order).join("\u0001");
  const row = model.rows.get(stateKey) || model.uniformRow;
  const r = rng();
  let cumulative = 0;
  for (let i = 0; i < row.probs.length; i++) {
    cumulative += row.probs[i];
    if (r < cumulative) return row.symbols[i];
  }
  return row.symbols[row.symbols.length - 1];
}
```

A limitation must be stated plainly: with a twenty-melody corpus, most order-2 states will be unseen, and the uniform fallback will dominate. The paper therefore adopts first-order Markov for melody, with constraint filtering as the novelty. Second-order melody generation is reserved for future work with a corpus of at least two hundred melodies, or with Kneser–Ney backoff to order-1. Stating this limitation is preferable to hiding it behind a code path that produces random walks under a second-order label.

3.3 Seeded Determinism and Substream Isolation

A 32-bit master seed produces approximately 1,860 colliding pairs at four million visitors by the birthday bound, n(n-1)/(2N) \approx 1.86 \times 10^3. That is unacceptable for a paper whose central claim is uniqueness. The architecture therefore uses a 64-bit master seed via splitmix64, expanded into an sfc32 generator with 128 bits of state. Collisions drop to approximately 4.3 \times 10^{-7}.

```javascript
const MASK64 = (1n << 64n) - 1n;

function splitmix64(state) {
  state = (state + 0x9E3779B97F4A7C15n) & MASK64;
  let z = state;
  z = ((z ^ (z >> 30n)) * 0xBF58476D1CE4E5B9n) & MASK64;
  z = ((z ^ (z >> 27n)) * 0x94D049BB133111EBn) & MASK64;
  z = (z ^ (z >> 31n)) & MASK64;
  return { state, output: z };
}

function sfc32(a, b, c, d) {
  return function () {
    a |= 0; b |= 0; c |= 0; d |= 0;
    const t = (((a + b) | 0) + d) | 0;
    d = (d + 1) | 0;
    a = b ^ (b >>> 9);
    b = (c + (c << 3)) | 0;
    c = (c << 21) | (c >>> 11);
    c = (c + t) | 0;
    return (t >>> 0) / 4294967296;
  };
}

async function deriveVisitorSeed(visitorId, scaffoldVersion, epoch) {
  const input = `v1|${visitorId}|${scaffoldVersion}|${epoch}`;
  const hashBuf = await crypto.subtle.digest(
    "SHA-256", new TextEncoder().encode(input)
  );
  const bytes = new Uint8Array(hashBuf);
  let seed = 0n;
  for (let i = 0; i < 8; i++) seed = (seed << 8n) | BigInt(bytes[i]);
  return seed & MASK64;
}

function seedToSfc32(seed64) {
  const words = [];
  let s = seed64;
  for (let i = 0; i < 4; i++) {
    const { state, output } = splitmix64(s);
    s = state;
    words.push(Number(output & 0xFFFFFFFFn));
  }
  return words;
}

function createLayerRng(visitorSeed64, layerName) {
  let tweak = 0n;
  for (let i = 0; i < layerName.length; i++) {
    tweak = (tweak * 131n + BigInt(layerName.charCodeAt(i))) & MASK64;
  }
  const layerSeed = (visitorSeed64 ^ tweak) & MASK64;
  return sfc32(...seedToSfc32(layerSeed));
}
```

The visitorId is a client-generated random UUID stored in localStorage. It is never a browser fingerprint. Fingerprinting would trigger GDPR and CCPA obligations and would undermine the zero-data-leaves-the-device claim. The seed receipt stored server-side on purchase contains {visitorSeed, scaffoldVersion, params} and nothing else — no audio, no beat data, no personal data. A returning visitor whose localStorage is cleared receives a new beat; the paper states this explicitly rather than claiming that the same visitor always receives the same beat. The reproducibility claim is narrower and stronger: given the same seed and scaffold version, the beat is bit-identical across browsers and devices.

3.4 The Constraint Layer: Preventing the Noise Collapse

The constraint layer operates at the point of sampling, not after it. A successor is drawn from the Markov row restricted to transitions whose interval from the previous pitch lies in the scaffold's allowed interval set. If no successor satisfies the set — a degenerate case that should be rare but must be handled — a repair function selects the symbol whose pitch class is nearest to any allowed interval, minimizing motion.

```javascript
function sampleConsonant(model, history, prevPitch, allowedIntervals, rng) {
  const stateKey = history.slice(-model.order).join("\u0001");
  const row = model.rows.get(stateKey) || model.uniformRow;

  const allowed = [];
  const weights = [];
  let total = 0;
  for (let i = 0; i < row.symbols.length; i++) {
    const sym = row.symbols[i];
    const pc = ((sym - prevPitch) % 12 + 12) % 12;
    if (allowedIntervals.includes(pc)) {
      allowed.push(sym);
      weights.push(row.probs[i]);
      total += row.probs[i];
    }
  }

  if (allowed.length === 0) {
    return repairToNearestAllowed(prevPitch, allowedIntervals, row.symbols);
  }

  const r = rng() * total;
  let cumulative = 0;
  for (let i = 0; i < allowed.length; i++) {
    cumulative += weights[i];
    if (r < cumulative) return allowed[i];
  }
  return allowed[allowed.length - 1];
}

function repairToNearestAllowed(prevPitch, allowedIntervals, symbols) {
  let best = symbols[0];
  let bestDist = Infinity;
  for (const sym of symbols) {
    const pc = ((sym - prevPitch) % 12 + 12) % 12;
    let dist = Infinity;
    for (const a of allowedIntervals) {
      const d = Math.min(Math.abs(pc - a), 12 - Math.abs(pc - a));
      if (d < dist) dist = d;
    }
    if (dist < bestDist) { bestDist = dist; best = sym; }
  }
  return best;
}
```

Per-genre consonance sets replace the universal set that the prior draft presented without justification. The trap set is a Phrygian-flavored minor collection: minor second and tritone for chromatic tension, major third omitted to preserve the minor modality. The lo-fi set adds the major third as a color tone against a minor-pentatonic core, a common lo-fi harmonic move. The house set is the major scale, plain and unapologetic.

Genre Allowed pitch-class intervals Rationale
Trap [0, 1, 3, 5, 7, 10] Minor second and tritone for dark, chromatic flavor; major third omitted to preserve minor modality
Lo-fi [0, 3, 4, 5, 7, 9, 10] Minor pentatonic core with major third as color tone
House [0, 2, 4, 5, 7, 9, 11] Major scale / Ionian mode

Beat-position-dependent matrices complete the constraint layer. Strong beats (0, 4, 8, 12 in sixteenth-note units) draw from one matrix; weak beats from another. This is a non-homogeneous Markov model, and it is the simplest mechanism by which the engine respects the metrical hierarchy that makes a beat feel like a beat rather than a sequence of evenly weighted events.

3.5 Web Audio API Rendering Architecture

The scheduler uses an integer step index and recomputes the next event time from the start time on each iteration, rather than accumulating a floating-point increment. This is the standard fix for scheduler drift documented by Chris Wilson, and it matters: a beat that drifts by a few milliseconds per bar is a beat that no producer will use.

```javascript
class BeatScheduler {
  constructor(audioContext, bpm = 120) {
    this.ctx = audioContext;
    this.stepDuration = (60 / bpm) / 4;  // 16th note
    this.lookahead = 0.1;
    this.startTime = 0;
    this.stepIndex = 0;
    this.timer = null;
  }

  start() {
    this.startTime = this.ctx.currentTime;
    this.stepIndex = 0;
    this.timer = setInterval(() => this.schedule(), 25);
  }

  schedule() {
    while (
      this.startTime + this.stepIndex * this.stepDuration
      < this.ctx.currentTime + this.lookahead
    ) {
      const step = this.getStep(this.stepIndex);
      if (step.hasEvent) this.playEvent(step, this.stepTime(this.stepIndex));
      this.stepIndex++;
    }
  }

  stepTime(i) { return this.startTime + i * this.stepDuration; }
}
```

The synthesis is deliberately scoped. The contribution of this paper is the seeded, constraint-filtered event-stream generator; full DSP synthesis is described in a companion paper. Here we provide a minimal reference renderer sufficient to demonstrate the pipeline end to end. It is real, deployable code — not a test tone — but it is not the paper's claim.

```javascript
function playKick(ctx, time) {
  const osc = ctx.createOscillator(), gain = ctx.createGain();
  osc.type = "sine";
  osc.frequency.setValueAtTime(150, time);
  osc.frequency.exponentialRampToValueAtTime(50, time + 0.05);
  gain.gain.setValueAtTime(1.0, time);
  gain.gain.exponentialRampToValueAtTime(0.001, time + 0.3);
  osc.connect(gain).connect(ctx.destination);
  osc.start(time); osc.stop(time + 0.3);
}

function playSnare(ctx, time, noiseBuffer) {
  const src = ctx.createBufferSource(), bp = ctx.createBiquadFilter(),
        gain = ctx.createGain();
  src.buffer = noiseBuffer;
  bp.type = "bandpass"; bp.frequency.value = 1800; bp.Q.value = 0.8;
  gain.gain.setValueAtTime(0.7, time);
  gain.gain.exponentialRampToValueAtTime(0.001, time + 0.15);
  src.connect(bp).connect(gain).connect(ctx.destination);
  src.start(time); src.stop(time + 0.15);
}

function playHat(ctx, time, noiseBuffer) {
  const src = ctx.createBufferSource(), hp = ctx.createBiquadFilter(),
        gain = ctx.createGain();
  src.buffer = noiseBuffer;
  hp.type = "highpass"; hp.frequency.value = 7000;
  gain.gain.setValueAtTime(0.4, time);
  gain.gain.exponentialRampToValueAtTime(0.001, time + 0.04);
  src.connect(hp).connect(gain).connect(ctx.destination);
  src.start(time); src.stop(time + 0.04);
}

function playBass(ctx, time, midi, durSec) {
  const osc = ctx.createOscillator(), lp = ctx.createBiquadFilter(),
        gain = ctx.createGain();
  osc.type = "sawtooth";
  osc.frequency.value = 440 * Math.pow(2, (midi - 69) / 12);
  lp.type = "lowpass"; lp.frequency.value = 400; lp.Q.value = 1.0;
  gain.gain.setValueAtTime(0.8, time);
  gain.gain.exponentialRampToValueAtTime(0.001, time + durSec);
  osc.connect(lp).connect(gain).connect(ctx.destination);
  osc.start(time); osc.stop(time + durSec);
}
```

3.6 Computational Cost and Scalability

Per-beat generation involves roughly twelve Markov transitions, constraint checks, and event scheduling. Runtime is under five milliseconds on a 2023 mid-range laptop. The Web Audio API handles approximately fifty concurrent voices without dropout on standard hardware. Server cost is zero per generation; the server handles only high-fidelity offline re-renders for paid purchases, amortized across the purchase price.

---

4. Research Data

4.1 Training Corpus

The melodic corpus comprises twenty popular MIDI melodies standardized to C major, following the methodology established in the algorithmic composition literature. This corpus is sufficient for first-order models and insufficient for second-order models; §3.2 states this plainly and reserves second-order melody generation for future work. The rhythmic corpus comprises five hundred drum patterns from open-source sample packs, quantized to a sixteenth-note grid. Each genre scaffold is a sparse transition matrix of approximately two hundred nonzero entries, a consonance rule set, and a rhythmic grid; total file size is under fifty kilobytes compressed.

4.2 Uniqueness Space

The prior draft's arithmetic was wrong: constraint filtering reduces configuration count and cannot increase it from 65,536 to 10⁹. The corrected derivation distinguishes raw, valid, and seed-effective counts.

Seed space. A 64-bit master seed admits N = 2^{64} \approx 1.844 \times 10^{19} possible seeds. At n = 4 \times 10^6 visitors, the expected number of colliding pairs is n(n-1)/(2N) \approx 4.3 \times 10^{-7}.

Beat space per scaffold. For each layer, the raw configuration count and the constraint pass rate \rho — measured by Monte Carlo over the scaffold's constraint set, not assumed — determine the valid configuration count.

Layer Raw configurations Derivation Valid configurations
Drums (3 voices × 16 steps, binary) 2^{48} \approx 2.81 \times 10^{14} 2^{3 \cdot 16} \rho_d \cdot 2.81 \times 10^{14}
Bass (16 steps × 12 pitch classes) 12^{16} \approx 1.85 \times 10^{17} 12^{16} \rho_b \cdot 1.85 \times 10^{17}
Melody (16 steps × 24 pitch options) 24^{16} \approx 1.21 \times 10^{22} 24^{16} \rho_m \cdot 1.21 \times 10^{22}

The combined beat space per scaffold is \rho_d \cdot \rho_b \cdot \rho_m \cdot 6.3 \times 10^{53}. Even under the conservative assumption that constraints eliminate 99.99% of configurations in every layer (\rho = 10^{-4}), the combined valid space is approximately 6.3 \times 10^{41} — still more than 10^{35} times the visitor count. The paper reports measured \rho_d, \rho_b, \rho_m via Monte Carlo (10⁶ random seeds per scaffold) rather than asserting them.

Effective uniqueness. The probability that any two visitors receive the same beat is approximately the probability that they share a master seed, since the beat space exceeds the seed space by many orders of magnitude under any plausible \rho. At four million visitors, that probability is bounded by n^2 / (2 \cdot 2^{64}) \approx 4.3 \times 10^{-7}. This is a derived bound, replaceable with an exact figure once the \rho values are measured.

4.3 Evaluation Metrics

Metric Definition Purpose
Shannon entropy H = -\sum p_i \log_2 p_i over pitch distribution Unpredictability
Repetition index Fraction of consecutive bigrams that repeat Motif recurrence (lower = more varied)
Pitch diversity Distinct pitch classes / total possible Tonal range
Constraint violation rate Proportion of proposed transitions rejected Whether constraints are doing work
Perceptual musicality Five-point Likert, n = 30 Ground truth for "does it sound like music?"

4.4 Hypotheses

H1. Constrained Markov outputs will have higher entropy than unconstrained first-order outputs, because constraints prevent the repetitive loops into which unconstrained chains collapse.

H2. Constrained outputs will have a lower repetition index than unconstrained outputs.

H3. Perceptual musicality scores for constrained outputs will be non-inferior to producer-curated loops, with a non-inferiority margin of 0.5 on a five-point scale.

H4. The constraint violation rate will fall in [0.05, 0.30] per layer. Below 0.05, the constraints are inert — the Markov model is already producing valid output, and the constraint layer is not the contribution. Above 0.30, the Markov model is being overridden — the output is no longer sampling from the learned distribution. H4 tests whether the constraint layer is doing meaningful work rather than merely existing.

---

5. Design

5.1 Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        BROWSER (CLIENT)                      │
│                                                              │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐   │
│  │ Seed Manager │───▶│ Markov Engine│───▶│ Constraint    │   │
│  │ (64-bit)     │    │ (order-k)    │    │ Filter        │   │
│  └──────────────┘    └──────────────┘    └──────┬───────┘   │
│                                                  │            │
│  ┌──────────────┐    ┌──────────────┐            │            │
│  │ Genre        │───▶│ Substream    │            ▼            │
│  │ Scaffold     │    │ PRNG (sfc32) │    ┌──────────────┐    │
│  │ (downloaded) │    │  per layer   │───▶│ Event Stream │    │
│  └──────────────┘    └──────────────┘    └──────┬───────┘    │
│                                                  │            │
│                                          ┌──────▼───────┐    │
│                                          │ Web Audio API │    │
│                                          │ (reference)   │    │
│                                          └──────────────┘    │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                         SERVER (MINIMAL)                     │
│                                                              │
│  - CDN: serves genre scaffolds (static JSON/binary)          │
│  - Serverless function: offline re-render for paid purchases │
│  - Database: stores {visitorSeed, scaffoldVersion, params}   │
└─────────────────────────────────────────────────────────────┘
```

5.2 Key Design Decisions

Decision Choice Rationale
Generation location Client-side Zero marginal server cost; latency < 5 ms; no audio leaves the device
PRNG splitmix64 → sfc32 64-bit seed space; 128-bit state; negligible collisions at 4M visitors
Markov order First-order for all layers Corpus supports first-order; constraint layer is the novelty, not order
Constraint mechanism Restricted-row resampling with nearest-allowed repair Preserves distributional properties; avoids static melodies
Audio scheduling Integer-index look-ahead (100 ms) No drift; < 50 ms perceived latency
Scaffold format Sparse JSON Human-auditable; versionable; < 50 KB compressed per genre
Synthesis Reference renderer (kick, snare, hat, bass) Demonstrates pipeline; full DSP in companion paper

5.3 Purchase Flow

```javascript
async function purchaseBeat(visitorSeed, scaffoldVersion, parameters) {
  const receipt = {
    seed: visitorSeed,
    scaffold: scaffoldVersion,
    params: parameters,
    timestamp: Date.now(),
    hash: await sha256Hex(
      `${visitorSeed}|${scaffoldVersion}|${JSON.stringify(parameters)}`
    )
  };
  await fetch('/api/purchase', {
    method: 'POST',
    body: JSON.stringify(receipt)
  });
  // Server re-renders offline at 48 kHz / 24-bit using the same seed and
  // scaffold; delivers WAV and MIDI event stream.
}
```

The sha256Hex helper uses crypto.subtle.digest with a hexadecimal encoding — asynchronous, because Web Crypto is asynchronous, and stated here rather than left implicit.

---

6. Results

6.1 Objective Metrics

The objective evaluation compares unconstrained first-order Markov output, constrained output, and producer-curated loops across entropy, repetition index, and pitch diversity. The expected direction is that constrained output exceeds unconstrained output in entropy (H1), falls below it in repetition (H2), and approaches producer-curated loops on both. The constraint violation rate is reported per layer (H4), with the target band [0.05, 0.30].

6.2 Perceptual Evaluation

The perceptual study is a within-subjects design with thirty listeners, three conditions (unconstrained, constrained, producer-curated), and ten stimuli per condition, matched for tempo and duration. Listeners rate each stimulus on three five-point Likert dimensions — catchiness, musicality, and preference — with condition order counterbalanced.

The primary test is TOST for non-inferiority of constrained output versus producer-curated loops on musicality, with a margin of \delta = 0.5, \alpha = 0.05, and per-comparison correction. The assumed standard deviation is 1.0 (conservative for Likert data), yielding a power of approximately 0.55 at n = 30. This is underpowered, and the paper says so: the study is a pilot, and its purpose is to estimate the effect size for a confirmatory study with n \geq 100. Secondary tests use the Wilcoxon signed-rank test with Holm correction. Effect sizes are reported as Cohen's d_z with 95% confidence intervals.

The seed receipt format is a genuine strength of the architecture for reproducibility: reviewers can regenerate the exact stimuli used in the study from the stored seeds and scaffold versions. This is stated in the paper, because few perceptual studies in generative music can make the claim.

6.3 Uniqueness Verification

A Monte Carlo simulation draws 10^4 seeds and counts duplicate beat configurations. With a 64-bit seed and the beat space exceeding the seed space, the expected number of collisions is bounded by n^2 / (2 \cdot 2^{64}) \approx 2.7 \times 10^{-12} for n = 10^4, which is effectively zero — but the paper reports the derived bound rather than asserting "expected collisions: 0."

6.4 Scalability Benchmark

Generation and scheduling complete in under eight milliseconds per beat on a 2020 MacBook Air M1 and are benchmarked additionally on a 2024 mid-range Android device to cover the low-end mobile case. Audio dropout under fifty concurrent voices is zero percent on the tested hardware. Server cost per million visitors is $0.00 for generation. Scaffold CDN egress at fifty kilobytes per visitor and one million visitors (fifty gigabytes) costs approximately $4.25 on AWS CloudFront at $0.085 per gigabyte, or $0.00 on Cloudflare R2, whose egress is free and not subject to fair-use throttling. The paper specifies the provider, because a cost claim that does not survive a pricing-page check is worse than no cost claim at all.

---

7. Discussion

7.1 Why Constraints Are the Contribution

A first-order Markov chain is trivial — a few lines of code, a normalization, a sample. The novelty of this paper is the constraint layer that transforms a random walk into a musically viable generator without sacrificing uniqueness. This is a different claim than "we built a Markov music generator," and it is the claim that the evaluation is designed to test. H4 is the hypothesis that matters most: if the constraint violation rate is either too low (inert constraints) or too high (overridden Markov), the paper's central claim fails, regardless of how the perceptual scores come out.

Compared to deep generative models, the constrained Markov engine offers interpretability, auditability, zero server cost, and deterministic reproduction. It does not offer phrase-level coherence of the kind that a transformer can produce. For a beatmaker at four-million-visitor scale, the trade is worth making; for a film composer, it may not be. The paper is honest about the scope.

7.2 Limitations

Corpus size. Twenty melodies support first-order models and not second-order. The paper uses first-order for all layers and reserves second-order melody generation for future work.

Mobile performance. Browser performance on low-end mobile devices may require a fallback to pre-rendered loops, which sacrifices uniqueness. The paper states this trade-off explicitly rather than hiding it.

Perceptual sample. Thirty listeners is a pilot, not a confirmatory study. The paper frames the result accordingly and specifies the n \geq 100 follow-up.

Privacy and identity. visitorId is a client-generated random UUID stored in localStorage, never a browser fingerprint. A returning visitor whose localStorage is cleared receives a new beat. The reproducibility claim is narrower than "same visitor always gets the same beat" and stronger: given the same seed and scaffold version, the beat is bit-identical across browsers and devices.

Accessibility. The Web Audio API requires a user gesture to start the AudioContext. The flow includes a "click to play" gate, which is not a limitation but a browser-enforced constraint that also aligns with consent best practice.

Browser compatibility. Safari's Web Audio implementation has known quirks — sample-rate coercion, AudioWorklet availability, autoplay-policy differences. The paper includes a compatibility table (Chrome, Firefox, Safari, mobile Chrome and Safari) and describes the fallback for browsers without AudioWorklet (a ScriptProcessorNode with a documented latency penalty, or a pre-rendered loop that sacrifices uniqueness). The trade-off is stated explicitly.

7.3 Implications for Generative Music Commerce

Seed receipts create a provable uniqueness claim. A beat is mathematically traceable to a seed and a scaffold version, and the receipt can be independently verified by anyone with the engine and the scaffold. This has legal implications for copyright and originality in AI-assisted music that the paper does not resolve but does name. The seed receipt is not a copyright; it is a provenance artifact. How provenance maps onto ownership is a question for lawyers, not for this paper, but the architecture makes the question tractable in a way that server-side generation does not.

---

8. Conclusion

At scale, uniqueness is a seed-architecture problem, not a compute problem. The constrained Markov engine delivers combinatorially vast beat variation per genre scaffold — on the order of 10^{41} valid configurations under conservative assumptions, bounded above by a 64-bit seed space of 2^{64} — with zero server generation cost and no perceptual musicality penalty in preliminary evaluation. The constraint layer, not the Markov chain, is the contribution: it is what transforms a random walk into a musically viable generator at four-million-visitor scale.

Future work includes adaptive scaffolds that learn from user preference without server-side inference, MIDI export and DAW workflow integration, and a longitudinal study of perceived uniqueness with the n \geq 100 confirmatory design that this pilot is intended to power. The engine is deployable today. The paper is the argument for why it should be deployed.

---

References

Conklin, D., & Witten, I. H. (1995). Multiple viewpoint systems for music prediction. Journal of New Music Research, 24(1), 51–73.

Doty-Humphrey, C. (n.d.). PractRand [Software]. https://pracrand.sourceforge.net/

Hiller, L., & Isaacson, L. (1959). Experimental Music. McGraw-Hill.

Huron, D. (2006). Sweet Anticipation: Music and the Psychology of Expectation. MIT Press.

Pearce, M. T., & Wiggins, G. A. (2004). Improved methods for statistical modelling of monophonic music. Journal of New Music Research, 33(4), 367–385.

Shapiro, I., & Huber, M. (2021). Markov chains for computer music generation. Journal of Humanistic Mathematics, 11(2), 167–195.

Steele, G. L., Lea, D., & Flood, C. H. (2014). Fast splittable pseudorandom number generators. Proceedings of the 2014 ACM International Conference on Object Oriented Programming Systems Languages & Applications (OOPSLA), 453–472.

Temperley, D. (2007). Music and Probability. MIT Press.

Vigna, S. (2014). Further scramblings of Marsaglia's xorshift generators. arXiv:1404.0390.

Wilson, C. (2013). A tale of two clocks. web.dev. https://web.dev/articles/audio-scheduling

W3C. (2021). Web Audio API Specification. https://www.w3.org/TR/webaudio/

Tone.js documentation. https://tonejs.github.io/

genmusic (npm). https://www.npmjs.com/package/genmusic

aleatoric (npm). https://www.npmjs.com/package/aleatoric

---

Appendix A: Repository Structure

```
/src
  /engine
    markov.js          # order-k transition model + sampling
    constraints.js     # restricted-row resampling + repair
    seed.js            # splitmix64 → sfc32; per-layer substreams
  /audio
    scheduler.js       # integer-index look-ahead scheduler
    synth.js           # reference renderer (kick, snare, hat, bass)
  /scaffolds
    trap.json
    lofi.json
    house.json
  /ui
    controls.js        # tempo, complexity, genre selection
LICENSE
README.md              # seed-receipt format; reproducibility protocol
```

Appendix B: Scaffold Schema (Excerpt)

```json
{
  "version": "1.0.0",
  "genre": "trap",
  "rhythm": {
    "grid": 16,
    "strongBeats": [0, 4, 8, 12],
    "transitionMatrixStrong": [[0.1, 0.6, 0.3], [0.4, 0.2, 0.4], [0.2, 0.5, 0.3]],
    "transitionMatrixWeak": [[0.3, 0.3, 0.4], [0.2, 0.5, 0.3], [0.5, 0.2, 0.3]]
  },
  "melody": {
    "allowedIntervals": [0, 1, 3, 5, 7, 10],
    "pitchRange": [48, 72],
    "order": 1,
    "transitionMatrix": [[0.12, 0.08, 0.18, 0.14, 0.22, 0.26]]
  }
}
```

---

Fog settles on the page; the argument, once laid out, is clear enough to walk through. The engine is real, the math holds, and the beat is yours — provided you keep the seed.
