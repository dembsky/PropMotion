# Shadows and Lights: The Silent Traps

## Contents

- Trap 1: deferred shadows die under MSAA
- Trap 2: intensity-0 lights are skipped entirely
- Trap 3: shadowBias is ignored for directional lights
- Trap 4: forward shadow is screen modulation, categoryBitMask included
- Trap 5: alpha textures cast square shadows
- The reliable combo
- Animating shadow visibility
- The neutral light budget
- Face-on metal is a black hole
- Debug recipe
- The renderer's own ceilings

Every trap here fails silently: no console output, no assertion, just a
missing or wrong shadow. Read the debug recipe before spending an iteration
on light angles.

## Trap 1: deferred shadows die under MSAA

`shadowMode = .deferred` never renders a shadow while the view has
`antialiasingMode` set to any multisampling value. Not dim, not offset:
absent, with zero errors. Since product stages want MSAA (geometry edges over
a transparent background), deferred is effectively unusable here. Use
`.forward`.

## Trap 2: intensity-0 lights are skipped entirely

A light with `intensity = 0` is not "invisible but still casting"; the
renderer skips it, shadow pass included. You cannot make an invisible shadow
caster by zeroing intensity. Instead aim a full-intensity directional light
from behind and above the actor: every camera-facing surface is back-facing
to it, so its Lambert contribution to the visible image is zero while the
shadow falls forward onto visible floor.

## Trap 3: shadowBias is ignored for directional lights

For directional lights, `shadowBias` does nothing. Sweeping it across two
orders of magnitude produces pixel-identical renders. If you are fighting
shadow acne or self-shadow lines on a directional light, bias is not the
lever; see the shadowColor ramp below.

## Trap 4: forward shadow is screen modulation, categoryBitMask included

Forward shadow mode does not evaluate lighting per fragment. It darkens every
fragment covered by the shadow map, with no N-dot-L gate and no respect for
the light's `categoryBitMask`. Consequences:

- Surfaces facing away from the shadow light still get darkened if the map
  covers them. An object lying flat inside its own shadow shows a hard
  penumbra line across itself in that pose (invisible while upright, because
  there the modulation is uniform).
- You cannot exclude the caster from its own shadow via categories; a proxy
  caster plus a category-masked light renders pixel-identical to the naive
  setup.

The pragmatic fix when a pose makes self-shadowing visible: fade
`shadowColor` to alpha 0 for the duration of that pose (hide the fade inside
the motion that enters the pose) and restore it on the way out. Keep a spread
contact blob under the actor so grounding never disappears.

## Trap 5: alpha textures cast square shadows

The shadow map ignores texture alpha. A transparent-cornered label plane or
decal casts the shadow of its full rectangle. Set `castsShadow = false` on
every alpha-textured plane and let the solid geometry cast the silhouette.

## The reliable combo

```swift
sun.castsShadow = true
sun.shadowMode = .forward
sun.shadowColor = UIColor.black.withAlphaComponent(0.28)
sun.shadowRadius = 14                       // softness, tune with map size
sun.shadowSampleCount = 48
sun.shadowMapSize = CGSize(width: 2048, height: 2048)
sun.automaticallyAdjustsShadowProjection = false   // auto-fit clips the pool
sun.orthographicScale = 5
sun.zNear = 1; sun.zFar = 30

catcherMaterial.lightingModel = .shadowOnly        // invisible except shadows
catcherMaterial.writesToDepthBuffer = false
```

Forward mode plus a `.shadowOnly` catcher works with MSAA and a transparent
background. Fix the shadow projection to a known box around the stage; the
automatic fit cuts the pool's near edge with a straight line once the camera
sees enough floor.

## Animating shadow visibility

Three rules, all learned the hard way:

1. Never toggle `castsShadow` at runtime. With any meaningful `shadowRadius`,
   a flat object casts a wide blurred penumbra from its first frame, so a
   binary switch pops a gray halo in and out in one frame. Keep `castsShadow`
   permanently on and drive visibility with `shadowColor` alpha, ramped in
   sync with the motion (for example, fade in over the first tenth of an
   entrance, out over the last fifth of an exit).
2. Do not animate light properties with `CAAnimation` through dotted
   keyPaths from the node (`"light.shadowColor"`). It can fail silently: no
   error, no effect. What works reliably:
   - `SCNTransaction` for immediate implicit fades:

     ```swift
     SCNTransaction.begin()
     SCNTransaction.animationDuration = 0.55
     light.shadowColor = UIColor.black.withAlphaComponent(0)
     SCNTransaction.commit()
     ```

   - `SCNAction.customAction` on the light's node for delayed or sequenced
     fades, setting the value per frame from the engine's clock:

     ```swift
     let fade = SCNAction.customAction(duration: duration) { node, elapsed in
         let t = min(1, Double(elapsed) / duration)
         node.light?.shadowColor = UIColor.black
             .withAlphaComponent(maxAlpha * (1 - t)).cgColor
     }
     lightNode.runAction(.sequence([.wait(duration: delay), fade]), forKey: "shadowOut")
     ```

3. Whatever mechanism animates the shadow must be swept by the central
   cleanup: actions with `removeAllActions`, implicit fades by resetting the
   value inside a `SCNTransaction` with `disableActions = true`. A stale fade
   pins the presentation exactly like a stale CAAnimation.

## The neutral light budget

When the scene must render a texture color-faithfully (for example a 3D copy
that hands off to a pixel-identical 2D view), the illumination on a flat
surface must be exactly neutral. SceneKit's neutral exposure is intensity
1000, so:

```
ambient + directional * cos(angle between light and surface normal) = 1000
```

A sum over 1000 clips light grays to white, irreversibly; a texture gray of
0.82 can render near 0.97 at an 18 percent overshoot, and the brightening is
plainly visible at any 2D-to-3D handoff. Keep whatever ratio of ambient to
directional you like, and scale the pair so the flat-surface sum is 1000.
Verification: count dark pixels of a known glyph before and after the
handoff; do not trust the eye.

## Face-on metal is a black hole

A mirror-finish PBR material viewed head-on reflects the environment zone
directly behind the camera. Studio-style environment maps are deliberately
dark there, so face-on chrome renders as a gray or black hole. Brightening
that zone globally trades one bug for another: the extra light blows out the
same surface in other poses and pollutes every other reflective material.

The production pattern, when a painted or approved base look must stay while
"the light moves":

- Keep the approved base material or artwork untouched.
- Overlay a wafer-thin duplicate surface (an annulus or shell a fraction of a
  millimeter proud of the base) with `lightingModel = .constant`, black
  diffuse, `blendMode = .add`, `writesToDepthBuffer = false`, and its OWN
  image in `material.reflective`, black everywhere except a few bright
  streaks.
- Position the streaks so the rest pose samples pure black: the overlay adds
  nothing at rest (the approved look alone), and a device tilt or actor
  rotation sweeps a streak across the surface, reading as moving light.

And a testing rule: never judge a metal material in a single pose. Face-on is
the degenerate case; check it tilted, edge-on, and in motion.

## Debug recipe

- A shadow is missing: FIRST give the scene a visible floor (a plain gray
  lambert plane) and look again. This one move distinguishes "the shadow does
  not render at all" (mode/MSAA/intensity trap) from "the shadow renders
  somewhere I cannot see" (aim/projection) and saves entire guessing
  iterations over directions and intensities.
- A rendering artifact resists reasoning: build an offscreen reproduction
  with `SCNRenderer`, render variants with elements toggled on and off, and
  pixel-diff the outputs. Visual inspection of full frames lies; a variant
  that "looks clean" can be pixel-identical to the broken one.
- One-frame glitches at SwiftUI boundaries: extract frames from a screen
  recording and diff consecutive frames numerically; the eye cannot localize
  single-frame anomalies.

## The renderer's own ceilings

Two limits worth knowing before a rig grows past the three-light recipe: a
node is lit by at most eight lights - the renderer drops the rest without a
word, and which eight win is not yours to choose - and every omni or spot
light should carry an `attenuationEndDistance`, because a light with no end
distance is evaluated for every node in the scene no matter how far it
sits. The standard rig here stays well under both, but a stage that gains
per-actor accent lights (a swarm, a multi-actor pile) hits the cap earlier
than intuition says.
