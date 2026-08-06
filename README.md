# PropMotion

Make one 3D object perform in your SwiftUI app.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
![Claude Code](https://img.shields.io/badge/Claude%20Code-agent%20skill-d97757)
![Swift](https://img.shields.io/badge/Swift-SwiftUI%20%2B%20SceneKit-F05138?logo=swift&logoColor=white)
![Platform](https://img.shields.io/badge/platform-iOS-blue)
![Version](https://img.shields.io/badge/version-1.6.2-green)

[![PropMotion demo reel](media/skill-demo-poster.png)](media/skill-demo.mp4)

Four actors, one continuous take, everything built and animated with the
skill's recipes: roll-in entrances, an interactive grab-and-throw, a
burnout with particle smoke, and a loaded rack going down. Click the frame
to watch (26 s).

You have a SwiftUI app and one 3D object that has to look expensive: a
product, a badge, a coin, a wheel. This skill teaches an agent to build the
whole scene around it in SceneKit: a transparent stage composited over your
layout, studio lighting with a real shadow, an entrance that rolls in with
correct physics, drag and throw that feels heavy, haptics on every impact.

No game engine, no 3D artist, no asset files. Geometry is built from
primitives in code, and every animation is baked keyframes driven by a
small hand-rolled integrator, so motion is deterministic, interruptible,
and cheap. The references also cover the parts that normally cost days of
trial and error: shadows that silently refuse to render, metal that draws
black, the first-frame shader hitch, SwiftUI remounts that replay
entrances, and cutting several scenes into one continuous sequence.

The skill package itself is named `scenekit-product-stages` (the install
id); PropMotion is the name on the poster.

## Repository layout

- `scenekit-product-stages/` is the skill itself (SKILL.md plus
  references/). This is the only folder you install or upload.
- `README.md`, `evals.md`, and `LICENSE` are repository-level material for
  human readers and are not part of the skill package.

## Install

Claude Code: copy the `scenekit-product-stages` folder into
`~/.claude/skills/` so the file lives at
`~/.claude/skills/scenekit-product-stages/SKILL.md`. Claude Code discovers
it automatically; the references directory is loaded on demand as the task
requires.

Claude.ai: zip the `scenekit-product-stages` folder and upload it via
Settings > Capabilities > Skills.

## Testing

`evals.md` contains manual test scenarios: positive triggers, a paraphrased
trigger, and negative triggers that must not load the skill. Run each in a
fresh Claude Code session with the skill installed.

## License

MIT. See LICENSE; the skill frontmatter declares the same license. Swap it
for your preferred license before publishing if MIT does not fit.
