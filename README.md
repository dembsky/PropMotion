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

## Why this exists

I build Gymscle, a gym tracker with small 3D moments in it. Getting SceneKit to do this kind of work is mostly undocumented. The
docs tell you what an SCNNode is. They don't tell you that deferred shadows
silently break under MSAA. That a square environment map gets ignored
without a single log line and your chrome renders black. That the first
frame of every fresh SCNView compiles Metal shaders right in the middle of
your entrance animation.

I learned all of it the slow way, correcting the agent session after
session. At some point writing the corrections down became cheaper than
repeating them. This repo is that document.

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
