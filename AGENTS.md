# AGENTS.md

Guidance for AI coding agents (and curious humans) working on Dungeon Dance.
The most common reason to be here is to fiddle with the fighters — their AI,
their weapons, their bodies — so that path is documented first.

## The whole game is one file

`index.html` — CSS, title-screen HTML, and one `<script>` (~2,300 lines).
No dependencies, no build. Edit, reload, play. **Chrome caches aggressively
on localhost: hard-reload (Cmd+Shift+R) or bump a `?v=` query param after
every edit, and verify your change actually loaded before debugging it.**

## Map of the script

Everything is in declaration order; these are the landmarks to search for:

| Search for | What lives there |
| --- | --- |
| `Tunables` | global feel constants (movement, dash, burst, triad regen, damage) |
| `const WEAPONS` | weapons as items: reach, inertia, speed, swing costs, carry weight |
| `const ARCHETYPES` | a body + a carried weapon + an `ai` temperament, per fighter kind |
| `armsSpd / legsMove …` | level-up multipliers (ARMS / LEGS stacks) |
| `stepFencer` | biomechanics: the triad, movement, the blade spring, muscle impulses |
| `pairCombat` | blade-vs-blade: swept contact search + the unified impulse collision |
| `bladeVsBody` | shields, presses, wounds, knockback, and the kill/XP/drop path |
| `foeIntent` | the AI state machine (all fighters share it; temperament parameterizes it) |
| `genFloor` | dungeon procgen: rooms + L-corridors on a 60 px tile grid → wall rects |
| `castFloor` | who spawns on each floor (the content tables) |
| `dungeonUpdate` | aggro/line-of-sight, leashing, healing, stairs, loot swap, level-ups |
| `drawFencer / drawDungeon` | all rendering (Canvas 2D, camera = `cam`) |

## Fiddling with the agents (the fighters' AI)

Every fighter runs the same state machine (`foeIntent`); its **personality is
entirely data** — the `ai` object on its archetype:

```
ai: {aggression, dashiness, retreat, seekPickup, range, blockBias, poke, charge}
```

- `aggression` — probability of choosing attack over strafe when in reach
- `dashiness` — how freely it spends stamina on dashes (nimble ≈ 2.2)
- `retreat` — how long it disengages after an exchange
- `range` — preferred distance as a multiple of weapon reach (spear 1.35, brawler 0.8)
- `blockBias` — shield-first play: guard, wait for the counter window
- `poke` — point-blank straight stabs vs committed sweeping cuts
- `charge` — pounce/shield-rush probability (the cat's leap is `charge` without `blockBias`)

New enemy = new `ARCHETYPES` entry (body stats + weapon + `ai`) + a line in
`castFloor` to spawn it. The rat and cat are the minimal examples — the cat's
twin claws come entirely from its weapon's `offs: [-0.38, 0.38]` (multi-segment
weapons need no other code).

**Do not add artificial handicaps (reaction lag, aim error) — the design rule
is that the AI plays by the same physics, and difficulty comes from kit and
temperament.** This was an explicit design decision; past attempts to add
perception lag were reverted.

## Physics invariants (break these and combat feels wrong fast)

- **One impulse formula.** All blade contact resolves in `pairCombat` via the
  rigid-rod impulse (`I = 10000 × weapon.I`, lever arms, restitution 0.45).
  Never special-case a parry or a beat; tune inertia and costs instead.
- **Swept contact.** Blades move up to ~85 px/frame; every contact check runs
  across sub-frame poses `[1/6 … 1]`, and near-parallel crossings are caught
  by the `closestBlades` outer-half proximity check. Single-frame checks WILL
  tunnel.
- **Contact normals at the contact moment.** Normals are evaluated at the
  rewound pose (`bladeAt(f, tC)`), not the post-step pose; only blades
  rotating *into* contact get rewound.
- **Balance drains only through `drainBalance`** — it feeds the COG marker,
  the audio, and the stumble trigger. Side-channel subtraction breaks the
  fall guarantee.
- **Muscles cap drive speed, not contact speed.** The `wCap` clamp only binds
  when muscles are adding speed; impulse-launched blades may exceed it.

## Testing changes

The loop is deterministic enough to test headlessly in the browser console:

```js
startGame('dungeon');   // or 'duel' / 'practice'
running = false;        // freeze the rAF loop
// step frames by hand:
stepFencer(player, 1/60, {burst:false, moveX:1, moveY:0, dash:false,
                          aimX:foes[0].x, aimY:foes[0].y}, foe);
for (const f of foes) if (!f.out) stepFencer(f, 1/60, dungeonIntent(f, 1/60), player);
resolveCombat(1/60);
dungeonUpdate(1/60);
```

Useful assertions after a change: floor connectivity (BFS over open tiles —
every foe and the stairs reachable from the spawn), nobody spawned inside a
wall rect, a constructed blade crossing produces symmetric `Δw` on both
fighters, and a few-thousand-frame soak with random inputs throws nothing.
Set `player.health = 100` inside soak loops to keep them running.

When constructing blade-contact tests by hand, remember contact needs a
genuine segment **crossing** (or outer-half proximity) during the swept
frame — collinear blades sliding past each other are correctly ignored, and
a bad test geometry looks exactly like a collision bug.

## Style

- Plain ES2020+, no frameworks; `'use strict'` at the top.
- Comments explain *design intent* (why the mechanic exists), not what the
  next line does. Match the existing voice.
- Feel constants live at the top or on the data tables — resist burying magic
  numbers in logic.
