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
| `segSegClosest` | the capsule-capsule primitive (Ericson §5.1.9) all blade contact runs on |
| `pairCombat` | blade-vs-blade: swept capsule search, tip-tip instability, bind ejection, the unified impulse |
| `bladeVsBody` | shields, presses, wounds, knockback, and the kill/XP/drop path |
| `buildSdf / sdfAt` | signed distance field to the walls (15px chamfer grid, per floor) |
| `foeIntent` | the AI state machine (all fighters share it; temperament parameterizes it) |
| `genFloor` | dungeon procgen: rooms + L-corridors on a 60 px tile grid → wall rects |
| `castFloor` | who spawns on each floor (the content tables) |
| `flowTo / flowDir` | BFS flow fields (Dijkstra maps) — how blocked AI paths around walls |
| `bladeVsWalls` | swept blade-vs-stone contact + hard penetration resolution |
| `dungeonUpdate` | aggro/line-of-sight (`losClear`), leashing, healing, stairs, loot swap, level-ups |
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
  Never special-case a parry; tune inertia, radii, and costs instead.
- **Blades are capsules, not line segments.** Contact = closest centerline
  distance (`segSegClosest`) against the summed `weapon.rad` values, swept
  across sub-frame poses `[1/6 … 1]` (single-frame checks WILL tunnel; blades
  move ~85 px/frame). Two deliberate extra contact cases carry the feel:
  **tip-tip** (two points meeting head-on take their normal along the line —
  points deflect, they don't slide past), and **bind ejection** (dominant
  axial sliding blends the normal toward the blade axis, the soft
  body-compliance direction — two driving thrusts beat each other aside).
  Don't "simplify" either away; without them thrust-fencing goes silent.
  One escape hatch: for ~0.7s after a fencer stands up (`upT`), a blade pair
  that BEGINS the frame already crossed ghosts apart freely — a stumble can
  leave blades tangled (limp blades take no contact), and a tangle has no
  contact moment to rewind to, so impulses would only jitter-bind. The gate
  matters: deep binds legitimately end frames crossed and must keep ejecting.
- **Contact normals at the contact moment.** Normals are evaluated at the
  rewound pose (`bladeAt(f, tC)`), not the post-step pose; only blades
  rotating *into* contact get rewound.
- **Balance drains only through `drainBalance`** — it feeds the COG marker,
  the audio, and the stumble trigger. Side-channel subtraction breaks the
  fall guarantee.
- **The shield always stops steel.** Its rebound is physics; `bladeCd` may
  only rate-limit presentation and costs (gating the whole rebound once let
  blades drive through for ~8 frames after any clang). The arc is also
  absolute cover via the contact-angle backstop in `bladeVsBody` —
  point-blank spears used to worm past the chord segment once the hand was
  inside the shield radius.
- **Muscles cap drive speed, not contact speed.** The `wCap` clamp only binds
  when muscles are adding speed; impulse-launched blades may exceed it.
- **Steel never ends a frame inside stone.** `bladeVsWalls` sweeps sub-frame
  poses against the wall SDF and Newton-resolves penetration at the **first**
  penetrating sample from the hand — the entry point. (Not the deepest: thin
  walls have an SDF ridge mid-wall where the gradient is ambiguous or points
  out the FAR side, i.e. through the wall.) A radially-pinned blade shoves
  the body back instead, capped at 30px/frame. `resolveCombat` runs a second
  pass because contact impulses can fling a blade wallward after its own pass
  ran, and `bladeVsBody` voids any wound whose hand→contact line crosses a
  wall. Together these make stabbing through walls impossible — don't replace
  any layer with a visual-only fix.
- **Blocked AI walks the flow field, never the straight line.** When
  `losClear` fails, `dungeonIntent` swaps the intent's move vector for the
  BFS field direction (downhill to close, uphill to flee). Don't add
  wall-feelers or steering hacks on top; fix the field.

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
