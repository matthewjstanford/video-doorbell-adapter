# Doorbell Circuit

This document describes the doorbell circuit project, why it is needed, and how I propose to solve the problem.

## Current Configuration

As of the start of this project the configuration is:

* Two Ubiquiti Doorbell buttons deployed, one at the front door, one at the side door
* The transformer is 3016MT 16V 30VA Doorbell Transformer
* Both doorbells are configured to use the doorbell adapter
* Chime is Broan mechanical chime
* Doorbell buttons are connected to the network via WiFi

### Theory of Operation

This is my understanding of how the system operates.

1. 16VAC is supplied by the transformer to the chimes
1. The doorbell button is connected to the chimes so that the current being supplied to the doorbell button passes through the chime coil.
1. Under normal conditions (doorbell button is not pushed) the doorbell button can draw a small amount of current (how much?) and not activate the chime.
1. When the button is pressed a relay is closed that shorts the wires in the doorbell button. This causes a large amount of current to flow through the circuit, which activates the chime coil.
1. The doorbell buttons contain a small battery that allows it to handle surges of power draw without activating the chime.

## Problem

There are two distinct issues that I believe can both be solved with a single circuit design.

1. Under periods of sustained high load (on very cold days in the winter, when the battery is presumably underperforming) the additional current draw will cause the chimes to hum. The hum is the striker vibrating within the coil.
1. I would like to replace the cheap Broan mechanical chimes with a much nicer 8 tone chime. These type of chimes are not compatible with the Ubiquiti style doorbell buttons.
    1. The chime is connected directly to the transformer.
    1. Activating the chime requires shorting a pin to one of the transformer leads. Since current isn't flowing continually through the doorbell buttons it can't work with a video doorbell. (See wiring diagram below)
    1. Since the trigger mechanism is quite different (current draw vs shorting a pin) there is no direct way to get the Ubiquiti buttons to work with the new chime.
    
**Simple Chime Wiring**

![Simple Chime Wiring](https://github.com/matthewjstanford/video-doorbell-adapter/blob/master/images/simple%20doorbell%20chime%20wiring.jpg "Simple Chime Wiring")

**New Chime Wiring**

![NuTone Chime Wiring](https://github.com/matthewjstanford/video-doorbell-adapter/blob/master/images/NuTone%20Doorbell%20Wiring%20Diagram.png "NuTone Chime Wiring")

**UniFi Protect Wiring**

![UniFi Protect Wiring](https://github.com/matthewjstanford/video-doorbell-adapter/blob/master/images/UniFi%20Protect%20Wiring%20Diagram.png "UniFi Protect Wiring")

## Idea

The general idea is to design a circuit that will sit in between the chime and the doorbell buttons. This circuit will constantly supply the buttons with current, and then will detect when current draw exceeds a threshold. Upon triggering the circuit will close a relay for a short period of time. The relay pins can be used to activate the Broan style current chime, or can be used to short the pin on the new style chime.

## Goals

Net power consumption on this circuit is a bit of a concern. Each additional doorbell button adds more load. The 30VA transformer should be quite sufficient for two doorbells and either chime configuration. We should keep the power consumption of the new circuit to a minimum.

I'd like to do this without using a microcontroller of any sort. We should be able to design this circuit with discrete components and build adjustability into things with potentiometers.

## Unknowns

The Ubiquiti doorbell buttons come with "doorbell adapters" that go in parallel with the doorbell button itself. What do those do? Are the capacitors?

How many doorbell buttons/chime configurations should the design support? It should support at least two, but should it support 3 or 4? In case more are added?

## Circuit Design

* A current transformer will be used to measure the current that is sent to the doorbell button.
* The CT output will be rectified into DC (peak detector) and fed into an OpAmp.
* The OpAmp will be configured as an inverting comparator with a potentiometer on the Vref to make the threshold tunable.
* The output of the OpAmp will be fed into a 555 timer in Monostable configuration.
* The 555 timer output can directly actuate a relay.
* Most of the circuit design is DC, so we will need to convert from 16VAC to 12VDC. Since power consumption is a primary goal we should use a buck/switching power supply. A switching regulator like the LM2575 should work.

## Discrete Components

And a brief description of why these specific components were selected.

* OpAmp - LM358
    * I had a bunch of these from a previous project.
* Switching Regulator - LM2575-12WT
    * A switching regulator was selected for power efficiency sake, and 12V was selected because it is near the input AC voltage and also because we will need to operate relays. Lower voltage relays are available, but we want to manage current by using higher voltage.
* Current Transformer - CT05-1000
    * This was selected for its small form factor and turn ratio. 1000:1. A 1A primary current will result in 1mA secondary.
* Burden Resistor - 6k (1A (est)/1000 = 1mA; 6V (half of 12V) / 1mA = 6k)
    * Assuming the doorbell will consume 1A when the button is pressed, we want around a 6V voltage spike as input to the OpAmp.

## Things I learned while talking with ChatGPT

* The LM358 is a "single source" OpAmp, so it does not need a virtual ground
* A simple peak detector, like what we are considering here, should work great
* A 16 VAC source (our Doorbell transformer) will be close to 21 VDC once rectified. Using LM2575 to regulate it to 12 V should be totally fine.
* A linear voltage regulator is sometimes required after a switching regulator like the LM2575. This is because the switching regulator can introduce noise. In our case, however, this likely won't be an issue. No linear regulator should be required.

## Sub Assemblies

### Power Supply

### Peak Detector and Amplifier

### Timer

The timer portion of this circuit is necessary so that we can simulate a normal "press" of a doorbell button. This corresponds to the amount of time the relay stays closed.

The target should be around how long a doorbell button is normally held in the pressed position. But how long is that? Based on my own observation I suspect this is somewhere between 1/2 and 1 second.

As such I want the timer to stay activated somewhere in that range, but it should also be adjustable.

From the datasheet:

> For mono-stable operation, any of these timers can be connected as shown in Figure 9. If the output is low, application of a negative-going pulse to the trigger (TRIG) sets the flip-flop (Q goes low), drives the output high, and turns off Q1. Capacitor C then is charged through RA until the voltage across the capacitor reaches the threshold voltage of the threshold (THRES) input. If TRIG has returned to a high level, the output of the threshold comparator resets the flip-flop (Q goes high), drives the output low, and discharges C through Q1.

> Monostable operation is initiated when TRIG voltage falls below the trigger threshold. Once initiated, the sequence ends only if TRIG is high for at least 10 μs before the end of the timing interval. When the trigger is grounded, the comparator storage time can be as long as 10 μs, which limits the minimum monostable pulse width to 10 μs. Because of the threshold level and saturation voltage of Q1, the output pulse duration is approximately tw = 1.1RAC. Figure 11 is a plot of the time constant for various values of RA and C. The threshold levels and charge rates both are directly proportional to the supply voltage, VCC. The timing interval is, therefore, independent of the supply voltage, so long as the supply voltage is constant during the time interval.
Applying a negative-going trigger pulse simultaneously to RESET and TRIG during the timing interval discharges C and reinitiates the cycle, commencing on the positive edge of the reset pulse. The output is held low as long as the reset pulse is low. To prevent false triggering, when RESET is not used, it should be connected to VCC.

![Monostable sample circuit](https://github.com/matthewjstanford/video-doorbell-adapter/blob/master/images/555%20monostable%20diagram.png "555 Monostable Schematic")

![Pulse duration](https://github.com/matthewjstanford/video-doorbell-adapter/blob/master/images/555%20pulse%20duration.png "Pulse duration")

Using these charts we can see that with a 10uF capacitor and a 100k resistor (Ra in diagram) we should be in the neighborhood of 1 second pulse duration. Varying the resistance with a 100k potentiometer will allow us to make adjustments to reduce the pulse duration.

## Construction Steps

1. 