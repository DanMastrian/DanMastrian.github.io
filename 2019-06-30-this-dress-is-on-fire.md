---
title: 'This Dress is on Fire'
---
# Motivation

TODO

# Design

I wanted to add enough LEDs to the dress so that I could display animations that synced with the music. I also wanted the LEDs to be hidden, so that the dress looked relatively normal until the big moment partway through the song. And then I wanted it to be bright. Like, crazy, what-the-heck-is-happening bright. It had to be durable enough to withstand an eight-year-old wearing it in the vicinity of grabby friends. And I wanted the whole thing to be battery-powered, for mobility and convenience, and to help keep the electronics hidden until the show started.

I was also mindful of the fact that I *really* didn't want to allow the song title to become ironic. So while I was pretty sure I designed and built it safely enough, I also wanted to make the power box could be quickly disconnected from the actual dress, just in case of an emergency.

# LED matrix

Considering cost, power draw, and size, I aimed for building a matrix of about 600 NeoPixels.

I bought two 5-meter reels of RGBW (RGB + warm white) LEDs, each containing 300 LEDs. I wanted the warm white to give a nice firey glow. And I got the IP67(???) version, with the strip encased in a soft silicone sheath to protect the electronics from the kids, and vice versa.

I looked at a denser version of the strips too, but building a matrix of the same physical dimensions would have exceeded my power budget and my dollar budget by quite a bit.

To use as a test bed before I started cutting up expensive LED strips, I also picked up two pre-built 8x32 RGB matrices, giving me 512 pixels total. Close enough, though the difference became important later.

The hardest part of building the actual matrix was convincing myself that I had chosen the right dimensions to have the matrix fit properly in the dress. I really did not want to get it wrong and waste LEDs, so I had to eventually force myself to commit to the design and start cutting.

I cut right through the silicone and PCB in one push with an Xacto knife. The final layout was 24x24, giving me 576 pixels to work with. I could have added a 25th column, but because the LEDs came in reels of 300, I ended up with two leftover strips of 12 that I would have had to solder together, and it just wasn't worth the extra hassle for one more column. 576 LEDs seemed like plenty.

# So...much...soldering

I like soldering, but this was daunting for me.

I had about 100 solder joints to make on the matrix itself: power, ground, data in, and data out on each of 24 strips. The wires had to be soldered to the copper pads on the strips, some of which were smaller than others if I wasn't precise enough with the Xacto knife earlier.

And I really needed them to be solid joints for two reasons. The first was that they had to survive a wiggly kid wearing them. The second was that I was turning the strips into a matrix in the simple way: as a single chain laid out in a zig-zag pattern, driven by a single data line. This meant if any of the data lines broke, it would hose not just that strip, but all strips that came after it in the chain.

Some of the pads needed desoldering first to remove the JST pigtails that came with the strips. The reel was also made up of 1-meter PCB strips, so every meter there was already a joint where two PCBs were soldered together. Unfortunately the way the math worked out, I had to cut through those when I made my strips of 24. So those had some blobs of solder that needed to be removed before trying to make a new joint with the wires.

I'm not the best at soldering, and was worried I would end up putting too much heat into the components by the time I did all of this. Surprisingly, I didn't end up toasting a single pixel.

I used silicone-insulated stranded wires for everything, because they were very flexible and could resist the abusive heat of my amateur soldering.

The 10-meter total length of the strip meant that I had to provide power at multiple points to avoid voltage sag. It seemed to make the most sense to just power each column from the top, and then take the power lines from each half of the matrix, and form them into a tree, with the trunks being low-gauge 5VDC and ground wires that wrapped around the left and right sides of the dress at the waist to plug into the lighting controller in the back.

I used a tree layout for the power wires, because I had no idea how to make a reasonable 13-way solder joint to combine 12 wires into 1, especially because I wanted the "trunk" wire to continue in the same direction, rather than doubling back in the wrong direction as would happen with a wire nut. I figured I could handle 3-to-1 though, and so I just repeated that a few times. I twisted the wires together, and then wrapped a thin bare solid copper wire around the joint, crimped it tight with pliers, and then filled the whole mess with solder.

Then I hid my shame under some heat-shrink tubing. I was worried that the shrink tubing could slip or break, and I'd have a joint on the 5V line shorting to the joint on the ground line. I really did not want a short occurring on the power lines of a LiPo-powered project. So I made sure that the joints on the 5V lines occurred an inch or so away from the joints on the ground lines.

# Lighting Controller

## Microcontroller: Feather M4 Express

## Power supply

600 NeoPixels drawing an estimated 20mA each put the total power draw at 12A. They could peak much higher than that with all channels at full brightness, but I decided that 12A seemed like a reasonable power budget. 60 watts is a lot more than most dresses consume, anyway.

So I needed a battery that could provide 12A at 5V for at least several minutes. I settled on an RC car battery: a LiPo 2S (7.2V) 2200mAh pack, rated at 30C discharge. I figured that should easily provide the current I needed, and could last roughly 10 minutes at that rate.

[TODO: balance charger]

To drop the voltage back down to the 5V needed by the NeoPixels, I got a [DROK product link] power supply. It was nice and compact, had decent-looking heat sinks and an adjustable output voltage, and could *basically* handle the current draw. It was close enough, and I figured I would only be hitting the peak current for very brief periods.

## Time synchronization

To have graphics synced to the music, the lighting controller needed to know where the beats were in the song. The song runs at a consistent 93 BPM, so the duration of each beat is simple to calculate.

For the first big integration test, I had the matrix display at simple metronome, showing the beats of each measure. This would give easy visual confirmation if the lights were properly synced with the beats in the song. It worked on the first try! Thanks, elementary-school arithmetic!

Well, it sort of worked. The synchronization seemed perfect at the beginning, but by the end of the 150-second song, it was clear that enough drift had occurred that it was now noticeably out-of-sync with the beat.

## NeoPixel DMA

The normal NeoPixel library suspends interrupts while sending data to the NeoPixels, to get the precise timing needed on the data signal. That completely screws up the values returned by the millis() and micros() functions.

I knew that would make it impossible to sync with the music, so I was instead using the NeoPixel_ZeroDMA library, which uses Direct Memory Access to free up the CPU from dealing with sending each data bit to the NeoPixels, allowing interrupts to stay on and the clocks to stay accurate.

But the clock was still drifting, so what the heck?

I poked around on the internet and found that the clock on the microcontroller had about a 1% error. At first glance that seemed low, but over the course of a 150-second song, that's 1.5 seconds, or about half a measure. Very noticeable.

## External RTC/TCXO

Fortunately, I had a [RTC] on hand, which I was planning to use for a different project. It's supposed to be really accurate, but the smallest time slice it tracks is seconds. However, it does have a pin that outputs the raw 32kHz oscillator signal, and that must be accurate too.

I hooked this up to an interrupt on the M4, counted my own ticks (1/32768th of a second), and calculated elapsed milliseconds from that.

Much to my surprise, this worked on the very first try. I just dropped it in, changed where the rest of the code got the elapsed time from, and the synchronization was perfect!

## Scripting/cues
## Radio

A precise clock would be no use if it didn't start at the same time as the music. So I added cheap ($1) little 2.4GHz radios to the controllers; the lighting controller (attached to the dress) would trigger the music to start, and so it would know exactly when that happened.

This was a critical part of the project, because it allowed the timing to be precise without having my daughter or me have to carefully synchronize starting the music and the lights.

However, this also turned out to be the weak link in the project. It worked almost perfectly dozens of times at home, but in each of the four real performances the trigger signal failed to reach the audio controller on the first try, requiring me to reset the devices and try again. In one case, the distance was much greater than what I had tried at home, because I wasn't expecting the audio mixer to be so far from the stage. I also wonder if there was interference from wireless mics or other devices in the room.

Given more time, I would have looked into boosting the broadcast power, or switching to a different radio, or implementing some retry logic on the trigger signal.

But, each time it worked on the second try, so all in all it wasn't that bad.

# Audio Controller

The audio controller was a separate little box that would sit near the venue's audio mixer and plug into it, providing the music playback. It was simple, with just three parts:

- NRF24 radio, to receive the trigger signal
- Adafruit sound board, which played the OGG audio file
- Arduino, to tie the two together

[TODO status lights, buttons]

# Metronome

# Harness

# Strain relief

# Connectors

Mini-XLR