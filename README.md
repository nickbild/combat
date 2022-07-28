# Combat AI

The 2-player only Atari classic Combat can now be played against an AI opponent. It runs entirely on a 2600 — no external processing or sensors involved!

![](https://github.com/nickbild/combat/raw/main/media/combat_ai.png)

## How It Works

### Brief Summary

I started with a well-known disassembly of Combat (by Harry Dodgson, later improved by Nick Bensema and Roger Williams).  To this, I added a K-nearest neighbor algorithm that uses examples from a training dataset to make a prediction about the best course of action for the AI-controlled tank to take.

### Details

There were a number of problems to overcome to fit an AI algorithm into an already tightly-packed 2 KB ROM on the highly resource-constrained Atari 2600.  First, a bit about the platform — the Atari 2600 has an 8-bit MOS 6507 CPU running at 1.19 MHz and a paltry 128 bytes of RAM.  Most of the processing cycles are consumed by drawing graphics to the screen.  There is no video buffer; CPU instructions must be in sync with the display, and pixel information is specified as it is needed.  This means that nearly all game logic (handling joystick input, collision detection, firing missiles, sound, etc.) needs to fit into the vertical blanking (VBLANK) period when nothing is being drawn to the screen.

Unfortunately, there are not many cycles available during the VBLANK cycle because Combat needs them to, well, play Combat.  To make this work, I needed more cycles than are available in a VBLANK, so using the leftover cycles was out of the question.  My solution was to hijack the entire VBLANK for 3 frames out of each second to run my machine learning algorithm.  Since the 2600 displays 30 frames per second, that still allowed the normal game logic to run 27 (out of 30) times each second.  I also spread these interruptions around evenly (not consecutive frames), so they are completely transparent.  No difference in response times can be detected between original Combat and Combat AI.

That gave me enough cycles, but all of the additional instructions and data could not fit in the 2 KB ROM.  This was an easy fix though, the 6507 can address 4KB ROMs, and many Atari cartridges were 4 KB in size (or even 8 KB through bank switching), so I could just modify the code to extend the ROM size.

In order to predict the best action for the AI player to take, I needed to know the X- and Y-coordinates of each tank.  The Y-coordinates are already tracked, so that was no problem, but the Atari does not track absolute X positions of sprites.  Instead, sprites are moved relative to their current position (e.g.: move 5 pixels left).  To get around this, I needed to create variables to track each time an X-position was updated in any way, and keep a running total of all changes.  To fit this in with the normal game logic I needed to reclaim some cycles, so I disabled engine sounds (which take a surprising amount of instructions), and a few things that were specific to non-tank game modes.

Finally, I needed to insert some code to simulate joystick inputs for player 2 based on the predictions made by the machine learning model.  With the code I had already removed, I had just enough cycles to slide this in.  The result is a 1-player version of Combat, in which the player 2 tank will aggressively pursue you and fire missiles in your direction when in range.  I focused entirely on the tank game (variation 1), so planes and jets are not presently supported.

## Media

## Bill of Materials

## About the Author

[Nick A. Bild, MS](https://nickbild79.firebaseapp.com/#!/)
