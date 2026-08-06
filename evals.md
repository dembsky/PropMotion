# Evaluations

Test scenarios for the scenekit-product-stages skill, following the
evaluation structure recommended in the official skill authoring guidance.
This file lives at the repository root and is not part of the skill
package; the skill itself is the `scenekit-product-stages` folder. There is
no built-in runner; use each case as a manual test in a fresh Claude Code
session with the skill installed.

Evals 1-3 are direct triggers tied to functional expectations. Eval 4 is a
paraphrased trigger with no framework keywords in the obvious places. Eval
5 lists queries that must NOT load the skill. For a baseline comparison,
run evals 1-3 once with the skill disabled and compare how many user
corrections each session needs before reaching the same fix.

## Eval 1: shadow missing under MSAA

```json
{
  "skills": ["scenekit-product-stages"],
  "query": "My SceneKit scene in a SwiftUI app has an invisible floor plane that should catch a shadow from a directional light, but no shadow ever renders. There are no errors in the console. The SCNView uses antialiasingMode = .multisampling4X and the light uses shadowMode = .deferred.",
  "expected_behavior": [
    "Triggers the scenekit-product-stages skill from the SCNView, SceneKit, and shadow terms",
    "Reads references/shadows-and-lights.md and identifies the deferred-plus-MSAA conflict as the cause, noting it fails silently",
    "Recommends shadowMode = .forward with a .shadowOnly catcher material, and warns that intensity-0 lights are skipped entirely",
    "Suggests the debug recipe of adding a visible gray lambert floor before tuning light angles or intensities"
  ]
}
```

## Eval 2: first-frame hitch on entrance

```json
{
  "skills": ["scenekit-product-stages"],
  "query": "The first time my 3D product animation appears in the app, the entrance drops frames at the start. Every later replay is perfectly smooth and there is nothing in the console. The scene is built in makeUIView and the entrance animation starts immediately.",
  "expected_behavior": [
    "Triggers the skill from the first-frame hitch and jank phrasing",
    "Reads references/first-frame-and-warmup.md and attributes the hitch to Metal pipeline compilation during a fresh SCNView's first draw",
    "Recommends prepare([scene]) in the background, parking the actor offstage, and delaying the first entrance behind a covered warm-up window, with a generation-token guard",
    "Mentions the alternative keep-the-scene-mounted-warm strategy for repeated plays, and the rule to never start an animation on frame one"
  ]
}
```

## Eval 3: face-on metal renders black

```json
{
  "skills": ["scenekit-product-stages"],
  "query": "I switched a chrome part of my SceneKit product model to a physically based metal material and now, viewed head-on, it renders as a gray-black hole. Tilted, it looks fine. Brightening the environment map washed out other surfaces instead of fixing it.",
  "expected_behavior": [
    "Triggers the skill from the SceneKit metal material terms",
    "Reads references/shadows-and-lights.md and explains that a face-on mirror reflects the environment zone behind the camera, which studio maps keep dark",
    "Recommends keeping the approved base look and overlaying a thin additive layer (constant lighting model, add blend mode) with its own reflection map that is black in the rest-pose sampling zone",
    "Advises testing metal materials in multiple poses, never only face-on"
  ]
}
```

## Eval 4: paraphrased trigger, grounding a floating object

```json
{
  "skills": ["scenekit-product-stages"],
  "query": "The 3D hero object on my iOS app's home screen looks like it is floating in the air. There is no sense of contact with the ground under it and the whole thing reads as pasted on. It renders in an SCNView over my SwiftUI layout.",
  "expected_behavior": [
    "Triggers the skill from the SCNView, SwiftUI, and 3D hero-object phrasing despite no mention of shadows or lights",
    "Reads references/stage-recipe.md and identifies missing ground contact as the cause",
    "Recommends the contact blob (soft dark gradient plane) plus an invisible .shadowOnly catcher with a forward-mode shadow light",
    "Warns that deferred shadows fail silently under MSAA if the user tries shadows first"
  ]
}
```

## Eval 5: queries that must NOT trigger

The skill must stay unloaded for these. If it loads, tighten the
description's negative triggers.

```json
{
  "skills": ["scenekit-product-stages"],
  "queries": [
    "My RealityKit entity in a visionOS app does not cast a shadow on the floor anchor.",
    "How do I add a drop shadow to a button in SwiftUI? The .shadow modifier looks too heavy.",
    "Help me set up an ARKit session that places furniture in the user's room.",
    "Write a Three.js scene with a spinning product model for my website."
  ],
  "expected_behavior": [
    "Does not load the skill for any of these queries",
    "Answers RealityKit, visionOS, ARKit, plain SwiftUI, and web 3D questions without SceneKit stage patterns"
  ]
}
```
