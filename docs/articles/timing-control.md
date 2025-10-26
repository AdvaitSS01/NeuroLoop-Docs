# Advanced Timing Control in Closed-Loop Experiments

## 1. Introduction

Precise timing is the backbone of closed-loop neuroscience experiments. Multiple devices — DAQs, cameras, Arduinos, stimulators — must act in concert. Misalignment of even a few milliseconds can corrupt data or invalidate experimental results.  

This article consolidates lessons from prior work on:

- Synchronising DAQ, camera, and Arduino events  
- Managing finite sampling tasks and rearming  
- Concurrent arming of analog and digital outputs with Bonsai  

It then extends these ideas to **systematic timing strategies**, clock hierarchies, and experimental design principles.

---

## 2. Timing Challenges Across Devices

1. **Hardware clocks differ**:  
   - DAQ internal engine (PFI routing, timing engine)  
   - Camera frame clock (exposure start vs acquisition timestamp)  
   - Arduino loop timing (serial processing + event handling)

2. **Software-induced delays**:  
   - Bonsai nodes execute serially unless threaded (`ObserveOn`)  
   - Reactive scheduling can introduce microsecond-to-millisecond shifts

3. **Finite-mode tasks**:  
   - Without rearming, DAQ outputs once and halts  
   - `autostart` can hide lifecycle problems  

4. **Trigger propagation**:  
   - Hardware routing (PFI lines) is deterministic  
   - Software-triggered edges are less reliable  

**Insight:** All timing issues are fundamentally about **three clocks** — hardware, software, and cognitive.

---

## 3. Temporal Hierarchy Design

To achieve deterministic control:

1. **Select a master clock** (usually the DAQ)  
   - All devices derive timing relative to this source  
2. **Align slave devices**  
   - Camera trigger lines wired to DAQ PFI  
   - Arduino timestamps events on DAQ edges  
3. **Implement task lifecycle control**  
   - Finite-mode tasks recreated and rearmed per trial  
   - Explicit `.Start()` call to control arming moment  
4. **Thread-level separation**  
   - ObserveOn(NewThreadScheduler) ensures concurrent readiness  
5. **Verification**  
   - Qualitative: consistent output across repeated trials  
   - Quantitative: oscilloscope traces, jitter analysis  

> Key principle: deterministic timing requires enforcing a single source of truth and respecting device-level execution constraints.

---

## 4. Multi-Device Graph (Canonical Pattern)

Timer --> Script (white noise) --> ObserveOn --> AnalogOutput  
      --> Script (TTL) --> ObserveOn --> DigitalOutput  

Trigger Source --> RisingEdge --> DO line --> PFI1  
Both AO and DO use PFI1 as StartTrigger  
Camera trigger line receives PFI1 edge  
Arduino timestamps events on PFI1 reception  

**Explanation:**  
- Timer produces trial markers.  
- Script nodes generate waveforms or TTL pulses.  
- ObserveOn ensures each output node arms independently, preventing serial-race issues.  
- Hardware triggers (PFI lines) enforce a deterministic temporal reference.

---

## 5. Operational Principles (Design Rules)

1. **Finite-mode buffers**: match `buffer size == sample count` exactly.  
2. **Recreate tasks per trial**: ensures rearming and fresh handles.  
3. **Disable autostart**: explicit `.Start()` provides precise control.  
4. **Thread isolation**: ObserveOn(NewThreadScheduler) prevents serial execution races.  
5. **Prefer hardware triggers**: physical routing (PFI) is more reliable than software edges.  
6. **Avoid Thread.Sleep() hacks**: non-deterministic and unreliable.  
7. **Active error detection**: attach Visual Studio or log task states to catch silent failures.  
8. **Single clock authority**: DAQ as master; all devices obey it.

---

## 6. Conceptual Insight (Temporal Framing)

Synchronization requires **three clocks**:

- **Hardware time** — device engines, PFI routing  
- **Software time** — Bonsai execution semantics  
- **Cognitive time** — experimenter’s planning and trigger design  

Align all three for deterministic outcomes. Treat each output node as a **temporal agent** that must be prepared, armed, and synchronized explicitly.

---

## 7. Classification (Metadata)

- **Domain:** Systems Neuroscience / Experimental Control  
- **Software layer:** Bonsai (DAQmx nodes, Script, ObserveOn, Scheduler)  
- **Hardware layer:** NI DAQ (PFI1, PFI2, AO/DO lines), Camera, Arduino  
- **Category:** Synchronization — Finite Sampling, Triggering, Concurrency, Multi-Device Alignment  

---

## 8. Summary & Next Steps

Achieving reliable timing in closed-loop experiments requires:

- A **single master clock** (DAQ)  
- Explicit task lifecycle management (recreate + `.Start()`)  
- Concurrent arming via threading (`ObserveOn + NewThreadScheduler`)  
- Hardware trigger routing to enforce determinism  

**Next steps for experimental practice:**

1. Quantitative verification: jitter histograms, oscilloscope traces.  
2. Reproducible Bonsai graphs for all timing patterns.  
3. Sharing canonical C# nodes for DO/AO with rearm and trigger properties.  

**NeuroHardware entry:** This article formalizes **timing control patterns** as a canonical node in the NeuroHardware lexicon: failure modes, mechanisms, and operational rules for deterministic multi-device experiments.

