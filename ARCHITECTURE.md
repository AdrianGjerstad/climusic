# Architecture

This document describes the foundation and inner workings of this program.

## Table of Contents

- [1 | The Sound Engine](#1--the-sound-engine)
  - [1.1 | 64-bit Precision](#11--64-bit-precision)
  - [1.2 | RAM Usage](#12--ram-usage)

## 1 | The Sound Engine

This section describes how the core sound engine works.

### 1.1 | 64-bit Precision

Digital music has a big problem when it comes to being na&iuml;ve in development: volume control. Because of the way digital music works, you output it at any volume level you want. The problem is with who masters to what end level. There are people who aim to get the resultant song or other track to be one quarter of maximum (-12 dBFS), some people who master to half of maximum (-6 dBFS), etc. The problem comes with beginners who don't know the diffference between the scale of decibels you learned about in elementary school, and decibels full scale.

I can even say for myself, that when I first started in audio, I had no idea that 0 dBFS was the maximum volume, so I would frequently try to go louder than that, and it would distort. This is called digital clipping. It occurs when an audio signal has too much amplitude to be sufficiently described within the space it was given. In digital file formats, this occurs when the number representing any individual sample was computed to a value larger than the maximum signed integer limit for the bit depth used in the format.

For example, if I am using a bit depth of 24 bits, I am allowed 3 bytes per sample. So if I compute a sample to be `0x1 23 45 67`, which is larger than can fit in 23-bits (two's complement is used for signed numbers), I must clamp it down to `0x7F FF FF`.

Obviously this is terrible if a file has a clipped waveform, but what about audio processing within the DAW? This is why this program uses 64-bit double precision floating point numbers for representing its samples. I know that that is quite a bit of mouthful, but can often be shortened down to be interpreted (by software enthusiasts at least) as a double data type. This data type uses 64 bits, but also stores data in the format of IEEE-754 (binary representation of numbers with decimal points).

As a result, the maximum and minimum values are, quite literally, more than a googol (100 zeroes), even more than one centillion (300 zeroes), if you know what that word means. This means that we can have an enormous amount of headroom in amplitude space for effect processing, but still clamp it back down with minimal loss when it comes time.

Side note: we are using decimal based arithmetic instead of integer based because of the way the decibel scale works. The decibel scale is a logarithmic scale that has a doubling point around 6 (six). IEEE-754, the standard that defines decimal based arithmetic for computers also makes it so that the larger the number, the less precision you will be able to hold to the right of the decimal point, making its precision logarithmic.

The following text contains the actual definition to be used for implementation.

The engine will use 64-bit floating point representations for samples, as defined in IEEE-754. Its clipping point will occur at &#177;1e0, or `0x3FF0000000000000` in IEEE-754 format. Anything above that point should be handled just like any other value. The only point at which it will make a difference is at the Master OUT pipe, at which it will clamp values down to this point, and convert to integers to be handled by an external destination, be it disk, speakers, or otherwise.

### 1.2 | RAM Usage

Random Access Memory, or RAM for short, is the very component of your computer that allows it to do so much. Typically, you will see laptops around these days with 8 to 16 gigabytes of memory.

Although operating systems and kernels like Linux do some technological trickery to keep the amount of memory in use as low as possible, there is only so much memory a computer might have availble to it. And in walks the entire world of audio software.

As you may have heard before, with terms like bit depth and sample rate, audio takes up a ton of space for super short durations of time. To think about how audio works, we can draw a comparison to how modern video works.

In reality, your computer screen, a YouTube video, etc. are only showing you a series of still images, updating fast enough to make your brain perceive its input as a moving image, hence Motion Pictures. This is exactly what happens with audio, only the images are much smaller, and there are *MANY* more "frames per second."

To draw a direct corellation between audio and video, the size of a frame is your bit depth, and the number of frames you show each second is your sample rate. The name given to an audio frame is a "sample."

Starting off with some math, we know that the engine uses 64-bit (double precision) floating point values. Each number is 8 bytes long, aka, a quadrouple word. The minimum sample rate possible for consumer-grade music to not sound muffled is 44,100 hertz, or 44.1 kilohertz. This is because the average maximum frequency that can be heard by a human ear is 20,000 hertz. The extra 2,050 hertz on the end is there because of the way digital music gets recorded from anologue sources. As you may imagine, that is a story for another day.

So, finally, assuming we are using 64-bit values at a sample rate of 44,100 Hz, a single second of audio takes:

```
8 * 44,100 = 352,800 bytes = ~350 KB
```

Yikes! A single second of audio takes about 350 kilobytes of audio! This is worstened by the fact that most consumer grade music is stereo-based, meaning there is a single channel for each ear. That doubles this calculation to allocate 700 kilobytes of memory for a single second of audio. This doesn't count all of the memory that might be required to do calculations elsewhere in the program, or making more than just a sound effect that lasts 1 second.

Okay so clearly we can't just load our entire program into RAM and live to tell the tale, so we need to try something else. This is where modern digital audio processing happens. In the modern world, computers load what are called sample buffers into RAM to do calculations on. These blocks are often a power of two, and are what your driver is referring to when it asks you for the buffer size you want to use. For the purposes of this calculation, we will use a buffer size of 128 samples.

```
8 * 128 = 1024 = 1 KiB
```

Nice! So just one channel of 64-bit audio in 128-sample chunks only takes up a single kibibyte! As you can imagine, this is heaps better, and also doesn't rely on the sample rate, so, when resampling (upsampling/downsampling), we don't have to do a bunch of re-alignment. Obviously our program can still become this demanding, but it is the user's fault if they try to create 100+ audio sources.

As a result, we have a control within the sound engine that directly affects memory usage and indirectly affects CPU load. This is known as the buffer size, and will be what you use to set your RAM usage.

