# Validation

Measurements taken with the shipped defaults unless stated otherwise. All timings are on an
Apple M1.

Most of what follows is re-runnable rather than transcribed. The **Validation** panel in the
app runs 20 checks: published performance for the five hardware presets, the plug-nozzle
contour against its design throat area across the expansion range, two one-dimensional
identities, and six steady-state solver checks. Fast checks return at once; the full set takes
about three minutes. `await validate('fast')` and `await validate('all')` do the same from the
console and return the rows. Bands are one-sided where the quantity is an ideal figure real
hardware cannot reach.

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

## Cut cells

Solid cells are masked rather than cut, so a wall that crosses a cell at an angle used to be
rounded to whole cells. Each cell now also carries the fraction of its volume that is fluid and
the open fraction of each of its faces, sampled 8 x 8 from the same `solidAt` that defines every
other shape in the tool. Face fluxes are weighted by the open fraction, and the force the wall
exerts follows from the aperture imbalance, which carries the true surface normal without that
normal ever being formed: a uniform pressure over a closed surface exerts no net force, so the
imbalance fixes the wall term exactly. With every aperture open the term reduces to `p/r`, the
usual axisymmetric source, so an uncut cell is untouched.

The fluid fraction is carried but not used to divide the flux sum. Dividing by it is the
textbook cut cell and is conservative, but a sliver cell then sets the timestep for the whole
grid: at a fluid fraction of 0.15 the effective CFL is nearly seven, and the solver diverges in
under 10 microseconds. Using the full cell volume leaves a local O(h) smearing at the wall, the
same order the scheme already carries there.

No change to the cases that were already converged: the reference bell reads 90 % of ideal mass
flow against 91 % before, on-axis exhaust velocity 2636 m/s against 2631, and flat-nose drag at
Mach 2 is 257 N (C_D 1.48) against 261 N (C_D 1.50).

## Body-fitted nozzle grid

A uniform grid has to resolve a throat a millimetre across inside a domain a metre long, and
pays for that spacing everywhere. The radial coordinate is now remapped per axial station so
that a fixed share of the radial cells lies between the two walls of the nozzle passage, and
grid lines run along the walls instead of cutting across them. Away from the nozzle the mapping
relaxes to the uniform grid, so the plume and the external flow are unaffected.

The mapping is stored on cell corners, which keeps neighbouring cells sharing a face exactly.
Axial faces stay perpendicular to the axis and only their area changes; radial faces follow the
mapping and carry a tilted normal, so the Riemann problem there is solved in the face frame and
the flux rotated back. Cell volumes come from the revolved quadrilateral. The wall-pressure and
hoop terms are written so that a uniform pressure produces exactly zero net force, which holds
on the mapped grid because the face normals of a closed contour sum to zero and their radial
components sum to the planar area.

Same configurations, body fitting off and on:

| case | grid | cells | cells across throat | wall time | mdot vs ideal | c vs ideal |
|---|---|---|---|---|---|---|
| bell, uniform | 450 x 112 | 50 k | 12.8 | 8 s | 89.8 % | 93 % |
| bell, fitted | 450 x 112 | 50 k | 30.0 | 11 s | **94.7 %** | 93 % |
| aerospike, uniform | 1020 x 256 | 261 k | 15.2 | ~180 s | 73.6 % | 96 % |
| aerospike, fitted | 510 x 128 | 65 k | 19.2 | 29 s | **84.0 %** | **98.2 %** |
| aerospike, uniform, matched cost | 510 x 128 | 65 k | 7.6 | 29 s | 58.4 % | 89.1 % |

The bell gains five points of discharge on the same cell count for 1.4 times the run time, the
extra cost being the smaller timestep the finer throat cells impose. The aerospike is the larger
result: better than the uniform grid gave at four times the cell count and six times the run
time, and at matched cost the comparison is 84.0 % against 58.4 %.

Refinement still works through the throat rather than only the plume, because the share given to
the passage has a floor rather than being a fixed cell count: at radial resolutions of 112, 128,
192 and 256 the plug nozzle gets 17.9, 19.2, 22.1 and 29.4 cells across its annular gap.

The case this was built for is the XRS-2200 preset, where the annular gap is a small fraction of
the throat radius. A uniform grid puts 0.7 cells across it at 128 radial cells and 1.0 at 192;
the mapping gives 4.9 and 8.0 at the same cost. That is still short of the 25 the discharge
needs, and the geometry there is extreme, with the throat inclined 97 degrees, but it moves the
case from unrepresentable to marginal.

What the mapping does not change: flat-nose drag at Mach 2 reads 1.48 either way, and a uniform
ambient still integrates to zero net force, so free-stream preservation survives the tilted
faces.

## Aerospike

The plug nozzle is a method-of-characteristics contour whose annular throat area is set equal to
pi*throat^2, so the 1-D reference figures are the same as for a bell with the same throat and
exit radii.

### Geometry

The throat is inclined at the Prandtl-Meyer angle of the exit Mach number, which is 48 degrees
at epsilon 3.45, so the flow has to be turned from axial before it gets there. The turn is a
pair of circular arcs about a common centre, one for the centrebody and one for the cowl, three
gap widths in radius. Concentric arcs keep the gap constant through the turn, which makes the
passage area fall monotonically to exactly the throat area and no further. Upstream of the turn
the centrebody is a cylinder and the cowl carries the whole contraction on a curve that is flat
at both ends, so the inner wall has no curvature for the flow to separate off.

Checked against the built geometry rather than the design formulae: walking the cowl surface and
taking, for each point, the smallest revolved area of a segment spanning to the centrebody gives
a minimum passage area equal to the design throat area to three figures, on the slant from the
spike root to the cowl lip, at every expansion ratio from 1.07 to 8.2.

The first version of this used cubic Hermite curves for both walls, with zero slope at the
chamber and the throat angle at the root. Over a convergent section long compared with the
radius change, that curve overshoots: the centrebody bulged outward and reached 56 degrees in
mid-section before flattening to the design angle, and the flow separated off it. Measured mass
flow at epsilon 1.31 was 75 % of ideal with the Hermite contour and is 89 % with the arcs.

### Discharge

Kerolox at 20 bar into still Earth air, throat 14 mm, 192 radial cells, t = 2.2 ms. Expansion
ratio sets both the throat angle and the width of the annular gap.

| epsilon | throat angle | gap | cells across gap | mdot vs ideal | c vs ideal |
|---|---|---|---|---|---|
| 1.07 | 6.7 deg | 10.6 mm | 30.9 | 91 % | 99 % |
| 1.31 | 17.3 deg | 8.1 mm | 23.5 | 89 % | 99 % |
| 1.99 | 32.9 deg | 5.6 mm | 16.4 | 81 % | 96 % |
| 3.45 | 48.0 deg | 4.0 mm | 11.4 | 69 % | 93 % |
| 3.45, 320 cells | 48.0 deg | 4.0 mm | 18.9 | 74 % | 102 % |

At a wide gap the plug nozzle matches the bell exactly: 91 % is what the bell reads at the same
chamber conditions. The deficit appears as the gap narrows, and it is mass flow only. Effective
exhaust velocity stays within a few percent of ideal throughout, which is what makes the
altitude comparison below usable.

Four things it is not. It is not the geometry: the minimum passage area is the design throat
area to three figures. It is not stagnation pressure loss upstream: following the peak-Mach
streamline, total pressure holds at 18.8 to 19.1 bar from the injector all the way to M = 1.06,
against 19.0 bar in the chamber. It is not viscous: with molecular viscosity and the Smagorinsky
constant both set to zero the reading moves by one point. It is not the measurement: integrating
the propellant mass flux across successive axial planes gives 76 % at the throat itself and 74 %
at the exit plane 60 mm downstream, with nothing entering the radial sponge.

What is left is the wall treatment inside a narrow slot. The flux profile across the throat shows
a healthy sonic core with low-momentum layers on both walls, and those layers are a roughly fixed
number of cells thick, so they cost a share of the gap that grows as the gap narrows. Turn radius
confirms it: at epsilon 3.45 a radius of 1.5 gap widths reads 72 %, 3 reads 70 % and 6 reads
62 %, which tracks the length of narrow annulus the flow has to traverse rather than the
sharpness of the turn. Three gap widths is kept, being the usual floor for turning a duct without
separating the inner wall.

The panel reports cells across the gap and asks for 25.

### Altitude response

Bell and aerospike, same throat area and expansion ratio 3.45, identical grid and domain,
t = 2.2 ms. Chamber pressure sets the expansion condition.

| p_c | p_e/p_a | nozzle | mdot | vs ideal | thrust | c = F/mdot | vs ideal |
|---|---|---|---|---|---|---|---|
| 5 bar | 0.25, over-expanded | bell | 160 g/s | 92 % | 274 N | 1710 m/s | 111 % |
| 5 bar | 0.25, over-expanded | aerospike | 131 g/s | 75 % | 222 N | 1695 m/s | 110 % |
| 20 bar | 0.99, matched | bell | 639 g/s | 92 % | 1517 N | 2373 m/s | 96 % |
| 20 bar | 0.99, matched | aerospike | 466 g/s | 67 % | 1057 N | 2266 m/s | 92 % |
| 60 bar | 2.96, under-expanded | bell | 1917 g/s | 91 % | 4976 N | 2596 m/s | 97 % |
| 60 bar | 2.96, under-expanded | aerospike | 1411 g/s | 67 % | 3570 N | 2531 m/s | 95 % |

Effective exhaust velocity, aerospike divided by bell: 0.991 over-expanded, 0.955 matched, 0.975
under-expanded. The plug nozzle is closest to the bell at the over-expanded end, which is the
direction compensation would push, but the margin is a few percent across a 12:1 range of
chamber pressure.

No useful altitude compensation at this expansion ratio, and the reason is visible in the
over-expanded row: both nozzles read above 100 % of the 1-D figure there. Fixed-geometry theory
charges the full (p_e - p_a)*A_e debit over the whole exit area, which assumes the nozzle flows
full; the real over-expanded bell separates, and the recirculating gas downstream of separation
sits nearer ambient, so it pays less than the debit. Separation is the bell's own compensation
mechanism at modest epsilon, and there is little for a plug nozzle to recover. Plug nozzles win
at high expansion ratios, where separation moves far enough up the bell to be destructive; the
annular gap at those ratios is a small fraction of the throat radius and is not resolvable on
this grid.

### When a plug nozzle is worth it

A plug nozzle only earns anything where a bell would be forced to separate, so the question is
when a bell's usable expansion ratio is small. Sweeping chamber pressure, taking the Summerfield
criterion p_e/p_a = 0.35 as the bell's limit at sea level, and comparing against a plug carrying
a vacuum-sized ratio of 150:

| p_c | largest epsilon a bell can carry | mean Isp gain from the plug |
|---|---|---|
| 20 bar | 7 | +40 s |
| 50 bar | 15 | +29 s |
| 100 bar | 25 | +22 s |
| 200 bar | 43 | +16 s |
| 300 bar | 59 | +13 s |

High chamber pressure does the plug nozzle's job for it. At 300 bar a bell already reaches
within 14 s of the vacuum optimum, which does not pay for a centrebody that has to be cooled and
a base that has to be filled. At 20 bar the gap is 40 s and the argument changes. This is a
large part of why aerospikes have not flown: staged combustion arrived first and took most of
the prize.

The second case has nothing to do with altitude. Throttling drops chamber pressure, which
over-expands a fixed bell exactly as climbing under-expands it. An epsilon 25 bell at sea level:

| throttle | 100 % | 60 % | 40 % | 30 % | 20 % |
|---|---|---|---|---|---|
| p_e/p_a | 0.35 | 0.21 | 0.14 | 0.10 | 0.07 |
| 1-D Isp | 377 s | 337 s | 286 s | 235 s | 134 s |

Every one of those is separated, with the side loads that go with it. A plug holds its exit
condition because the plume boundary is ambient pressure rather than a wall. Landing burns and
boostback are this regime.

So the configurations worth simulating at high expansion ratio are low to moderate chamber
pressure engines that fly from sea level to vacuum on one nozzle, and anything that throttles
deeply. Upper stages are not among them: in vacuum there is nothing to compensate and a bell is
lighter. Neither is Mars ascent, where ambient is already negligible, nor Venus, where the
optimum ratio is 1.1.

Four things at high expansion ratio are outside what one-dimensional theory can supply, which is
what makes the simulation worth its cost there. The plume's outer boundary is set by ambient
pressure and is the compensating mechanism itself. Real spikes are truncated to 20 to 40 % of
the contour, and at high ratios the spike is slender enough that the base is a large fraction of
the exit area, with a recirculation pressure that adds to or subtracts from thrust. Below the
design altitude the flow over the spike carries recompression shocks, and the surface pressure
distribution under them is the thrust. And a descent burn puts the plume against a supersonic
freestream with ambient rising through the burn.

These are ideal one-dimensional figures. Summerfield is an approximate criterion, and the plug
is credited at sea level with the performance of a matched nozzle, which is optimistic by
perhaps 5 to 10 %.

### Integration bound

The plume has no wall, so the exit-plane integral is bounded by the exhaust tracer. Cumulative
thrust against integration radius, at epsilon 2.04:

| radius | 20 mm (lip) | 30 mm | 35 mm | 45 mm | 70 mm (domain) |
|---|---|---|---|---|---|
| cumulative thrust | 431 N | 921 N | 1195 N | 1213 N | 1176 N |

Flat beyond about 35 mm, varying 1.6 % out to the domain edge, so the integral is converged with
respect to where it stops.

## Comparison with flown hardware

Four engines whose performance is published, set up from their own chamber conditions and area
ratio. Nothing is fitted: propellant gamma, molecular weight and flame temperature go in, and
specific impulse comes out of the area-Mach relation.

| engine | epsilon | Isp sea level | published | Isp vacuum | published |
|---|---|---|---|---|---|
| V-2 (A-4), LOX/ethanol, 15 bar | 3.3 | 216 s | 203 s | 252 s | 239 s |
| F-1, LOX/RP-1, 70 bar | 16 | 276 s | 263 s | 318 s | 304 s |
| RS-25, LOX/LH2, 206 bar | 69 | 375 s | 366 s | 456 s | 452 s |
| XRS-2200 aerospike, LOX/LH2, 58 bar | 58 | 210 s | 339 s | 454 s | 439 s |

The first three read high by 6.4 %, 4.9 % and 2.5 % at sea level and 5.4 %, 4.6 % and 0.9 % in
vacuum. That is the right sign and the right size: these are ideal figures, and a real engine
delivers the product of its combustion efficiency and its nozzle efficiency, usually 94 to 97 %
for a large chamber and better for a high-pressure one. Applying 95 % to the V-2 gives 205 s
against 203 measured, and 95.5 % to the F-1 gives 264 s against 263.

The XRS-2200 row is the one that does not agree, and it is the aerospike point. At sea level a
bell of area ratio 58 has an exit pressure 0.07 times ambient, and one-dimensional theory,
which assumes the nozzle flows full, charges the whole (p_e - p_a)*A_e debit and returns 210 s.
The real aerospike delivers 339 s because its plume boundary is ambient pressure rather than a
wall, so it does not pay that debit. The 129 s gap is altitude compensation, and it is the
reason plug nozzles are built. In vacuum, where there is nothing to compensate, the model and
the engine agree to 3 %.

That case also marks the edge of what this tool can resolve. Set up as an aerospike rather than
a bell, an area ratio of 58 puts the throat angle at 89 degrees and the annular gap at 1.2 mm on
a 14 mm throat, which is 2.3 cells at the highest resolution the browser will run. Resolving it
to the 25 cells the discharge needs would take a radial cell size of 0.03 mm across a 180 mm
domain, so of order 10^8 cells.

### V-2 at full scale

Modelled at its real dimensions: 203 mm throat, 370 mm exit, 15 bar, LOX/ethanol at 2973 K.

| quantity | model | published |
|---|---|---|
| mass flow, 1-D | 123 kg/s | 123 kg/s (55 alcohol + 68 LOX) |
| thrust, 1-D | 261 kN | 245 kN |
| Isp, 1-D | 216 s | 203 s |
| mass flow, measured | 91 kg/s | |
| thrust, measured | 167 kN | |
| Isp, measured | 188 s | 203 s |

The 1-D mass flow lands on the published figure exactly, which is an independent check on the
chosen gamma, molecular weight and flame temperature, since it comes from c\* and the throat
area alone. The measured block shows the usual pattern: mass flow and thrust each read low, by
27 % and 36 %, while Isp reads only 7 % low because the two deficits largely cancel in F/mdot.
At 23 cells across the throat radius but only 12 axially, this nozzle is less well resolved
along the axis than the reference configuration, and the deficit is correspondingly larger.

### V-2 trajectory

The sounding-rocket configuration, fired vertically: 540 kg of instruments in place of the
1000 kg warhead, 12,040 kg on the pad. Published V-2 vertical flights from White Sands reached
109 km, 121 km typical, and 134 km on the Albert II mission.

Propellant mass needs care. The quoted tank capacities total 9,726 kg, but 123 kg/s over the
published 65 s burn is 8,000 kg, and the larger figure gives a burn time of 79 s and a burnout
speed well above the 1,600 km/h the vehicle is recorded as reaching. The 8,000 kg consumed
figure is used here; it reproduces the 65 s burn and the burnout speed together.

| quantity | model | published |
|---|---|---|
| mass on the pad | 12,017 kg | 12,040 kg |
| burn time, 1-D | 64 s | 65 s |
| burnout speed | 1,439 m/s | 1,600 m/s peak, on the flatter operational trajectory |

Apogee depends on how hard the trajectory is fast-forwarded, because the flow field has to keep
up with an ambient pressure that is falling underneath it:

| fast-forward | flight seconds per ms of flow | apogee |
|---|---|---|
| 10,000x | 10 | 189 km |
| 3,162x | 3.2 | 157 km |
| 1,000x | 1.0 | 148 km |

Converging downward towards the observed band from above, and still 10 to 35 % high at the
lowest fast-forward that finishes in reasonable time. Two contributions: the measured mass flow
is low, which stretches the burn from 65 s to 90 s and buys extra impulse, and the residual
fast-forward error. The panel flags the quasi-steady violation in all three of these runs.

### Traveler IV

USC Rocket Propulsion Laboratory, April 2019, the first entirely student-built rocket to pass
the Karman line. Published: 103.6 km apogee with a stated uncertainty of 5.0 km, 8 inch
airframe, a solid motor of 42,000 lbf-s total impulse burning for 13 s with a peak thrust of
21.6 kN, top speed 1,515 m/s, apogee at T+151 s. Masses are not published and are inferred here
from the total impulse and the reported 17 g peak: 150 kg on the pad, 83 kg of propellant.

| quantity | model | published |
|---|---|---|
| burn time | 13 s | 13 s |
| burnout speed | 1,551 m/s | 1,515 m/s top speed |
| apogee | 118 km | 103.6 +/- 5.0 km |
| time to apogee | 165 s | 151 s |
| peak drag | 2,994 N at Mach 3.3, 8.2 km | not published |

Burnout speed is within 2.4 %. Apogee is 14 % high, of which roughly half is the fast-forward
error measured on the V-2 above and the rest is the inferred mass and the drag model, which
excludes skin friction. USCRPL's own six-degree-of-freedom reconstruction needed a 20 % drag
penalty to match their flight data, so a model that omits skin friction reading high on apogee
is the expected direction.

### What these cases exercise

The 1-D block is validated to a few percent against flown hardware across a 14:1 range of
chamber pressure and a 20:1 range of area ratio. The trajectory integrator is validated to
within about 15 % on apogee, with the residual attributable to two known and separately measured
effects. The measured CFD block is the weakest link: absolute mass flow and thrust are
resolution-limited, and only their ratio is reliable.

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
