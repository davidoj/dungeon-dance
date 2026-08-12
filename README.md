# Dungeon Dance

A top-down sword-fighting roguelike where the fencing is real physics and the
dungeon is level design for it. One HTML file, no dependencies, no build step.

**Play it:** open `index.html` in a browser, or serve it
(`python3 -m http.server`) and hit localhost.

## What it is

Swords here are not hitboxes on animation frames. Each blade is a rigid rod
driven by muscles with a speed cap and a range of motion; blade-on-blade
contact is resolved by a single impulse formula, so beats, binds, parries and
presses all *emerge* rather than being scripted. Damage is the relative
velocity of steel and flesh at the moment of contact.

Under that sits the resource triad the whole design hangs on:

- **Health** caps how fast stamina regenerates.
- **Stamina** caps how fast balance regenerates.
- **Balance** is the tactical currency — spent by swinging, by being struck,
  by clashing blades, by dashing. At zero you stumble, and stumbling in the
  wrong company ends runs.

So a wounded fencer tires, a tired fencer totters, and position — footwork —
is how you spend less of all three than the other fencer does.

### The dive

**DESCEND** starts a three-floor roguelike run through a procedurally
generated dungeon (rooms and corridors on a tile grid):

1. **The Warrens** — rats (a poking snout, 33 hp) and cats (twin claws and a
   pounce). The boss room holds a cat and two rats at once.
2. **The Garrison** — lightly armed fencers, a shield somewhere on the floor,
   and a pair of sergeants at the stairs.
3. **The Proving Hall** — armed, levelled duelists (quickblade, heavy blade,
   shield, spear) who **drop what they carry**, and a 200 hp champion with an
   ally guarding the end of the run.

Rules of the dungeon:

- Duelists hold their posts and wake **on sight** (line-of-sight, not just
  range). Waking one wakes its room. Break contact long enough and they walk
  home.
- **Weapons are loot, not classes.** You descend a plain fencer with a plain
  sword. Press `E` over a dropped weapon to swap — yours falls where you
  stand. A heavy blade swings slower, costs more balance to swing, and slows
  your walk, but its inertia soaks clashes: enemy steel bounces off while your
  arms barely feel it. A quickblade is the opposite bargain — light and fast,
  but every clash wrenches *you*.
- **Level-ups grow capability, never numbers.** Kills bank experience; each
  level is a choice between **ARMS** (swing faster, swings cost less) and
  **LEGS** (walk faster, dash further, dashes cost less). There is no +12%
  damage in this game.
- Clearing a room refills your stamina; clearing a floor refills your
  health — the full clear beats rushing the stairs.
- **Walls turn steel.** A swing that meets stone rebounds and staggers your
  blade — narrow corridors are thrust country, and backing a crowd into a
  doorway so they come one at a time is the oldest roguelike trick made
  literal.
- Death ends the run. That's the game.

**DUEL** and **PRACTICE** keep the older arena modes: pick archetypes
(classic / nimble / strong / bulwark / spear), fight 1v1 or 1v2, first to
three touches.

## Controls

| Input | Effect |
| --- | --- |
| `WASD` | footwork (backpedaling is slower — and telegraphed) |
| mouse | the blade chases your aim; the arm's base point drags behind it |
| click | a muscle impulse toward the mouse — aligned it's a stab, across the foe it's a swipe; reversing a live swing costs balance |
| `SPACE` | dash — dashing *under* your lean is a recovery step that restores balance |
| `E` | take a weapon from the floor |
| `T` | slow motion (duel and practice only — the whole game already runs at 90% speed) |
| `M` | mute |

Watch the red **X**: it's your centre of gravity, kicked around by every
impulse you take or spend. When it reaches the ring, you go down.

## Lineage

Rebuilt from the *Footwork* design (2016 repos `Footwork` / `FootworkSil`,
Sil- and Dark-Souls-inspired; December 2025 `footwork2` prototype) — the
balance mechanic finally gets the dungeon it was pointed at.

See [AGENTS.md](AGENTS.md) for a map of the code, how to tune the fighters'
AI, and how to test changes — written for AI coding agents and humans alike.
