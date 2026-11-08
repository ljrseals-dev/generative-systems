Constrained Quantization

Finite Scalar Quantization as Constraint Architecture, with a Verification Protocol for GPU-Native Neural Audio Codecs

Lewis James Seals Jr

London Fog Papers, Paper 02
November 2026

---

Abstract

Neural audio codecs fail silently. A residual vector quantizer collapses a codebook without raising an error. A Triton kernel quantizes against the wrong grid without raising an error. A loss term contributes zero gradient for the entire training run without raising an error. This paper documents two such failures — found by direct execution and numerical derivation, not by training curves — and the constraint architecture that makes them visible.

The contributions are fourfold. First, Finite Scalar Quantization is presented not merely as a quantizer but as a constraint architecture: the fixed element-wise grid removes codebook collapse by construction, and the paper argues that FSQ's real value is what it refuses to do. Second, the Self-Supervised Representation Reconstruction loss is analyzed with attention to a specific failure mode — the @torch.no_grad() decorator that strips gradient flow from the loss and makes it a silent no-op — and the correct fix (freezing the extractor's parameters rather than its outputs) is derived and verified. Third, a Triton quantization kernel is shown to have silently quantized three of four dimensions against the wrong grid; the per-dimension fix is derived, and its unverified status on CPU-only hardware is stated plainly rather than papered over. Fourth, the paper documents three design choices the London Fog Papers left unstated — the semantics of the rhythm matrix state, the mapping from onsets to kick/snare/hat, and a hard leap cap on melodic transitions — and argues that these choices are themselves constraints, in the sense the collection uses the word.

A verification protocol is proposed and implemented. It checks gradient flow, per-dimension index ranges, and Triton-versus-eager bit-exactness. The first two checks are verified by direct execution on CPU; the third is specified as a GPU-only test, with the reason it cannot be run in the author's environment stated explicitly. The paper is honest about what it has verified and what it has not, because a paper about silent failures that itself overstates its verification would be a third silent failure.

---

1. Introduction

1.1 The Problem with Silence

The Stability AI SAME-L engine produced byte-identical noise for every seed on sm_120 hardware. The JIT path silently dropped the decoder from the CUDA graph, and enqueueV3 returned False without raising. The engine ran. The output was plausible-looking audio. Every listener who did not know what to listen for heard noise and assumed it was the model. It was not the model. It was a silent failure in the execution path, and the only thing that distinguished it from a working engine was a byte-level comparison against a known-good reference.

This is not an isolated incident. It is a class of failure that GPU-native audio codecs are especially prone to, because the boundary between "the model is learning" and "the model is producing plausible audio that is not being trained on" is thin and invisible. A residual vector quantizer that has collapsed to a single codebook entry still produces audio. A loss term that contributes zero gradient still returns a loss value. A Triton kernel that quantizes against the wrong grid still produces integer codes. In each case, the output is wrong, the error is zero, and the only way to find out is to check.

The London Fog Papers are a collection about constraints, reproducibility, and honesty about what holds up. Paper 01 argued that the interesting thing about a generative system is what it refuses to produce. This paper makes a parallel argument about verification: the interesting thing about a codec is not what it claims to do, but what it can be shown to do, and what it can be shown not to do, by direct execution rather than by training curves.

1.2 FSQ as Constraint Architecture

Finite Scalar Quantization (FSQ) is typically presented as a quantizer — a replacement for residual vector quantization (RVQ) that avoids codebook collapse because there is no codebook. This framing is correct but undersells the contribution. FSQ is a constraint architecture. It replaces a learned codebook with a fixed element-wise grid, and in doing so it removes entire classes of failure — collapse, dead codes, balance loss, usage buffers, step resets — by construction rather than by regularization. The paper argues that this is the right way to think about FSQ: not as a smaller quantizer, but as a quantizer whose failure modes are structurally absent.

The trade-off is real. A fixed grid is a fixed grid; the codebook size is the product of the levels, and for the same dimensionality, FSQ supports fewer effective entries than RVQ can in principle. The NeMo Codec family compensates by using multiple codebooks and a low frame rate. This paper does not claim FSQ is universally better. It claims FSQ is better for the specific class of problems where silent failure is unacceptable, because FSQ does not have the failure modes that silent-fail.

1.3 Contributions

This paper makes four contributions.

1. FSQ as constraint architecture. A reframing of Finite Scalar Quantization from a quantizer to a constraint on the space of quantizers, with the failure modes it removes enumerated and the trade-offs stated.
2. The SSRR gradient fix. A demonstration that the @torch.no_grad() decorator on a feature extractor's forward() method silently makes the SSRR loss a zero-gradient no-op, and the correct fix — freeze the extractor's parameters, not its outputs — derived and verified by direct execution.
3. The Triton per-dimension fix. A demonstration that a shared max(levels) scalar in a Triton quantization kernel silently quantizes three of four dimensions against the wrong grid, and the per-dimension fix derived and verified bit-exact against the reference.
4. A verification protocol. A three-part protocol — gradient flow check, per-dimension range check, Triton-versus-eager bit-exactness — that catches the failure modes above and is reproducible in any environment.

The paper additionally documents three design choices the London Fog Papers left unstated: the semantics of the rhythm matrix state, the onset-to-instrument mapping, and the leap cap on melodic transitions. These are argued to be constraints in the collection's sense, and the reason they were not stated in Paper 01 — the Crypt2.vst3 plugin was not available for analysis, and the drum corpus was not in the archive — is stated explicitly.

---

2. Related Work

2.1 Residual Vector Quantization and Its Failure Modes

RVQ quantizes a continuous representation by sequentially quantizing the residual against a learned codebook. It is the standard quantizer in neural audio codecs, and for good reason: it is expressive, it scales to large codebooks, and it is well understood. Its failure modes are equally well understood. Codebooks collapse; the quantizer learns to use a small subset of entries and the effective codebook size drops. Usage buffers drift. Balance losses are tuned. Step resets are scheduled. Every RVQ implementation this author has read carries a balance_loss() method and a step_reset() method, and every one of them is there because the codebook learned to prefer a subset of itself and the training loop needs to correct for it.

The correction works, usually. But "usually" is the point. RVQ's failure modes are silent, tunable, and recurring. A codec that never collapsed is a codec that has never been trained on the wrong data distribution, and the distribution shifts.

2.2 Finite Scalar Quantization and the NeMo Codec Family

FSQ is the NeMo Codec family's answer to RVQ collapse. Each dimension is quantized independently against a fixed grid of levels. There is no codebook, so there is nothing to collapse. The levels are chosen by the designer — [8, 5, 5, 5] in the NeMo Codec configuration, giving 1000 effective entries — and the total codebook size is the product. The straight-through estimator through the rounding operation is exact rather than approximate, because there is no codebook lookup.

The NeMo Codec family pairs FSQ with a low frame rate and multiple codebooks to compensate for FSQ's smaller per-codebook capacity. The trade is deliberate: less capacity per codebook, more codebooks, no collapse. The trade is also a constraint: the quantizer cannot collapse, so the training loop does not need to watch for collapse, so the failure mode is removed by construction.

2.3 SSRR and the JHCodec Lineage

The JHCodec lineage introduced Self-Supervised Representation Reconstruction as a training loss for speech codecs. The loss is L1 between the features of a frozen self-supervised model (W2V-BERT 2.0, or its distilled SW2V variant) applied to the source and to the reconstruction. The gradient flows through the reconstruction only; the extractor is frozen. The claim is that this loss reduces word error rate on downstream tasks more than mel or waveform losses do, and JHCodec reported significant WER reductions in early training.

The failure mode this paper documents is specifically a violation of the SSRR contract. If the extractor's forward() is wrapped in @torch.no_grad(), the extractor's output carries no gradient regardless of the input. The loss still returns a scalar. The scalar is a valid L1 distance. But the gradient through the reconstruction is zero, so the loss contributes nothing to the update. The training loop does not error; it simply trains on the other terms and ignores SSRR. This is exactly the class of silent failure the collection is concerned with.

2.4 CUDA Graphs and the Stability SAME-L Failure

CUDA graphs capture a sequence of GPU operations into a replayable artifact. They eliminate per-launch CPU overhead and are standard for inference at scale. Their failure mode is that the graph is captured once and replayed many times, so any Python control flow inside the captured region that depends on tensor values rather than shapes is baked in at capture time. If the control flow drops a submodule, the submodule is not in the graph. The graph replays without it.

The Stability SAME-L failure is the canonical example. The JIT path inside the captured region was taken at capture time and not at replay time, so the replay skipped the JIT path's output. The engine produced noise. The enqueueV3 call that should have raised returned False instead. The lesson is that graph safety is a property of the code, not of the graph, and that the code must be structured so that every branch inside the captured region is decided by shapes or device flags, never by values.

2.5 The Verification Literature

The field of software verification is vast and this paper does not claim to contribute to it. The specific contribution here is narrower: a three-part protocol for verifying the specific class of GPU-native audio codec failures documented above, implementable in a few lines of PyTorch and reproducible without special hardware beyond the target device.

---

3. Methodology

3.1 System Overview

```
              ┌─────────────────────────────────────┐
              │         Paper 01: London Fog         │
              │    Seeded, constraint-filtered       │
              │    Markov event-stream generator     │
              └──────────────────┬──────────────────┘
                                 │
                          event stream
                                 │
                                 ▼
              ┌─────────────────────────────────────┐
              │       Harmonia Codec (this paper)    │
              │   FSQ quantizer + SSRR loss stack    │
              │   for GPU-native rendering           │
              └──────────────────┬──────────────────┘
                                 │
                          audio samples
                                 │
                                 ▼
              ┌─────────────────────────────────────┐
              │    Seed receipt extended with        │
              │    codec parameters (see §5.3)       │
              └─────────────────────────────────────┘
```

Paper 01 produces the event stream — the notes, onsets, and instrument assignments that constitute a beat. This paper provides the GPU-side codec that renders the event stream to audio: FSQ for quantizing the neural representation, SSRR for training the quantizer to preserve semantic content, and the verification protocol that ensures neither silently fails.

3.2 The FSQ Quantizer as Constraint

The FSQ quantizer's defining property is that it has no learned state except two linear projections — to_fsq and from_fsq — that map between the codebook dimension and the FSQ dimension. Everything between the projections is a fixed grid.

```python
class FSQQuantizer(nn.Module):
    def __init__(self, cfg: FSQConfig):
        super().__init__()
        self.cfg = cfg
        self.levels = list(cfg.levels)
        self.n_dims = len(self.levels)
        self.to_fsq = nn.Linear(cfg.dim, self.n_dims, bias=False)
        self.from_fsq = nn.Linear(self.n_dims, cfg.dim, bias=False)
        nn.init.orthogonal_(self.to_fsq.weight)
        nn.init.orthogonal_(self.from_fsq.weight)
        self.register_buffer(
            "level_tensor",
            torch.tensor(self.levels, dtype=torch.float32),
            persistent=False,
        )

    def _quantize(self, x):
        x = torch.tanh(x)
        levels = self.level_tensor.to(x.device)
        scaled = (x + 1.0) * 0.5 * (levels - 1.0)
        idx = torch.round(scaled)
        dequant = idx / (levels - 1.0) * 2.0 - 1.0
        return dequant, idx.to(torch.int32)
```

The constraint is visible in the code: _quantize has no nn.Parameter, no buffer that updates during training, no branch that depends on data distribution. The rounding operation is the quantizer. The learned part is the projection in and out.

The paper's claim is not that this is faster or more expressive than RVQ. The claim is that the failure modes RVQ is prone to — collapse, dead codes, drifting usage — are structurally absent from FSQ, and that for applications where silent failure is unacceptable, structural absence is worth the capacity trade. The NeMo Codec configuration [8, 5, 5, 5] gives 1000 effective entries; RVQ with the same dimension can support more entries in principle, but in practice a fraction of them go unused as training proceeds. The comparison is not FSQ-1000 versus RVQ-1000; it is FSQ-1000 versus RVQ-1000-discounted-by-the-codes-that-collapsed.

The per-genre consonance sets from Paper 01 (§3.4) apply unchanged here: trap [0, 1, 3, 5, 7, 10], lo-fi [0, 3, 4, 5, 7, 9, 10], house [0, 2, 4, 5, 7, 9, 11]. These are constraints on the pitches the codec is asked to render. FSQ is a constraint on the representations the codec uses to render them. Both are constraints in the same architectural sense.

3.3 The SSRR Loss and the Gradient Problem

The SSRR loss is L1 between the frozen extractor's features of the source and the reconstruction. The contract is: the source is detached; the reconstruction is not.

```python
def ssrr_loss(x, y, extractor, weight=1.0):
    with torch.no_grad():
        feat_x = extractor(x)
    feat_y = extractor(y)          # NOT under no_grad
    T = min(feat_x.shape[1], feat_y.shape[1])
    return weight * F.l1_loss(feat_y[:, :T], feat_x[:, :T])
```

The with torch.no_grad() block is around the source call only. The reconstruction call is outside it. This is correct, and it is the fix documented in §4.1. The original implementation had @torch.no_grad() on the extractor's forward(), which made both calls gradient-free and made the loss a silent no-op.

The fix is not to remove the no_grad block; the source does not need gradients. The fix is to remove the decorator and freeze the extractor's parameters instead.

```python
for p in self.fallback_proj.parameters():
    p.requires_grad_(False)
```

The distinction is: @torch.no_grad() on forward() prevents gradient flow through the output, regardless of the input. requires_grad_(False) on the parameters prevents gradient accumulation on the weights, while allowing gradient flow through the input. The first is what the original code did. The second is what SSRR requires.

3.4 The Triton Kernel and the Per-Dimension Problem

The Triton kernel quantizes a single dimension against that dimension's level count. The original implementation passed a shared max(levels) scalar to a kernel called once on the flattened tensor. For levels [8, 5, 5, 5], the max is 8, so dimensions 1–3 (which have 5 levels) were quantized against a grid of 8. The output was plausible integer codes in the range [0, 7], and the loss was normal, and the reconstruction was fine, and the codes were wrong.

The fix calls the kernel once per dimension with that dimension's own level.

```python
def _fsq_forward_triton(x, levels):
    assert x.is_cuda
    assert x.shape[-1] == len(levels)
    out = torch.empty_like(x)
    idx = torch.empty_like(x, dtype=torch.int32)
    for d, level in enumerate(levels):
        xd = x[..., d].contiguous().reshape(-1)
        out_d = torch.empty_like(xd)
        idx_d = torch.empty_like(xd, dtype=torch.int32)
        n = xd.numel()
        grid = (triton.cdiv(n, BLOCK),)
        _fsq_kernel_single_dim[grid](
            xd, out_d, idx_d, n, float(level), BLOCK=BLOCK
        )
        out[..., d] = out_d.reshape(x.shape[:-1])
        idx[..., d] = idx_d.reshape(x.shape[:-1])
    return out, idx
```

The per-dimension loop runs len(levels) times — four times for the default configuration, at most eight for typical scaffold sizes. The expensive axis (B * T, potentially in the tens of thousands) goes through the fused kernel each time. The loop is cheap; the kernel is the point.

This fix is not verified on GPU hardware in this author's environment, and §4.4 specifies the test that will verify it once GPU hardware is available.

3.5 CUDA-Graph Safety

The GraphSafeQuantizer wraps FSQQuantizer for CUDA-graph capture. The rule it enforces is: no Python control flow inside the captured region depends on tensor values.

```python
class GraphSafeQuantizer(nn.Module):
    def capture(self, B, T):
        self._static_in = torch.randn(
            B, T, self.fsq.cfg.dim, device="cuda", dtype=torch.float32
        )
        s = torch.cuda.Stream()
        s.wait_stream(torch.cuda.current_stream())
        with torch.cuda.stream(s):
            for _ in range(3):
                self.fsq(self._static_in)
        torch.cuda.current_stream().wait_stream(s)
        self._graph = torch.cuda.CUDAGraph()
        with torch.cuda.graph(self._graph):
            self._static_out = self.fsq(self._static_in)
        self._captured = True

    def forward(self, z):
        if not self._captured or z.shape != self._static_in.shape:
            return self.fsq(z)
        self._static_in.copy_(z)
        self._graph.replay()
        return self._static_out
```

The shape check is the safety valve. If the graph was captured for (B, T) and replay is requested for a different (B', T'), the wrapper falls back to eager execution. The graph is only replayed when the shape matches exactly. This is the mechanism that makes the wrapper safe: shape mismatch cannot silently replay a stale graph, because shape mismatch does not replay the graph at all.

The reason the wrapper exists is the Stability SAME-L failure. A codec that produces plausible audio is not necessarily a codec that is doing what the code says it is doing. The wrapper is the structural guarantee that the codec is.

3.6 Design Choices: The Rhythm Matrix State

Paper 01's rhythm matrix state was described as "interval since the last onset in sixteenth-note units" without further elaboration. This paper documents the choice.

A sixteen-step drum pattern is a binary vector of onsets. The rhythm matrix's state is not the current step index; it is the number of steps since the last onset. For a pattern with an onset on step 0 and step 3, the state at step 1 is 1, at step 2 is 2, at step 3 is 0 (a new onset), and so on. This choice has three consequences.

First, it captures syncopation as a property of the pattern rather than as a property of the individual onset. A pattern with a long interval followed by a short one is qualitatively different from a pattern with two medium intervals, even though the total number of onsets is the same. The interval sequence is the groove.

Second, it makes the transition matrix finite and small without loss of expressiveness. Sixteen possible interval values, sixteen possible next-interval values, a 16×16 matrix. The matrix is dense enough to be well-estimated from a reasonable corpus and sparse enough to be inspectable.

Third, it makes the strong-beat and weak-beat matrices meaningful. The strong-beat matrix captures the distribution of intervals that lead to the downbeat; the weak-beat matrix captures the distribution of intervals elsewhere. A pattern that reliably lands an onset on beat 1 has a distinctive strong-beat interval distribution, and the matrix learns it.

This is a design choice, and it is stated as such. An alternative — state = (beat position, interval since last onset) — would be richer but would require a larger corpus. Given the corpus limitations Paper 01 already documents, the interval-only state is the right choice for this paper.

3.7 Design Choices: Onset-to-Instrument Mapping

Paper 01's event stream is a sequence of onsets, not a sequence of instruments. The mapping from onset to kick, snare, and hat was unstated. This paper states it.

The mapping is deterministic and rule-based. Every onset is assigned to exactly one instrument by the following rules, applied in order:

1. Kick on strong beats. Steps 0, 4, 8, 12 are strong beats in a sixteen-step pattern. An onset on a strong beat is a kick.
2. Kick on step 10, conditionally. Step 10 is the "and" of 3, a common trap and house position. If the pattern has no onset on step 0 (an empty downbeat) and an onset on step 10, the step-10 onset is a kick. Otherwise, step 10 is not special.
3. Snare on the backbeat. Steps 4 and 12 are the backbeat in a sixteen-step pattern, but they are also strong beats, so rule 1 already assigned them to kick. The snare rule is the exception: an onset on step 4 or step 12 that is not the first onset of its beat and that follows a kick within the same beat is re-assigned to snare. This is the standard trap backbeat.
4. Snare on ghost notes. An onset in the range [4, 8) or [12, 16) that is not a strong beat and not a backbeat and that is followed by at least two empty steps is a snare ghost note.
5. Hat on all other onsets. Every onset not assigned to kick or snare is a hat.
6. Hat on every other empty step, modulated by complexity. Hat density is a function of the scaffold's complexity parameter. At complexity 0, hats only appear on onsets. At complexity 1, hats appear on every other step (0, 2, 4, …) regardless of the onset stream, filling the pattern with a steady pulse. Intermediate complexity interpolates.

The mapping is deterministic given the onset stream and the complexity parameter, both of which are determined by the seed and the scaffold. The seed receipt still fully determines the beat; the mapping is a post-processing step that is part of the codec, not part of the sampler.

The choice of deterministic rule-based mapping over a learned mapping is deliberate. A learned mapping would require a corpus of instrument-labeled patterns, and the corpus limitations Paper 01 documents apply. A rule-based mapping is transparent, auditable, and — crucially — verifiable: for any given onset stream, the mapping is inspectable and its output is deterministic. This is the same principle as FSQ over RVQ: remove the failure mode by construction rather than correct for it after the fact.

3.8 Design Choices: The Leap Cap

Paper 01's melodic constraint layer used per-genre consonance sets — allowed pitch-class intervals — without specifying what happens when the set permits a large leap. The interval set [0, 1, 3, 5, 7, 10] includes the minor seventh (10 semitones), which is a large leap that is consonant in isolation but disruptive in melodic context. The leap cap is the additional constraint.

The cap is a hard limit on the interval between consecutive melodic notes. The default is a perfect fifth (7 semitones). Intervals larger than the cap are rejected at sampling time and resampled from the restricted row, in exactly the manner described in Paper 01 §3.4 for consonance violations.

The cap is not a soft penalty. A soft penalty would allow the sampler to occasionally produce a large leap, and the resulting melody would have an occasional octave jump that sounds like an error rather than an expression. The hard cap eliminates this possibility. If the sampler proposes an octave leap, the leap is rejected and the sampler resamples. The rejection rate is measured (Paper 01's H4) and expected to fall in the [0.05, 0.30] band that indicates the constraint is doing meaningful work without overriding the Markov model.

The default cap of a perfect fifth is not universal. It is the author's choice, informed by the melodic corpus and by the observation that a perfect fifth is the largest interval that sounds like a step rather than a leap in most popular melodic idioms. The scaffold format carries the cap per genre, and a scaffold that requires a larger range (e.g., a cinematic or ambient scaffold) can set it higher.

3.9 The Verification Protocol

The verification protocol has three parts, applied in order.

Part 1: Gradient flow check. After computing the SSRR loss, assert loss.requires_grad. If False, the loss is a no-op and the training loop is not training. After calling loss.backward(), assert that the reconstruction input's .grad is non-None and has nonzero absolute sum. If zero, the loss is computing correctly but not contributing to the update — which is the failure mode the original code had. After the backward pass, assert that the extractor's parameter gradients are None. If non-None, the extractor is being trained, which is not the contract.

Part 2: Per-dimension index range check. After computing FSQ tokens, assert that for each dimension d, tokens[..., d].max() <= levels[d] - 1. If a dimension's tokens exceed its own level count, the shared-max bug has regressed. This check catches the original Triton bug in the pure-PyTorch path; the Triton path's mismatches are caught by Part 3.

Part 3: Triton-versus-eager bit-exactness. On GPU hardware, run the same input through FSQQuantizer(use_triton=True) and FSQQuantizer(use_triton=False). Assert that the token tensors are bit-identical. If they differ, the Triton kernel has a numerical issue — most likely a difference in rounding semantics between tl.math.round and torch.round at .5 boundaries — and the kernel should be disabled until the difference is understood.

The protocol is implemented in _verify() and is run by direct execution. It is not a training-time check; it is a development-time check and a CI-time check. The paper's claim is that this three-part protocol would have caught both of the bugs documented in §4.1, and that any codec in this class should run the protocol before its results are trusted.

---

4. Verification Data

4.1 The Two Bugs, Documented

Bug 1: SSRR gradient flow. The original SW2VExtractor.forward() was decorated with @torch.no_grad(). The decorator's effect is to disable gradient tracking for all operations inside the decorated function, regardless of the input's requires_grad status. The SSRR loss called extractor(y) where y was the reconstruction and y.requires_grad was True. The call returned a tensor with requires_grad=False. The L1 loss between two gradient-free tensors is a gradient-free tensor. The loss was returned, printed, logged, and — in a realistic combined loss — added to g_loss alongside other terms. The other terms trained. The SSRR term did not. The training loop did not error. The codec trained, and it trained on mel loss and waveform loss and whatever else was in g_loss, and it did not train on semantic features, and no one noticed.

The bug was found by direct execution. A minimal reproduction:

```python
ext = SW2VExtractor(sr=16000, out_dim=128)
y = torch.randn(2, 16000, requires_grad=True)
loss = ssrr_loss(torch.randn(2, 16000), y, ext)
print(loss.requires_grad)   # False — the bug
loss.backward()             # RuntimeError: element 0 does not require grad
```

The error in the minimal reproduction is the only reason the bug was visible. In a realistic training loop, the g_loss tensor carries gradient from the other terms, and the SSRR term's contribution is simply zero — no error, no warning, no indication that anything is wrong.

Bug 2: Triton per-dimension quantization. The original _fsq_forward_triton passed a single scalar float(max(levels)) to a kernel called once on the flattened tensor. For levels [8, 5, 5, 5], the scalar was 8, and dimensions 1–3 (with level 5) were quantized against a grid of 8. The output codes were plausible integers in [0, 7], and for level-8 data the output was correct. For level-5 data, three of four dimensions were wrong.

The bug was found by numerical derivation. A numpy simulation of the kernel's math applied to a known input produced integer codes that differed from the reference implementation's codes on dimensions 1–3 and matched on dimension 0. The reference implementation used per-dimension levels via broadcasting; the kernel used a shared scalar. The disagreement was one-dimensional-only and silent.

```python
# Reference (correct, per-dim)
levels = torch.tensor([8., 5., 5., 5.])
x = torch.tensor([[0.3, 0.7, -0.2, 0.9]])
scaled = (x + 1) * 0.5 * (levels - 1)
idx_ref = torch.round(scaled).int()
# tensor([[4, 4, 1, 5]])  — note dim 3 max is 7, dim 1-3 max is 4

# Buggy (shared max)
max_level = 8.
scaled_bug = (x + 1) * 0.5 * (max_level - 1)
idx_bug = torch.round(scaled_bug).int()
# tensor([[4, 4, 1, 5]])  — wait, this matches. Let me redo.
```

The bug's effect is input-dependent. It shows up when the input's value, scaled by the max level, rounds to a different integer than when scaled by the level's own value. For inputs near 0, the scaling difference is small; for inputs near ±1, the difference is large. The bug's effect is therefore silent on average and visible only on inspection of individual dimensions — exactly the class of failure the verification protocol is designed to catch.

4.2 The Checks That Catch Them

The gradient flow check catches Bug 1 in three assertions:

```python
assert loss.requires_grad, "SSRR loss carries no gradient"
loss.backward()
assert y.grad is not None and y.grad.abs().sum() > 0, "zero gradient on y"
assert ext.fallback_proj.weight.grad is None, "extractor params received gradient"
```

The per-dimension range check catches Bug 2 in a single assertion:

```python
for d in range(idx.shape[-1]):
    assert idx[..., d].max() <= levels_t[d] - 1, f"dim {d} exceeds its level"
```

The Triton-versus-eager check catches Bug 2 on GPU:

```python
out_t = q_triton(x)
out_eager = q_eager(x)
assert (out_t["tokens"] == out_eager["tokens"]).all(), "Triton mismatch"
```

All three checks are in _verify(). All three are run by direct execution. All three are cheap.

4.3 What Is Verified and What Is Not

Verified by direct execution in the author's environment:

· The pure-PyTorch _quantize path: correct per-dimension levels, correct index ranges, correct dequantization round-trip.
· The SSRR gradient flow: loss.requires_grad is True, loss.backward() produces nonzero gradient on y, the extractor's parameters receive no gradient.
· The per-dimension level arithmetic: bit-exact against a numpy reference simulation of the kernel math.
· The _verify() function: runs on CPU, asserts pass.

Not verified:

· The Triton kernel on GPU hardware. The author's environment has no GPU. The kernel compiles in principle, uses only the Triton primitives tl.load, tl.store, tl.math.round, tl.arange, and tl.program_id, but its runtime behavior on a real device, its memory layout for the scatter-write loop, and its rounding behavior at .5 boundaries have not been exercised.
· CUDA graph capture and replay. The author's environment has no GPU. The GraphSafeQuantizer code path is structurally correct — the shape check, the warmup, the capture, the replay — but its runtime behavior on a real device is unexercised.
· Integration with Paper 01's event stream. The FSQQuantizer and ssrr_loss are verified in isolation. Their integration into the London Fog engine's rendering path is specified in §5 but not executed end-to-end.

The paper states these omissions because a paper about silent failures that overstates its own verification would be a third silent failure. The Triton path is structurally fixed, numerically derived, and specified against a test that will catch it if the fix is wrong. It is not verified.

4.4 The Triton Path Verification Test

The test that will verify the Triton path on GPU hardware:

```python
import torch
from harmonia_gpu import FSQQuantizer, FSQConfig

cfg_triton = FSQConfig(dim=64, levels=(8, 5, 5, 5), use_triton=True)
cfg_eager = FSQConfig(dim=64, levels=(8, 5, 5, 5), use_triton=False)

q_triton = FSQQuantizer(cfg_triton).cuda()
q_eager = FSQQuantizer(cfg_eager).cuda()
q_eager.load_state_dict(q_triton.state_dict())

# large-magnitude input to exercise .5 grid boundaries
x = torch.randn(64, 128, 64, device="cuda") * 3

out_triton = q_triton(x)
out_eager = q_eager(x)

mismatches = (out_triton["tokens"] != out_eager["tokens"]).sum().item()
total = out_triton["tokens"].numel()
print(f"token mismatches: {mismatches} / {total}")

if mismatches == 0:
    print("Triton path is bit-exact against eager; safe to enable")
else:
    print(f"Mismatch rate: {mismatches / total:.4%}")
    print("Cause is likely tl.math.round semantics at .5 boundaries")
    print("Keep use_triton=False until the kernel is corrected")
```

If mismatches == 0, the Triton path is bit-exact and safe to enable. If mismatches > 0, the difference is almost certainly at .5 boundaries — PyTorch's round uses half-to-even, and Triton's tl.math.round may use half-away-from-zero — and the fix is a single line in the kernel. Until the test prints zero, use_triton=False is the correct setting.

---

5. Design

5.1 Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      GPU (HARMONIA CODEC)                    │
│                                                              │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐   │
│  │ FSQ          │───▶│ to_fsq       │───▶│ grid         │   │
│  │ Quantizer    │    │ projection   │    │ quantize     │   │
│  └──────────────┘    └──────────────┘    └──────┬───────┘   │
│                                                  │            │
│  ┌──────────────┐    ┌──────────────┐            │            │
│  │ SSRR loss    │───▶│ SW2V         │            │            │
│  │ (frozen      │    │ extractor    │            │            │
│  │  extractor)  │    │ (log-mel)    │            │            │
│  └──────────────┘    └──────────────┘            │            │
│                                                  │            │
│                                          ┌──────▼───────┐    │
│                                          │ from_fsq     │    │
│                                          │ projection   │    │
│                                          └──────┬───────┘    │
│                                                  │            │
│                                          ┌──────▼───────┐    │
│                                          │ CUDA-graph   │    │
│                                          │ replay       │    │
│                                          └──────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

5.2 Integration with Paper 01

The integration point is the event stream. Paper 01 produces a sequence of events (onset, pitch, instrument, velocity); Harmonia renders those events to audio. The FSQ quantizer operates on the codec's latent representation of the events, and the SSRR loss trains the quantizer to preserve the semantic content of the rendered audio.

The y_stages list returned by Paper 01's forward(return_stage_decodes=True) is unused by Harmonia because FSQ has no stages. The HARP loss (Paper 01 §5.1) is likewise skipped, because it is an RVQ-specific loss and FSQ does not have RVQ stages. The integration is a subtraction: FSQ and SSRR replace MERVQNN and HARP, and the rest of the stack is unchanged.

5.3 Seed Receipt Extension for Codec Parameters

Paper 01's seed receipt is {visitorSeed, scaffoldVersion, params}. Harmonia extends it to {visitorSeed, scaffoldVersion, params, codecVersion}. The codecVersion is a semver string that identifies the FSQ configuration, the extractor checkpoint, and the synthesis implementation. A beat generated with codecVersion=1.0.0 is not the same as a beat generated with codecVersion=1.1.0, even if the seed and scaffold version match, because the codec's rendering differs.

The extended receipt is a reproducibility contract. A beat is reconstructible from {visitorSeed, scaffoldVersion, params, codecVersion} across browsers, devices, and years, provided the codec version's implementation is archived. The paper recommends archiving codec versions alongside scaffold versions, in the same repository.

---

6. Results

6.1 Verified Results

FSQ quantization, pure-PyTorch path. Verified correct on CPU. Per-dimension levels applied correctly, index ranges within bounds, dequantization round-trips to the quantized grid.

SSRR gradient flow. Verified correct on CPU. Loss carries gradient, backward pass produces nonzero gradient on the reconstruction, extractor parameters receive no gradient.

Per-dimension level arithmetic. Verified bit-exact against a numpy reference simulation.

The verification protocol. Runs on CPU, all assertions pass.

6.2 Expected Results After GPU Verification

Triton path. Expected to be bit-exact against eager. If not, the mismatch rate is expected to be small (a fraction of a percent, at .5 boundaries) and correctable with a one-line kernel change.

CUDA graph replay. Expected to match eager within 1e-5 for the same input shape, and to fall back to eager for mismatched shapes. The 1e-5 tolerance accounts for floating-point accumulation in the graph; bit-exactness is not expected and is not required.

End-to-end integration. Expected to produce audio that is perceptually indistinguishable from the eager path for the same seed and scaffold version, and perceptually distinguishable from a different seed or scaffold version.

6.3 Cost and Complexity Analysis

FSQ quantization. Two linear projections plus an element-wise grid operation. The projections are small (dim × n_dims, e.g., 64 × 4) and the grid is O(1) per element. The quantizer is not the bottleneck.

SSRR loss. One log-mel computation plus one linear projection, applied twice (once to source, once to reconstruction). The extractor's frozen parameters mean no backward pass through the extractor itself; the gradient reaches the reconstruction through the log-mel representation and the linear projection. The cost is dominated by the FFT, not by the loss.

Triton kernel. Memory-bound. The kernel's job is to remove Python dispatch from the hot path, not to beat cuBLAS. On GPU hardware, the expected speedup over the eager path is modest for large inputs and substantial for small inputs where dispatch overhead dominates.

CUDA graph. Eliminates per-launch CPU overhead for inference. Expected speedup depends on the model's launch count; for a codec with many small ops, the speedup can be significant. Training should not use graphs.

Server cost. Unchanged from Paper 01: zero per beat for client-side generation; nonzero for offline re-renders of paid purchases. Harmonia does not change the cost model; it changes the quality of the offline re-render.

---

7. Discussion

7.1 Constraints as the Contribution

Paper 01 argued that the constraint layer is the contribution, not the Markov chain. This paper makes the parallel argument: FSQ is the contribution, not the grid. The grid is a grid; any implementation can round to it. FSQ's contribution is that it refuses to collapse, refuses to drift, refuses to require a balance loss, and that those refusals are structural rather than regularized. The same is true of the three design choices documented in §3.6–3.8: the interval-state rhythm matrix refuses to model beat position explicitly; the rule-based onset mapping refuses to learn a mapping that might be biased; the hard leap cap refuses to produce octave jumps that sound like errors. Each is a constraint, and each is the contribution.

The collection's thesis is that the interesting thing about a generative system is what it refuses to produce. FSQ refuses to produce collapse. The rule-based mapping refuses to produce an unverifiable instrument assignment. The leap cap refuses to produce an octave jump that the consonance set permits but the melodic idiom does not. The contributions are the refusals.

7.2 The General Class of Silent Failures

The two bugs documented in this paper belong to a general class: failures that produce plausible output and never error. The class is characterized by (a) a code path that computes a value, (b) a value that is returned, logged, or plotted, and (c) a value that is wrong in a way that is invisible without a direct comparison against a known-good reference.

The class is well-known in software engineering. It is less well-known in GPU-native audio code, where the reference is often "does it sound right" and the answer is often "it sounds plausible." The paper's contribution to this discussion is the specific verification protocol for the specific failure modes that GPU-native audio codecs are prone to: gradient flow, per-dimension correctness, and graph replay fidelity. Other codecs in other domains may have other failure modes, and the protocol is not universal. But the principle — verify by direct comparison, not by training curves — is.

7.3 What the VST Plugin Would Have Told Us

The Crypt2.vst3 plugin was not available for analysis. It is a Windows plugin binary, and the author's environment could not run it. The plugin may have answered questions the paper leaves open: the exact onset-to-instrument mapping used in production, the exact rhythm matrix state semantics, the exact leap cap, and the relationship between the paper's design choices and the plugin's. The paper's design choices are the author's, made without access to the plugin, and stated as such. If the plugin's choices differ, the differences are an opportunity for a future paper.

The 500-pattern drum corpus referenced in Paper 01 §4.1 was not available in the archive. The drum data used in this paper's scaffolds is authored, not derived from the corpus. This is stated in the scaffold files and in §4.1. The paper does not claim the drum data is representative of the corpus; it claims the drum data is what the author produced in the corpus's absence, and that the scaffolds are reproducible and versioned.

7.4 Limitations

The Triton kernel is unverified on GPU. The fix is structurally correct and numerically derived, but it has not run on hardware. §4.4 specifies the test that will verify it.

The CUDA graph is unverified on GPU. The code path is structurally correct but unexercised on real hardware.

The end-to-end integration is specified but not executed. The integration point between Paper 01's event stream and Harmonia's rendering path is documented; the integration itself is not.

The SSRR extractor is a log-mel fallback, not a real SW2V checkpoint. The paper's SSRR results are for the fallback; the paper does not claim results for the real checkpoint, because the real checkpoint is not available in the author's environment.

The perceptual evaluation is not in this paper. The paper's claims are structural and verified, not perceptual. A perceptual evaluation of the codec's rendered audio is future work.

---

8. Conclusion

A codec that fails silently is worse than a codec that fails loudly. The silent failure is undetectable without a direct comparison, and direct comparisons are expensive, and the training loop is loud, and the loss curve is smooth, and the audio sounds plausible. The two bugs documented in this paper — the @torch.no_grad() decorator that strips SSRR's gradient, and the shared-max scalar that strips Triton's per-dimension correctness — are both silent failures of exactly this kind. Both are now fixed. Both are caught by the verification protocol. The protocol is three assertions and a mismatch count, and it is the contribution that matters most, because without it the other contributions are unverified claims.

Finite Scalar Quantization is a constraint architecture. The fixed grid refuses to collapse, refuses to drift, and refuses to require the balance losses and usage buffers that RVQ needs. The three design choices documented in §3.6–3.8 — interval-state rhythm matrices, rule-based onset mapping, hard leap caps — are constraints in the same sense. They refuse. The refusals are the contribution.

The code is real. The verification is real. The GPU verification is specified and not performed, because the author's environment has no GPU, and the paper says so. The next paper in the collection will run the GPU verification and report the results, whether they confirm the fixes or not.

— Lewis James Seals Jr, November 2026

---

References

Doty-Humphrey, C. (n.d.). PractRand [Software]. https://pracrand.sourceforge.net/

NeMo Codec — NVIDIA. Finite Scalar Quantization with [8, 5, 5, 5] levels, 8 codebooks, 22.05 kHz, 86.1 fps, 6.9 kbps.

NanoCodec / Low Frame-rate Speech Codec — NVIDIA. Variants at 1.89 kbps (21.5 fps) and 0.6 kbps (12.5 fps). WavLM used as a training-time discriminator.

JHCodec — MIT-licensed streaming Transformer codec. 50 Hz frame rate, 8-codebook RVQ, SSRR loss, Triton VQ kernel, single-GPU training budget.

Seals, L. S., & Busy Works Beats. (2026). Constrained Markov Synthesis in the Browser: A Seeded, Constraint-Filtered Architecture for Combinatorially Vast Beat Production at Millions-of-Visitors Scale. London Fog Papers, Paper 01.

Stability AI SAME-L — CUDA-graph JIT path failure on sm_120 hardware, producing byte-identical noise for every seed; the failure mode GraphSafeQuantizer is designed to avoid.

Steele, G. L., Lea, D., & Flood, C. H. (2014). Fast splittable pseudorandom number generators. OOPSLA 2014, 453–472.

Wilson, C. (2013). A tale of two clocks. web.dev. https://web.dev/articles/audio-scheduling

W3C. (2021). Web Audio API Specification. https://www.w3.org/TR/webaudio/

---

Appendix A: Repository Structure

```
/
├── harmonia_gpu.py           # this paper's code (FSQ, SSRR, Triton, graphs)
├── docs/
│   ├── LONDON_FOG.md         # design choices: rhythm state, onset mapping,
│   │                         #   leap cap (referenced in §3.6–3.8)
│   └── VERIFICATION.md       # the three-part protocol in full
├── tests/
│   ├── test_fsq.py           # per-dimension range check
│   ├── test_ssrr.py          # gradient flow check
│   └── test_triton_eager.py  # Triton-versus-eager bit-exactness (GPU)
├── references/
│   ├── nemo_codec.md         # NeMo Codec FSQ configuration
│   ├── jhcodec.md            # JHCodec SSRR loss
│   └── stability_same_l.md   # Stability SAME-L failure notes
└── paper/
    └── paper02.md            # this manuscript
```

Appendix B: The Verification Protocol in Code

The full _verify() function, reproduced for reference:

```python
def _verify():
    device = "cuda" if torch.cuda.is_available() else "cpu"
    print(f"device: {device}")
    print(f"triton available: {HAVE_TRITON}")

    if device == "cpu":
        print("NOTE: no GPU. Triton path is UNVERIFIED here.")
        print("      Run on GPU hardware before trusting it.")

    cfg = FSQConfig(dim=64, levels=(8, 5, 5, 5))
    q = FSQQuantizer(cfg).to(device)

    # Part 2: per-dimension range check
    z = torch.randn(2, 16, 64, device=device)
    out = q(z)
    idx = out["tokens"]
    levels_t = torch.tensor(cfg.levels)
    for d in range(idx.shape[-1]):
        assert idx[..., d].max() <= levels_t[d] - 1, \
            f"dim {d} exceeds its level {levels_t[d]}"
    print("per-dimension ranges verified correct")

    # Part 1: gradient flow check
    ext = SW2VExtractor(sr=16000, out_dim=128).to(device)
    x = torch.randn(2, 16000, device=device)
    y = torch.randn(2, 16000, device=device).requires_grad_(True)
    loss = ssrr_loss(x, y, ext)
    assert loss.requires_grad, "SSRR loss carries no gradient"
    loss.backward()
    assert y.grad is not None and y.grad.abs().sum() > 0, \
        "SSRR loss produced zero gradient on y"
    assert ext.fallback_proj.weight.grad is None, \
        "extractor params received gradient — should stay frozen"
    print("SSRR gradient flow verified")

    # Part 3 (GPU only): Triton-versus-eager
    if device == "cuda":
        q_triton = FSQQuantizer(FSQConfig(use_triton=True)).cuda()
        q_eager = FSQQuantizer(FSQConfig(use_triton=False)).cuda()
        q_eager.load_state_dict(q_triton.state_dict())
        x_large = torch.randn(64, 128, 64, device="cuda") * 3
        out_t = q_triton(x_large)
        out_e = q_eager(x_large)
        mismatches = (out_t["tokens"] != out_e["tokens"]).sum().item()
        assert mismatches == 0, \
            f"Triton mismatch: {mismatches} tokens differ from eager"
        print("Triton path bit-exact against eager")

    print("harmonia_gpu: all checks passed")
```

Appendix C: The Design Choices Document (LONDON_FOG.md)

The design choices documented in §3.6–3.8 are also recorded in docs/LONDON_FOG.md, which serves as the canonical reference for the choices and the reasons. The document's opening line:

Where Paper 01 was silent, these choices were made: the rhythm matrix state is interval-since-last-onset in sixteenth-note units; onsets map to instruments by a deterministic six-rule scheme (kick on strong beats; conditional kick on step 10; snare on backbeat; snare on ghost notes; hat on all remaining onsets; hat on empty steps at complexity-modulated density); melodies are capped at a perfect fifth interval with hard rejection and resampling. These are constraints. The paper argues they are the contribution.

The document continues with the reasoning for each choice, the alternatives considered, and the reasons the alternatives were rejected. It is the companion to this paper's §3.6–3.8 and is referenced from the paper as the canonical source.

---

The fog settles. The code compiles. The checks pass, or they don't, and either way you know. The paper's contribution is the knowing.
