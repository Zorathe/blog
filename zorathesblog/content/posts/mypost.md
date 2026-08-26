+++
#type = "post"
#url = "/posts/myposts.md"
date = '2026-07-03T07:05:59-04:00'
draft = false
title = 'Note Detection Using Fast Fourier Transform'
tags = ["FFT", "Fast Fourier Transform", "Signal and Systems"]
[cover] 
    image = 'img/abstract-wave.png'
    alt = 'This is a post image'
    caption = '' 
+++

## How FFT Works in a Note Detector

An FFT (Fast Fourier Transform) is one of the most useful tools for building a **note detector** because it converts a short chunk of audio from the **time domain** into the **frequency domain**.

The frequency domain makes it possible to determine which frequencies are present in the audio and use that information to estimate the musical note being played.

For example, an **A4** has a fundamental frequency of approximately **440 Hz**.

---

## Start with the Microphone Signal

A microphone takes in audio input by measuring the changes in air pressure. 

For example, take this waveform:

``` 
amplitude
   ^
   |    /\      /\      /\
   |   /  \    /  \    /  \
---+--/----\--/----\--/----\----> time

```

If the audio is sampled at 44,100 samples per second, the microphone produces:
```
x[0], x[1], x[2], ... x[N-1]
```

To get a proper representation of the signal, you have to sample a part of the incoming signal. 

For example:

44,100 samples/sec × 0.0464 sec ≈ 2048 samples

Using this, we can feed 2048 samples into the FFT at a particular time.

## What the FFT Does

The FFT takes the audio samples and breaks them up into their individual frequencies. 

The original audio is in the time domain:

```
amplitude
   ^
   |    /\      /\      /\
   |   /  \    /  \    /  \
---+--/----\--/----\--/----\----> time
```
The FFT transforms it into the frequency domain:
```
magnitude
   ^
   |           /\
   |          /  \
   |         /    \
   |________/______\______________> frequency
             440 Hz
```

The peak around 440 Hz indicates that there is strong energy at approximately 440 Hz.

## FFT Frequency Bins

The FFT divides the frequency range into frequency bins.

The spacing between bins is defined as:

Δf=Fs/N

where:

    Fs = sample rate
    N = FFT size
    Δf = frequency resolution

For example, with:
```
Fs = Sample rate = 44,100 Hz

N = FFT size     = 2048
```
we get:

Δf = 44,100Hz / 2048  = 21.533 Hz

The FFT bins are therefore approximately:
```
Bin 0  →   0 Hz
Bin 1  →  21.5 Hz
Bin 2  →  43.1 Hz
...
Bin 20 → 430.7 Hz
Bin 21 → 452.2 Hz
...
```

Notice that each bin is a multiple of Δf. 
An A4 at 440 Hz falls between bins 20 and 21.
This is why a simple FFT doesn't necessarily give you an exact frequency.

## Estimating the Actual Frequency

Suppose the FFT produces this:
```
magnitude
   ^
   |                  /\
   |                 /  \
   |                /    \
   |_______________/______\____________
                  431    452
                    frequency
```
The actual frequency may be somewhere around:

440 Hz

rather than exactly at one of the FFT bins.

A good note detector can use peak interpolation to estimate the frequency more accurately than simply selecting the nearest FFT bin.

## Converting Frequency to a Musical Note

Once the detector estimates the fundamental frequency f, it can convert that frequency into a MIDI note number.

The formula is:
```
m=69+12log⁡2(f/440)
```
for 440 Hz:
```
m=69
```
MIDI note 69 corresponds to A4.

The result can then be rounded to the nearest MIDI note:
```
m_note=round⁡(m)
```
For example:
```
Frequency	Approximate Note
261.63 Hz	C4
293.66 Hz	D4
329.63 Hz	E4
392.00 Hz	G4
440.00 Hz	A4
493.88 Hz	B4
```
## The Complication: Harmonics

This is one of the most important concepts when building a note detector.

A real instrument does not produce only its fundamental frequency.

If someone plays A4, whose fundamental is approximately 440 Hz, the sound may contain:
```
440 Hz   ← fundamental
880 Hz   ← 2nd harmonic
1320 Hz  ← 3rd harmonic
1760 Hz  ← 4th harmonic
2200 Hz  ← 5th harmonic
...
```
The FFT might therefore look something like:
magnitude
```
   ^
   |       /\ 
   |      /  \             /\
   |     /    \           /  \
   |    /      \    /\   /    \
   |___/________\__/__\_/______\____> frequency
      440      880 1320 1760
       ↑
   fundamental
```   

These additional frequencies are called harmonics.

## Why the Largest FFT Peak Isn't Always the Note

A naive detector might do this:

1. Run FFT
2. Find largest peak
3. Assume that frequency is the note

This can fail.

For example, suppose the fundamental at 440 Hz is relatively quiet but the second harmonic at 880 Hz is very strong.

The FFT could show:

440 Hz  → moderate
880 Hz  → very strong
1320 Hz → strong

A naive detector might conclude:
880 Hz → A5
But the musician actually played:
440 Hz → A4
Therefore, a practical note detector needs to reason about harmonic relationships, not just the strongest frequency.

## Detecting the Fundamental

One useful strategy is to look for a frequency f whose multiples also appear in the spectrum.

For example:
```
f       = 440 Hz
2f      = 880 Hz
3f      = 1320 Hz
4f      = 1760 Hz
```
If all of these frequencies have significant energy, that provides strong evidence that:
```
fundamental = 440 Hz
```
the detector can therefore search for a frequency whose harmonic series best matches the observed spectrum.

Conceptually:
```
         Observed spectrum
440 Hz   █████████
880 Hz   ███████
1320 Hz  █████
1760 Hz  ████
```
This is much stronger evidence than simply looking for the largest peak.

## Frequency Resolution vs. Time Resolution

There is an important tradeoff when choosing the FFT size.

The frequency resolution is:
```
Δf=Fs/N
```
Increasing N improves frequency resolution.

For example, at a 44.1 kHz sample rate:
```
FFT Size	Frequency Resolution
512	        86.1 Hz
1024	        43.1 Hz
2048	        21.5 Hz
4096	        10.8 Hz
8192	        5.38 Hz
```
A larger FFT gives you more precise frequency information.

However, it requires a longer chunk of audio.

For example:

4096 / 44100 ≈ 0.0929 seconds

So a 4096-sample window contains about 93 ms of audio.

That means the detector needs to observe roughly 93 ms before it has that entire window available.

This creates a tradeoff:
```
Smaller FFT
    ↓
Shorter delay
    ↓
Worse frequency resolution
```
```
Larger FFT
    ↓
Longer delay
    ↓
Better frequency resolution
```
A real-time note detector needs to choose an appropriate compromise.
## Zero Padding

You may see FFT implementations that take a relatively small audio window and append zeros before calculating the FFT.

For example:
```
2048 audio samples
        ↓
append zeros
        ↓
8192-point FFT
```
Zero padding produces more closely spaced FFT output points.

However, it does not provide four times as much frequency information.

It effectively interpolates the spectrum.

The true frequency resolution is still primarily determined by the duration of the original audio window.

So:

Zero padding
     ≠
More actual information

It can nevertheless make peak estimation easier because the spectrum is sampled more densely.
## Window Functions

Another important part of FFT processing is applying a window function before calculating the FFT.

Suppose we take a section of a continuous waveform:
```
|----------------|
^                ^
start            end
```
The FFT effectively treats the selected section as if it repeats.

If the waveform doesn't line up perfectly at the boundaries, the artificial discontinuity creates additional frequencies.

This produces spectral leakage.
Hann Window

A common solution is to multiply the samples by a Hann window.

The Hann window is:
```
w[n]=0.5−0.5cos⁡(2πnN−1)
```
The audio samples are then multiplied by the window:
```
xw[n]=x[n]w[n]
```
Conceptually:
```
Original audio:

████████████████████████

Hann window:

  ▂▃▅██████████████▅▃▂
```
The ends of the window smoothly approach zero.

This generally produces cleaner FFT peaks and reduces spectral leakage.
## A Basic FFT Note Detector Pipeline

A simple FFT-based detector can follow this pipeline:
```
Microphone
    ↓
Audio samples
    ↓
Take a short window
    ↓
Apply Hann window
    ↓
FFT
    ↓
Calculate magnitude spectrum
    ↓
Find spectral peaks
    ↓
Estimate fundamental frequency
    ↓
Convert frequency → MIDI
    ↓
Convert MIDI → note name
    ↓
C4, D4, A4, etc.
```
A more sophisticated detector can additionally perform:
```
Peak interpolation
Harmonic analysis
Noise rejection
Amplitude thresholding
Temporal smoothing
Confidence estimation
```
## FFT Output and Magnitude

The FFT produces complex numbers.

For each frequency bin, the FFT gives something like:
```
X[k]=a+bi
```
The magnitude of that frequency component is:
```
∣X[k]∣=a2+b2
```
For a note detector, the magnitude is usually what we care about when looking for frequency peaks.

A spectrum can therefore be represented as:
```
Frequency       Magnitude

100 Hz             ██
200 Hz             ███
300 Hz             ██
400 Hz             █████████
500 Hz             ███
600 Hz             ██
...
```
The peaks provide information about the frequencies present in the sound.
## Real-Time Processing

A real-time note detector doesn't necessarily process one window and then wait for the next one.

Instead, it can use overlapping windows.

For example:
```
Window 1:
████████████████

Window 2:
        ████████████████

Window 3:
                ████████████████
```
If the FFT size is 2048 samples, you might advance by only 512 samples each time.

This gives:

FFT size:    2048 samples
Hop size:     512 samples
Overlap:       75%

The detector can therefore produce frequent updates while still getting the frequency resolution of a 2048-sample FFT.

This is particularly useful for real-time tuners and musical instruments.
## Smoothing the Detected Note

The detected frequency can jump around from one FFT window to another.

For example:
```
Frame 1 → 439.2 Hz
Frame 2 → 441.1 Hz
Frame 3 → 438.7 Hz
Frame 4 → 440.5 Hz
Frame 5 → 439.8 Hz
```
These are all essentially A4.

A detector can apply temporal smoothing so the displayed note doesn't constantly jump around.

For example:

Raw frequency:
```
439.2
441.1
438.7
440.5
439.8
442.0
438.9
```

Smoothed:
```
439.8
440.0
439.9
440.1
440.0
```
The detector can also require a note to remain stable for several frames before changing the displayed note.

## Conclusion

The FFT does not detect musical notes. It deconstructs the signal into its individual frequencies. 

The note detector analysis those frequencies and determines which fundamental frequency best represents each one. 

The musical mapping then maps those fundamental frequencies to a corresponding musical note.

So the overall process is:

![FFT Flowchart](/FFT-flowchart-block.png)
















<!-- ![A transformer architecture](/transformer.png) -->