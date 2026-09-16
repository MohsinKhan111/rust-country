# How it works

Notes on the parts that were interesting to build. Everything lives in
`rust-country.html` — roughly 6,000 lines, no dependencies.

## Constraints

One level. One self-contained file. No build step, no bundler, no external
assets of any kind — no images, audio, fonts or CDN. Nothing goes over the
network at runtime. 60fps on a weak machine is a requirement, not a goal.

Those rules drove nearly every decision below.

## The loop

Physics runs a fixed 60Hz step with an accumulator, decoupled from rendering.
A jump clears the same gap on a 144Hz monitor and a struggling laptop, and
identical input produces identical positions. Nothing moves by a raw frame
delta.

The accumulator is clamped, so a long stall (an alt-tab, a garbage collection
pause) advances one tick rather than fast-forwarding the world.

## Art

Every sprite, tile, prop and parallax layer is drawn procedurally in code at
boot and baked into offscreen canvases. Runtime drawing is blits only — never
per-frame shape drawing. Boot runs one stage per frame behind a progress bar,
because generating it all in a single synchronous block is a second of black
screen on a game whose first impression is responsiveness.

The 3D read comes from lighting, not detail. Each sprite is authored as a
height map, then shaded from two terms: a lambert for shape, and **absolute
height for depth**. The height term is what separates a recessed panel from a
raised one — a lambert alone cannot, because a flat region has no gradient, so
every flat surface resolves to the same tone. On top of that: a four-step
colour ramp, one light direction, a rim light and a contact shadow.

Left-facing frames are baked separately rather than flipped at draw time, so
the light stays on the same side of the world.

## Character

SCRAP is seven shapes and no more. Earlier versions had an antenna, a visor, a
chest badge, a body seam and tread lugs; at 22px tall that is not detail, it is
noise — the head merged into the body and the cyan optic, the whole focal
point, disappeared. Three tonal bands (dark treads, warm body, light head) with
a notched neck, so the silhouette has a step in it.

Movement is acceleration-based throughout, with a turnaround that decelerates
harder than it accelerates. That asymmetry is most of what makes the treads
feel like they have mass.

SCRAP falls under his own gravity constant, separate from the one governing
projectiles and enemies. Jump height is `v²/2g` and airtime is `2v/g`, so
raising jump velocity alone raises both — and the level's tightest gaps sit
about two pixels outside a running jump. Scaling velocity and gravity together
leaves horizontal reach untouched while the arc gets taller.

## Audio

All synthesised through Web Audio — oscillators, filtered noise, envelopes.
No audio files. Music is generated per beat and changes with the section.

## Testing

Movement and level geometry are measured by a headless Chrome harness that
drives the real game one physics tick at a time and searches every sensible
input timing. A result of "this gap cannot be crossed" means no timing crosses
it, not that one attempt failed.

This was not optional. A representative sample of bugs this project hit:

- A wall built across the only route with no doorway, making the level
  unfinishable from the second section onward.
- A boss that never spawned, because one comparison operator armed its trigger
  backwards — and every test had reached the boss by teleporting into its
  section, which starts the fight by hand and never consults the trigger.
- A weapon that missed its only valid target by eight pixels at every range,
  so the boss took zero damage across a two-minute automated fight.
- A one-tile crawl gap threaded through a wall, which an 18px character cannot
  fit through.
- Three "gaps" with the floor left in underneath: 112px pits you could fall
  into and never climb out of.
- A staircase of scenery that let you walk over the roof of two whole sections.

Every one of them rendered perfectly. None would have been caught by looking at
a screenshot, which is the entire argument for measuring instead.

## Save data

`localStorage` holds four things: best time, fewest deaths, most scrap, and the
mute flag. Reads and writes are wrapped in try/catch, because storage throws
outright in some private browsing modes and a save failure must never take the
game down with it.
