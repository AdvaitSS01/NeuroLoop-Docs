# Measuring and Characterizing Temporal Jitter in Closed-Loop Experiments

## 1. Concept definition

**Goal:** define a systematic method to measure and characterize **temporal jitter** in closed-loop neuroscience experiments that integrate multiple devices (DAQ, camera, Arduino).  

**Temporal jitter** refers to the variability in the timing between events that are supposed to occur at fixed intervals or in fixed causal order. It represents the deviation between intended and actual timing of hardware-software interactions.  

Target outcome: establish quantitative metrics and measurement techniques for assessing jitter in a multi-device experimental loop.

---

## 2. Context & motivation

In closed-loop systems, **timing defines causality**. The experiment’s validity depends not only on correct signal order but also on **precise temporal alignment**.  
Even small drifts or inconsistencies—on the order of milliseconds—can distort the interpretation of neural or behavioral responses.  

Common contributors to jitter:
- Variable acquisition rates (camera, DAQ)
- Software scheduler latency (Bonsai threading)
- Communication delays (Arduino serial, DAQ trigger routing)
- Hardware interrupt conflicts

Quantifying jitter allows researchers to separate **biological variability** from **instrumental variability**.

---

## 3. Observed behaviour & failure modes

### 3.1 Symptoms
- TTL pulses that appear phase-shifted across trials  
- Irregular frame acquisition intervals  
- Inconsistent camera-DAQ trigger alignment  
- Arduino-triggered actions delayed by unpredictable microseconds-to-milliseconds  

### 3.2 Typical observations
| Source | Expected interval | Observed variability | Comment |
|--------:|------------------:|--------------------:|----------|
| NI DAQ output | 1 ms | ±0.05 ms | Hardware-level stable |
| Camera trigger (via PFI) | 2 ms | ±0.2 ms | Dependent on exposure |
| Bonsai operator events | variable | ±1–3 ms | Scheduler-dependent |
| Arduino serial responses | 0.5 ms | ±0.5–2 ms | USB latency + interrupt load |

**Key insight:** even microsecond-level discrepancies propagate across hardware boundaries, amplifying perceived latency and distorting real-time control.

---

## 4. Mechanistic analysis

| System layer | Source of jitter | Control strategy |
|--------------:|----------------:|-----------------:|
| **Hardware** | Clock phase drift, interrupt overlap | Use hardware-timed triggers and shared clocks |
| **Software** | Thread scheduling delays | Use dedicated ObserveOn scheduler and minimal CPU load |
| **Communication** | USB buffer unpredictability | Prefer TTL or direct DAQ routing over serial |
| **Acquisition timing** | Variable frame exposure | Lock exposure and frame rate explicitly |

**Core principle:** jitter is not a single parameter but an **interaction between clocks** operating asynchronously across devices.

---

## 5. Implementation — steps to measure and quantify jitter

### 5.1 Direct measurement via TTL timestamps
- Connect all trigger lines (DAQ, camera, Arduino) to **separate DAQ input channels**.  
- Record their digital states simultaneously.  
- Extract **rising-edge timestamps** and compute pairwise temporal differences.

### 5.2 Bonsai-based timestamp logging
- Use **Timestamp node** downstream of each event stream (e.g., `DigitalOutput`, `CameraFrame`).  
- Log timestamps to CSV.  
- Align streams by nearest-neighbour matching of timestamps.  
- Compute Δt = |t₁ - t₂| across all matched pairs.

### 5.3 Quantification metrics
- **Mean offset (latency):** average delay between two streams.  
- **Standard deviation (jitter):** variability in that delay.  
- **Histogram of Δt:** visualization of temporal spread.  
- **Cumulative deviation:** total drift over experiment duration.

### 5.4 Validation
- Cross-check jitter using **oscilloscope or logic analyzer** for hardware confirmation.  
- Compare Bonsai timestamps with hardware capture logs.

---

## 6. Behavioural verification

Observed results in test loops:
- Hardware-only trigger chain (DAQ→Camera) jitter ≤ 0.1 ms  
- DAQ→Arduino (via TTL) jitter ≈ 0.3 ms  
- Bonsai event propagation (software level) jitter 1–3 ms depending on CPU load  

Controlled experiments confirmed:
- Hardware synchronization is highly stable.  
- Software-synchronized events require CPU prioritization and scheduler control.

---

## 7. Operational principles (design rules)

1. **Always timestamp at hardware and software levels.**  
2. **Use hardware triggers as reference clocks** wherever possible.  
3. **Minimize USB or serial dependencies** for time-critical events.  
4. **Run Bonsai on dedicated scheduler threads** for consistent event timing.  
5. **Log raw timestamps continuously** for post-hoc drift analysis.  
6. **Quantify jitter before interpreting physiological variability.**

---

## 8. Conceptual insight (temporal framing)

Closed-loop control relies on **coherent temporal mapping** between cause and effect.  
If multiple clocks define “now,” then **jitter is the disagreement between those clocks**.  

Temporal stability thus represents a form of *causal precision* — the degree to which an experimental system maintains consistent timing alignment between its sensory and actuation loops.

---

## 9. Classification (metadata)

- **Domain:** Systems Neuroscience / Temporal Analysis  
- **Software layer:** Bonsai (Timestamp, Subject, Join, CSVWriter)  
- **Hardware layer:** NI DAQ (PFI sync), Vimba camera, Arduino TTL lines  
- **Category:** Jitter Analysis, Synchronization, Real-Time Verification  

---

## 10. Summary & next steps

Temporal jitter can be **measured, visualized, and controlled** through systematic timestamping and hardware synchronization.  

**Next steps:**
- Incorporate direct DAQ timestamping into all future trials.  
- Build automated Bonsai nodes for jitter computation.  
- Develop graphical representations to visualize drift across experiment duration.  

---

**NeuroHardware entry:**  
Establishes a reproducible framework for quantifying timing reliability across multi-device closed-loop systems in neuroscience experiments.
