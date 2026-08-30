# Rocket Sim

An interactive, axisymmetric compressible-flow sandbox for a rocket's aft end. A hot-gas
injector feeds a combustion chamber and a converging–diverging nozzle you can reshape with
sliders, while the vehicle flies nose-first through an atmosphere you choose. Exhaust and
atmosphere are **two different gases**, each with its own γ and molecular weight, so the plume
boundary is a real contact surface. Everything runs in the browser on WebGPU.

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
| **Chamber & injector** | chamber pressure, ignition rise time, chamber radius and length |
| **Propellant** | six exhaust compositions as buttons, plus flame temperature, γ and R directly |
| **Nozzle** | throat radius, exit radius, converging and diverging lengths, and a **conical or bell** divergent contour |
| **Airframe** | nose cone shape and length, forebody length, wall thickness |
| **Atmosphere** | six worlds as buttons, plus ambient pressure on a log slider from 10 Pa to 10 MPa, temperature, flight speed, surface gravity, γ and R directly |
| **Vehicle & flight** | acceleration on/off, level or vertical flight, dry mass, propellant mass, trajectory fast-forward |
| **Fluid** | viscosity, Smagorinsky constant, tracer fade |
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

### Presets

Eight, each setting nozzle, chamber, propellant **and** atmosphere together, because those
choices are not independent:

| preset | what it shows |
|---|---|
| **Sea-level booster** | kerolox on Earth, ε 3.4, matched — Isp 251 s |
| **Vacuum upper stage** | hydrolox at 1 kPa, ε 40 — **Isp 442 s**, and under-expanded, because in vacuum the optimum ε is infinite and every real vacuum nozzle is truncated to save mass |
| **Over-expanded** | that same hydrolox bell at sea level: separation, and the shock system moving inside the nozzle |
| **Under-expanded** | stubby kerolox at 60 bar, a clean train of shock diamonds |
| **Supersonic flight** | pointy nose at Mach 2 at 10 km — bow shock, shoulder expansion and base flow together |
| **Mars ascent** | methalox into 0.64 kPa of CO₂, a pressure ratio of 3141 |
| **Venus surface** | 250 bar against 92 bar of back pressure. ε 1.1, Isp 159 s: the nozzle is nearly a plain hole |
| **Titan flight** | methalox at Mach 1.5 through cold dense nitrogen — the entrainment case |

### Things worth trying

- Click **Hydrolox**, then **Kerolox**, and watch Isp move by a third while thrust barely
  changes. C_F is a weak function of γ; c* is not.
- Put the **Vacuum upper stage** nozzle on **Venus** and watch it refuse to start.
- Switch the nose to **flat** at Mach 2 for a detached bow shock, then **pointy** to attach it.
- Fire the default engine into **Jupiter** and compare the plume with Earth at the same
  pressure. Only the ambient gas changed.
- Move the measurement plane downstream and watch the thrust integral stop meaning thrust.
- Tick **accelerate**, set **vertical**, raise the fast-forward, and watch drag climb as v²
  until burnout — then watch the vehicle coast, thinning air lifting the thrust as it goes.
- Launch the same vehicle from **Mars** and from **Venus**: 0.016 kg/m³ against 65 kg/m³ of
  ambient, and 3.7 m/s² against 8.9 of gravity.

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
anywhere in the field, two mass flows, and the axial force balance — thrust, pressure drag and
the net of the two. Thrust and drag also appear live in the readout over the flow and in the
header of exported video.

**Flight** turns that force balance into trajectory numbers: airspeed, current mass, burn time,
thrust-to-weight, acceleration, measured Isp and ideal Δv — plus altitude and how far the
ambient pressure has fallen, once you are climbing.

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

## Thrust, drag and flight

### Where the two numbers come from

They are separate integrals over separate surfaces, which is why they do not double-count.

**Thrust** is the momentum-plus-pressure integral across the nozzle exit plane, over the exit
area only: ∫(ρu² + p − p_ambient) dA.

**Drag** is the (p − p_ambient) integral over the *external* wetted surface — the nose, the
body and the base annulus. It is computed as one axial slab per thread: each slab's surface
spans some range of radii, and the axial force is the gauge pressure over that ring's projected
area, signed by whether the ring faces forward or aft. Pressure is sampled from the fluid cell
adjacent to the surface on the side the flow is on — upstream of a forward-facing ring, aft of
the base. Written with a surface radius that is zero ahead of the nose and the nozzle exit
radius aft of the base, the slabs telescope correctly and a uniform ambient pressure integrates
to exactly zero force, which is the check that the bookkeeping is right.

**Skin friction is not included.** The boundary layer is unresolved at this cell size, so any
wall-shear number would be a function of the grid rather than of the flow. What is reported is
pressure drag, which at supersonic speed is the term that responds to nose shape — and the term
this tool is useful for. Expect real total drag to be higher, by roughly a third on a slender
body.

### Does the drag number stand up?

Three nose shapes at Mach 2, everything else identical, C_D referenced to the frontal area:

| nose | drag | C_D | expected |
|---|---|---|---|
| flat | 261 N | **1.50** | 1.5–1.8 for a flat-faced cylinder; the normal-shock stagnation pressure alone gives 1.65 |
| pointy (27° half-angle) | 94 N | 0.54 | cone wave drag plus base drag |
| rounded | 95 N | 0.55 | as pointy, at this stubby fineness ratio |

The second independent check is the velocity scaling. During an accelerating run, drag went
10.8 N at 144 m/s → 113 N at 447 m/s: a speed ratio of 3.1 against a drag ratio of 10.5, where
v² would predict 9.6.

### Two clocks, and they are not the same

This trips people up, so the readout now names both every time they appear.

- **Flow time**, milliseconds. The CFD clock: how much gas dynamics has been simulated. It is
  the `flow t` in the readout and the axis of the plot.
- **Flight time**, seconds. The trajectory clock: how far into the burn the vehicle is.

They tick at the same rate only at 1× fast-forward, and 1× is useless: the flow settles in
about 2 ms while a burn lasts seconds, so at real time the trajectory would never visibly move.
The readout therefore always prints the factor —

```
flow t = 1.537 ms
flight T+4.78 s   = flow t × 3162 fast-forward
```

— and a third clock, wall-clock time, appears only in the performance line, now labelled
`ms of flow per wall second` so it cannot be confused with either.

### Flying a trajectory

<img src="docs/img/flight-controls.png" alt="The vehicle and flight controls" width="330">

Tick **accelerate** and the airspeed stops being a boundary condition and becomes a state. The
vehicle accelerates under thrust minus drag, burns propellant at the rate the solver is already
measuring, and shuts the chamber down when the tanks are dry — ramped down over the ignition
time rather than cut, because a step would launch an expansion wave as violent as the starting
shock. There is **no consumption-rate input** because there is nothing to input: ṁ is measured,
so burn time follows. The Δv it reports is the propulsive figure only, with no gravity or drag
losses subtracted.

**Level** flight is the axial force balance alone. **Vertical** adds the weight term and tracks
altitude, and the atmosphere then thins as you climb: pressure falls as exp(−h/H). So a nozzle
matched at the surface becomes under-expanded on the way up while drag falls away with the
density. Altitude is measured from wherever the run started.

A vertical launch from the Earth surface at 3162× fast-forward, 2 kg dry and 1 kg of kerolox:

| flight time | altitude | airspeed | ambient | thrust | drag | state |
|---|---|---|---|---|---|---|
| T+3.7 s | 0.6 km | 426 m/s | 93 % | 484 N | 98 N | climbing, T/W 16.3 |
| T+5.6 s | 1.7 km | 673 m/s | 82 % | 493 N | 284 N | tanks nearly dry |
| T+7.5 s | 3.0 km | 607 m/s | 70 % | 13 N | 260 N | burnt out, coasting |
| T+11.3 s | 4.7 km | 348 m/s | 57 % | −2 N | 100 N | still coasting up |

Thrust *rises* slightly on the way up as the back pressure falls, drag peaks and then collapses
with the density, and after burnout the vehicle coasts and slows under drag and weight. Two
things it will tell you about rather than fake: if thrust-to-weight is below 1 it says the
vehicle will not lift off, and if the climb decelerates to zero it stops there and says so,
because a nose-first solver cannot model falling back tail-first.

![Mid-burn on a vertical climb](docs/img/flight.png)

**Gravity is constant with altitude and there is no flight-path angle.** Over the few
kilometres a burn like this covers, the inverse-square correction is a fraction of a percent;
a gravity turn is a different tool.

### Why thrust goes negative after burnout

Because after burnout it is not thrust. The exit-plane integral ∫(ρu² + p − p_ambient) dA is
only "thrust" while the engine is producing flow. With the chamber ramped down to ambient, the
nozzle is an open pipe with ambient gas being dragged through it, and the same integral reads a
small *negative* number — a few newtons of internal drag on a dead engine. That is correct, and
the panel now says so explicitly the moment burnout happens, as does the readout.

### The honest problem with watching a burn

The flow settles in about 2 ms. A burn lasts seconds. Those clocks are three or four orders of
magnitude apart, so the coupling is **quasi-steady**: the trajectory is integrated on its own
clock and the fast-forward factor is a control rather than something that can be derived.

At the default 20× the airspeed moves about 6 m/s while the flow is settling — properly
quasi-steady, and far too slow to sit and watch a five-second burn. Crank it and the bow shock
builds in seconds, but the flow is then chasing a boundary condition it never catches. The
panel computes that directly — how far the airspeed moves, and how much the ambient pressure
changes, during one settling time — and puts a red flag above the numbers past 5 %. The table
above was run at 3162×, well past that threshold, which is why it is a demonstration rather
than a result.

---

## Two gases

The exhaust and the atmosphere are separate species. Y is the exhaust mass fraction, carried
as ρY and **active rather than passive**: it sets the local equation of state. For a mixture of
two calorically perfect gases at a common temperature the specific heats add by mass fraction,
which is exact, and the rest follows —

```
cv = Y·cv_e + (1−Y)·cv_a        R = Y·R_e + (1−Y)·R_a
γ  = (cv + R) / cv              p = ρRT = (γ−1)ρe
```

— so a contact surface between plume and atmosphere is a jump in the equation of state, not
only in temperature. Two rows of buttons pick the ends:

<img src="docs/img/gas-controls.png" alt="The propellant and atmosphere control groups" width="330">

### Propellants

Exhaust composition at roughly optimum mixture ratio and a few tens of bar. There is no
chemistry: the buttons set what gas leaves the injector and how hot it is.

| propellant | γ | M (g/mol) | T_c (K) | model c\* | model vacuum Isp at ε 60 | real engines |
|---|---|---|---|---|---|---|
| Kerolox, RP-1/LOX | 1.24 | 22.2 | 3570 | 1763 m/s | 336 s | RD-180 338 s, Merlin Vac 348 s |
| Hydrolox, LH2/LOX | 1.22 | 12.0 | 3400 | 2353 m/s | **454 s** | RS-25 452 s, RL10 465 s |
| Methalox, CH4/LOX | 1.20 | 20.5 | 3540 | 1849 m/s | 361 s | Raptor Vac ≈ 380 s |
| Hypergolic, N2O4/MMH | 1.23 | 21.5 | 3200 | 1701 m/s | 326 s | AJ10 ≈ 320 s |
| Solid, APCP | 1.18 | 27.0 | 3200 | 1540 m/s | 305 s | high-performance ≈ 300 s |
| Cold gas, N₂ | 1.40 | 28.0 | 290 | 429 m/s | 76 s | 60–80 s |

That column of Isp figures is the validation that matters most: nothing in the code was tuned
to produce it. Chamber pressure, throat area and the gas properties go in; c\* and Isp come out
of the area–Mach relation, and they land within a few percent of the engines they describe.
Hydrolox at 454 s against an RS-25's 452 s is the one to look at.

The remaining gap — the model reads a little low for methalox and solids — is chemistry. These
are **frozen-flow** numbers: specific heats are constant, so nothing dissociates in the chamber
and nothing recombines on the way down the nozzle. A real engine sits between the frozen and
equilibrium limits and recovers a few percent that this does not.

### Atmospheres

| world | composition | γ | M (g/mol) | pressure | T | ρ | sound speed | g | scale height | p_c to choke |
|---|---|---|---|---|---|---|---|---|---|---|
| Earth | N₂/O₂ | 1.40 | 29.0 | 101 kPa | 288 K | 1.23 kg/m³ | 340 m/s | 9.81 | 8.4 km | 1.8 bar |
| Mars | 96% CO₂ | 1.29 | 43.4 | 0.64 kPa | 210 K | 0.016 kg/m³ | 228 m/s | 3.72 | 10.8 km | 0.01 bar |
| Venus | 96% CO₂ | 1.29 | 43.4 | 9.2 MPa | 737 K | **65.2 kg/m³** | 427 m/s | 8.87 | 15.9 km | **165 bar** |
| Jupiter | 89% H₂, 10% He | 1.43 | **2.22** | 1 bar level | 165 K | 0.162 kg/m³ | **941 m/s** | 24.8 | 25 km | 1.8 bar |
| Titan | 95% N₂ | 1.40 | 27.3 | 147 kPa | 94 K | 5.12 kg/m³ | 200 m/s | 1.35 | 21.2 km | 2.6 bar |
| Vacuum | — | — | — | 10 Pa | — | 10⁻⁴ kg/m³ | — | 0 | — | none |

**Scale height is not an input.** H = RT/g falls straight out of the gas constant, the
temperature and the gravity already set, and it reproduces every published figure here to
within a few percent — 8.4 km for Earth against a real 8.5, 15.9 for Venus against 15.9,
21.2 for Titan against 21.

The last column is the one that surprises people. A nozzle only chokes above about 1.8× ambient,
so **on the surface of Venus an engine below 165 bar chamber pressure does not start at all** —
the throat stays subsonic and the bell works as a diffuser. The panel says so when you try it.

### Same engine, three worlds

Identical kerolox engine and nozzle geometry, schlieren, at steady state. Only the atmosphere
was changed.

**Earth**, 101 kPa. Expansion ratio 3.4 is matched here: the plume leaves parallel with weak
shock cells and a shear layer that rolls up quickly against dense air.

![Kerolox on Earth](docs/img/matched.png)

**Mars**, 0.64 kPa — a pressure ratio of 3141. Even a 40:1 bell cannot expand against this
much vacuum, so the plume keeps expanding after it leaves, into a vast diffuse cone.

![Methalox on Mars](docs/img/world-mars.png)

**Venus**, 92 bar. At 250 bar chamber pressure the pressure ratio is 2.7, barely above choking,
so the optimum expansion ratio is 1.1 — the nozzle is very nearly a plain hole, the exhaust
velocity collapses to 1123 m/s and Isp to 159 s. The atmosphere is denser than the exhaust.

![Kerolox on Venus](docs/img/world-venus.png)

### Same engine, same pressure, different gas

This is the pair that isolates what a second species buys. Both are the default kerolox engine
at 1 bar back pressure, so the nozzle flow inside is identical. The only difference is what it
is expanding *into*.

Earth's air is heavy and slow: M 29, sound speed 340 m/s, so the plume is hypersonic relative
to it and the shear layer breaks up within a few diameters — that is the image above. Jupiter's
hydrogen is M 2.22 with a sound speed of 941 m/s, so the same plume is only mildly supersonic
relative to the ambient, the shear layer stays thin, and the shock-cell train survives the
whole length of the domain:

![The same engine firing into a hydrogen atmosphere](docs/img/atm-jupiter.png)

A single-gas model cannot show this at all. With one γ and one R, the plume boundary carries
only a temperature jump; here it carries a factor of thirteen in molecular weight.

### Mixing

Exhaust fraction on Titan at Mach 1.5, where the ambient nitrogen is 5.1 kg/m³ — four times as
dense as Earth air. The bright core is pure exhaust; the fade is real entrainment, and at the
exit plane the axis is already only 84% exhaust.

![Exhaust fraction on Titan](docs/img/mixing.png)

Species diffusion runs at unity Schmidt number using the same eddy viscosity as the momentum
equations, so it is the sub-grid model that does the mixing — with the usual caveat that axial
symmetry forbids the three-dimensional instabilities a real shear layer uses.

### The numerical cost of a second species

Conservative schemes for multi-component flow are known to generate spurious pressure
oscillations at material interfaces, because the energy equation is conservative while γ is
not uniform. This one uses HLLC with each side carrying its own γ, and takes the species flux
as the mass flux times the mass fraction from whichever side the contact wave came from, which
is what keeps mass and species consistent.

Measured directly: a contact discontinuity between kerolox exhaust and air at uniform pressure
and velocity, no viscosity, in an empty domain. The exact answer is that it simply advects.

| after | pressure spread | velocity spread |
|---|---|---|
| 200 steps | 0.56 % of p₀ | 1.2 % |
| 1000 steps | 0.37 % | 0.73 % |

Sub-percent, one-sided, and **decaying** rather than growing as the interface smears. Against
the pressure ratios this tool actually deals in — 20:1 and up — it is invisible. It is not zero,
and a quasi-conservative scheme would make it zero at the cost of exact energy conservation.

## How it works

Conserved state `U = (ρ, ρu_z, ρu_r, E)` as cell averages on a finite-volume grid in
cylindrical coordinates, so the slice knows it is a slice: the `1/r` terms are carried through
and the axis needs no special case.

- **HLLC** approximate Riemann solver, **MUSCL** reconstruction with a minmod limiter,
  **SSP-RK2** in time. Each side of a face carries its own γ, and the mass fraction is
  reconstructed alongside (ρ, u, p) so that species and mass stay consistent at second order.
- **Species flux from the same Riemann solve**: the mass flux times the mass fraction on
  whichever side the contact wave came from.
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
- Viscous stress, viscous heating and conduction at constant Prandtl number, each cell using
  its own mixture c_p and R, plus species diffusion at unity Schmidt number, plus optional
  Smagorinsky sub-grid viscosity.
- Timestep from a GPU max-reduction of the acoustic wave speed.

Six compute dispatches per step (two RK stages × flux-z, flux-r, update). The whole frame is
planned on the CPU and submitted in **one command encoder**, with per-stage uniforms addressed
by dynamic offset — a dispatch costs a few microseconds and a submit costs far more.

### Not modelled

Chemistry, above all: specific heats are constant, so nothing dissociates in the chamber and
nothing recombines as the flow expands. This is the frozen-flow limit, and a real nozzle
recovers a few percent that this does not. Also no combustion, no injector elements, no
radiation, ablation, film cooling or nozzle flexure — the chamber is simply *held* at a
stagnation state. Axial symmetry forbids the three-dimensional instabilities that break a real
shear layer up, so the Smagorinsky term stands in for them; a stand-in, not a substitute.

---

## How well it works

Default configuration: kerolox (γ 1.24, M 22.2, 3570 K) at 20 bar, through a 25 mm chamber, an
8 mm throat and a 14.75 mm bell exit (ε = 3.40), into Earth air at 101 kPa, still. That exit
radius is the optimum for this pressure ratio, so the default is a matched nozzle: exit pressure
comes out at 1.01× ambient. Measured at the exit plane at **t = 2 ms** — the readings are still
drifting at 1.2 ms and settle by 2.

### It chokes, and the mass flow saturates

Sweeping chamber pressure at fixed geometry, against
`ṁ = p_c·A_t·√(γ/RT₀)·(2/(γ+1))^((γ+1)/2(γ−1))`:

| p_c / p_ambient | ideal ṁ | measured propellant ṁ | ratio | *all* gas crossing the plane |
|---|---|---|---|---|
| 3.0 | 34.2 g/s | 28.5 g/s | 0.83 | 38.9 g/s |
| 4.9 | 57.0 g/s | 52.2 g/s | 0.92 | 61.7 g/s |
| 9.9 | 114.0 g/s | 102.1 g/s | 0.90 | 110.2 g/s |
| 19.7 | 228.1 g/s | 207.7 g/s | **0.91** | 213.1 g/s |

Above about 5:1 the discharge coefficient settles at a steady **0.90–0.92** and the flow scales
linearly with chamber pressure — the throat has stopped listening to the ambient. Nothing in the
setup imposes this; it falls out of a plenum boundary and a Riemann solver. The missing 8–10 %
is the boundary-layer displacement thickness at the throat, which is what a real nozzle loses
too.

The first row is **not** a solver error, and reading it as one is the trap. At ε = 3.40 the
nozzle needs roughly 20:1 to flow full. At 3:1 it is badly over-expanded, the jet separates and
recirculates, and the exit plane is the wrong place to measure the throat's mass flow: 36 % more
gas crosses that plane than the throat passes, and the difference is entrained air. The two
columns diverging *is* the diagnostic.

### It converges

Same case at three radial resolutions:

| radial cells | cells across throat R | grid | ṁ vs ideal | exit-plane thrust | on-axis u_z | solver time for 2 ms |
|---|---|---|---|---|---|---|
| 64 | 7.3 | 257 × 64 | 80 % | 415 N (74 %) | 2628 m/s | 2.8 s |
| 112 (default) | 12.8 | 450 × 112 | 91 % | 478 N (85 %) | 2631 m/s | 17 s |
| 176 | 20.1 | 708 × 176 | 90 % | 482 N (86 %) | 2716 m/s | 88 s |

Monotone, and converged to a couple of percent by the default. The controlling number is
**cells across the throat radius**, not the total cell count — the panel shows it live and
warns below six. Plume shock-cell structure further downstream is still resolution-sensitive;
the throat and the exit plane are not.

### It loses what a real nozzle loses

At the default, measured exit-plane momentum-plus-pressure is 85 % of 1-D ideal thrust (562 N
ideal, 478 N measured). The missing 15 % is boundary layer, non-uniform exit profile and the
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
typical link carries a handful; with all 43 parameters off their defaults it comes to 329
characters.

The fragment is used rather than the query string because it never reaches a server, and
because assigning `location.hash` works on `file://` URLs where Chrome throws on
`history.replaceState`.

Links are validated on the way in — values clamped to their control's range, unknown keys and
unparseable numbers ignored — so a hand-edited `#tr=99999&re=-50&ra=abc&nt=77` loads as a
valid configuration rather than breaking. Round-tripping is tested: encoding a fully
non-default configuration, resetting everything, then decoding restores all 43 settings with
zero mismatches.

Back and forward restore the configuration *and* restart the run — otherwise you would be
watching one nozzle's flow inside another nozzle's geometry. Dragging a slider replaces the
current history entry; a discrete action pushes one.

## Performance

On an Apple M1, at the 30 ms default frame budget:

| quality | grid | cells across throat R | steps/frame | ms of flight per second | to steady state |
|---|---|---|---|---|---|
| Draft | 257 × 64 | 7.3 | 111 | 0.72 | ~3 s |
| **Balanced** (default) | 450 × 112 | 12.8 | 43 | **0.11** | ~19 s |

Balanced is the one to believe and Draft is the one to sweep parameters with — 9× the
throughput for a discharge coefficient that reads 80 % instead of 91 %. The readout at the top
of the screenshot above is a Balanced run.

Two species cost roughly 3× the throughput of one: every primitive state now needs the mass
fraction fetched alongside the conserved variables, so the flux kernels do twice the memory
traffic, and the mixture rule is evaluated per cell rather than hoisted into a constant. The
`update` kernel passes its four neighbours into the Smagorinsky term rather than re-reading
them, which recovers about 40 % of that.

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
quote a thrust. Every performance number is a frozen-flow number: real chemistry recovers a few
percent that this does not. And the drag is pressure drag: real total drag is higher.

---

## Provenance

Forked from [BenWheatley/Airzooka](https://github.com/BenWheatley/Airzooka). The compressible
solver kernels, the WebGPU plumbing, the video exporter and the shareable-link machinery come
from there.
