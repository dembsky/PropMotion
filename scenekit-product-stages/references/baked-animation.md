# Baked Keyframe Animation

## Contents

- Why baked keyframes beat timers
- The cue sheet: designing a multi-phase beat
- Designing weight
- The baking recipe
- Model value first, then the animation
- fillMode and cleanup bookkeeping
- Generation tokens
- Stealing a node mid-animation
- Rolling: spin slaved to distance over radius
- Rolling an actor along a floor path
- The visible window: eases on paths that leave the frame
- The start tangent of an appended path
- Draw the line: debugging motion with a trail
- Channels beyond transforms
- Springs as authoring material
- One property, one owner
- Shared timing curves for play and scrub

## Why baked keyframes beat timers

Two ways to run choreography: mutate node properties from a timer or display
link on the main thread, or precompute the whole motion and hand it to Core
Animation as a `CAKeyframeAnimation`.

Bake. The baked animation runs on the render server, so a main-thread stall
(a SwiftUI diff, a database write, a JSON parse) never stutters it. It also
turns choreography into data: one pure function of time producing arrays,
which you can unit test, scrub, and reason about. And a motion designed as a
single C1-continuous function has no seams; the same design built from
overlapping scheduled animations has a seam at every junction.

The rule scopes to choreography. A one-shot property tween that is not a
beat - retinting a material after a picker tap, easing an overlay's opacity
- is exactly what an implicit `SCNTransaction` is for; wrapping such tweens
in the bake machinery is ceremony without benefit.

## The cue sheet: designing a multi-phase beat

An ambitious beat (a roll-in with a lift and a seat, a throw with a topple
and a settle) is designed before it is baked. Write it down as a cue sheet:
the phases of the motion in order, each with its time window, what happens,
and the values that change. The cue sheet lives as a comment directly above
the pose function and is the human-readable contract for the whole beat:

```swift
/// Cue sheet: roll-in entrance, 1.8 s total
///
/// phase      window (s)     action
/// approach   0.00 - 0.90    rolls in from offstage left, decelerating
/// lift       0.90 - 1.30    front rim lifts, actor tips toward the camera
/// seat       1.30 - 1.80    settles flat on the display face, spin parks at 0
///
/// cues: contact tick at 0.90, settle thud at 1.80
```

The phase boundaries then live in exactly one struct, and everything reads
from it: the pose function, the scheduled cues (haptics, token-guarded
continuations), the tests.

```swift
struct EntrancePhases {
    let approachEnd: Float = 0.90
    let liftEnd: Float = 1.30
    let total: Float = 1.80
}

func poseAt(time t: Float) -> Pose {
    if t < phases.approachEnd { return approach(t) }
    if t < phases.liftEnd { return lift(t - phases.approachEnd) }
    return seat(t - phases.liftEnd)
}
```

Rules that keep a multi-phase beat honest:

- The cue sheet is written before the bake, not reverse-engineered from it.
  When the choreography comes from a user's description, draft the cue sheet
  first and confirm the sequence before writing any sampling code; phase
  boundaries are design decisions, and they are much cheaper to move in a
  table than in code.
- Phase times and cue times appear in one place only. A literal time inside
  the sample loop or inside a scheduled haptic is a bug waiting for the next
  retiming.
- Classify every seam. A design seam (a glide easing into a settle, a hold
  drifting into an exit) must be C1: value and velocity at the end of one
  phase equal those at the start of the next. An impulse seam (launch,
  impact, catch) is a deliberate velocity discontinuity, and it is legal
  only at a contact event, always paired with a cue on the sheet (haptic,
  shadow hit, shiver). Because `poseAt` is pure, a unit test can sample
  both sides of every boundary and assert the intended class; an unplanned
  C0 seam shows up on screen as a kick in the middle of a phrase.
- The `cues:` line is not decoration. Each cue becomes a token-guarded
  scheduled continuation (see Generation tokens below), and shadow changes
  ride the same clock as the pose
  ([shadows-and-lights.md](shadows-and-lights.md) covers animating
  `shadowColor` alpha instead of toggling `castsShadow`).
- When timing changes during tuning, the cue sheet comment changes in the
  same commit. A sheet that disagrees with the struct is worse than none.

## Designing weight

A heavy actor is sold by timing decisions made at the cue-sheet stage, not
by material or scale. When drafting the sheet:

- Anticipation: a launch is preceded by a short dip against the coming
  motion. The mass gathers before it leaves; without the dip the actor
  reads as weightless, yanked by an invisible string.
- Time asymmetry: mass rises slowly and falls fast. A flight whose upward
  and downward legs take equal time reads as a balloon, however correct the
  parabola.
- Landings are events, not freeze-frames. A landing continues past
  touchdown: a rebound from around 20 percent of the impact speed that keeps the
  travel drift, then a damped rock or shiver ringing out. Stopping dead on
  contact reads as the video pausing. (For many actors landing into a pile,
  [instancing-and-swarms.md](instancing-and-swarms.md) extends this to
  neighbor nudges.)
- Rigid actors shiver, soft actors squash. Metal takes a high-frequency
  decaying oscillation (20-30 Hz) added onto one channel after an impact;
  squash-and-stretch belongs to rubber, and even there a hard material gets
  a capped hint, never cartoon jelly.
- Every impulse lands a cue. The heavier the motion claims to be, the more
  the impact must be felt through the hand: impact haptic intensity scales
  with the claimed weight, soft ticks are for design-seam moments only.

## The baking recipe

Compute every animated channel as a dense array sampled uniformly in time,
with all easing baked into the values:

```swift
let duration: Float = 1.8
let sampleCount = Int(duration * 80)     // 80-120 samples per second
var positions: [NSValue] = []
var spins: [NSNumber] = []

for step in 0...sampleCount {
    let t = duration * Float(step) / Float(sampleCount)
    let pose = poseAt(time: t)           // one pure function of time
    positions.append(NSValue(scnVector3: pose.position))
    spins.append(NSNumber(value: pose.spin))
}

func baked(_ keyPath: String, _ values: [Any]) -> CAKeyframeAnimation {
    let animation = CAKeyframeAnimation(keyPath: keyPath)
    animation.values = values
    animation.duration = TimeInterval(duration)
    animation.calculationMode = .linear   // easing lives in the values
    return animation
}
```

At 80 to 120 samples per second, linear interpolation between samples is
below visual threshold, and `calculationMode = .linear` with uniform key
times means the value arrays are the single source of truth. No
`timingFunction` stacking, no per-segment curves.

For motion along a curved path, resample the path at equal arc-length steps
first, then apply one easing to arc progress. Easing the curve parameter
instead of arc length makes speed vary with curvature, which the eye reads as
physics being wrong.

## Model value first, then the animation

Set the node's model property to the final pose before adding the animation:

```swift
node.position = finalPosition          // model = where it ends
node.addAnimation(baked("position", positions), forKey: "ride")
```

The animation drives the presentation for its duration; when it completes and
is removed, the node is already at the model value, so there is no snap and
no need for `fillMode` tricks on standard entrances.

## fillMode and cleanup bookkeeping

Sometimes an animation must hold its end state while other work happens (an
exit pose held until the next install). That is `fillMode = .forwards` with
`isRemovedOnCompletion = false`, and it comes with two sharp edges:

- While such an animation is attached, it pins the presentation and silently
  overrides every model value you set afterwards.
- Animations live per object. `node.removeAllAnimations()` does not touch
  animations attached to the node's child geometries, their materials, a
  light, or running `SCNAction`s.

The cure is one central sweep, called from every entry point that takes
manual control (scrub, reset, retarget, gesture grab):

```swift
func clearAnimations() {
    actorNode.removeAllAnimations()
    for child in actorNode.childNodes {
        child.geometry?.removeAllAnimations()
    }
    lightNode.removeAllAnimations()
    lightNode.removeAllActions()          // sequenced fades ride actions
    lightNode.light?.removeAllAnimations()
}
```

Rule: every new form of animation added to the scene (a node transform, a
geometry uniform, a light property, an action) must be added to this sweep in
the same commit.

## Generation tokens

Choreography schedules future work: delayed entrances, chained transitions,
haptics at contact times. Any of it can be superseded by a user action or a
state change. Guard every continuation with a generation token:

```swift
private var generationToken = 0

func runEntrance() {
    generationToken += 1                  // supersede anything pending
    let token = generationToken
    // ... bake and attach animations ...
    DispatchQueue.main.asyncAfter(deadline: .now() + duration) { [weak self] in
        guard let self, self.generationToken == token else { return }
        self.settleHaptic()
    }
}
```

Every beat (entrance, exit, transition, throw) increments the token at its
start and captures it for its scheduled work. An interrupted beat's pending
closures then no-op. With multiple independent beat families, give each its
own token, and have interactions bump all of them (a grab must cancel the
entrance's tick haptics and a drop's roll-home alike).

## Stealing a node mid-animation

When a gesture grabs an animating node, the visible pose is the presentation,
not the model. Snapshot first, then remove:

```swift
let shownX = travelNode.presentation.position.x
let shownSpin = spinNode.presentation.eulerAngles.z
clearAnimations()
travelNode.position.x = shownX            // snap model to what the eye sees
spinNode.eulerAngles.z = shownSpin
```

Order matters: removing animations first snaps the node to its (stale) model
value for a frame, a visible teleport. Snapshot presentation, remove
animations, write the snapshot into the model, then let the gesture drive.
This is what makes baked choreography fully interruptible: the grab catches
the actor exactly where it is.

## Rolling: spin slaved to distance over radius

Wheels and discs must roll, never slide. Do not animate spin as its own
timing curve; derive it from traveled distance in the same sample loop:

```swift
let rolledAngle = traveledArcLength / wheelRadius   // radians
```

Because spin and position come from the same samples, they stay coupled under
any easing, any interruption, any scrub position. Refinements from
production:

- To park a printed face upright at the end, count the angle backward from
  the total (`(progress - 1) * totalArc / radius` ends at exactly zero).
- During a drag, the same rule applies incrementally: grounded horizontal
  movement adds `-(dx / radius)` to spin; airborne movement carries the
  actor with spin frozen.
- After free interactions, spin is an accumulated angle. Exits must roll the
  delta for the distance actually traveled, not jump to an absolute mapping.
- On a DEPTH path (travel with a z component), a fixed-axis euler spin is
  wrong and the eye knows it - the actor reads as a sliding 2D sprite. Roll
  about the axis perpendicular to the instantaneous travel direction:
  accumulate `simd_quatf(angle: distance / radius, axis: normalize(up x
  direction))` per sample and bake the quaternions on `orientation`.
  Interactive straight-line rolling lives on a separate node with a plain
  euler angle - and the NESTING ORDER is load-bearing: the euler roll node
  must be the PARENT and the quaternion tumble node the child. Nested the
  other way, the frozen tumble tilts the roll axis into the depth plane
  and left-right rolling reads as rolling toward and away from the camera.
  Stealing snapshots both presentations.
- When reusing a choreography across actors, audit its SYMMETRY assumptions.
  A coin-like actor (badge, medal) can seat with a half-turn flourish to yaw
  -pi because both faces are identical; an asymmetric actor (a wheel with a
  face and an open back) seated at -pi lands BACK to camera, and the extra
  half turn reads as a face-edge-face stutter right at the seat. For
  asymmetric actors sweep only the leftover path yaw so the seat lands at
  yaw 0 mod 2pi, and guard divisions by the now-small sweep.

## Rolling an actor along a floor path

Rolling a wheel-like actor around a curve on the floor (a lap, a loop, a
figure) composes the rolling rule with steering, and every piece below was
learned the hard way:

- Hierarchy: heading parent, roll child. Bake the heading (euler y) to
  follow the path tangent and the roll (euler about the axle) inside it;
  the parent continuously realigns the roll axis, so no per-sample
  quaternions are needed. Unwrap the heading across samples - atan2 jumps
  at pi.
- Derive the roll's SIGN from the constraint, never intuition: write
  v_center = omega x (center - contact) for your exact node conventions and
  read off which sign rolls forward. A printed face is the instant visual
  test - it must turn with the ground.
- Character comes from the actor's construction: a single-track actor leans
  into turns, a rigid wheel PAIR stays upright and never hops. A flourish
  that contradicts the actor's build reads as a bug, not charm.
- Keep path derivatives analytic. Curvature from finite differences over
  equal-arc-resampled points is second-derivative noise (the resampled
  polyline is piecewise linear); resample the curve PARAMETER at equal arc
  steps and evaluate tangent and curvature from the closed form at that
  parameter.
- Design the figure in SCREEN space, not floor space. A raised camera maps
  depth to screen height: loops laid across the stage read as an infinity
  sign, loops laid into depth stack vertically and read as the digit 8. A
  self-intersecting curve offers several tangent branches at the crossing
  (a symmetric lemniscate crosses at 90 degrees; squashed aspects bring the branches nearer, ~80) - check every branch
  before concluding the staging is forced.
- The figure must dwarf the actor: a loop smaller than the actor's own
  length is obscured by it and reads as broken motion, roughly 2x the
  longest dimension is the floor. Scale the total arc to an INTEGER number
  of circumferences and the printed face parks upright by construction.
- Screen pace, not path pace: weight arc speed by camera distance (gentle
  exponent, composed inside the easing so end slopes stay zero) and let an
  enveloped camera dolly ride the near passes. Normalize any such derived
  channel to zero at rest - a smooth-max has a floor at zero input, and an
  envelope leaks it as an unmotivated camera drift at the beat's edges.
- Debug with a trail, not with eyes: see "Draw the line" below - it has
  its own section because it keeps earning it.

## The visible window: eases on paths that leave the frame

An entrance or exit path has segments outside the frustum, and an easing
curve does not know that. Tune the ease against the VISIBLE WINDOW of the
path, not its endpoints, or the ramps land where nobody can see them:

- A symmetric smoothstep on a path that starts offscreen spends its slow
  attack crawling through the invisible stretch - on screen that is pure
  dead air before the actor appears. The audience reads "nothing is
  happening", not "it is easing in".
- A pure ease-out dies at the far end: if the path also EXITS offscreen,
  the actor visibly stalls at the frame edge before creeping out - and the
  decaying tail pads seconds of invisible crawl onto the beat, stretching
  the whole sequence for nothing.
- The rule: the arc-rate must be alive (nonzero slope) at BOTH edges of
  the visible window. Ramp up and down offscreen if the shape needs it,
  but what crosses the frame edge moves.
- A clean composite shape for enter-perform-exit paths: a fast-attack
  curve carries the entry and the featured middle (tuned on its own
  clock), and from the middle's end the arc-rate is FROZEN at the
  junction value (C1) all the way offstage - constant-speed exit, no
  taper, and the beat's duration shrinks to what the viewer actually
  sees. Keep the cue inverse piecewise to match.

## The start tangent of an appended path

A path appended to a RESTING actor (a wake-up-and-leave, a retreat after a
fall) must start with its tangent exactly on the actor's current roll
axis. Heading is derived from the sampled tangent, so any sideways
component in the first control point snaps the yaw in the bake's first
frame - and actor length amplifies the angle: 6 degrees is invisible on a
ball and moves the ends of a two-meter bar a full hand's width in one
frame. Use a cubic bezier and place the FIRST control point dead on the
axis (zero lateral offset); let the curve develop from the second control
point on. The same discipline applies at any junction where a bake picks
up from a pose the viewer has had time to study.

Rolling BACKWARD along an appended path (the actor leaves without turning
to face its exit) is legitimate and often better than a half-turn: keep
the yaw continuous by solving heading from the NEGATED tangent, and feed
the spin the signed arc (dot of the step with the heading's floor
direction) so the roll direction stays honest through the whole curve.

## Draw the line: debugging motion with a trail

When a motion bug is reported - "the actor jumps", "the curve looks
wrong", "it clips something on the way out" - draw the path before
arguing with the animation. Render the bake's OWN points (the same
resampled array the sampler consumes) as a flat ribbon on the floor:
triangle strip, ~4 screen points wide, constant-lit red, emission set,
`readsFromDepthBuffer = false`, high rendering order, `castsShadow` off.
Fifteen lines of code, and the invisible becomes reviewable:

- If the ribbon and the motion disagree, the bake is wrong. If they
  agree, the problem is the path's SHAPE or the staging - a different
  fix entirely. This one distinction kills half the hypothesis space in
  a single frame.
- The ribbon exposes what stills and playback both hide: a loop smaller
  than the actor, a corner where a smooth arc was intended, a path that
  crosses wreckage the actor was supposed to clear, an exit that ends
  inside the frustum.
- It is also a DIRECTION tool, not only a debugging tool: show the drawn
  line to the person art-directing the motion and iterate on the line -
  cheaper than iterating on the finished bake, and their sketch and your
  ribbon speak the same language.
- Keep it behind a toggle or a temporary call, and delete or disable it
  when the shape is approved; a leftover trail in a recording is the one
  bug the reviewer never forgives.

## Channels beyond transforms

The bake does not care what its key path points at. Anything SceneKit can
animate is a channel for `poseAt`, sampled in the same loop and attached on
the same clock as the transforms:

- Shadow and light properties - this skill already rides `shadowColor`
  alpha instead of toggling `castsShadow`.
- Camera lens values - a reveal flight bakes `focusDistance` against the
  dolly so focus arrives with the camera.
- Shader-modifier uniforms via their material key path - an engraving glow
  or a ripple amplitude becomes one more column in the sample loop (mind
  the linear-space trap in
  [stage-recipe.md](stage-recipe.md)).
- Morpher weights. Give the actor's geometry an `SCNMorpher` whose targets
  share the base topology, and `morpher.weights[i]` is an animatable key
  path like any other: a soft actor's squash pose or a face's shape key
  bakes into the choreography instead of fighting it from a timer. The
  cleanup sweep must then include the morpher's node like every other
  animated object.

The test for "should this be a channel": if it must stay synchronized with
the motion, it goes through `poseAt` and the bake; if it is a reaction to a
tap with no beat around it, an `SCNTransaction` tween is enough.

## Springs as authoring material

A settle's feel does not have to be hand-tuned decay constants. A perceptual
spring - defined by nothing but a duration and a bounce - can be SAMPLED
OFFLINE: query its value and velocity at each step of the sample loop and the
spring becomes rows in the baked arrays like any other math, with its settling
duration handed to the cue sheet as the phase length. Two payoffs specific to
a 3D stage:

- The actor can settle with the same feel as the host app's own chrome: match
  the (duration, bounce) pair the surrounding UI uses and the 3D beat and the
  2D transitions stop feeling like two different products on one screen.
- A grab's release velocity feeds the spring's initial velocity directly, so
  the hand-off from finger to settle carries momentum instead of restarting
  from rest - the same rule this file already applies to hand-rolled physics.

## One property, one owner

Every animated property on the stage has exactly one owner, and the owner is
the bake. Three corollaries:

- SwiftUI animation and Core Animation must never both drive the same
  property. An ancestor's `withAnimation` will happily interpolate the
  representable's frame or a bridged value mid-beat; armor the stage by
  clearing the inherited animation from the transaction at the stage's
  boundary.
- Never route per-frame stage values (camera position, node transforms)
  through a SwiftUI animatable wrapper: that evaluates a view body on the
  main thread every frame to move a value the render server could have moved
  for free - the exact stall the bake exists to avoid.
- Interruption semantics differ by family, and it matters when stealing:
  spring-family animations RETARGET, inheriting in-flight velocity; curve-
  family animations run additively alongside the old one. Baked keyframes do
  neither, which is why this file's snapshot-steal (presentation into model,
  then clear) is the only interruption that works for them - the manual
  equivalent of a spring's velocity-preserving retarget.

## Shared timing curves for play and scrub

If a motion can both play (CA animation) and scrub (a slider setting values
directly), define its easing once as cubic bezier control points and derive
both consumers from that single definition:

- the played side: `CAMediaTimingFunction(controlPoints:)`
- the scrub side: solve the bezier's x(t) = progress numerically (Newton with
  a few iterations converges; bisection also works) and evaluate y(t)

A frame frozen by the scrubber then matches the same instant of a played run
to sub-pixel accuracy, and the two code paths cannot drift apart.
