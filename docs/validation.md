# Validation

Measurements taken with the shipped defaults unless stated otherwise. All timings are on an
Apple M1.

## Reference configuration

Kerolox (γ 1.24, M 22.2 g/mol, T_c 3570 K) at 20 bar, through a 25 mm chamber, an 8 mm throat
and a 14.75 mm bell exit (ε = 3.40), into Earth air at 101 kPa, still. The exit radius is the
optimum for this pressure ratio, so exit pressure comes out at 1.01× ambient.

Readings are taken at the exit plane at **t = 2 ms of flow time**. At 1.2 ms they are still
drifting; by 2 ms they are steady.

## 1-D reference performance

Propellant properties in, c\* and Isp out of the area–Mach relation. Nothing in the code is
fitted to these figures.

| propellant | γ | M (g/mol) | T_c (K) | c\* | vacuum Isp at ε 60 | published |
|---|---|---|---|---|---|---|
| Kerolox, RP-1/LOX | 1.24 | 22.2 | 3570 | 1763 m/s | 336 s | RD-180 338 s, Merlin Vac 348 s |
| Hydrolox, LH2/LOX | 1.22 | 12.0 | 3400 | 2353 m/s | 454 s | RS-25 452 s, RL10 465 s |
| Methalox, CH4/LOX | 1.20 | 20.5 | 3540 | 1849 m/s | 361 s | Raptor Vac ≈ 380 s |
| Hypergolic, N2O4/MMH | 1.23 | 21.5 | 3200 | 1701 m/s | 326 s | AJ10 ≈ 320 s |
| Solid, APCP | 1.18 | 27.0 | 3200 | 1540 m/s | 305 s | high-performance ≈ 300 s |
| Cold gas, N₂ | 1.40 | 28.0 | 290 | 429 m/s | 76 s | 60–80 s |

The model reads low for methalox and solids. These are frozen-flow figures: specific heats are
constant, so no dissociation occurs in the chamber and no recombination during expansion. Real
engines lie between the frozen and equilibrium limits.

## Choking

Chamber pressure swept at fixed geometry, against
`ṁ = p_c·A_t·√(γ/RT₀)·(2/(γ+1))^((γ+1)/2(γ−1))`:

| p_c / p_ambient | ideal ṁ | measured propellant ṁ | ratio | all gas crossing the plane |
|---|---|---|---|---|
| 3.0 | 34.2 g/s | 28.5 g/s | 0.83 | 38.9 g/s |
| 4.9 | 57.0 g/s | 52.2 g/s | 0.92 | 61.7 g/s |
| 9.9 | 114.0 g/s | 102.1 g/s | 0.90 | 110.2 g/s |
| 19.7 | 228.1 g/s | 207.7 g/s | 0.91 | 213.1 g/s |

Above about 5:1 the discharge coefficient is a steady 0.90–0.92 and mass flow scales linearly
with chamber pressure. Nothing in the setup imposes this; it follows from the plenum boundary
and the Riemann solver. The 8–10 % deficit is boundary-layer displacement at the throat.

The first row is not a solver error. At ε = 3.40 the nozzle requires about 20:1 to flow full.
At 3:1 it is heavily over-expanded, the jet separates and recirculates, and the exit plane is
not a valid station at which to measure throat mass flow: 36 % more gas crosses the plane than
the throat passes, the difference being entrained ambient air. Divergence between the last two
columns is the diagnostic for this condition.

## Grid convergence

Reference configuration at three radial resolutions:

| radial cells | cells across throat R | grid | ṁ vs ideal | exit-plane thrust | on-axis u_z | solver time for 2 ms |
|---|---|---|---|---|---|---|
| 64 | 7.3 | 257 × 64 | 80 % | 415 N (74 %) | 2628 m/s | 2.8 s |
| 112 (default) | 12.8 | 450 × 112 | 91 % | 478 N (85 %) | 2631 m/s | 17 s |
| 176 | 20.1 | 708 × 176 | 90 % | 482 N (86 %) | 2716 m/s | 88 s |

Monotone and converged to a few percent at the default. The controlling parameter is cells
across the throat radius rather than total cell count; the panel reports it and warns below six.
Plume shock-cell structure further downstream remains resolution-sensitive.

## Thrust deficit

At the reference configuration the measured exit-plane momentum-plus-pressure integral is 85 %
of 1-D ideal thrust (562 N ideal, 478 N measured). The deficit is boundary layer, non-uniform
exit profile and finite-rate startup, all of which 1-D theory excludes.

## Drag

Three nose shapes at Mach 2, otherwise identical, C_D referenced to frontal area:

| nose | drag | C_D | expected |
|---|---|---|---|
| flat | 261 N | 1.50 | 1.5–1.8 for a flat-faced cylinder; normal-shock stagnation pressure alone gives 1.65 |
| pointy (27° half-angle) | 94 N | 0.54 | cone wave drag plus base drag |
| rounded | 95 N | 0.55 | as pointy at this fineness ratio |

Velocity scaling gives an independent check. During an accelerating run, drag went from 10.8 N
at 144 m/s to 113 N at 447 m/s: a speed ratio of 3.1 against a drag ratio of 10.5, where v²
predicts 9.6.

Skin friction is excluded; see the note in the panel.

## Aerospike

The plug nozzle is a method-of-characteristics contour whose annular throat area is set equal to
pi*throat^2, so the 1-D reference figures are the same as for a bell with the same throat and
exit radii.

Geometry check, run against the built contour rather than against the design formulae. Walking
the cowl surface and taking, for each point, the smallest revolved area of a segment spanning to
the centrebody gives a minimum passage area of 615.8 mm^2 against a design throat area of
615.8 mm^2, on the slant from the spike root to the cowl lip. There is no unintended constriction
anywhere else in the passage, and the throat is where the construction puts it.

### Altitude response

Kerolox, throat 14 mm, exit 26 mm, epsilon 3.45, chamber 38 mm, into still Earth air at 101 kPa.
Bell and aerospike on an identical 846 x 192 grid, 7.8 cells across the annular gap, t = 2.2 ms.
Chamber pressure sets the expansion condition.

| p_c | p_e/p_a | nozzle | measured mdot | vs ideal | thrust | vs ideal | c = F/mdot | vs ideal |
|---|---|---|---|---|---|---|---|---|
| 5 bar | 0.25, over-expanded | bell | 162 g/s | 93 % | 274 N | 102 % | 1690 m/s | 110 % |
| 5 bar | 0.25, over-expanded | aerospike | 117 g/s | 67 % | 192 N | 72 % | 1639 m/s | 107 % |
| 20 bar | 0.99, matched | bell | 640 g/s | 92 % | 1521 N | 88 % | 2375 m/s | 96 % |
| 20 bar | 0.99, matched | aerospike | 520 g/s | 74 % | 1202 N | 70 % | 2313 m/s | 94 % |
| 60 bar | 2.96, under-expanded | bell | 1915 g/s | 91 % | 4983 N | 89 % | 2602 m/s | 98 % |
| 60 bar | 2.96, under-expanded | aerospike | 1523 g/s | 73 % | 3874 N | 69 % | 2544 m/s | 95 % |

Effective exhaust velocity, aerospike divided by bell: 0.970 at p_e/p_a 0.25, 0.974 at 0.99,
0.978 at 2.96. Flat to under one percent across a 12:1 range of chamber pressure.

No altitude compensation is visible at this expansion ratio, and the flatness of that ratio says
why: the bell is not losing anything for the plug nozzle to recover. Both readings above 100 %
in the over-expanded row are the 1-D reference being wrong rather than the solver being
optimistic. Fixed-geometry 1-D theory charges the full (p_e - p_a)*A_e debit over the whole exit
area, which assumes the nozzle flows full; the real over-expanded bell separates, and the
recirculating gas downstream of separation sits nearer ambient, so it pays less than the debit.
Separation is the bell's own compensation mechanism at modest epsilon. The configurations where
plug nozzles win are high expansion ratios, where separation moves far enough up the bell to be
destructive; the annular gap at those ratios is a small fraction of the throat radius and is not
resolvable on this grid.

### Discharge coefficient

The aerospike passes 73 % of ideal choked mass flow where the bell passes 91 %. Three
measurements localise it.

Station scan, propellant mass flux integrated across successive axial planes from just past the
throat to 90 mm downstream of the exit, at 1128 x 256:

| station | throat+1 | +5 | +15 | +30 | +45 | exit | +40 | +90 |
|---|---|---|---|---|---|---|---|---|
| mdot | 512 g/s | 513 | 506 | 506 | 508 | 517 | 492 | 499 |

Flat at 73 % from the throat outward, with nothing entering the radial sponge, so no mass is
lost in the plume. The throat itself passes 73 %.

Resolution, same configuration:

| radial cells | cells across gap | grid | mdot vs ideal | thrust vs ideal | c vs ideal |
|---|---|---|---|---|---|
| 128 | 5.2 | 564 x 128 | 66 % | 58 % | 89 % |
| 192 | 7.8 | 846 x 192 | 73 % | 73 % | 100 % |
| 256 | 10.3 | 1128 x 256 | 73 % | 71 % | 97 % |

Exhaust velocity converges. Mass flow does not; it plateaus at 73 %.

Viscosity, at 192 radial cells: 73 % viscous against 72 % with molecular viscosity and the
Smagorinsky constant both set to zero. The deficit is not boundary-layer displacement.

What remains is the wall representation. Solid cells are masked, not cut, so a wall is a
staircase. The throat here is inclined 48 degrees to an axis-aligned grid, which is the worst
case for a staircase, and the blockage is roughly one cell on each of the two walls bounding a
gap 10 cells wide. A bell throat, whose wall is nearly parallel to the axis at the same station,
does not pay it. The bias is numerical and it cancels out of F/mdot, which is why exhaust
velocity converges while mass flow does not. Rank plug-nozzle designs on effective exhaust
velocity.

### Integration bound

The plume has no wall, so the exit-plane integral is bounded by the exhaust tracer. Cumulative
thrust against integration radius, at epsilon 2.04:

| radius | 20 mm (lip) | 30 mm | 35 mm | 45 mm | 70 mm (domain) |
|---|---|---|---|---|---|
| cumulative thrust | 431 N | 921 N | 1195 N | 1213 N | 1176 N |

Flat beyond about 35 mm, varying 1.6 % out to the domain edge, so the integral is converged with
respect to where it stops.

## Two-species contact discontinuity

Conservative schemes for multi-component flow can produce spurious pressure oscillations at
material interfaces, because the energy equation is conservative while γ is not uniform.

Test: a contact between kerolox exhaust and air at uniform pressure and velocity, zero
viscosity, empty domain. The exact solution is pure advection.

| after | pressure spread | velocity spread |
|---|---|---|
| 200 steps | 0.56 % of p₀ | 1.2 % |
| 1000 steps | 0.37 % | 0.73 % |

Sub-percent, one-sided, and decaying as the interface smears. Against the pressure ratios this
tool operates at, 20:1 and above, the effect is not visible. A quasi-conservative scheme would
remove it at the cost of exact energy conservation.

## Inherited solver checks

The following were measured on the same kernels before the two-species change and are unaffected
by it.

- **Sod shock tube** against the exact Riemann solution, 601 cells: 0.17 % / 0.18 % / 0.09 % L1
  error in density, velocity and pressure, with zero overshoot at every resolution tested. Grid
  convergence 0.88 and 0.80 in L1, first order, which is correct for a solution containing
  discontinuities.
- **Speed of sound**: a 1 % Gaussian pressure pulse propagated at 344 m/s against a theoretical
  343.1, an error of 0.26 %. Total mass drifted by 4 × 10⁻⁴ %.
- **Choking through a sharp orifice**: mass flow plateaus above the critical pressure ratio at
  87 % of ideal, the discharge coefficient of a sharp-edged short tube. The raised-cosine
  contraction used here has no vena contracta, and its coefficient is correspondingly near 1.0.

## Performance

At the 30 ms default frame budget:

| quality | grid | cells across throat R | steps/frame | ms of flow per wall second | to steady state |
|---|---|---|---|---|---|
| Draft | 257 × 64 | 7.3 | 111 | 0.72 | ~3 s |
| Balanced (default) | 450 × 112 | 12.8 | 43 | 0.11 | ~19 s |

Two species cost roughly 3× the throughput of one: every primitive state requires the mass
fraction alongside the conserved variables, so the flux kernels do twice the memory traffic, and
the mixture rule is evaluated per cell. The `update` kernel passes its four neighbours into the
Smagorinsky term rather than re-reading them, recovering about 40 % of that.

Cost is close to linear in the radial cell count rather than quadratic: the timestep is set by
the cell size, so a finer grid costs both more cells and more steps.

## Shareable links

Encoding a fully non-default configuration, resetting everything, then decoding restores all 46
settings with zero mismatches. Worst-case length is 347 characters against a commonly cited safe
limit of 2000.

Values are clamped to their control's range on the way in and unknown keys are ignored, so a
hand-edited fragment such as `#tr=99999&re=-50&ra=abc&nt=77` loads as a valid configuration.
