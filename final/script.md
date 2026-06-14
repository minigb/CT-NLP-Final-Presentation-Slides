# Presentation Script
## S_plan: Toward Symbolic-Aware Music Language Models
### CT-NLP Final Presentation — June 2026

---

## Slide 1 — Title (~20 sec)

Good afternoon, everyone. I'm Minhee, presenting on behalf of our team.

Audio language models hear timbre, texture, acoustic energy well. What they don't hear — reliably — is musical structure. Key, meter, phrase boundaries — those live in MIDI, not in waveforms. We asked: can we inject that symbolic knowledge into a language model in a way rigorous enough to actually verify?

---

## Slide 2 — Problem (~30 sec)

The gap is on the slide. The model handles timbre and loudness well enough, but key, exact instrument identity, meter, phrase structure — essentially invisible to audio codecs.

The real challenge isn't building the bridge. It's designing an evaluation that distinguishes *symbolic content* from *extra token capacity*. That distinction is what this entire project is organized around.

---

## Slide 3 — Three Research Questions (~35 sec)

So we gave ourselves three tests, designed to be hard to pass.

RQ1: does an aligned symbolic stream help more than a corrupted version of itself? RQ2: is the gain specifically from *alignment* — meaning aligned has to beat both shuffled and dummy simultaneously? RQ3: does the stream actually encode musical structure we can independently verify through probing?

Every experiment in this project is evidence for one of these three.

---

## Slide 4 — Approach (~35 sec)

UniAudio 2.0 uses two sequential streams — R_a for reasoning, then C_a for acoustics. We insert S_plan right between them. It's a compact RVQ codec trained on MIDI-derived features: pitch-class, instrument presence, meter, density. The total sequence fits within UniAudio's context limit with zero truncation.

Why not just use raw MIDI? It blows the context immediately — and you can't uniformly shuffle or dummy-replace a variable-length sequence. S_plan's fixed-length design is what makes the controlled comparison possible.

---

## Slide 5 — Evaluation Protocol (~40 sec)

Four conditions, used in every comparison. Aligned: correct MIDI, correctly timed. Shuffled: different window's MIDI — statistics intact, alignment broken. Dummy: zero or random codes — pure capacity, no content. Reason-only: no S_plan slot at all.

The claim rule is strict: aligned has to beat both shuffled and dummy. Shuffled tests whether token statistics alone explain the gain. Dummy tests whether just having more tokens explains it. If aligned only beats one, we have no clean claim.

---

## Slide 6 — Prerequisite: Codec Table (~25 sec)

Before any downstream result means anything, the codec has to work. Binary feature accuracy is essentially perfect, MSE is minimal, and codebook utilization reaches 72 to 81% — no collapse.

If the codec were noisy, any difference downstream could be encoding artifacts, not symbolic content. These numbers rule that out.

---

## Slide 7 — Prerequisite: Feature Heatmap (~15 sec)

Here's what that looks like. Ground truth on the left, S_plan reconstruction on the right. I couldn't tell them apart when we first plotted this. The codec is not losing information. One boundary: this is symbolic feature reconstruction, not MIDI transcription or waveform generation.

---

## Slide 8 — RQ1: Proxy Task (~30 sec)

Does aligned S_plan actually reduce acoustic prediction error? First test: frozen codec, lightweight head. Look at the table — aligned beats both shuffled and dummy. The gap versus shuffled shows that temporal alignment is driving it, not just MIDI statistics in general.

Caveat: this is a lightweight proxy setup. Does it hold end-to-end?

---

## Slide 9 — RQ1: Stream-Native Replication (~25 sec)

So we ran it again — full integration, matched positional encoding, nothing frozen. Same direction. Aligned still wins, by a consistent margin.

That matters because "proxy with frozen codec" and "full end-to-end" are completely different setups. Same direction in both means this is replication, not coincidence.

---

## Slide 10 — RQ1: Primary Result (~50 sec)

Third test. Hardened. All four conditions, identical hyperparameters, same random seed. Only the stream variant changes.

Look at the table. Reason-only is at 5.859. Any symbolic stream at all — shuffled, dummy — drops to around 5.614. That 0.245 gap is the capacity effect: extra tokens help, regardless of content. Expected and welcome.

But aligned goes to 5.600 — another 0.014 below both controls. Small, but that's the number we're claiming: the alignment-specific signal, on top of the capacity gain.

And the gate value confirms it. The model opens the slot more for aligned than for shuffled or dummy. It's treating the three conditions differently.

---

## Slide 11 — RQ2: Alignment Decomposition (~40 sec)

The top table decomposes the total improvement: most of it is capacity, a small piece is alignment-specific. But the bottom table is the actual argument for RQ2.

Three completely independent comparisons — the gap is always negative, always in the same direction. The magnitude shrinks as training gets more constrained, which makes sense. But the sign never flips. One result could be noise. Three in the same direction rules out a single-run artifact. RQ2 is positive.

---

## Slide 12 — RQ3: MIR Probing (~45 sec)

For RQ3, we froze S_plan embeddings and trained probes to recover musical attributes. The key thing here is the zero-feature control — a probe trained on nothing, just the label distribution. That's the bar. If you can't beat it, your numbers mean nothing.

Weak key has a gap of over 0.4 — that's real signal. Pitch-class and instrument macro-F1 also beat the control.

But look at the red rows. Instrument micro-F1 and meter accuracy are *lower* than the zero-feature control. Those aren't S_plan failures — those are cases where common instruments and common meters so dominate the dataset that the prior alone wins. High accuracy is not evidence. The zero-feature comparison is mandatory.

---

## Slide 13 — RQ3: Axis Safety (~25 sec)

So what can we actually claim? We drew a hard line. Above it: key, pitch-class, instrument macro-F1 — where we beat the zero-feature baseline. Below it: micro-F1 and meter — excluded entirely.

And we're explicit that two controls are still pending before the claims are final. This is what we can defensibly say now.

---

## Slide 14 — RQ3: Qualitative Figure (~15 sec)

One more thing — here's what S_plan actually captures. Ground truth MIDI on the left, S_plan reconstruction on the right. They're essentially identical. The symbolic content is preserved at this resolution.

---

## Slide 15 — Design Constraint (~45 sec)

We didn't get here on the first try. Two failures.

First: we appended S_plan tokens as a side-channel, outside the native streaming position. All three conditions — aligned, shuffled, dummy — got *worse* than reason-only, and were indistinguishable from each other. The model learned to ignore that slot.

Second: a cross-attention adapter. The gate converged to essentially zero. Stable training, zero fusion.

Same root cause both times: symbolic tokens have to enter through the same positional encoding structure as the other streams. Without that, the model can't integrate them. This is why the positive RQ1 and RQ2 results are non-trivial — we had to find the right interface before any positive signal was possible.

---

## Slide 16 — Limitations (~30 sec)

Five gaps, quickly. We've never decoded audio — everything is on token CE, no perceptual comparison yet. We use oracle MIDI at test time, which doesn't exist for real recordings. All results are on Slakh only. Two MIR probing controls are pending. And text-language grounding is not trained.

The most urgent next step is decoding waveforms from aligned versus shuffled checkpoints — converting the token-level claim into something you can actually hear.

---

## Slide 17 — Conclusion (~40 sec)

Three questions, three answers. RQ1: aligned S_plan reduces acoustic CE versus both corrupted controls, consistent across all three setups. RQ2: the gain is alignment-specific — three independent runs, same direction. RQ3: qualified yes — key and pitch-class confirmed, meter and micro-F1 excluded.

But the contribution I'd point to isn't any single number. It's the evaluation framework — aligned, shuffled, dummy, plus zero-feature probing. That protocol is what forced us toward a non-trivial result, and it's reusable for any kind of symbolic injection beyond S_plan.

Thank you.

---

*Target: ~10 minutes*
