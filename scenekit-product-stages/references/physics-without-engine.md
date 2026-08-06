# Physics Without the Engine

## Contents

- Why not SCNPhysicsBody
- The integrator core
- The world as plain numbers
- Caps at release, never mid-flight
- Contacts drive haptics
- Secondary motion from the same samples
- The settle: respect the throw, recover the stage
- Shake-to-topple: a rim-pivot integrator
- A compound rigid fall about a pivot line
- Soft bodies: squash from the contact list
- Testability

## Why not SCNPhysicsBody

For a choreographed product stage (one actor, a floor, a couple of walls),
`SCNPhysicsBody` gives you a live simulation you cannot fully own: results
vary run to run, the motion cannot be baked or scrubbed, interrupting it
cleanly is awkward, and tuning feel means fighting a black box.

The alternative: a hand-rolled fixed-step
integrator run synchronously at gesture release, producing sample arrays that
are baked into `CAKeyframeAnimation` (see
[baked-animation.md](baked-animation.md)). Properties:

- Deterministic: same release velocity, same flight, every time.
- Bake-able: the output is keyframes, so it runs on the render server and
  composes with all other baked choreography.
- Interruptible: a grab snapshots the presentation and takes over; nothing is
  simulating in the background.
- Testable: the integrator is a pure function from initial conditions to
  sample arrays.

The whole "simulation" for a several-second throw integrates in well under a
millisecond. There is no runtime cost argument for the engine here.

## The integrator core

Fixed step, simple Euler, one gravity constant, a small struct of feel
parameters per actor variant:

```swift
struct BounceProfile {
    let restitution: Float     // 0.2 heavy and dead ... 0.5 light and lively
    let wobble: Float          // peak bank wobble a hard impact kicks off
    let tossHeadroom: Float    // max apex above release, scene units
}

func integrateThrow(x0: Float, y0: Float, vx0: Float, vy0: Float,
                    profile: BounceProfile) -> ThrowBake {
    let dt: Float = 1 / 120
    let gravity: Float = 43.6       // 9.81 m/s^2 mapped to scene units
    var x = x0, y = y0, vx = vx0, vy = vy0
    var xs: [Float] = [x], ys: [Float] = [y]
    var contacts: [(time: Float, speed: Float, wall: Bool)] = []
    var grounded = y <= 0 && vy <= 0
    var time: Float = 0

    while time < 6 {                // hard cap: no infinite bakes
        time += dt
        if grounded {
            let decel = rollFriction * dt
            vx = abs(vx) <= decel ? 0 : vx - decel * (vx > 0 ? 1 : -1)
            x += vx * dt
        } else {
            vy -= gravity * dt
            y += vy * dt
            x += vx * dt
            if y <= 0 {             // floor contact
                y = 0
                let speed = -vy
                contacts.append((time, speed, false))
                let rebound = profile.restitution * speed
                if rebound * rebound / (2 * gravity) < restApex {
                    vy = 0; grounded = true    // apex too small: it rests
                } else {
                    vy = rebound
                }
                vx *= 0.95          // the touch costs a little speed
            }
        }
        if abs(x) >= wallX {        // wall contact
            x = x > 0 ? wallX : -wallX      // clamp unconditionally
            if x * vx > 0 {                 // reflect only OUTWARD motion
                contacts.append((time, abs(vx), true))
                vx = -profile.restitution * vx
            }
        }
        xs.append(x); ys.append(y)
        if grounded, abs(vx) < 0.05 { break }
    }
    return ThrowBake(xs: xs, ys: ys, contacts: contacts, dt: dt)
}
```

Choices that matter:

- dt = 1/120 both integrates stably at these speeds and doubles as the
  keyframe sample rate; the arrays go straight into the bake.
- Map gravity from real scale: decide what one scene unit is in centimeters
  and convert 9.81 m/s2. Real gravity is what makes a small on-screen object
  feel heavy.
- End bounce with an apex test (would the rebound rise above a threshold?),
  not a velocity epsilon; it terminates cleanly without micro-bounce shimmer.
- Bound the loop with a hard time cap so no parameter combination can bake
  forever.

## The world as plain numbers

There is no collision system. The floor is `y = 0`. The walls are
`abs(x) >= wallX`. The ceiling is a clamp in the gesture layer. That is the
entire world model, and it is enough because the stage is designed: you know
where its edges are.

Branching on app state lives right in the integrator, and that is a feature.
Example: reaching a wall fast enough toward a direction the app can navigate
turns the bounce branch into an exit branch (keep at least a minimum exit
velocity, integrate until fully offstage, and schedule the navigation
callback at the sample time where the actor is visually gone). Too slow, or
no destination: an ordinary wall bounce the user can feel as a dead end.

## Caps at release, never mid-flight

Clamp energy once, when the gesture hands over, and let the flight be honest:

```swift
let headroom = min(profile.tossHeadroom, max(ceiling - y, 0))
let tossCap = sqrt(2 * gravity * headroom)      // apex-derived velocity cap
vy = vy > 0 ? min(vy, tossCap) : max(vy, -slamCap)
vx = max(-hurlCap, min(hurlCap, vx))
```

Deriving the vertical cap from allowed apex height (v = sqrt(2gh)) keeps the
guarantee geometric: the actor cannot leave the stage upward no matter the
fling. Clamping mid-flight instead produces visible ceilings and broken
parabolas.

## Contacts drive haptics

The integrator records contacts; after baking, schedule one haptic per
significant contact at its sample time, guarded by the same generation token
as the bake:

```swift
for contact in contacts.prefix(5) where contact.speed > 0.4 {
    let intensity = min(1, 0.25 + contact.speed / 9)
    DispatchQueue.main.asyncAfter(deadline: .now() + TimeInterval(contact.time)) {
        [weak self] in
        guard let self, self.throwToken == token else { return }
        UIImpactFeedbackGenerator(style: profile.hapticStyle)
            .impactOccurred(intensity: intensity)
    }
}
```

Cap the count (a rattle of twelve haptics feels like a malfunction), floor
the speed (a grazing touch is not an event), and re-check the token so a grab
mid-flight cancels the remaining taps. Intensity from impact speed is what
sells mass; a heavier actor variant pairs a deader restitution with a heavier
generator style.

## Secondary motion from the same samples

Everything secondary derives from the same arrays in the same pass:

- Contact blob: scale and opacity from the height gap per sample (thinner
  and fainter as the actor rises), baked onto the blob node.
- Impact wobble: at each hard contact, kick a damped oscillation and sum
  them per sample: `amp * sin(f * (t - t0)) * exp(-d * (t - t0))`, baked onto
  a bank rotation. Never a separate scheduled animation; it belongs to the
  same function of time.
- Rolling spin: grounded segments add `-(dx / radius)` to spin per sample
  (see [baked-animation.md](baked-animation.md)).

## The settle: respect the throw, recover the stage

A designed stage usually wants the actor back at its mark, but snapping back
insults the throw. The pattern that feels right: treat the floor as a shallow
bowl. Wherever the actor stops, hold the spot for about a second (the throw
was respected, you saw it land), then ease it home over a duration scaled by
distance, spin still slaved to the traveled distance, with a soft haptic as
it seats. All of it is appended to the same bake, so the entire
throw-rest-return is one interruptible animation.

## Shake-to-topple: a rim-pivot integrator

A second, separate integrator handles falling flat, pivoting on the rim:

- Teeter: a short growing oscillation (about 0.3 s) whose last sway hands its
  amplitude and angular velocity to the fall, so there is no seam between the
  rock and the give.
- Fall: `omega += k * gravity * sin(angle) * dt; angle += omega * dt`, with k
  around 0.8 for a disc tipping about its rim. Torque growing with the sine
  of the lean gives the characteristic slow start that accelerates.
- Impact and rattle: when the face reaches flat, record the impact, invert
  omega times a deadened restitution (face-on contact is deader than a
  rolling rim; around half the bounce restitution reads right), repeat until
  the rebound is negligible. A dropped-coin rattle for free.
- Geometry: the pivot is the rim, not the center. Per sample, pitch by the
  angle and compensate position by r * (cos(angle) - 1) vertically and
  -r * sin(angle) in depth (r = pivot-to-center distance) so the contact
  edge never leaves the floor.
- Gate the trigger: require a real oscillation (several accelerometer spikes
  over a threshold within a window, spaced apart), so a single bump never
  triggers it; scale the threshold by actor variant so heavier variants
  demand real violence.

## A compound rigid fall about a pivot line

A structure falling over WITH its cargo (a loaded rack tipping backward,
shelf and all) is one rigid rotation about the ground edge it pivots on -
one angle array drives every part, so nothing can desync:

- Define the pivot line (the upstage edge of the feet) and express every
  moving part as an offset `(dy, dz)` from it. The fall transform is one
  closed form: leaning `theta` backward maps `(dy, dz)` to
  `(dy cos + dz sin, dz cos - dy sin)`. Rack frame, cargo, anything else:
  same transform, same angle samples, same clock.
- Integrate the give as an inverted pendulum (`omega += k * sin(theta) *
  dt`) and detect PER-PART contacts inside the loop: the cargo often
  lands first - a bar's rolling radius beats the height of a tipped cup
  well before the frame lies flat (~78 degrees, not 90). First contact is
  the thud; the frame clangs flat separately. Two impacts, two cues.
- After its contact, a part separates from the rigid transform and gets
  its own short tail (a dead pop, a decaying slide) appended to the same
  channel arrays - one bake, piecewise sources.
- The teeter that precedes the give must hand off C1. The trap: a grown
  sine with a `max(0, ...)` clamp pins the angle at zero through the
  sine's negative half-cycle - the structure freezes bolt upright for a
  beat, then the fall starts with a hardcoded angular velocity, a naked
  velocity jump at a non-contact seam. The clean shape is a grown
  raised-cosine swell that ENDS AT ITS PEAK, with the growth envelope flat
  across the final half-cycle - a still-growing envelope re-enters its own
  slope as handoff velocity. At the peak the swell's angular velocity is
  zero, gravity takes it from rest, and the seam is C1.
- Duration bookkeeping: N samples span N-1 intervals. `count * dt` plays
  the bake a fraction slow and every scheduled cue leads the presentation
  by up to a sample at the impact - `(count - 1) * dt`.
- Companion channels follow the cargo, not the script: the contact shadow
  stays at the cargo's height until ITS contact cue grounds it; snapping
  it dark at the fall's first frame puts a grounded shadow under a thing
  still up in the air.

## Soft bodies: squash from the contact list

A rubber actor sells its material through squash-and-stretch, and the
integrator already knows every impact. Add a squash channel at bake time:
each recorded contact contributes a half-sine envelope (~0.12 s) with
amplitude from impact speed (capped ~0.28); floor hits flatten y and bulge
x/z, wall hits flatten x and bulge y/z. Two rules make it read right:

- Scale about the CONTACT POINT, not the actor's center: park a dedicated
  squash node at floor level with the geometry offset upward inside it
  (travel > lift > squash(y = floor) > geometry(y = +radius)). Scaling
  that node presses the ball INTO the floor; scaling the center makes it
  hover while deforming.
- Sample the squash from the same contact list as the haptics, into the
  same clock as position - a separate timer drifts.

Spin on a bouncing body is angular-momentum STATE, never a slave of
current velocity. Slaving spin to vx means every wall bounce reverses the
rotation instantly mid-air - the single clearest tell of fake physics.
Integrate omega separately: grounded rolling locks it to -vx/radius,
airborne it stays constant through wall bounces, and each floor contact
re-grips it partway toward rolling (blend ~60% toward -vx/radius).

Two entrance traps from the bouncing-ball build:

- If an entrance SPAWNS the actor outside the walls, the first integrator
  step reads the spawn point as a wall collision and the actor ping-pongs
  itself dead at the boundary. But scope any arming grace to the entrance
  ONLY: the same grace in the shared throw integrator lets a ball RELEASED
  in the held-over-wall stretch zone fly off screen forever (walls never
  arm). For throws, keep walls always on, reflect only OUTWARD motion, and
  clamp position unconditionally - an inward release in the stretch zone
  then re-enters instead of being flung back out.
- A bounced arrival along a depth path is two channels on one clock:
  horizontal path fraction and a decaying vertical bounce train. Keep the
  horizontal near-linear (soft landing only); an eased ride spends the
  early time crawling, the whole bounce train dies while the actor is
  still tiny in the background, and the visible half of the arrival reads
  as floating on a string.

## Testability

Because each integrator is a pure function from initial conditions and a
profile to sample arrays, tests are ordinary unit tests: assert the actor
never exceeds the ceiling for any release velocity, assert termination under
the time cap across a parameter grid, assert the final resting pose, assert
contact counts are monotone in restitution. None of this is possible against
a live physics engine.
