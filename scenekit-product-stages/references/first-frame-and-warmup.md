# The First Frame and Warm-Up

## Contents

- Symptoms
- Why it happens
- Fix A: prepare, park, delay
- Fix A+: two-key ignition (gate on prepare's completion)
- Particle pipelines compile separately
- Fix B: keep the scene mounted warm
- Texture prewarming off-main
- Rules
- Proving the cure

## Symptoms

- The very first time the stage appears in an app session, the entrance
  animation drops frames at its start. Every later run of the same animation
  is perfectly smooth.
- Nothing in the console. No warnings, no errors.
- Profiling shows the hitch only on the first appearance, then never again.

If a scene is silky except for its debut, this is the trap.

## Why it happens

A fresh `SCNView`'s first draw compiles the Metal pipeline state it needs:
PBR shading, the forward shadow pass, the MSAA resolve, any shader modifiers.
That compilation is synchronous with the first rendered frames. If
`makeUIView` mounts the scene and starts the entrance in the same frame, the
compile lands inside the animation's opening frames and eats them. Afterwards
the pipelines are cached for the process lifetime, which is why frame two
onward, and every later entrance, is smooth.

The same mechanism bites during handoffs: building Metal state at the moment
a SwiftUI view hands off to a SceneKit view causes a visible stall or a
one-frame flash exactly at the seam.

## Fix A: prepare, park, delay

For a stage that mounts fresh each time, three moves together:

```swift
func makeUIView(context: Context) -> SCNView {
    let view = SCNView()
    // ... view flags, scene, pointOfView ...
    context.coordinator.install(item)

    // 1. Park the actor at the doorway (offstage), so the warm-up
    //    window shows an empty stage, not a frozen actor mid-pose.
    context.coordinator.parkOffstage()

    // 2. Compile every pipeline in the background.
    if let scene = view.scene { view.prepare([scene], completionHandler: nil) }

    // 3. Hold the first entrance until the window has passed.
    context.coordinator.scheduleFirstEntrance()
    return view
}
```

```swift
extension StageCoordinator {
    static let warmUpDelay: TimeInterval = 0.35

    func scheduleFirstEntrance() {
        let token = generationToken
        DispatchQueue.main.asyncAfter(deadline: .now() + Self.warmUpDelay) { [weak self] in
            guard let self, self.generationToken == token else { return }
            self.runEntrance()
        }
    }
}
```

Notes:

- 0.3 to 0.4 seconds is enough in practice, and a presenting sheet or push
  transition covers the pause completely; the user never sees a wait.
- Token-guard the delayed start (see
  [baked-animation.md](baked-animation.md)); if a state change schedules a
  transition meanwhile, the newer beat supersedes the parked entrance.
- Parking matters as much as delaying. Without it, the warm-up window shows
  the actor standing dead center before it "enters", which reads worse than
  the hitch. If the parked spot can project INSIDE the frame (a deep
  doorway point gets pulled centerward by perspective), hide the actor's
  node while parked and unhide on the entrance's first frame - the park
  position being "offstage" in world units does not guarantee offscreen.

## Fix A+: two-key ignition (gate on prepare's completion)

A fixed delay alone RACES the compile. On a cold device, and especially on
the Simulator (slowest Metal compile), `prepare` can outlast any curtain you
picked, and the entrance starts mid-compile anyway - same hitch, now
intermittent. Fire the entrance only when BOTH keys turn: the minimum
curtain delay has passed AND `prepare`'s completion handler reported done.
Add a hard fallback (about 2 s) in case the handler never fires.

```swift
view.prepare([scene]) { [weak coordinator] _ in
    DispatchQueue.main.async { coordinator?.markPipelinesWarm() }
}
coordinator.scheduleFirstEntrance()
```

```swift
func markPipelinesWarm() {
    pipelinesWarm = true
    fireFirstEntranceIfReady()
}

func scheduleFirstEntrance() {
    firstEntranceToken = generationToken
    DispatchQueue.main.asyncAfter(deadline: .now() + Self.warmUpDelay) { [weak self] in
        self?.curtainPassed = true
        self?.fireFirstEntranceIfReady()
    }
    DispatchQueue.main.asyncAfter(deadline: .now() + 2.0) { [weak self] in
        guard let self, !self.pipelinesWarm else { return }
        self.pipelinesWarm = true          // prepare never called back
        self.fireFirstEntranceIfReady()
    }
}

private func fireFirstEntranceIfReady() {
    guard pipelinesWarm, curtainPassed,
          let token = firstEntranceToken, generationToken == token else { return }
    firstEntranceToken = nil
    runEntrance()
}
```

The completion handler may arrive on a background queue - hop to main.

## Particle pipelines compile separately

`prepare([scene])` does NOT cover particle systems that are added later
(an impact burst, a celebration). The first particle draw compiles its own
Metal pipeline state - in the exact frame the effect fires, which for an
impact effect is the one frame that must not drop. Warm it during an idle
moment with a one-shot burst rendered at zero opacity: drawn, so the state
gets built, but never seen.

```swift
func warmParticlePipelines() {
    let holder = SCNNode()
    scene.rootNode.addChildNode(holder)
    for sprite in effectSprites {
        let system = SCNParticleSystem()
        system.loops = false
        system.emissionDuration = 0.02
        system.birthRate = 100
        system.particleLifeSpan = 0.2
        system.particleImage = sprite
        system.blendMode = .alpha           // MATCH the real effect's flags:
        system.isLightingEnabled = false    // lit and unlit compile different
        // opacity-over-life controller pinned to 0 - invisible but drawn
        system.propertyControllers = zeroOpacityControllers()
        holder.addParticleSystem(system)
    }
    holder.runAction(.sequence([.wait(duration: 1), .removeFromParentNode()]))
}
```

Match the warm-up system's render-affecting flags (blend mode, lighting,
sorting) to the real effect - a lit and an unlit particle system compile
different pipeline variants, and warming the wrong one buys nothing.

## Fix B: keep the scene mounted warm

When the same kind of scene plays repeatedly (one animation per item in a
sequence), do not tear the view down between plays. Build the scene once,
keep the SCNView mounted, and retarget it cheaply for each new item:

- Swap the texture on the existing material
  (`material.diffuse.contents = newImage`), reset animated state to pose
  zero, and trigger the play with a counter that the representable bumps
  (diffed in `updateUIView`), not by remounting.
- If item dimensions can vary (different aspect ratios), resize the
  dependent geometry and any per-geometry uniforms at retarget time. A warm
  scene sized for the first item silently distorts the third.
- The warm view has already compiled every pipeline, so each play starts
  clean with zero first-frame cost.

Cost: the view holds GPU resources while mounted. For a bounded flow (an
import, a wizard) that is the right trade; release everything when the flow
ends.

One more seam to respect with a warm scene: SwiftUI and SceneKit apply
changes on different frame schedules. If the SwiftUI content underneath the
scene changes in the same instant the scene retargets, the mismatch can flash
for one frame. Structure handoffs so the content under the 3D view is
identical on both sides of the swap.

## Texture prewarming off-main

If entrances need generated or processed textures (rounded corners, resizes),
produce them before they are needed and off the main thread:

```swift
enum TexturePipeline {
    nonisolated static func prewarmTexture(_ image: CGImage) {
        _ = processedTexture(image)   // fills the cache
    }
}
```

- `NSCache` and `UIGraphicsImageRenderer` are thread-safe; a nonisolated
  static pipeline is fine and keeps the work off the critical path.
- Cap processed sizes far below the GPU's texture limit (16384 px on
  modern iPhones, 8192 px on older GPUs) and at the size the
  stage actually renders (a 1600 px long side is plenty for a card-sized
  actor); oversizing doubles memory for zero visible gain.
- Give the cache a real byte budget and empty it when the stage leaves for
  good; an app-lifetime static cache otherwise sits on dead bitmaps.

## Rules

- Never start an animation on the same frame as a fresh SCNView's first
  render.
- `prepare(_:completionHandler:)` plus a short covered delay is the cheap
  general cure; a warm mounted scene is the strong cure for repeated plays.
- The warm-up window must look intentional: park the actor offstage or show
  an empty stage, never a frozen mid-pose.

## Proving the cure

Every cure in this file can be VERIFIED, and the claim "the warmup works" is
falsifiable:

- Wrap the warmup window in an `os_signpost` interval from code - begin at
  mount, end when the first entrance starts. In an Instruments trace this
  turns "somewhere near launch" into an exact window to inspect.
- The Animation Hitches instrument attributes every hitch to the app or to
  the system. The pass criterion is zero APP-attributed hitches inside the
  entrance window, measured on a COLD launch - the pipeline compile happens
  once per process, so a warm relaunch proves nothing.
- Instruments' representable-update lane separates the two ways a stage
  kills frames: long `updateUIView` runs (a diffing bug - the state key is
  not absorbing SwiftUI's redraw traffic) versus a hitch inside the render
  itself (scene cost). The cures are different; measure before choosing.
- The Simulator cannot run this verification. Its SwiftUI instrument lane
  records empty and its GPU timing is not the device's; hitch measurement
  is on-device only. Canvas previews are worse still - they do not render
  the Metal layer at all, so the only preview mode that says anything about
  a stage is preview on a physical device.

One lifecycle nuance extends the "60 Hz work stops off-window" rule:
backgrounding the app does NOT cancel `.task` work or stop a display-driven
loop - those die on view-tree teardown, and backgrounding is not a view-tree
event. A stage with any per-frame work watches the scene phase explicitly
and pauses itself when the app leaves the foreground.
