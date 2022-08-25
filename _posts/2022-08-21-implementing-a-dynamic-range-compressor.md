---
layout: post
title:  "Implementing a dynamic range compressor"
author: Jaza Syed
date:   2022-08-21
tags: audio dsp
description: "A discussion of digital dynamic range compression"
status: published
---

{%- include mathjax.html -%}

# Introduction
I've recently reminded myself of the basics of digital audio synthesis and DAW-based music production, using Ableton Live's excellent [Learning Synths](https://learningsynths.ableton.com/) and Kadenze's Ableton Live [course](https://www.kadenze.com/courses/sound-production-in-ableton-live-for-musicians-and-artists/info) amongst other resources. One effect I explored in detail is [dynamic range compression](https://en.wikipedia.org/wiki/Dynamic_range_compression) which is used primarily to reduce the relative amplitudes of excessively loud sections of an audio track. I learnt the basics of compression from this [tutorial](https://www.youtube.com/watch?v=iY6qalrsrbM). I discovered that
- compression is poorly explained in introductory literature, especially attack and release
- there is very little information online about how to implement a compressor in practice

This post covers the basics of compression, common misconceptions about it and provides guidance for a simple practical implementation from a mathematical perspective. I refer throughout to Ableton Live's default Compressor plugin as a reference implementation. The process of writing this also inspired a novel approach to attack and release parameters for compression.

# Background
Compression works by applying gain to a signal dependent on the signal's amplitude. The signal amplitude is determined by some [envelope detector](https://en.wikipedia.org/wiki/Envelope_detector). There are many methods of calculating the envelope, which I'm not going to expand upon in this post. If negative gain is applied the signal is _compressed_ and if positive gain is applied the signal is _expanded_. _Sidechaining_ is when a signal other than the signal being compressed or expanded is used to determine the envelope.

Generally speaking we can define the input signal (to be compressed / expanded) $i(t)$, the output signal $c(t)$, the envelope signal $e(t)$ and a transformation function for the signal level $T(e)$ which determines the desired output amplitude for a given envelope amplitude. All signal levels are expressed in decibels. Compressors have options to set the envelope algorithm and $T(e)$. A typical $T(e)$ reduces the signal level linearly above a given threshold:

$$
\begin{equation} 
\label{eq:hardknee}
T(e) = \begin{cases} 
    e & e\leq e_{\text{th}} \\
    (e-e_{\text{th}})/r + e_{\text{th}} & e\gt e_{\text{th}} \\
    \end{cases}
\end{equation} 
$$

The ratio $r$ (where $r>1$ for compression) gives the dB amount the envelope is required to rise to give an increase in output level of 1dB if the envelope level is above the threshold $e_{\text{th}}$. We can then define the output for an extremely simple compressor

$$ \begin{equation} \label{eq:noattack} c(t) = i(t) + (T(e(t)) - e(t)) \end{equation} $$

Note that adding $T(e(t)) - e(t)$ from the signal in dB is equivalent to multiplication by $\frac{T(e(t))}{e(t)}$ if the signal levels were on a linear scale. The quantity $T(e(t)) - e(t)$ is known as the *gain reduction*, which we will refer to as $R(t)$. Most compressors also include a setting called *makeup gain* which is applied to the signal at all times - this is used to restore the average signal amplitude after compression.

Most compressors have a parameter called knee, which controls how sharp $T(e)$ is around the threshold. This is typically set in dB and corresponds to the range of the input level over which the compressor transitions to full operation. A knee of 0dB as in \eqref{eq:hardknee} is known as a *hard knee*.

Varying the ratio with a hard knee | Soft knee 
---|---
![](/img/compression/audio-compressor_large.webp) |![](/img/compression/Knee.png)

Compressors do not apply the gain reduction instantly as in \eqref{eq:noattack}. They have parameters called *attack* and *release* which are typically specified in milliseconds that control how quickly the gain reduction is changed when the signal level changes.

The definitions given for these are typically similar to those given in the [Ableton Live reference manual](https://www.ableton.com/en/manual/live-audio-effect-reference/#24-9-compressor):
- Attack defines how long it takes to reach maximum compression once a signal exceeds the threshold
- Release sets how long it takes for the compressor to return to normal operation after the signal falls below the threshold

While these definitions are strictly speaking (mostly) true, they do not provide a good intuition for what the attack and release actually do. They suggest that attack and release are events that occur in the compressor when the signal rises or falls above the threshold, which is not true. We can demonstrate this with the GUI of the compressor in Ableton Live, compressing a signal that alternates between two notes at different levels above the threshold and varying the attack and release times.

Equal attack and release times | Higher attack time | Higher release time
--- | --- | ---
![](/img/compression/equal-attack-release.png)|![](/img/compression/longer-attack.png)|![](/img/compression/longer-release.png)

The yellow line shows the applied gain reduction, the blue line the threshold and the grey silhouette the input signal level. All levels are in dB with 0dB at the grey dotted line. Each note is held for 1000ms.
In both examples, the signal stays entirely above the threshold. Attack and release however still both affect the compressor's behaviour! Comparing the left and middle screenshots, we can see that increasing the attack time increases the time taken for the gain reduction to increase in magnitude when the envelope level increases. Correspondingly, comparing the left and right screenshots, we can see that increasing the release time increases the time taken for gain reduction to decrease in magnitude when the envelope level decreases. We can therefore understand the attack and release as follows

- Attack controls how fast the signal level is reduced to reach the target
- Release controls how fast the signal level is increased to reach the target

The "standard" definitions given in the Live manual make sense now as defining special cases of attack and release, where the envelope level is a step function crossing the threshold. It is still however not completely true as it suggests that the target compression is reached after the attack or release time - we can observe from the gain reduction curves only a fraction is reached within the this time!

# Implementing a compressor

From Google searching I couldn't find much information on how to implement a compressor digitally *including attack and release* until I found the [presentation](http://c4dm.eecs.qmul.ac.uk/audioengineering/compressors/documents/Reiss-Tutorialondynamicrangecompression.pdf) mentioned in the introduction (which is not easily accessible). I tried to come up with an approach myself before I found that resource which follows below.

A typical definition of attack time is the time is the time taken for the level reduction to reach a given fraction (normally two thirds) of the target level reduction. A given time period for a given fractional reduction suggests an underlying mechanism of _exponential decay_ towards the target value. The screenshots of the gain reduction dynamics from Ableton Live's compressor also look like exponential decay. 

We define the actual gain reduction as $R'(t)$ (in contrast to the target gain reduction $R(t)$). Given times $t_1$ and $t_2$ where $t_2 > t_1$ we want to define an exponential decay of $R(t_2)$ towards the target $R'(t_2)$ from the current value $R(t_1)$. We can therefore define for some constant $C$ and a decay time constant $\tau$

$$
\begin{align*} 
R'(t_1) - R(t_2) = Ce^{-\frac{t_1}{\tau}} \\ 
R'(t_2) - R(t_2) = Ce^{-\frac{t_2}{\tau}} \\ 
\end{align*}
$$

This covers both attack and release as $C$ can be negative or positive. Rearranging for $R'(t_2)$ and eliminating $C$ gives the iterative update rule

$$
\begin{equation} R'(t_2) = (R'(t_1) - R(t_2))e^{-\frac{t_2 - t_1}{\tau}} + R'(t_2) \end{equation}
$$

where for attack time $\tau_A$ and release time $\tau_B$

$$
\tau = \begin{cases} 
  \tau_A & R(t_2) \lt R'(t_1) \\
  \tau_R & R(t_2) \gt R'(t_1) \\
  \end{cases}
$$
 
As the gain reduction is always less than 0 for the compression case (as opposed to expansion), attack is when the target gain reduction is lower than the current value so more negative gain has to be applied. We can therefore define the output signal:

$$ \begin{equation} c(t) = i(t) + R'(t) \end{equation} $$

The approach I derived is equivalent to simply applying an exponential smoothing filter to the envelope (where the smoothing factor changes depending whether the envelope is rising or falling) as suggested on page 13 of the QMUL presentation, then scaling the input as in \eqref{eq:noattack}.

# Greater control of gain reduction dynamics

Learning about this gave me an idea - options for adjusting compression time dynamics seem quite limited. What if there were more options for attack and release behaviour than just a single time constant and operation modes? What if a producer could even visually design the shape of the decay behaviour towards the target? 

<div markdown="1" style="margin-left:auto;margin-right:auto;">

|![](/img/compression/linear-mode.png)|
|---|
|Attack and release behaviour in linear mode|

</div>

Looking back to Ableton Live's compressor, there are two envelope modes: *Logarithmic* and *Linear*. In linear mode the dynamics are no longer simple exponential decay. When the compressor attacks, the gain reduction increases very quickly and then decays more smoothly to the target value. When the compressor release, the gain reduction decreases linearly for a while and then decays more smoothly to the target value. This suggests the algorithm uses a different method for determining the iterative update depending on the how far the compressor is from the target gain reduction.

Single attack and release parameters allow the user to define different time dynamics when the gain reduction is above or below the target. Going back to exponential decay, my proposal is that this can be generalised to multiple time constants for different ranges of difference from the target. The ranges can be expressed in terms of the ratio of the gain reduction error to the current target value. We can define the current proportional error $f(t_2)$

$$f(t_2) = \frac{R(t_2) - R'(t_1)}{R(t_2)} $$

The decay rate can then be varied as a function of this quantity. For example we can define attack and release with two phases, with the phase threshold at half of the target gain reduction:

$$
\tau = \begin{cases} 
  \tau'_A & f(t_2) \le -0.5 \\
  \tau_A & -0.5 \lt f(t_2) \le 0 \\
  \tau_R &  0 \lt f(t_2) \le 0.5 \\
  \tau'_R &   f(t_2) > 0.5 \\
  \end{cases}
$$

There could be more than four ranges and the thresholds on $f(t_2)$ could be varied as well. The resulting shapes of the attack and release could be represented and edited visually, which would enable a user to program the exact compression dynamics they want. I think this would be particularly useful for creating rhythmic effects in a sidechain scenario.

# Project idea
I decided in order to learn more about implementing audio DSP in practice an interesting project would be to implement a compressor similar in spirit to Ableton Live's compressor. A user could upload a sound file and loop it while adjusting the compression parameters and observing the behaviour changes as they would in a DAW plugin. It could also include some implementation of the extra attack and release parameters described above and even a graphical editor to do this intuitively.

I came across an interesting project [web-synth](web-synth) by github user [Ameobea](https://github.com/ameobea/) which is browser-based music production tool with a really interesting tech stack. It has a Typescript based frontend which uses the WebAudio API to call custom DSP code written in Rust and compiled to WebAssembly to maintain performance. Using this stack to make the demo seems like a good way of simultaneously building an accessible browser-based tool while also writing performant low-level code for the core algorithm. Some further steps to take are:

1. Prototype the compression algorithm
2. Prototype a frontend in Typescript
3. Try and get the two working together

