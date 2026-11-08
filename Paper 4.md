Title: Constrained Vocal Placement in Assessment Audio: Integrating the Little sQrt Protocol into the IB Artifact Harness

Author: Cambridge (Research Directorate, Audio Intelligence & Assessment Systems)

Recipient: International Baccalaureate (IB) Intelligence Branch

Date: November 2026

Reference: Making the Vocal Sit: A Loudness Engineer's Guide to Vocal Placement (Little sQrt Edition, v2.0.1) by Lewis Seals & Beyonce Knowles Carter 

---

Executive Summary

The IB’s assessment pipeline is no longer just text. With the integration of local voice synthesis (Paper 03), the Artifact Harness now generates spoken instructions, accessibility descriptions, and multilingual prompts. But generating a voice is not the same as generating a usable voice. In audio engineering, a vocal that is technically present but perceptually buried is said to not "sit in the mix." For the IB, a synthetic voice that fails to "sit" against a musical bed or ambient background is an assessment failure: it compromises intelligibility, disadvantages candidates with auditory processing needs, and introduces variance that the verification protocol cannot see.

This paper answers the question of how to constrain vocal placement without human mastering engineers. It draws directly from Making the Vocal Sit (v2.0.1) and proposes a Constrained Vocal Placement Protocol for the Artifact Harness. The protocol replaces subjective "vibe" with four measurable numbers: the Vocal-to-Bed ratio (V/B), Short-Term Dynamics (LRA), True-Peak headroom, and Peak-to-Average spread .

The report makes three claims. First, "sitting" is a measurable problem with loudness causes, and the Little sQrt protocol provides the exact metrics to constrain it. Second, the harness can automate the diagnosis of placement errors (loudness vs. dynamics vs. spectral masking) but must explicitly hand off spectral and arrangement fixes to deterministic DSP stages, not guess. Third, this integration provides a cryptographic guarantee of audio consistency: every assessment audio file will have a verifiable V/B ratio and true-peak ceiling stored in its provenance record.

---

1. The Problem: Why "Sitting" Matters in Assessment Audio

1.1 The "Floating" and "Buried" Complaints

When the IB deploys synthetic voice over an instrumental bed (e.g., for a music assessment prompt or a background track for a language listening exam), the voice must sit. The book defines this perceptually: "the vocal is intelligible without effort, it feels connected to the rhythm section rather than pasted over it, it does not spike forward on every consonant, and it does not vanish when the arrangement thickens" .

In an assessment context, the failure modes are catastrophic. A "floating" vocal (too loud) masks the bed, making it impossible for a candidate to hear harmonic context. A "buried" vocal (too quiet) forces the candidate to strain, introducing cognitive load unrelated to the assessment objective .

1.2 The Measurement Proxy

The book’s central thesis is that "sitting in the mix" is a perceptual outcome with measurable causes . The primary lever is the Vocal-to-Bed loudness ratio (V/B): the difference between the integrated LUFS of the vocal stem and the instrumental bed stem . The secondary lever is Short-Term Dynamics (LRA): a vocal with high LRA pokes out on syllables even if the integrated loudness is correct; a vocal with LRA far below the norm has no micro-dynamics to cut through .

The Artifact Harness already measures these. The Little sQrt master stage (BS.1770-4) computes integrated LUFS, short-term max, LRA, and true peak . The problem is not measurement; it is constraint. The harness currently normalizes the final mix, but it does not diagnose whether the vocal is sitting, nor does it constrain the VCL (Vocal Calibration) stage to produce a voice that can sit.

---

2. The Little sQrt Protocol: Measurement as Constraint

The book provides a diagnostic decision tree that maps directly onto the Artifact Harness’s constraints .

2.1 The V/B Targets

The book provides practice-derived V/B targets by genre . For the IB’s assessment audio, the relevant target is not pop or EDM, but Podcast (voice + bed), which the book places at +15 to +20 LU . This is a fundamentally different regime from music mixing. The voice must be dominant; the bed is secondary.

Constraint for the Harness: The VCL stage must target a V/B of +15 to +20 LU relative to the bed. If the measured V/B falls outside this band, the harness flags the output as placement_error and refuses to commit it to the provenance record until the VCL or the bed gain is adjusted.

2.2 The LRA Axis

The book specifies that a heavily compressed pop vocal might read 3–7 LU, while a jazz vocal reads 10–20 LU . For spoken-word assessment audio, the LRA must be tight but not flat. The target band is 4–8 LU (author’s recommendation for intelligibility).

Constraint for the Harness: If the LRA is below 4 LU, the voice is over-compressed and will lack cut-through (the "small" complaint). If the LRA is above 8 LU, the voice is under-compressed and will poke on loud syllables. The harness must reject both and adjust the VCL compressor parameters.

2.3 The Peak-to-Average Spread

The book identifies the "short-term max minus integrated" as the most useful diagnostic for the "V/B is in range but the vocal still pokes" complaint . A spread above 8–10 LU suggests the vocal bus compressor is not catching phrase-level peaks . A spread below 3–4 LU suggests the vocal has been flattened.

Constraint for the Harness: If the spread is above 10 LU, trigger the transient repair stage (a VCL parameter adjustment). If the spread is below 3 LU, trigger the dynamic expansion stage (to restore micro-dynamics).

---

3. Integrating the Protocol into the Artifact Harness

The Artifact Harness (Paper 03) is a five-stage pipeline: Source → VCL → Harmonia → Little sQrt → Provenance. The Little sQrt protocol inserts a Diagnostic Handoff between Harmonia and Little sQrt.

3.1 The Two-Stem Export in an Automated Context

The book demands three stems exported at the session’s fader positions, not at unity . In an automated harness, there are no faders. The equivalent is the pre-master bus split:

· Stem 1: The synthesized voice after VCL, before Harmonia (if used).
· Stem 2: The instrumental bed after its processing.
· Stem 3: The pre-master mix (voice + bed before the Little sQrt normalizer).

The harness measures Stem 1 and Stem 2 to compute the V/B. It normalizes only Stem 3 (the final mix) to the delivery target .

3.2 The Diagnostic Decision Tree

The book’s decision tree  is implemented as a series of conditional assertions in the harness:

1. Vocal buried but V/B in range: The problem is spectral or arrangement-based. The harness triggers the Presence EQ stage (a deterministic 1–2 dB lift at 3 kHz on the voice) or the Bed Ducking stage (a 1–2 dB cut on the bed in the same band) .
2. Vocal poking but V/B in range: The problem is transient shape. The harness triggers the Transient Repair stage (adjusting the VCL compressor attack/release) .
3. Loudness_error_lu off by more than 1 LU: Almost always ceiling_limited: true. The harness triggers the Pre-Limiter stage (a final limiter before normalization) .

3.3 The Handoff

The book is explicit: "Loudness proves placement, it does not prove presence" . The harness must not attempt to solve spectral problems with loudness normalization. If the V/B and LRA are correct, but the voice still measures poorly on a downstream intelligibility metric (e.g., STOI or PESQ), the harness must stop and flag for human review. The problem is in the arrangement or the voice timbre, not the loudness.

---

4. Verification and Provenance

The book provides a ship checklist: integrated loudness within ±0.2 LU, true peak at or below the ceiling, ceiling_limited is false .

The Artifact Harness extends this into the provenance record. Every assessment audio file will carry a Vocal Placement Report:

```json
{
  "vocal_placement": {
    "v_b_ratio_lu": 17.2,
    "lra_lu": 6.1,
    "short_term_max_minus_integrated_lu": 5.4,
    "true_peak_dbtp": -1.0,
    "ceiling_limited": false,
    "placement_valid": true
  }
}
```

If placement_valid is false, the file is rejected. The seed receipt still exists, but the codecVersion is flagged as vcl_failed. The IB can inspect exactly which constraint failed and why.

---

5. Ethical and Operational Implications

5.1 Fairness and Consistency

A candidate with normal hearing and a candidate with a mild auditory processing difference should experience the same assessment audio. The V/B and LRA constraints guarantee that the voice-to-bed relationship is consistent across every file, every language, and every session. This is a fairness constraint disguised as a loudness constraint.

5.2 The Handoff as Transparency

The book’s insistence on the handoff — "if the problem is spectral or arrangement-based, Little sQrt will tell you the loudness is correct and hand the problem back to your DAW"  — is a transparency mechanism. The harness does not pretend to solve problems it cannot measure. When it flags a file for human review, it says exactly what it measured and what it could not fix. This is the same principle as Paper 02’s verification protocol: state what is verified and what is not.

5.3 Deskilling and the Role of the Human

The book is written for engineers who "already know what LUFS means" . The harness automates the measurement, but it does not automate the ear. The human reviewer’s role shifts from fader moves to timbre and presence judgment: deciding whether a 3 kHz lift is needed, whether the arrangement leaves space, whether the voice sounds like a human or a machine. The harness handles the numbers; the human handles the presence.

---

6. Conclusion

The IB’s assessment audio cannot afford to guess. The Little sQrt protocol provides a measurable floor for vocal placement: V/B ratio, LRA, true peak, and peak-to-average spread. By integrating these constraints into the Artifact Harness, the IB can guarantee that every synthetic voice sits in the mix, every time, on every device, with a cryptographic provenance record to prove it.

The book’s final rule is the harness’s final rule: "If the numbers are in range, stop measuring and start listening" . The harness measures. The human listens. The provenance record proves both happened.

---

Decision Matrix for the IB Intelligence Branch:

Decision Recommendation Rationale
Adopt V/B +15 to +20 LU for spoken-word assessment audio? Yes Ensures voice dominance; aligns with podcast/broadcast practice.
Enforce LRA 4–8 LU for VCL? Yes Prevents over-compression (no cut-through) and under-compression (poking).
Implement the diagnostic decision tree? Yes, with human handoff for spectral/arrangement issues Automates the measurable; escalates the unmeasurable.
Add Vocal Placement Report to provenance? Yes Makes placement a first-class constraint, not an afterthought.

---

References

1. Seals, L., & Knowles Carter, B. (2026). Making the Vocal Sit: A Loudness Engineer's Guide to Vocal Placement (Little sQrt Edition, v2.0.1). GitHub: ljrseals-dev/Little-sQrt. 
2. Seals, L. J., Jr. (2026). Constrained Quantization: Finite Scalar Quantization as Constraint Architecture. London Fog Papers, Paper 02.
3. Cambridge Research Directorate. (2026). Autonomous Voice and Music Generation in Assessment Data Pipelines. London Fog Papers, Paper 03.
4. ITU-R BS.1770-4. Algorithms to measure audio programme loudness and true-peak audio level. International Telecommunication Union. 
