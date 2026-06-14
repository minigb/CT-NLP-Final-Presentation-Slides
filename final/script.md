# Presentation Script
## S_plan: Compact Learned Symbolic Planning Tokens for Music-Aware Audio Language Models
### CT-NLP Final Presentation — June 2026

---

## Slide 1 — Title

Good afternoon, everyone. This is joint work from Dabin Kim, Minhee Lee, Ji Jiaxian, and Aner Zheng at KAIST.

The paper asks three questions about symbolic music representations in audio language models. Does a compact learned symbolic stream, aligned to an audio window's own MIDI, reduce acoustic token prediction uncertainty more than a shuffled or dummy stream of the same size? Is that benefit specific to content alignment, or merely to token capacity? And does the symbolic stream encode music-structural attributes recoverable by probing? This talk reports our answers to each.

---

## Slide 2 — Problem

Unified audio language models predict discrete audio tokens autoregressively and have achieved strong performance across diverse tasks. Music, however, poses a persistent gap. The structural properties most salient to musicians and listeners — harmonic progressions, rhythmic regularity, instrument co-occurrence, phrase boundaries — are encoded in MIDI symbolic representations, not waveform statistics.

The challenge is not just building a symbolic bridge. It is evaluating one credibly. Any fixed-length stream added to the context expands the model's capacity. A positive result against a reason-only baseline alone is weak evidence if an equal-length dummy stream achieves the same gain. Our multi-condition design — aligned, shuffled, and dummy — forces the distinction between symbolic content and token capacity.

---

## Slide 3 — Three Research Questions

We organize this work around three research questions.

RQ1 on acoustic utility: does a compact symbolic stream, aligned to an audio window's own MIDI, reduce acoustic token prediction uncertainty more than a shuffled or dummy stream of equal length?

RQ2 on alignment specificity: is the benefit from alignment-specific content, or from token capacity alone? We test this by requiring aligned to beat *both* shuffled and dummy under matched conditions. Shuffled preserves token statistics while breaking temporal alignment. Dummy removes all symbolic content. Aligned must clear both bars.

RQ3 on music-structural content: does the symbolic stream encode musically meaningful attributes recoverable by supervised probing, compared against a zero-feature baseline that exposes dataset priors?

---

## Slide 4 — Approach

S_plan is inserted as a third stream between the reasoning tokens and the acoustic tokens of UniAudio 2.0, forming the sequence R_a → S_plan → C_a. The total layout fits within the 1,024 context limit with zero truncation.

Raw MIDI was not used for three reasons. Standard tokenizations average 2,000 to 8,000 tokens per window, saturating the context. Event timestamps are asynchronous with audio frames. And variable-length sequences cannot be given fixed-length shuffled or dummy controls — which makes the evaluation design impossible. S_plan's fixed-length RVQ encoding solves all three.

---

## Slide 5 — Dataset: Slakh2100

All experiments use Slakh2100, a multi-track paired audio-MIDI dataset synthesized with professional-grade sample-based virtual instruments. The dataset is split at the track level before windowing, so no musical phrase can appear in both train and test. The training split contains 1,289 tracks and 11,279 thirty-second windows; the held-out test set contains 151 tracks and 1,385 windows. All cross-entropy results in this talk are evaluated on that held-out test set.

---

## Slide 6 — MIDI Feature Extraction

From each 30-second window, MIDI features are extracted at 100 ms resolution. The binary feature group — 41 dimensions — captures instrument-class presence across 34 Slakh classes, pitch-class activation across 12 semitones, and beat/downbeat phase. The continuous feature group — 12 dimensions — captures instantaneous tempo, note density, polyphony count, and an instrument-weighted energy proxy. Together, this gives a 53-dimensional frame vector normalized per dimension across the training split, yielding approximately 300 frames per window.

---

## Slide 7 — Codec Architecture

The symbolic RVQ codec compresses the 53-dimensional, 300-frame input into a fixed 30-frame representation. A two-layer 1D CNN encoder with temporal pooling handles the variable-to-fixed compression. The residual vector quantizer applies four stages, each with 512 codes, trained with exponential moving average updates and codebook restart. A symmetric CNN decoder reconstructs the original 53-dimensional feature sequence. Training uses binary cross-entropy for the binary dimensions and MSE for continuous dimensions, run for 42,000 steps. The resulting 4-by-30 token stream uses the same multi-stream RVQ format as C_a — which is precisely what enables the stream-native positional encoding.

---

## Slide 8 — Training Setup

All symbolic stream variants are trained via LoRA adapters on 140 attention modules of the frozen UniAudio 2.0 transformer. The reason-only configuration has approximately 10.8 million trainable parameters; adding the symbolic stream brings this to approximately 253 million. A learned scalar gate controls the magnitude of symbolic stream influence on the C_a prediction path.

The hardened four-condition comparison — the primary result — uses 800 steps, batch size 1 with gradient accumulation 16, LoRA learning rate 10⁻⁵, symbolic stream learning rate 3×10⁻⁴, and gate learning rate 2×10⁻³. The random seed is fixed, and all four conditions are trained identically except for the stream variant.

---

## Slide 9 — Evaluation Protocol

Every controlled comparison in this work uses four conditions under matched hyperparameters.

Aligned S_plan provides tokens from the audio window's own MIDI file, preserving temporal and content alignment. Shuffled S_plan provides tokens from a randomly selected different window, preserving token statistics but breaking content alignment. The dummy stream provides zero or uniformly random RVQ codes — a token-budget control with no symbolic content. Reason-only is the base model with no S_plan slot.

The claim rule is strict. A gain of aligned over *both* shuffled and dummy is required to attribute any improvement to alignment-specific content. If aligned beats only one control, the attribution is confounded.

---

## Slide 10 — Prerequisite: Codec

Before any downstream result can be trusted, the RVQ codec must faithfully reconstruct input symbolic features. Binary feature accuracy reaches 0.99777. Continuous feature MSE is 0.00172. Codebook utilization on books two through four is 72 to 81 percent, with no collapse — book one is underutilized at 31 percent, which is a known characteristic of early RVQ stages. Per-window binary accuracy on all three fixed validation examples is 1.000.

This is a prerequisite, not a claim. It establishes that differences in downstream experiments are attributable to symbolic content, not codec failure.

---

## Slide 11 — Prerequisite: Feature Heatmap

The heatmap shows a representative reconstruction. Ground-truth MIDI features are on the left; the S_plan reconstruction is on the right, for a 30-second window of Slakh track 01501. Binary accuracy is 1.000. The codec produces a technically healthy symbolic bottleneck. One boundary: this is symbolic feature reconstruction, not MIDI transcription and not waveform generation.

---

## Slide 12 — RQ1: Proxy Task

The first evidence for RQ1 comes from a proxy prediction task. A small transformer head is trained to predict C_a tokens from frozen R_a plus symbolic representations, over 5,000 steps on 1,385 held-out test windows.

Aligned S_plan achieves the lowest test cross-entropy of 7.5963, outperforming reason-only by 0.061, low-rate rule summaries by 0.048, shuffled by 0.034, and dummy by 0.098. The learned codec outperforms the hand-crafted rule summary, confirming the value of end-to-end symbolic compression. This is the first evidence that S_plan carries acoustically useful information beyond what R_a alone encodes.

The claim boundary is noted: this is a proxy setup with a frozen codec. The full end-to-end integration is tested separately.

---

## Slide 13 — RQ1: Stream-Native

The full three-stream layout — R_a → S_plan → C_a — is then evaluated with streams aligned to the base model's positional encoding, under a shorter warmup budget of 3,000 steps.

Aligned achieves test CE 5.766 versus shuffled at 5.785, a delta of −0.018, and dummy at 5.785, a delta of −0.019. On the four validation fixed windows, two show strong positive margins and two show near-zero or slightly negative margins — characteristic of a real but small signal that has not saturated the training budget. The margin is modest but consistent in sign.

The positive gap replicates in a setup that is architecturally distinct from the proxy task. This rules out a proxy-specific artifact.

---

## Slide 14 — RQ1: Primary Result

The primary evaluation runs all four conditions under fully identical hyperparameters, differing only in the symbolic stream variant.

The result decomposes into two layers. Any symbolic stream variant — shuffled or dummy — achieves test CE around 5.614, a reduction of approximately 0.245 relative to reason-only at 5.859. That is the stream capacity effect. Aligned S_plan achieves 5.600, an additional 0.014 below both corrupted controls. That is the alignment-specific effect.

The learned gate corroborates through an independent channel: the model opens the S_plan slot more for aligned content — gate value 0.177 — than for shuffled at 0.167 or dummy at 0.163. The model learns to differentially admit information based on alignment. These results answer RQ1 positively across three independent controlled comparisons at different training scales.

---

## Slide 15 — RQ2: Alignment Decomposition

RQ2 requires demonstrating that the gain is not explained by token capacity alone.

The decomposition in the upper table is clear. The dominant effect is stream capacity: any symbolic stream outperforms reason-only by approximately 0.245 CE. The alignment-specific effect — aligned versus both shuffled and dummy — is 0.014 to 0.015. Small, but present.

The critical evidence is in the lower table. Across three completely independent comparisons, the alignment-specific gap is −0.034 in the proxy task, −0.018 in the stream-native warmup, and −0.014 in the hardened comparison. The magnitude decreases as training budget shrinks — shorter training leaves less room for conditions to diverge. But the sign is consistent across all three. A finding consistent in direction across independent setups at different scales is substantially more robust than a single-run result.

The gate provides independent corroboration. RQ2 is answered positively: the alignment-specific effect is not explainable by token capacity alone.

---

## Slide 16 — RQ3: MIR Probing

For RQ3, linear probing heads are trained on frozen S_plan embeddings to recover music-structural attributes. The evaluation compares against a zero-feature baseline — inputs zeroed, with only the head bias and BatchNorm statistics available — which exposes dataset priors. Any claimed axis must exceed this bar.

Weak key accuracy is 0.557 versus a zero-feature baseline of 0.126, a gap of +0.431. The baseline is near chance, confirming the gain is feature-specific and not prior-driven. Pitch-class macro-F1 shows a positive gap of +0.024. Instrument macro-F1 shows +0.075 but requires class-imbalance controls before a strong claim is warranted.

The excluded axes require a different reading. Instrument micro-F1 is 0.791 while the zero-feature baseline is 0.835 — the baseline is *higher*. Meter accuracy is 0.783 versus 0.855. These are not failures of S_plan. They are attributes dominated by common-class and common-meter priors in the Slakh corpus. High absolute accuracy is not evidence. The zero-feature gap is the only meaningful diagnostic here.

RQ3 receives a qualified positive answer: S_plan encodes key and pitch-class structure well above the prior baseline. Meter and instrument micro-F1 are excluded until label-shuffle controls confirm feature-specific learning.

---

## Slide 17 — RQ3: Axis Safety

This table formalizes the claim boundary for each MIR axis. Above the separator: weak key, pitch-class, and instrument macro-F1, where S_plan exceeds the zero-feature baseline. These are the paper-safe claims, always reported with the baseline gap, not as standalone numbers. Below the separator: instrument micro-F1 and meter accuracy, excluded entirely because the zero-feature baseline exceeds S_plan.

Two controls — label-shuffle and target-prior — are pending. The probing claims are provisional until those pass.

---

## Slide 18 — RQ3: Qualitative

As a qualitative illustration, this slide shows the piano roll comparison for window 3 of Slakh track 01501, covering 90 to 120 seconds. Ground-truth MIDI is on the left; the pitch-region reconstruction from the S_plan codec is on the right. Binary accuracy is 1.000, MSE is 0.00130. The symbolic structure is reconstructed faithfully at this resolution.

---

## Slide 19 — Design Constraint

The positive RQ1 and RQ2 results depend critically on the stream-native interface. Two preliminary comparisons demonstrate that naive alternatives fail, and these failures are not incidental engineering obstacles — they are reproducible under controlled conditions.

Side-channel insertion: inserting S_plan tokens as additional context rows through the released UniAudio 2.0 interface produced test CE of 5.511 for aligned, 5.512 for shuffled, and 5.510 for dummy — all three worse than reason-only at 5.360, and statistically indistinguishable from one another. The model ignored symbolic content when it was injected as an external side-channel incompatible with the model's multi-stream positional encoding.

LoRA adapter with gating: adding LoRA adapters stabilized training to validation CE 5.412, but the gate converged to 3.36×10⁻⁴, near-zero. Training stability is not fusion evidence.

Both results isolate the same root cause. Symbolic tokens must enter through the same stream-native positional encoding as R_a and C_a. These two failures motivated the stream-native design and establish it as a necessary constraint, not merely an engineering preference.

---

## Slide 20 — Limitations

Five limitations must be stated precisely.

First, no decoded generation evidence. All experiments evaluate cross-entropy on C_a token prediction under oracle symbolic input. There is no waveform-level evidence yet that S_plan conditioning changes the perceptual quality or structural fidelity of decoded audio.

Second, oracle symbolic input. S_plan tokens are derived from ground-truth MIDI files. A practical inference-time system requires either audio-to-MIDI transcription or a separate audio-to-S_plan predictor, neither of which is trained.

Third, no LLM-supervised tokenizer. S_plan is trained via reconstruction and MIR objectives only, not via an LLM decoder objective. Text-language grounding — music captioning, symbolic QA — is outside the current scope.

Fourth, pending MIR controls. Label-shuffle and target-prior controls for probing are incomplete. Instrument macro-F1 and several other axes cannot be claimed as robust symbolic evidence until those controls pass.

Fifth, Slakh-only scope. All experiments use a synthesized paired audio-MIDI corpus. Generalization to real-world recordings is outside the current scope.

---

## Slide 21 — Conclusion

The paper's central claim is conservative but supported. S_plan is a compact symbolic codec with acoustic-token utility and recoverable music-structural content under controlled Slakh experiments.

For RQ1: aligned S_plan reduces C_a cross-entropy in three independent controlled comparisons at different training scales. Most rigorously in the matched four-condition evaluation, test CE is 5.600 versus 5.859 for reason-only.

For RQ2: aligned consistently beats both shuffled and dummy controls. The alignment-specific effect is in the range of −0.014 to −0.034 CE across all three comparisons, positive and consistent in sign, corroborated by gate values that open more for aligned content.

For RQ3: MIR probing recovers weak key accuracy +0.431 and pitch-class +0.024 above the zero-feature baseline. Meter and instrument micro-F1 are excluded as prior-dominated.

Two negative results establish stream-native interface as a necessary design constraint.

The central contribution is methodological as much as empirical. Credible symbolic-fusion claims require multi-condition evaluation with matched corrupted controls, zero-feature MIR baselines, and honest reporting of what each comparison can and cannot support. This evaluation framework applies broadly to any future work claiming symbolic grounding in audio language models.

Thank you.
