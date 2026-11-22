# f-16-game
# AFTERBURNER

An F-16 flight simulator in C11, built on SDL2 and OpenGL 3.3 core. The flight
dynamics are the NASA TP-1538 six-degree-of-freedom model with its published
wind tunnel tables, not a hand-tuned approximation.

Targeted at a 2017 12-inch MacBook (Core i5-7Y54, Intel HD Graphics 615), which
is the reason for most of the rendering decisions in here.

## Build

```bash
brew install sdl2 cglm
```

Everything else lives in `vendor/` and needs no installation.

```bash
cmake -S . -B build -G Xcode
cmake --build build --config RelWithDebInfo
```

Open `build/Afterburner.xcodeproj` and press Run, or:

```bash
./build/RelWithDebInfo/afterburner.app/Contents/MacOS/afterburner
```

Command line: `--air` starts trimmed at 3 km, `--final [NM]` starts on a
stabilised final approach, `--view 0|1|2` picks chase, cockpit or orbit,
`--shot out.png --shot-time 2` renders a frame and exits,
`--novsync` uncaps the frame rate so the renderer's real cost is measurable,
`--render-scale 0.4..1.0` sets what fraction of the window the world is drawn
at (see below), `--fight [N]`, `--skill 0..1`, `--setup 0..3`, `--waves` and `--guns` start an
engagement, and
`--map 0..3`, `--seat CM` and `--throttle 0..1` set up a specific state for a
scripted screenshot. `--touch` brings up the tablet's on-screen controls and
lets the pointer stand in for a finger, and `--size WxH` opens the window at a
given size — together they are how the iPad layout is checked without an iPad,
since it is anchored to the screen edges and only right at the right aspect.

### iPad

There is a native iOS build, on OpenGL ES 3.0, flown from on-screen controls.
See [ios/README.md](ios/README.md) — including the part about this particular
Mac being capped at Xcode 15.2, which matters for installing onto anything
newer than iPadOS 17.2.

**Pause moved from space to `P`.** Space is the gun. That is where it is in
every combat simulator there has ever been, and reaching for it in a fight and
getting a pause instead costs the fight.

## Tests

**Anything under `src/physics/` requires this to stay green.**

```bash
cd build && ctest -C RelWithDebInfo
```

Four suites: `flight_test` (physics correctness, plus terrain and landcover
distribution), `combat_test` (guidance, the launch envelope, ballistics and the
opposing pilot),
`handling_test` (40, whether the thing can actually be flown -- roll stop
accuracy, a 360 degree aileron roll, a loop, a sustained banked turn, and the
authority available from a short tap, all driven through the real input
shaping with simulated keypresses) and `touch_test` (the iPad's on-screen
controls: the hit testing, and the layout invariants that keep a control from
sliding under the HUD, off the bottom edge, or under another control at an
aspect ratio nobody looked at).

or directly:

```bash
clang -std=c11 -O2 -Isrc tests/flight_test.c src/physics/*.c src/world/heightfield.c -lm -o /tmp/ft && /tmp/ft
```

The suite exists because the dangerous bugs in a flight model are all silent:
they do not crash, they do not warn, and the aircraft still flies. Sign errors
found and fixed during development include a pitch rate feedback term that was
acting as a pitch *diverger* (the aircraft sat perfectly still for twelve
seconds before winding itself into a tumble) and a sideslip feedback loop with
positive gain that the airframe's own weathercock stability was strong enough
to mask entirely.

The worst of them was a quaternion kinematics equation that multiplied the
body rate on the wrong side, so body rates were applied in the *world* frame:
rolled ninety degrees and pulling, the nose climbed toward the sky instead of
sweeping round the horizon. Wings level the two forms are algebraically
identical, so it survived every trim, takeoff, limiter and load-factor test in
the suite. **Rotation conventions can only be tested from a non-trivial
attitude**, and both suites now do.

## Controls

| | |
|---|---|
| `W`/`S` or up/down | stick fore and aft (`S` pulls) |
| `A`/`D` or left/right | roll |
| `Q`/`E` | rudder and nose wheel steering |
| `F` / `C` | speed brake open / close (it stays where you leave it) |
| `M` | **mouse stick** -- the pointer becomes an absolute stick position |
| `1`/`2`/`3`/`4` | throttle: idle / 50% / military / full afterburner |
| `[` `]` `\\` | pitch trim down / up / clear |
| alt, `X` | fine control, centre the stick |
| space | **gun** (held) |
| return | launch a missile |
| `L` | flares |
| tab | designate the next bandit |
| `K` | next engagement (cycles four entry setups) |
| `Z` | moving map: off / 4 / 11 / 27 nm |
| `H` | declutter the HUD |
| `+`/`-`, page up/down, or shift/ctrl | throttle (afterburner past 77%) |
| `B` | wheel brakes |
| `G` | landing gear |
| `V` | chase / cockpit / orbit view |
| `U` | turbulence: off / light / moderate / severe |
| `R` / `T` / `Y` | reset to runway / to trimmed flight / to a 4 nm final |
| `N` | seat height (cycles) -- raise it to see over the nose on final |
| `P`, `.` | pause, single step |
| `F12` | screenshot |

A game controller is used when present: right stick flies, triggers are rudder,
left stick Y is throttle, shoulder buttons work the speed brake.

### Sticks that are not gamepads

`SDL_IsGameController()` is only true for devices in SDL's mapping database,
and that database is a database of *gamepads* — Xbox, PlayStation, Switch. No
flight stick is in it, no HOTAS throttle is in it, and no drone controller is
in it. So the simulator falls back to the raw joystick interface for anything
unrecognised, with the axis assignment given explicitly:

```bash
./build/RelWithDebInfo/probe_input
```

Plug the device in and wiggle everything; it prints what enumerated, whether
SDL has a mapping for it, and live axis values so each control can be matched
to an index. Then:

```bash
./build/RelWithDebInfo/afterburner.app/Contents/MacOS/afterburner --joy-axes 0,-1,3,2
```

Order is roll, pitch, yaw, throttle; a leading `-` inverts. Append `,rel` when
the throttle axis is **spring-centred** — a drone controller's left stick is,
and read absolutely it would peg the engine at 50% the moment you let go, so
`rel` integrates the deflection as a rate instead.

If `probe_input` finds nothing, the device is not presenting a HID joystick
interface at all. Most drone controllers do not: they speak a vendor protocol
over USB to their own app, and there is nothing on this side that can change
that.

### If the operating system thinks you are typing

It used to. SDL2 turns text input on during video initialisation and leaves it
on, which hands every key press to the input method editor before the
application sees it — so with a composing input method selected, `W A S D` open
the candidate window and are swallowed, and on a Latin layout holding a letter
down pops up the press-and-hold accent panel instead of repeating. `input.c`
calls `SDL_StopTextInput()` at start-up and again on every focus gain, and
reads physical **scancodes** rather than layout-dependent keysyms, so the
bindings mean the same thing on any keyboard layout. Command-key chords are
passed through to the system rather than also firing the bare key.

The throttle is on `+`/`-` and page up/down as well as shift/ctrl, so it never
has to be held on a modifier key — which is what trips Sticky Keys.

With the mouse stick active the pointer is confined to the window: it is the
stick, and it has to be able to reach the stops without leaving the frame or
clicking on whatever is behind the simulator.

**Without a joystick, press `M`.** A key is a switch, so the only stick
positions a keyboard can hold are the ones its ramp happens to be passing
through -- which is why precise rolling and sustained turns are so hard on the
keyboard alone. The pointer holds 30% of travel indefinitely and releases it
exactly when asked. Keyboard axes stay live alongside and override on any key
press.

## Layout

```
src/physics/     no OpenGL, no I/O; linked into the sim and the tests alike
  aero_f16.c       TP-1538 tables and their interpolation
  engine.c         F100 thrust deck and spool dynamics
  flightmodel.c    6-DOF state, forces and moments, RK4
  fcs.c            fly-by-wire: g command, AOA limiter, roll rate, yaw damper
  gear.c           oleo struts, tyre friction, latched weight-on-wheels,
                   nose-wheel steering authority scheduled with groundspeed
  aero_extra.c     wave drag and ground effect -- fitted, not TP-1538 data
  turbulence.c     Dryden gusts as Ornstein-Uhlenbeck processes
  trim.c           Newton solve for straight and level flight
src/core/
  stick.c          keyboard and pointer shaping, trim -- no SDL, so it is testable
src/world/
  heightfield.c    procedural elevation, shared by physics and renderer
  terrain.c        baked heightmap plus a camera-warped grid
  scatter.c        instanced ground clutter; the only source of speed cues low down
  mountain_patch.h a modelled mountain range, baked to elevation data
src/render/        shaders, camera, HUD, frame assembly
src/model/         glTF loading with dimensional read-back validation
```

### Frame budget

The window is 1280x720 but the drawable on a Retina panel is 2560x1376 — three
and a half million fragments — and HD Graphics 615 cannot pay for a detailed
ground shader across all of them sixty times a second. The world is therefore
rendered into an off-screen buffer at `--render-scale` (0.75 by default, which
is 56% of the pixels) and upscaled, and **the HUD is drawn afterwards at native
resolution** with the window's 2x MSAA. Those are two different problems:
broad continuous surfaces survive being scaled and one-pixel green strokes do
not.

Two things learned the hard way while getting this back to a locked 60:

- **Integer multiplies are not cheap on Gen9.** The ground shader originally
  computed its noise from the same uint32 hash the heightfield uses. It cost
  17 ms a frame on its own. The same noise as a 256x256 texture sampled with
  `GL_LINEAR` — which *is* bilinear value noise, done in the sampler — is
  effectively free.
- **Detail must be faded on screen-space footprint, not on range.** `fwidth`
  gives the metres of ground one pixel covers; ground that is near but seen
  edge-on packs hundreds of metres into a pixel, and a distance-based fade
  leaves the fine bands at full strength there. That is what smears the middle
  distance into streaks.

### Conventions

Everything is NED: x north, y east, **z down**. Body axes are x forward,
y right, z down, matching the aerodynamic data. The single place this meets
OpenGL's y-up clip space is `mat4_lookat_ned`.

Aircraft geometry is baked into body axes at load time with the origin at the
centre of gravity, so the render model matrix is the attitude quaternion and
nothing else.

### The moving map

**Row is north, column is east.** A texture's first index is its s axis and s
is what the panel maps to x, so baking it the other way round — which is how
the terrain heightmap is stored, and what this was copied from — transposes the
chart. A transposed map is not obviously wrong at a glance: it still looks like
terrain, the aircraft is still in the middle, and every heading you fly is
reflected about the north-east diagonal.

`Z` cycles a north-up moving map through 4, 11 and 27 nautical miles. North-up
rather than track-up on purpose: a map that rotates under you does it exactly
when you are turning, which is the one moment you needed to read it.

The chart is baked once at start-up — hypsometric tint plus a hillshade from
the elevation gradient, which is what topographic charts have looked like for a
century and for a good reason: tint alone gives heights you cannot see shape
in, hillshade alone gives shape with no idea how high anything is. Forest comes
from the same `hf_forest_density()` the world itself is painted from, so a wood
on the chart is a wood out of the window.

Over it: the runway with a minimum symbol size (46 m at 27 nm is a third of a
pixel, and drawn honestly it disappears into the dashes of its own extended
centreline), ten kilometres of dashed extended centreline off each threshold,
a range ring, the aircraft, and bearing and distance to the field.

### Two things that make the aircraft sit in the world

- **A ground shadow**, which is not decoration. Without one there is no cue at
  all for height in the last few metres, and that is exactly where a landing is
  won or lost — the flare is judged on how fast the ground is coming up, and a
  shadow closing on the aircraft is the strongest signal a pilot gets for it.
  It is a flat cut-out of the planform, *translated* along the sun ray rather
  than stretched, because the shadow of a flat shape on flat ground is exactly
  a translation; the stretch one expects comes from the aircraft's own depth,
  which is second order at these sizes.
- **The afterburner plume** past 78% power, with shock diamonds — sized for
  daylight. A full-augmentation plume at night is metres long and spectacular;
  in sunlight it is a couple of metres of core and a shimmer, and rendering the
  night version in daylight is the usual way this effect goes wrong.

### Flying it

Two things about the handling are worth knowing because they are modelled
rather than assumed:

- **The speed brake is real and you will need it.** Four petals to 60 degrees
  in about two seconds, 0.062 in CD — roughly five per cent of the aircraft's
  weight at approach speed, or three degrees of glide path. It also **blows
  down**: the actuator cannot hold full deflection against high dynamic
  pressure, so at 300 m/s you get less than half of it. Without that it would
  be a supersonic air brake no F-16 has ever had.
- **Nose-wheel steering fades out with groundspeed.** A constant gain is what
  makes a simulated aeroplane feel wrong on the ground: the deflection that
  turns neatly on the ramp is a swerve at rotation speed, and the takeoff roll
  becomes a series of over-corrections no real pilot has to fly. Above about a
  hundred knots directional control is the rudder's job. The schedule is
  asserted directly rather than through the yawing moment, because tyre slip
  angle falls off with speed on its own and the two are indistinguishable in a
  screenshot.

### Landing

This is the hardest thing in the aeroplane, and most of what made it hard was
missing equipment rather than difficult flying. The airframe was never the
problem: 11 degrees of angle of attack trims at about 160 KCAS, a three degree
path needs 4.3 m/s of sink, and the gear absorbs 6. What was absent was every
instrument you would actually land with. Now present:

- **An AOA bracket** beside the flight path marker. The F-16 is flown on angle
  of attack on final, not on airspeed — 11 degrees whatever the aircraft
  weighs, because AOA already has the weight folded into it. The index closes
  up against the bracket when on speed.
- **An instrument approach**: glide path and centreline deviation on ICAO
  scales (0.7 and 2.5 degrees full scale), plus distance to the threshold. The
  runway end is whichever one you are pointed at.
- **A PAPI** at each threshold. Four lights, white above their set angle and
  red below, so two white and two red is on the path. Every runway an F-16
  lands on has one, and judging three degrees by eye through a fixed 60 degree
  window with no depth perception is considerably harder than doing it in the
  real aircraft.
- **`Y`, or `--final [NM]`** — reset onto a stabilised final: gear down, on
  speed, on the path. Landing used to be reachable only by flying a whole
  circuit for one attempt at it.

**One thing is worse here than in the real aeroplane, and cannot be fixed from
this end.** The forward view is blocked lower than an F-16's is: the integrated
control panel in this cockpit mesh rises nearly to the windscreen base, so at
eight degrees nose up on final the runway sits behind it. The design eye point
cannot be moved to fix it — fifteen centimetres higher and the camera is
outside the canopy sill. `N` raises the seat up to 10 cm, which recovers a few
degrees, and the last mile can be flown on the deviation bars, AOA and radar
altitude. The chase view sees everything.

### Air combat

`--fight`, or `K` in flight, sets up a neutral merge: the bandit starts eight
kilometres ahead, co-altitude, pointed at you. Neutral matters — spawning the
AI behind you is not a fight and spawning it in front teaches nothing about the
merge.

**The missile is a real weapon with a real envelope.** Proportional navigation,
a 45 degree seeker gimbal, a boost motor of about 29 kN·s and 30 g of
manoeuvre — *when it has the air to make it*, since lateral force is q·S·CN and
falls away with both altitude and speed. Turning costs energy through induced
drag, so a hard break at the right moment does not out-turn the missile, it
makes the missile too slow to finish the turn. `combat_test` pins all of it,
including that a long shot defended in time **misses** and the same defence
from 1.8 km does not. A missile that always hits is not a weapon, it is a
cutscene.

### Aiming the missile

`SHOOT` used to mean "there is something in front of you". It now means **this
shot will arrive**, and the instrument that says so is a proper dynamic launch
zone on the left of the HUD:

- **Rmax** — the furthest the missile can reach a target that keeps flying
  straight. A shot there arrives with nothing left and any turn defeats it.
- **Rne** — the no-escape range, hatched: it still arrives even if the target
  turns tail and runs the instant the motor lights. Inside this band the shot
  is yours.
- **Rmin**, a caret at the present range, and the time of flight.

None of those are curve fits. They are found by **flying the actual missile
forward** at a coarse step and bisecting on range, so the envelope is whatever
the weapon in `weapon.c` can really do — and it moves with the geometry the way
the real one does:

| aspect | Rmax | Rne |
|---|---|---|
| head-on | 18.0 km (the seeker's lock limit) | 6.1 km |
| beam | 9.6 km | 5.6 km |
| tail chase | 5.3 km | 5.3 km |

In a tail chase Rmax and Rne collapse together, because a target already
running away has nothing left to escape with. The whole envelope costs 2.3 ms
to compute, so it is refreshed five times a second rather than every frame.

The envelope is **not an interval** and cannot be found by bisecting up from
the minimum: against a fast crosser there is a hole at short range where the
missile simply cannot turn hard enough, and treating a failure at the minimum
as "no envelope" reported an empty zone for exactly the beam shots that are
perfectly takeable at two kilometres. So it scans first, then refines the outer
edge.

Alongside it: **tab** designates a target and keeps it designated (the box says
`TGT`), and the seeker has to **settle** — 0.7 seconds of steady tracking
before it will accept a hand-off, shown by the designator's corner ticks
growing from short to long. A shot cannot be snapped the instant the nose
sweeps past.

**Four entry setups**, cycled by `K`: neutral head-on, offset abeam, defensive
(he is behind you) and offensive (you are behind him). One entry geometry,
however well made, is one fight — by the third time you know exactly what the
merge will look like.

**Three kinds of opponent.** These are not difficulty levels — they all fly the
same aeroplane to the same limits — they are different answers to the same
geometry, and they lose to different things. Measured over a 70 second
approach:

| | slowest it got | mean angle-off on the run-in | time on the trigger |
|---|---|---|---|
| aggressor | 240 m/s | 20° | 5.3 s |
| energy fighter | 250 m/s | 23° | 1.5 s |
| wingman | 250 m/s | 42° | 2.3 s |

The aggressor will follow you into a scissors and can be beaten by refusing to
trade energy. The energy fighter will not: it takes the vertical instead of
hauling its nose round at nine g, and has to be caught at the top of its climb.
The wingman is not trying to kill you at all — it brackets wide and waits for
you to commit to its leader, which is a different problem, because turning to
defeat one of them presents your tail to the other.

**`--waves`** keeps sending them, a little better each time, and rearms you
with one missile fewer than last time. **`--guns`** takes the missiles away
from everybody: a pure BFM fight.

**Flares (`L`) are the answer to a missile, and only if you turn.** A decoy
that stays with the aeroplane is in the same place as the aeroplane and
decides nothing; it has to be given somewhere else to be. Measured over twenty
trials each, from the beam: no flares 0/20 defeated, flares while flying
straight 6/20, **flare and break 16/20**, and flares from directly astern —
where the tailpipe outshines them — 2/20. The `MISSILE` warning counts down in
**seconds to impact**, because that is the number a break is timed off; a
range tells you only that a missile exists.

The obvious implementation of seduction rolls the dice for every burning flare,
and it is wrong: a salvo every half second with a four-second burn puts twenty
in the air, twenty rolls a step is twenty times the intended rate, and every
shot is decoyed regardless of aspect or whether the target bothered to turn. A
seeker sees one cluster, not twenty chances.

**The AI flies the same aeroplane through the same control laws.** It emits a
`PilotInput` and nothing else: no direct attitude commands, no exemption from
the AOA limiter, no seeing through its own tail. It rolls its lift vector onto
where it wants to go and then pulls, gated on the cosine of the roll error,
because that is the only way the aeroplane can be flown. It has a reaction lag,
an aiming error, and it will run out of energy and go and get some back.

Bugs found by the suite that a screenshot never would have:

- The FCS overshot its own 9 g ceiling to **12.1 g** — the ultimate load factor
  — at high dynamic pressure, because the gains were sized at 250 m/s and
  control power scales with q. The aircraft came apart in response to a demand
  the control laws had already limited. Gains are now scheduled downward on q,
  which is what the real aircraft does and why it cannot be overstressed by the
  stick.
- **No gravity bias.** Proportional navigation says what acceleration nulls
  the line-of-sight rate; it knows nothing about gravity. An autopilot that
  commands exactly that flies one g low the whole way, and over a ten second
  shot the trajectory sagged nearly five hundred metres — arriving low, and
  diving straight into the ground the moment the seeker lost lock. "My
  missiles fly into the dirt" is what a missing g-bias looks like from the
  cockpit. A real autopilot adds back the component of gravity across the
  flight path so the fins hold the missile up.
- A 180 degree reversal is ill-conditioned: `want` and `fwd` are nearly
  antiparallel, so whatever small vertical component the requested direction
  carried decided the entire turn plane. A quarter of a unit of "descend" was
  enough to turn a reversal into a split-S, and the AI arrived at the ground
  from six kilometres still perfectly on course.

### The ground

The landscape is what tells a pilot their height, speed and drift, so it has to
have structure at every scale the aircraft flies at. Four detail bands (820 m,
115 m, 17 m and 3.2 m) are each faded out when a pixel grows past about a third
of their wavelength, so something is always resolvable and nothing ever
aliases. Field parcels give the one thing noise cannot — straight edges of a
known size — and are kept off the woods, off slopes and off the aerodrome.

Forests come from `hf_forest_density()` in `heightfield.c`, and they come from
it **once**: `scatter.c` plants the billboard trees from it and `terrain.c`
bakes it into a texture for the ground shader, rather than the shader carrying
a second copy of the same noise to be kept in step by hand. `flight_test` pins
the resulting distribution, because a mask that has quietly become all-forest
or no-forest still renders perfectly well — just with nothing distinct on it.

The runway carries real ICAO markings at real sizes: threshold bars, 20 m
designator numerals, an aiming point 400 m from each threshold, touchdown zone
groups at 150 m intervals, paving lanes and rubber deposits. They are not
decoration — on a visual approach the aiming point is what the flight path
marker is flown onto.

### Scenery and the approach corridor

The mountain range is a downloaded model that turned out to be a 512x512
heightfield wearing 524288 triangles. `tools/bake_mountain.py` resamples it
into elevation data, so the terrain function places it and it becomes real
ground: collidable, shaded by the terrain shader, and free of triangles.

Where the ranges sit is a flying decision, not a scenic one. The runway lies
along the north axis, so both approaches pass through y = 0; every range is
pushed at least 8.5 km off that axis and a corridor function damps whatever is
left. `flight_test` pins the result -- nothing in the approach corridor exceeds
260 m over 26 km, and a 3 degree glideslope to either end is clear.

The airport model is scenery only. It sits beside the runway, scaled from its
control tower rather than from its own painted runway, and the strip the
aircraft actually uses stays the procedural one the gear model and the takeoff
regression were built against.

### Model calibration

`model_load` re-measures the loaded vertices and refuses to run if the span or
length drift more than 8% from the real aircraft. This check is not optional:
silent rescales — a bounding box computed before a transform, a leftover scale
factor on a leaf node — produce an aircraft that looks only slightly small and
is wrong in every moment arm. See `src/core/config.h` for how `MODEL_SCALE`
was derived and cross-checked.

## Credits and licence

See [CREDITS.md](CREDITS.md). The airframe model is CC-BY-NC-SA-4.0: attribution
required, **no commercial use**, share-alike. It is not redistributed with this
source tree.