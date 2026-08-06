# PropMotion

A Claude Code skill for making 3D objects move well in SwiftUI apps.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
![Claude Code](https://img.shields.io/badge/Claude%20Code-agent%20skill-d97757)
![Swift](https://img.shields.io/badge/Swift-SwiftUI%20%2B%20SceneKit-F05138?logo=swift&logoColor=white)
![Platform](https://img.shields.io/badge/platform-iOS-blue)
![Version](https://img.shields.io/badge/version-1.6.2-green)

<p align="center">
  <img src="media/skill-demo.gif" width="360" alt="PropMotion demo reel" />
</p>

## What it's for

Sometimes an app needs one 3D moment. A product spinning on a pedestal. An
achievement badge that rolls in and thuds. A coin that drops when the user
hits a goal. Not a game, not AR. One object that moves well.

SceneKit does all of this and sits in any SwiftUI layout, but the knowledge
of how is scattered and half of it is undocumented. The docs tell you what
an SCNNode is. They don't tell you that deferred shadows silently break
under MSAA, or that the first frame of a fresh SCNView compiles shaders in
the middle of your entrance animation. This skill is the missing manual,
written for an agent.

## What you can make with it

- an object that rolls onto the screen, settles, and casts a real shadow
  over your plain SwiftUI background
- grab it, throw it, it bounces off the edges; every impact taps the
  haptic engine
- weight you can see: heavy things rise slow, fall fast, land with a thud
  and a shiver
- coins raining into a pile, marbles colliding, a wheel doing a burnout
  with smoke
- several scenes cut into one continuous clip, like the one above

All geometry from primitives in code, all motion baked keyframes with
hand-rolled physics. No Unity, no Blender, no assets to license.

## What's inside

One SKILL.md the agent reads first, and ten reference files it pulls in
when the task calls for them. Roughly:

- how a stage is built. Transparent SCNView over your SwiftUI layout,
  a camera that behaves, three lights and a separate shadow rig
- how motion is designed. Cue sheets before code, baked keyframes instead
  of timers, a tiny integrator instead of a physics engine
- how it survives real fingers. Grabs, throws, haptics, and what happens
  when the user interrupts a beat halfway through
- how single scenes get cut into a sequence like the clip above
- the traps. Each one named, each one with the fix that actually works

The writing is opinionated because the alternatives were tried and looked
worse on screen.

## Install

Claude Code: copy the `scenekit-product-stages` folder into
`~/.claude/skills/`, so the file lands at
`~/.claude/skills/scenekit-product-stages/SKILL.md`. That folder is the
skill; everything else in this repo is for you, not the agent.

Claude.ai: zip the `scenekit-product-stages` folder and upload it under
Settings > Capabilities > Skills.

Then ask for something concrete. "A coin that drops onto the table, bounces
twice and settles" is a better first prompt than "add 3D".

## Testing

`evals.md` has the scenarios I use to check the skill triggers when it
should and stays quiet when it shouldn't. Run them in a fresh session.

## License

MIT. Take it, ship things with it.
