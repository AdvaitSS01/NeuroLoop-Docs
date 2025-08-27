# Syncing NI-DAQ with the Rest of the World (or, Why My Camera Was Always Late)

If you’ve ever tried to run a Bonsai experiment with multiple devices — a DAQ, a camera, maybe even an Arduino — you’ll know the pain: nothing ever seems to agree on **time**.  

At first, I thought the NI-DAQ would just magically “be the clock.” Spoiler: it isn’t, unless you tell it to.

---

## The Premise

I wanted to run a closed-loop experiment where:

- The DAQ generated TTL pulses for camera triggers and valves.
- The camera frames had to align with those pulses.
- Extra metadata (like eye state from image analysis) went to the Arduino.

Simple enough in theory. In practice, I ended up with timestamp drifts, dropped frames, and one very sulky DAQ that seemed to “do its own thing.”

---

## What Went Wrong

1. **Camera drift**: VimbaCapture timestamps didn’t line up with DAQ pulses.  
2. **Bonsai timestamps ≠ device time**: Bonsai shows when *it* processed the event, not when the device actually generated it.  
3. **Multiple masters**: I was effectively letting the DAQ, the camera, and Bonsai all believe they were in charge.

---

## What We Tried

- **Using Bonsai timestamps directly** → looked fine at first, but subtle misalignments accumulated.  
- **Trusting the camera’s timestamps** → still didn’t line up with the DAQ edges.  
- **Comparing across devices** → I had to actually plot everything to see how bad the drift was.  
- **Skipping extra timestamps** → since Arduino produced fewer samples, I tried forcing alignment by skipping “extra” ones in camera/DAQ streams. Hacky, not sustainable.  

---

## The Breakthrough

The key realization: **the DAQ can be the global clock — but only if you make it.**

- Use the DAQ’s **start trigger** (PFI lines) as the “true” clock source.  
- Configure the DAQ **both as generator (triggers) and listener (re-armed tasks)**.  
- Wire the DAQ trigger into the camera’s external trigger input.  
- Bonsai just passes signals around — don’t trust its internal timestamps for synchronization.  

---

## What Finally Worked

1. **DAQ generates a digital edge** at the start of each trial.  
2. **Camera trigger line** comes straight from the DAQ, so every frame is tied to that same edge.  
3. **Arduino timestamps** its events relative to when it *receives* DAQ pulses.  
4. In Bonsai:  
   - DAQ DigitalOutput + Camera VimbaCapture share the same trigger.  
   - Re-armed DAQ tasks allow repeating without restarting Bonsai.  

---

## The Takeaway

- Bonsai timestamps are fine for debugging, not for science.  
- If you want sub-millisecond alignment, **force everything to follow the DAQ’s clock**.  
- Rearming + external triggers = the difference between chaos and clean synchrony.  

Or, in one line:  
**Pick one clock. Make everything else obey it.**  

---

*Next up: how rearming tasks saved me from the “one-shot DAQ problem.”*

