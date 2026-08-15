# Relief Actors

A die-struck object - a medallion, a coin face, a club badge - is not a
primitive assembly and not a shader deformation: it is a HEIGHTFIELD,
authored on the CPU from distance fields, closed into a solid mesh, and
dressed by a CLASS MAP that puts several materials on one geometry.
Built and verified on a vintage enamel medallion: polished rim, enamel
ring, fine concentric metal lines, a recessed patterned field, and logo
strokes laid as enamel cells between raised metal partitions.

## Contents

- Height and class painted together
- Profiles: parabolic, never circular
- The silhouette needs three guarantees at once
- Normals from a blurrier copy
- Judge flat actors front-on, frozen
- Hash arithmetic that crashes silently
- Know the renderer's ceiling

## Height and class painted together

Author TWO rasters in the same pass over the same grid: the height (in
world units) and a class byte per texel (gold, enamel, ivory cell,
recessed field...). The height becomes the mesh; the class map becomes
the per-texel material textures:

- walk the class raster once and emit `diffuse`, `metalness`, and
  `roughness` images from a small table of finishes;
- the mesh gets ONE material with those textures, and every ridge,
  cell, and partition lands pixel-exact on its own finish because both
  rasters came from the same distance fields.

This is the difference between "a gold disc with a sticker" and
champagne-gold ridges rising out of glassy enamel: the metal partitions
are geometry AND material, a fraction of a millimeter tall, from one
authoring pass. Artwork arrives as distance fields (letter strokes,
rings, diamond grids), so every feature has a signed distance to drive
both the height profile and the class boundary.

## Profiles: parabolic, never circular

The tempting dome profile `sqrt(1 - t^2)` has INFINITE slope at its rim:
the triangulation turns that edge into a vertical wall that shimmers
with aliasing at any resolution. Use the parabolic arch instead:

```swift
func ridge(_ t: Float) -> Float { t * (2 - t) }   // t in 0...1
```

Finite slope where the feature meets the ground, C1 at the crest, and
visually still "a struck ridge". Every raised feature in the actor -
rim, hairlines, partitions, the ridge profile of letter strokes - is
this arch over its distance field.

## The silhouette needs three guarantees at once

A closed relief mesh is a front shell and a back shell meeting at the
outline. The failure mode is a ragged gray halo around the silhouette
that SURVIVES single fixes, because three mechanisms overlap:

1. The smoothing blur leaks height past the outline mask and builds a
   wide, flat mirror APRON around the shape. Fix: shave a small constant
   off the field AFTER the blurs; the apron dies, the features barely
   notice.
2. Front and back shells meet at exactly zero height, and the coincident
   triangles z-fight as speckle. Fix: a minimum field thickness
   (`max(h, epsilon)`) so the shells are always separated by a real
   side wall.
3. An infinite-slope edge profile grinds the outline into aliasing -
   the parabolic arch above is the third leg, not an independent nicety.

All three at once, or the halo returns wearing a different face. And
diagnose by ZOOMING INTO THE EDGE of a frozen frame, not by cycling
guesses - the three sources look identical at arm's length and
completely different at 8x.

## Normals from a blurrier copy

A mirror finish prints every quantization step of the heightfield as a
contour terrace - the position data is fine, but normals derived from it
inherit every stair. Keep two copies of the field:

- the SHARP field builds vertex positions (features stay crisp);
- a more heavily blurred copy feeds the normal derivation (shading goes
  buttery).

This is the CPU-mesh equivalent of a normal map disagreeing with
displacement on purpose. Without it, polishing the metal makes the
terraces MORE visible, not less.

## Judge flat actors front-on, frozen

A broad, nearly flat face is a mirror reflecting ONE small zone of the
environment. Two consequences:

- Give every piece full doming (a fraction of the inflate on the crown,
  ZERO flat plateaus) - a plateau reflects a single environment value as
  a dead, uniform sheet that reads as toy plastic.
- Evaluate the actor FRONT-ON on a frozen frame. A turntable spin
  masks dead plateaus, because the reflection sweeps and shimmers; the
  resting pose the user actually stares at is where the flatness shows.

Environment structure for mirror-flat actors is its own topic - see the
finite-cards section of [hdr-environments.md](hdr-environments.md).

## Hash arithmetic that crashes silently

Procedural noise hashes love expressions like
`UInt32(bitPattern: Int32(ix &* K1 &+ iy &* K2))`. The `&*` wraps in
Int's 64 bits, but the NARROWING `Int32(...)` init does not wrap - it
TRAPS at runtime on overflow. Worse, when the mesh builds inside an
async task, the crash surfaces as... nothing: no console output, an
empty stage, a scene that "just never appears".

Rule: feed hash inputs into the target width up front with
`UInt32(truncatingIfNeeded:)` and do ALL the arithmetic there. And when
a stage is inexplicably empty, check the system's crash reports before
debugging scene logic - a background SIGTRAP is invisible from the app.

## Know the renderer's ceiling

Process lesson, learned the expensive way: when the visual reference is
an OFFLINE or AI render (path-traced multi-bounce reflections, area-light
soft shadows), name the real-time rasterizer's ceiling BEFORE the first
polish round and agree with the owner what "good enough" means. SceneKit
has one reflection bounce from a static environment and no area shadows;
iterating toward a path-traced look approaches that ceiling
asymptotically, with each round costing more than the last.

You can tell you are at the ceiling when every next improvement is a NEW
TECHNIQUE (screen-space tricks, jitter, another environment rebuild)
instead of a new value for an existing dial. That is the moment to stop
and renegotiate the target, not to schedule another round.
