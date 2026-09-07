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

## Postscript: the J-2T-250K read zero, and why

Running the full suite after the fixes above left one check failing: "J-2T-250K: the engine is
not sealed", reading exactly 0 % of choked. It fails identically on the commit before these
changes, so it is not a regression from them, but it was a real failure and worth chasing.

It is not a sealed duct. The geometry is right (minimum passage 1.0034 times design, apertures
open at every station) and the engine fires: field Mach 2.6 at sea level, 5.8 in vacuum, with
121 kg/s of exhaust crossing the throat against a 258 kg/s ideal.

The fault is where the mass flow was being measured. Mass flow is integrated on the probe
plane, which sits at `zExit`. For a truncated plug `zExit` is exactly the spike base:
zRoot + spikeBuilt = 7961 + 1624 = 9585 for the XRS-2200, and 9330 + 2046 = 11376 for the
J-2T-250K, both equal to their `zExit` to the millimetre. That plane cuts the base
recirculation, and exhaust arrives there by entrainment rather than with the jet. Tracking the
tracer front on the XRS-2200 gives 8825 mm at t = 2.10 s, 9061 at 2.69, 9226 at 3.13, 9297 at
3.38, 9367 at 3.79, 9462 at 4.56: a few hundred millimetres per second, decelerating, against
a jet moving at kilometres per second. Until that front crosses the plane the integral reads
exactly zero; when it crosses, at t = 4.94 s, the measurement radius jumps from 18 mm to
1657 mm and the reading jumps from 0 to 10.6 % in one step.

So the check was timing out rather than measuring. The XRS-2200 crossed with seconds to spare
and passed at a marginal 10.6 %. The J-2T-250K is the largest preset, needs 2085 mm of front
travel instead of 1660, and runs at about 39 seconds of wall clock per second of flow time; it
was still short of the plane when the 150-second budget expired, and reported the zero that
means a sealed duct.

The check asks whether propellant is leaving the engine at all, so it now asks at the throat:
the probe plane is moved to 15 % of the way from the throat to the exit whenever the nozzle is
a plug. A bell's exit plane is a real exit and is left alone. On the same stalled J-2T state
that read 0 %, moving the plane from 11376 mm to 9604 mm changed the reading to 47.2 %, which
matches the 121/258 measured directly off the buffers. Both plug presets now pass with margin
rather than by seconds: XRS-2200 58.5 %, J-2T-250K 52.5 %.

The lesson is the same one as above. A measurement that reads exactly zero deserves to be
checked against the raw field before it is believed, in either direction: here it was the
instrument, not the engine.

## Suite state

All 33 checks pass on the current code. The reference bell reads 94.69 % of choked, unchanged
from before the symmetry-plane fix, so the revolved path is untouched by it; uniform ambient
still integrates to 0.00 N of drag; the aerospike reads 78.52 % mass flow and 96.11 % exhaust
velocity; the two plug seal checks now read 58.5 % and 52.5 % instead of 10.6 % and 0 %.
