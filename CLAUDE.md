# CLAUDE

We are building the project 'CircuitBreaker'. This is a sega saturn homebrew game built using the jo-engine.

The game is inspired by Tron film (1982) light cycle scene and Rez video game (2001). It will be primarily be a first-person perspective competitive racer, but the game will 'crash' and the player find themselves on an unexpected adventure outside the realm of initial possibilities, on the run from a powerful watchdog who is trying to handle your exception.

Primarily the controls of the player is paramount - similar to development focus of Super Mario 64, where Shigeru Miyamoto spent months on making the movement perfect. Secondary, it aims to push the boundary of the sega saturn visually and audibly, while maintaining a consistent frame-rate above 30fps always. Lastly, the game will contain different game play types, to keep the game interesting, to be a homage to Computer Video Games - without feeling too overwhelming.

## Art Direction

- Bold geometry
- limited vibrant palette
- extreme contrast don’t depend on polygon counts or texture detail
- sense of scale and vast size
- Cyberpunk
- Glass-esk feel and sound effects, similar to Tron's 'Cesta' game

## Technical

Sega Saturn architecture was largely underutilized, and with your help, we will squeeze as much power as possible.

Memory sizes are very limited, all functions and variable usage needs to carefully considered.

### Maths on the SH-2: avoid division

The Saturn's SH-2 CPUs have no divide instruction and no FPU, so arithmetic that's cheap on a PC is expensive here:

- **Division (`/`, `%`)** compiles to a software library call (`__sdivsi3`), tens of cycles each. That includes division by a constant like `/ 1024`: the SH-2 can only shift by 1, 2, 8 or 16 bits per instruction, so even that isn't free.
- **Floats** are emulated in software and are far slower still. Use integer or 16.16 fixed point (`FIXED`) at runtime.
- **No division in per-frame loops over vertices, columns, pieces or pixels.** Instead:
  - compare scaled values rather than dividing them (e.g. keep a dot product x1024 instead of `/ 1024`)
  - precompute reciprocals or tables at load time
  - take a fast path for the common case (e.g. axis-aligned trail segments step by `+-s`)
  - use powers of two where a scale is free to choose
- Where a division must stay, keep it per object (once per strip or per frame), never per element, and comment why it's there.
- Measure on the Saturn, not the PC: the PC hides this cost completely (`wall_clip` looked cheap in a host benchmark and wasn't on hardware).
- The SH7604 does have an on-chip division unit (memory-mapped, ~39 cycles), which GCC doesn't use; worth considering for a hot path that can't avoid dividing.

The controls and the way the player (user!) feels is the up-most importance. Its got to feel great to play - Instant responsiveness, Predictability and consistency, Simulated space with weight and physicality, Intent-honoring forgiveness, Juice / multi-sensory feedback.

## 2 Players

The competitive racer can be played with 2 players via either split-screen or with a far isometric view.

## Testing

We need to unit-test where possible, as visual tests are harder to achieve. Where possible, create automated E2E tests to ensure core functionality is retained when adjusting new features as we go.

## Goals

It is paramount that I learn along side the development. It is no good AI changing the code if I do not understand why, how, what. Ensure code is commented, and if possible, write the code incrementally that allows me to understand - like pair programming together.

## Learnings

Read `SKILLS.md` (next to this file) at the start of every session: build and test commands, tool quirks, Saturn/SGL facts and mistakes already made. Add to it whenever something takes time to find out.
