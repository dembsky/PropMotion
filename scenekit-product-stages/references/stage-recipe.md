# The Stage Recipe

## Contents

- The transparent stage
- UIViewRepresentable and the coordinator
- State-key diffing in updateUIView
- Node hierarchy: one node per motion concern
- Camera
- The lighting rig
- Contact blob and shadow catcher
- Programmatic environment maps
- Programmatic textures
- Impact particles on a stage
- Shader modifiers on stage actors
- The stage must keep its identity

## The transparent stage

The whole idea: the 3D actor performs on whatever screen hosts it. No skybox,
no colored background, no visible rectangle. The host SwiftUI layout provides
the backdrop and the SCNView floats over it.

```swift
let view = SCNView()
view.backgroundColor = .clear
view.isOpaque = false
view.antialiasingMode = .multisampling4X
// scene.background.contents = UIColor.clear is set inside buildScene():
// view.scene is still nil at this point, so setting it through
// view.scene?.background here would silently no-op.
```

Size the stage larger than the visual slot when motion needs room: an actor
that rises toward the camera or sweeps diagonally will clip against a
tight view frame with a hard line. Give margins on every side the motion can
reach, and align the interior content with the host layout mathematically
(compute the view frame and center from the slot size, not by eye).

## UIViewRepresentable and the coordinator

The coordinator is the scene's owner and the only object that mutates it. Mark
it `@MainActor`; SceneKit's scene graph must be mutated on the main thread.

```swift
struct ProductStage: UIViewRepresentable {
    let item: Item

    func makeCoordinator() -> Coordinator { Coordinator() }

    @MainActor
    final class Coordinator {
        let cameraNode = SCNNode()
        private let travelNode = SCNNode()  // position along paths
        private let yawNode = SCNNode()     // heading
        private let spinNode = SCNNode()    // roll around the actor's axis
        private(set) var stateKey = ""

        func markState(_ key: String) { stateKey = key }

        func buildScene() -> SCNScene {
            let scene = SCNScene()
            scene.background.contents = UIColor.clear
            travelNode.addChildNode(yawNode)
            yawNode.addChildNode(spinNode)
            scene.rootNode.addChildNode(travelNode)
            // camera, lights, floor pieces added here
            return scene
        }
    }
}
```

Build the scene exactly once, in `makeUIView`. Rebuilding in `updateUIView`
throws away compiled Metal state and replays everything.

## State-key diffing in updateUIView

SwiftUI calls `updateUIView` for reasons that have nothing to do with your
scene: a parent re-render, an environment change, a size change. If the update
path unconditionally reinstalls the actor or replays the entrance, the stage
flickers and restarts constantly.

Derive a state key from exactly the inputs that should visibly change the
scene, and diff it:

```swift
private var stateKey: String {
    "\(item.id)-\(unit.rawValue)-\(isLocked)"
}

func updateUIView(_ view: SCNView, context: Context) {
    guard context.coordinator.stateKey != stateKey else { return }
    context.coordinator.markState(stateKey)
    context.coordinator.transition(to: item)
}
```

Include a value in the key only if the scene must react to it. Example: if a
locked variant renders differently in light and dark mode but the normal
variant does not, include the color scheme in the key only when locked, so an
appearance switch never replays the normal actor's entrance.

## Node hierarchy: one node per motion concern

Never pile position, heading, roll, and wobble onto one node. Give each
degree of freedom its own node in a fixed parent chain:

```
root
  tiltNode      (optional gyro parallax for the whole world)
    travelNode  (position along entrance paths, drag x)
      liftNode  (height above the floor only)
        pitchNode (fall-over rotation only)
          yawNode  (heading while rolling, resting yaw)
            spinNode (the roll itself, around the actor's axis)
              actor geometry
```

Payoffs: independent CAKeyframeAnimations attach to different nodes and never
fight over a keyPath; a gesture can freeze one concern (spin) while another
keeps animating (lift); the floor-bound contact blob can ride `travelNode`
position without inheriting rotations.

## Camera

- `camera.projectionDirection = .horizontal` with a horizontal field of view:
  the actor's rendered size then follows the stage's width, so the host can
  add vertical room (for shadows or motion) without rescaling the hero.
- Standing-eye framing: raise the camera above the floor and pitch it slightly
  down (for a floor at y = -1 and an actor around the origin, a camera near
  y = 1.5, z = 5 with about -14 degrees pitch works). Far floor points then
  project higher and smaller, which is how a person sees an object on the
  ground.
- Keep the camera static. A tripod reads as real; a dolly reads as synthetic,
  especially composited over a flat UI.

## The lighting rig

Three visible lights sculpt the actor; a fourth invisible one casts the
shadow; ambient is only a floor value.

```swift
// Warm key from upper front-left: form and color.
key.type = .directional; key.intensity = 900
key.color = warmNearNeutral   // close to neutral so textures keep their hue

// Cool whisper of fill from the opposite side.
fill.type = .directional; fill.intensity = 160

// Hard rim from upper-rear: the edge that separates a dark actor
// from a dark backdrop.
rim.type = .directional; rim.intensity = 450

// Ambient floor value only. High ambient is a fog machine.
ambient.type = .ambient; ambient.intensity = 140
```

The shadow sun is a separate directional light aimed from behind and above the
actor, tilted toward the camera. Because every camera-facing surface is
back-facing to it, its full intensity contributes nothing visible to the
sculpt (Lambert term is zero), while the shadow falls forward onto visible
floor. Do not try intensity 0: the renderer skips zero-intensity lights
entirely, shadow included.

Pin the shadow projection explicitly. The automatic fit can clip the shadow
pool with a straight edge once the camera sees enough floor:

```swift
sun.castsShadow = true
sun.shadowMode = .forward           // deferred never renders under MSAA
sun.shadowColor = UIColor.black.withAlphaComponent(0.28)
sun.shadowRadius = 14
sun.shadowSampleCount = 48
sun.shadowMapSize = CGSize(width: 2048, height: 2048)
sun.automaticallyAdjustsShadowProjection = false
sun.orthographicScale = 5
sun.zNear = 1
sun.zFar = 30
// Aim by vector, then back the node off along its own axis so the
// zNear...zFar range brackets the stage.
let dir = simd_normalize(simd_float3(-0.15, -2.2, 1.0))
sunNode.simdPosition = -10 * dir
sunNode.simdLook(at: sunNode.simdPosition + dir)
```

See [shadows-and-lights.md](shadows-and-lights.md) for every trap in this area.

## Contact blob and shadow catcher

Two pieces make an object believably touch an invisible floor:

1. The contact blob: a small plane with a radial dark gradient texture, lying
   just above the floor, `lightingModel = .constant`, `writesToDepthBuffer =
   false`, `renderingOrder = -1`, `castsShadow = false`. It fakes the ambient
   occlusion pinch a directional light cannot produce, the dark line that
   tells the eye rubber meets ground. Parent it to the travel node so it
   follows position but never rotation, and deliberately outside the lift
   node: when the actor jumps, the blob stays down, thinning and fading with
   the gap, which is what tells the eye the actor is airborne.
2. The shadow catcher: a large invisible plane at floor height whose material
   has `lightingModel = .shadowOnly` and `writesToDepthBuffer = false`. It
   renders nothing but received shadows (the AR trick), so the real cast
   shadow appears on the transparent stage.

A middle gradient stop matters for the blob: a linear dark-to-clear ramp reads
as fog; holding near-full darkness through the middle and then falling off is
what reads as contact.

## Programmatic environment maps

Chrome and other reflective PBR materials need something to reflect. On a
transparent stage there is no skybox, so paint one:

```swift
scene.lightingEnvironment.contents = environmentMap()
scene.lightingEnvironment.intensity = 1.45

func environmentMap() -> UIImage {
    // 2:1 aspect = spherical projection, one of the cube-map layouts
    // SceneKit recognizes. Pin scale to 1: texture pixels do not care
    // about screen scale (see Programmatic textures below).
    let format = UIGraphicsImageRendererFormat()
    format.scale = 1
    let size = CGSize(width: 1024, height: 512)
    return UIGraphicsImageRenderer(size: size, format: format).image { ctx in
        // Deep dark base gradient.
        // Two vertical white "softbox" streaks at distinct azimuths.
        // A faint floor-bounce band along the bottom edge.
        // Optionally a warm ceiling strip.
    }
}
```

The layout trap: `lightingEnvironment` (and `background`) accept only the
cube-map layouts SceneKit recognizes - a 2:1 spherical projection, 6:1 or
1:6 strips, or an array of six images. A SQUARE image assigned there is
silently ignored: no error, no warning, and every reflective material
renders as if the environment were pure black. (`material.reflective` on a
single material is different - it also accepts a square sphere map, which
is what the additive-sheen overlay in
[shadows-and-lights.md](shadows-and-lights.md) uses.)

Design rule: keep the zone the camera faces (what a head-on mirror reflects)
dark. A bright zone behind the camera blows out face-on metal; a dark one
keeps reflections as moving streaks that only appear when the object tilts.
This is also the basis of the additive-sheen fix for face-on metal, covered in
[shadows-and-lights.md](shadows-and-lights.md).

## Programmatic textures

`UIGraphicsImageRenderer` covers most product-stage texture needs without any
asset files: label faces (dark disc, ring, centered bold text), tiny tiled
crosshatch patterns for knurled grips (tile a 32 px image with
`wrapS/wrapT = .repeat` and a `contentsTransform` scale), radial gradients for
contact blobs.

Two production rules:

- Set the renderer format's `scale` to 1 and size in pixels explicitly. The
  default screen-scale renderer triples bitmap dimensions and can blow past
  the GPU's maximum texture size (16384 px on any iPhone that runs a modern
  iOS; 8192 px on older GPUs), which aborts the process. Even inside the
  limit the cost is 9x the pixels per texture (3x per axis on modern
  iPhones) - a 1024 pt canvas silently uploads as 3072x3072, ~38 MB of RGBA.
  This bites hardest on LIVE-RETARGETED textures (a face re-rendered per
  keystroke of a name being engraved): every render and GPU upload pays the
  9x. Texture pixels are sampled in 3D; screen scale means nothing to them.
  Size the canvas to the actor's on-screen projection instead.
- Cache generated textures in an `NSCache` with a real `totalCostLimit`
  (count the bytes of value and key if the key is a source bitmap), and
  release the cache when the stage leaves for good. Texture generation is
  thread-safe with these APIs, so prewarming can and should run off-main.

One more for live retargets: coalesce. When a representable diffs several
inputs that all feed one texture (name and unit on the same engraved face),
set a dirty flag per input and render ONCE per update pass - two inputs
changing in the same pass must not pay the render twice.

## Impact particles on a stage

A dust or debris burst under the actor (a slam, a landing) has its own trap
list, learned the visible way:

- Dust is MATTE: `blendMode = .alpha` with `sortingMode = .distance`.
  Additive blending turns dust into fire.
- Keep particles UNLIT and pick a flat color per light rig (dark smoke on a
  dark stage, pale chalk on a light one). Lit particles sample the actor's
  cast shadow and render as a dirty smudge hugging its silhouette.
- Emitter volumes must not intersect the actor. A sphere emitter at the
  contact point births particles IN FRONT of the actor's face, which read
  as a bright blob painted on the actor itself. Flat shapes at floor level
  only; a thin cylinder's `.surface` + `.surfaceNormal` birth makes a
  radial ground burst for free.
- The opposite trap for a FACE-ON actor: an emitter tucked safely behind it
  sits inside the actor's screen silhouette, and short-lived particles die
  before they clear it - the effect runs and nothing is ever visible. Split
  the effect into mirrored emitters at the silhouette's edges with outward
  drift (tire smoke: two plumes beside the contact patch) so the clouds
  billow into open screen space.
- Realism is scale and death, not volume: particles a fraction of the
  actor's size, sub-second lives, size growing while opacity dies (via
  `propertyControllers`), strong `dampingFactor` for air drag. One layer of
  big, long-lived softies reads as fog. Two layers - a soft floor ring plus
  a handful of small ballistic grit specks pulled down by a hard
  acceleration - read as dust.
- For a sustained plume (tire smoke, steam), density must come from COUNT,
  not per-particle opacity: many small dim puffs (opacity ~0.2) overlapping
  read as a boiling mass; a few large bright sprites read as cotton balls
  glued to the actor. Pair with a CLUSTER-of-blobs sprite (several offset
  radial gradients in one texture) - a single radial gradient reads as a
  perfect glowing ball at any size - plus brightness-only
  `particleColorVariation` and fast rotation to break up the surface.
- Particle pipelines compile at their own first draw, in the effect's first
  frame; warm them ahead of time (see
  [first-frame-and-warmup.md](first-frame-and-warmup.md)).

## Shader modifiers on stage actors

When an actor's surface must move or breathe per-vertex (a jelly wobble, a
ripple, an engraving that lights up), swapping geometries is the wrong tool.
A shader modifier is a short piece of Metal injected into the material's own
pipeline - the stage keeps its one scene graph and the deformation rides the
GPU. The traps are all in how values get in:

- A modifier is attached as a plain string on the material or geometry; it
  declares its dials with `#pragma arguments` and receives values from Swift
  through `setValue(_:forKey:)`. That KVC bridge is also what makes the dial
  a CHANNEL: the key path is animatable, so a baked `CAKeyframeAnimation` can
  drive a uniform on the same clock as the actor's transforms (see
  [baked-animation.md](baked-animation.md), channels beyond transforms).
- The silent color trap: values pushed through the KVC bridge arrive in
  LINEAR space, while every `UIColor` you own is sRGB. `diffuse.contents`
  converts for you; a uniform does not. Push 0.13 raw and the charcoal
  renders washed-out light gray, because 0.13 sRGB is ~0.015 linear - an
  order of magnitude off. Linearize first
  (`c <= 0.04045 ? c / 12.92 : pow((c + 0.055) / 1.055, 2.4)`), and suspect
  this trap whenever a material suddenly brightens after moving a color from
  `contents` to a uniform.
- Like every other pipeline on this stage, a modifier compiles at its first
  draw. A material that gains its modifier mid-performance hitches exactly
  like a cold particle system; warm it with the rest of the scene
  ([first-frame-and-warmup.md](first-frame-and-warmup.md)).
- The documentation's snippets are GLSL; on iOS the modifier compiles as
  Metal, where the GLSL symbol names do not exist. Time is
  `scn_frame.time`, frame-level transforms live on `scn_frame`
  (`projectionTransform`, `viewTransform`, ...), per-node transforms on
  `scn_node`. Write `u_time` and the whole material silently falls back
  to magenta.
- Helper functions go BEFORE `#pragma arguments`. Everything between that
  pragma and `#pragma body` is parsed as argument declarations; a function
  placed there is shredded into garbage argument types
  (`C3DBaseTypeFromMetalString: unknown type name 'return'` warnings) and
  the compile fails.
- A magenta actor IS a modifier compile error. The full Metal error, with
  the generated source, prints to the app's log - on the Simulator,
  `xcrun simctl spawn <udid> log show --predicate 'process == "YourApp"'`
  and search for `FATAL ERROR : failed compiling shader`.
- A scene whose only motion lives in the shader clock never schedules a
  second frame: the on-demand render loop cannot see shader time, so the
  effect draws once and freezes. Set `rendersContinuously = true` on the
  view, and prove motion with a pixel-diff of two screenshots, not by
  eye. For streams and flows driven by a speed dial, see
  [fog-and-mist-sheets.md](fog-and-mist-sheets.md) - the phase-clock trap
  there bites any speed-varying shader pattern, not just fog.

## The stage must keep its identity

The stage is a pile of expensive state SwiftUI cannot see: warmed Metal
pipelines, generated textures, a scene graph, generation tokens - all of it
lives behind `makeUIView`. SwiftUI destroys and rebuilds views by IDENTITY,
not by appearance, and any identity change tears that pile down and rebuilds
a cold stage mid-session: the first-frame hitch returns in front of the
user, pending cues die with their tokens, every texture renders again.

The identity killers, and what they cost a 3D stage:

- `if showStage { Stage() }` destroys the stage on every hide. Hide with
  opacity instead; an invisible stage stays warm and re-enters instantly.
- `.id(value)` remounts on every change of `value`. As a DELIBERATE lever it
  is legitimate - a replay button that re-mounts the stage to re-run the
  first entrance is exactly this - but wired to anything that changes on its
  own it is a scheduled demolition.
- A branch per size class (`if compact { PhoneStage() } else { PadStage() }`)
  parks two separate state trees: rotating the device or resizing in Stage
  Manager crosses the boundary and silently rebuilds the scene.
- Type-erasing conditional wrappers around the stage change the view's type
  between renders, which is an identity change with extra steps.

Separately from identity, there are update storms. The representable STRUCT
is recreated on every parent redraw - only the SCNView persists - and every
recreation runs `updateUIView`. The state-key diff absorbs that, but two
things defeat it:

- A closure parameter (`onGrab: () -> Void`) is incomparable, so SwiftUI can
  never skip the view. The stage takes VALUE triggers (ints, keys, flags)
  and reports back through its coordinator; it never takes closures.
- Every `@Environment` key the stage reads subscribes it to that key's
  traffic. Read as few as possible.

And because the struct is rebuilt constantly, its init must be empty:
geometry, materials, and textures belong to the coordinator, built once
behind `makeUIView`.

Two layout notes with 3D-specific consequences: `ignoresSafeArea` goes on
the stage's container, not on a backdrop sibling - a silently inset stage
shifts the whole camera framing relative to the design comp with no error
anywhere. And when the camera's aspect or frustum must follow a measured
container, read the measurement with `onGeometryChange` (delivered after
layout); a GeometryReader measures during layout, and a stage whose size
depends on its own measurement is a layout loop waiting to spin.
