# Deformable Surfaces

A closed surface that breathes - a liquid-metal blob, a jelly wobble, a
poked balloon - as one dense mesh and one geometry shader modifier. The
fog sheet ([fog-and-mist-sheets.md](fog-and-mist-sheets.md)) waves an open
translucent ribbon; this is its solid sibling: an opaque PBR actor whose
whole surface deforms, whose reflections must survive the deformation,
and whose skin answers the finger.

## Contents

- The blob: noise on the unit sphere
- The octave budget: liquid vs rock
- Normals must be rebuilt, and analytic slope is not enough
- Tap ripples as uniform slots
- Mesh density buys ring roundness
- Hit-testing a surface the CPU cannot see
- Amplitude as an animatable dial

## The blob: noise on the unit sphere

Sample 3D value-noise fbm on the vertex's unit direction and displace
along it; drift the noise domain on the stage clock and the surface
morphs like slow liquid:

```metal
float body = orbFbm(dir * 1.05
    + float3(0.11 * clock, 0.07 * clock, -0.05 * clock)) - 0.5;
float d = amplitude * 1.25 * body;
_geometry.position.xyz = dir * (1.0 + d);
```

Sampling on the DIRECTION (not the raw position) keeps the field
continuous across the whole closed surface - no seam at the sphere's UV
wrap, because the noise never sees the parametrization. The clock is the
stage's integrated clock ([rendering-contract.md](rendering-contract.md)),
so the morph freezes with everything else.

All modifier plumbing rules from the stage recipe apply: helpers BEFORE
`#pragma arguments`, `scn_frame`/`scn_node` naming, linear-space color
uniforms, `rendersContinuously` for shader-driven motion.

## The octave budget: liquid vs rock

The difference between molten metal and an asteroid is the octave table.
High octaves put wrinkles on the surface, and the rebuilt normals AMPLIFY
them - every small bump becomes a bright reflection wiggle:

```metal
// Liquid: the first octave dominates, the tail whispers.
return 0.78 * orbNoise(p)
     + 0.17 * orbNoise(p * 2.1 + 7.7)
     + 0.05 * orbNoise(p * 4.2 + 3.1);
```

Weights near 0.6/0.28/0.12 rendered a convincing rock. Tune the tail
octaves against screenshots of the reflections, not the silhouette - the
silhouette forgives wrinkles long before the highlights do.

## Normals must be rebuilt, and analytic slope is not enough

A mirror-finish actor IS its reflections; displaced vertices with the
sphere's original normals render dead, frozen highlights on a moving
surface. The fog sheet tilts its normal by the wave's analytic slope -
that works for one sinusoid. For fbm plus ripples, differentiate
numerically: displace two tangent neighbors and cross the edges.

```metal
float3 up = (abs(dir.y) < 0.98) ? float3(0,1,0) : float3(1,0,0);
float3 tanU = normalize(cross(up, dir));
float3 tanV = cross(dir, tanU);
float eps = 0.014;
float3 dirU = normalize(dir + eps * tanU);
float3 dirV = normalize(dir + eps * tanV);
// displace all three directions with the SAME function...
float3 normal = normalize(cross(pU - p0, pV - p0));
if (dot(normal, dir) < 0.0) { normal = -normal; }
_geometry.normal = normal;
```

Three full displacement evaluations per vertex sounds expensive; it is
not. A 176-segment sphere (~126K vertices) with three-octave fbm plus
four ripple evaluations, times three samples, holds 60 fps on a phone -
the GPU eats hash noise for breakfast. Keep `eps` under a quarter of the
shortest wavelength you displace with, or the derivative aliases.

## Tap ripples as uniform slots

Capillary rings from a poke, without touching geometry from the CPU: a
fixed set of `float4` uniforms, each one ripple - `xyz` the unit-sphere
contact direction, `w` the birth time ON THE STAGE CLOCK. Idle slots park
at a birth far in the past; new pokes claim slots round-robin.

```metal
float orbRipple(float3 n, float4 r, float t) {
    float age = t - r.w;
    if (age <= 0.0 || age >= 3.5) { return 0.0; }
    float ang = acos(clamp(dot(n, r.xyz), -1.0, 1.0));
    // The poke: a dent that relaxes fast.
    float dent = -0.20 * exp(-ang * ang / 0.045) * exp(-4.0 * age);
    // Rings spread behind a front travelling ~0.9 rad/s.
    float front = 0.9 * age;
    float mask = 1.0 - smoothstep(front, front + 0.25, ang);
    float rings = 0.075 * sin(17.0 * ang - 13.0 * age)
        * exp(-2.2 * ang) * exp(-1.5 * age) * mask;
    return dent + rings;
}
```

The pieces that make it read as liquid rather than a decal:

- Distance is the GEODESIC angle (`acos(dot)`), so rings stay round as
  they wrap the far side.
- The front mask keeps rings from existing where the wave has not yet
  traveled - without it the whole envelope blinks on at birth.
- Phase speed (`13/17`) and front speed agree, so crests neither outrun
  nor lag their own wavefront.
- Births on the stage clock mean a frozen clock can show a ripple
  mid-flight: back-date the birth when frozen
  (`birth = frozenTime - 0.7`) and the snapshot catches the rings.

Four slots cover real fingers; a fifth simultaneous tap steals the oldest
ripple and nobody has ever noticed.

## Mesh density buys ring roundness

The ripple wavelength sets the vertex budget: `k = 17` on a unit sphere
is a wavelength of ~0.37 rad, and you want 8-10 vertex rows per
wavelength before the rings polygonize. A 176-segment `SCNSphere` gives
that with margin; halving it prints visibly faceted rings near the poke.
Spend vertices here rather than on octaves - a dense mesh with a quiet
octave tail beats a coarse mesh with detail noise every time.

## Hit-testing a surface the CPU cannot see

`SCNView.hitTest` tests the UNDISPLACED geometry - the shader's bulges
are invisible to it, so a tap on a lobe that swells past the base radius
misses. Give the actor an oversized invisible collider and test that:

```swift
let collider = SCNSphere(radius: 1.3)          // base radius 1.0
let material = SCNMaterial()
material.colorBufferWriteMask = []             // draws nothing
material.writesToDepthBuffer = false
```

Hit-test with `.all` and filter for the collider node by name; normalize
the local hit coordinate into the unit direction the ripple uniform
expects. The collider is a child of the deforming node, so the direction
lands in the same space the noise samples - user rotation composed on the
node costs nothing extra.

## Amplitude as an animatable dial

The blob's temper is one scalar uniform, and the KVC bridge makes it a
free animation channel:

```swift
SCNTransaction.begin()
SCNTransaction.animationDuration = 0.6
material.setValue(amplitude, forKey: "blobAmount")
SCNTransaction.commit()
```

At zero the actor is a perfect sphere - which is also a diagnostic: if
the blob ever renders suspiciously spherical, check who zeroed the dial
before blaming the shader. A slider wired to this uniform is the whole
jelly-wobble control surface; no per-frame CPU, no re-bake.
