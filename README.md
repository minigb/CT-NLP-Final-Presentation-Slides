# Experiment Progress Slides

Open `exp_progress_slides.html` in a browser to present the current experiment journey, or use `main.tex` as the Overleaf entrypoint for the Beamer version.

The HTML deck is a standalone file with inline CSS and no external dependencies. The Beamer deck is Overleaf-oriented and should be compiled with XeLaTeX when Helvetica Neue is available. Both versions use a minimal Helvetica Neue-style layout and mark missing or not-yet-proven academic links in red text.

An interactive demo companion is available under `demo/`. It is designed to be published through GitHub Pages so the Beamer/Overleaf deck can link to playable audio examples:

- local path: `demo/index.html`
- intended public URL: https://dabinkim0.github.io/symbolic-fused-lalm-experiments/demo/
- source style reference: https://dongchaoyang.top/UniAudio2Demo/

The demo is split into three blocks:

- existing UniAudio behavior to preserve,
- existing music downstream proxy improvement,
- new symbolic-related music downstream tasks.

It also keeps the notation boundary explicit: `S_plan` is the current learned symbolic planning token, while `S_LLM` is the planned LLM-supervised symbolic tokenizer/token stream and has not been trained yet.

The Beamer deck uses a research-journey format for each major stage:

- `EXP Objective`: why the experiment was run.
- `Results`: the key numerical or qualitative outcome.
- `Discussion`: how to interpret the result academically.
- `Next Step`: why the following experiment was designed.

Yellow boxes mark the main takeaway that should be emphasized during team discussion.

The Beamer deck also includes a multi-slide `Metrics` section covering:

- completed music-related output diagnostics,
- token-level CE/PPL causal proxy metrics,
- MIR-structured symbolic recovery metrics,
- missing downstream metrics that still require experiments.

Red text in the metrics section means the evaluation is not yet sufficient for a paper claim and should be treated as a required future experiment.

The opening slides explicitly ground the experiment design in UniAudio2.0:

- Paper: Yang et al., "UniAudio 2.0: A Unified Audio Language Model with Text-Aligned Factorized Audio Tokenization," arXiv:2602.04683, 2026.
- Paper URL: https://arxiv.org/abs/2602.04683
- Demo URL: https://dongchaoyang.top/UniAudio2Demo/
- Code URL: https://github.com/yangdongchao/UniAudio2
- Checkpoint URL: https://huggingface.co/Dongchao/UniAudio2_ckpt

For HTML PDF export:

1. Open `exp_progress_slides.html` in Chrome or a Chromium browser.
2. Print with landscape orientation.
3. Enable background graphics.
4. Save as PDF.

For local demo preview:

1. From this directory, run `python -m http.server 8081`.
2. Open `http://localhost:8081/demo/`.
3. Check that the Slakh audio controls play in all three demo blocks.

For Overleaf PDF export:

1. Upload or import this repository into Overleaf.
2. Set the compiler to XeLaTeX.
3. Set `main.tex` as the main file.
4. Compile and export the generated PDF.

Local note: this machine currently does not have `xelatex`, `lualatex`, or `pdflatex` installed, so the Beamer PDF was not compiled locally.

Primary source documents:

- `doc/summary/260612_exp_journey_key_records.md`
- `doc/plan/260612_overleaf_reconstruction_guideline.md`
- `doc/plan/260527_project-plan.md`
