# Presentation Script
## S_plan: Toward Symbolic-Aware Music Language Models
### CT-NLP Final Presentation — June 2026

---

## Slide 1 — Title

Good afternoon, everyone. I'm Minhee, and I'm presenting on behalf of Dabin, Jiaxian, Aner, and myself.

Think about what a music language model actually hears. It hears timbre, texture, acoustic energy. What it doesn't hear — reliably — is what key a piece is in, how the chord progression moves, where the phrase boundaries are, or even which instrument is playing which stem. Those things live in MIDI. Not in waveforms.

So the question we asked ourselves was: can we take that symbolic MIDI knowledge and actually inject it into an audio language model — in a way that's controlled, verifiable, and rigorous enough to publish? That's what this project is.

---

## Slide 2 — Problem

And the gap is real. The model handles timbre, loudness, rough tempo well enough. But key signature, exact instrument identity, precise meter, phrase structure — those are essentially invisible to audio codecs. MIDI captures all of them exactly.

Now here's the thing. You could just throw symbolic tokens into the context and see if numbers go up. But that would prove nothing. Extra tokens alone could explain any gain — the model has more to work with. So the real challenge isn't building the system. It's designing an evaluation that can actually distinguish *content* from *capacity*. That distinction is what this entire project is organized around.

---

## Slide 3 — Three Research Questions

So we gave ourselves three tests. And they're designed to be hard to pass.

RQ1: does an aligned symbolic stream actually reduce acoustic prediction error — *more than* a shuffled or dummy stream of the same size? Not just "does adding something help" — more than a corrupted version of itself.

RQ2: is the gain specifically from alignment? Aligned has to beat both shuffled and dummy simultaneously. If it only beats one, we can't isolate the cause.

RQ3: does the symbolic stream actually encode musical structure we can independently verify? Not just "the loss went down" — can we probe for it?

These three questions are the skeleton of the paper. Every experiment is evidence for one of them.

---

## Slide 4 — Approach

UniAudio 2.0 works by predicting two sequential token streams. First R_a — the reasoning stream, 8 codebooks, 151 frames. Then C_a — the acoustic stream, 8 codebooks, 376 frames. We insert S_plan right between them. Four codebooks, 30 frames — one frame per second of audio.

S_plan is trained as an RVQ codec on MIDI-derived features: pitch-class distribution, instrument presence, meter, note density. All compact, all fixed-length. The total sequence becomes 557 tokens — well within the 1,024 limit, zero truncation.

Why not just feed raw MIDI? Raw MIDI is anywhere from 2,000 to 8,000 tokens per window. It blows the context immediately. And more importantly — you can't uniformly shuffle or dummy-replace a variable-length sequence. Our evaluation protocol *requires* fixed-length. S_plan's design is what makes the controlled comparison possible in the first place.

---

## Slide 5 — Evaluation Protocol

Here's the core of the evaluation design. Four conditions, used in every single comparison.

Aligned: the S_plan slot gets tokens from the *same* audio window's MIDI — temporally correct, content correct. Shuffled: tokens from a *different* window — same statistics, alignment broken. Dummy: zeros or random codes — pure capacity, no content. Reason-only: no S_plan slot at all.

The claim rule is strict. Aligned has to beat *both* shuffled and dummy. Shuffled tells us whether token statistics alone explain the gain. Dummy tells us whether just having more tokens explains it. If aligned only beats one, or neither, there's no claim to make. That's the bar we set for ourselves.

---

## Slide 6 — Prerequisite: Codec Table

Before we can trust anything downstream, we need to know: does the codec actually work? Binary feature accuracy is 0.998 — essentially perfect. Continuous feature MSE is 0.00172. And look at codebook utilization: books 2 through 4 at 72 to 81%. No collapse.

This matters because if the codec were noisy, any downstream difference between aligned and shuffled could come from encoding artifacts, not from symbolic content. With these numbers, that concern is gone. We're not chasing noise.

---

## Slide 7 — Prerequisite: Feature Heatmap

And here's what that looks like. Two heatmaps: ground truth on the left, S_plan reconstruction on the right. Rows are feature dimensions, columns are 30 time frames. Binary accuracy on this window: 1.000.

You can stare at them — I couldn't tell them apart either when we first plotted this. That's the point. The codec is not losing information.

One thing to be clear about: this is symbolic feature reconstruction, not MIDI transcription or waveform synthesis. S_plan encodes the compact representation. That's the boundary.

---

## Slide 8 — RQ1: Proxy Task

So does aligned S_plan actually help predict acoustic tokens? The first test is a proxy task — codec frozen, lightweight prediction head, five thousand steps.

Aligned gets 7.596. Shuffled gets 7.631. Dummy gets 7.694. Aligned wins both. And the gap versus shuffled — 0.034 — tells you that temporal alignment is driving it. It's not just that MIDI statistics are useful. It has to be *this window's* MIDI.

But I want to be honest: this is a lightweight setup. Frozen codec, minimal integration. Is the effect real when we go end-to-end? That's the next question.

---

## Slide 9 — RQ1: Stream-Native Replication

So we ran it again. Full integration this time — the actual R_a → S_plan → C_a layout, matched positional encoding, nothing frozen.

Aligned: 5.766. Shuffled: 5.785. Dummy: 5.785. Gap of −0.018 and −0.019.

The result holds. And that matters. Because "proxy task with frozen codec" and "full end-to-end integration" are two completely different setups. Same direction in both. That's replication, not coincidence.

---

## Slide 10 — RQ1: Primary Result

Third test. This is the hardened one. All four conditions, identical hyperparameters, same random seed. The only thing that changes is what goes into the S_plan slot.

Reason-only is at 5.859. Now add any symbolic stream at all — shuffled, dummy — and you drop to around 5.614. That 0.245 improvement is the capacity effect. The model simply benefits from having more tokens. Expected, and welcome.

But aligned goes to 5.600. Another 0.014 below both controls. That's the number we're claiming — the alignment-specific signal, on top of the capacity gain.

And look at the gate values. The model opens the S_plan slot more when the content is aligned — 0.177 — than when it's shuffled — 0.167 — or dummy — 0.163. The model is not treating these three identically. It knows the difference.

---

## Slide 11 — RQ2: Alignment Decomposition

Now — is that 0.014 actually meaningful, or noise from one run?

The top table decomposes the effect. The capacity gain — any symbolic stream versus reason-only — is 0.245. The alignment-specific gain on top — aligned versus both controls — is 0.014. Small. But look at the second table.

Three independent comparisons: −0.034 in the proxy task, −0.018 in the stream-native warmup, −0.014 in the hardened test. Three setups, same direction, every time. The magnitude decreases as the training becomes more constrained — which makes sense — but the sign never flips.

That consistency is the argument. One result could be noise. Three results in the same direction, across different architectures and training regimes, is a pattern. RQ2 is positive.

---

## Slide 12 — RQ3: MIR Probing

For RQ3 we ask a different question: what's actually *inside* those S_plan tokens?

We froze the embeddings and trained supervised probes to recover musical attributes. But here's the critical thing: we didn't just compare against a naive baseline. We used a zero-feature control — a probe trained on literally nothing, just the label distribution. That tells you what accuracy you get from prior alone, with zero learned representation. That's the bar.

Weak key accuracy: S_plan gets 0.557, zero-feature gets 0.126. Gap of 0.431. That's real signal. Pitch-class macro-F1: 0.854 versus 0.830. Some signal. Instrument macro-F1: 0.501 versus 0.426.

Now look at the red rows. Instrument micro-F1 is *lower* for S_plan than for the zero-feature control. Meter accuracy too. These aren't S_plan failures — they're cases where common instruments and common meter signatures dominate the dataset so completely that a probe with zero features can exploit the prior. High accuracy is not evidence. The zero-feature comparison is mandatory.

---

## Slide 13 — RQ3: Axis Safety

So what can we actually claim? We drew a hard line.

Above the line: weak key with a gap of 0.431 — our strongest result, safe to claim. Pitch-class at 0.024 — positive but we need one more control. Instrument macro-F1 at 0.075 — some signal, pending class-imbalance analysis.

Below the line: instrument micro-F1 and meter accuracy — excluded. No amount of qualification makes a negative gap claimable.

And we say explicitly on this slide that two controls are still pending. We're not presenting these as complete probing results — we're presenting what we can defensibly say now.

---

## Slide 14 — RQ3: Qualitative Figure

One more thing about what S_plan actually captures. Left is the ground-truth MIDI piano roll. Right is the pitch-region reconstruction from the S_plan codec. Ninety to 120 seconds of a Slakh track. Binary accuracy: 1.000.

The two images are essentially identical. The symbolic content is preserved at this resolution.

Again — boundary: this is feature reconstruction. S_plan is not transcribing MIDI from audio, and it's not generating waveforms. It's encoding and recovering the compact representation.

---

## Slide 15 — Design Constraint

Now — we didn't arrive at this design on the first try. We failed twice, and I want to tell you about those failures because they're actually what makes the positive results meaningful.

First attempt: we appended S_plan tokens as a side-channel — outside the native streaming position. Look at the table. Every condition got *worse* than reason-only. Aligned, shuffled, dummy — all roughly the same, all degraded. The model learned to ignore that slot entirely. Inserting tokens somewhere they don't belong, with mismatched positional encoding, just adds noise.

Second attempt: a cross-attention adapter. The idea was to graft in symbolic information via attention. The gate converged to 0.00034. Training was stable — but the fusion never happened. The adapter was technically active and practically useless.

Same root cause both times: symbolic tokens have to enter through the *same positional encoding structure* as R_a and C_a. That's what "stream-native" means. Without that, the model has no way to integrate the information. This is why RQ1 and RQ2 are non-trivial — not because the final numbers are large, but because getting any positive signal required getting the interface exactly right first.

---

## Slide 16 — Limitations

Let me be explicit about what we cannot claim.

The most important gap: we have never decoded audio. Every result you saw today is on semantic token cross-entropy. We don't know if aligned sounds better than shuffled. That's the next experiment — decode C_a from both checkpoints, run a perceptual comparison.

We also rely on oracle MIDI. At test time on real recordings, the aligned MIDI doesn't exist. We need an audio-to-S_plan predictor to close that loop, and we don't have one.

All of this is on Slakh — synthetic, MIDI-rendered music. Real recordings are a different world. We have no evidence of transfer.

And on the probing side, two controls are still pending. The claims there are provisional until we run them.

---

## Slide 17 — Conclusion

Three questions, three answers.

RQ1: yes, aligned S_plan reduces acoustic token CE relative to both corrupted controls. 5.600 versus 5.859. Consistent direction across all three comparison setups.

RQ2: yes, the gain is specifically from alignment, not token budget. The gap is small — 0.014 — but it holds across three independent runs. And the gate values confirm the model is treating aligned tokens differently.

RQ3: qualified yes. Weak key and pitch-class are above the zero-feature baseline. Meter and micro-F1 are excluded.

But what I think the actual contribution here is — beyond the individual numbers — is the evaluation framework. Aligned versus shuffled versus dummy, combined with zero-feature probing, is a protocol for asking precise questions about symbolic injection. That design is what forced us toward a non-trivial positive result, and it's reusable for anything you'd want to inject into a language model, not just S_plan.

Thank you.

---

*Total estimated time: ~12–14 minutes*
