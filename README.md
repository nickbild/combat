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

### Code

The Atari that we know and love may have went out of business many years ago, but the brand has traded hands several times since then.  Since the properties have not been abandoned, I cannot post the full source code.  What I will do instead, is post my additions to the code, along with the byte position of the ROM that they should be inserted into (using the well-known disassembly mentioned above as the reference, so that the byte positions and label names are sure to match up).  Anyone that already owns Combat can then use that disassembly as a starting point to convert it to Combat AI.

* Define the following variable names:

```
P0_X    =     $E6
P0_HM   =     $E7
P1_X    =     $E8
P1_HM   =     $E9
P_TEMP   =    $EA
MIN_SCORE_LO =   $EB
MIN_SCORE_HI =  $EC
V_COUNT    =  $ED
CUR_SCORE_LO =   $EE
CUR_SCORE_HI =  $EF
PREDICT = $F0
PREDICT_NUM = $F1
```

* After byte $f017, insert:

```
inc V_COUNT
	lda V_COUNT
	cmp #1
	beq Prediction1
	cmp #7
	beq Prediction2
	cmp #15
	beq Prediction3
	jmp DoNormalLogic

Prediction1
	JSR  SCROT              ; Calculate Score Offsets
							; Score display gets screwed up if this
							; isn't included in the logic skip frames.
	
	jsr RunPrediction		; Run K-nearest neighbor algorithm
							; to calculate best heading for P1.

	jmp SkipNormalLogic

Prediction2
	JSR  SCROT              ; Calculate Score Offsets
							; Score display gets screwed up if this
							; isn't included in the logic skip frames.
	
	jsr RunPrediction2		; Run K-nearest neighbor algorithm
							; to calculate best heading for P1.

	jmp SkipNormalLogic

Prediction3
	lda #00
	sta V_COUNT
	
	JSR  SCROT              ; Calculate Score Offsets
							; Score display gets screwed up if this
							; isn't included in the logic skip frames.
	
	jsr RunPrediction3		; Run K-nearest neighbor algorithm
							; to calculate best heading for P1.

	jmp SkipNormalLogic

DoNormalLogic
```

* Before byte $f02c, insert:

```
SkipNormalLogic
```

* Before byte $f036, insert:

```
; NAB - Reset horiz. movement trackers (HMCLR above).
	lda #$00
	sta P0_HM
	sta P1_HM
```

* Before byte $f05c, insert:

```
; NAB - Update player X position tracking (HMOVE above).
	pha

	lda P0_HM
	lsr
	lsr
	lsr
	lsr
	sta P_TEMP

	cmp #$08
	bcs NotMovingLeftP0
	; Moving left
	lda P0_X
	sec
	sbc P_TEMP
	sta P0_X
	jmp DoneMovingP0
NotMovingLeftP0
	; Moving right
	lda #$10
	sec
	sbc P_TEMP
	clc
	adc P0_X
	sta P0_X
DoneMovingP0

	lda P1_HM
	lsr
	lsr
	lsr
	lsr
	sta P_TEMP

	cmp #$08
	bcs NotMovingLeftP1
	; Moving left
	lda P1_X
	sec
	sbc P_TEMP
	sta P1_X
	jmp DoneMovingP1
NotMovingLeftP1
	; Moving right
	lda #$10
	sec
	sbc P_TEMP
	clc
	adc P1_X
	sta P1_X
DoneMovingP1

	pla
```

* Before byte $f27c, insert:

```
; NAB - Track horiz. movement register.
	cpx #$00
	bne NotP0
	sta P0_HM
NotP0
	cpx #$01
	bne NotP1
	sta P1_HM
NotP1
```

* Comment out bytes $f2f6 - $f2f8

* Before byte $f320, insert:

```
; NAB: Adjust P1 position here.
	ldy PREDICT
	sty DIRECTN+1

	cpx #$01
	bne MoveNotP1
	BIT  GameOn            ; LO nibble.
	BMI  NoFreezeJSAuto        ; Branch if game on (via bit 7).
	jmp MoveNotP1
NoFreezeJSAuto
	lda #$01
MoveNotP1
```

* Before byte $f3bf, insert:

```
; NAB: Control P1 missle launches.
	cpx #$01
	bne DoNotAutoLaunch
	lda MIN_SCORE_HI
	bne DoNotAutoLaunch
	lda MIN_SCORE_LO
	cmp #$30
	bcc DoNotAutoLaunch
	jmp Launch
DoNotAutoLaunch
```

* At $f410, insert:

```
rts
```

* Comment out bytes $f3bf - $f443.

* Comment out bytes $f4e4 - f4e6.

* Comment out bytes $f53a - $f547.

* Comment out bytes $f59f - f5a6.

* Before byte $f5bd, insert:

```
; NAB: Initial player X positions (for tracking).
	lda #11
	sta P0_X
	lda #143
	sta P1_X

	lda #$08
	sta PREDICT
```

* After byte $f5be, insert:

```
; NAB: Reset player horizontal motion trackers and vertical screen line count.
	sta P0_HM
	sta P1_HM
	sta V_COUNT
```

* After byte $f5c5, insert:

```
; NAB: machine learning algorithm.
RunPrediction
	; Find nearest neighbor.
	ldy #255
	sty MIN_SCORE_LO
	sty MIN_SCORE_HI

	ldy #13
	jmp NNloop

RunPrediction2
	ldy #24
	jmp NNloop

RunPrediction3
	ldy #36

NNloop

	lda #0
	sta CUR_SCORE_LO
	sta CUR_SCORE_HI

	; Subtract X
	
	; X0
	lda TrainXp0,y
	cmp P0_X
	bcs X0isLess		; train < actual
	lda P0_X
	sec
	sbc TrainXp0,y
	sta CUR_SCORE_LO	; X0 difference
	jmp DoneX0
X0isLess
	sec
	sbc P0_X
	sta CUR_SCORE_LO	; X0 difference
DoneX0

	; X1
	lda TrainXp1,y
	cmp P1_X
	bcs X1isLess		; train < actual
	lda P1_X
	sec
	sbc TrainXp1,y		; X1 difference
		
	; X0+X1 difference
	clc
	adc CUR_SCORE_LO
	sta CUR_SCORE_LO
	lda #0
	adc CUR_SCORE_HI
	sta CUR_SCORE_HI
	
	jmp DoneX1
X1isLess
	sec
	sbc P1_X			; X1 difference
	
	; X0+X1 difference
	clc
	adc CUR_SCORE_LO
	sta CUR_SCORE_LO
	lda #0
	adc CUR_SCORE_HI
	sta CUR_SCORE_HI

DoneX1

	; Subtract Y

	; Y0
	lda TrainYp0,y
	cmp TankY0
	bcs Y0isLess		; train < actual
	lda TankY0
	sec
	sbc TrainYp0,y		; Y0 difference
	
	; X0+X1+Y0 difference
	clc
	adc CUR_SCORE_LO
	sta CUR_SCORE_LO
	lda #0
	adc CUR_SCORE_HI
	sta CUR_SCORE_HI

	jmp DoneY0
Y0isLess
	sec
	sbc TankY0			; Y0 difference
	
	; X0+X1+Y0 difference
	clc
	adc CUR_SCORE_LO
	sta CUR_SCORE_LO
	lda #0
	adc CUR_SCORE_HI
	sta CUR_SCORE_HI

DoneY0

	; Y1
	lda TrainYp1,y
	cmp TankY1
	bcs Y1isLess		; train < actual
	lda TankY1
	sec
	sbc TrainYp1,y		; Y1 difference
		
	; X0+X1+Y0+Y1 difference
	clc
	adc CUR_SCORE_LO
	sta CUR_SCORE_LO
	lda #0
	adc CUR_SCORE_HI
	sta CUR_SCORE_HI
	
	jmp DoneY1
Y1isLess
	sec
	sbc TankY1			; Y1 difference
	
	; X0+X1+Y0+Y1 difference
	clc
	adc CUR_SCORE_LO
	sta CUR_SCORE_LO
	lda #0
	adc CUR_SCORE_HI
	sta CUR_SCORE_HI

DoneY1

	; Compare current score with minimum score.
	; If less, save it as the minimum and remember the prediction class.
	
	LDA CUR_SCORE_HI  ; compare high bytes
	CMP MIN_SCORE_HI
	BCC ScoreLower ; if NUM1H < NUM2H then NUM1 < NUM2
	BNE ScoreHigher ; if NUM1H <> NUM2H then NUM1 > NUM2 (so NUM1 >= NUM2)
	LDA CUR_SCORE_LO  ; compare low bytes
	CMP MIN_SCORE_LO
	BCC ScoreLower ; if NUM1L < NUM2L then NUM1 < NUM2
	jmp ScoreHigher

ScoreLower
	lda CUR_SCORE_LO
	sta MIN_SCORE_LO
	lda CUR_SCORE_HI
	sta MIN_SCORE_HI
	sty PREDICT_NUM
ScoreHigher

	dey
	beq DonePrediction ; For first time through.
	cpy #13
	beq DonePrediction2 ; For second time through (second set of data).
	cpy #25
	beq DonePrediction3 ; For third time through (third set of data).
	jmp NNloop
DonePrediction3

	; Store best predicted direction.
	ldy PREDICT_NUM
	lda TrainClass,y
	sta PREDICT

DonePrediction
DonePrediction2

	rts
```

* After byte $f7f4, insert:

```
; NAB: Training data.
TrainXp0
	.byte 255
	.byte 0
	.byte 0
	.byte 0
	.byte 0
	.byte 0
	.byte 0
	.byte 0
	.byte 0
	.byte 0
	.byte 0
	.byte 0
	.byte 0
	.byte 0
	.byte 0
	.byte 0
	.byte 0
	.byte 0
	.byte 0
	.byte 0
	.byte 0
	.byte 0
	.byte 0
	.byte 0
	.byte 0
	.byte 0
	.byte $3e
	.byte $26
	.byte $33
	.byte $3c
	.byte $18
	.byte $2b
	.byte $2a
	.byte $34
	.byte $26
	.byte $31
	.byte $1c

TrainYp0
	.byte 255
	.byte 56
	.byte 56
	.byte 56
	.byte 56
	.byte 56
	.byte 95
	.byte 95
	.byte 95
	.byte 95
	.byte 95
	.byte 134
	.byte 134
	.byte 134
	.byte 134
	.byte 134
	.byte 173
	.byte 173
	.byte 173
	.byte 173
	.byte 173
	.byte 212
	.byte 212
	.byte 212
	.byte 212
	.byte 212
	.byte $4e
	.byte $46
	.byte $70
	.byte $87
	.byte $64
	.byte $75
	.byte $7a
	.byte $ae
	.byte $83
	.byte $b8
	.byte $a0

TrainXp1
	.byte 255
	.byte 148
	.byte 148
	.byte 148
	.byte 148
	.byte 148
	.byte 148
	.byte 148
	.byte 148
	.byte 148
	.byte 148
	.byte 148
	.byte 148
	.byte 148
	.byte 148
	.byte 148
	.byte 148
	.byte 148
	.byte 148
	.byte 148
	.byte 148
	.byte 148
	.byte 148
	.byte 148
	.byte 148
	.byte 148
	.byte $41
	.byte $5b
	.byte $37
	.byte $29
	.byte $73
	.byte $5d
	.byte $4c
	.byte $4c
	.byte $4e
	.byte $1d
	.byte $56

TrainYp1
	.byte 255
	.byte 56
	.byte 95
	.byte 134
	.byte 173
	.byte 212
	.byte 56
	.byte 95
	.byte 134
	.byte 173
	.byte 212
	.byte 56
	.byte 95
	.byte 134
	.byte 173
	.byte 212
	.byte 56
	.byte 95
	.byte 134
	.byte 173
	.byte 212
	.byte 56
	.byte 95
	.byte 134
	.byte 173
	.byte 212
	.byte $86
	.byte $58
	.byte $4d
	.byte $7c
	.byte $86
	.byte $5f
	.byte $92
	.byte $80
	.byte $70
	.byte $d5
	.byte $71

TrainClass
	.byte 255
	.byte 8
	.byte 7
	.byte 7
	.byte 6
	.byte 6
	.byte 9
	.byte 8
	.byte 8
	.byte 7
	.byte 7
	.byte 9
	.byte 9
	.byte 8
	.byte 7
	.byte 7
	.byte 10
	.byte 9
	.byte 9
	.byte 8
	.byte 7
	.byte 10
	.byte 9
	.byte 9
	.byte 8
	.byte 8
	.byte 4
	.byte 7
	.byte 10
	.byte 15
	.byte 7
	.byte 9
	.byte 7
	.byte 9
	.byte 8
	.byte 2
	.byte 7
```

* At $f7fc, change ORG statment to:

```
	ORG $FFFC
```

## Media

## Bill of Materials

## About the Author

[Nick A. Bild, MS](https://nickbild79.firebaseapp.com/#!/)
