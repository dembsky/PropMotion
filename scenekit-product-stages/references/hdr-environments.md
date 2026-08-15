# HDR Environments

Believable metal on a stage. The painted 2:1 canvas from
[stage-recipe.md](stage-recipe.md) is fine for tinting a satin actor; the
moment the actor IS its reflections - chrome, gold, a liquid-metal blob -
the painted map betrays you, and the fix is a different pipeline, not a
better painting.

## Contents

- Cartoon metal: the LDR ceiling
- Real HDRIs, and the UIImage trap
- The camera's HDR pipeline
- Rotating an environment that cannot rotate
- Finish notes: clearcoat, pearl, black metal
- A light theme needs dark furniture
- Sourcing and bundling

## Cartoon metal: the LDR ceiling

The symptom set, visible from across the room:

- The main highlight is one flat paper-white blob with a hard edge, as if
  cut from vinyl.
- Reflections posterize into a handful of gray bands instead of gradients.
- Every finish looks like the same plastic with a different tint.

The cause is the environment's dynamic range, not the material. An image
painted through CoreGraphics tops out at 1.0, so a softbox and the wall
glow around it hold the SAME white - there is nothing for tone mapping to
roll off and nothing for roughness to spread. A real studio has lamps ten
to a hundred times brighter than its walls; that ratio is what draws the
hot core, the falloff, and the gradient detail a mirror surface lives on.
No amount of painting inside 0..1 recovers it.

## Real HDRIs, and the UIImage trap

Use a photographic `.hdr` (Radiance) panorama and hand SceneKit the FILE
URL:

```swift
scene.lightingEnvironment.contents =
    Bundle.main.url(forResource: "studio_small_08_1k", withExtension: "hdr")
```

SceneKit reads Radiance files directly and keeps the full range. The trap
is helpfulness: loading the file into a `UIImage` first (or any re-encode
through an 8-bit context) clamps it straight back to LDR with no error,
and the cartoon returns while the code looks upgraded. URL in, nothing in
between.

The 2:1 spherical layout rule from the stage recipe still applies - a
square image is silently ignored. A 1k panorama (1024x512, ~1.5 MB) is
plenty for reflections on a phone actor; reserve 2k+ for scenes where the
environment itself is visible as a background.

## The camera's HDR pipeline

The environment provides the range; the camera must render it:

```swift
camera.wantsHDR = true
camera.wantsExposureAdaptation = false
camera.bloomThreshold = 1.15
camera.bloomIntensity = 0.22
camera.bloomBlurRadius = 8
```

- `wantsHDR` turns on tone mapping: highlights roll off smoothly instead
  of clipping at the display's white.
- Exposure adaptation OFF is deliberate: an exposure that drifts with
  scene content renders a different frame for the same clock value, which
  breaks the frozen-clock snapshot contract
  ([rendering-contract.md](rendering-contract.md)). Fix exposure and own
  the picture.
- Bloom's threshold sits ABOVE 1.0 so only true HDR sources glow - the
  lamps, never the whole bright side of the actor. Oversized bloom reads
  as a white fringe around the entire silhouette; start subtle and grow
  only with evidence.

## Rotating an environment that cannot rotate

`lightingEnvironment` has no orientation control, and where the lamps land
in the reflection is composition - a frontal softbox prints one big blob
dead ahead, while the same lamp upper-left reads as studio window light.
The lever that does exist: rotate the CAMERA RIG instead.

```swift
let envRig = SCNNode()
envRig.eulerAngles.y = 2.1        // tune against screenshots
envRig.addChildNode(cameraNode)
scene.rootNode.addChildNode(envRig)
```

With a rotationally symmetric actor, a radial contact shadow, and an
`SCNFloor`, the framing is pixel-identical - only the reflection layout
moves. Anything screen-space-derived must be built rig-aware from the
start: a trackball gesture that lifts its axes through the camera's world
orientation ([gestures-and-haptics.md](gestures-and-haptics.md)) absorbs
the rig rotation for free, one that hardcodes world axes fights it.

## Finish notes: clearcoat, pearl, black metal

- A full `clearCoat` is a MIRROR LAYER. On a white dielectric it turns
  pearl into chrome with a white core - the two-material family collapses
  into one look. Pearl wants `clearCoat` around 0.3-0.4 with roughness
  around 0.2: gloss over cream, not silver.
- Black metal (obsidian, gunmetal) is drawn entirely by the environment's
  midtones. Against a detail-free dark map it renders as a black hole
  with pinprick glints; against a real studio it reads as material. If a
  dark finish looks broken, suspect the map before the material.
- Chrome tolerates more roughness than instinct suggests (0.08-0.12).
  With a real HDRI behind it, roughness spreads gradient detail; with an
  LDR map it just grays the vinyl blob out.

## A light theme needs dark furniture

A white actor in an all-white world vanishes: its shape lives in the dark
things it reflects. A bright studio HDRI works because the photograph
carries a dark floor, stands, and ceiling contrast for free. When swapping
environments per app theme, pick the light-theme panorama for its dark
content, not despite it - a bright infinity cove WITH visible dark floor
beats a clean white gradient every time.

Theme swap is a straight contents-and-intensity change (the URL swap is a
hard cut; let the SwiftUI backdrop animate the transition around it):

```swift
scene.lightingEnvironment.contents = environments[theme]
scene.lightingEnvironment.intensity = theme == .dark ? 1.0 : 0.9
```

## Sourcing and bundling

- Poly Haven publishes photographic studio HDRIs under CC0 - no license
  text to carry, safe for a shipped app. The `studio_small_*` series is
  the product-shot workhorse: `08` is softboxes over a dark cove (dark
  themes), `09` a bright infinity cove (light themes).
- Bundle the `.hdr` files as plain resources. With XcodeGen, any
  non-source file inside a sources folder lands in the resources build
  phase; verify once with
  `ls Build/Products/.../App.app | grep hdr` - a missing resource fails
  soft (nil URL, black reflections), not loud.
