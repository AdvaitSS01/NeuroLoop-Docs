# Synchronisation of NI-DAQ, Camera, and Arduino in Closed-Loop Experiments

## 1. Objective

Ensure deterministic alignment of multiple data streams in a closed-loop experimental rig:

- **DAQ (NI-DAQ)**: generate TTL pulses for camera triggers and valve actuation.
- **Camera (VimbaCapture)**: acquire frames precisely aligned with TTL triggers.
- **Arduino**: record and respond to external events, such as eye-state signals.

The goal is **sub-millisecond synchrony** between all devices, minimizing drift, jitter, or dropped events.

---

## 2. Initial Configuration

- **DAQ tasks**: DigitalOutput generating trial pulses; initially assumed Bonsai timestamps would suffice.
- **Camera**: VimbaCapture node, acquiring frames asynchronously.
- **Arduino**: listening to TTL lines and logging events.

**Observed issues**:

1. Camera timestamps drift relative to DAQ pulses.
2. Bonsai timestamps reflect processing time, not physical event generation.
3. Multiple devices acting as “masters” caused inconsistencies.

---

## 3. Device Roles and Temporal Hierarchy

Establish **device authority** for synchronization:

| Device | Role | Temporal Reference |
|--------|------|------------------|
| NI-DAQ | Master clock | Generates trial edges via PFI lines |
| Camera | Slave | Triggered by DAQ edge |
| Arduino | Slave | Records events relative to DAQ pulse |

**Key principle**: one deterministic clock (DAQ) governs all timing.

---

## 4. Synchronization Methodology

### 4.1 DAQ Task Configuration

- **DigitalOutput**: generates trial start pulses.
- **Rearming enabled**: supports repeated trials without restarting tasks.
- **Autostart disabled**: `.Start()` explicitly called after task creation.
- **Buffer/sample alignment**: finite mode with `buffer_size == sample_count`.

### 4.2 Camera Triggering

- External trigger input wired to DAQ digital line.
- VimbaCapture configured to acquire frames on DAQ rising edges.
- Internal Bonsai timestamps ignored for analysis; only used for debug visualization.

### 4.3 Arduino Integration

- TTL input wired to DAQ pulse line.
- Events timestamped relative to reception of DAQ pulses.
- Optional: filters applied in Bonsai to reduce frame-to-frame noise.

---

## 5. Additional Experiments Attempted

- Cascading triggers (AO exports to PFI2 → DO): failed because AO did not start early enough to generate the export (catch-22).
- Adding `Thread.Sleep()` or `WaitUntilDone`: ineffective — only masked the race, did not ensure reliable arming.
- Relying on software triggers alone: proved brittle compared to physical line routing.

---

## 6. Behavioural Verification

- For hardware triggering, used a Bonsai-scripted rising edge on a digital line physically wired to **PFI1**.  
- Qualitative verification across many trials showed both outputs beginning together on the trigger when using custom nodes + `ObserveOn` threading configuration.  
- Observed behaviour: TTL trains and analog waveform started as expected, with consistent trial-to-trial execution.

> Note: quantitative jitter and latency measurements (oscilloscope traces, timestamped DAQ logs) are planned for a follow-up post; current validation is functional and trial-consistent.

---

## 7. Operational Graph

Timer → Script (DAQ pulses) → ObserveOn → DigitalOutput → PFI1 → Camera External Trigger
↘ Arduino listens TTL → Event Logging


- Both Camera and Arduino events reference DAQ rising edge.  
- `ObserveOn` / `NewThreadScheduler` ensures concurrent task readiness.  
- DAQ tasks explicitly rearmed between trials.  

---

## 8. Operational Principles (Design Rules)

1. **Finite-mode tasks require exact buffers:** In finite sampling, match `buffer size == sample count` exactly.  
2. **Recreate DAQ tasks per trial:** Do not reuse finite-mode tasks; recreate them in `OnNext()` to ensure rearm and fresh handles.  
3. **Disable autostart:** Use explicit `.Start()` after initializing the task to control the exact arming moment.  
4. **Use thread-level isolation for concurrent arming:** `ObserveOn(NewThreadScheduler)` gives each output its own thread context and prevents serial-arming race conditions.  
5. **Prefer hardware routing for determinism:** Physically wiring a digital line into PFI1 is more reliable than purely software-triggered signals.  
6. **Avoid `Thread.Sleep()` for synchronization:** Sleep/delay hacks do not guarantee proper arming; they only add nondeterministic timing windows.  
7. **Catch silent failures via active debugging:** Bonsai may not surface DAQmx errors; attach Visual Studio or add logging (`ConsoleWrite`) to detect task state issues.  

---

## 9. Conceptual Insight (Temporal Framing)

Synchronization is a three-clock alignment problem:

- **Hardware time** — the DAQ’s internal engine (PFI routing, timing engine).  
- **Software time** — Bonsai’s reactive scheduler and node execution semantics.  
- **Cognitive time** — the experimenter’s mental model of event ordering and causality.  

A reliable system aligns all three: tasks must be prepared (software), routes must be correctly wired (hardware), and the experimenter must conceive triggers and trial structure coherently (cognitive). Treat each output node as a *temporal agent* that must be explicitly prepared and synchronized.

---

## 10. Classification (Metadata)

- **Domain:** Systems Neuroscience / Experimental Control  
- **Software layer:** Bonsai (DAQmx nodes, Script, ObserveOn, Scheduler)  
- **Hardware layer:** NI DAQ (PFI1, PFI2, AO/DO lines)  
- **Category:** Synchronisation — Finite Sampling, Triggering, Concurrency  

---

## 11. Summary & Next Steps

Deterministic alignment of DAQ, camera, and Arduino in Bonsai requires:

- explicit DAQ task lifecycle management (recreate + `.Start()`),  
- single clock authority (DAQ) governing all events,  
- and explicit thread separation for concurrent arming (`ObserveOn` + `NewThreadScheduler`).  

**Next practical steps:**

- Share the **C# source** for the custom DO/AO nodes (planned follow-up).  
- Add **quantitative verification** (oscilloscope traces and jitter histograms).  
- Convert the configuration into a reproducible Bonsai graph file and a minimal reproducible example.  

**NeuroHardware entry:** this document is intended to be a canonical node in the NeuroHardware lexicon: it captures a repeatable synchronization pattern (failure modes, mechanism, and design rules) applicable to other DAQ+visual-pipeline integrations.

