# ESoC 2026 — CLAAS Challenge: Stone Detection from Harvester Audio

This is my solution repo for the CLAAS "Embedded AI for Predictive Sensor Systems in Agriculture 4.0" challenge at European Summer of Code 2026.

The problem: harvesters ingest stones from the field. The metal detector on the header catches metallic ones, but by then the damage might already be happening. The question is whether the microphone already picked up the impact sound before the metal detector fired — and if so, can we build a model that catches it earlier. Non-metallic stones are the harder case, since there's no sensor for them at all.

---

## What's in this repo

```
Notebook_1_Initial_Investigation.ipynb   — first look at the MF4 files, channels, sample rates
Notebook_2_all_channels.ipynb            — all 5 channels plotted together, episode structure
Notebook_3_labeling.ipynb                — found two bugs in initial labeling, fixed them
Notebook_4_stone_audio_fingerprint.ipynb — audio feature analysis around confirmed stone events
model_training.ipynb                     — ROCKET, TimeSeriesForest, 1D-CNN + ONNX quantization
APPROACH.md                              — notes on what I tried and what I found
```

The MF4 data files are not included (they're confidential per the challenge terms and also ~250MB). You'll need access to the original challenge repo to get them, then drop them into a `data/` folder at the root.

---

## Setup

Tested on macOS with Python 3.14. Should work on 3.10+ without changes.

```bash
git clone https://github.com/Amith71965/esoc2026-claas-solution.git
cd esoc2026-claas-solution

# Create the virtual environment
python3 -m venv .venv
source .venv/bin/activate   # Windows: .venv\Scripts\activate

# Core dependencies
pip install asammdf numpy pandas scipy matplotlib jupyter ipykernel scikit-learn librosa

# PyTorch (CPU build — no GPU needed)
pip install torch --index-url https://download.pytorch.org/whl/cpu

# ONNX tooling (for the quantization cell in model_training.ipynb)
pip install onnx onnxruntime onnxscript

# sktime — the challenge suggests using it, install from PyPI like this:
pip install sktime[classification]

# Register the kernel so Jupyter picks it up
python -m ipykernel install --user --name=esoc2026-claas --display-name "ESoC 2026 CLAAS"
```

Then open any notebook in VS Code or JupyterLab and select the "ESoC 2026 CLAAS" kernel.

One thing worth knowing: sktime's `RocketClassifier` depends on numba, which has a version constraint against Python 3.14 in their pyproject.toml. But if numba is already installed, it works fine — the constraint just means pip won't pull it in automatically. Run `pip install numba` separately if ROCKET fails to import.

---

## Running the notebooks

Run them in order. Each one builds on the previous.

**Notebook 1** loads the MF4 files and does a basic sanity check — channel names, sample rates, data ranges. Nothing surprising except that the `Status` channel is stored as byte strings (`b'On'`/`b'Off'`), not booleans like the README says. I raised this as an issue with the CLAAS team.

**Notebook 2** plots all 5 channels together for each run. The main thing here is seeing the episode structure — when the header is On vs Off — and getting a feel for when voltage spikes happen relative to the status transitions.

**Notebook 3** is where I found two problems with the initial labeling logic and fixed them. Worth reading even if you skip straight to modeling, because those bugs directly affect training labels.

**Notebook 4** digs into the audio. Takes confirmed stone events (VoltageSignal > 2000 while header is On), extracts 500ms of audio before each spike, and computes four features: short-time RMS, high-frequency energy ratio, kurtosis, and MFCC variance. Peak RMS separates stone from normal at 9x median ratio, which is the most useful single feature.

**model_training.ipynb** trains three models on those features, evaluates with leave-one-run-out CV, then exports the CNN to ONNX and quantizes it to INT8. See `APPROACH.md` for honest notes on what the results actually mean.

---

## Data note

Put the MF4 files in `data/` at the root:

```
data/
  Messung_2025-05-09_08-59-34.mf4
  Messung_2025-05-14_16-02-23.mf4
  Messung_2025-05-20_16-30-26.mf4
  Messung_2025-10-01_09-42-16.mf4
  Messung_2025-10-01_17-18-12.mf4
```

The WAV files are optional — they're just the audio channel exported separately so you can listen to them in Audacity or any audio player. Everything in the notebooks reads from the MF4 files.

---

## Dependencies summary

| Package | Why |
|---------|-----|
| asammdf | Reading MF4 files |
| librosa | MFCC extraction, spectrograms |
| scipy | Kurtosis computation |
| sktime | ROCKET and TimeSeriesForest classifiers |
| torch | 1D-CNN training |
| onnx / onnxruntime / onnxscript | Model export and INT8 quantization |
| scikit-learn | Oversampling, decision tree fallback |

---

*Part of my application for ESoC 2026. Active discussion with the CLAAS/GC.OS team on the [challenge issue tracker](https://github.com/european-summer-of-code/esoc2026-challenge-claas/issues).*
