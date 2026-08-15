# Mesh Surgery

Cutting an actor apart at runtime: a finger swipe becomes a plane, the
mesh splits into two SEALED halves, and the smaller piece tumbles to the
floor - clip the very tip and only the tip drops, halve the actor and the
lesser half goes. Built and verified on a slice-a-fruit mechanic where
whatever remains can be sliced again and again.

The pipeline stays inside this skill's philosophy: the cut is pure
geometry code, and the fall is the hand-rolled integrator from
[physics-without-engine.md](physics-without-engine.md) baked to
keyframes - no `SCNPhysicsBody` anywhere.

## Contents

- The mesh as plain arrays
- The cut plane from a swipe
- Clipping into two sealed halves
- The cap: convexity makes it trivial
- Cap UVs: one radial artwork for every cross-section
- The integrals decide the gameplay
- The fall of an arbitrary chunk
- Hold to aim, commit on release
- Scripted cuts that survive re-slicing
- Staging notes
- Honest limits

## The mesh as plain arrays

`SCNGeometry` buffers are write-only for this job; keep the authoritative
mesh as plain arrays and rebuild the geometry after every cut:

```swift
struct SliceMesh {
    var positions: [SIMD3<Float>] = []
    var normals: [SIMD3<Float>] = []
    var uvs: [SIMD2<Float>] = []
    var skinIndices: [Int32] = []   // the textured outer surface
    var capIndices: [Int32] = []    // the flat cut faces
}
```

Two index groups over ONE vertex pool become two `SCNGeometryElement`s
with `geometry.materials = [skin, flesh]` - the outer artwork and the
cross-section artwork live on the same mesh, and both survive any number
of further cuts because the clipper treats the groups symmetrically.

Generate the starting shape yourself (a UV sphere is a few lines) so you
own the data. For ellipsoid variants, scale positions and remember the
normal is NOT the scaled direction: for scale `s`, the ellipsoid normal
is `normalize(direction / s)` - the gradient of the implicit surface.

## The cut plane from a swipe

A swipe on glass defines the plane a blade would sweep: the segment,
extruded along the view direction.

```swift
let projected = view.projectPoint(SCNVector3(actorCenter))   // read .z
let a = view.unprojectPoint(SCNVector3(start.x, start.y, projected.z))
let b = view.unprojectPoint(SCNVector3(end.x, end.y, projected.z))
let normal = simd_normalize(simd_cross(simd_normalize(b - a),
                                       cameraNode.presentation.simdWorldFront))
let plane = CutPlane(normal: normal, d: -simd_dot(normal, a))
```

Unproject both endpoints AT THE ACTOR'S DEPTH (project the actor center
once, reuse its screen-space z), so the segment lies in the
camera-parallel plane through the actor. `simdWorldFront` of the camera
node is the view direction. Move the plane into the actor's local space
through the PRESENTATION transform - the actor may be mid-bob when the
finger releases.

## Clipping into two sealed halves

Per triangle, classify the three vertices by signed distance (snap
`|d| < 1e-5` to zero), and:

- all on one side: copy the triangle whole (an index map deduplicates
  copied vertices);
- mixed: Sutherland-Hodgman the triangle against the plane TWICE, once
  keeping each half-space, fan-triangulate each resulting polygon with
  fresh vertices, and record the two crossing points.

Crossing vertices interpolate position, normal, and UV by
`t = da / (da - db)`; renormalize the lerped normal. A vertex exactly on
the plane lands in both halves - the degenerate slivers this can produce
have zero area and zero volume, and cost nothing.

The clip must run over BOTH index groups (skin and caps), because from
the second cut onward the plane crosses earlier cut faces too. If either
output half ends up empty, the plane missed the mesh: return nil and
ignore the gesture - never leave a half-open shell.

## The cap: convexity makes it trivial

The crossing points of all clipped triangles trace the cut's boundary
loop. Sealing it usually means general polygon triangulation - but not
here: a ball intersected with any number of half-spaces stays CONVEX, so
every cross-section of every piece is a convex polygon, and the cheap
path is provably correct:

1. Weld the crossing points on a coarse grid (~5e-4 units) - shared
   edges emit duplicates.
2. Build a 2D basis on the plane (`u` = any perpendicular to the normal,
   `v = cross(normal, u)`), sort the welded points by
   `atan2(dot(p - c, v), dot(p - c, u))` around the centroid `c`.
3. Fan-triangulate from the centroid.

Winding and normals: sorted ascending-angle around `+n`, the fan
`(center, p[i], p[i+1])` faces `+n`. The half that keeps the NEGATIVE
side gets that cap as-is; the positive half gets the ring reversed with
normal `-n`. Both halves are now closed - which is what unlocks
everything below.

## Cap UVs: one radial artwork for every cross-section

Give cap vertices planar UVs around the loop centroid, normalized by the
loop's max radius:

```swift
uv = SIMD2(dot(offset, u) / loopRadius * 0.5 + 0.5,
           dot(offset, v) / loopRadius * 0.5 + 0.5)
```

The loop rim always lands at HALF the texture width from the center, so
one radially painted flesh image (core, pith ring, rind edge, a seed
ring) serves every cut of every piece at every size. Paint the artwork
treating `side/2` as "the surface" and layer inward. The mapping is
exact for the first cut of a round actor and drifts gracefully on later
cuts whose cross-sections are circle-arc-plus-chord shapes.

## The integrals decide the gameplay

Sealed halves are closed meshes, so signed-tetrahedron sums are valid:

```swift
let tetVolume = simd_dot(a, simd_cross(b, c)) / 6
volume += tetVolume
weighted += (a + b + c) / 4 * tetVolume        // centroid accumulator
```

Iterate skin AND cap indices, take `abs(volume)` at the end, and the
game rule is one comparison: the SMALLER volume falls, the larger stays.
Gate slivers with a threshold RELATIVE to the current full volume
(~1%) - an absolute threshold tuned for one actor silently breaks a
smaller one.

Recenter the falling piece's positions on its own centroid before
building its node, and place the node at the centroid's world position:
the baked tumble then rotates about the physical center of mass instead
of orbiting a stale origin. The remaining piece keeps its coordinates
untouched - it must not visibly shift when the geometry swaps.

## The fall of an arbitrary chunk

A cut piece is not a disc or a ball; its floor contact depends on its
orientation. The integrator stays hand-rolled by scanning SUPPORT
SAMPLES - a subsample of the piece's own vertices:

- take every k-th vertex (~200 total) PLUS the min/max vertex on each
  axis, so the truest extremes are always present and the silhouette
  never sinks between samples;
- per step, rotate the samples by the current orientation quaternion;
  the lowest rotated sample is the contact candidate.

The step itself: gravity on the velocity, position integration, exact
quaternion integration (`simd_quatf(angle: |omega| dt, axis:)` composed
onto the orientation), then the contact:

```swift
if position.y + lowestY < floorY {
    position.y = floorY - lowestY                  // clamp out
    let vContact = velocity + cross(omega, lowest) // point velocity
    if vContact.y < 0 {
        let lever = cross(lowest, up)
        let j = -(1 + restitution) * vContact.y
              / (1 + dot(lever, lever) / inertia)  // unit mass
        velocity.y += j
        omega += cross(lowest, up * j) / inertia
        velocity.x *= 0.72; velocity.z *= 0.72     // friction bite
        omega *= 0.85
    }
}
```

A scalar inertia (`0.4 * maxSupportRadius^2`, the sphere formula) is a
deliberate approximation - the lever term is what matters, because an
off-center contact converting fall speed into spin is exactly what makes
a chunk landing on its cap edge kick over and tumble. Seed the flight
with a push along the cut normal (away from the kept piece), a touch of
lift, and a tip-over omega about the horizontal axis perpendicular to
the push. End on a rest test (near floor, small speed, small spin), snap,
and bake `position` plus `orientation` (quaternion `SCNVector4`) as two
keyframe tracks of duration `(count - 1) * dt`. Contacts drive haptics
as always.

## Hold to aim, commit on release

The gesture contract that makes the cut controllable: while the finger
is down, the line from the anchor to the current position is only
PREVIEWED - stream it to the host as view-space points and let SwiftUI
draw the blade (a bright line, a soft glow, a dot on the pivot) in an
overlay with hit testing off. The player can pivot the blade around the
anchor indefinitely; the slice fires once, on release, along the final
line. Guard the whole gesture behind the stage's readiness flag and
clear the preview on cancel and on reset - a blade line surviving its
stage is a bug the player sees immediately.

## Scripted cuts that survive re-slicing

Screenshot hooks want deterministic cuts, and chained cuts must keep
landing after the actor shrinks. Place the scripted plane FRACTIONALLY
along the CURRENT mesh's extent in the cut direction:

```swift
let along = positions.map { dot($0, normal) }
let offset = along.min()! + (along.max()! - along.min()!) * fraction
```

A "tip" preset (~0.75) and a "half" preset (~0.47) then work on whatever
remains - a plane fixed to the original shape misses the leftover half
entirely on cut two and the hook silently no-ops. Chain presets with a
comma list argument, and fire each through the bounded token-guarded
retry from [gestures-and-haptics.md](gestures-and-haptics.md) so a cut
scheduled during the entrance lands in the first clean frame after it.

## Staging notes

- Cut faces stand nearly parallel to the view axis, where a key and a
  shadow sun both graze off - the first build rendered them near-black.
  Give the scene a generous ambient floor (and judge the cross-section
  color at EVERY angle the mechanic can produce, not just the pose you
  built first).
- A one-blink white sheet at the cut plane (an additive `SCNPlane`
  oriented to the normal, faded in 0.05 s and out 0.16 s) is the slash
  the eye expects between the halves parting.
- The remainder recoils off the cut - a small nudge opposite the push on
  its OWN node in the hierarchy, so the recoil never fights the hover or
  the travel channel.
- Cap the floor population (~6 pieces, oldest fades out); a floor of
  forty slivers is clutter, not comedy.

## Honest limits

- Fallen pieces ignore each other - the world model is still floor plus
  walls, so a piece landing on an earlier piece interpenetrates it. If
  the mechanic makes that common, pieces need pairwise contacts from
  [multi-actor-physics.md](multi-actor-physics.md).
- The trivial cap rests on convexity. A concave actor (a torus, a mug)
  breaks both the angle-sort (self-intersecting fan) and possibly the
  single-loop assumption (one plane, two separate holes); that job needs
  real polygon triangulation with hole support before this reference
  applies.
