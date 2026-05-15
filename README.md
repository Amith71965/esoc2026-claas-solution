# ESoC 2026 challenge - CLASS - Embedded AI for Predictive Sensor Systems in Agriculture 4.0

This repository contains trial datasets for the CLAAS "Embedded AI for Predictive Sensor Systems in Agriculture 4.0" challenge at European Summer of Code 2026, and suggested trial tasks.

For Q&A with experts from CLAAS and mentors from GC.OS, you can use the [GitHub issue tracker](https://github.com/european-summer-of-code/esoc2026-challenge-claas/issues) in the repository (monitored during the ESoC 2026 application period).


## Trial datasets

### Description of data

#### Data origin

The data were created by CLAAS in field experiments with agricultural machinery. They were simplified and obfuscated for the purpose of the ESoC 2026 challenge.

The field experiments are described in the following section,
in a substantially simplified form that corresponds to the simplified data as shared.

The files were shared by CLAAS in early May 2026.

#### Experimental design

In the field experiments, a harvester drives through a field.
The relevant parts of the harvester for the experiment are:

* the engine of the harvester, which has a certain speed
* the header at the front of the harvester, which takes up the plants, cuts and chops them.
  The header can be turned on or off. When turned on, it can additionally be set to
  different cut lengths (which correlates with uptake speed).
* two sensors within the header, a microphone and a metal detector.
  The microphone records an audio signal, and the metal detector records a voltage,
  which goes up if metal is present in the header.

The data was acquired under real world conditions,
for the purpose of measuring how the sensors behave if stones enter the header.

This data should be used to develop a model for early detection of stones in the
header, so the engine can be switched off.

The experimental protocol is divided into five runs, with multiple experimental
episodes.

* in each run, the harvester drives through the field in representative
  real world conditions, across multiple episodes.
* For each episode, the header is turned on to harvest (plants on the actual field).
* Within the episode, it is randomized unformily (with an undisclosed probability)
  whether a metallic stone will be artificially inserted into the header. 
* If the stone is inserted, it is expected that it will be detected by the metal
  detector, and also recorded by the microphone. The metal detector has an inbuilt
  trigger to shut down the header shortly after a spike.
* It may also happen that a stone already on the field, i.e., one not artificially
  inserted, is ingested by the harvester. In this case, the header may shut down
  once the stone blocks the mechanism, or the stone is small and will be mixed
  into the harvest.
* Once the header shuts down, or after a certain time elapses (if there was no stone),
  the episode ends.

The experimental question is whether it is possible to detect stones early
by using the audio signal, and with which accuracy, as measured by:

* average advance time of detection compared to the metal detector signal (if the stone were metallic)
* true positive rate
* false detection rate per time unit in representative real world operation of the header

For the purpose of the experiment, you can consider:

* the runs to be representative of overall operations
* the episodes to be representative of episodes within a run
* the data as a general inspiration for a "real" experiment
  where you might have more similar data,
  e.g., 100s of runs and dozens of episodes within a run.

#### Files

The trial dataset can be found [here](https://github.com/european-summer-of-code/esoc2026-challenge-claas/tree/main/data).

The trial dataset consists of:

* five `mf4` (MDF4 format) files corresponding to 5 runs of the harvester,
  each containing potentially multiple episodes
* five `wav` files for illustration of the audio signal also contained in the `mf4` files.
  This is illustration in the sense that the `wav` makes it easy to listen to,
  but the data is already contained int he `mf4` files as one of the channels.

The `mf4` files are in standard MDF4 format, each contains five time series channels:

* `Sensor1`: audio recordings by the microphone in the header, amplitude; float
* `VehicleSpeed`: speed of the harvester; float
* `CutLength`: cut length of the header; float, correlates with header uptake speed
* `VoltageSignal`: metal detector signal, float; spike = detection of metal in the header 
* `Status`: whether the header is switche on or off, boolean (1=on, 0=off)

#### Challenge

An AI model should be able to detect stone uptake in the header early, as measured by

* average advance time of detection compared to the metal detector signal (if the stone were metallic)
* true positive rate
* false detection rate per time unit in representative real world operation of the header

The AI should detect not only metallic stones, but also non-metallic stones,
assuming that non-metallic stones cause a signal similar to metallic stones except
for being undetectable via the metal detector.

## Trial tasks

### 1. reading the data

Write python code that reads the data into structures that can be consumed by
common ML / AI packages such as `sktime`, `scikit-learn`, `torch` (at least one).

Include tests.

### 2. building a model

Write some python code which, using any AI model, does the following:

given a live signal, detect stones early.

Showcase your code on some use cases, evaluate your model appropriately on the data provided, and include tests.

Bonus 1: write a generative model to create more similar data. Test your approach on 100s of runs with dozens of episodes each.

Bonus 2: pretend the model needs to fit on an automotive microcontroller with 2 MB RAM,
inference mode only. Keep all the models you have developed (they can be larger), but
make a choice what you would put on the microcontroller, and produce corresponding artefacts.

Bonus 3: where components (e.g., data preprocessors, metrics) are not available in `sktime`, open issues of things that you would like to see in `sktime` to tackle ths problem -
or contribute them to `sktime` as an appropriate object type.

### Bonus: towards a pre-prototype

Using the example data, and possibly code from the above tasks, prepare a showcase how you would approach the general problem.

This should be an indicative pre-prototype only for the purpose of explaining your general approach. Please do not spend too much time on this - especially on graphical user interfaces or productionization.

(you can publish your code under a permissive license if you want)


## License

The contents of this repository, including but not restricted to the challenge datasets,
are not public domain. Please do not distribute further.

By accessing or using any data, documents, code, or other materials contained in this private repository, you acknowledge and agree that all content is confidential and proprietary. The materials are provided solely for the purpose of evaluating applications for European Summer of Code. You may not copy, share, distribute, publish, disclose, or use any content from this repository for any purpose outside of the application or interview process without prior written permission. Unauthorized use or disclosure is strictly prohibited.
