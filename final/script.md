# Presentation Script
## S_plan: Compact Learned Symbolic Planning Tokens for Music-Aware Audio Language Models
### CT-NLP Final Presentation — June 2026

---

## Slide 1 — Title

Good afternoon, everyone. This is joint work from Dabin Kim, Minhee Lee, Ji Jiaxian, and Aner Zheng at KAIST.

---

## Slide 2 — Problem

Unified audio language models predict discrete audio tokens autoregressively and have achieved strong performance across diverse tasks. Music, however, the structural properties like harmonic progressions, rhythmic regularity, instrument co-occurrence, phrase boundaries, are encoded in MIDI symbolic representations, not waveform statistics.

So we propose to inject a compact symbolic stream into the LALM context, aligned to the model's positional encoding, to test whether it can help the model use musical structure. 


---

## Slide 3 — Three Research Questions

Research questions are about music-aware understanding, alignment-specific use of symbolic content, and music-structural understanding. The metric evidence is primarily cross-entropy on acoustic token prediction under oracle symbolic input, with MIR probing as a secondary metric for music-structural understanding.

<!-- 
We organize this work around three research questions.

RQ1 is the high-level question about music-aware understanding: does a compact symbolic stream, aligned to an audio window's own MIDI, help the model use musical structure? The metric evidence is narrower: lower acoustic-token CE under aligned S_plan compared with shuffled and dummy controls. So we separate the claim from the measurement: understanding is the goal, and CE gain is the initial controlled evidence.

RQ2 asks whether the apparent understanding gain is alignment-specific. The conceptual claim is that correct symbolic-audio alignment matters, not just extra token capacity. The metric evidence is narrower: aligned S_plan must beat both shuffled and dummy under matched conditions. Shuffled preserves token statistics while breaking temporal alignment. Dummy removes all symbolic content. Aligned must clear both bars.

RQ3 asks whether S_plan supports music-structural understanding. Here the metric evidence comes from MIR probing: can frozen S_plan embeddings recover musically meaningful attributes above a zero-feature baseline that exposes dataset priors? -->

---

## Slide 4 — Approach

The proposed method is like this. The existing UniAudio pipeline already has a text tokenizer, a reasoning codec, audio understanding and generation experts, and reconstruction tokens for audio output.

So we add a symbolic tokenizer, S_plan. It takes MIDI-derived musical features, compresses them with a compact RVQ codec, and inserts the resulting symbolic tokens between the reasoning tokens and the acoustic tokens.
<!-- The total layout fits within the 1,024 context limit with zero truncation. -->

<!-- We use this compact fixed-length codec instead of raw MIDI because raw MIDI is too long and irregular for uniform controls. -->

---

## Slide 5 — Dataset: Slakh2100

All experiments use Slakh2100, a multi-track paired audio-MIDI dataset synthesized with professional-grade sample-based virtual instruments.
<!-- The dataset is split at the track level before windowing, so no musical phrase can appear in both train and test. The training split contains 1,289 tracks and 11,279 thirty-second windows; the held-out test set contains 151 tracks and 1,385 windows. All cross-entropy results in this talk are evaluated on that held-out test set. -->

---

## Slide 6 — MIDI Feature Extraction

From each 30-second window, MIDI features are extracted at 100 ms resolution from the ground-truth MIDI file, not transcribed from audio. The binary feature group — 41 dimensions — captures instrument-class presence across 34 Slakh classes, pitch-class activation across 12 semitones, and beat/downbeat phase. The continuous feature group — 12 dimensions — captures instantaneous tempo, note density, polyphony count, and an instrument-weighted energy proxy. Together, this gives a 53-dimensional frame vector normalized per dimension across the training split, yielding approximately 300 frames per window. The codec then compresses those approximately 300 frames to a fixed 30-frame representation, or one symbolic frame per second of audio.

---

## Slide 7 — Codec Architecture

The symbolic RVQ codec compresses the 53-dimensional, 300-frame input into a fixed 30-frame representation. A two-layer 1D CNN encoder with temporal pooling handles the variable-to-fixed compression. The residual vector quantizer applies four stages, each with 512 codes, trained with exponential moving average updates and codebook restart. A symmetric CNN decoder reconstructs the original 53-dimensional feature sequence. Training uses binary cross-entropy for the binary dimensions and MSE for continuous dimensions, run for 42,000 steps. The resulting 4-by-30 token stream uses the same multi-stream RVQ format as C_a — which is precisely what enables the stream-native positional encoding.

---

## Slide 8 — Training Setup

All symbolic stream variants are trained via LoRA adapters on 140 attention modules of the frozen UniAudio 2.0 transformer. The reason-only configuration has approximately 10.8 million trainable parameters; adding the symbolic stream brings this to approximately 253 million. A learned scalar gate controls the magnitude of symbolic stream influence on the C_a prediction path.

The hardened four-condition comparison — the primary result — uses 800 steps, batch size 1 with gradient accumulation 16, LoRA learning rate 10⁻⁵, symbolic stream learning rate 3×10⁻⁴, and gate learning rate 2×10⁻³. The random seed is fixed, and all four conditions are trained identically except for the stream variant.

---

## Slide 9 — Evaluation Protocol

Every controlled comparison uses the same four conditions under matched hyperparameters: aligned S_plan, shuffled S_plan, dummy stream, and reason-only.

---

## Slide 10 — RQ1: Music-Aware Understanding / Proxy Task

The first evidence for RQ1 comes from a proxy prediction task. A small transformer head is trained to predict C_a tokens from frozen R_a plus symbolic representations, over 5,000 steps on 1,385 held-out test windows.

Aligned S_plan achieves the lowest test cross-entropy in this table. This is not a direct understanding metric; it is an initial performance signal. Under the corrupted controls, lower CE suggests the model benefits from aligned symbolic musical content, rather than from the extra token slot alone. The important comparison is that aligned beats both corrupted controls, and also beats the hand-crafted rule summary.

At this point, this is still a proxy setup with a frozen codec. The full end-to-end integration is tested separately.

---

## Slide 11 — RQ1: Music-Aware Understanding / Stream-Native

The full three-stream layout — R_a → S_plan → C_a — is then evaluated with streams aligned to the base model's positional encoding, under a shorter warmup budget of 3,000 steps.

Aligned again gives the lowest test CE, with a small but consistent edge over both shuffled and dummy. This keeps the metric statement concrete: the CE improves. The broader interpretation is that aligned symbolic structure is helping the model in a direction consistent with better music-aware understanding. On the fixed validation windows, some cases are clearly positive and some are near-zero or slightly negative, which is what we would expect from a real but still small signal under a short warmup budget.

The positive gap replicates in a setup that is architecturally distinct from the proxy task. The effect is not confined to the proxy setup. It also fits within the model context without truncation, which matters because the goal is not just to add symbolic information, but to add it in a usable stream-native form.

---

## Slide 12 — RQ1: Music-Aware Understanding / Primary Result

The primary evaluation runs all four conditions under fully identical hyperparameters, differing only in the symbolic stream variant.

The result decomposes into two layers. First, any symbolic stream helps over reason-only; that is the stream-capacity effect. Second, aligned S_plan is still lower than both shuffled and dummy; that smaller residual gap is the alignment-specific effect.

The learned gate corroborates through an independent channel: the model opens the S_plan slot more for aligned content than for shuffled or dummy. These results give positive initial evidence for RQ1 across three independent controlled comparisons at different training scales. The achievement language is music-aware understanding; the metric evidence is lower CE, with broader understanding tests and waveform-level evidence left as next steps.

---

## Slide 13 — RQ2: Alignment-Specific Understanding

RQ2 uses stronger goal language — alignment-specific understanding — but the evidence is still metric-based. We need to show that the CE gain is not explained by token capacity alone.

The decomposition in the upper table is clear. The dominant effect is stream capacity: simply adding a symbolic stream already helps over reason-only. The alignment-specific effect is smaller, but it is the part that remains when aligned is compared against shuffled and dummy.

The critical evidence is in the lower table. Across three independent comparisons, aligned is consistently better than shuffled. The magnitude changes with setup and training budget, but the direction is stable. A finding consistent in direction across independent setups is substantially more robust than a single-run result.

The gate provides independent corroboration: aligned content receives the strongest gate value, followed by shuffled, then dummy. So the achievement is alignment-specific understanding, and the metric evidence is a small but consistent CE advantage plus the gate ordering.

---

## Slide 14 — RQ3: Music-Structural Understanding / MIR Probing

For RQ3, the high-level question is music-structural understanding. The metric evidence is MIR probing. Linear probing heads are trained on frozen S_plan embeddings to recover music-structural attributes. The evaluation compares against a zero-feature baseline — inputs zeroed, with only the head bias and BatchNorm statistics available — which exposes dataset priors. The reliable axes should exceed this bar.

Weak key has the clearest gap over the zero-feature baseline, so it is the strongest feature-specific signal. Pitch-class also shows a positive gap. Instrument macro-F1 is positive too, but it still needs class-imbalance controls before we treat it as a strong result.

The excluded axes require a different reading. For instrument micro-F1 and meter accuracy, the zero-feature baseline is actually higher than S_plan. These are not failures of S_plan. They are attributes dominated by common-class and common-meter priors in the Slakh corpus. High absolute accuracy is not evidence; the comparison against the baseline is the meaningful diagnostic.

RQ3 receives a qualified positive answer: the probing metrics support key and pitch-class structure well above the prior baseline. Meter and instrument micro-F1 are excluded until additional controls are complete.

---

## Slide 15 — RQ3: Music-Structural Understanding / Supported Axes

This table summarizes the support by MIR axis. Weak key has the strongest support, pitch-class shows a positive feature-specific signal, and instrument macro-F1 is positive but sensitive to class imbalance. Instrument micro-F1 and meter accuracy are prior-dominated because the zero-feature baseline is higher.

The important habit is to report each axis with its zero-feature gap. Absolute accuracy alone is not enough for these probing results.

---

## Slide 16 — RQ3: Music-Structural Understanding / Qualitative

As a qualitative illustration, this slide shows a piano roll comparison. Ground-truth MIDI is on the left; the pitch-region reconstruction from the S_plan codec is on the right. The symbolic structure is reconstructed faithfully at this resolution. This is still symbolic feature reconstruction, not MIDI transcription or waveform generation.

---

## Slide 17 — Current Scope and Next Steps

Five limitations must be stated precisely.

First, no decoded generation evidence. All experiments evaluate cross-entropy on C_a token prediction under oracle symbolic input. There is no waveform-level evidence yet that S_plan conditioning changes the perceptual quality or structural fidelity of decoded audio.

Second, oracle symbolic input. S_plan tokens are derived from ground-truth MIDI files. A practical inference-time system requires either audio-to-MIDI transcription or a separate audio-to-S_plan predictor, neither of which is trained.

Third, no LLM-supervised tokenizer. S_plan is trained via reconstruction and MIR objectives only, not via an LLM decoder objective. Text-language grounding — music captioning, symbolic QA — is outside the current scope.

Fourth, MIR controls in progress. Label-shuffle and target-prior controls will strengthen the probing analysis. Instrument macro-F1 and several other axes need those controls before being used as headline symbolic evidence.

Fifth, Slakh-only scope. All experiments use a synthesized paired audio-MIDI corpus. Generalization to real-world recordings is outside the current scope.

---

## Slide 18 — Conclusion

The central result is focused but supported. S_plan is a compact symbolic codec that gives initial controlled evidence for music-aware understanding, alignment-specific use of symbolic content, and recoverable music-structural understanding under Slakh experiments.

For RQ1: aligned S_plan gives CE gains in three independent controlled comparisons at different training scales. The clearest evidence comes from the matched four-condition evaluation. This is initial evidence toward music-aware understanding, not a standalone understanding benchmark.

For RQ2: aligned consistently beats both shuffled and dummy controls in CE. The alignment-specific effect is small, but it is consistent in sign and corroborated by the gate opening more for aligned content. That is the metric evidence for alignment-specific understanding.

For RQ3: MIR probing shows the clearest feature-specific gain for weak key, with a smaller positive signal for pitch-class. That is the metric evidence for music-structural understanding. Meter and instrument micro-F1 are excluded as prior-dominated.

The contribution is methodological as much as empirical. Matched corrupted controls, zero-feature MIR baselines, and a clear separation between capacity effects and alignment-specific content make the symbolic-fusion evidence interpretable. This evaluation framework applies broadly to future work on symbolic grounding in audio language models.

Thank you.

---

## Appendix Slide 19 — Why Not Just Append Symbolic Tokens?

This backup slide explains why the symbolic stream has to be integrated as a native stream, rather than appended as an external side channel.

In the side-channel version, S_plan tokens were inserted as additional context rows through the released UniAudio 2.0 interface. That naive insertion made all symbolic variants worse than reason-only, and aligned, shuffled, and dummy were nearly indistinguishable. In other words, the model did not actually use the symbolic content when it was injected through an interface that did not match the model's multi-stream positional encoding.

A second preliminary adapter attempt also stabilized training without producing meaningful fusion: the gate stayed near zero. So the lesson is architectural, not just hyperparameter-level. S_plan must enter through the same stream-native R_a → S_plan → C_a interface used by the acoustic streams.

---

## Appendix Slide 20 — Symbolic Codec Reconstruction

Before any downstream result can be trusted, the RVQ codec must faithfully reconstruct input symbolic features. The table shows high binary accuracy, low continuous error, and healthy codebook use without collapse. In particular, the later codebooks show substantial utilization, so the result is not explained by codebook collapse. The fixed validation windows also reconstruct cleanly.

This validates the codec before downstream comparison. It means later differences are not easily explained by reconstruction failure.

---

## Appendix Slide 21 — Feature Heatmap

The heatmap shows a representative reconstruction. Rows are feature dimensions — instrument flags, pitch-class bits, and continuous values — and columns are one-second time frames. Ground-truth MIDI features are on the left; the S_plan reconstruction is on the right. The key point is that the reconstruction closely follows the target, so the codec produces a technically healthy symbolic bottleneck. This figure is about symbolic feature reconstruction; it is not MIDI transcription or waveform generation.
