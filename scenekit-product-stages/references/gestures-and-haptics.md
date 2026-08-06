# Gestures and Haptics

## Contents

- Screen-to-world without hit testing
- The grab protocol
- Soft clamps while held
- Release velocity
- Haptic grammar
- Preference and Reduce Motion gating
- Visibility lifecycle for 60 Hz work
- Scripted beats versus live fingers
- Simulator notes
- Sharing the screen with SwiftUI gestures
- The invisible stage

## Screen-to-world without hit testing

A product stage has one actor, so `SCNView.hitTest` is overkill and adds
per-touch cost. Convert gesture points to scene units in closed form from the
camera frustum:

```swift
// Distance from the camera eye to the actor's plane, and the camera's
// vertical field of view at that plane, give the visible height in units.
var unitsPerPoint: Float {
    guard let view, view.bounds.height > 0 else { return 0.01 }
    let visibleHeight = 2 * eyeToActorDistance * tan(verticalFOV * .pi / 360)
    return Float(visibleHeight / view.bounds.height)
}
```

Then a pan translation in points times `unitsPerPoint` is a translation in
scene units at the actor's depth. One caveat: know which axis your FOV
anchors. With `projectionDirection = .horizontal` the horizontal FOV is
fixed and the vertical one derives from the view's aspect ratio (and vice
versa), so compute the visible extent for the dimension your conversion uses,
from the actual `view.bounds` aspect, or drags will track slightly off on
other screen shapes.

The same closed-form thinking applies to stage limits: derive the lift
ceiling by solving where the view's top edge intersects the actor's plane
given the camera position, pitch, and FOV. A linear estimate clips the actor
on some screens; the exact projection never does.

## The grab protocol

Grabbing an animating actor, in strict order (details in
[baked-animation.md](baked-animation.md)):

1. Refuse the grab during exit windows (the actor is about to be replaced;
   a grabbed ghost with a stale drag base teleports).
2. Bump every generation token: the entrance's scheduled haptics, a flight's
   pending impacts, a chained transition, all superseded.
3. Snapshot `.presentation` values for every animated channel.
4. Remove animations through the central sweep.
5. Write the snapshots into the model values; record them as drag bases.

Then `.changed` events drive the model directly, and only if `.began` was
accepted (guard the dragging flag; a `.changed` after a refused `.began`
must be ignored wholesale).

## Soft clamps while held

While the finger holds the actor, hard walls feel like bugs. Use asymptotic
give past every soft limit and a hard clamp only where the world is truly
rigid:

```swift
// Sideways: free across the stage, exponential give past the wall.
func heldX(for raw: Float) -> Float {
    let magnitude = abs(raw)
    guard magnitude > wallX else { return raw }
    let over = magnitude - wallX
    let held = wallX + give * (1 - exp(-over * 0.4 / give))
    return raw < 0 ? -held : held
}

// Upward: 1:1 to a knee, asymptotic give under the ceiling.
// Downward: dead. The floor is exactly where the actor stands.
```

The exponential form has no discontinuity at the limit and never quite
reaches the asymptote: the actor can lean past the edge but never leave the
hand. Grounded horizontal movement should roll the actor
(`spin -= dx / radius`); airborne movement carries it with spin frozen, which is
what a held object does.

## Release velocity

`UIPanGestureRecognizer.velocity(in:)` is in points per second; multiply by
`unitsPerPoint` and feed the result to the integrator (see
[physics-without-engine.md](physics-without-engine.md)). Flip the y sign
(screen y grows downward, stage y grows upward). Apply the per-variant energy
caps at this handoff, never during flight.

## Haptic grammar

Haptics are the physics the fingers verify. The grammar that reads as real:

- Impact haptics at the integrator's recorded contact times, scheduled with
  `asyncAfter` and re-checked against the generation token so an interrupted
  flight goes silent.
- Intensity from impact speed, floored and capped:
  `min(1, base + speed / divisor)`. A grazing contact fires nothing.
- A contact-count cap per bake (five or six); a long rattle of taps reads as
  a malfunction.
- Generator style per actor variant (soft for light variants, heavy for
  heavy ones); mass becomes something the wrist believes.
- Soft single ticks for choreography beats (a settle, a seat) at lower
  intensity than any impact.
- Reuse PREPARED generators, one per style, cached for the stage's lifetime:

  ```swift
  @MainActor
  enum StageHaptics {
      static let soft = UIImpactFeedbackGenerator(style: .soft)
      static let medium = UIImpactFeedbackGenerator(style: .medium)
      static let rigid = UIImpactFeedbackGenerator(style: .rigid)
  }
  ```

  A generator built inline inside each scheduled `asyncAfter` closure
  allocates and spins the Taptic engine up cold on every tick - a baked
  flight schedules a handful of them.

## Preference and Reduce Motion gating

- Every haptic call sits behind the app's haptics preference. One global
  check function; no exceptions.
- `UIAccessibility.isReduceMotionEnabled` gates ambient motion: gyro
  parallax, idle sway, shake-driven extras. Choreographed functional motion
  (an entrance that conveys state change) may remain, but decorative motion
  must not.
- Smooth gyro input (low-pass, for example `value * 0.78 + target * 0.22`)
  and clamp the tilt range; raw attitude jitters.

## Visibility lifecycle for 60 Hz work

A motion-update loop mutating the scene at 60 Hz on the main thread must not
outlive the stage's visibility. The reliable signal for a
`UIViewRepresentable` is window membership, not SwiftUI appearance callbacks:

```swift
final class StageView: SCNView {
    var onWindowChange: ((Bool) -> Void)?
    override func didMoveToWindow() {
        super.didMoveToWindow()
        onWindowChange?(window != nil)
    }
}
```

Start device-motion updates when the view gains a window, stop them when it
loses one (tab switches and covering pushes detach the view; popping back
reattaches it). Two extra rules from production:

- On stop, discard the gyro reference attitude. The device may sit at a
  completely different orientation on the next start, and a stale reference
  slams the parallax to its clamp on the first callback. Re-reference lazily
  on the first new sample.
- Also wire `dismantleUIView` to the same stop, for the teardown path.

## Scripted beats versus live fingers

A stage inside a scripted sequence has two masters: the script's beats and
the user's hands, and they collide in ways single-owner stages never see.
Four rules, each from a real corpse:

- A trigger-diff latches BEFORE the action runs. If the action then
  refuses on a guard (`!isDragging`, actor mid-beat), the beat is consumed
  and gone - the script marches on with a silently missing pose, load, or
  finale, and the failure surfaces minutes later as "the reel just
  stopped". Every scripted beat that can meet a finger gets a BOUNDED
  RETRY between the request and the run: poll every ~0.15s until the
  guards pass, give up after a budget (~3s). Never a single shot with a
  silent guard.
- A scheduled poll is interruptible work and carries a generation token
  like any other continuation. An un-tokened topple poll waits out the
  user's grab and then knocks the actor over AGAIN the moment they set it
  down - technically "the beat finally passed its guards", visibly a
  haunted object. Snapshot the drop token at request time; any grab or
  throw retires the poll.
- Gestures must refuse actors that are not on stage yet. A pan or shake
  during the warm-up curtain (actor still hidden, first entrance pending)
  bumps the generation tokens, the pending entrance's token check fails
  forever, and nothing else ever unhides the actor - a permanently empty
  stage with no error. Guard every gesture entry point on the actor being
  mounted AND visible before touching any token.
- Stage code that STEALS the actor from a live gesture (a transition
  firing mid-hold) must close the gesture's callback contract - emit the
  release callback before clearing the drag flag. The swallowed `.ended`
  otherwise leaves the host's interaction state latched, and every
  host-side decision built on "is the finger down" is wrong from then on.

## Simulator notes

The simulator has no accelerometer, so accelerometer-based shake detection is
dead there. Support the system shake gesture as a parallel path (the view
must return true from `canBecomeFirstResponder` and override
`motionEnded(_:with:)`); the simulator sends it via its shake menu command.
Gate the fallback to run only when device motion is unavailable, so hardware
always goes through the real per-variant thresholds.

## Sharing the screen with SwiftUI gestures

The stage's pan recognizer is a UIKit citizen living inside a SwiftUI world,
and when the host screen adds gestures of its own, arbitration is decided by
rules worth knowing before the fight starts: a plain SwiftUI `.gesture`
LOSES to recognizers already owned by the view or its children (the stage's
pan wins by default), a high-priority gesture wins over them, and
coexistence requires consent from BOTH sides - the coordinator answers yes
to simultaneous recognition for the stage's recognizers, and the SwiftUI
side attaches its gesture as simultaneous rather than plain. A scrolling
container is the hard case: it will take the pan for itself, and that is a
design decision (which axis belongs to whom), not a bug to out-shout.

## The invisible stage

A transparent SCNView with a pan recognizer is NOTHING to VoiceOver: no
frame, no name, no action. The stage must claim its own presence - one
accessibility element covering the stage, labeled as the actor
("chrome wheel"), its value reporting the current state ("resting",
"mid spin"). Interaction maps naturally: the grab-and-turn becomes an
adjustable action (swipe up and down step the rotation), and each beat
becomes a named custom action ("burnout", "roll a lap") so the choreography
is triggerable without ever seeing it.

Reduce Motion is a taxonomy, not a switch. Ambient and looping motion - the
idle drift, a turntable, gyro parallax - is REMOVED outright and a static
pose stands in; one-shot beats may survive, but their travel shortens and
their entrances become fades rather than flights. Haptics are not motion
and stay. The existing rule (gate the extras) is the floor; classifying
each motion into removed / softened / kept is the actual job.
