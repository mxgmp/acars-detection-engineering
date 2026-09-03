# ACARS Detection Engineering

An exploratory aviation cybersecurity project testing whether timing between ACARS records can provide a useful anomaly signal, and examining what happens when the measurement unit does not consistently represent a logical message.

The project started with a simple question:

> Can message timing alone, independent of message content, surface behaviour worth an analyst's attention?

The answer turned out to be more interesting than a list of anomalies.

The analysis went through several iterations: an initial baseline that mixed different message cadences, a scoring inconsistency discovered after the methodology was corrected, and finally a protocol-level unit-of-analysis issue identified through `msgNum` and `blockId`.

## The Question

Can message timing alone, independent of message content, surface behaviour worth an analyst's attention?

The goal is not to classify a message as malicious from timing alone. The timing signal is treated as an exploratory first-layer indicator that would need contextual and protocol-aware validation before being used for detection or triage.

## Dataset

The analysis uses publicly available VDL2/ACARS data from the VDLiq feeder network.

The sample contains decoded aviation datalink records including:

- timestamps
- aircraft identifiers
- flight information
- ACARS labels
- message and block identifiers
- message content

Raw data is intentionally excluded from this repository.

## Method

### Initial Timing Feature

The first version measured the time since the previous decoded record for the same aircraft.

This created a problem: different ACARS labels can occur at different natural cadences, so combining them into one baseline can distort the resulting timing signal.

### Same-Label Timing

The feature was refined to measure the interval between consecutive records for the same:

- aircraft registration
- flight
- ACARS label

A robust baseline was then built using the median interval and Median Absolute Deviation (MAD).

The exploratory score is:

```text
absolute_deviation = |observed interval - median interval|
mad_score = absolute_deviation / MAD
```

This is a raw MAD-based deviation score, not a modified z-score.

### Scoring Consistency Check

After the timing feature was corrected, the downstream score calculation was still referencing the earlier registration-level baseline columns.

The baseline columns were corrected at their source so that deviation scoring, candidate summaries, and visualisations all used the same group-level baseline.

## Protocol Structure and Unit of Analysis

The analysis then examined the raw `msgNum` and `blockId` fields.

This exposed an important limitation: a decoded dataset record is not consistently equivalent to an independent logical ACARS message.

For example:

| Stream | Raw records | Unique `msgNum` | Span |
|---|---:|---:|---:|
| N261BZ / MX0263 | 15 | 2 | 36.25 s |
| N341NB / NW1638 | 18 | 18 | 142.08 s |
| N635FR / F92582 | 14 | 4 | 36.42 s |

N261BZ and N635FR therefore contain substantially fewer unique message numbers than raw records. Their row-level timing cannot be interpreted as logical-message cadence.

This is the main methodological finding of the project:

> A statistically valid timing score is only useful when the feature is defined on the correct protocol-level unit.

## Exploratory Results

The corrected timing analysis produced large deviations from local baselines in several streams.

However, after the protocol structure was examined, those scores were treated as exploratory decoded-record timing signals rather than validated evidence of anomalous aircraft behaviour.

The project therefore does not claim that the identified records represent malicious activity, spoofing, interference, or compromise.

## Limitations

This is a small exploratory sample with limited observation windows.

The current implementation does not perform logical ACARS message reassembly, so decoded-record timing cannot yet be treated as a validated behavioural cadence signal.

A next iteration would reconstruct logical messages using protocol-aware fields such as `msgNum` and `blockId`, validate the reconstruction against known multi-block sequences, and then measure inter-message timing.

Further validation would also require larger datasets, longer observation periods, and known benign and malicious examples.

## Reproducibility

The main analysis is contained in:

`01_acars_exploration.ipynb`

Install the Python dependencies with:

```bash
pip install -r requirements.txt
```

Raw data is excluded from this repository.

## Data Provenance

The source dataset is the publicly available VDL2/ACARS dataset from the vdliq feeder network.

Dataset licensing and attribution requirements should be followed when obtaining or redistributing the upstream data. This repository does not redistribute the raw dataset.

## Project Status

**Exploratory analysis complete.**

The current project demonstrates the development and validation process for a timing-based ACARS detection feature, including methodology correction, pipeline consistency checking, and identification of a protocol-level unit-of-analysis limitation.

The next technical step is protocol-aware logical-message reconstruction before treating timing as a behavioural detection feature.