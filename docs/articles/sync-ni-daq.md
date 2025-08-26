---
title: "Synchronising NI DAQ Analog and Digital Output in Bonsai Without LabVIEW"
date: 2025-08-26
layout: post
categories: [Bonsai, DAQ, Guides]
tags: [NI-DAQ, Bonsai, analog, digital, synchronization]
---

## Introduction

Real-time synchronisation of multiple data streams and triggers is a core requirement in systems neuroscience, especially in closed-loop experiments. Minimising latency is essential — but if you’re using National Instruments' DAQ hardware without LabVIEW, relying instead on open tools like Bonsai, you’re in for a confusing ride — the kind where you don’t even know if you’re holding the map the right way up.

DAQ hardware is powerful — internal routing, programmable timing engines, and triggering flexibility — but the real challenge is not the hardware. It’s understanding the complex DAQmx libraries and their behaviour in tools that abstract them away. NI DAQs are like dragons that respond only to ancient, undocumented spells.

This post outlines the core problems I encountered while synchronising NI DAQ analog and digital output lines inside Bonsai, and how I eventually solved them by blending Bonsai’s reactive programming model with DAQ-level control — all without any external accessories like LabVIEW. The solutions here rely on tweaking both the Bonsai graph and custom C# code.

---

## Background: Bonsai, DAQmx, and the Black Box

Bonsai provides a DAQmx package with AnalogOutput, DigitalOutput, and related IO nodes. These are minimal wrappers around DAQmx functions, designed to get you started with visual pipelines. But things get tricky when you move beyond static IO.

I needed to synchronise:

- A digital output signal (e.g., TTL square wave) to trigger a camera or another device
- An analog output signal (e.g., white noise, ramp, or sine wave) to control stimulation

Both signals needed to start on a shared trigger — routed to PFI1. I used a script to generate both signals, connected them to the respective output nodes, and expected the common trigger to start both outputs in sync. Simple idea. Didn’t work. Like two sprinters waiting for a starting gun, but only the one closest to the ref hears it and takes off.

---

## Problem 1: Digital Output — Finite Sampling Weirdness

The digital output node in Bonsai accepts an OpenCV `Mat` formatted as `{1, N}` — where N is the number of samples. But here’s the catch — in **finite sampling mode**, you must match your **buffer size exactly** to your **sample count**. Unlike continuous mode, you can’t have extra samples lounging around.

Initially, I tried sending a TTL waveform like `[0,1,0,1,...]` over 10,000 samples at 1 kHz. But what I got was a **single TTL pulse**, then nothing — no errors either.

### Debugging It with Visual Studio

Attaching Visual Studio to the running Bonsai process revealed:

- The task was **not being recreated** after each trigger.
- Sample buffers weren’t flushed correctly.
- `autostart` behaviour in Bonsai wasn’t granular enough for DAQ control.
- **Rearm** functionality was missing — repeated triggering simply didn’t work unless manually reset.

### Fix: Writing a Custom C# Digital Output Node

I wrote a new node in C# that:

- Accepts a 1D matrix and repeats it internally
- Recreates the task each time data is received
- Allows configuration of sample rate and buffer size
- Explicitly disables autostart
- Exposes a `StartTrigger` property for external triggering
- Arms the task and calls `.Start()` manually inside `OnNext()`
- Includes `Rearm = true` for repeated trials

This gave me proper finite-sampled TTL trains, with behaviour predictable across trials.

---

## Problem 2: Analog Output Needs the Same Treatment

Analog output had similar issues — Bonsai’s default node assumes simple, one-shot execution. I wanted white noise output sampled at 1000 Hz for 15 seconds. Using a `Script` node to generate the waveform, connected to the analog output with `autostart = false`, I expected it to trigger along with the digital output.

Only one of them fired. The other stayed silent.

---

## Problem 3: Triggering Two Outputs Simultaneously — and Failing

At this point, both custom nodes worked individually. But together?

- DO and AO shared the same `StartTrigger` (PFI1).
- Only one line executed on trigger — whichever one was declared first.

It’s as if Bonsai politely queues your nodes in single file, despite your desperate plea for them to start together like a firing squad.

I suspected a race condition or hardware conflict on shared start triggers. I tried modifying the AO node to export a signal to another PFI (PFI2) and use that as the trigger for DO. Still didn’t work.

Then I added a **1.5 second delay** between arming and triggering, thinking both nodes would be ready by then. No dice. Delaying doesn’t guarantee both nodes are armed because Bonsai’s default execution is serial, not parallel.

---

## Realisation: Bonsai Runs Nodes Synchronously

Bonsai, bless its heart, believes in order. Even when what you really want is organised chaos.

The core issue: Bonsai’s default scheduling — nodes execute serially, meaning one gets armed, the other doesn’t. No matter how clever your trigger routing, only one node actually sees the signal in time.

I even explored inserting `Thread.Sleep()` and playing with `WaitUntilDone`. Those were hacks — ineffective at solving the root problem.

---

## Final Fix: ObserveOn and Asynchronous Threading

The reactive model of Bonsai shines — but only if you understand threading.

The fix:

- Insert `ObserveOn` before each output node
- Set the `Scheduler` property to a custom thread scheduler (using `Externalize`)
- Connect `NewThreadScheduler` nodes to enforce asynchronous execution

Now each output node operates on its own thread — like giving each output its own room, so they stop fighting over who gets the bathroom first.

---

## Physical Wiring and Trigger Graph

To send the actual trigger signal, I used a `DigitalOutput` line from Bonsai (via a scripted rising edge), physically wired to `PFI1` on the DAQ. That line acts as the master trigger for both DO and AO nodes.

I also explored internal signal routing where AO exports a trigger signal to PFI2 for DO to listen to. Elegant in theory, but it failed because AO wasn’t starting in time to generate the export — a catch-22 born of serial execution.

---

## Bonsai Graph Summary

pgsql
CopyEdit
Timer → Script (white noise) → ObserveOn → AO
      ↘︎ Script (TTL)       → ObserveOn → DO

Trigger Source → RisingEdge → DO line → PFI1
Both AO and DO nodes use PFI1 as their .StartTrigger

# Lessons Learned (The Real Debugging Process)

Here are things that matter but rarely get discussed:

- **Finite vs Continuous Sampling:** Buffer sizes must exactly match sample counts in finite mode.  
- **Rearm is critical:** Without enabling it, DAQ tasks won’t respond on the next trial. DAQmx isn’t the kind of guest who refills their own plate at dinner — you have to invite them back every time.  
- **Silent failures are dangerous:** Bonsai won’t throw DAQ errors unless explicitly caught — use Visual Studio + `ConsoleWrite` nodes.  
- **Threading matters:** Execution order in Bonsai is serial unless split using `ObserveOn` and custom schedulers.  
- **Delays and Sleep don’t fix race conditions:** Only proper threading guarantees concurrent arming.  
- **Physical wiring is still best:** Sending a digital pulse via hardware remains the most reliable trigger.  
- **Internal routing is tricky:** Exported signals from AO can’t trigger DO if AO never starts — again, catch-22.  

---

# Reflections

Synchronising AO and DO in Bonsai without LabVIEW is not trivial. But with a mix of custom node design, reactive programming, and careful hardware routing, it’s doable — just not out of the box, and certainly not without a few existential questions along the way.

You need to:

- Understand how `ObserveOn` separates thread contexts  
- Be explicit about `autostart`, task creation, and trigger assignment  
- Test each node independently before syncing them  

This post is just a starting point. In a future one, I’ll share the full C# code for the custom DO and AO nodes.


