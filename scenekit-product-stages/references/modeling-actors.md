# Modeling Product Actors From Primitives

Lessons from building believable hero objects (a 255/35 R19 wheel matched to a
photo reference) out of SceneKit primitives, with no imported assets. Every
rule here was earned by a render that looked wrong first.

## Contents

- Start from the real object's numbers, not your eye
- Recessed faces need annuli, not capped cylinders
- Rounded silhouettes: the tube-plus-torus recipe
- Pattern legibility: angles beat counts
- Concavity by per-instance tilt (euler order trick)
- Leave the gaps open
- Relief features are geometry, not paint
- Satin metal on a dark stage
- Actors that arrive as files

## Start from the real object's numbers, not your eye

Eyeballed proportions drift toward "icon" shapes: too thin, too round, too
symmetric. Before modeling, take 60 seconds to derive two or three ratios from
the real object's spec sheet and anchor the model to them.

Worked example, a 255/35 R19 wheel (tire radius normalized to 1.0):

- Overall diameter ~660 mm, rim 19 in = 483 mm. Rim radius = 0.73 of tire
  radius, so the rubber sidewall band is ~27% of the radius. The first
  eyeballed pass used 20% and read like a rubber-band toy.
- Tire section width 255 mm = 39% of the diameter. The first pass used 17%
  and the toppled wheel read as a coin. 23% was the stage compromise: meaty,
  but the roll entrance still reads as a wheel, not a drum.

The point is not photorealism; it is that one or two verifiable ratios stop
the model from sliding into caricature, and the numbers are one search away.

## Recessed faces need annuli, not capped cylinders

`SCNCylinder` always renders its end caps. Any performer whose face is
supposed to be recessed (a wheel dish, a speaker cone, a jar mouth) must be
built from `SCNTube` rings, whose caps are annuli, or the cap walls off the
recess and hides everything behind it. The first wheel render was a black
disc: solid rubber caps in front of the entire rim.

## Rounded silhouettes: the tube-plus-torus recipe

A soft-edged solid of revolution (tire, gasket, cushion) is three primitives:

```swift
let shoulder = min(width * 0.15, radius * 0.12)
// body wall, pulled in by the shoulder radius
let body = SCNTube(innerRadius: rimR, outerRadius: radius - shoulder, height: width)
// working band at full radius, tucked between the shoulders
let tread = SCNTube(innerRadius: rimR + 0.08, outerRadius: radius, height: width - shoulder * 2)
// torus shoulders round the edge where face meets band
for side in [Float(-1), 1] {
    let ring = SCNTorus(ringRadius: radius - shoulder, pipeRadius: shoulder)
    let ringNode = SCNNode(geometry: ring)
    ringNode.position = SCNVector3(0, side * (width / 2 - shoulder), 0)
    actorNode.addChildNode(ringNode)
}
```

Keep the shoulder modest: at `width * 0.22` the tire ballooned into a donut;
`width * 0.15` reads as an inflated low-profile sidewall. The shoulder is the
single biggest cartoon-vs-real dial in the silhouette.

## Pattern legibility: angles beat counts

A wheel's spoke pattern (or any radial pattern) is read by its ANGULAR
relationships, not by counting elements. A five-double-spoke rim with arms
spread evenly reads as ten plain spokes - exactly what the reference does not
look like. The double-spoke read comes from two facts:

- Arms are EVENLY spaced at the rim: for 5 pairs, outer ends land 36 degrees
  apart, so `outerX = rimR * tan(18 deg)`.
- Arms CONVERGE into pairs at the hub: inner ends nearly touch, forming five
  narrow Vs.

Draw one pair as a single `UIBezierPath` with two tapered quads and extrude
with `SCNShape` (small `chamferRadius` so every edge catches the key light),
then instance it five times rotated by 72 degrees. Keep blades thin and
tapered (wider at hub than rim); blade thickness is what kills the look in
both directions - needles read cheap, slabs read like a fan.

## Concavity by per-instance tilt (euler order trick)

SceneKit applies `eulerAngles` as Rz * Ry * Rx - the x rotation hits the
geometry FIRST, then z places it around the wheel. So one line tilts every
instance about its own radial axis:

```swift
spoke.eulerAngles = SCNVector3(dishTilt, 0, Float(i) * (2 * .pi / 5))
```

With the spoke path pointing along +Y and the node pushed back in z, a tilt
of ~0.24 rad lifts the rim end of each blade toward the tire's face while the
hub end sinks into the dish: a deep concave wheel face from flat extrusions.
Sink the hub, bolt wells, and center cap to matching depths so the recess
reads as one surface.

## Leave the gaps open

Do not close a see-through performer with a floor disc "so the interior looks
dark". Openings between spokes (or handles, or lattice) must show the stage
behind the actor - especially once it lies down and the floor should be
visible through the rim. Depth cues from real holes beat a painted-dark drum
every time. If the interior needs presence, use a narrow annulus lip, never a
full disc.

## Relief features are geometry, not paint

A surface feature that exists to DO something on the real object (tire tread
grooves, knurling, vents) is a depression or protrusion, and the eye checks
for that at grazing angles: painted-on dark stripes stay flat in the
silhouette and under moving light, and the fake is obvious the moment the
actor rotates. The cheap fix for revolution solids is stacked rings: build
the working band as N rib tubes at full radius over one recessed floor tube
(a few percent of the radius deep, near-black, high roughness). Four real
channels out of six SCNTubes - no displacement maps, no imported mesh - and
the crown shades, highlights, and notches the outline correctly from every
angle. Keep only the fine work (sipes, hairlines) in the texture.

Identify the RELIEF CLASS before building. A road-tire tread is NOT blocks:
it is continuous circumferential ribs whose only real depressions are the
channels between them; the "block" impression comes from hairline S-wave
sipes and shoulder slots, which belong in high-contrast crown texture, not
geometry. Physically-gapped block rings read as an off-road tire from any
distance. Match the relief class of the reference photo first; only then
pick between rib tubes plus sipe texture (road tires, vinyl records,
machined parts) and true instanced blocks (off-road tread, cooling fins,
chocolate bars).

When a band genuinely needs BLOCKS, instance small chamfered `SCNBox`es
around the radius:
outward face gets the lit material, every groove-facing face gets the
near-black constant, stagger the phase per ring and twist alternate rings
about their radial axis for diagonal cuts. Get the solid-to-void ratio
from the reference before styling: on working surfaces the MATERIAL
dominates - blocks nearly touch and grooves are thin channels (a few
percent of the pitch). Equal-width blocks and gaps read as tiles floating
in black, and no texture detail rescues an inverted ratio. Two further
traps: (1) do NOT
`flattenedClone` the assembly - flattening scrambles material assignment on
multi-element geometries (a 6-material box, a 4-material tube) and the
whole band renders with the wrong faces lit; 150 nodes sharing two
geometry instances render fine unflattened. (2) box faces are per-face
full-texture, so design the tile as ONE block top, not a wrapping strip.

The technique has a second paired trap: a recess must render DARKER than the
surface around it, and SceneKit fights you twice. First, Fresnel: the flat
channel walls get viewed at grazing angles, where ANY dielectric goes
mirror-like and reflects the environment map as glowing rings (metalness 0
does not save you; near-full roughness dampens it). Second, and worse,
plain lambert: with no ambient occlusion, a channel wall that happens to
face the key light renders BRIGHTER than the curved surface beside it -
the recess reads inverted, instantly fake. Matte roughness alone cannot
fix the second one. Bake the occlusion instead: give every interior
surface of a recess (walls and floor) a near-black `.constant` material -
the one lighting model no light can brighten - and keep the real PBR only
on the outer working surface. SCNTube's material elements are [outer,
inner, top, bottom], so a rib ring is one line: `[crown, dark, dark,
dark]`.

## Satin metal on a dark stage

PBR metal multiplies the environment map by the diffuse color. On a dark
stage environment, `metalness 0.85` with a dark diffuse (0.31) renders nearly
black from most angles - the material has nothing bright to reflect. For the
satin alloy look keep diffuse mid-grey (~0.42) with metalness ~0.8 and
roughness ~0.36, and let the darkness come from the lighting, not the
albedo. Tilted faces (the concave dish) exaggerate this: they see even less
of the environment's bright bands, so err brighter than the photo suggests.

## Actors that arrive as files

Everything above builds performers from primitives, which is why this skill
rarely debugs asset importers. When a hero object does arrive as a file
(.scn, .dae, .abc - SceneKit's documented scene-source formats), gate it at
the stage door instead of patching it mid-performance: load with the
consistency-check option, then demand by name every node the choreography
will steer, and fail loudly if either check does not pass. A model whose
grip node is missing or whose hierarchy differs from the modeler's last
export will otherwise fail as a beat that silently animates nothing. Fix
rejects at the source asset, re-run the same gate, and only then let the
actor on stage.
