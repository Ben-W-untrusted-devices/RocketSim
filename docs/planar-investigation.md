# Planar mode: open investigation

Working notes. Not documentation; this records where the planar (linear-aerospike)
path stands so the thread can be picked up without re-deriving it.

## What is verified

- Metric weights: every area and volume carries a weight of either the radius or one.
- Throat area is height times span, expansion ratio is the ratio of the two rather than
  its square, and the plug contour uses the planar Angelino construction where mass
  conservation across a characteristic is a length rather than an area.
- Minimum passage over all spanning surfaces is 1.000 times design.
- Throat height comes out exactly as set, unlike the annular case where it is forced to
  A_t/(2 pi r).
- The axisymmetric path is unchanged: every check still passes.

## The discrepancy

| case | planar | axisymmetric |
|---|---|---|
| bell, eps 3.4 | 88 % of choked | 94.7 % |
| plug, eps 3.45 | 68 to 75 % | 79 % |

Total pressure is flat at 18.9 bar from the injector through to Mach 1.4, so the duct is
clean and there is no upstream loss. Mass flow reads the same at the throat as downstream,
so it is not the plume or the measurement bound. The flux profile across the throat shows
deficient layers on both walls.

The direction is the anomaly. An axisymmetric throat has wall all the way round it, so its
area deficit goes as 2 delta*/R. A planar half-slot has one wall and a symmetry plane, so
its deficit should go as delta*/h, which is *less*. Planar reading worse than axisymmetric
is backwards from that argument, which is why this looks like a fault rather than a
difference between two genuinely different nozzles.

## Ruled out

- Upstream stagnation pressure loss: flat to Mach 1.4.
- Plume clipping or the tracer-bounded measurement radius: same figure at the throat.
- The 1-D reference: A_t = height times span checks out against an independent calculation.
- Geometry: minimum passage is 1.000 times design.
- The turn radius clamp (see below): fixing it moved 60 % to 75 %, and 75 % is still short
  of 79 %.

## Leading suspect

The viscous terms carry axisymmetric-only pieces that were never switched:

- `muEff` forms the hoop strain `stt = u_r / r` and feeds `stt^2` into the Smagorinsky
  strain invariant.
- The radial momentum Laplacian carries `- u_r / r^2`.
- The dilatation carries `+ u_r / r`.

None of these exist in a planar slice. Worse, in planar the symmetry plane is at y = 0 and
the flow fills the region right down to it, so `r -> 0` inside the nozzle core rather than
inside solid metal. Those terms then blow up exactly where the flow is, producing spurious
eddy viscosity and a spurious transverse momentum source through the middle of the jet.

In the axisymmetric case the equivalent region is the axis, which for a bell is genuinely
the centre of the flow too, but the terms are correct there.

## Two faults already found and fixed while chasing this

1. **Silent turn clamp.** Turning a duct on a radius under a couple of gap widths separates
   its inner wall. The converging section holds the turn, so a taller throat needs more of
   it. The code clamped the radius to half a gap width and said nothing.
   **Caveat on the headline number:** the configuration that triggered this was one written
   by hand for testing, taking the axisymmetric preset and switching it to planar, which
   makes the throat four times taller while leaving the converging length at 20 mm. No real
   engine hit it: the planar XRS-2200 needs 164 mm and has 400. So the 60 % to 75 % gain
   was fixing a bad test setup, not a simulator fault, and it does **not** account for the
   remaining discrepancy. The silent clamp is still a genuine defect in error reporting and
   is now refused outright with the required length quoted.

2. **Thrust integral not weighted by exhaust fraction** while the mass flux was, so a plug
   nozzle was credited with entrained air momentum, and near the base with air recirculating
   backwards through the plane. This affected the axisymmetric solver too: effective exhaust
   velocity had been reading above ideal, which cannot happen. Bells are unaffected because
   their exhaust fraction is one across the exit.

## Next steps

1. Switch off the hoop terms in the viscous block and `muEff` when planar, and re-measure
   the planar bell. If it moves to about 95 % that is the answer.
2. If not, run the planar bell inviscid. Viscosity being irrelevant would rule the whole
   block out and point at the flux or boundary treatment instead.
3. A planar resolution sweep, run with enough budget to actually settle; the earlier attempt
   at 224 radial cells hit the time limit mid-transient and told us nothing.
