# coreDAQ Python API — Programmer’s Manual

This document is the **official programmer’s manual** for the `coredaq_py_api` Python driver.  
It explains **all public functionality**, starting from a quick “getting started” guide and moving on to a detailed API reference with practical examples.

The API is designed for **photonic and optoelectronic measurements**, with emphasis on:
- calibrated optical power readout
- low-noise operation via oversampling
- gain control and autogain (LINEAR front end)
- single-shot snapshots and timer-based acquisitions
- optional external triggered start
- environmental monitoring (temperature & humidity)

---

## Table of Contents
- [Getting started](#getting-started)
- [Core concepts](#core-concepts)
  - [Front-end types](#front-end-types)
  - [Oversampling and sampling frequency](#oversampling-and-sampling-frequency)
  - [Gain stages (LINEAR)](#gain-stages-linear)
  - [Zeroing (LINEAR)](#zeroing-linear)
  - [LOG deadband (LOG)](#log-deadband-log)
- [API reference](#api-reference)
  - [Lifecycle](#lifecycle)
  - [Identity and device type](#identity-and-device-type)
  - [Port discovery](#port-discovery)
  - [Configuration](#configuration)
  - [Gain control](#gain-control)
  - [Zeroing](#zeroing)
  - [Single-shot measurements (snapshots)](#single-shot-measurements-snapshots)
  - [Acquisition control](#acquisition-control)
  - [Bulk data transfer](#bulk-data-transfer)
  - [Environmental sensors](#environmental-sensors)
  - [Utility conversions](#utility-conversions)
- [Examples](#examples)
  - [Single measurement](#single-measurement)
  - [Timer-based acquisition (free-running)](#timer-based-acquisition-free-running)
  - [Timer-based acquisition (external trigger)](#timer-based-acquisition-external-trigger)

---

## Getting Started

The coreDAQ Python API provides a simple, explicit interface for controlling the device, taking measurements, and transferring data for analysis.

---

### Requirements

The core driver depends only on standard scientific Python packages:

```bash
pip install pyserial numpy
```

## Connecting to the device and reading optical inputs

```python

from coredaq_py_api import CoreDAQ
import time


daq = CoreDAQ("/dev/tty.usbmodem2057396453331") # Set your CoreDAQ port here

print("Device:", daq.idn()) # Identification string

print("Device Frontend Type:", daq.frontend_type()) # Identify connected frontend variant

daq.set_gain1(0)  # CH1 gain index = 0
                  # Transimpedance / power range mapping:
                  #   G0 → ~1 kΩ   (≈ 4 mW max)
                  #   G1 → ~2 kΩ   (≈ 2 mW)
                  #   G2 → ~5 kΩ   (≈ 800 µW)
                  #   G3 → ~10 kΩ  (≈ 400 µW)
                  #   G4 → ~50 kΩ  (≈ 80 µW)
                  #   G5 → ~100 kΩ (≈ 40 µW)
                  #   G6 → ~1 MΩ   (≈ 4 µW)
                  #   G7 → ~10 MΩ  (≈ 400 nW)
                  # Higher index = higher transimpedance (more sensitivity, lower max power)

# Sampling rate
daq.set_freq(50_000)

# Measure a snapshot with average of 5 frames 
print("Snapshot 5 frames (mV):", daq.snapshot_mV(5)[0]) # Output in mV 

print("Gains : ", daq.snapshot_mV(5)[1]) # Returns current gain indices

print("Snapshot 5 frames (Watts):", daq.snapshot_W(5)) # Output in Watts

daq.close() # Close the connection cleanly

```



## Core Concepts

This section explains the key concepts behind the coreDAQ system and Python API.  
Understanding these ideas will help you configure measurements correctly and interpret results with confidence.

---

### Front-end Types

coreDAQ devices are available with two fundamentally different analog front ends.  
The front-end type is detected automatically at connection time and determines which features are available.

#### LINEAR Front End
- Photodiode readout via a **linear transimpedance amplifier (TIA)**
- Discrete, selectable gain stages
- Lowest noise and best absolute accuracy
- Ideal for precision measurements, low-noise experiments, and calibration work

Features:
- Gain switching (8 gain stages)
- Factory zero correction
- Software (soft) zeroing
- Optional autogain during measurements

Typical use cases:
- Low-noise optical power monitoring
- Ring-resonator sweeps
- Extinction ratio measurements
- Detector characterization

#### LOG Front End
- Logarithmic amplifier with LUT-based calibration
- Extremely large dynamic range
- No gain switching
- Offset handled via a configurable deadband

Features:
- No gain control
- No zero subtraction
- Voltage-to-power conversion via LUT
- Deadband to suppress low-level offset drift

Typical use cases:
- Very wide dynamic range measurements
- Power monitoring across many orders of magnitude
- Situations where absolute low-noise performance is less critical

You can query the front-end type at runtime:
```python
daq.frontend_type()   # "LINEAR" or "LOG"
```

## Oversampling and Sampling Frequency

coreDAQ uses digital oversampling to improve measurement quality by averaging multiple ADC conversions internally before producing one output sample.

Oversampling trades **bandwidth for signal-to-noise ratio (SNR)**:

- Higher oversampling → lower noise, better SNR
- Higher oversampling → lower maximum sampling frequency

The supported oversampling indices range from **OS 0 to OS 6**.

| Oversampling index | Maximum sampling frequency |
|-------------------:|---------------------------|
| OS 0 | 100 kS/s |
| OS 1 | 100 kS/s |
| OS 2 | 50 kS/s |
| OS 3 | 25 kS/s |
| OS 4 | 12.5 kS/s |
| OS 5 | 6.25 kS/s |
| OS 6 | 3.125 kS/s |

General guidelines:
- Use **OS 0–1** for fast, time-resolved signals
- Use **OS 3–6** for low-noise, quasi-static measurements
- For long-term monitoring, higher OS values are recommended

The API automatically enforces valid oversampling / frequency combinations.  
If an invalid combination is requested, it adjusts to a safe configuration and emits a warning.

Example:
```python
daq.set_oversampling(4)
daq.set_freq(10_000)
```

## Zeroing and Offset Handling

coreDAQ handles offsets differently depending on the front-end type.  
This section explains **how zeroing works for LINEAR and LOG front ends**, and what the user should expect in each case.

---

## Zeroing — LINEAR Front End

LINEAR front ends exhibit a small DC offset at the ADC input.  
This offset is primarily caused by:
- op-amp output swing limitations
- rail-to-rail behavior in a single-supply design

### Key Properties
- The offset is **independent of the gain stage**
- Zeroing is applied **per channel** (CH1–CH4)
- Zero correction is performed in **ADC codes**
- Zeroing is always applied automatically during measurements

---

### Factory Zero (Default)

Each device ships with **factory-measured zero offsets**, one per channel.

- Stored in device firmware
- Queried via `FACTORY_ZEROS?`
- Loaded automatically when the Python driver starts
- Used as the default zero correction

These factory zeros correct for:
- ADC offset
- analog front-end DC offset
- assembly-related tolerances

Example:
```python
# read currently active zero offsets (ADC codes)
print(daq.get_linear_zero_adc())#
```

