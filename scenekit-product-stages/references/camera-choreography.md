# Camera Choreography

Everything else in this skill animates the actor under a fixed camera. This
reference covers the inverse: the camera as the performer - reveal flights,
orbit interaction, and the staging that makes a moving camera read well.
Built and verified on a demo stage (a wheel on a podium).

## Contents

- The orbit rig: one number per axis
- The baked reveal flight
- Focus rides the dolly
- Interaction: drag-orbit, inertia, fly home
- Staging for a moving camera
- Matte surfaces under downlights
- Constraints: tracking for free, and where they must not go

## The orbit rig: one number per axis

Never orbit by translating the camera and re-aiming it - the look-at
correction fights the motion and drifts. Build a rig:

```swift
orbitNode (euler.y = heading)
  > pitchNode (euler.x = elevation, negative looks down)
    > cameraNode (position.z = distance)
```

Heading, elevation, and dolly become three scalar channels. Everything the
skill knows about baked animation applies unchanged: bake keyframes onto
the three nodes over one clock, steal the rig at its `presentation` pose
exactly like stealing an actor, guard beats with generation tokens.

## The baked reveal flight

The classic product reveal: open LOW and CLOSE (the actor filling the
frame, detail first), pull back while rising and swinging to the hero
framing. One eased parameter drives all channels so the flight reads as a
single camera move, not three:

```swift
for step in 0...steps {
    let s = Float(step) / Float(steps)
    let ease = s * s * (3 - 2 * s)
    yaw.append(open.yaw + (hero.yaw - open.yaw) * ease)
    pitch.append(open.pitch + (hero.pitch - open.pitch) * ease)
    distance.append(open.distance + (hero.distance - open.distance) * ease)
}
```

Fire it behind the first-frame warm-up gate like any entrance. Reduce
Motion: set the hero pose directly and skip the flight.

## Focus rides the dolly

`wantsDepthOfField` with a fixed `focusDistance` breaks the moment the
camera dollies. Bake `focusDistance` from the SAME distance samples as the
dolly channel and add the animation to the camera object (`SCNCamera` is
`SCNAnimatable`); the actor stays sharp while the background blur
breathes. Moderate `fStop` (4-5) - shallow DoF at product scale reads as
miniature.

## Interaction: drag-orbit, inertia, fly home

- Drag: steal the rig, then heading = base + dx * k, elevation clamped to
  a range that never goes under the floor or over the pole.
- Flick: the throw integrator reduced to one angular dimension - decay the
  release angular velocity exponentially (`speed *= exp(-dt / tau)`), bake
  the yaw samples. No walls needed; heading is unbounded.
- Double-tap home: fly to the NEAREST equivalent heading -
  `target + round((current - target) / 2pi) * 2pi` - unwinding accumulated
  laps looks broken.
- Idle drift (slow `repeatForever` rotate action) resumes on a delayed,
  token-guarded schedule after any interaction; the actor turntable is a
  separate slow action so the product keeps presenting itself.

## Staging for a moving camera

A moving camera needs a WORLD: on a bare void the orbit itself becomes
invisible (nothing parallaxes). Minimum staging that sells it:

- A podium under the actor - the orbit reads against its ellipse.
- `SCNFloor` with mild reflectivity (0.3, falloff a few units): the
  mirrored actor is the cheapest, strongest grounding cue SceneKit has.
  Keep the floor material dark lambert; reflections carry the interest.
- Lights and shadow rig stay world-fixed; only the rig moves.

## Matte surfaces under downlights

A large flat top under strong downlights will NOT go dark just because
its albedo is dark, twice over:

- PBR keeps a broad specular sheen for any roughness; a 0.07 and a 0.025
  albedo render the same mid-gray pool. If the design says "matte charcoal
  pedestal", use `.lambert` - killing specular entirely is the point.
- sRGB gamma lifts small linear values hard: with ~1600 units of summed
  downlight, a 0.05 lambert albedo is a readable charcoal; 0.15 is
  already silver.

## Constraints: tracking for free, and where they must not go

SceneKit can steer a node every frame by itself: a look-at constraint keeps
a camera pointed at the actor through a whole baked flight (turn the gimbal
lock flag on, or the rig flips upside down crossing the pole), and a
billboard constraint keeps a flat label or sprite facing the lens from any
angle. `influenceFactor` below 1 turns a hard lock into a lazy follow - a
camera that trails the actor's motion by a beat reads more human than a
rigid track.

One placement rule keeps this safe: constraints are for the RIG and the set
dressing, never for the performer. A constraint rewrites its node's
transform after animations are applied, every frame, silently - put one on
a node whose channels you bake and the constraint wins, and the beat you
authored simply does not happen. The actor stays baked; the things WATCHING
the actor may constrain.
