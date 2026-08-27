+++
date = '2026-08-25T21:44:01-04:00'
draft = false
title = 'Note Detection Using Fast Fourier Transform'
tags = ["FFT", "Fast Fourier Transform", "Signal and Systems"]
[cover] 
    image = 'img/abstract-wave.png'
    alt = 'This is a post image'
    caption = '' 
+++

The Fast Fourier Transform (FFT) is one of the most powerful tools for building a **note detector** because it converts a short chunk of audio from the **time domain** into the **frequency domain**.

The frequency domain makes it possible to determine which frequencies are present in the audio and use that information to estimate the musical note being played.

For example, an **A4** has a fundamental frequency of approximately **440 Hz**.

---

## Start with the Microphone Signal

A microphone takes in audio input by measuring the changes in air pressure. 



If the audio is sampled at 44,100 samples per second, the microphone produces:
```
x[0], x[1], x[2], ... x[N-1]
```

To get a proper representation of the signal, you have to sample a part of the incoming signal. 

For example:

44,100 samples/sec × 0.0464 sec ≈ 2048 samples

Using this, we can feed 2048 samples into the FFT at a particular time.


## What the FFT Does

The FFT takes the audio samples which is in time domain and converts them to frequency domain. This allows the audio samples to be broken down into its individual frequencies.

![FFT](/FFT-convert.jpg)


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
```

Notice that each subsequent harmonic is a multiple of the fundamental frequency.

## Why the Largest FFT Peak Isn't Always the Note

A basic note detector might:

1. Run FFT
2. Find largest peak
3. Assume that frequency is the note

This can fail.

For example, a musician plays the note A4 which produces a 440Hz sound which is relatively quiet but a second harmonic at 880 Hz which is the loudest frequency picked up.

A naive detector might conclude:
```
880 Hz → A5
```

But the musician actually played:
```
440 Hz → A4
```
The 880Hz is the second harmonic of the 440Hz fundamental. 

Therefore, a practical note detector needs to take in the harmonic relationships into account, not just the strongest frequency.

## Detecting the Fundamental

Taking into account the harmonic relationships, look at a frequency f and its multiples that may appear in the spectrum.

Using the example from earlier:
```
f       = 440 Hz
2f      = 880 Hz
3f      = 1320 Hz
4f      = 1760 Hz
```
Since 2f = 880 Hz, 880Hz is a multiple of the 440Hz fundamental so the note detector should output A4.

Using this relationship, the detector can search for a frequency whose harmonic series best matches the observed spectrum.

This is much stronger evidence than looking for the largest peak.

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

## Hann Window

When performing the FFT, the FFT may produce some unwanted artifacts in the form of spectral leakage. This is where the Hann window function comes in handy. The Hann window function reduces that aliases by smoothing the signal.  

Mathematically, this is done by multiplying the samples by a Hann window. 

The Hann window is:
```
w[n]=0.5−0.5cos⁡(2πnN−1)
```
The audio samples are then multiplied by the window:
```
xw[n]=x[n]w[n]
```
In essence :

![Hann Window](/hann-window.jpg)

The ends of the window smoothly approach zero.

This generally produces cleaner FFT peaks and reduces spectral leakage.
## A Basic FFT Note Detector Pipeline

Putting it all together, a simple FFT-based detector may have this structure:
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
439.2 Hz
441.1 Hz
438.7 Hz
440.5 Hz
439.8 Hz
442.0 Hz
438.9 Hz
```

Smoothed:
```
439.8 Hz
440.0 Hz
439.9 Hz
440.1 Hz
440.0 Hz
```
The detector can also require a note to remain stable for several frames before changing the displayed note.

## Conclusion

The FFT does not detect musical notes. It deconstructs the signal into its individual frequencies. 

The note detector analysis those frequencies and determines which fundamental frequency best represents each one. 

The musical mapping then maps those fundamental frequencies to a corresponding musical note.

So the overall process is:

![FFT Flowchart](/FFT-flowchart-block.png)







