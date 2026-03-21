---
layout: post
title: Desmos Music and Vernier Stopwatches
---
For the first time any audio can be played on desmos.
<iframe src="https://www.desmos.com/calculator/tmqsxilkye" width="100%" style="min-height:75vh"></iframe>
</br>
<div style="text-align: center;">
  <img src="{{ '/assets/callipers.png' | relative_url }}" alt="Vernier callipers measuring a stopwatch" width="50%">
</div>

# Introduction
For a quite some time now desmos has been used to make simple melodies. The first person I could find was 
[this reddit post by twitter user @aknauft](https://www.reddit.com/r/desmos/comments/b9ut82/music_in_desmos_tetris/). This worked by getting the notes of a melody and creating a graph where the y value was the frequency. A more detailed guide can be found in [this amazing youtube video by Ranny Bergamotte](https://www.youtube.com/watch?v=9BTZiUPcBf0). In 2023 desmos released the `tone()` function that revolutionised music in desmos. It allowed for any sine wave to be played via `tone(frequency, amplitude)`. The natural continuation to this is to try and play some arbitrary music but not many have succeeded in playing audio that is close to the source material.

# Initial Attempts
The first people to try and import audio into desmos ran a fourier transform on the data converting the audio files into a series of frequencies, amplitudes and phases. Then by simply ignoring the phases and playing the frequencies and amplitudes you get a sub-par recreation of the audio. The earliest implementations I could find for this were by [github user alorans](https://github.com/alorans/DesmosAudio), [github user ascpixi](https://github.com/ascpixi/desmos-dft-sampler) and [github user Y0UR-U5ERNAME](https://www.desmos.com/calculator/4kvrynyljo). None of these guarantee perfect reconstruction as the cruicial phase information is missed out.

## Why Phase?
Phase is normally not detectable by our ears but relative phase is. For waves of the same frequency it is only detectable as a change in the volume but the effect of phase on waves of different frequencies is more complex as it changes the timbre of the sound.

# Cracking Phase Calibration



## WebAudio API
The JS WebAudio API has some quirks in the way it operates. An inspection of the source code reveals that when the frequency of a tone is changed the phase isn't altered presumably to avoid popping or crackling when the frequency is altered. This behaviour can be exploited by "calibrating" the phase by playing some audio which sets the phase to the intended value and then playing the tone at the desired frequency and amplitude.

## Short Delay Approach
Since phase changes over time we can just wait for a short period of time however this time delay is too small and the builtin desmos ticker can only fire every 20ms or so.

## Relative Phase Approach
Since we can only work in large intervals of time we can borrow a page from the vernier callipers. By using two scales of different but similar intervals we can measure values smaller than either of the scales. Similarly by using two `tone()` functions at different frequencies we can move the phase by a small amount by waiting a longer time. By changing the frequencies we can get a different phase difference 
