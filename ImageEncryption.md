---
layout: default
title: Home
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
Images, like any other data format can very easily be encrypted using standard algorithms like RSA or AES but those result in a meaningless blob that can't be used anywhere. Sometimes it's worth keeping some properties to allow existing systems that process images to play well with the data.

# Basic Encryption
This step is fairly easy to implement, we can take a block of a few pixels, encrypt them, and interpret the resulting data also as pixels. This allows us to upload the encrypted image on social media or chat apps that restrict file uploads to images. However some apps do additional processing like scaling the image and the new algorithm for encryption doesn't work anymore. When the image is scaled the algorithm usually involves some interpolation so the resulting pixels are different from the starting pixels and the decryption breaks

The knee-jerk reaction to trying to work with encrypted data is [:Homomorphic Encryption](#HomomorphicEncryption). However there are a [:few specific issues](#IssuesWithHomomorphicEncryption) that make this not optimal.

## :x Homomorphic Encryption
Homomorphic Encryption is a scheme where operations can be performed on the encrypted data which corresponds to an operation on the initial data. This can be written as `encrypt(f(a)) = g(encrypt(a))` or in the binary operation case `encrypt(f(a,b)) = g(encrypt(a),encrypt(b))`.

## :x Issues with Homomorphic Encryption
In Homomorphic Encryption there is no control over the algorithm used on the encrypted image, this is both in terms of the actual logic of the algorithm and also that it won't involve the required operators that map from the unencrypted domain to the encrypted domain. Secondly the calculations are usually performed in floating point and converted to integers at the end which doesn't play well with most Homomorphic Encryption schemes

# Problem Statement
Before we move on with the potentials solutions we have to define the problem to be solved first:
  - Encryption must convert images to images.
  - Images must be in a commonly used format preferably the same format as the input image. [:Why?](#whyACommonFormat)
  - The encrypted image must be the same size as the initial image. [:Why?](#whySameSize)
  - When the encrypted image is scaled down and decrypted the result should be a scaled version of the initial image, i.e. `Decrypt(Scale(Encrypt(Img))) = Decrypt(Enc(Scale(Img))) = Scale(Img)`, The above property should hold for most common scaling algorithms.
  - Encryption must be "backed" by a standard encryption algorithm to ensure security.
  - Encryption may be lossy as long as the image is mostly preserved.

## :x Why a Common Format
This is so that the image can be rendered and scaled by the application

## :x Why Same Size
This is so that the user need not waste extra bandwidth for downloading large images just because they are encrypted

# Frequency Domain
Now that we have specified the constraints the most obvious solution is to work with the image in the frequency domain with a [:Direct Cosine Transform](#DCT).	This is because a scaling operation is [:analogous to a crop](#DCTAndScaling) of the top left part of the DCT of the corresponding size.

## :x DCT
DCT is an operation that converts an image to a same shaped matrix of frequencies with the lower frequencies being at the top left and the higher frequencies at the bottom right. The coefficients are real valued unlike a fourier transform and they represent a sum of cosines. It is an invertible operation and theoretically no data is lost however floating point errors will creep in.

## :x DCT and Scaling
The scaling process deletes high frequency data and leaves the low frequency data. When an image is scaled usually the high quality detail is smoothed away leading to the deletion of the high frequency data but the low frequency data which tells more about the image overall is preserved. In other words scaling is basically a resampling of the signal which would lead to the high frequency region to get cutoff.

# The Naïve Encryption
The naïve way to encrypt would be to use AES. This is by taking 2 coefficients at a time and encrypting them. [:Why?](#why2Coefficients) the resulting coefficient can be converted to an image through an inverse DCT to get the final encrypted image
This method has an issue the resulting image will have amplitudes which aren't bounded(we need the image amplitudes to be between 0 and 255).

## :x Why 2 Coefficients
Each coefficient will be represented as a double precision float(double/float64), since AES takes 128 bits as an input two of them together can form the input and the encrypted result of 128 bits can be split back into two floats, while decrypting if any pair is split due to the cropping it has to be neglected but the rest of the image can be decrypted resulting in a slightly smaller image.

# Clever Workaround
The reason that the image has such a high amplitude is that the maximum value that could be stored in the image is very high and as a result a random number picked will have a large amplitude in general. The trick is to realise that data can be stored in the float even if the magnitude of the float is constrained a lot. This can be done by [:exploiting the bits of the floats](#BitManipulation). This will be lossy as it is not possible to encode all the data but the important stuff can be encrypted and stored without compromising the raw amplitude of the double.

## :x Bit Manipulation
The IEEE 754 states that from right to left the bits represent 1 sign bit, 11 for the exponent biased by 1023 and 52 bits of the mantissa. The mantissa doesn't encode a lot of information regarding the image, especially the ones with the lower significance so we will only preserve the first 4. Including the sign and exponent this gives 16 bits which can be encrypted with 8 other values. The encrypted 16 bits are then stored in the MSBs of the mantissa and the rest is filled with a 1 followed by zeroes to prevent floating point errors from corrupting the data. The sign bit is set to 1 and the exponent is set to -1 (1022). The exponent is set to -1 as it ensures the magnitude of the number will be from (64 to 127) or (-127 to -64) which fits inside our bound.

# The Second Hurdle
When this new method is used the resulting image has pixels with values from 0 to 255 but they are all still floating points but we require integers. If we round the floats to the nearest integer the error is too high and it changes the DCT coefficients by enough to change the data stored which changes the critical data stored and renders the image in a corrupted state. [:A feeble attempt using smaller datatypes](#AFeebleAttempt). can be tried to decrease the amount of data stored to make it more resilient to corruption but it is simply not enough.

## :x A Feeble Attempt
Instead of [:manipulating the bits of the floats](#BitManipulation) using float64, using a (half/float16) allows for 1 sign bit, 5 for the exponent biased by 15 and 10 bits of the mantissa. Using 2 bits of the mantissa gives 8 bits which can be encrypted with 16 other values and stored in the first 8 bits of the mantissa.

# Breakthrough
The core issue in the previous attempts was the [:mode of the AES encryption](#AESModes) making it a block cipher. This caused small errors to propagate throughout the data corrupting MSBs. Using a stream cipher ensures that a corruption in any bits only affects those bits and the MSBs which store the crucial information is protected.

## :x AES Modes
AES is a block cipher that takes in 16 bytes of data and encrypts it, however there are multiple ways of using the cipher to encrypt large quantities of data. The most straightforward way is to split the data into 16 byte chunks and encrypt them individually which is known as Electronic Code Book(ECB). This method has weaknesses as it can reveal some patterns about the data. To fix this Cipher Block Chaining(CBC) mode was created where the first block's plain text is XORed with an Initialisation Vector and each succeeding block's plain text is XORed with the previous block's cipher text.

Both of these modes behave like a block cipher but there is a mode called Counter(CTR) which allows the cipher to behave like a stream cipher. This is done by using a Nonce(Number used once) and a counter. The counter and Nonce are concatenated and encrypted with the key. The cipher output is XORed with the plain text to make the cipher text. For the next block the counter is incremented and the process is continued. This allows for changes in the plain text to not spread across the cipher text.

# Small Tweaks
Since the 2D DCT has to be encrypted with a stream cipher it has to be flattened to one dimention and the indexing should not depend on the size of the image. The solution was to take "⅃" shaped nesting sections and encrypting them sequentially so every pixel gets the same index no matter the size of the image. The [:manipulation of the float bits](#BitManipulation2) also has to be modified to ensure most of the information comes through.

## :x Bit Manipulation2
Back again using float32 the sign bit is XORed then the exponent is kept as is to keep the magnitude of the float from increasing too much. The mantissa is shifted to the right and encrypted from the first set bit to ensure that the total float is lesser than the initial value so that the encrypted image is has values between 0-255 and the reverse is done while decrypting.

# Contact me 
satindra.r@gmail.com or on discord at saturn.255

