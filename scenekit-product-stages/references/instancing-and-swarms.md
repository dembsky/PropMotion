# Instancing and Swarms

Celebration moments - coin jackpots, confetti, debris showers - need
dozens of actors at once. This reference covers what changes at N=40
(built and verified on a forty-coin jackpot) and what deliberately does NOT
change.

## Contents

- One geometry, N transforms
- Piles are slots, not physics
- Solve the parabola backward
- Tumble to a rest pose
- Landings are events, not freeze-frames
- Seeded randomness or nothing
- Stagger by scheduling, not by keyframe padding
- Destruction bursts: when the engine's particles win
- Budget notes

## One geometry, N transforms

Build ONE geometry instance and ONE material set, and give every swarm
node the same references - the memory cost of forty coins is one mesh.
SceneKit still issues a draw call per node, which is fine at this scale;
what kills you is forty separate geometries with forty texture sets.
`flattenedClone` is NOT an option for a swarm that animates per-node
(and it scrambles multi-material geometries anyway - see
modeling-actors.md).

## Piles are slots, not physics

A pile that reads "poured" is choreography: precompute a SLOT for every
actor - concentric layers, wide at the bottom, one coin thickness per
level, jittered radius and angle so the stack is not printed - and fly
each actor INTO its slot. No collisions, no settling simulation, no
z-fighting: every landing is exact, and the pile is identical every run.
Squash the depth axis (z * ~0.55) so the heap composes for a front-on
camera.

## Solve the parabola backward

Ballistics with a known destination is algebra, not simulation:

```swift
vx = (slot.x - launch.x) / T
vz = (slot.z - launch.z) / T
vy = (slot.y - launch.y + g/2 * T*T) / T
```

Pick a flight time per coin (seeded, ~0.6-0.95 s), derive the launch
velocity, bake the exact parabola. Every coin lands in its slot at its
scheduled moment - the fountain shape emerges from the slots' geometry.

## Tumble to a rest pose

Random-axis tumble sells the flight, but every coin must LAND like a
coin: accumulate the spin quaternion over the flight and slerp it toward
the flat rest orientation (random yaw only) over the last ~20% with a
smoothstep. The blend is invisible mid-air and guarantees a clean pile.

## Landings are events, not freeze-frames

A coin that stops dead in its slot reads as a sprite. Extend every
actor's bake past touchdown:

- One or two decaying REBOUNDS from the impact speed (`0.2 * |vy_land|`,
  each 0.35 of the last), with a few centimeters of drift that the coin
  KEEPS - the rest position is slot plus drift.
- A damped ROCK on the rim after the last rebound
  (`amp * exp(-t/0.14) * sin(2pi * 7t)` about a random horizontal axis,
  composed onto the flat rest quaternion).
- The PILE REACTS: at each landing moment, nudge up to two nearby
  already-resting actors (a ~1 cm y-bump and a ~3-degree tilt-and-back,
  0.15 s) - scheduled token-guarded, skipped if the neighbor is still
  animating. Without this the heap absorbs forty hits like concrete.
- Re-triggering on a standing pile must not blink it away: visible
  actors HOLD their pose until their own launch moment, then erupt from
  where they lie toward rotated slots - the pile churns instead of
  teleporting.

## Seeded randomness or nothing

Variety comes from a seeded LCG, not `random()`: an identical shower on
every replay is debuggable (a coin clipping the camera reproduces), and
choreography tweaks are judgeable because only the tweak changed.

## Stagger by scheduling, not by keyframe padding

Each coin's animations start via a token-guarded `asyncAfter` at its own
delay (~50 ms apart), not by padding every keyframe array with hold
frames. Forty scheduled closures are trivial; forty arrays with variable
prefixes are a bug farm. Haptics: tick every FOURTH landing - forty
impacts read as a buzz.

## Destruction bursts: when the engine's particles win

This skill argues against `SCNPhysicsBody` for choreographed actors, and
that stands. But an actor CRUMBLING into hundreds of grains that fall,
bounce, and rest on the floor is the honest exception: at N in the
hundreds, node swarms and baked keyframes stop being economical, and a
one-shot `SCNParticleSystem` with real collisions is the right tool.

```swift
let system = SCNParticleSystem()
system.loops = false
system.emissionDuration = 0.06
system.birthRate = 11_000            // one flash ~ birthRate x duration
system.emitterShape = SCNSphere(radius: actorRadius * 0.96)
system.birthLocation = .volume       // born THROUGHOUT the actor
system.birthDirection = .surfaceNormal
system.particleVelocity = 1.15       // the burst must actually disperse
system.isAffectedByGravity = true
system.colliderNodes = [floorNode]
system.particleDiesOnCollision = false
system.particleBounce = 0.28
system.particleFriction = 0.25       // INVERTED dial - see below
```

The knobs that are not what they look like:

- `particleFriction` is inverted from physical intuition: 1.0 slides
  freely, 0.0 sticks dead. A LOW value (~0.25) is what brings a grain to
  rest after its last hop. Exactly the kind of dial to look up, never
  guess.
- Particles never collide with EACH OTHER, so a timid launch velocity
  keeps the actor's shape all the way down and it lands as one puddle.
  A real crumble needs enough outward push (plus variation and spread)
  to scatter its grit across the floor.
- Debris legibility: a hard-silhouette sprite (an irregular chunk with a
  lit face and a shadow face) reads as MATTER; a soft radial disc reads
  as snow regardless of physics. Spin via angular velocity sells
  tumbling shards. House rules from the stage still apply: matte, unlit,
  alpha-blended, distance-sorted, dying by fade - and tiny against the
  actor; realism is scale and death, not volume.
- Hide the actor the same frame the burst starts; warm the particle
  pipeline at install with the zero-opacity trick from
  [first-frame-and-warmup.md](first-frame-and-warmup.md).

The price, stated openly: the engine's particle simulation is NOT
seedable, so a destruction stage is exempt from the bit-identical
frozen-snapshot contract of
[rendering-contract.md](rendering-contract.md). It proves itself by a
motion pixel-diff between two captures, never by frame equality - write
that into the stage's comment so the exemption is a decision, not a
surprise.

## Budget notes

- ~40 animated nodes with one shared geometry: no measurable cost on any
  modern device; prepare([scene]) still warms it.
- One shared shadow POOL under the pile that fades in with the landings;
  per-actor blobs would double the node count for an effect nobody can
  separate visually.
- If the swarm ever needs hundreds, that is particle territory
  (stage-recipe.md) - node swarms are for actors that must land in exact
  places or be individually interactive.
