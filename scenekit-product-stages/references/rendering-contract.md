# The Rendering Contract

A shader-driven stage that can PROVE itself: same scene, same clock, same
pixels - every launch, every device, every refresh rate. This is the
difference between a pretty demo and a stage you can regression-test; the
motivating review said it plainly: metal effects regress first at resize,
dark mode, and 60/120 Hz boundaries. The contract is cheap when built in
from the start and miserable to retrofit.

## Contents

- One injectable clock owns all motion
- The freeze hook
- Back-dating events on a frozen clock
- What else must be pinned
- The proof kit
- Synthetic gestures, and their two traps

## One injectable clock owns all motion

Everything that moves rides ONE clock, advanced by dt in the renderer
delegate - the same integrated clock that fixes the speed-dial teleport
in [fog-and-mist-sheets.md](fog-and-mist-sheets.md), promoted from a
phase fix to the architecture:

```swift
func renderer(_ renderer: SCNSceneRenderer, updateAtTime time: TimeInterval) {
    let dt = Float(min(time - (lastTime ?? time), 0.1))
    lastTime = time
    if frozen == nil, !paused { clock += dt }
    let value = frozen ?? clock
    material?.setValue(value, forKey: "stageClock")   // shader motion
    spinNode?.simdOrientation = pose(at: value)       // node motion
}
```

The rules that make it a contract:

- dt-INTEGRATED, never frame-counted: a 120 Hz device takes twice the
  steps through the same second and lands on the same picture.
- The clock drives shader uniforms AND node poses. A pose left on
  `SCNAction` or a timer keeps moving while the shader freezes - the
  contract must extend to the node graph.
- Gesture inertia integrates on the same dt (see the trackball in
  [gestures-and-haptics.md](gestures-and-haptics.md)), so a flick feels
  identical at every refresh rate.
- One writer: the render thread owns the node's transform. Gestures hand
  rotation increments and velocities to the clock object under a lock;
  nobody else touches the node.

## The freeze hook

The clock is injectable through a launch argument, so any run can be
pinned to a fixed instant:

```
simctl launch <udid> com.example.app -example orb -freeze 5
```

`frozen` overrides the clock; the delegate keeps running (SceneKit still
renders continuously) but every frame renders the same instant. Verified
result worth stating exactly: two independent launches with the same
freeze argument produce BIT-IDENTICAL screenshots - `ImageChops.difference`
returns an empty bbox. That is the baseline every future change is
diffed against.

## Back-dating events on a frozen clock

Interactive events are born ON the clock (`birth = clock.current`), which
makes them freezable too - but an event born AT the frozen instant has
age zero forever and shows nothing. The hook back-dates it:

```swift
poke(at: center, birthOffset: frozen == nil ? 0 : -0.7)
```

A `-poke` launch argument plus `-freeze 5` renders the ripple mid-flight,
deterministically - the interaction becomes a snapshotable state instead
of a thing you can only see live.

## What else must be pinned

The clock is necessary, not sufficient. The rest of the determinism
checklist:

- `wantsExposureAdaptation = false`. Adaptive exposure renders the same
  clock instant differently depending on what the camera saw before -
  the frames LOOK equal and diff nonzero. Fix exposure by hand
  ([hdr-environments.md](hdr-environments.md)).
- No wall-clock or randomness in the render path: no `Date()`,
  `CACurrentMediaTime()` offsets, or `random` in anything the clock
  drives. Seeded randomness baked at build time is fine.
- Reduce Motion pauses the clock rather than branching the scene: the
  paused stage still answers taps (events born on a paused clock render
  their full life once unpaused, or back-dated while paused).
- State the picture depends on must be reachable from launch arguments
  (`-material gold -orblight -amp 0.75 ...`), or the snapshot matrix
  cannot be scripted.

## The proof kit

Three cheap measurements, each answering one question:

- MOTION: two live screenshots seconds apart, pixel-diff. Nonzero proves
  the scene animates (the `rendersContinuously` trap fails this within
  seconds). Never judge motion by eye from one still.
- DETERMINISM: launch, freeze, screenshot, terminate; repeat; diff. Empty
  bbox or it is not a contract. Run per theme and per finish - the matrix
  is exactly the launch-argument space.
- INTERACTION: freeze the clock, screenshot, perform the gesture,
  screenshot, diff. On a frozen clock NOTHING else moves, so a diff
  bounded to the actor's area is proof the gesture (and only the
  gesture) acted. This separates "the blob changed because it breathes"
  from "the blob changed because the drag worked".

`showsStatistics` behind a `-stats` launch argument covers the last
review demand - frame time read on a real device, not asserted from the
simulator.

## Synthetic gestures, and their two traps

A drag can be scripted from outside the simulator: a 30-line CGEvent tool
(mouse down, stepped drags, mouse up) pointed at the Simulator window,
whose frame System Events reports. Two traps, both paid for:

- STALE WINDOW FRAMES. Read the window's position IMMEDIATELY before
  posting events. A frame cached from an earlier step aimed a drag at a
  moved window and landed it on a SwiftUI slider - the "rotation test"
  actually dragged the blob amplitude to zero, and the perfect sphere in
  the diff looked like a shader bug.
- THE HUMAN IN THE LOOP. An open Simulator window belongs to the user,
  and they WILL interact with it between your launches. When scripted
  runs show impossible state - a finish nobody launched, a dial nobody
  set - suspect fingers before code. Assert the expected state from the
  screenshot itself (which segment is selected, where the slider sits)
  rather than trusting the launch arguments alone.
