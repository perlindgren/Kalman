# Kalman model for vinyl position tracking

The goal is to estimate the (relative) position, speed and acceleration of a vinyl record in real time (and based on that reconstruct another audio signal as if the sound had been on the vinyl in the first place).

## Setup and measurement

The vinyl record holds a tracking signal $(l,r)$, where $l$ and $r$ are sinusoids with nominal frequency at 1kHz (at 33 1/3 rpm), where $l$ and $r$ are separated by 90 degrees phase shift, as to make it possible to determine direction.

The signal $(l,r)$ is sampled using a computer sound card directly from the output of the Moving Magnet (MM) pickup without pre-amplification (this is possible due to the relatively high output of the MM).

Samples come in chunks of 16-2048 readings dependent on sound card settings. Sample frequency is typically 44.1kHz or more. Assuming nominal playback velocity (33 1/3 rpm), one period amounts to 44 samples. At 10 times the nominal velocity each period amounts to 4 samples. (Notice 10 kHz is well below the Nyquist limit of signal reconstruction.)  

## Sources of errors

The physical characteristics of the vinyl + MM pickup + hand movement may introduce errors. Imperfections (scratches and dirt) on the surface may cause transients, affecting both or only a single channel (the stereo signal is encoded by 90 degrees offset). Wear will cause output level reduction and distortion. Dirt on the stylus may affect volume output and/or distortion in various amounts. Hand movement on the platter/record, may introduce offsets, and even make the stylus lose contact with the surface (the needle may jump off the record landing on the same or nearby track). Thus the perceived quality of the tracking relies on a robust estimation. (Tests show however that under normal operation the signal is surprisingly clean.)

---

## Naive implementation

One simple approach is to search the input stream for zero crossings for each signal, and feed that to a state machine recording position and direction. While this works it will give position updates at discrete times where crossings are detected. If the vinyl moves at low speed, crossings will appear less frequently and some sample frames may have no zero crossings (thus no position updates). That is problematic for reconstructing an audio signal from the position estimates (without further modelling).

Another problem with this approach is that noise in the signal(s) may trigger false state updates (zero crossings). To that end, filtering of the input signals can be done, however the filter needs to accommodate both the case of high velocity (where we ultimately may have only 4 readings to determine two crossings) and the case where the velocity goes towards zero. For truthful reproduction we want to track slow movements as well. The energy of the signal depends on the velocity (proportional?), so signal to noise ratio decreases towards 0, stressing the need for larger filter windows at low velocity.

While the approach has been implemented using an averaging filter with static size (5 samples) and tracking has shown to work in practice, it may be further improved on (by filter window adoption/median instead of averaging etc.) The main problem is the lack of prediction, where complete frames (down to 16 in size) would have no position updates.

Example:
1/10 nominal velocity. 100Hz freq. amounts to 400 crossings/sec (Hz), 1/400 = 0.0025s interval.

Sample freq 44.1kHz, buffer 16 bytes, amounts to 16/44100 = 0.00036s of audio data. Only 1/7 frames will have a crossing.

This is not the only problem at hand. Already mentioned is the need for filtering of sensor inputs. However, manual tuning, and ad-hoc fixes seems inappropriate, we better seek a more robust solution.

---

## Kalman after state transition detection

One possible approach would be to use a Kalman filter that would operate on the output from the State machine described above.

### Model

Let us first model the physical properties of the record/platter.

$x_{n+t} = x_n + v_n * t + a_n*t^2$

So given that we have estimated the position, velocity and acceleration at time $n$, we can estimate the new position at time $n + t$.

---

## Kalman directly on the input signal

An interesting observation is that the velocity (and acceleration) of the record is directly proportional to frequency (and frequency changes) of the sampled signal. Problem is now to how to extract them from the acquired samples.

Another observation is that the two signals we sample have "common modes" (same frequency, and phase shifted by 90 degrees. That might be helpful to suppress the influence of bad input data.

Assume we treat the input data stream in chunks of 16 samples.

### System Model

<img src="https://latex.codecogs.com/gif.latex?
\begin{bmatrix} w(k+1) \\ acc(k+1) \\ A(k+1) \\ B(k+1) \end{bmatrix} = \begin{bmatrix}
1 & \Delta t & 0 & 0 \\ 
0 & 1 & 0 & 0 \\ 
0 & 0 & 1 & 0 \\ 
0 & 0 & 0 & 1
\end{bmatrix}
\begin{bmatrix} w(k) \\ acc(k) \\ A(k) \\ B(k) \end{bmatrix} 
"/> 

Where $w$ is the phase, $acc$ is the phase acceleration. $A$, $B$ are individual is the gains for the input channels (slowly changing).

### Measurement Model
<img src="https://latex.codecogs.com/gif.latex?
 \begin{bmatrix}
x(k) \\ 
y(k) 
\end{bmatrix} =
\begin{bmatrix}
A(k) \cdot w(k) \cdot sin \left ( w(k) \cdot t + A(k) \right ) \\ 
A(k) \cdot w(k) \cdot cos \left ( w(k) \cdot t + B(k) \right )
 \end{bmatrix}
"/> 

<!-- ### Measurement Model2
<img src="https://latex.codecogs.com/gif.latex?
 \begin{bmatrix}
x(k) \\ 
y(k) 
\end{bmatrix} =
\begin{bmatrix}
A(k) \cdot sin \left ( w(k) \cdot t + A(k) \right ) \\ 
A(k) \cdot w(k) \cdot cos \left ( w(k) \cdot t + B(k) \right )
 \end{bmatrix}
"/>  -->


