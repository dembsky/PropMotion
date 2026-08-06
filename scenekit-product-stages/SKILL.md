---
name: scenekit-product-stages
description: Build and debug SceneKit scenes where one 3D object (a product, badge, coin, wheel) performs on a transparent stage inside a SwiftUI app - studio lighting, real shadows, baked keyframe choreography, hand-rolled physics, gestures and haptics, multi-scene sequencing. Use when working with SCNView or SceneView in SwiftUI, UIViewRepresentable 3D scenes, product or hero-object animation, roll or spin entrances, a first-frame hitch when a scene appears, shadows missing or wrong, metal rendering black, or choreographed 3D motion that must stay interruptible. Not for RealityKit, ARKit, visionOS, or full game worlds.
license: MIT
metadata:
  author: Mateusz Dembek
  version: 1.6.2
---

# PropMotion

Make one 3D object perform in your SwiftUI app. Production patterns for a
specific, common job: a polished product actor on a transparent SceneKit
stage - entrances, exits, throws, shadows, haptics, and the silent traps
that cost days.

## When to use SceneKit at all

Be honest about the framework's position:

- SceneKit is in maintenance mode. For new apps with heavy 3D needs (asset
  pipelines, USD/USDZ, AR, large worlds) prefer RealityKit.
- SceneKit is still the fastest path to a decorative 3D actor inside a SwiftUI
  app: a transparent `SCNView` composites over any SwiftUI layout, geometry
  shader modifiers are a single MSL string, CoreAnimation interop is mature,
  and everything here runs on plain UIKit views with no session setup.
- If the 3D element is one hero object with choreographed motion, this skill's
  recipes apply directly. If it is a full interactive world, stop and consider
  RealityKit first.

## The core stage recipe

A stage is: transparent `SCNView`, a `@MainActor` coordinator that owns the
scene graph, a camera at standing eye height, a three-light rig plus a
dedicated shadow light, a contact blob, and a shadow catcher. The actor
performs via baked keyframe animations.

```swift
struct ProductStage: UIViewRepresentable {
    let item: Item

    // The key must change ONLY when the scene must visibly change.
    private var stateKey: String { "\(item.id)" }

    func makeCoordinator() -> Coordinator { Coordinator() }

    func makeUIView(context: Context) -> SCNView {
        let view = SCNView()
        view.backgroundColor = .clear   // the stage composites over SwiftUI
        view.isOpaque = false
        view.antialiasingMode = .multisampling4X
        view.scene = context.coordinator.buildScene()
        view.pointOfView = context.coordinator.cameraNode
        context.coordinator.install(item)
        context.coordinator.markState(stateKey)
        // First-frame warm-up, two-key ignition: compile shaders off
        // the critical path, park the actor offstage, and gate the first
        // entrance on BOTH a minimum delay and prepare's completion.
        context.coordinator.parkOffstage()
        if let scene = view.scene {
            view.prepare([scene]) { _ in
                DispatchQueue.main.async { context.coordinator.markPipelinesWarm() }
            }
        }
        context.coordinator.scheduleFirstEntrance()
        return view
    }

    func updateUIView(_ view: SCNView, context: Context) {
        // State-key diffing: SwiftUI re-renders must never replay entrances.
        guard context.coordinator.stateKey != stateKey else { return }
        context.coordinator.markState(stateKey)
        context.coordinator.transition(to: item)
    }
}
```

The pieces, in build order:

1. Transparent view flags (`backgroundColor = .clear`, `isOpaque = false`),
   MSAA 4x. The host screen provides the backdrop; the scene has no box.
2. Coordinator owns the node hierarchy, decomposed one node per motion
   concern (travel, lift, yaw, spin), so independent animations never fight
   over a single transform.
3. Camera: `projectionDirection = .horizontal` so the actor's size follows
   stage width; raised position with a slight downward pitch reads like a
   standing observer. Keep it static; a moving camera reads synthetic.
4. Lights: warm key, cool low fill, hard rim, ambient only as a floor value,
   plus a separate shadow-casting directional light aimed from behind-above
   the actor so it never disturbs the visible sculpt.
5. Ground contact: a soft dark gradient plane (the contact blob) under the
   actor plus an invisible `.shadowOnly` catcher plane for the real shadow.
6. Reflective materials sample a small programmatic environment map, not a
   photo. Keep the zone behind the camera dark.
7. All choreography is baked `CAKeyframeAnimation`, guarded by generation
   tokens so any new beat supersedes pending work.

Full detail with code: [references/stage-recipe.md](references/stage-recipe.md)

## The traps that cost the most time

| Trap | Fix |
| --- | --- |
| Deferred shadows never render when MSAA is on, with zero console errors | Use `shadowMode = .forward` plus a `.shadowOnly` catcher. Debug any missing shadow by first giving the scene a visible gray lambert floor. |
| The first frame of a fresh `SCNView` compiles Metal pipelines in the middle of your entrance animation | `prepare([scene])` in the background, park the actor offstage, delay the first entrance. Never start an animation on frame one. |
| `fillMode = .forwards` + `isRemovedOnCompletion = false` pins the presentation, and removal is per object | One central `clearAnimations` that sweeps the node, every child geometry, the lights, and running actions, called from every entry point. |
| Face-on real metal renders as a gray or black hole | A mirror viewed head-on reflects the environment zone behind the camera. Keep the approved painted base and add a thin additive layer with its own reflection map. |
| Toggling `castsShadow` pops a blurred penumbra in one frame; `shadowBias` and the light's `categoryBitMask` are ignored for forward directional shadows | Keep `castsShadow` on permanently and animate `shadowColor` alpha, synchronized with the motion. |
| A fixed warm-up delay still hitches on cold devices and the Simulator; a particle effect's first frame compiles its own pipeline and drops exactly when it fires | Two-key ignition: gate the entrance on the minimum delay AND `prepare`'s completion handler. Warm particle pipelines with a zero-opacity burst matching the real effect's flags. |
| The default `UIGraphicsImageRenderer` format inherits screen scale, inflating every generated texture 9x in pixels - deadly for textures re-rendered live (per-keystroke engraving) | Pin the renderer format's scale to 1 and size the canvas to the actor's on-screen projection; coalesce multi-input retargets to one render per update pass. |
| A square image assigned to `scene.lightingEnvironment` is silently ignored - zero reflections, every mirror material renders black, no console output | Paint the environment map in a recognized cube-map layout, easiest a 2:1 spherical canvas (1024x512). Only `material.reflective` accepts a square sphere map. |

## Reference map

- [references/stage-recipe.md](references/stage-recipe.md): the full stage,
  view setup, coordinator, node hierarchy, camera, lighting rig, contact blob
  and catcher, programmatic environment maps and textures, impact particles
  on a stage, shader modifiers on stage actors (the linear-space uniform
  trap), keeping the stage's SwiftUI identity (remount and update-storm
  traps).
- [references/first-frame-and-warmup.md](references/first-frame-and-warmup.md):
  the Metal pipeline-compile hitch at first draw and both cures, warm-up
  with a delayed entrance (upgraded to two-key ignition gated on prepare's
  completion), or keeping the scene mounted warm and retargeting it;
  particle pipeline warm-up; proving the cure with signposts and on-device
  hitch traces.
- [references/shadows-and-lights.md](references/shadows-and-lights.md): every
  silent shadow trap, the reliable forward + shadowOnly combo, animating
  shadow visibility, the neutral light budget, face-on metal.
- [references/baked-animation.md](references/baked-animation.md): why baked
  keyframes beat timers, the cue sheet for designing multi-phase beats
  before baking them, seam classification (C1 vs contact impulses),
  designing weight, dense sampling, cleanup bookkeeping, generation
  tokens, stealing a node mid-animation, rolling without sliding, rolling
  along floor paths (steering, screen-space staging, debug trails), channels
  beyond transforms (morpher weights, lens values, shader uniforms), springs
  as authoring material, one property one owner.
- [references/physics-without-engine.md](references/physics-without-engine.md):
  the hand-rolled fixed-step integrator baked to keyframes, walls and floor as
  plain numbers, contact-driven haptics, a rim-pivot topple, and why this
  beats SCNPhysics for choreographed scenes.
- [references/gestures-and-haptics.md](references/gestures-and-haptics.md):
  pan-to-grab without hit testing, soft clamps while held, release velocity,
  impact haptics, scripted beats surviving live fingers (bounded retries,
  tokened polls, hidden-actor gesture guards, steal closes the hold
  contract), Reduce Motion as a taxonomy, VoiceOver access to an invisible
  stage, coexisting with SwiftUI gestures.
- [references/sequencing-stages.md](references/sequencing-stages.md):
  directing several stages as one film - a master clock with beats as
  data, wall-time drift traps, cuts on motion after the exit clears,
  pre-mounting the next stage for warmup, a stage outliving its own cut
  (lingering smoke), re-basing the timeline on a user interaction, a
  recording lead, verifying cuts frame by frame.
- [references/modeling-actors.md](references/modeling-actors.md): building
  believable hero objects from primitives - real-world ratios before
  eyeballing, annuli for recessed faces, tube-plus-torus silhouettes, radial
  pattern legibility, per-instance tilt for concave faces, open gaps, relief
  features as geometry (never paint), satin metal albedo on dark stages,
  gating actors that arrive as files.
- [references/camera-choreography.md](references/camera-choreography.md):
  the camera as the performer - the orbit rig, baked reveal flights, focus
  riding the dolly, drag-orbit with inertia and fly-home, SCNFloor
  reflections as grounding, matte staging under downlights, constraints for
  tracking rigs (and why they never go on the actor).
- [references/instancing-and-swarms.md](references/instancing-and-swarms.md):
  dozens of actors at once - one geometry for the swarm, slot-based piles
  instead of physics, parabolas solved backward from the landing spot,
  tumble blended to a rest pose, seeded randomness, stagger by
  scheduling, budget notes.
- [references/multi-actor-physics.md](references/multi-actor-physics.md):
  several actors in one baked simulation - N state vectors on one clock,
  pairwise collisions (separate, then exchange when approaching), the
  freeze-the-world grab, hit-testing which actor the finger picked,
  per-actor contact lists for squash and haptics.

## Review checklist

Before shipping a stage, verify:

- [ ] `backgroundColor = .clear` and `isOpaque = false` on the SCNView; the
      stage composites with no visible box or seam.
- [ ] No animation starts on the view's first frame; `prepare` runs and the
      first entrance is delayed or the scene is kept mounted warm.
- [ ] Shadows use forward mode with a `.shadowOnly` catcher; nothing relies on
      deferred shadows, `shadowBias`, or light `categoryBitMask` gating.
- [ ] Alpha-textured planes (labels, decals) have `castsShadow = false`, or
      they will cast square shadows.
- [ ] Shadow visibility changes ramp `shadowColor` alpha; `castsShadow` is
      never toggled at runtime.
- [ ] If the scene must render a texture color-faithfully, ambient plus
      directional times cos(incidence) sums to neutral (1000) for a flat
      surface; no clipped light grays.
- [ ] Any image on `lightingEnvironment` (or `background`) uses a
      recognized cube-map layout - 2:1 spherical, 6:1 or 1:6 strip, or six
      images; a square image there is silently ignored and reflective
      materials render black.
- [ ] Every async continuation (delayed entrances, scheduled haptics, chained
      transitions) captures a generation token and re-checks it before acting.
- [ ] Any code that steals an animating node snapshots `.presentation` values
      into the model before calling `removeAllAnimations`.
- [ ] Baked animations set the model to the final pose before `addAnimation`;
      nothing depends on `fillMode = .forwards` without a cleanup sweep.
- [ ] Multi-phase beats have a cue sheet comment above the pose function
      that matches the phase-boundary struct; phase and cue times live in
      that one struct, never as literals in the sample loop or in scheduled
      haptics. Every seam is classified: design seams are C1, impulse seams
      occur only at contact events and each lands a cue.
- [ ] Rolling objects slave spin to distance divided by radius; nothing slides.
- [ ] Eases on paths that enter or leave the frame keep a nonzero arc-rate
      at both edges of the VISIBLE window; a path appended to a resting
      actor starts with its tangent dead on the actor's roll axis (actor
      length amplifies any first-frame heading snap).
- [ ] Any scripted beat that can meet a live finger uses a bounded,
      token-guarded retry, never a single shot behind a silent guard; every
      gesture entry point refuses a hidden or unmounted actor before
      bumping tokens.
- [ ] When a motion bug is reported, the path is drawn as a ~4pt flat
      ribbon (the bake's own points, unlit, depth-test off) BEFORE tuning
      anything - and the ribbon is removed once the shape is approved.
- [ ] In a multi-scene sequence: the script clock advances by wall-time
      deltas from a static publisher, every cut lands after the outgoing
      exit clears the frame, and the next stage pre-mounts parked offstage.
- [ ] Haptics respect the app's haptics preference; motion-driven extras
      (gyro parallax, shake) respect Reduce Motion.
- [ ] 60 Hz work (motion callbacks, per-frame updates) stops when the view
      leaves the window and restarts on reattach.
- [ ] `updateUIView` diffs a state key; a pure SwiftUI re-render replays
      nothing.
- [ ] The stage's SwiftUI identity is stable: never conditionally present
      (hide with opacity), no closure parameters on the representable, and
      `.id` used only as a deliberate remount lever.
- [ ] The stage exists for VoiceOver: one accessibility element with a label
      and value, gestures mirrored as adjustable or named actions, and
      looping ambient motion removed (not slowed) under Reduce Motion.
- [ ] Textures are capped well under the GPU's texture size limit (16384 px
      on modern iPhones, 8192 px on older GPUs) and any texture cache is
      released when the stage leaves for good.
- [ ] Generated textures render at format scale 1, sized to the actor's
      on-screen projection; live-retargeted textures coalesce to one render
      per update pass.
- [ ] The first entrance is gated on `prepare`'s completion handler, not a
      fixed delay alone; particle pipelines are warmed (zero-opacity burst,
      flags matching the real effect) before their first real frame.
- [ ] Impact particles are alpha-blended, distance-sorted, unlit with a
      per-rig flat color, and their emitter shapes never intersect the actor.
- [ ] Haptic generators are prepared and reused per style, never built
      inline inside scheduled closures.
