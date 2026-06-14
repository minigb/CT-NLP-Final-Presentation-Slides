# Presentation Script
## S_plan: Toward Symbolic-Aware Music Language Models
### CT-NLP Final Presentation — June 2026

---

## Slide 1 — Title (~20 sec)

Good afternoon, everyone. This work is presented on behalf of Dabin Kim, Minhee Lee, Ji Jiaxian, and Aner Zheng.

Audio language models handle timbre and texture well. What they do not reliably encode, however, is musical structure — key signatures, meter, phrase boundaries. These attributes live in MIDI, not in waveforms. The question this work asks is: can we inject that symbolic knowledge into a language model in a way that is controlled and verifiable?

---

## Slide 2 — Problem (~30 sec)

The table on this slide captures the gap directly. Audio tokens encode timbre and loudness well, but key, instrument identity, meter, and phrase structure are essentially invisible to acoustic codecs.

The challenge, however, is not building the bridge between symbolic and acoustic representations. The challenge is evaluating it rigorously. Any extra tokens added to a context can improve a model's predictions. The core question is whether the gain comes from symbolic *content*, or merely from *capacity* — having more tokens to work with. This distinction is what the entire evaluation protocol is built around.

---

## Slide 3 — Three Research Questions (~35 sec)

This gives us three research questions, each designed to be hard to pass.

RQ1 asks whether an aligned symbolic stream reduces acoustic prediction error more than a shuffled or dummy stream of equal size. RQ2 asks whether the gain is specifically due to temporal alignment — requiring aligned to outperform *both* shuffled and dummy simultaneously. RQ3 asks whether the symbolic stream encodes music-structural attributes that can be independently verified through probing.

Every experiment in this work is evidence for one of these three questions.

---

## Slide 4 — Approach (~35 sec)

S_plan is inserted as a third stream between the reasoning and acoustic streams of UniAudio 2.0. It is a compact RVQ codec trained on MIDI-derived features — pitch-class distribution, instrument presence, meter, and note density — producing a fixed-length symbolic representation per audio window. The total sequence fits within the model's context limit with zero truncation.

Raw MIDI was not used for two reasons: it saturates the context with thousands of tokens, and its variable length makes uniform shuffling and dummy replacement impossible. The fixed-length design of S_plan is precisely what enables the controlled comparison protocol.

---

## Slide 5 — Evaluation Protocol (~40 sec)

Every comparison in this work uses four conditions. Aligned S_plan provides tokens from the correct audio window's MIDI — symbolically accurate and temporally aligned. Shuffled S_plan provides tokens from a different window — statistics are preserved, but alignment is broken. The dummy stream contains zero or random codes, isolating the pure capacity effect. Reason-only is the base model with no S_plan slot.

The claim rule is strict. Aligned must outperform *both* shuffled and dummy. Shuffled tests whether token statistics alone explain the gain. Dummy tests whether extra tokens alone explain it. If aligned beats only one of the two controls, no clean attribution can be made.

---

## Slide 6 — Prerequisite: Codec (~25 sec)

Before any downstream result can be trusted, the codec itself must be verified. Binary feature accuracy is essentially perfect, continuous feature MSE is minimal, and codebook utilization reaches up to 81% with no collapse.

This is a prerequisite, not a result. Codec noise would contaminate every subsequent comparison — any difference between aligned and shuffled could reflect encoding artifacts rather than symbolic content. These numbers establish that concern is unfounded.

---

## Slide 7 — Prerequisite: Heatmap (~15 sec)

The heatmap makes this concrete. Ground-truth MIDI features are on the left; the S_plan reconstruction is on the right. The two panels are visually indistinguishable. The codec preserves what it encodes. One important boundary: this is symbolic feature reconstruction, not MIDI transcription or waveform generation.

---

## Slide 8 — RQ1: Proxy Task (~30 sec)

The first evidence for RQ1 comes from a proxy prediction task — frozen codec, lightweight prediction head. As shown in the table, aligned S_plan outperforms both the shuffled and dummy conditions. The gap against shuffled specifically demonstrates that temporal alignment is driving the improvement, not simply the presence of MIDI-derived token statistics.

One caveat: this is a proxy setup. The codec is frozen and the head is lightweight. Whether the effect holds in full end-to-end integration is tested next.

---

## Slide 9 — RQ1: Stream-Native (~25 sec)

The full stream-native interface replicates the result. With complete R_a → S_plan → C_a integration and matched positional encoding, aligned continues to outperform both controls by a consistent margin.

This matters because a frozen-codec proxy and a full end-to-end system are architecturally distinct setups. The same directional result in both constitutes replication, not coincidence.

---

## Slide 10 — RQ1: Primary Result (~50 sec)

The primary and most rigorous test is the hardened four-condition comparison. All conditions share identical hyperparameters and a fixed random seed — the only variable is which stream fills the S_plan slot.

The result can be read in two layers. Any symbolic stream at all — shuffled or dummy — reduces CE by roughly 0.245 relative to reason-only. That is the capacity effect: the model benefits from having more tokens, regardless of their content. Aligned goes further still, by an additional 0.014 below both controls. That is the alignment-specific signal being claimed.

The gate values corroborate this reading. The model opens the S_plan slot measurably more for aligned content than for shuffled or dummy. It is not treating the three conditions identically.

---

## Slide 11 — RQ2: Alignment Decomposition (~40 sec)

The top table decomposes the total improvement into its two components: the majority is capacity, and a small but consistent piece is alignment-specific.

The bottom table is the argument for RQ2. Three completely independent comparisons yield the same directional result. The magnitude decreases as training becomes more constrained, which is expected — but the sign never reverses. One result could be attributed to noise. Three results in the same direction, across different training regimes and architectures, rule out a single-run artifact.

RQ2 is answered positively.

---

## Slide 12 — RQ3: MIR Probing (~45 sec)

For RQ3, supervised probes are trained on frozen S_plan embeddings to recover music-structural attributes. The critical reference column is the zero-feature control — a probe trained on an all-zeros input, which captures accuracy achievable from label prior alone, with no learned representation. Any claim of structural encoding must clear that bar.

Weak key beats the control by 0.431. Pitch-class and instrument macro-F1 also show positive gaps.

The red rows require a different interpretation. Instrument micro-F1 and meter accuracy fall *below* the zero-feature control. These are not failures of S_plan — they are attributes so dominated by class-frequency priors in the Slakh dataset that even a blank probe performs well. High accuracy alone is not evidence. The zero-feature gap is the only meaningful metric here.

---

## Slide 13 — RQ3: Axis Safety (~25 sec)

The table draws a hard line between what can and cannot be claimed. Above the separator: weak key, pitch-class, and instrument macro-F1, where S_plan clears the zero-feature baseline. Below it: instrument micro-F1 and meter accuracy, excluded entirely.

Two additional controls remain pending. These claims are provisional pending those results.

---

## Slide 14 — RQ3: Qualitative (~15 sec)

As a qualitative illustration, the piano roll comparison shows ground-truth MIDI on the left and the S_plan pitch-region reconstruction on the right. They are essentially identical. The symbolic content is preserved at this resolution.

---

## Slide 15 — Design Constraint (~45 sec)

The positive RQ1 and RQ2 results did not emerge from the first design. Two prior approaches failed, and understanding why is part of the contribution.

In the first attempt, S_plan tokens were inserted as a side-channel outside the native streaming position. All three conditions — aligned, shuffled, and dummy — degraded below the reason-only baseline and became indistinguishable from one another. The model learned to ignore that slot.

In the second attempt, a cross-attention adapter was used to fuse symbolic information. The gate converged to near-zero. Training was stable; fusion never occurred.

The root cause in both cases is the same: symbolic tokens must enter through the same positional encoding structure as the other streams. Without that, the model has no coherent way to integrate them. This is precisely what makes the final positive results non-trivial — the right interface had to be discovered before any useful signal could emerge.

---

## Slide 16 — Limitations (~30 sec)

Five gaps must be stated clearly. No waveform has been decoded — all results are on semantic token cross-entropy. Oracle MIDI is assumed at inference, which does not exist for real audio. All experiments are conducted on Slakh only. Two MIR probing controls are pending. Text-language grounding has not been trained.

The most immediate next step is decoding C_a from the aligned and shuffled checkpoints and conducting a perceptual comparison — converting a token-level claim into something audibly verifiable.

---

## Slide 17 — Conclusion (~40 sec)

Three research questions, three answers. RQ1: aligned S_plan reduces acoustic CE against both corrupted controls, consistently across all three experimental setups. RQ2: the gain is alignment-specific — three independent comparisons, same direction. RQ3: a qualified positive — key and pitch-class confirmed against the zero-feature baseline, meter and micro-F1 excluded.

Beyond the individual results, the broader contribution is the evaluation framework itself. The aligned / shuffled / dummy protocol, combined with zero-feature probing, provides a principled method for attributing gains to symbolic content rather than capacity. This protocol is not specific to S_plan — it applies to any form of structured information injection into a language model.

Thank you.

---

*Target: ~10 minutes*
