# Blind Digital Communication Receiver using GNU Radio
 
**Digital Communications**                                                                                                                                                                                                                                                                                                                                                                                                 
**Technologies:** GNU Radio, MATLAB
 
## Overview
 
This project implements a **configurable, blind receiver** in GNU Radio that decodes an unknown complex baseband IQ signal, given only the sampling rate. No prior knowledge of the symbol rate, carrier frequency offset, or modulation scheme is assumed all of these are estimated directly from the received waveform, and the receiver chain is then configured accordingly to recover the transmitted bitstream.
 
The full pipeline moves from raw IQ samples all the way to decoded symbols:
 

*Unknown IQ Signal → Parameter Estimation → CFO Correction → RRC Filtering
→ Gardner Timing Synchronization → Constellation Decoding → Bitstream Output*

 
## Problem Statement
 
Given a captured complex baseband signal and its sampling rate only, the goal is to:
1. Estimate the symbol rate and samples per symbol (SPS).
2. Estimate and correct the carrier frequency offset (CFO).
3. Identify the modulation scheme from the constellation.
4. Recover the transmitted symbol/bit stream.
## Approach
 
### 1. Symbol Rate & SPS Estimation
The symbol rate is extracted using a **squaring / autocorrelation-style spectral method**:
 
- Compute ***x(t) · x*(t) → |x(t)|²**, which removes the carrier phase and frequency information from the signal.
- The periodicity that remains in the resulting spectrum directly reveals the symbol rate, seen as a spectral peak.
- For the reference signal used in this project: sampling rate = 2.048 MHz, estimated symbol rate ≈ 256 kHz → **SPS = 2048k / 256k = 8**.
This analysis was done in GNU Radio, using the squaring/spectral method described above on the captured IQ data.
 
### 2. Carrier Frequency Offset (CFO) Estimation & Correction
 
**Method: Raise Signal to Power**
 
CFO is estimated using the **4th-power method**, implemented in GNU Radio using successive Multiply blocks to compute:
 
- x²(t) -> 2 multiplications
- x⁴(t) -> 4 multiplications
The resulting signal is then passed to a **Frequency Sink (FFT)** for spectral inspection.
 
**Observation:** Peaks were observed after 2 multiplications and again after 4 multiplications. The peak after 4 multiplications corresponds to:
 

f_peak = 4 × f_cfo

 
**Final Estimation:** The actual CFO is obtained as:
 

f_cfo = f_peak / 4 = 90 / 4 Hz ≈ 22.5 Hz

 
This was cross-checked in **MATLAB**, which gave a CFO value of **22.6034 Hz** — consistent with the GNU Radio estimate.
 
Once estimated, CFO is corrected in GNU Radio using a **complex exponential multiplication** block that de-rotates the constellation and stabilizes the symbol clusters.
 
- Estimated CFO for the reference signal: **≈ 22.63 Hz**
### 3. Constellation Estimation & Modulation Identification
 
**Without CFO correction:**
- The constellation rotates continuously due to the uncorrected frequency offset.
- 4 points (clusters) are observed, smeared along a circular path.
- Inference at this stage: **QPSK or 4-QAM**.
With CFO corrected, the signal is passed through a **QT GUI Constellation Sink** for visual inspection
 
- Number of distinct, stable symbol clusters observed -> **4 clusters**
- Inference: **QPSK (4-QAM)** modulation
### 4. Receiver Architecture
 
The receiver consists of a sequence of processing blocks including File Source, Throttle, Carrier Frequency Offset (CFO) correction, Root Raised Cosine (RRC) filtering, Symbol Synchronization, Constellation visualization and decoding, and finally a File Sink for storing the output. This flow ensures proper conditioning, synchronization, and decoding of the received signal.
 

File Source → Throttle → CFO Correction → RRC Filter → Gardner Symbol Sync → Constellation Decoder → File Sink

 
**File Source and Throttle**
The File Source block is used to read the recorded IQ samples and provide the complex baseband signal as input. The Throttle block limits the data rate to the specified sampling rate (2.048 MHz), ensuring stable and controlled processing during offline execution.
 
**Carrier Frequency Offset Correction**
The received signal may contain a small carrier frequency offset. This is corrected by generating a complex exponential signal using cosine and sine components and multiplying it with the received signal. Mathematically, the corrected signal is expressed as:
 

*x_corrected(t) = x(t) · e^(−j2π·f_cfo·t)*

 
The estimated CFO is approximately 22.63 Hz (estimated using the 4th-power method in GNU Radio, cross-checked in MATLAB), which is small but still corrected to improve system performance.
 
**Matched Filtering (RRC Filter)**
A Root Raised Cosine (RRC) filter is used for matched filtering to reduce noise and inter-symbol interference (ISI). Based on prior estimation, the symbol rate is approximately 256 kHz, and the samples per symbol are 8. The RRC filter shapes the signal optimally for symbol detection.
 
**Symbol Synchronization**
Symbol timing recovery is performed using the Gardner Timing Error Detector. This block aligns the sampling instants with the symbol boundaries and converts the signal to one sample per symbol, which is essential for accurate decoding.
 
**Constellation Analysis and Decoding**
The QT GUI Constellation Sink is used to visualize the signal constellation. From the observed four distinct clusters, the modulation scheme is identified as QPSK. The Constellation Decoder then maps the received symbols to discrete values (0–3) based on the QPSK constellation.
 
**Output Storage**
The decoded symbols are stored in a binary file using the File Sink block.
## Results
 
| Parameter | Estimated Value |
|---|---|
| Sampling Rate | 2.048 MHz (given) |
| Symbol Rate | ≈ 256 kHz |
| Samples per Symbol (SPS) | 8 |
| Carrier Frequency Offset (CFO) | ≈ 22.63 Hz |
| Modulation Scheme | QPSK |
| Output | Decoded symbol stream (values 0–3), 2 bits/symbol |
 


 

 
