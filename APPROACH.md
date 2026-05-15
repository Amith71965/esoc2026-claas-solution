# What I've tried and where things stand

These are working notes more than a polished writeup. I'll keep updating this as the work progresses.

---

## Starting point

The dataset is five MF4 files — one per recording session — with five time series channels inside each:

- `Sensor1`: audio from a microphone in the header, at 44kHz
- `VoltageSignal`: metal detector reading, at 3kHz
- `VehicleSpeed`: 2Hz
- `CutLength`: header cut length setting, 10Hz
- `Status`: header On/Off, 10Hz

Total: 18 harvesting episodes across 5 runs, about 28 minutes of audio. That's the whole dataset.

Before doing anything with models, I spent time just reading the data carefully. A few things stood out immediately.

---

## Things I caught before modeling

**The Status channel isn't what the README says.** The README describes it as `boolean (1=on, 0=off)`. The actual values are `b'On'` and `b'Off'` — byte strings, dtype `|S3`. Any code that tries to read it as integers will break. I opened an issue with the CLAAS team about this.

**Run 1 (May 9) has no real stone events.** With the VoltageSignal threshold set to 2000, run 1 has zero detections. The max voltage in that run is 513. Either there were no metallic stones inserted in that session, or the stones were small enough that the signal stayed low. I'm treating run 1 as a clean baseline run.

**The episode detection logic had a bug.** The original approach looked for `Off → On` transitions to find episode starts. That works fine unless the recording starts while the header is already running — in which case the first episode is completely missed. Run 2 (May 14) has exactly this problem: the header is already On at t=0, so the first ~30 seconds of harvesting gets dropped. Fixed by seeding the episode detection with the initial status at t=0.

**A second bug: the voltage threshold was too low.** The first threshold I used (`median + 5 * std` of the whole run) produced a threshold around 370 for run 2. That run has a max voltage of 10,668. At 370, the code was flagging normal signal noise as stone events, generating 19 false labels outside any header-On period. Switched to a fixed threshold of 2000 with a sustain requirement (spike must stay above threshold for at least 5 consecutive samples), which cleanly separates the genuine stone events.

I raised the threshold question with the CLAAS team in [issue #2](https://github.com/european-summer-of-code/esoc2026-challenge-claas/issues/2) since I still don't know what their actual detector uses internally.

---

## Audio feature analysis

Before jumping to models I wanted to know if there's actually anything to detect. Does the audio 500ms before a VoltageSignal spike look any different from audio during normal harvesting?

I extracted:
- Stone windows: 500ms of audio before each confirmed voltage spike (10 total)
- Normal windows: random 600ms chunks from within the same On periods, at least 2 seconds away from any spike (63 total)

Then computed four features on the pre-spike audio:

| Feature | Stone median | Normal median | Ratio |
|---------|-------------|---------------|-------|
| Peak RMS (10ms windows) | 0.383 | 0.042 | 9.2x |
| Max kurtosis (20ms windows) | 1.006 | 0.462 | 2.2x |
| HF energy ratio (>4kHz) | 0.0019 | 0.0011 | 1.7x |
| MFCC temporal variance | 10.61 | 7.48 | 1.4x |

Peak RMS is the standout. Stone impacts are roughly 9x louder in short energy bursts than normal harvesting. That makes sense physically — a stone hitting a metal blade at speed creates a sudden, sharp amplitude transient.

The high-frequency ratio and MFCC variance are weaker. The spectral shape doesn't shift as dramatically as I expected, which suggests the stone impact energy is spread across frequencies rather than concentrated in a specific band.

---

## Models

I tried three things, all evaluated with leave-one-run-out cross-validation (hold out one run, train on the other four, repeat five times).

**ROCKET** (via sktime): Uses random convolutional kernels on the MFCC time series. Input shape `[73, 13, 43]` — 73 windows, 13 MFCC coefficients, 43 time frames. Result: TPR=0.00 across all folds. With only 8-9 training stone examples per fold and 43 time steps, the random kernels don't find anything consistent.

**TimeSeriesForest** (via sktime): Ensemble of 200 trees over random intervals, same MFCC input. Result: TPR=0.25 mean, 3.2 false alarms/hr. Gets some stone events right in the May runs but misses the October ones entirely.

**1D-CNN** (PyTorch): Raw audio input `[batch, 1, 26460]`, three strided conv layers, global average pooling, linear classifier. Data augmentation on training stone windows (time shifts, noise, amplitude scaling) to compensate for only having 10 examples. Result: TPR=1.00, 14.9 false alarms/hr.

I want to be honest about what those numbers mean. The CNN getting TPR=1.00 is not a reliable result. Each fold has 1-2 stone test windows. Getting "2 out of 2 correct" is two binary decisions — statistically meaningless. The 14.9 false alarms per hour is more informative and more concerning. In practice that means the detector would trigger roughly every 4 minutes on normal harvesting, which would make it unusable.

ROCKET scoring 0.00 also doesn't tell me ROCKET is bad — it tells me 10 training examples isn't enough for it to learn anything useful.

Fundamentally, 10 labeled stone events is not enough data to train or evaluate anything. The confidence intervals on every number here are enormous. You'd need at least 100 stone events before any metric starts to be meaningful.

---

## The quantization piece

The challenge asks about fitting the model on a microcontroller with 2MB RAM. I exported the trained CNN to ONNX and ran INT8 dynamic quantization via `onnxruntime`.

- FP32: 87KB
- INT8: 31KB
- Prediction agreement: 98.6%

Both easily fit in 2MB. There's also a decision tree fallback trained on just two features (peak RMS and max kurtosis) that serializes to 2KB, which would be the most practical MCU option if you can afford to run the feature extraction in firmware.

One note on the tooling: `torch.onnx.export` in PyTorch 2.12 uses a new export path by default that has a shape inference issue with `onnxruntime`'s quantizer. The fix is to run `quant_pre_process` from onnxruntime before calling `quantize_dynamic`. This isn't documented well anywhere — I found it by reading the error message carefully.

---

## What's next

The honest answer is that the models need more data before they mean anything. The natural next step is synthetic data generation — build a generative model for stone impact audio and use it to create 100+ runs with dozens of episodes each. The README (Bonus 1) explicitly calls this out.

That means:
1. Model what "normal harvesting audio" looks like (probably something like a short-time stationary Gaussian process conditioned on CutLength and VehicleSpeed)
2. Model what a stone impact sounds like (brief high-amplitude transient, plausibly from the confirmed metallic stone events we have)
3. Mix them together to generate synthetic episodes
4. Retrain and re-evaluate on the synthetic data at realistic scale

That's the work in progress.

---

*Last updated: May 2026*
