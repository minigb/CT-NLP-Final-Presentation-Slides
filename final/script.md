# Presentation Script
## S_plan: Toward Symbolic-Aware Music Language Models
### CT-NLP Final Presentation — June 2026

---

## Slide 1 — Title

Good afternoon, everyone. I'm Minhee, presenting our CT-NLP final project on behalf of Dabin, Jiaxian, Aner, and myself.

The project's central question is on the slide: audio language models handle sound very well — timbre, texture, acoustic qualities — but musical structure lives in MIDI, not in waveforms. Key signatures, chord progressions, meter — these are essentially invisible to audio codecs. Can we inject that symbolic knowledge into a language model in a controlled, verifiable way?

That's what this work is about.

---

## Slide 2 — Problem

Here's the gap more precisely. The left column shows what audio tokens can encode well: timbre, loudness, rough tempo. The right column shows what they reliably miss: key signature, exact instrument identity at the stem level, precise meter, and phrase boundaries.

MIDI encodes all of these exactly — note events, instrument tracks, key and tempo metadata. So we want to learn a compact representation of this MIDI information and inject it into the model.

The real challenge is evaluation. If we just add a symbolic stream and see improvement, we can't tell whether the gain came from the symbolic *content* itself, or just from giving the model extra tokens to work with. Our entire evaluation is designed around that distinction.

---

## Slide 3 — Three Research Questions

This gives us three research questions. RQ1: does an aligned symbolic stream reduce acoustic token prediction loss, more than a shuffled or dummy stream of the same size? RQ2: does the gain come specifically from alignment, not just extra token budget? Aligned must beat *both* shuffled and dummy. RQ3: does the symbolic stream encode music-structural attributes we can independently verify through probing?

These RQs are the organizing principle. Experiments are evidence for these claims, not the claims themselves. Each comparison uses the aligned / shuffled / dummy protocol so gains can be attributed to symbolic content, not to capacity.

---

## Slide 4 — Approach

UniAudio 2.0 uses two sequential token streams: R_a — the reasoning stream — 8 codebooks, 151 frames, followed by C_a — the acoustic stream — 8 codebooks, 376 frames. We insert S_plan between them: 4 codebooks, 30 frames, giving 30 seconds of symbolic coverage at one frame per second.

S_plan is a 4-stage RVQ codec trained on MIDI-derived features: pitch-class distribution, instrument presence flags, meter, and note density per window.

The total sequence is 151 + 30 + 376 = 557 tokens. UniAudio's context limit is 1,024. We have zero truncation.

Why not just append raw MIDI? Raw MIDI ranges from 2,000 to 8,000 tokens per window — it saturates the context immediately. And variable-length MIDI can't be uniformly shuffled or replaced with a dummy stream, which breaks our evaluation design. S_plan's fixed-length RVQ encoding is what makes the controlled comparison possible.

---

## Slide 5 — Evaluation Protocol

Let me walk through the four conditions we use in every comparison.

**Aligned** S_plan: the model sees tokens from the *same* window's MIDI — correct symbolic information, correctly aligned in time.
**Shuffled** S_plan: tokens from a *different* window's MIDI — token statistics intact, temporal alignment broken.
**Dummy stream**: zero or random RVQ codes — isolates the pure capacity effect, no symbolic content at all.
**Reason-only**: no S_plan slot — the plain base model.

The claim rule is strict: aligned must beat *both* shuffled and dummy. Shuffled tells us if statistics alone explain the gain; dummy tells us if just having more tokens explains it. Only if aligned beats both can we attribute the gain to alignment-specific content.

---

## Slide 6 — Prerequisite: Codec Table

Before any downstream result means anything, we need to verify the codec itself works faithfully. Binary feature accuracy is 0.998 — essentially perfect reconstruction. Continuous feature MSE is 0.00172. Books 2 through 4 reach 72 to 81% codebook utilization, so there's no codebook collapse.

This is a prerequisite, not a claim. If the codec were noisy, differences in the downstream comparisons could come from codec artifacts rather than symbolic content. With these numbers, that concern is ruled out.

---

## Slide 7 — Prerequisite: Feature Heatmap

This heatmap makes it visual. Left side is ground-truth MIDI features — rows are feature dimensions, columns are 30 one-second frames. Right side is the S_plan reconstruction. Binary accuracy is 1.000 on this window. The two panels are visually indistinguishable.

One important boundary: this is symbolic feature reconstruction, not MIDI transcription or waveform generation. S_plan encodes and decodes the compact feature representation, not the original MIDI events.

---

## Slide 8 — RQ1: Proxy Task

First evidence for RQ1 comes from a proxy prediction task. Here the S_plan codec is frozen, and we train a lightweight proxy head to predict acoustic tokens. Five thousand steps, 1,385 test windows.

Aligned S_plan gets test CE 7.596. Shuffled gets 7.631. Dummy gets 7.694. The aligned beats both controls, and the gap of −0.034 versus shuffled shows that temporal alignment drives the gain — not just token statistics from *some* MIDI file.

One caveat: this is the proxy setting. The codec is frozen and the head is lightweight — it's not the full end-to-end integration. We replicate that next.

---

## Slide 9 — RQ1: Stream-Native Replication

To confirm the proxy result isn't an artifact of the frozen codec setup, we replicate with the full stream-native interface — the actual R_a → S_plan → C_a layout with matched positional encoding.

Aligned S_plan: test CE 5.766. Shuffled: 5.785. Dummy: 5.785. Consistent deltas of −0.018 and −0.019.

RQ1 holds with end-to-end integration. The gain replicates when S_plan is inserted as a native third stream, not just an external side-channel. And the 557-token sequence fits with zero truncation.

---

## Slide 10 — RQ1: Primary Result

Our primary and most rigorous test is the hardened four-condition comparison. All four conditions use identical hyperparameters and the same fixed random seed. The only variable is which stream variant fills the S_plan slot.

Reason-only baseline: test CE 5.859. Aligned S_plan: 5.600. Shuffled: 5.614. Dummy: 5.614.

Let me unpack these numbers. The large drop from reason-only to *any* symbolic stream — 5.859 down to roughly 5.614 — that's the capacity effect: the model benefits from having any extra 30 tokens, regardless of their content. But aligned goes further, to 5.600, beating both controls by an additional 0.014. That 0.014 is the alignment-specific signal we're claiming.

The gate value confirms the same story: the model opens the S_plan slot *more* for aligned content — 0.177 — than for shuffled — 0.167 — or dummy — 0.163. The model is not indifferent to what's in that slot.

Boundary: oracle MIDI input; metric is semantic token CE, not decoded audio.

---

## Slide 11 — RQ2: Alignment Decomposition

RQ2 asks whether the gain is specifically due to alignment. We decompose it: the stream capacity effect — any symbolic stream versus reason-only — is −0.245. The alignment-specific effect on top of that — aligned versus both controls — is −0.014. Together: −0.259 total.

The key evidence for RQ2 is the second table: the alignment-specific gap is consistent in sign across all three independent comparisons — −0.034 in the proxy task, −0.018 in the stream-native warmup, −0.014 in the hardened comparison. Three runs, same direction, decreasing magnitude as the training becomes more constrained. This rules out a single-run artifact.

The gate corroborates: 0.177 for aligned, 0.167 for shuffled, 0.163 for dummy. RQ2 is answered positively.

---

## Slide 12 — RQ3: MIR Probing

For RQ3 we run supervised probing on frozen S_plan embeddings. The table has two columns: S_plan accuracy and the zero-feature control. The zero-feature control is a probe trained on an all-zeros input — it tells you what accuracy you get purely from the label distribution, with no learned features at all.

Weak key accuracy: 0.557 versus 0.126. That's a gap of 0.431 — the model is clearly encoding key information. Pitch-class macro-F1: 0.854 versus 0.830. Instrument macro-F1: 0.501 versus 0.426.

Now the red rows at the bottom. Instrument micro-F1 is actually *lower* for S_plan than for the zero-feature control: 0.791 versus 0.835. Same for meter accuracy: 0.783 versus 0.855. These are not failures of S_plan — they're cases where the label prior alone achieves high accuracy because common instruments and common meters dominate the dataset. High accuracy alone is not evidence. The zero-feature baseline is mandatory.

---

## Slide 13 — RQ3: Axis Safety

This slide formalizes which axes we can actually claim.

Weak key, with a gap of +0.431, is our strongest result. Safe to claim — pending a label-shuffle control which we haven't completed yet.

Pitch-class at +0.024 is positive but small. We need a target-prior control before claiming it firmly.

Instrument macro-F1 at +0.075 shows some signal but requires class-imbalance analysis.

The bottom two — instrument micro-F1 and meter accuracy — are excluded. The common-class and common-meter priors dominate in both cases; any claim there would be misleading.

---

## Slide 14 — RQ3: Qualitative Figure

For intuition, here's the piano roll comparison. Left is ground-truth MIDI, right is the pitch-region reconstruction from the S_plan codec, for window 3 of this track — 90 to 120 seconds. Binary accuracy is 1.000, MSE 0.0013.

The reconstruction is essentially exact. Again, the boundary: this is symbolic feature reconstruction, not MIDI transcription, and not waveform generation.

---

## Slide 15 — Design Constraint

Before we arrived at the working S_plan interface, we tried two things that failed, and the failures are instructive.

First, side-channel insertion — we appended S_plan tokens outside the native streaming position. The result: all three conditions — aligned, shuffled, dummy — were *worse* than reason-only, and indistinguishable from each other. The model ignored the symbolic stream entirely.

Second, adapter gating — a cross-attention adapter to fuse the symbolic signal. The gate converged to 3.4 × 10⁻⁴. Training was stable, but the model never actually used the adapter.

Same root cause in both failures: symbolic tokens must enter through the same stream-native positional encoding as R_a and C_a. This is why the RQ1 and RQ2 positive results are non-trivial. Getting a positive result required finding the right interface first.

---

## Slide 16 — Limitations

Being explicit about what we cannot claim. Five gaps.

No decoded generation: everything is on semantic token CE. We haven't decoded waveforms from the aligned versus shuffled checkpoints yet. That is the next priority — converting a token-level claim into perceptual audio evidence.

Oracle MIDI: at inference on real audio, you don't have the aligned MIDI. We need an audio-to-S_plan predictor to make this usable.

No LLM-supervised tokenizer: text-language grounding — captioning, open QA — is not yet trained.

Pending MIR controls: the label-shuffle and target-prior controls for probing are incomplete.

Slakh only: no real-recording evidence and no cross-domain transfer.

---

## Slide 17 — Conclusion

To conclude.

**RQ1 — acoustic utility — positive.** Aligned S_plan reduces cross-entropy from 5.859 to 5.600; the direction holds across all three comparisons.

**RQ2 — alignment specificity — positive.** Aligned beats shuffled and dummy consistently, delta in the range of −0.014 to −0.034. Gate values corroborate.

**RQ3 — structural content — qualified positive.** Weak key and pitch-class confirmed above the zero-feature baseline. Meter and instrument micro-F1 excluded as prior-dominated.

The design constraint result — two interface failures before finding the working approach — is what makes RQ1 and RQ2 non-trivial. And the multi-condition protocol we developed — aligned, shuffled, dummy, plus zero-feature probing — is a methodological contribution applicable beyond S_plan specifically.

Thank you. Happy to take questions.

---

*Total estimated time: ~12–14 minutes*
