# Flappy Bird – Controlled Maze

A complete arcade game implemented entirely in hardware on an Intel DE-10 FPGA board, with a twist on the original: **the player does not control the bird — the player controls the maze.**

The bird drifts on its own, and the player raises and lowers the pipes to open a path for it.

Final project for *Electrical Engineering Laboratory 1A*, Technion – Israel Institute of Technology.
**Grade: 100 (project and course).**

---

## Table of Contents

- [Gameplay](#gameplay)
- [Controls](#controls)
- [Features](#features)
- [System Architecture](#system-architecture)
- [Module Overview](#module-overview)
- [Game Controller FSM](#game-controller-fsm)
- [Pipe Manager Algorithm](#pipe-manager-algorithm)
- [Debugging with SignalTap](#debugging-with-signaltap)
- [Tools and Technologies](#tools-and-technologies)
- [Repository Contents](#repository-contents)
- [Team](#team)

---

## Gameplay

The bird moves through the screen on a randomized trajectory that the player cannot influence.
Instead, the player shifts the pipes up and down to line up an opening before the bird reaches them.

Every pipe passed increases the score. Every collision costs a life. When lives run out, the game
ends and the player returns to the menu to pick a difficulty and start again.

Difficulty affects the number of starting lives, the scroll speed, and how aggressively new pipes
are spawned.

---

## Controls

| Key | Action |
| --- | --- |
| `8` | Move pipes up |
| `2` | Move pipes down |
| `4` / `6` | Move between options in the menu |
| `Enter` | Confirm the selected menu option |
| `0` | Activate diamond mode — shields the bird from incoming missiles |
| `+` | Return to the menu after being disqualified |

The project demo is capped at two minutes, so two keys let us reach the states worth showing
without having to play through them:

| Key | Action |
| --- | --- |
| `7` | Jumps the score straight to 49, so the high-score behaviour can be demonstrated |
| `1` | Ends the run immediately, so the game-over screen can be shown without losing all three lives |

Input is read from a PS/2 keyboard through a dedicated keypad decoder module.

---

## Features

- **Inverted control scheme** — the player steers the obstacles, not the character
- **Three difficulty levels** — Easy, Medium and Hard, selectable from an in-game menu
- **Lives system** — heart matrix display, with the starting count determined by difficulty
- **Live scoring** — seven-segment and on-screen score display, updated as pipes are cleared
- **640×480 VGA output** — animated sprites, scrolling background, clouds and ground layer
- **On-board audio** — background music that changes per screen, driven by a tone decoder and
  melody player
- **Randomized behavior** — pipe spawning, gap placement and bird motion driven by a
  pseudo-random generator
- **Diamond mode** — a special shielded state that protects the bird from incoming missiles

---

## System Architecture

The design is built as a hierarchical schematic (BDF) top level over SystemVerilog modules.

```
                        TOP_FLAPPY
                             |
   +--------------+----------+----------+--------------+
   |              |          |          |              |
 clock         keyboard  game_      pipe_manager    sound
 divider        (PS/2)   controller                (audio)
                             |
                         objects_mux
                             |
                      VGA_Controller
                             |
                        VGA output
```

- **clock** — clock division and millisecond/second counters for timing and animation
- **keyboard** — PS/2 interface and keypad decoder
- **game_controller** — central finite state machine; owns lives, score, difficulty and game flow
- **pipe_manager** — spawns, moves, and tracks the pipe array; reports collisions and scoring
- **objects_mux** — composites all drawable objects into a single pixel stream
- **VGA_Controller** — generates sync signals and drives the 640×480 display
- **sound** — selects and plays the melody matching the current screen

---

## Module Overview

| Module | Responsibility |
| --- | --- |
| `game_controller` | Main FSM: menu, difficulty, gameplay, game over |
| `pipe_Manager` | Pipe array management, spawning, speed and scoring logic |
| `pipe_Blocks` | Individual pipe rendering and position tracking |
| `pipes_mux` | Multiplexes the active pipe outputs into the draw pipeline |
| `Bird_Block` | Bird sprite rendering and randomized vertical motion |
| `VGA_Controller` | VGA timing, sync generation and pixel output |
| `objects_mux` | Priority multiplexer for all on-screen objects |
| `Background_draw` | Scrolling background and ground layer |
| `cloud_blocks` | Animated cloud layer |
| `ScoreBlock` | Score tracking and on-screen digit rendering |
| `HeartDisplay` | Lives display as a heart matrix |
| `menuBlock` | Menu screen rendering and difficulty selection |
| `game_over_block` | Game over screen |
| `rocketBlock` | Incoming-missile object used in diamond mode |
| `SoundSelector` | Chooses the melody for the current screen |
| `ToneDecoder` | Converts note data into output tones |
| `random` | Pseudo-random source for spawning and bird motion |
| `KBD` | PS/2 keyboard interface |

`VGA_Controller`, `KBD`, `ToneDecoder` and `random` were supplied by the course as ready-made
building blocks. Every other module was written or substantially modified by us, along with all
the interfacing between them — reading player input from the keyboard decoder and driving the
game from the random source included.

Sprite graphics are stored as bitmap modules and `.mif` memory initialization files.

---

## Game Controller FSM

```
          +---------+
          | IDLE_ST |  initial state
          +----+----+
               | Start
               v
          +---------+
    +---->| MENU_ST |  choose difficulty (keys 4 / 6)
    |     +----+----+
    |          |
    |   Enter: Easy / Medium / Hard
    |          |
    |     +----+-----+-----------+
    |     v          v           v
    | +--------+ +--------+ +---------+
    | |EASY_ST | | MED_ST | | HARD_ST |
    | +----+---+ +----+---+ +----+----+
    |      |          |          |
    |      +----------+----------+
    |                 | No lives left
    |                 v
    |        +----------------+
    +--------+  GAME_OVER_ST  |
        +    +----------------+
```

Each difficulty state configures the starting number of lives, the scroll speed and the spawn
behavior, then hands control to the shared gameplay logic. Pressing `+` on the game-over screen
returns to the menu.

---

## Pipe Manager Algorithm

The pipe manager keeps an array of pipe slots and, on every frame, decides which ones are active,
which one matters for scoring, and when to create a new one.

1. **Active + Rightmost** — scan the pipe array and identify which slots are active and which one
   is furthest right
2. **Random + Spawn** — when there is room, request a random gap position and activate a new pipe
3. **Closest_loc / Select** — determine the pipe closest to the bird; this is the one evaluated for
   collision and for scoring
4. **Score** — when the selected pipe is cleared, raise the score signal
5. **Colors** — route the correct color data for each active pipe to the draw pipeline
6. **Speed** — scale pipe movement according to the selected difficulty

---

## Debugging with SignalTap

One bug worth documenting: the score occasionally incremented when no pipe had actually been passed.

Capturing the design on real hardware with the **SignalTap** logic analyzer showed that at the moment
the `SCORE` signal rose, the `prev` variable pointed to pipe `1` — while `select` read `0000`, an
index that does not correspond to any existing pipe.

That mismatch isolated the fault to the selection logic rather than to the scoring logic, which is
where the fix belonged.

---

## Tools and Technologies

- **HDL:** SystemVerilog
- **Schematic capture:** Quartus Block Diagram Files (BDF) for the hierarchical top level
- **Toolchain:** Intel Quartus Prime
- **Hardware:** Intel DE-10 FPGA board
- **Verification/Debug:** SignalTap logic analyzer
- **Peripherals:** VGA display output, PS/2 keyboard input, on-board audio
- **Workflow:** two-person team using Git for version control

---

## Repository Contents

```
.
├── README.md
├── FlappyBirdControlledMaze.pptx    Final project presentation
└── FlappyBirdControlledMaze.qar     Quartus project archive
```

---

## Notes

- Development was split into rounds of sub-tasks across the two team members, with earlier lab
  modules reused where possible to stay within the board's memory budget.
- AI tools were used for bitmap asset generation and as an aid while interpreting SignalTap
  captures; the RTL we wrote, and the integration of the whole design, is our own.

---

## Team

**Ofek Amir** and **Dor Mandel**

Faculty of Electrical and Computer Engineering, Technion – Israel Institute of Technology
