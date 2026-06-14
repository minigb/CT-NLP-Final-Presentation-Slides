# Symbolic-Aware LALM Demo

This is a static companion site for the experiment-progress presentation.

Recommended public URL after GitHub Pages is enabled for the slide repository:

```text
https://dabinkim0.github.io/symbolic-fused-lalm-experiments/demo/
```

Sections:

1. UniAudio 2.0 Audio / Speech Tasks.
2. UniAudio 2.0 Music Tasks.
3. Symbolic-Aware MIDI / Music Tasks.

The first section is backed by a new runnable compatibility scaffold in
`source/demo`. It covers Text-to-Speech, Text-to-Sound, and Text-Instructed TTS
using the English prompts from the public UniAudio2.0 demo. Audio-Instructed TTS
is excluded for now. For these non-music tasks, the current page only presents
UniAudio2.0 baseline generations. A claim that symbolic augmentation preserves
speech/audio quality requires a trained Symbolic-Aware generation checkpoint and
paired inference on the same prompts. The current demo page includes 8 generated
UniAudio2.0 baseline wavs copied from
`source/demo/outputs/uniaudio2_compat_20260612_173602/`, with Symbolic-Aware
comparison slots intentionally left as `-`.

The preservation readiness audit is generated with:

```bash
python source/demo/audit_generation_preservation_readiness.py
```

It writes `source/demo/outputs/preservation_readiness_audit.json` and `.md`.
The current audit says the baseline prompt/decode setup is ready, but the
Symbolic-Aware decoded generation checkpoint is still missing.

For the music-task block, the page now separates mechanistic token/stream
evidence from direct UniAudio2.0 improvement claims.  EXP025 and EXP029 should
be read as "why the symbolic stream should help", not as downstream task
improvements.  The direct music-claim hardening path is EXP034A/EXP034B: matched
no-symbolic or open-gate baselines versus aligned/shuffled/dummy `S_plan` stream
variants under the same update budget.

```bash
cd /home/dabinkim_pt/symbolic-fused-lalm
CUDA_DEVICE=0 SESSION=exp034_music_claim_hardening \
  MAX_STEPS=1000 LIMIT_VALID=512 LIMIT_TEST=512 \
  bash source/exp/exp034_run_music_claim_hardening_tmux.sh
```

If the tmux session disappears immediately, inspect:

```bash
tail -n 120 exp_outputs/060528_uniaudio2-understanding-eval/exp034_music_claim_hardening/matched_lora_stream_s1000_seed260634/logs/exp034A_tmux.log
```

The launcher now keeps the shell open on exit when `KEEP_OPEN=1`, and supports
`DRY_RUN=1` for checking the wrapper and batch-shape path before a full run.

After the tmux run finishes:

```bash
python source/exp/exp034_summarize_music_claim_hardening.py \
  --run-root exp_outputs/060528_uniaudio2-understanding-eval/exp034_music_claim_hardening/matched_lora_stream_s1000_seed260634
```

The third section exposes symbolic MIDI/music task boxes.  The completed
symbolic downstream-style task is MIR attribute recovery: given `S_plan`, recover
instrument set, key, meter, tempo, and density from a 30 s Slakh window.  MIDI-to
Audio Generation and Audio-to-MIDI Transcription remain future slots.

The code-organization plan for attaching EXP034B results and exporting
EXP025/EXP029/EXP024 visuals is:

```text
doc/plan/260613_demo_evidence_ingestion_code_plan.md
```

The page intentionally distinguishes `S_plan` from `S_LLM`:

- `S_plan`: current learned symbolic planning token.
- `S_LLM`: planned LLM-supervised symbolic tokenizer/token stream, not trained yet.

Inference provenance:

- EXP006 frozen UniAudio2 sanity rows are kept as compatibility placeholders;
  the current demo page does not claim them as symbolic contributions.
- EXP024 symbolic tokenizer reconstruction uses a validation fixed example from
  the learned `S_plan` RVQ autoencoder. Its piano-roll comparison includes the
  GT MIDI piano roll and a feature-derived reconstructed proxy roll; it is not
  generated MIDI. Additional fixed-window gallery slots are kept even when their
  figures are not yet exported.
- EXP025 acoustic-token utility is shown as the bridge experiment between
  reconstruction and UniAudio-style stream control. It reports aggregate
  semantic-token CE/PPL only, not decoded waveform generation.
- EXP001 UniAudio2 music QA diagnostic uses a Slakh test-split QA sample and
  held-out Slakh QA aggregate metrics.
- EXP029 stream-control reports held-out test aggregate metrics; its currently
  saved per-sample demo card uses validation fixed-example artifacts and now
  separates strong-margin from weak-margin examples.
- EXP031A MIR recovery reports held-out test aggregate metrics for the
  symbolic-aware MIDI/music task family; its currently saved per-sample demo
  cards use validation fixed-example artifacts. EXP032 five-case spectrum rows
  are included with `-` placeholders where media exports are missing.
- Output MIDI piano-roll comparison is not claimed yet because EXP031A outputs
  MIR JSON/caption proxies rather than generated MIDI.

All playable audio clips are copied from project-local Slakh artifacts to avoid
shipping AudioCaps/YouTube-derived audio assets. Piano-roll figures are rendered
from the corresponding project-local Slakh MIDI files with:

```bash
python doc/slide/demo/build_pianorolls.py
```

EXP024 symbolic tokenizer reconstruction figures are rendered with:

```bash
python doc/slide/demo/build_splan_reconstruction_figures.py
```
