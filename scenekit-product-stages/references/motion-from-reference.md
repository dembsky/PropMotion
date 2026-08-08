# Motion from Reference

Reproducing the motion of a real object - an appliance flap, a lid, a
mechanism the client owns and can film - is a different job from designing
motion freely. The reference is the contract, and the fastest way to blow two
days is to trust the wrong part of it. Everything here was paid for on a real
product: a wall unit's air flap that rides a hidden guided linkage, animated
against photos, frames, and finally the owner's own hands.

## Contents

- What a reference can and cannot tell you
- Model the path, not the mechanism
- The calibration rig: let the owner pose the actor
- From saved poses to a curve
- The uniform Catmull-Rom trap
- Prove continuity numerically before you build
- Seamless state loops, and leaving them from anywhere
- Measuring meshes: the trimmed-index trap

## What a reference can and cannot tell you

- Still photos carry POSES. They do not carry the path between poses, and
  they do not carry the mechanism. Two rigs with identical end poses can move
  through completely different intermediate poses.
- Video carries ORDER and CHARACTER: which edge separates first, what
  reveals when, whether the motor eases or runs constant. It still does not
  carry numbers you can type into a transform.
- Perspective lies about angles. A curved shell photographed from below reads
  as a different rotation than its chord actually has; every degree you
  "measure" off a photo of a non-flat part is a guess.
- Confidence is not correlated with correctness here. A derived axis with
  four decimal places looks like a measurement and is still a guess. Treat
  every spatial number as a hypothesis until it survives an external check:
  a mesh measurement, a numeric invariant, or the owner's eye on live
  controls.

## Model the path, not the mechanism

The instinct is to reverse-engineer the mechanism: find the hinge, place the
axis, rotate. Resist it. Real products ride guided linkages, cams, and
four-bar mechanisms whose travel no single fixed pivot reproduces. A pivot
model fitted to the end pose will "verify" perfectly - and lie about every
intermediate frame, because end-pose constraints do not determine the path.

The path itself is the only honest model: a sequence of keyframe poses with
an interpolation whose quality you can prove. This is the same doctrine as
[baked-animation.md](baked-animation.md) - choreography as data - applied to
data you must first capture.

If a mechanism model keeps failing sideways (each fix reveals a new wrong),
that is the tell: stop fitting, start capturing.

## The calibration rig: let the owner pose the actor

The person who owns the physical object is a better measuring instrument
than any amount of photo reading. Give them instruments, not questions:

- Direct pose controls in a debug view: translation from the rest pose plus
  rotation about the actor's own centre. Three sliders cover any rigid 2D
  motion; six cover 3D. The sliders must drive the POSE, never mechanism
  parameters - a slider for "hinge depth" presumes the mechanism you are
  trying not to guess.
- A save button that appends the current pose to a JSON file. Each saved
  pose is one keyframe. The owner steps the real object through its motion
  (many appliances can be paused mid-travel) and matches the model to it,
  phase by phase.
- A play button that runs the interpolated curve through the saved poses,
  so authoring and review live in the same screen.

```swift
struct ActorPose: Codable {
    var lift: Float    // metres up from the rest pose
    var depth: Float   // metres towards the camera
    var turn: Float    // radians about the actor's own centre
}
```

Five to ten saved poses reproduce a motion that three days of mechanism
fitting could not. The rig is throwaway; delete the controls once the
keyframes are approved and bake them as data.

## From saved poses to a curve

Hand-saved keyframes arrive unevenly spaced and slightly noisy. Three steps
before interpolating:

1. Parametrize by distance, not by index. Assign each keyframe a phase from
   cumulative travel in pose space, with rotation weighted as arc length on
   a representative lever (for a 10 cm panel, radians times 0.04 m works).
   Index-based spacing makes the curve rush short segments and crawl long
   ones.
2. Merge stacked saves. Two keyframes within a millimetre of combined travel
   force the curve into a stop-and-go at that spot; keep the later one.
3. Interpolate with span-weighted tangents. Cubic Hermite with
   finite-difference tangents divided by the actual knot spans is C1: the
   velocity entering each keyframe equals the velocity leaving it. A natural
   cubic spline (C2) is smoother still. Both pass exactly through every
   saved pose.

```swift
// Span-weighted finite differences: C1 at every knot.
tangent[i] = (p[i + 1] - p[i - 1]) / (t[i + 1] - t[i - 1])
// Endpoints: one-sided differences over their single span.
```

## The uniform Catmull-Rom trap

The textbook Catmull-Rom basis assumes equally spaced knots. Feed it
distance-parametrized keyframes and normalize t per segment, and the curve
still passes through every pose - but the velocity STEPS at every knot,
because the tangents ignore that neighbouring spans differ. On screen this
reads as rhythmic stutter precisely at the authored keyframes.

The trap is expensive because it masquerades as a rendering problem. The
stutter survives every timing fix - display links, baked animations, engine
ownership - since the jerks are baked into the path itself. Two full
debugging rounds went to the renderer before the curve was interrogated.
When smooth playback of a keyframed motion stutters, check the curve math
first: it is the only suspect the renderer cannot exonerate.

## Prove continuity numerically before you build

The interpolation is pure math, so its quality is provable without a device.
Keep the curve code in its own file with no engine imports, compile it into
a command-line harness beside the app, and gate on it:

```
swiftc -O check.swift MotionPath.swift -o check && ./check
```

The check samples the curve densely (about 180 poses), differentiates twice,
and prints the maximum acceleration step between adjacent samples. On the
flap's real keyframe set: uniform Catmull-Rom 7.6, span-weighted Hermite
0.047, natural cubic 0.011. A hundredfold gap - visible in numbers before a
single build, invisible in code review.

## Seamless state loops, and leaving them from anywhere

Many real objects have an oscillating state (a sweeping flap, a scanning
head) that must loop forever and stop cleanly:

- Drive the loop's phase with a cosine over the sub-range of the path the
  state actually sweeps: `phase = mid + amplitude * cos(2π t)`. The cycle
  starts and ends at the same pose with zero velocity at both reversals, so
  one baked cycle under `SCNAction.repeatForever` (or a repeated
  `CAKeyframeAnimation`) loops without a seam and reverses without a snap.
- Sweep sub-ranges come from function, not frames: a flap that aims an air
  stream tilts within its deployed range and never travels back towards
  closed, whatever a mid-swing photo seems to show. Ask the owner what the
  state DOES before reading its extent off stills.
- Leaving the loop: the exit animation must start from the pose the loop is
  currently showing. Recover it from the animation clock - the loop is a
  known function of its playback time, so current phase is exact, no
  presentation-layer reads needed - then run the exit path from that phase,
  scaling duration by remaining travel so the apparent motor speed stays
  constant from any starting point.

```swift
let t = fmod(loopTime, cycleDuration) / cycleDuration
let phase = mid + amplitude * cos(2 * .pi * Float(t))
// exit: sample the path from `phase` down to 0,
// duration = fullExitDuration * phase, floored for very short exits
```

## Measuring meshes: the trimmed-index trap

Reference-driven motion eventually needs real numbers off the asset: where
an edge sits, how deep the part is. ModelIO answers in two minutes:

```swift
let asset = MDLAsset(url: url)
let bounds = asset.boundingBox   // honest for the asset as a whole
```

One trap when the actor was cut out of a larger scanned or exported model:
the cut is often made by trimming the INDEX buffer only, so the sub-mesh's
vertex buffer still carries every vertex of the original object. Walk the
raw vertex buffer and you measure the whole product while believing you
measured the part - bounds off by the size of the entire casing, with no
error anywhere. Collect only vertices referenced by the submesh index
buffers:

```swift
for case let submesh as MDLSubmesh in mesh.submeshes ?? [] {
    let map = submesh.indexBuffer.map()
    // read UInt16/UInt32 indices, collect the referenced set,
    // then fetch positions for those indices only
}
```

The same discipline applies to `SCNGeometry` sources: geometry elements are
index buffers, and only referenced vertices belong to the part.
