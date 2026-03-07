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

# The Inner Workings
Desmos probably uses an OscillatorNode from the WebAudio API to play audio. The API allows for the frequency of the played tone to be modified while it is running. Let's have a look at some snippets of the SpiderMonkey engine by the Mozilla foundation.

```
void ComputeSine(float* aOutput, uint32_t aStart, uint32_t aEnd,
                   const float aFrequency[WEBAUDIO_BLOCK_SIZE],
                   const float aDetune[WEBAUDIO_BLOCK_SIZE]) {
    for (uint32_t i = aStart; i < aEnd; ++i) {
      // We ignore the return value, changing the frequency has no impact on
      // performances here.
      UpdateParametersIfNeeded(i, aFrequency, aDetune);

      aOutput[i] = fdlibm_sinf(mPhase);

      IncrementPhase();
    }
  }
```
This looks fairly simple so let's go into the two important functions used here.
```
  bool UpdateParametersIfNeeded(size_t aIndexInBlock,
                                const float aFrequency[WEBAUDIO_BLOCK_SIZE],
                                const float aDetune[WEBAUDIO_BLOCK_SIZE]) {
    ... //[Compute finalFrequency

    if (finalFrequency == mFinalFrequency) {
      return false;
    }

    mFinalFrequency = finalFrequency;
    mPhaseIncrement = 2 * M_PI * finalFrequency / mSource->mSampleRate;
    return true;
  }

 void IncrementPhase() {
    const float twoPiFloat = float(2 * M_PI);
    mPhase += mPhaseIncrement;
    if (mPhase > twoPiFloat) {
      mPhase -= twoPiFloat;
    } else if (mPhase < -twoPiFloat) {
      mPhase += twoPiFloat;
    }
  }
```

From this it is easy to understand that changing the frequency does not reset the phase of the playing wave but instead keeps sample value(`aOutput[i]`) constant at the time of switch.

This is important as when the frequency of the `tone()` in desmos is modified the same procedure is carried out.

# Cracking Phase Calibration
To set the internal phase of the oscillator we could simply wait for a very short delay before starting each tone. Let's try and calculate this delay

## Short Delay Approach

Taking the frequency as 440Hz and a phase shift of π, we need to find the delay(dt) [:which comes out to be 1/880 of a second](#math1) or about 1.136 ms

This amount of accuracy or precision is too much to be expected as the desmos ticker usually fire every 20ms

<iframe src="https://www.desmos.com/calculator/ir0bh5nlq3" width="50%" height="50%"></iframe>

### :x math1

```
sin(2π * 440 * (t - dt)) = sin(2π * 440 * t + π)
sin(2π * 440 * t - 2π * 440 * dt) = sin(2π * 440 * t + π)
2π * 440 * t - 2π * 440 * dt + 2nπ = 2π * 440 * t + π
2nπ - 2π * 440 * dt = π
2n - 880 * dt = 1
```
for n = 1,
```
2 - 880 * dt = 1
1 = 880 * dt
dt = 1/880
```

## Beats Approach
