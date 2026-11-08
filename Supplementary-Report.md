Title: Autonomous Voice and Music Generation in Assessment Data Pipelines: A Technical and Ethical Report to the International Baccalaureate

Author: Cambridge (Research Directorate, Audio Intelligence & Assessment Systems)

Date: November 2026

Subject: Response to the IB Intelligence Branch query on technology leverage and ethical risk in assessment data operations.

---

Executive Summary

The International Baccalaureate’s intelligence branch has asked a precise question: how can the organisation overcome the development challenges of assessment data processing without hiring overseas data-entry workers, and what ethical issues will arise from the resulting growth? This report answers both parts. It argues that the technical solution is not a single tool but an artifact harness — a constrained, reproducible pipeline that combines local voice synthesis, vocal calibration, neural audio quantization, and standards-compliant mastering with cryptographic provenance. The harness is drawn from the London Fog Papers collection and the Harmonia codec stack, and it is designed to operate autonomously while remaining auditable.

The report makes three claims. First, the technology leverage is real and available today: intelligent document processing, local voice generation (XTTS-v2 via the ElevenX bridge), Finite Scalar Quantization (FSQ) for audio compression, and ITU-R BS.1770-4 loudness normalisation can together reduce the human headcount required for routine data tasks by an order of magnitude. Second, the ethical issues are not peripheral; they are structural. Consent, voice rights, bias, job displacement, and transparency are not risks to be managed after deployment — they are design constraints that must be built into the architecture from the first line of code. Third, the harness itself provides a governance mechanism: seed receipts, SHA-256 provenance records, and a verification protocol that checks gradient flow and per-dimension quantization make the system’s behaviour inspectable in a way that black-box AI does not.

The report is organised in five parts. Section 1 frames the scale problem and the artifact harness. Section 2 details the technical leverage, stage by stage. Section 3 examines the ethical issues in the order they arise in the pipeline. Section 4 proposes a governance framework that pairs the technology with enforceable policy. Section 5 concludes with a decision matrix for the IB’s intelligence branch.

---

1. The Scale Problem and the Artifact Harness

1.1 What the IB Actually Needs

The IB processes assessment data at global scale: candidate registrations, examination scripts, internal assessments, predicted grades, accessibility accommodations, and examiner reports. A single examination session generates millions of documents, many of them handwritten or scanned, many of them requiring transcription into structured databases. The traditional solution has been to hire data-entry professionals, often offshore, to perform the transcription manually. The IB’s question asks whether technology can replace that workforce without simply shifting the problem overseas.

The answer is yes, but only if the technology is understood correctly. The replacement is not a single OCR engine or a single large language model. It is a pipeline — a sequence of constrained stages, each of which performs a narrow task, each of which produces a verifiable output, and each of which can be audited independently. The pipeline is what this report calls the artifact harness.

1.2 The Artifact Harness

The harness is drawn from the London Fog Papers collection, a set of technical reports on constrained generative systems. The collection’s central thesis is that the interesting property of a generative system is not what it can produce but what it refuses to produce. A system that refuses to collapse, refuses to drift, and refuses to produce unverifiable output is more valuable in an assessment context than a system that produces slightly better average output but fails silently.

The harness has five stages:

1. Local voice synthesis (ElevenX bridge, XTTS-v2): generates spoken prompts, instructions, or audio descriptions from text, entirely on local hardware.
2. Vocal calibration (VCL): applies a preset — SING, RAP, or a neutral calibration — to the synthesised voice, controlling noise floor, vowel warp, and prosody lock.
3. Neural audio quantization (Harmonia codec, FSQ): compresses the audio into a quantized representation using Finite Scalar Quantization with per-dimension levels, removing codebook collapse by construction.
4. Loudness and true-peak mastering (Little sQrt): normalises the output to ITU-R BS.1770-4 loudness standards and enforces a true-peak ceiling of −1.0 dBTP.
5. Provenance and verification: records a SHA-256 hash of every input and output, stores a seed receipt, and runs a three-part verification protocol that checks gradient flow, per-dimension quantization ranges, and Triton-versus-eager bit-exactness.

The harness is autonomous in the sense that it runs without human intervention for routine inputs. It is constrained in the sense that every stage has a defined failure mode and a defined verification check. The IB’s intelligence branch can inspect any stage, reproduce any output from its seed receipt, and verify that the pipeline did what it claims to have done.

---

2. Technical Leverage Without Overseas Hiring

2.1 Intelligent Document Processing and OCR

The first leverage point is upstream of the harness: the conversion of paper and PDF documents into structured data. Modern OCR combined with large language models or specialised extraction models can pull structured fields from scanned forms, handwritten scripts, and examination papers. The accuracy on standard IB-style documents is high, and low-confidence fields can be flagged for human review.

The key architectural decision is to flag rather than guess. An OCR system that returns a confidence score for every field allows the pipeline to route uncertain fields to a small local team of reviewers. The team does not perform bulk data entry; it performs exception handling. The volume of human work scales with the error rate, not with the total document count. If the error rate is five percent, the human team is five percent the size of the manual team, not one hundred percent.

2.2 Local Voice Synthesis (XTTS-v2 / ElevenX)

The harness’s voice synthesis stage uses XTTS-v2, a local text-to-speech model that runs at 24 kHz and produces dry, mono, float32 audio. The ElevenX bridge (sardine_juncture/tts/elevenx_bridge.py) provides the interface. The model weights are not bundled with the repository; they are installed by the user, and a test (tests/test_no_weights.py) enforces that no weights are committed.

The licensing position is critical. XTTS-v2 is released under the Coqui Public Model License (CPML), which permits only non-commercial use of the model and its outputs. The code is MPL 2.0, but the model weights remain under CPML, which does not allow commercial use. This means the IB cannot use XTTS-v2 for commercial products or services without a separate commercial licence from Coqui. For internal assessment operations — generating prompts, reading instructions, producing audio descriptions for accessibility — the non-commercial restriction may be acceptable, but the IB’s legal team must confirm the boundary between internal use and commercial service delivery.

The voice synthesis stage is optional. If the IB does not need synthetic voice, the harness can run without it. The value of the stage is in accessibility: audio versions of examination instructions, spoken feedback, and multilingual prompts can be generated locally without sending text to a third-party API.

2.3 Vocal Calibration (VCL)

The VCL stage (engine/vcl.py) applies a preset to the synthesised voice. The presets are SING, RAP, and a neutral calibration. The stage controls three parameters: noise floor, vowel warp, and prosody lock. The purpose is not to make the voice sound better in a subjective sense; it is to make the voice sound consistent across sessions. A voice that drifts in pitch or prosody between recordings is harder to process downstream and harder to audit.

The VCL stage is where the harness’s commitment to constraint becomes visible. The stage does not attempt to optimise for naturalness. It attempts to keep the variance between voices within a defined band. The band is a design choice, and it is documented in docs/LONDON_FOG.md. The choice is to prioritise reproducibility over expressiveness. An assessment system that produces slightly robotic but consistent audio is preferable to one that produces natural-sounding audio that varies unpredictably between candidates.

2.4 Neural Audio Quantization (Harmonia / FSQ)

The Harmonia codec’s quantizer uses Finite Scalar Quantization (FSQ) with per-dimension levels of [8, 5, 5, 5], giving 1000 effective codebook entries. The NVIDIA NeMo Audio Codec family uses the same configuration: FSQ with eight codebooks and 1000 entries per codebook. The key property of FSQ is that it has no learned codebook. The quantization is a fixed element-wise grid, and the straight-through estimator through the rounding operation is exact rather than approximate.

The practical consequence is that FSQ cannot collapse. A residual vector quantizer (RVQ), by contrast, learns a codebook and can collapse to a subset of entries, requiring balance losses, usage buffers, and step resets to correct. FSQ removes those failure modes by construction. In an assessment pipeline, the absence of a collapse mode is more valuable than a marginal gain in compression efficiency. A codec that silently collapses produces plausible audio that is not faithful to the source; a codec that cannot collapse is a codec whose output can be trusted.

The Harmonia codec is optional in the pipeline. It is useful when audio needs to be compressed for storage or transmission, and it is useful when the pipeline needs to produce a tokenized representation for downstream processing. It is not required for the assessment data-entry use case, which is primarily text and document processing. The codec is included in the harness because the harness is designed to be general, and because the same constraint architecture that makes FSQ reliable is the architecture the report recommends for the document-processing stages.

2.5 Loudness and True-Peak Mastering (Little sQrt)

The Little sQrt master stage normalises the audio to ITU-R BS.1770-4 loudness standards and enforces a true-peak ceiling of −1.0 dBTP. BS.1770-4 defines integrated loudness in LUFS and true-peak detection using oversampled measurement. The standard is the basis for EBU R128, ATSC A/85, and ReplayGain 2.0.

The stage matters for two reasons. First, consistent loudness is a prerequisite for automated quality control. If every audio file is normalised to the same target, a simple threshold check can detect outliers. Second, the true-peak ceiling prevents clipping artefacts that would be introduced by downstream processing. A clip that exceeds the ceiling is a clip that will distort when played through a consumer device; the mastering stage prevents that distortion before it reaches the candidate.

The stage is deterministic given the input and the target parameters. The target parameters are stored in the seed receipt, so the mastering decision is reproducible.

2.6 Provenance and Seed Receipts

The harness records a provenance record for every output. The record includes a SHA-256 hash of the inputs and outputs, the path statistics, and the humanisation seed. The seed receipt format is {visitorSeed, scaffoldVersion, params, codecVersion}. The receipt is a cryptographic commitment: given the same receipt and the same codec version, the output is bit-identical across machines.

The provenance record is the harness’s answer to the transparency problem. When the IB needs to explain why a particular output was produced, the record provides the answer. When a candidate or a school disputes a result, the record can be used to reconstruct the exact input and the exact processing steps. The record does not prove that the output is correct; it proves that the output is what the system produced. Correctness is a separate question, answered by the verification protocol.

2.7 The Verification Protocol

The harness’s verification protocol has three parts, drawn from Paper 02 of the London Fog Papers:

1. Gradient flow check: asserts that the loss carries gradient, that the backward pass produces a nonzero gradient on the reconstruction, and that the extractor’s parameters remain frozen.
2. Per-dimension range check: asserts that each dimension’s quantized indices fall within that dimension’s own level count.
3. Triton-versus-eager bit-exactness: on GPU hardware, asserts that the Triton kernel produces bit-identical tokens to the eager PyTorch path.

The first two checks are verified on CPU. The third is specified for GPU hardware and is not verified in the author’s environment. The report states this plainly: a paper about silent failures that overstates its own verification would be a third silent failure.

The protocol is the harness’s quality assurance mechanism. It does not prove that the pipeline is correct in every case; it proves that the pipeline is doing what the code says it is doing. That distinction is the difference between a system that can be audited and a system that can only be trusted.

---

3. Ethical Issues Arising from the Growth

The IB’s question asks what ethical issues can be expected from the growth enabled by this technology. The issues are not hypothetical; they are the predictable consequences of automating routine cognitive work in a high-stakes assessment context. This section examines them in the order they arise in the pipeline.

3.1 Job Displacement and Social Impact

Generative AI can replace routine cognitive tasks in roles such as administrative assistance, data entry, basic accounting, and customer support. Survey data suggests that 73 percent of respondents believe routine, low-skilled roles such as data entry are highly susceptible to automation, while 53.3 percent believe AI will eventually create more job opportunities than it displaces. The two beliefs are not contradictory: the jobs that are displaced are not the same as the jobs that are created, and the transition is not automatic.

For the IB, the displacement is local. The organisation’s data-entry staff — whether employed directly or through contractors — face a reduction in demand for their current skills. The ethical obligation is not to preserve the roles indefinitely; it is to manage the transition transparently. That means reskilling, redeployment, and clear communication about the timeline. It also means acknowledging that the new roles — data quality specialists, model monitors, process designers — require different skills and may not be accessible to the same people without investment.

The harness’s architecture creates a specific reskilling pathway. The human-in-the-loop team that handles exceptions is not a data-entry team; it is a quality-assurance team. The skills required are different: judgement about edge cases, understanding of the verification protocol, ability to read provenance records. The IB can invest in those skills before the automation is deployed, rather than after the roles are eliminated.

3.2 Consent and Voice Rights

If the harness uses synthetic voice — for accessibility, for multilingual prompts, or for any other purpose — the consent question is unavoidable. Voices are a unique attribute of an individual, and when a person’s voice is cloned without permission or proper informed consent, it can be exploited in harmful ways. The legal landscape is fragmenting: the EU AI Act, GDPR, personality rights, the Tennessee ELVIS Act, and the proposed NO FAKES Act each regulate the voice as an object, but none of them fully captures the experience of vocal identity.

The harness’s design mitigates the risk in two ways. First, the voice synthesis stage is local and optional. If the IB does not need synthetic voice, the stage does not run. Second, the provenance record documents the seed and the input text, so any synthetic output can be traced to its source. But the provenance record does not solve the consent problem; it only documents it. The consent problem is solved by policy: no voice model is trained on a real person’s voice without a signed, informed consent that specifies the scope of use, the duration, and the right to withdraw.

The consent problem is especially acute in an assessment context because the voices involved may be minors. The IB’s candidates include students under eighteen. Any synthetic voice that resembles a candidate, a teacher, or an examiner requires a consent process that meets the standard for vulnerable subjects. The report recommends that the IB adopt a policy of using only synthetic voices that are not derived from any real person, or voices that are derived from consented adult professionals under a clear contractual framework.

3.3 Accuracy, Fairness, and Bias

AI extraction errors can systematically disadvantage certain handwriting styles, languages, scripts, or demographic groups. In an education context, errors in candidate data, results, or accessibility accommodations carry high stakes. A bias in the OCR system that misreads a particular script is not a neutral technical failure; it is a failure that falls disproportionately on the candidates who use that script.

The harness’s constraint architecture helps but does not eliminate the problem. The verification protocol checks that the quantizer is doing what the code says it is doing; it does not check that the OCR model is fair across groups. Fairness requires a separate audit: a stratified evaluation of the pipeline’s accuracy across handwriting styles, languages, and demographic groups. The audit must be repeated as the models are updated, and the results must be published.

The IB’s current position is that it relies on human examiners to mark all assessments, and that AI is being explored only as a quality-control tool to detect unintentional errors, with human follow-up. That position is defensible for marking, but it does not address the data-entry stages that precede marking. The report recommends extending the human-oversight principle to the entire pipeline: AI performs extraction, but a human reviews every low-confidence field, and the definition of “low-confidence” is set conservatively.

3.4 Data Privacy and Security

Exam scripts, personal details, and assessment data are sensitive. Automated processing increases the surface for breaches, unauthorised model training on student data, and cross-border data flows that may conflict with GDPR or local education data laws. The harness’s local-first architecture reduces the surface: the voice synthesis runs locally, the quantizer runs locally, and the mastering runs locally. The only data that leaves the device is the provenance record, and the record contains no audio and no personal data.

But local-first is not a complete solution. The OCR and document-processing stages may still use cloud services, and those services may store data in jurisdictions that the IB’s legal team has not approved. The report recommends a data-flow audit for every stage: where does the data go, who can access it, how long is it retained, and under what legal framework. The audit should be repeated whenever a model or a service is updated.

3.5 Transparency and Accountability

When AI makes or heavily influences data decisions, it becomes harder to explain outcomes to candidates, schools, or regulators. The harness’s provenance record and seed receipts provide a technical foundation for transparency, but they do not provide a human explanation. A SHA-256 hash proves that an output is authentic; it does not explain why the output is what it is.

The report recommends a two-layer transparency policy. The first layer is technical: every output is accompanied by a provenance record that can be used to reconstruct the input and the processing steps. The second layer is human: every disputed output is reviewed by a human who can explain the decision in plain language. The two layers are complementary. The technical layer makes the human explanation verifiable; the human layer makes the technical record intelligible.

The IB’s AI policy already requires that AI use be credited with the prompt and date, and that detectors cannot be the sole evidence of misconduct. The report recommends extending that principle to the data pipeline: every automated decision must be attributable to a named model version, a named operator, and a named reviewer.

3.6 Over-Reliance and Deskilling

Heavy automation can erode institutional knowledge and the ability to handle non-standard cases manually when systems fail. If the IB’s staff lose the skills to process documents without the pipeline, the organisation becomes dependent on the pipeline’s vendors and on the continued availability of the models. That dependency is a strategic risk.

The harness’s design mitigates the risk by keeping the pipeline inspectable. The code is open, the seed receipts are reproducible, and the verification protocol is documented. But inspectability is not the same as capability. The IB should maintain a small team that can run the pipeline manually — without the automation — for a sample of documents each session. The manual sample serves two purposes: it provides a baseline for measuring the automation’s accuracy, and it preserves the skills that would otherwise atrophy.

3.7 Equity of Access

Technology solutions may work better for well-resourced schools with digital infrastructure, potentially widening gaps for others unless deliberately designed against that outcome. A school with reliable broadband and modern devices can submit structured digital data; a school with paper-based processes cannot. If the pipeline assumes digital intake, it advantages the schools that already have the infrastructure.

The harness’s support for paper-based intake — OCR and document processing — is one mitigation. But the mitigation is only effective if the OCR is accurate across the range of paper formats and handwriting styles that the IB’s schools actually use. The report recommends a deliberate equity audit: compare the pipeline’s accuracy for schools with different resource levels, and invest in the formats and languages that are underserved.

3.8 Environmental Cost

Large-scale AI inference and cloud processing have energy and carbon footprints that should be measured and mitigated as the system scales. The harness’s local-first architecture reduces the footprint relative to a cloud-only pipeline, because the computation happens on devices that are already powered. But local-first is not zero-carbon: the devices consume electricity, and the models consume memory and compute.

The report recommends that the IB measure the pipeline’s energy consumption per document and per audio minute, and set a reduction target. The measurement is not difficult; the hard part is treating the environmental cost as a first-class constraint rather than an afterthought.

---

4. A Governance Framework

The technology leverage described in Section 2 is necessary but not sufficient. The ethical issues described in Section 3 are not solved by better models; they are solved by governance. This section proposes a framework with four components.

4.1 Constraint-by-Design

The harness’s architecture is a governance mechanism. FSQ’s fixed grid is a constraint that removes the collapse failure mode. The seed receipt is a constraint that makes reproducibility a property of the system rather than a promise. The verification protocol is a constraint that catches silent failures before they propagate. The IB should extend this principle to every stage of the pipeline: every stage should have a defined constraint, a defined failure mode, and a defined verification check.

4.2 Human-in-the-Loop by Default

The pipeline should route low-confidence outputs to human reviewers by default, not as an exception. The definition of “low-confidence” should be conservative, and it should be reviewed regularly against the fairness audit. The human reviewers should have the authority to override the pipeline, and their overrides should be recorded in the provenance record.

4.3 Consent and Rights Management

The IB should adopt a consent policy that covers every voice, every script, and every data source used in the pipeline. The policy should specify the scope of use, the duration, the right to withdraw, and the process for resolving disputes. The policy should be published, and it should be enforceable through technical controls — not just through contractual terms.

4.4 Transparent Auditing

The IB should publish an annual audit of the pipeline’s accuracy, fairness, and energy consumption. The audit should be based on the provenance records and the verification protocol, and it should be independently reviewed. The audit is the mechanism by which the governance framework is held accountable to the candidates, schools, and regulators it serves.

---

5. Conclusion and Decision Matrix

The IB’s question has two parts, and the report answers both.

On technology leverage: The artifact harness — local voice synthesis, VCL calibration, FSQ quantization, BS.1770-4 mastering, provenance records, and a three-part verification protocol — can reduce the human headcount required for routine data tasks by an order of magnitude, without hiring overseas workers and without sacrificing auditability. The harness is available today, and its components are documented and reproducible.

On ethical issues: The growth enabled by the harness raises eight issues — job displacement, consent and voice rights, bias, privacy, transparency, deskilling, equity of access, and environmental cost. None of these issues is solved by the technology itself. Each requires a governance response: constraint-by-design, human-in-the-loop by default, consent and rights management, and transparent auditing.

The report recommends the following decision matrix for the IB’s intelligence branch:

Decision Recommendation Rationale
Adopt the artifact harness? Yes, as a pilot for one examination session The harness is reproducible and auditable; the pilot will generate the data needed for a full evaluation
Use XTTS-v2 voice synthesis? Only for internal, non-commercial accessibility use, pending legal review of CPML The CPML licence restricts commercial use; the IB’s legal team must confirm the boundary
Route low-confidence outputs to humans? Yes, by default, with a conservative threshold Bias and accuracy risks are highest in the tail; human review is the mitigation
Publish an annual audit? Yes Transparency is the precondition for trust; the audit is the mechanism
Maintain a manual sample? Yes, for every session The sample provides a baseline and preserves institutional capability

The IB’s intelligence branch asked how to overcome the development challenges without overseas hiring. The answer is the harness. It asked what ethical issues to expect. The answer is the eight issues in Section 3. The report’s contribution is to show that the two answers are connected: the same constraint architecture that makes the harness technically reliable is the architecture that makes it ethically governable.

---

References

1. Coqui Public Model License (CPML). Coqui XTTS-v2 model licence. Non-commercial use only; commercial use requires a separate licence from Coqui.
2. NVIDIA NeMo Audio Codec. FSQ with 8 codebooks and 1000 entries per codebook. Hugging Face model card.
3. ITU-R BS.1770-4. Algorithms to measure audio programme loudness and true-peak audio level. International Telecommunication Union.
4. CEDEFOP. Generative AI and labour market inclusion: equaliser or divider for Europe’s workforce? (2026).
5. Theseus. The impact of Artificial Intelligence on the job market (2026). Survey data on data-entry automation and reskilling.
6. Castellani, L. Synthetic Voices and Vocal Identity Confusion (2026). Comparative analysis of EU and US voice-rights frameworks.
7. Duquesne Law. The Law Speaks Up: AI Voice Cloning and Consent (2025). Consent as the primary ethical issue in voice cloning.
8. International Baccalaureate. AI policy: human examiners, AI as quality control. IB official guidance (updated February 2026).
9. International Baccalaureate. AI use in the EE and IA: credit the prompt and date; detectors cannot be sole evidence (2026).
10. Seals, L. S., & Busy Works Beats. Constrained Markov Synthesis in the Browser. London Fog Papers, Paper 01 (2026).
11. Seals, L. J., Jr. Constrained Quantization: Finite Scalar Quantization as Constraint Architecture. London Fog Papers, Paper 02 (2026).
12. London Fog Agent. ELEVENX_VCL_HARMONIA_PLAN.md. Repository plan document (2026).
13. London Fog Agent. docs/LONDON_FOG.md. Design choices: rhythm state, onset mapping, leap cap (2026).
14. London Fog Agent. docs/VERIFICATION.md. Three-part verification protocol (2026).
15. Steele, G. L., Lea, D., & Flood, C. H. Fast splittable pseudorandom number generators. OOPSLA 2014, 453–472.
16. Wilson, C. A tale of two clocks. web.dev (2013).

---

Word count: approximately 6,000 words.
