# Fog and mist sheets

A continuous, directed stream of vapor - air out of a vent, steam off a
cup, mist under a waterfall - is not a particle job. Particles excel at
impacts and bursts; a steady stream built from them reads as dots, costs a
warm-up, and fights art direction. The production answer is ONE translucent
sheet of geometry with the whole picture painted by a shader pair: a
geometry modifier waves the mesh, and a fragment modifier paints a cloudy
body and every fade. The sheet is the stream.

This reference is the recipe, and the traps - several of which only appear
after the effect looks finished.

## The ribbon mesh

Author the sheet as a procedural ribbon: rows of quad pairs marched along
an arc. Three numbers shape a believable stream:

- a launch angle out of vertical (`tilt`) - steep launches foreshorten to
  nothing from a low camera, so keep it gentle and let the arc do the work;
- how much of that lean the stream sheds over its length (`curve`) - the
  waterfall arc every reference clip shows; a flat tilted plane reads as a
  curtain;
- a lateral drift (`sweep`) toward one end, because depth motion
  foreshortens to nothing and the on-screen diagonal is what actually
  draws the direction.

```swift
var point = SIMD3<Float>(0, 0, 0)
for row in 0...rows {
    let s = Float(row) / Float(rows)
    let angle = tilt + (endAngle - tilt) * s
    let direction = simd_normalize(
        SIMD3<Float>(-sweep * s, -cos(angle), sin(angle))
    )
    // Every fade that reads the viewing angle needs real normals;
    // a hand-built descriptor has none unless they are spelled out.
    let normal = simd_normalize(simd_cross(SIMD3<Float>(1, 0, 0), direction))
    // two vertices per row, uv.y carrying the flow coordinate
    point += direction * step
}
```

Give the UV's y-axis the flow coordinate (`flow = 1 - uv.y`, zero at the
vent): both shader stages will derive everything from it. Row density is
set by the SHORTEST ripple you will ever add, not the main wave: normals
tilted per-vertex alias into faint straight bands below roughly ten rows
per ripple cycle. A fast flutter at 7-8 cycles along the stream wants 80
rows, not 40.

## The fog body

The fragment stage paints the fog. The rules that separate mist from a
glowing decal:

- **Cloud, never a flat wash.** Two-tap value noise, three octaves, and a
  light domain warp - the warp bends the noise field itself, which is what
  separates cloudy mist from wobbly stripes. Scroll the field along flow
  at the pace real air would have; the scroll is what makes the fog READ
  as flowing.
- **Quintic softstep, not smoothstep.** Across a wide bright fog,
  smoothstep's curvature kink at its endpoints prints a visible Mach-band
  contour. `f*f*f*(f*(f*6-15)+10)` has zero curvature at both ends and
  leaves no onset line.
- **Jitter every fade edge.** Real mist has no straight boundaries: every
  envelope (side feathers, head ramp, tail dissolve, frame fade) gets its
  width or input bent by the noise field. A straight iso-contour prints
  its own soft edge. Pin the feather's ZERO point to the physical mesh
  border so visible fog never meets a mesh edge.
- **Dial-order-proof envelopes.** Derive the tail band from BOTH dials
  with `min()` so any slider order works and the fade always spans a
  minimum fraction of the stream; a degenerate band is a cliff and a dead
  slider.
- **Brighter downstream.** The eye reads the bright, thick end of a streak
  as its head. If the envelope is brightest at the source, the stream
  reads as SUCKING. A gentle gain toward the tail keeps it blowing.
- **Fold lighting.** Ride a sheen band on the SAME phase as the geometry
  wave, so bright bands sit on the geometric crests and travel with them -
  the visible proof of motion that survives a still frame.
- **Animated grain.** Per-pixel hash jittered every frame dissolves
  whatever soft banding survives - including plain 8-bit gradient banding
  on OLED - into shimmer the eye reads as moving air.

## Motion that survives a speed dial

The trap that ships: writing the phase as `waves * flow - speed * t` with
the scene's absolute clock. It looks correct until the speed CHANGES.

The phase's time derivative is `speed + speed' * t`, and `t` is the
scene's uptime - tens of seconds. Snap the dial and the pattern teleports.
Tween the dial (the obvious "fix") and the teleport becomes a visible
glide: for the whole transition the stream races forward, or - when
slowing down - flows BACKWARD into the machine. A user WILL flip the fan
from high to low and watch the air get sucked back in.

Phase must be the INTEGRAL of speed, not the product with absolute time.
Accumulate a stream clock once per rendered frame and hand it to the
shader as one uniform:

```swift
final class StreamClock: NSObject, SCNSceneRendererDelegate {
    weak var material: SCNMaterial?
    private var target: Float = 0, current: Float = 0
    private var clock: Float = 0
    private var lastTime: TimeInterval?
    private let lock = NSLock()

    func setTargetSpeed(_ speed: Float) { lock.lock(); target = speed; lock.unlock() }

    func renderer(_ renderer: SCNSceneRenderer, updateAtTime time: TimeInterval) {
        lock.lock()
        let dt = Float(min(time - (lastTime ?? time), 0.1))
        lastTime = time
        current += (target - current) * min(1, dt / 0.35)   // fan spool
        clock += current * dt
        let value = clock; let material = material
        lock.unlock()
        material?.setValue(value, forKey: "waveClock")
    }
}
```

The clock can only advance, so backward flow is impossible by
construction, and the eased speed makes every dial change read as the
machine spooling up or down. Route EVERY speed-coupled term through the
same clock - the wave phase, any secondary ripple, the fog advection -
with their multipliers applied to the clock, not to time. Terms that do
not follow the fan (ambient noise drift, sweep timers, grain) stay on
`scn_frame.time`.

## Folds: normals and thickness

Two facts collide the moment the sheet waves visibly:

1. **A geometry modifier moves vertices, not normals.** Any fade that
   reads the viewing angle - a grazing dissolve, a fresnel - sees the flat
   sheet's static normals and is blind to the folds the wave creates.
   Tilt the normal analytically in the geometry stage: differentiate the
   sway along flow (the dominant `cos(phase) * 2π * waves` term is
   enough), and rotate the normal toward the flow tangent by that slope.

2. **A fold silhouette is an overdraw accumulator.** Where the waving
   sheet turns edge-on, the view ray crosses many layers of it: projected
   density grows like `1/facing`, and the stack prints a bright hairline
   no blur or feather softens. Two banded attempts fail in
   characteristic ways - a `softstep` with a floor leaves
   `floor x stack` and the razor survives; widening the band instead
   paints straight DARK stripes along every constant-phase line of the
   wave. The clean answer is thickness compensation:

   ```metal
   float knee = 0.10 + 0.08 * wisp;   // jitter the knee like every edge
   float grazing = (1.0 + knee) * facing / (facing + knee);
   ```

   Scaling alpha by ~`facing` cancels the `1/facing` stacking exactly: it
   is smooth (no band edges to print), reaches zero at true tangency (no
   floor, no hairline), and leaves face-on fog untouched.

Even compensated, keep the wave inside the envelope it was tuned for: the
amplitude that looked right over a short visible run folds the sheet
edge-on when the dissolve moves outward and more of the stream survives.
Calm the wave along the tail - and give that calm its OWN ramp. A lateral
swing bend living in the same ramp variable dies silently, because its
whole reach sits exactly where the calm bottoms out. One motion channel,
one envelope.

## Dial envelopes are camera-relative

A tuned dial set encodes the camera it was tuned against. Move the effect
under a new camera (or extend its visible run) and the numbers stop
meaning what they meant: a dissolve that ended mid-stream was also hiding
the zone where the wave folds; a sweep reach compensated against a short
visible end lands invisibly inside a longer one. When porting a tuned
effect, first re-block the composition in screen space for the NEW camera
- direction, reach, visible run - then re-derive every envelope-coupled
constant from the new visible end, and only then compare details against
the reference.

The flat-sheet trick also buys exactly one camera. Seen near edge-on the
stream honestly dissolves (the compensation above), which means an orbit
control will find angles where the air disappears. That is the
construction's real limit: a full orbit needs a second sheet crossed at
ninety degrees, billboard-style, not more dials.

## Wiring the dials

In SceneKit every dial is a `#pragma arguments` uniform fed by KVC - and
both stages read the same keys, so nothing needs the parameter-packing
contortions engines with sealed material slots force. Two habits pay off:

- The KVC key path is an animation channel. A swing's reach ramp or a
  one-shot pressure pulse is one `SCNTransaction` (or a baked
  `CAKeyframeAnimation`) on the uniform - no per-frame CPU loop, and the
  sweep itself can ride the shader clock so the CPU is idle once the ramp
  settles.
- Match the transaction to the dial's role. Power fades take the long
  eased transaction; a slider the user drags must track the finger -
  apply it in its own short (~0.15 s) transaction, or the stage feels
  mushy and WYSIWYG dies.

Push colors through the linear-space conversion before they cross the KVC
bridge (the trap and the helper live in
[stage-recipe.md](stage-recipe.md), shader modifiers section).

## Keeping it alive, and proving it

A stage whose only motion is the shader clock never schedules a second
frame: SceneKit's on-demand render loop cannot see shader time, the sheet
draws once and freezes. Set `rendersContinuously = true` on the view.

Verify like an instrument, not a viewer:

- **Frozen scene:** pixel-diff two screenshots a few seconds apart; a
  `None` bounding box is a stopped renderer, whatever your eyes say.
- **Direction and pace claims:** sample a brightness profile along the
  stream axis, subtract the temporal mean per position (the static
  envelope otherwise anchors the correlation at zero), and correlate
  consecutive frames. The per-frame shift IS the flow: it should scale
  with the fan dial, spool smoothly through changes, and never go
  negative. This one measurement settles "it flows backward" disputes
  that stills cannot.
- **Fold artifacts:** brighten the captures 2x before judging edges; a
  hairline invisible at 1x on your monitor is plainly visible on an OLED
  in a dark room.
