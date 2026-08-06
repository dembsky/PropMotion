# Multi-Actor Physics and Hit-Testing

Every other physics recipe in this skill bakes ONE actor against dumb
numbers (walls, floor). This reference covers the step up: several actors
whose futures depend on each other, and gestures that must first decide
WHICH actor was grabbed. Built and verified on a three-marble scene
(colliding rubber marbles).

## Contents

- One simulation, N state vectors
- Pairwise collisions inside the step
- The held actor is solid - and it SHOVES
- Match the squash to the material
- A grab freezes the world
- Hit-testing a grab
- Squash and haptics own their contacts

## One simulation, N state vectors

Independent baked futures break the moment actors can touch: actor A's
keyframes cannot know that actor B will arrive. The fix is structural,
not incremental - ONE fixed-step integrator advancing an array of state
vectors `{x, y, vx, vy, spin, omega, grounded}` on one clock, emitting
one keyframe set PER actor at the end. Everything the single-actor
recipes established (restitution floors, directional wall reflection,
momentum spin, squash channels) runs per-actor inside the same loop.

Termination is collective: the bake ends when EVERY actor is grounded
and slow, plus the squash tail of the latest contact anywhere.

## Pairwise collisions inside the step

Collisions must be resolved along the CENTER-TO-CENTER NORMAL, in full
2D, even on a stage whose travel is nominally 1D. The x-only shortcut
(swap horizontal velocities when overlapping) produces every failure at
once: interpenetration, resting-contact jitter, and - the giveaway - a
ball dropped ON TOP of another teleporting sideways instead of rolling
off. Per pair (i < j), when center distance < 2r:

- Compute the normal `n = (dx, dy) / dist` (guard the degenerate
  zero-distance case).
- Separate first, unconditionally: push each body half the overlap
  along n, clamping y at the floor. If the push lifts a "grounded"
  body, un-ground it, or it freezes hovering.
- Apply the impulse only when APPROACHING: closing = (vj - vi) . n < 0;
  equal masses: `impulse = -(1 + e) * closing / 2`, subtract along n
  from one body, add to the other (e ~ 0.88, livelier than the floor).
- A vertical impulse component can knock a grounded body airborne;
  a grounded body never keeps a downward vy (the floor holds it).

The reward for the normal-based version: a marble dropped onto another
slides off along the contact tangent under gravity - stacking, rolling
off, and nudging all emerge with no extra code.

## The held actor is solid - and it SHOVES

While dragging, the finger position is a TARGET, not a teleport: after
computing the held pose, push it out of any other actor along the
contact normal until the two are just touching, if the neighbors cannot
yield.

But neighbors SHOULD yield: the held actor is kinematic (infinite
mass), so pushing it along the floor into another actor must send that
actor rolling. A baked future cannot express this - the finger is
unpredictable - so this is the one window where the stage runs LIVE: a
CADisplayLink for the drag's duration integrates the bystanders in
real time (same rules as the bake: friction, floor, walls, pairwise
contacts), with the held actor as a kinematic pusher - a bystander in
contact yields the full overlap and inherits the held actor's velocity
component along the contact normal. Release invalidates the link and
bakes the world from whatever state the live sim left, bystander
velocities included. Two traps: CADisplayLink RETAINS its target, so
kill it in dismantle or the coordinator lives forever; and clamp the
frame dt (a background hitch delivers a giant step that tunnels).

Two more, earned the visible way:

- Push from TARGET PENETRATION, not from held velocity. Deriving the
  shove from the held actor's actual motion deadlocks at slow speeds:
  the contact blocks the held actor, so its velocity is zero, so the
  bystander never moves. Instead, resolve the finger's raw target each
  tick - the bystander yields the full penetration and gets velocity
  `penetration / dt` (capped) along the normal. Slow shoves then
  bulldoze convincingly.
- Never project the held actor to tangent along the direction of the
  RAW TARGET: the instant the finger crosses the obstacle's center,
  that direction flips sign and the held actor teleports through to
  the far side. When the held actor must stop (its neighbor is
  wall-blocked), resolve along the direction to its PREVIOUS position.
- The gesture callback only RECORDS the target; the tick owns all
  resolution. Solving in both places races at two different clocks.
- Route every interactive rotation through the SIMULATION STATE, never
  only the node: the release re-bake starts from the state's angle, so
  a roll applied straight to `eulerAngles` snaps back on release and
  the actor's texture visibly jumps. Update `spin` in the state and
  mirror it to the node.
- The held actor must SWEEP to the finger's target in increments
  smaller than a radius, resolving contacts (bystander pushes, pairwise
  half-splits, wall clamps, a few relaxation passes) after each
  increment. This is the only solver shape that cannot flip a contact
  normal: every "resolve against the final target" scheme - single
  pass or iterated - explodes the moment the finger crosses an
  obstacle's center within one frame, ejecting the obstacle THROUGH
  the held actor to the far side. Chains need the relaxation passes
  inside each increment (a single pass leaves the middle body shoved
  halfway back into the held one); a jammed chain simply stops the
  sweep at the last increment that fit. Bystander velocities come from
  their TOTAL per-tick displacement, capped.

## Match the squash to the material

The squash channel is a material statement: a rubber ball takes ~0.28
peak deformation, a HARD marble a hint (~0.06, shorter envelope). Rigid
objects that visibly compress read as jelly - scale the squash cap and
duration to what the actor is made of, or drop the channel entirely for
stone and metal.

## A grab freezes the world

When one actor is grabbed mid-flight, the other actors' baked futures
are already wrong (they may have been due to collide with it). Freeze
EVERYTHING: snapshot every actor's presentation into its model, strip
all animations, zero the bystanders' velocities. The held actor follows
the finger; the release re-bakes the entire world from the frozen state
plus the throw velocity. One frame of global steal, one fresh
simulation - no half-alive keyframes to reconcile.

## Hit-testing a grab

With one actor, pan-anywhere-grabs is right and hit-testing is noise.
With several, the finger must pick one:

- Name the GEOMETRY node (`marble-2`), then walk each
  `SCNView.hitTest(_:options:)` result's ancestry to the named node -
  hit results return leaf geometry, not your logical actor.
- Use `SCNHitTestSearchMode.all` and take the first resolvable hit -
  the default closest-only can land on a non-actor (shadow plane,
  particle) and return nothing useful.
- Decide the miss policy consciously: a playful stage grabs the nearest
  actor on a near-miss; a precise tool ignores it.

## Squash and haptics own their contacts

Record contacts PER ACTOR with an axis tag (floor vs side): a ball-ball
impact squashes BOTH participants horizontally, a floor hit squashes
vertically, and the bake reads each actor's own contact list. For
haptics, sort the merged contact list by impact speed and schedule only
the strongest few - three actors settling can produce dozens of
contacts, and a haptic per contact reads as buzzing, not impacts.
