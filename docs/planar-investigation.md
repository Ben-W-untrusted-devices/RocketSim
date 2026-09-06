# Planar mode: the discharge investigation, and how it resolved

Working notes rather than documentation. This records a discrepancy that turned out not to
exist, three faults that did, and the method that told them apart, so neither the false
alarm nor the real fixes have to be re-derived.

## Conclusion first

There is no planar discharge fault. The apparent nine-point gap was an artefact of
comparing two different engines: the "axisymmetric bell" baseline was a shipped preset with
a contraction ratio of 9.8 and its own throat, exit and lengths, while the planar case was
a block of parameters written by hand with a contraction ratio of 3.1. Contraction ratio
drives discharge coefficient, so most of the gap was that, and the rest was everything else
that differed.

Held properly fixed, the two geometries agree.

## The controlled comparison

One parameter block, `planar` the only flag that moves. Because throat area goes as the
square of the radius when revolved and linearly with height when extruded, matching the
contraction ratio requires different chamber radii: 14.142 mm revolved against 25 mm
extruded both give 3.125. Everything else is identical, and so is the resulting mesh:
669 x 112 cells, dz 1349 um, dr 674 um, 27.9 cells across the throat.

Mass flow as a percentage of the one-dimensional choked value, time-averaged over eight
samples after the transient, plus or minus one standard deviation:

| | axisymmetric | planar |
|---|---|---|
| viscous | 84.85 +- 0.73 | 85.68 +- 0.00 |
| inviscid | 85.35 +- 0.85 | 85.83 +- 0.00 |

Planar is marginally *higher* in both rows, by less than the axisymmetric run's own scatter.
That is the direction the boundary-layer argument predicts, and the size of the viscous
debit matches it too: 0.50 points revolved against 0.15 planar, a ratio near the factor of
two or three expected from a full wetted perimeter against a single wall facing a symmetry
plane.

Contraction ratio, swept on that same grid with nothing else touched, accounts for the rest:

| axisymmetric, contraction 3.125 | 84.85 +- 0.73 |
| axisymmetric, contraction 9.77  | 90.74 +- 0.12 |

A planar run at contraction 9.77 reads 88.29 +- 0.39, but it is not a clean pair: matching
that contraction extruded needs a 78.2 mm half-height, which grows the domain and forces a
different radial extent and cell count. This is a real limitation of the comparison rather
than a result. The two geometries cannot hold both contraction ratio and body size fixed at
once, precisely because area is quadratic in one and linear in the other. The matched
3.125 pair is the comparison that isolates the solver, and it is clean.

## A separate finding, not a planar issue

The inviscid runs sit at about 85 % of choked, not near 100 %. The bulk of the shortfall is
numerical, from the resolution of the sonic line at this contraction ratio, and it is the
same in both geometries. It affects the axisymmetric solver equally and is a distinct
accuracy question from anything planar.

## What is verified about planar geometry

- Metric weights: every area and volume carries a weight of either the radius or one.
- Throat area is height times span, expansion ratio is the ratio of the two rather than its
  square, and the plug contour uses the planar Angelino construction, where mass
  conservation across a characteristic is a length rather than an area.
- Minimum passage over all spanning surfaces is 1.000 times design.
- Throat height comes out exactly as set, unlike the annular case where it is forced to
  A_t/(2 pi r).
- Free-stream preservation at the symmetry plane, after the fix below.

## Three faults found while chasing a discrepancy that was not there

1. **Symmetry-plane flux hard-zeroed.** `fluxR` returned zero flux at `ir == 0` with the
   comment that the axis has zero face area. True revolved, where anything written there is
   discarded. Extruded, the centreline is a symmetry plane with real area, so zeroing it
   left the first row of cells with pressure on one side and nothing on the other. Caught by
   a rest test with the engine off, uniform pressure, fluid at rest: the axisymmetric case
   held at 0.02 m/s, the planar case generated **231.55 m/s**, all of it at the centreline.
   Fixed by reflecting the state through the plane and letting the Riemann solver produce
   pure pressure and no transport, which is what a symmetry plane is and which costs nothing
   revolved. Planar rest test now reads 0.01 m/s.

   This was a serious correctness bug and it did not close the discharge gap, which is what
   eventually pointed at the comparison rather than the code.

2. **Axisymmetric-only viscous terms active in planar.** `muEff` formed the hoop strain
   `u_r / r` and fed its square into the Smagorinsky invariant; the radial momentum
   Laplacian carried `- u_r / r^2`; the dilatation carried `+ u_r / r`. None exist in a
   planar slice, and with fluid sitting on the symmetry plane rather than metal, they blow
   up in the middle of the jet. Now switched off when planar, with the four transverse
   Laplacians routed through a helper that carries the radius weighting only when revolved.
   Correct, and worth having, but it moved the number by nothing at all.

3. **Thrust integral not weighted by exhaust fraction** while the mass flux was, so a plug
   nozzle was credited with entrained air momentum, and near the base with air recirculating
   backwards through the plane. This one reached the axisymmetric solver too: effective
   exhaust velocity had been reading above ideal, which cannot happen. Bells are unaffected,
   their exhaust fraction being one across the exit.

Also fixed, and also not the cause: a **silent turn-radius clamp**. Turning a duct on a
radius under a couple of gap widths separates its inner wall; the code clamped to half a gap
width and said nothing. The configuration that triggered it was hand-written for testing,
taking the axisymmetric preset and flipping it to planar, which makes the throat four times
taller while leaving the converging length at 20 mm. No shipped preset hits it: the planar
XRS-2200 needs 164 mm and has 400. So the 60 % to 75 % improvement fixed a bad test setup,
not the simulator. The silent clamp was still a genuine error-reporting defect and is now
refused outright with the required length quoted.

## The methodological lesson

Twice in this investigation the anomaly was in the test harness, not the solver, and both
times the harness looked reasonable. What settled it was building a single parameter block
in which exactly one flag moves and then checking the resulting mesh dimensions matched
before trusting either number. Any planar-against-axisymmetric comparison should start by
printing the grid for both and refusing to proceed if they differ.
