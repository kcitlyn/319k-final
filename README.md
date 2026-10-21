# Cat Ninja

A Fruit Ninja–style game with a cat theme. It runs on a TI MSPM0G3507 LaunchPad
and was built as the ECE319H Lab 9 final project (Spring 2026) by Kaitlyn Chen
and Keerthana Mangalpally.

Fish and bombs fly up from the bottom of a 160×128 screen. You move a blade
cursor with an analog joystick and hold a button to slice. A second display
shows the score, lives, level and a reacting cat. A 5-bit DAC plays sound
effects, and a slide pot sets the volume.

## Gameplay

- **Joystick** moves the cursor. **Hold the top button** to slice. The cursor
  sprite changes and turns red while you are slicing.
- Each fish you slice scores points. Slicing several fish in one frame scores
  N²: 1 fish = 1, 2 fish = 4, 3 fish = 9. Combos flash "PURR-FECT x2!" or
  "PURR-FECT MEGA!" on the score display.
- **Bombs**: slicing one costs a life and plays an explosion. Letting a bomb
  fall off the screen costs nothing.
- **Missing a fish** (it falls off the bottom without being hit) costs a life.
  You start with 3 lives. At 0 the game goes to a "CAT-ASTROPHE!" game-over
  screen, which shows your score and the best score this power cycle.
- **Difficulty** goes from level 0 to 9. The level is 1 per 10 points or 1 per
  15 seconds, whichever is higher. Higher levels spawn more objects at once,
  spawn them more often, add more bombs, launch them faster and pull them down
  harder.
- **Bottom button** pauses during gameplay ("PAWS-ED"). On the pause screen,
  bottom resumes and top quits to the title screen.
- The menus can be shown in **English or Spanish** (Settings → Language).

## Hardware

| Part | Pins | Notes |
|---|---|---|
| MSPM0G3507 LaunchPad | — | Runs at 80 MHz (`Clock_Init80MHz`) |
| ST7735R LCD1 (gameplay, landscape 160×128) | SPI1: PB9 SCLK, PB8 MOSI, PB6 CS | Initialized with `INITR_BLACKTAB` |
| ST7735R LCD2 (status, portrait 128×160) | Shares SPI1, CS = PA26 | `LCD2.cpp` |
| Analog joystick (SparkFun COM-09032) | PA15 (ADC1 ch0), PA17 (ADC1 ch2) | Mounted rotated 90°. The X axis is inverted in software. |
| Slide pot (volume) | PB18 (ADC1 ch5) | Read without resetting ADC1, so the joystick keeps working |
| Top button (select/slice) | PA24 | Active-low, internal pull-up |
| Bottom button (back/pause) | PA18 | Active-low, internal pull-up |
| 3 life LEDs (through a ULN2003B) | PB19, PB12, PB17 | `LED_SetLives(n)` lights n of them |
| 5-bit DAC (audio) | PB0–PB4 | Driven by `../inc/DAC5.cpp` |

The pins are wired for the custom PCB. They do not match the pins of the
original Lab 9H starter project.

## How it works

### Timing

- **TIMG12** fires at 30 Hz and sets `Semaphore`. The main loop in
  `Lab9HMain.cpp` waits for it, then reads the slide pot, runs
  `Game_UpdateFrame()`, reads the input and steps the state machine. If the
  timer has not started yet, the loop falls back to a 33 ms delay. No LCD
  drawing happens in an interrupt.
- **SysTick** fires at about 11.025 kHz (80 MHz / 7256) and outputs one
  audio sample per tick to the DAC, scaled by the current volume (`Sound.cpp`).

### State machine (`game.cpp`)

This is a Valvano-style indexed FSM. Each state has an init function that
repaints the screen once when the state is entered, plus a 4-entry `Next[]`
table. The table is indexed by the value from `Game_GetInput()`: 0 means stay,
and 1–3 are the state's transitions. Button presses are edge-triggered
(`Switch_Pressed`), so holding a button does not trigger repeated transitions.

```
TITLE ──INFO──────▶ INSTRUCTIONS ──any btn──▶ TITLE
  │───SETTINGS──▶ SETTINGS ──▶ LANGUAGE / CREDITS ──▶ SETTINGS
  │                   └──bottom──▶ TITLE
  └───PLAY──────▶ GAMEPLAY ◀──bottom── PAUSED ──top──▶ TITLE
                    │  └──bottom──▶ PAUSED
                    └──lives == 0──▶ GAMEOVER ──any btn──▶ TITLE
```

A meow plays on every screen change except the change to game over, which
starts a looping sad sound instead.

### Rendering

- **Gameplay** redraws LCD1 one scanline at a time, from top to bottom. Each
  row draws the wood-plank floor, then the opaque pixels of every active
  sprite, then the cursor. Runs of opaque pixels are sent as one SPI write.
  This avoids flicker without a frame buffer, which the MSPM0G3507 doesn't
  have enough RAM for.
- Sprites are stored bottom-up in RGB565 (`images/images.h`). `0xFFFF` is
  treated as transparent. Each type has a whole sprite at an even index and a
  cut sprite at the next odd index. Slicing adds 1 to the type index, which
  switches it to the cut sprite.
- **LCD2** shows a 7-segment score, the lives, the level, a short message
  banner, and a cat that reacts: `excited_cat` in menus and gameplay,
  `sleeping_cat` when paused or after a game over. To keep SPI traffic low,
  it keeps a copy of the last values it drew and only redraws when one
  changes.
- Menus only redraw the option whose highlight changed.

### Debugging aids

- `HardFault_Handler` and `Default_Handler` save PC, LR, SP and xPSR from the
  fault into `g_fault_*` globals so you can read them in the debugger.
- PB27 toggles inside the TIMG12 interrupt so you can watch the timing on a
  scope or with TExaS.
- `Lab9HMain.cpp` also has standalone test programs: `main1` (special
  characters), `main3` (switches), `main6` (PA15 toggle), `main7` and `main8`
  (LEDs), and `main9` (switching between the two LCDs). To run one, rename it
  to `main`.

## Source layout

| File | Purpose |
|---|---|
| `Lab9HMain.cpp` | Startup, the 30 Hz timer interrupt, the main loop, fault handlers and test mains |
| `game.cpp` / `game.h` | The game: FSM, screens, fish physics and spawning, collisions, scoring, rendering |
| `LCD2.cpp` / `LCD2.h` | Driver for the second, portrait ST7735 that shows the status |
| `Joystick.cpp` / `JoyStick.h` | Joystick (two-channel ADC1) and slide pot reads, sharing one ADC1 setup |
| `Sound.cpp` / `Sound.h` | SysTick sample playback with volume, one-shot or looping |
| `sounds/sounds.h` | 5-bit PCM sounds: `meow`, `explosion`, `slice` |
| `images/images.h` | RGB565 sprites: fish (whole and cut), bomb/boom, cursors, cats, title logo |
| `Switch.cpp`, `LED.cpp` | Button and LED drivers |
| `ADC.c`, `Clock.c` | ADC and clock/PLL helpers from the course |
| `ST7735.h`, `Timer.h` | Copies of course library headers |
| `Fruit.*`, `Language.*` | Empty placeholders. That logic is in `game.cpp`. |
| `images/BmpConvert16.exe` | Converts a 24-bit BMP into a C array for `images.h` |
| `designdocuments/` | Spreadsheets for converting colors and trig values |
| `mspm0g3507.cmd`, `ticlang/` | Linker script and startup code |

## Building

1. Install **Code Composer Studio** with the TI Arm Clang compiler and
   **MSPM0-SDK 2.9.0.01** (the version listed in `.cproject`).
2. This project needs the ECE319K shared **`inc/` folder in its parent
   directory** (`../inc`). The project links `DAC5.cpp`, `LaunchPad.cpp`,
   `SPI.cpp`, `ST7735.cpp`, `SlidePot.cpp`, `TExaS.cpp` and `Timer.cpp` from
   there, and the source includes headers like `../inc/SPI.h`. `SPI.cpp` must
   provide `SPI_SelectDisplay()` for switching between the two LCDs.
3. Import the folder into CCS as a project (`ECE319K_Lab9H`), build, and
   flash it with the XDS110 debugger (`targetConfigs/MSPM0G3507.ccxml`).

`Debug/` holds build output that was committed to the repo. A clean build
regenerates it.

## Credits

- Game code: Kaitlyn Chen and Keerthana Mangalpally, ECE319H, UT Austin,
  Spring 2026.
- Based on Jonathan Valvano's ECE319K Lab 9H starter project. The starter was
  derived from TI MSPM0 SDK SPI examples and is released under the Simplified
  BSD License, © 2025 Jonathan Valvano. See
  http://users.ece.utexas.edu/~valvano/.
