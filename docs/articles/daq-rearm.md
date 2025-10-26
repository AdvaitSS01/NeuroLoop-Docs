# The DAQ That Only Listens Once: Rearming Finite Tasks in Bonsai

## 1. Introduction

Finite sampling tasks in NI DAQmx behave differently from continuous tasks: they output data once and then silently stop unless explicitly **rearmed**.  

This document presents a systematic account of the problem, the failed attempts, and the operational solution using a **custom C# DigitalOutput node in Bonsai**.

---

## 2. Context: Finite vs Continuous Sampling

**Continuous mode:** DAQ loops the buffer indefinitely; output continues without user intervention.  

**Finite mode:** DAQ plays a buffer of N samples once and stops.  

Key point: Bonsai’s DAQ nodes support both modes, but **finite tasks do not reset automatically**. Without explicit recreation or rearming, the task remains inactive after a single trial.

---

## 3. Symptom

- Trial 1: DAQ outputs waveform correctly.  
- Trial 2+: No output, no error. Task appears dead.  

Interpretation: task lifecycle in finite mode requires explicit handling; Bonsai’s default nodes do not rearm tasks automatically.

---

## 4. Debugging Attempts (Dead Ends)

### 4.1 Buffer Size Experiments
- Attempted sequences like `[0,1,0,1,...]` of varying lengths.  
- Observation: **buffer size must equal sample count exactly**.  
- Oversized or undersized buffers led to silent failures.

### 4.2 Autostart Control
- `autostart = true`: task fired once immediately.  
- `autostart = false`: task armed but did not start without manual trigger.  
- Neither mode provided consistent trial-to-trial operation.

### 4.3 Adding Delays (`Thread.Sleep`)
- Attempted delays between task creation and trigger.  
- Result: inconsistent behavior; **delays do not replace rearming**.

### 4.4 Creative Trigger Routing
- Routed one output’s trigger to another PFI line.  
- Observation: tasks not armed will not respond to triggers.  
- Conclusion: trigger routing cannot fix finite task lifecycle.

---

## 5. Breakthrough

Finite tasks **must be explicitly rearmed** or recreated after each trial.  
- This aligns the task lifecycle with trial execution.  
- Ensures predictable output across repeated trials.

---

## 6. Operational Solution: Custom Digital Output Node

The custom node implements:

- Task recreation for each new buffer  
- Buffer flush before task creation  
- `autostart = false` (manual control)  
- `.StartTrigger` exposed for external triggers  
- Explicit `.Start()` in `OnNext()`  
- `Rearm = true` for automatic readiness each trial  

**Outcome:** deterministic, trial-by-trial DAQ output.

---

## 7. Mechanistic Insight

- Core issue: **task lifecycle in finite mode**  
- Default Bonsai DAQ nodes do not handle rearming  
- Explicit recreation + rearm aligns DAQ state with experiment trial sequence  
- Ensures DAQ outputs every trial without silent failure

---

## 8. Operational Principles (Design Rules)

1. **Finite-mode tasks require exact buffers:** `buffer size == sample count`  
2. **Recreate tasks per trial:** ensures rearm and fresh handles  
3. **Disable autostart:** call `.Start()` manually for precise control  
4. **Use thread-level isolation for concurrent outputs:** `ObserveOn(NewThreadScheduler)`  
5. **Prefer hardware routing for determinism**: physical lines more reliable than software-only triggers  
6. **Avoid `Thread.Sleep()` for synchronization:** nondeterministic  
7. **Catch silent failures via active debugging:** use Visual Studio or `ConsoleWrite` logging  

---

## 9. Conceptual Insight (Temporal Framing)

Finite-task behavior can be understood as a **temporal alignment problem**:

- **Hardware time:** DAQ engine and PFI routing  
- **Software time:** Bonsai reactive scheduler, node execution  
- **Cognitive time:** experimenter’s planning of trials and triggers  

Rearming ensures alignment of hardware and software time, allowing cognitive planning to succeed.

---

## 10. Classification (Metadata)

- **Domain:** Systems Neuroscience / Experimental Control  
- **Software layer:** Bonsai (DAQmx nodes, Script, ObserveOn, Scheduler)  
- **Hardware layer:** NI DAQ (PFI1, PFI2, AO/DO lines)  
- **Category:** Synchronisation — Finite Sampling, Task Lifecycle, Concurrency  

---

## 11. Summary & Next Steps

**Deterministic operation of finite DAQ tasks requires:**

- explicit task lifecycle management (recreate + `.Start()`)  
- explicit rearming between trials  
- controlled trigger routing for trial synchronization  

**Next steps:**

- Share C# source for custom node  
- Integrate with AO + DO synchronization pipeline  
- Quantitative verification with timestamps and oscilloscopes  
- Create minimal reproducible Bonsai graph for reference  

**NeuroHardware entry:** this document provides a canonical workflow for finite-task rearming, capturing failure modes, operational principles, and reproducible strategies.

