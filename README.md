# ACARS Detection Engineering

### Exploratory validation of timing as an ACARS/VDL2 analysis feature

An exploratory aviation cybersecurity project testing whether timing between ACARS records can provide a useful anomaly signal, and examining what happens when the measurement unit does not consistently represent a logical message.

The project started with a simple question:

> Can timing between decoded ACARS records provide a stable and interpretable feature for anomaly analysis?

The answer turned out to be more interesting than a list of anomalies.

The analysis went through several iterations: an initial baseline that mixed different message cadences, a scoring inconsistency discovered after the methodology was corrected, and finally a protocol-level unit-of-analysis issue identified through `msgNum` and `blockId`.

## The Question

Can timing between decoded ACARS records provide a stable and interpretable feature for anomaly analysis?

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
timing_deviation_score = absolute_deviation / MAD
```

This is a raw MAD-based deviation score, not a modified z-score.

### Scoring Consistency Check

After the timing feature was corrected, the downstream score calculation was still referencing the earlier registration-level baseline columns.

The baseline columns were corrected at their source so that deviation scoring, candidate summaries, and visualisations all used the same group-level baseline.

## Protocol Structure and Unit of Analysis

The analysis then examined the raw `msgNum` and `blockId` fields.

This exposed an important limitation: a decoded dataset record is not consistently equivalent to an independent logical ACARS message.

For example:

| Stream | Label | Raw records | Unique `msgNum` | Span |
|---|---|---:|---:|---:|
| N261BZ / MX0263 | H1 | 15 | 2 | 36.25 s |
| N341NB / NW1638 | H1 | 16 | 16 | 136.11 s |
| N635FR / F92582 | H1 | 12 | 2 | 12.96 s |

Figures are grouped by registration, flight, and ACARS label, the same grouping used for scoring and for the notebook's Section 12 protocol-structure table; so these are per-label counts, not per-flight totals. N341NB/NW1638 and N635FR/F92582 each also carry a small number of records under other labels not shown here.

N261BZ and N635FR therefore contain substantially fewer unique message numbers than raw records. Their row-level timing cannot be interpreted as logical-message cadence.

This is the main methodological finding of the project:

> A statistically valid timing score is only useful when the feature is defined on the correct protocol-level unit.

## Exploratory Results

The corrected timing analysis produced large deviations from local baselines in several streams.

However, after the protocol structure was examined, those scores were treated as exploratory decoded-record timing signals rather than validated evidence of anomalous aircraft behaviour.

The project therefore does not claim that the identified records represent malicious activity, spoofing, interference, or compromise.

## Threat Model and Security Interpretation

This project is exploratory feature validation, not validated security detection. The timing deviation score demonstrates that a robust statistical baseline can be built and scored against decoded-record intervals; it does not demonstrate that deviations from that baseline are security-relevant.

No attacker behaviour is assumed to produce the observed deviations. The dataset was not chosen or filtered to contain known-malicious or known-benign examples, and the analysis does not model an adversary, an attack technique, or an expected effect on the data.

A security-relevant timing feature would need to derive from a concrete threat model and an expected protocol-level observable: a specific claim about what an attacker would need to do, and what that would look like in the decoded record stream. Timing becomes security-relevant when protocol behaviour depends on synchronisation, state transitions, sequencing, timeouts, or race conditions — none of which this notebook currently models.

## Limitations

This is a small exploratory sample with limited observation windows.

The current implementation does not perform logical ACARS message reassembly, so decoded-record timing cannot yet be treated as a validated behavioural cadence signal.

No threat model is defined for this feature. ACARS traffic may be event-driven or otherwise non-periodic, so deviation from a local timing baseline does not by itself establish security relevance.

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

The source dataset is the publicly available VDL2/ACARS dataset from the vdliq feeder network, published daily as Parquet files:

`https://github.com/Sky-Power-Services/vdliq-data/releases`

Dataset licensing and attribution requirements should be followed when obtaining or redistributing the upstream data (ODbL-1.0, attributed to the vdliq feeder network). This repository does not redistribute the raw dataset.

## Project Status

**Exploratory analysis complete.**

The current project demonstrates the development and validation process for a timing-based ACARS analysis feature, including methodology correction, pipeline consistency checking, and identification of a protocol-level unit-of-analysis limitation.

The next step for a detection-oriented iteration is a defined threat model and expected protocol-level observable, followed by logical-message reconstruction.