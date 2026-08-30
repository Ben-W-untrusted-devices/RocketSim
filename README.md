# Rocket Sim

An axisymmetric (r–z) **compressible Navier–Stokes** sandbox for a rocket's aft end: a
hot-gas injector feeding a combustion chamber, a parametric converging–diverging nozzle, and
the outside of the vehicle sitting in a freestream you choose. It runs entirely in the
browser on WebGPU — open `index.html`, no build step and no server needed.

Forked from [BenWheatley/Airzooka](https://github.com/BenWheatley/Airzooka), which asked the
same kind of question about a 3D-printed air cannon. The compressible solver, the WebGPU
plumbing, the video exporter and the shareable-link machinery are inherited from it; the
geometry, the boundary conditions, the measurements and the physics being asked about are
new.

**WebGPU is required.** Airzooka carried a WebGL2 fragment-shader fallback for its
*incompressible* solver. Nothing here is incompressible — the whole point is shocks, choking
and expansion — and a density-based scheme with an HLLC Riemann solver has no cheap
fragment-shader form. Rather than ship a second solver that answers a different question, the
fallback was deleted. Chrome 113+, Edge 113+, Safari 26+ and Firefox 141+ on Windows all
work; elsewhere Firefox may need `dom.webgpu.enabled`.

---

## What it models

Along the axis, nose to tail:

```
freestream in ──► nose cone ── forebody ── chamber ── converging ── throat ── diverging ──► plume
                  flat /                   ▲
                  pointy /                 │ injector plenum, held at
                  rounded                  │ stagnation p and T
```

| | |
|---|---|
| **Injector** | A band of cells at the head of the chamber pinned to a stagnation pressure and temperature, ramped in over an ignition rise time. The flux kernels treat it as ordinary fluid, so the nozzle **chokes on its own** rather than being told what mass flow to pass. |
| **Chamber** | Radius and length. Gas dynamics only. |
| **Nozzle** | Throat radius, exit radius, converging length, diverging length, and a **conical or bell** divergent contour. The contraction is a raised cosine so it meets both the chamber wall and the throat with zero slope. |
| **Nose cone** | Flat (blunt cylinder), pointy (conical), or rounded (ellipsoidal, zero slope at the shoulder). |
| **Environment** | Ambient **pressure** on a log slider from 0.01 to 125 kPa — sea level to about 65 km — ambient temperature, and ambient **air speed**, which is flight speed: in the vehicle frame it enters at the nose and washes down the body into the base region. |

The vehicle's outer radius is `max(chamber, nozzle exit) + wall thickness`, so the base is
always an annulus and there is always a base-flow region for the plume and the external flow
to fight over.

## What it does not model

Combustion, mixing, injector elements, multiple species, chemistry of any kind, radiation,
ablation, film cooling, nozzle flexure. The chamber is simply *held* at a stagnation state.

**One gas fills the whole domain.** γ and R are sliders, but they apply to the exhaust and to
the atmosphere alike. Air is γ = 1.4, R = 287; real rocket exhaust is nearer γ = 1.2,
R = 350. Setting those makes the plume expand correctly and makes the "air" outside stop
being air. Pick whichever end of the problem you are studying — and note that the default
Isp figures are low for the same reason: they are the Isp of hot *air*, not of combustion
products.

Axial symmetry forbids the three-dimensional instabilities that break a real shear layer up,
so the Smagorinsky term stands in for them. It is a stand-in, not a substitute.

---

## The solver

Conserved state `U = (ρ, ρu_z, ρu_r, E)` as cell averages on a finite-volume grid in
cylindrical coordinates.

- **HLLC** approximate Riemann solver, **MUSCL** reconstruction with a minmod limiter,
  **SSP-RK2** in time. First order at discontinuities, second elsewhere.
- **Thornber's low-Mach reconstruction fix.** Plain upwinding has dissipation that grows
  like 1/M. That matters more here than it did upstream: the external flow over the nose runs
  at Mach 0.2 while the plume next door is at Mach 4, in the same grid, and without the fix
  the slow side is dissipated away.
- **Reflecting-wall Riemann fluxes** rather than ghost cells — robust across the sharp
  throat and the nozzle lip.
- **Plenum boundary** at the injector: a Dirichlet region that participates normally in the
  flux computation. This is the standard reservoir boundary, and it is why the choking result
  below is a prediction rather than an input.
- **Freestream inlet** at z = 0 imposing (ρ, u_z, p) from the ambient sliders; zero-gradient
  outflow and far field; a sponge layer relaxing to the freestream at all three open
  boundaries so outgoing acoustics do not ring.
- Viscous stress, viscous heating and conduction at constant Prandtl number, plus optional
  Smagorinsky sub-grid viscosity.
- Timestep from a GPU max-reduction of the acoustic wave speed `(|u| + a)/dz + (|u| + a)/dr`.

Six compute dispatches per step (two RK stages × flux-z, flux-r, update), the whole frame
built into one command encoder and submitted once, per-stage uniforms addressed by dynamic
offset.

---

## What it predicts, and how well

Default configuration: 25 mm chamber, 8 mm throat, 13.5 mm exit (ε = 2.85), bell contour,
20 bar and 3000 K, at 100 kPa ambient, still air. Measured at the nozzle exit plane at
t = 1.2 ms, by which point the flow is steady.

### It chokes, and the mass flow saturates

Sweeping chamber pressure at fixed geometry, against
`ṁ = p_c·A_t·√(γ/RT₀)·(2/(γ+1))^((γ+1)/2(γ−1))`:

| p_c / p_ambient | ideal ṁ | measured ṁ | ratio |
|---|---|---|---|
| 3 | 44.5 g/s | 20.6 g/s | 0.46 |
| 5 | 74.2 g/s | 63.6 g/s | 0.86 |
| 10 | 148.4 g/s | 145.7 g/s | **0.98** |
| 20 | 296.7 g/s | 286.3 g/s | **0.97** |

Above about 10:1 the measured flow tracks the choked-throat formula to within 3 % and scales
linearly with chamber pressure, which is the signature of a choked throat: the nozzle has
stopped listening to the ambient.

The two low rows are **not** a solver error, and reading them as one is the trap. At ε = 2.85
the nozzle needs roughly 20:1 to flow full. At 3:1 it is grossly over-expanded, a shock
system sits inside the divergent section, the jet separates from the wall and recirculates —
so the *exit plane* is simply the wrong place to measure the *throat's* mass flow. The panel
reports both the tracer-weighted propellant flux and the total gas crossing the plane, and
the two diverge exactly when this is happening.

### It converges

Same case at three radial resolutions:

| radial cells | cells across throat R | grid | ṁ vs ideal | exit-plane thrust | on-axis u_z | wall time for 1.2 ms |
|---|---|---|---|---|---|---|
| 64 | 7.3 | 257 × 64 | 84 % | 414 N | 1934 m/s | 0.6 s |
| 112 (default) | 12.8 | 450 × 112 | 96 % | 477 N | 1938 m/s | 2.6 s |
| 176 | 20.1 | 708 × 176 | 98 % | 479 N | 1896 m/s | 13 s |

Monotone, and converged to a couple of percent by the default. This is worth stating plainly
because **the upstream Airzooka case did not converge** — its answer wandered by a factor of
two between resolutions, because the physics that mattered there was a chaotic shear layer
off a sharp orifice lip. Here the flow is dominated by a smooth accelerating nozzle, which is
a far better-posed problem. The plume's shock-cell structure downstream is still
resolution-sensitive; the throat and the exit plane are not.

The number that actually controls this is **cells across the throat radius**, not the total
cell count, and the panel shows it live and warns below six.

### It loses what a real nozzle loses

At the default, measured exit-plane momentum-plus-pressure is 86–87 % of 1-D ideal thrust
(552 N ideal, 477 N measured). The missing 13 % is boundary layer, non-uniform exit profile,
and the finite-rate startup — all things 1-D theory assumes away. Do not quote the absolute
number; a real nozzle of this size would also have wall heat transfer and a real gas.

### Inherited solver validation

These were measured on the upstream build of the same kernels and carry over unchanged:

- **Sod shock tube** against the exact Riemann solution, 601 cells: 0.17 % / 0.18 % / 0.09 %
  L1 error in density, velocity and pressure, with zero overshoot at every resolution
  tested. Grid convergence 0.88 and 0.80 in L1 — first order, which is correct for a solution
  containing discontinuities.
- **Speed of sound**: a 1 % Gaussian pressure pulse propagated at 344 m/s against a
  theoretical 343.1 — 0.26 % error, with total mass drifting by 4 × 10⁻⁴ %.
- **Choking through a sharp orifice**: mass flow plateaus above the critical pressure ratio
  at 87 % of ideal, which is the discharge coefficient of a sharp-edged short tube — a real
  vena-contracta effect. The smooth cosine contraction used here has no such contraction,
  which is why its coefficient is 0.97 rather than 0.87.

---

## Using it

**Enter** re-ignites, **space** pauses. `↻` marks parameters that rebuild the grid.

The panel carries two blocks of numbers, deliberately separated:

- **1-D ideal** — expansion ratio, exit Mach from the area–Mach relation, exit pressure and
  temperature, exit velocity, choked mass flow, thrust, C_F, c*, Isp, and the expansion ratio
  that would be *optimum* at the current ambient pressure. These are exact for what they
  assume: no separation, no boundary layer, uniform exit.
- **Measured** — what the grid actually did at the measurement plane.

Warnings fire on the things that matter: not choked, over-expanded past the Summerfield
separation criterion, under-expanded, a conical half-angle steep enough to cost real
momentum, an under-resolved throat, and a transonic freestream making the base region a
genuine interaction.

### Fields

| field | what to look at it for |
|---|---|
| exhaust fraction | where the propellant goes, and how much ambient air is entrained |
| speed \|u\| | the plume core |
| **Mach** | the sonic line in the throat, and where the plume goes supersonic |
| **schlieren** | shocks: bow shock, lip shocks, shock diamonds, Mach discs |
| pressure − ambient | over- and under-expansion, base pressure |
| temperature | recovery temperature on the nose, plume cooling |
| axial velocity u_z | recirculation and reversed flow |
| vorticity | shear layers — plume/freestream and the base region |

### Presets

Each one sets nozzle, chamber and atmosphere *together*, because the point is that a nozzle
is only ever right for one altitude. **Sea-level booster** and **Vacuum upper stage** are both
well-designed; **Over-expanded** is the vacuum bell fired at sea level and shows the
separation and the shock system moving inside the nozzle; **Under-expanded** is a stubby
nozzle at 60 bar and gives a clean train of shock diamonds; **Supersonic flight** puts a
pointy nose at Mach 2 at 10 km, where the bow shock, the shoulder expansion and the base
flow all show up together; **Cold gas thruster** removes combustion entirely.

### Things worth trying

- Switch the nose to **flat** at Mach 2 and watch a detached bow shock stand off the face,
  with a subsonic pocket behind it. Then switch to **pointy** and watch it attach.
- Take the vacuum nozzle down to sea level and watch the shock system walk *into* the bell.
- Set γ = 1.2 and R = 350 and see how much further the plume expands.
- Move the measurement plane downstream and watch the thrust integral stop meaning thrust.

---

## Video export

Exports MP4/H.264 by default, or WebM/VP9 or VP8. WebCodecs supplies encoded chunks but no
container, so both muxers are written here — a minimal Matroska writer and a minimal MP4
writer (ftyp / mdat / moov).

The interactive solver chooses its timestep adaptively, which is right on screen and wrong
for video: frames would represent unequal slices of time and the motion would be subtly,
invisibly wrong. **Export runs on a fixed schedule** — every video frame advances exactly the
same simulated interval, subdivided into as many equal sub-steps as stability requires. The
sub-step count varies between frames; the frame interval never does.

**Export all fields** writes one file per field from a *single* simulation. Stepping is the
expensive part and rendering is nearly free, so eight fields cost far less than eight runs.

**The colour scale is fixed for the whole clip.** On screen the range tracks the flow, which
is what you want while exploring; in a video it means a colour does not signify the same
thing from one frame to the next. Export therefore runs the shot once to find the peak, fixes
the range, then records — and caches the peak per configuration so a repeat export skips it.

Frames carry a burned-in clock, the configuration, the colour legend and a physical scale
bar, in bands above and below the image rather than on top of the flow.

---

## Shareable links

Every physical setting is serialised into the URL fragment, so a link reproduces a
configuration exactly. Only non-defaults are written and the keys are two characters, so a
typical link carries a handful of them; with all 35 parameters off their defaults it comes to
269 characters, against a commonly-cited safe limit of 2000.

The fragment is used rather than the query string because it never reaches a server, and
because assigning `location.hash` works on `file://` URLs where Chrome throws on
`history.replaceState`.

Links are validated on the way in — values clamped to their control's range, unknown keys and
unparseable numbers ignored — so a hand-edited `#tr=99999&er=-50&rs=abc&nt=77` loads as a
valid configuration rather than breaking. Round-tripping is tested: encoding a fully
non-default configuration, resetting everything, then decoding restores all 35 settings with
zero mismatches.

Back and forward restore the configuration *and* restart the run — otherwise you would be
watching one nozzle's flow inside another nozzle's geometry. Dragging a slider replaces the
current history entry; a discrete action pushes one.

---

## Performance

Measured on an Apple M1. The default 450 × 112 grid runs about **0.46 ms of flight per second
of wall clock**, so the roughly 1.2 ms it takes to reach steady state arrives in a few
seconds. Cost is close to linear in the radial cell count rather than quadratic: the timestep
is set by the cell size, so a finer grid pays twice, once in cells and once in steps.

The dominant lever is the **radial domain** slider, which trades far-field room against
resolution at fixed cell count — widen it for a bow shock that needs space, narrow it to put
cells where the throat is.

Two structural choices from upstream still carry the performance:

- **One command encoder per frame, one submit.** A WebGPU dispatch costs a few microseconds;
  a submit costs far more. Every stage of every step of a frame is planned on the CPU first,
  then queued together.
- **Axial cell stretching.** The plume is long and thin and the timestep is set by the axial
  wave speed, so axial cells can be about twice the radial size for free. Drop the stretch to
  1 when shock-cell spacing is what you are measuring.

---

## What to trust

**Structure.** Where the sonic line sits. Whether the flow separates inside the bell, and how
far up it. Shock-diamond spacing and Mach discs in an under-expanded plume. Whether a bow
shock is attached or detached. How the base flow and the plume interact. These are robust and
reproduce across resolutions.

**The 1-D numbers.** Exact for what they assume, and clearly labelled as such.

**Not the absolute measured figures.** They are converged to a few percent on the default
grid for the throat and the exit plane, which is good enough to rank designs and not good
enough to quote a thrust. And they are the thrust of a nozzle flowing hot air, not
combustion products.
