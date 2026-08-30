# Rocket Sim

An interactive, axisymmetric compressible-flow sandbox for a rocket's aft end. A hot-gas
injector feeds a combustion chamber and a converging–diverging nozzle you can reshape with
sliders, while the vehicle flies nose-first through an atmosphere you choose. Everything runs
in the browser on WebGPU: the flow is solved, measured and drawn about thirty times a second
while you drag things.

![The application: schlieren view of a matched nozzle at steady state, with the live readouts
and the control panel](docs/img/app.png)

It exists to answer design questions you cannot get from a spreadsheet — *where does the flow
separate, what does the plume do at this altitude, how much does the nose cone cost me* —
while still putting the 1-D isentropic numbers next to the simulated ones so you can see when
they disagree.

## Run it

Open `index.html`. No build step, no server, no dependencies.

**WebGPU is required** — Chrome 113+, Edge 113+, Safari 26+, or Firefox 141+ on Windows.
Firefox on Linux may need `dom.webgpu.enabled`. Without it the page says so rather than
showing a blank canvas.

---

## What you are looking at

The view is a slice through the axis, mirrored about the centreline. Grey hatching is
hardware; the orange band at the head of the chamber is the injector plenum; the vertical
yellow line is the measurement plane.

![Geometry: nose cone, forebody, chamber full of exhaust, injector plenum, converging section,
throat and bell](docs/img/geometry.png)

Nose to tail: a **nose cone** (flat, pointy or rounded), a solid **forebody**, the
**combustion chamber** with the **injector plenum** at its head, a **converging section**, the
**throat**, and the **diverging bell**. Ambient air enters at the left at your chosen flight
speed, flows over the vehicle, and meets the exhaust at the base.

The injector is a band of cells held at a stagnation pressure and temperature. The flux
solver treats it as ordinary fluid, so the nozzle **chokes on its own** — nothing tells it
what mass flow to pass. That is why the choked-flow result further down counts as a
prediction.

### What you can change

| group | parameters |
|---|---|
| **Chamber & injector** | chamber pressure and temperature, ignition rise time, chamber radius and length |
| **Nozzle** | throat radius, exit radius, converging and diverging lengths, and a **conical or bell** divergent contour |
| **Airframe** | nose cone shape and length, forebody length, wall thickness |
| **Environment** | ambient pressure on a log slider from 0.01 to 125 kPa (sea level to about 65 km), ambient temperature, and flight speed |
| **Gas** | γ, specific gas constant, viscosity, Smagorinsky constant, tracer fade |
| **Domain & solver** | plume domain length, radial domain, axial cell stretch, radial cells, CFL, frame budget |
| **Measurement** | where the measurement plane sits, from the exit plane downstream |

The body radius is `max(chamber, nozzle exit) + wall thickness`, so the base is always an
annulus and there is always a base region for the plume and the external flow to fight over.

---

## A nozzle is only right at one altitude

This is the thing the tool is best at showing. Same chamber, same throat, three different
relationships between exit pressure and ambient — and three completely different flows.

**Matched** — the default. Exit pressure 1.00× ambient. The plume leaves parallel, with only
weak shock cells and a shear layer rolling up against the still air.

![Matched nozzle, schlieren](docs/img/matched.png)

**Over-expanded** — the same 40:1 vacuum bell fired at sea level. Exit pressure is 0.04×
ambient, and the jet cannot fill the nozzle: it separates from the wall a short way past the
throat and the rest of the bell is filled with recirculating gas. The panel flags this before
you run it.

![Over-expanded nozzle showing the jet separated from the bell wall](docs/img/overexpanded.png)

**Under-expanded** — a stubby nozzle at 60 bar. Exit pressure is 10.9× ambient, so the plume
goes on expanding outside through a Prandtl–Meyer fan, over-shoots, and recompresses into a
long train of shock diamonds.

![Under-expanded nozzle with a train of shock diamonds](docs/img/underexpanded.png)

Six presets set nozzle, chamber, atmosphere **and gas** together, because those choices are
not independent: *Sea-level booster*, *Vacuum upper stage*, *Over-expanded*,
*Under-expanded*, *Supersonic flight*, *Cold gas thruster*.

---

## Flying it

Set an ambient air speed and the vehicle is in flight. The nose cone then matters, and the
base becomes a real interaction between the external flow and the plume.

All three at Mach 2 at about 10 km, everything else identical:

**Flat** — a detached bow shock stands off the face with a subsonic pocket behind it. The
cheapest way to see a bow shock and the worst way to fly.

![Flat nose at Mach 2 with a detached bow shock](docs/img/nose-flat.png)

**Pointy** — the shock attaches at the tip and lies along the cone.

![Pointy nose at Mach 2 with an attached oblique shock](docs/img/nose-pointy.png)

**Rounded** — ellipsoidal, zero slope at the shoulder: the low-drag compromise most real
vehicles fly.

![Rounded nose at Mach 2](docs/img/nose-rounded.png)

In all three, look aft as well: the external flow separates off the base annulus, and the
plume acts as an aerodynamic body that the freestream has to go around.

---

## Fields

Eight ways to look at the same instant. Switching is instant and does not disturb the run.

| | |
|---|---|
| **Mach** — the sonic line in the throat and where the plume goes supersonic. White contour at M = 1. ![](docs/img/field-mach.png) | **Schlieren** — shocks. Bow shocks, lip shocks, diamonds, Mach discs. ![](docs/img/field-schlieren.png) |
| **Temperature** — chamber heat, plume cooling, recovery temperature on the nose. ![](docs/img/field-temperature.png) | **Exhaust fraction** — where the propellant goes and how much ambient air is entrained. ![](docs/img/field-exhaust.png) |

Also available: speed |u|, pressure relative to ambient, axial velocity (for spotting
recirculation and reversed flow), and vorticity (for shear layers). Every field is drawn with
a labelled legend giving the quantity, its units and the numeric range actually used that
frame.

---

## The numbers

Two blocks, deliberately separated, and a time-series plot of axial velocity at the
measurement plane.

<img src="docs/img/panel-numbers.png" alt="The metrics panel" width="380">

**Nozzle — 1-D ideal** is classical isentropic theory: expansion ratio, exit Mach from the
area–Mach relation, exit pressure, temperature and velocity, choked mass flow, thrust, thrust
coefficient, c\*, Isp, and the expansion ratio that would be *optimum* at the current ambient
pressure. These are exact for what they assume — no separation, no boundary layer, a uniform
exit — and they update as you drag.

**Environment** is what the atmosphere sliders actually mean in density, sound speed and
freestream Mach.

**Measured** is what the grid did: on-axis and peak axial velocity, peak Mach at the plane and
anywhere in the field, and two mass flows.

Those two mass flows matter. `propellant flow` is weighted by the exhaust tracer and is the
one to compare against the choked-throat figure; `all gas across` is everything crossing the
plane, propellant and entrained air alike. When they diverge, the nozzle is entraining
ambient air — which is the signature of a separated or badly expanded jet. `momentum + Δp` is
the exit-plane momentum-plus-pressure integral over the nozzle exit area, which at the exit
plane is the thrust.

### Warnings

The panel argues with you when the configuration deserves it — an unchoked nozzle, separation
past the Summerfield criterion, over- or under-expansion, a conical half-angle steep enough to
cost real momentum, an under-resolved throat, a transonic freestream making the base region a
genuine interaction. Warnings that invalidate the numbers appear *above* them.

<img src="docs/img/panel-warnings.png" alt="Warning panel for a badly over-expanded nozzle" width="360">

---

## The cost of one gas

The defaults are γ = 1.2 and R = 350 J/kg·K, roughly kerolox exhaust. The solver carries no
species, so that gas is also the atmosphere. This is the model's largest deliberate
approximation, so here is exactly what it costs and why the default sits where it does.

### What the atmosphere loses

At 101.325 kPa and 288.15 K:

| | real air (1.4 / 287) | model (1.2 / 350) | error |
|---|---|---|---|
| density | 1.225 kg/m³ | 1.005 kg/m³ | **−18 %** |
| speed of sound | 340.3 m/s | 347.9 m/s | +2.2 % |

Density is the one that matters: 18 % low means dynamic pressure ½ρV², and every aerodynamic
force with it, is 18 % low. The sound speed is nearly right by luck — γR is 402 for air and
420 here, and the square root halves the difference — so freestream Mach for a given flight
speed is only about 2 % off.

Across a normal shock at Mach 2:

| | air | model | error |
|---|---|---|---|
| density ratio ρ₂/ρ₁ | 2.667 | 3.143 | +18 % |
| pressure ratio p₂/p₁ | 4.50 | 4.27 | −5 % |
| stagnation temperature T₀/T | 1.80 | 1.40 | **−22 %** |
| stagnation pressure p₀/p | 7.82 | 7.53 | −4 % |

Pressure is nearly right; temperature is not. At 223 K and Mach 2 the real stagnation
temperature is 401 K and the model says 312 K, so anything about aeroheating is badly
under-predicted.

Shock standoff scales with the density ratio, and it does show up. Same flat nose, same
geometry, same Mach 2, only the gas changed:

| gas | ambient ρ | bow-shock standoff |
|---|---|---|
| 1.4 / 287 | 0.3925 kg/m³ | 25.6 mm (0.92 body radii) |
| 1.2 / 350 | 0.3218 kg/m³ | 23.1 mm (0.83 body radii) |

The shock sits 10 % closer to the nose than it should.

### What the engine would lose, the other way round

If instead you kept air properties and used them for the exhaust — 3000 K, 20 bar, each
expanded to its own optimum for 100 kPa:

| | air (1.4 / 287) | exhaust (1.2 / 350) | difference |
|---|---|---|---|
| enthalpy ceiling √(2c_p T_c) | 2455 m/s | 3550 m/s | **+45 %** |
| optimum ε at 100 kPa | 2.90 | 3.63 | +25 % |
| exit velocity | 1862 m/s | 2225 m/s | +19 % |
| c\* | 1355 m/s | 1580 m/s | +17 % |
| Isp | 190 s | 227 s | **+19 %** |
| mass flow | 296.7 g/s | 254.5 g/s | −14 % |
| **thrust** | **552 N** | **566 N** | **+2.5 %** |

That last row is the whole argument. **Thrust barely notices** — C_F is a weak function of γ,
and the drop in mass flow almost exactly cancels the rise in exit velocity. But **Isp, c\*,
exit velocity and the optimum expansion ratio all move by 15–25 %**, and the enthalpy ceiling
by nearly half. Getting the exhaust wrong corrupts every performance number and resizes the
nozzle; getting the ambient wrong costs 18 % on density and a fifth on recovery temperature.

The exhaust is the more expensive end to get wrong. That is why the default sits there.

### Switching ends

Set **γ = 1.4, R = 287** whenever the question is about the outside of the vehicle — bow shock
shape, nose-cone comparison, drag, aeroheating — and read the engine block as nonsense while
you do. The **Cold gas thruster** preset does this legitimately rather than as a compromise:
cold nitrogen really is a γ = 1.4 gas, so that one preset has both ends right at once. Its Isp
of 69 s is correct — real cold-gas thrusters land at 60–80 s.

You can also buy back ambient density by lowering the ambient temperature — 236 K instead of
288 K at 100 kPa restores ρ = 1.225 kg/m³ — but the sound speed then reads 7 % low. Two knobs,
three things to match; something has to give.

### What stays right either way

The pressure ratio p_c/p_a is exact. Choking, the area–Mach relation and the nozzle's internal
gas dynamics are all exact for whatever γ is set. Plume shock-cell structure is driven mostly
by the exit pressure ratio and the geometry, so the *shape* of the plume is about right even
when the ambient density is not.

### The one thing no single-gas model can do

In reality the plume boundary is a contact discontinuity with a **molecular-weight jump**
across it: at equal pressure and temperature, exhaust is roughly 1.2× less dense than air
because its R is larger. With no species there is no such jump, and the density ratio across
the plume edge comes from temperature alone. Shear-layer growth, entrainment and the
plume-to-freestream momentum ratio are therefore off by something like 20 %, and **no choice of
γ and R fixes it** — the two gases would need different values at the same instant. That needs
a second species and a variable-γ Riemann solver.

---

## How it works

Conserved state `U = (ρ, ρu_z, ρu_r, E)` as cell averages on a finite-volume grid in
cylindrical coordinates, so the slice knows it is a slice: the `1/r` terms are carried through
and the axis needs no special case.

- **HLLC** approximate Riemann solver, **MUSCL** reconstruction with a minmod limiter,
  **SSP-RK2** in time.
- **Thornber's low-Mach reconstruction fix.** Upwind dissipation grows like 1/M, and this
  problem has external flow at Mach 0.2 sitting beside a plume at Mach 4 on the same grid.
  Without the fix the slow side dissolves.
- **Reflecting-wall Riemann fluxes** rather than ghost cells — robust across the sharp throat
  and the nozzle lip.
- **Plenum boundary** at the injector: a Dirichlet region that takes part normally in the flux
  computation, which is the standard reservoir boundary.
- **Freestream inlet** imposing (ρ, u_z, p) from the ambient sliders; zero-gradient outflow and
  far field; a sponge relaxing to the freestream at all three open boundaries so outgoing
  acoustics do not ring back in.
- Viscous stress, viscous heating and conduction at constant Prandtl number, plus optional
  Smagorinsky sub-grid viscosity — a stand-in for the three-dimensional mixing that axial
  symmetry forbids, not a substitute for it.
- Timestep from a GPU max-reduction of the acoustic wave speed.

Six compute dispatches per step (two RK stages × flux-z, flux-r, update). The whole frame is
planned on the CPU and submitted in **one command encoder**, with per-stage uniforms addressed
by dynamic offset — a dispatch costs a few microseconds and a submit costs far more.

### Not modelled

Combustion, mixing, injector elements, multiple species, chemistry of any kind, radiation,
ablation, film cooling, nozzle flexure. The chamber is simply *held* at a stagnation state.

---

## How well it works

Default configuration: 25 mm chamber, 8 mm throat, 15.25 mm exit (ε = 3.63), bell contour,
20 bar and 3000 K, at 100 kPa ambient in still air. Measured at the nozzle exit plane at
t = 1.2 ms, by which point the flow is steady.

### It chokes, and the mass flow saturates

Sweeping chamber pressure at fixed geometry, against
`ṁ = p_c·A_t·√(γ/RT₀)·(2/(γ+1))^((γ+1)/2(γ−1))`:

| p_c / p_ambient | ideal ṁ | measured propellant ṁ | ratio | *all* gas crossing the plane |
|---|---|---|---|---|
| 3 | 38.2 g/s | 24.9 g/s | 0.65 | 92.6 g/s |
| 5 | 63.6 g/s | 59.8 g/s | 0.94 | 91.7 g/s |
| 10 | 127.3 g/s | 127.9 g/s | **1.01** | 139.6 g/s |
| 20 | 254.5 g/s | 236.7 g/s | **0.93** | 244.6 g/s |

Above about 10:1 the measured flow tracks the choked-throat formula to within a few percent
and scales linearly with chamber pressure — the throat has stopped listening to the ambient.
Nothing in the setup imposes this; it falls out of a plenum boundary and a Riemann solver.

The first row is **not** a solver error, and reading it as one is the trap. At ε = 3.63 the
nozzle needs roughly 20:1 to flow full. At 3:1 it is grossly over-expanded, the jet separates
and recirculates, and the exit plane is simply the wrong place to measure the throat's mass
flow. Look at the last column: nearly four times more gas crosses the exit plane than the
throat passes, because most of it is entrained ambient air. The two columns diverging *is* the
diagnostic.

### It converges

Same case at three radial resolutions:

| radial cells | cells across throat R | grid | ṁ vs ideal | exit-plane thrust | on-axis u_z | solver time for 1.2 ms |
|---|---|---|---|---|---|---|
| 64 | 7.3 | 257 × 64 | 83 % | 417 N | 2336 m/s | 0.7 s |
| 112 (default) | 12.8 | 450 × 112 | 93 % | 480 N | 2356 m/s | 2.2 s |
| 176 | 20.1 | 708 × 176 | 95 % | 484 N | 2390 m/s | 10.7 s |

Monotone, and converged to a couple of percent by the default. The controlling number is
**cells across the throat radius**, not the total cell count — the panel shows it live and
warns below six. Plume shock-cell structure further downstream is still resolution-sensitive;
the throat and the exit plane are not.

### It loses what a real nozzle loses

At the default, measured exit-plane momentum-plus-pressure is 85 % of 1-D ideal thrust (566 N
ideal, 480 N measured). The missing 15 % is boundary layer, non-uniform exit profile and the
finite-rate startup — all things 1-D theory assumes away.

### Solver validation

- **Sod shock tube** against the exact Riemann solution, 601 cells: 0.17 % / 0.18 % / 0.09 %
  L1 error in density, velocity and pressure, with zero overshoot at every resolution tested.
  Grid convergence 0.88 and 0.80 in L1 — first order, which is correct for a solution
  containing discontinuities.
- **Speed of sound**: a 1 % Gaussian pressure pulse propagated at 344 m/s against a
  theoretical 343.1 — 0.26 % error, with total mass drifting by 4 × 10⁻⁴ %.
- **Choking through a sharp orifice**: mass flow plateaus above the critical pressure ratio at
  87 % of ideal, the discharge coefficient of a sharp-edged short tube — a real
  vena-contracta effect. The smooth cosine contraction used here has no vena contracta, which
  is why its coefficient sits near 1.0.

---

## Video export

Exports MP4/H.264 by default, or WebM/VP9 or VP8. WebCodecs supplies encoded chunks but no
container, so both muxers are written here — a minimal Matroska writer and a minimal MP4
writer.

The interactive solver chooses its timestep adaptively, which is right on screen and wrong for
video: frames would represent unequal slices of time and the motion would be subtly, invisibly
wrong. **Export runs on a fixed schedule** — every video frame advances exactly the same
simulated interval, subdivided into as many equal sub-steps as stability requires. The
sub-step count varies between frames; the frame interval never does.

**Export all fields** writes one file per field from a *single* simulation. Stepping is the
expensive part and rendering is nearly free, so eight fields cost far less than eight runs.

**The colour scale is fixed for the whole clip.** On screen the range tracks the flow, which
is what you want while exploring; in a video it means a colour does not signify the same thing
from one frame to the next. Export runs the shot once to find the peak, fixes the range, then
records — and caches the peak per configuration so a repeat export skips the scan.

Frames carry a burned-in clock, the configuration, the colour legend and a physical scale bar,
in bands above and below the image rather than on top of the flow.

## Shareable links

Every physical setting is serialised into the URL fragment, so a link reproduces a
configuration exactly. Only non-defaults are written and the keys are two characters, so a
typical link carries a handful; with all 35 parameters off their defaults it comes to 269
characters.

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

## Performance

On an Apple M1, the default 450 × 112 grid paces itself to **122 solver steps per frame at
28 ms, giving 36 fps and 0.37 ms of flight per second** — so the ~1.5 ms it takes to reach
steady state arrives in about four seconds. Those are the numbers in the readout at the top of
the screenshot above.

Cost is close to linear in the radial cell count rather than quadratic: the timestep is set by
the cell size, so a finer grid pays twice, once in cells and once in steps. The **radial
domain** slider is the other main lever — it trades far-field room against resolution at fixed
cell count, so widen it when a bow shock needs space and narrow it to put cells in the throat.

Steps per frame are auto-paced to a frame-time budget, because the timestep varies by orders
of magnitude across the parameter space and without pacing a small throat is a slideshow.

## What to trust

**Structure.** Where the sonic line sits. Whether the flow separates inside the bell, and how
far up. Shock-diamond spacing and Mach discs in an under-expanded plume. Whether a bow shock
is attached or detached. How the base flow and the plume interact. These are robust and
reproduce across resolutions.

**The 1-D numbers.** Exact for what they assume, and clearly labelled as such.

**Not the absolute measured figures.** They are converged to a few percent on the default grid
for the throat and the exit plane, which is good enough to rank designs and not good enough to
quote a thrust — and they are the thrust of a nozzle flowing one idealised gas, not combustion
products.

---

## Provenance

Forked from [BenWheatley/Airzooka](https://github.com/BenWheatley/Airzooka). The compressible
solver kernels, the WebGPU plumbing, the video exporter and the shareable-link machinery come
from there.
