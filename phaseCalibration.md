---
layout: default
title: Phase Calibration
---

<script src="https://ncase.me/nutshell/nutshell.js"></script>
<script>
  Nutshell.setOptions({
    startOnLoad: true,
  });
</script>

This Blog is made using [Nutshell](https://ncase.me/nutshell) click on the [:Special Links](#SpecialLinks) for more context

### :x SpecialLinks
Links with a ':' at the start

# Introduction
For a quite some time now desmos has been used to make simple melodies. The first person I could find was 
[this reddit post by twitter user @aknauft](https://www.reddit.com/r/desmos/comments/b9ut82/music_in_desmos_tetris/). This worked by getting the notes of a melody and creating a graph where the y value was the frequency. A more detailed guide can be found in [this amazing youtube video by Ranny Bergamotte](https://www.youtube.com/watch?v=9BTZiUPcBf0). In 2023 desmos released the `tone()` function that revolutionised music in desmos. It allowed for any sine wave to be played via `tone(frequency, amplitude)`. The natural continuation to this is to try and play some arbitrary music but not many have succeeded in playing audio that is close to the source material.

# Initial Attempts
The first people to try and import audio into desmos ran a fourier transform on the data converting the audio files into a series of frequencies, amplitudes and phases. Then by simply ignoring the phases and playing the frequencies and amplitudes you get a sub-par recreation of the audio. The earliest implementations I could find for this were by [github user alorans](https://github.com/alorans/DesmosAudio), [github user ascpixi](https://github.com/ascpixi/desmos-dft-sampler) and [github user Y0UR-U5ERNAME](https://www.desmos.com/calculator/4kvrynyljo). None of these guarantee perfect reconstruction as the [:cruicial phase information](#whyPhase) is missed out

## :x Why Phase
Phase is normally not detectable by our ears but relative phase is. It is easy to see why this is the case if we play two sine waves of the same frequency. At a phase difference of 0, we can hear the sound clearly but as the phase difference increases the audio gets quieter until a phase of π. When the relative phase is π both the waves cancel out and no audio is heard. Increasing it further makes it louder until a phase of 2π after which is decreases again. The effect of phase on waves of different frequencies is more complex but it changes the timbre of the sound.

# Speculating The Inner Workings
Desmos uses the WebAudioApi to play audio.

# Cracking Phase Calibration
