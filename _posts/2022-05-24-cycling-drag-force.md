---
layout: post
title: "How wind affects your speed on a bike"
description: ""
excerpt: "Let's look at how the wind affect your speed on a bike, the diminishing impact of power, and why you feel the drag suddenly passed a certain speed."
categories: [random-rambling]
tags: [random-rambling, maths]
use_math: true
---

If you are into biking, you've probably noticed that the relative wind that slows you down seem to appear all of a sudden. Slow down a little bit and it seems okay, accelerate back 5 km/h higher, and it feels really painful.

You probably have noticed as well that on some windy days, the way back you go almost 10 km/h faster in average, with the same amount of efforts applied.

Let's quicky look at the physics behind this.


## The Physics

**TLDR;** The power needed to maintain a given speed is cubic in the relative speed of the wind. The energy expenditure over a given distance is quadratic in the relative speed of the wind.

### The drag force

From [Wikipedia](https://en.wikipedia.org/wiki/Drag_(physics)), we get the following equation for the drag force:

$\displaystyle F = {\tfrac {1}{2}}\,\rho \,v^{2}\,C_{D}\,A$

where we have:

* $\rho$  is the density of the fluid
* $v$ is the speed of the object relative to the fluid
* $A$ is the cross sectional area
* $C_{D}$ is the drag coefficient (which depends on the shape)

So the intensity of the force is quadratric in the speed of the relative wind the cyclist is facing.

### The work

The force applies on a given position, and we have to integrate this over the trajectory to get the total **work** of the force.

$\displaystyle W = \int F(x) . dx$

Which, if we assume a straight line trajectory, gives us:

$\displaystyle W = F(x) . \vec{x}$

So we can see that the work done for a given distance grows quadratically with the speed at which we travel relative to the wind.

### The power

Then, to get the **power**, we have to compute at the work done over time (the derivative of the work with respect to time).

$\displaystyle P = \frac{dW}{dt} = F . \frac{d \vec{x}}{dt} = F . \vec{v}$

And so we see that the power is not quadratic with respect to the relative speed of the wind, but cubic instead. To go twice as fast against the wind, you need to generate 8 times the power against the wind, which is quite substantial!


## Consequences (if the drag force dominates)

For speeds at which the drag force dominates over the other forces (typically at high speed, the drag force is the main slowing factor), the following holds.

### Going 10% as fast requires 33% more power

To accelerate from $v_1$ to $v_2 = 1.1 v_1$, the relative increase in power is given by the ratio of $P_2$ divided by $P_1$:

$\displaystyle \frac{P_2}{P_1} = \frac{v_2^3}{v_1^3} = 1.1^3 = 1.331$

So you need to sustain 33.1% more power to gain a mere 10% speed. We can generalize and get a feel of this cubic cost by looking at the following table:

|Relative Speed Increase|Relative Power needed|
|----|-----|
|+10%|+33% |
|+20%|+73% |
|+30%|+120%|
|+40%|+174%|
|+50%|+338%|

### Resting by slowing down might not degrade your average speed so much

This is the same observation as the previous section but here we consider slowing down instead of speeding up:

|Relative Speed Increase|Relative Power needed|
|----|-----|
|-10%|73%  |
|-20%|51%  |
|-30%|34%  |

In short, if I divide my power expenditure by 2 temporarily to catch my breath, my speed will only decrease by 20%, which is not so much. This will not show too much too significantly on my average speed if done for a short period of time.

### Doubling the power only means going 26% faster

We can obviously inverse the question: how fast could I go if I double the amount of power I can sustain? (in the limit where other forces are negligible)

$\displaystyle \frac{P_2}{P_1} = 2 = \frac{v_2^3}{v_1^3} = \frac{f^3 v_1^3}{v_1^3} = f^3$ where $f$ is the speed increase factor

The only real solution of $\sqrt[3]{2}$ is approximatively 1.2599 and so doubling our amount of sustainable power (by increasing our VO2 max, our muscle efficiency with respect to oxygen consumption, our strength, etc) would "only" give us 26% increase in speed. 

### Doing the same parcours 10% faster requires 21% more energy

Because of the work grows quadratically with the relative speed of the wind, for a fixed distance in straight line (and in the limit where we can neglect the effect of other forces than the drag force, for instance at high speed), we have:

$\displaystyle \frac{W_2}{W_1} = \frac{v_2^2}{v_1^2} = f^2$ where $f$ is the speed increase factor

And so if we go 10% faster, $f^2 = 1.1^2 = 1.21$, and so we spend 21% more energy in doing so.

Another way to derivate the same result is considering that power will increase with the cube of $f$, but dividing by $f$ because we have to sustain that power for $f$ less amount of time (since we go through the distance $f$ times faster).

### Calories burned are not only a function of distance

The previous section showed us that merely saying "I did a 40 km bike this week-end" does not really allow to compute for the total energy expenditure (and so the real intensity of the workout).

In fact, someone could have done twice your distance and burn as much calories as you did by going slower. How slower? Let's compute that:

$\displaystyle W \propto v^2 d$ where $v$ is the speed and $d$ the distance

And so if the two persons spent equally as much energy but one person did twice the distance as the other, we have:

$\displaystyle d_1 v_1^2 = d_2 v_2^2$ and $d_1 = 2 d_2$
$\displaystyle \implies \frac{v_2^2}{v_1^2} = 2$

And so the person that did travel half the distance did it $\sqrt{2} \simeq 1.44$ times faster.

### Aerodynanism also hits the same dimishing returns but remains of paramount importance

The power as well as the energy are proportional to the "aerodynamism" factor $K$ as the power follows this equation:

$P = K v^3$ and $W = K v^2$

And so for the same speed, improving aerodynamism by 20% will decrease your power and energy expenditure by the same 20%.

But aerodynamism also hits the same diminishing returns as increasing power when we want to increase our speed.

For instance, for the same amount of sustained power (i.e. the same amount of energy or calories burned over a period of time) improving aerodynamism by 20% leads to going 7.7% faster:

$P_1 = K_1 v_1^3 = P_2 = K_2 v_2^3$ where $K_2 = 0.8 K_1$

$\displaystyle \implies \frac{v_2^3}{v_1^3} = \frac{K_1}{K_2} = \frac{1}{0.8}$
$\displaystyle \implies \frac{v_2}{v_1} = \sqrt[3]{1.25} \simeq 1.077$

Note that aerodynamism still remains of paramount importance, because if you want to increase your speed by 7.7%, you can either improve aerodynamism by 20% or increase your sustained power by 25%.
