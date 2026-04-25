# 2D Laser Powder Bed Fusion Simulation - FreeFEM++

<p align="center">
  <img src="https://img.shields.io/badge/FreeFEM++-Simulation-blue?style=for-the-badge&logo=gnu&logoColor=white"/>
  <img src="https://img.shields.io/badge/Laser%20PBF-Powder%20Melting-orange?style=for-the-badge"/>
  <img src="https://img.shields.io/badge/Melt%20Pool-Tracking-red?style=for-the-badge"/>
  <img src="https://img.shields.io/badge/Multi--Material-Cu%20%2B%20Steel-purple?style=for-the-badge"/>
  <img src="https://img.shields.io/badge/ParaView-VTK%20Export-green?style=for-the-badge"/>
  <img src="https://img.shields.io/badge/DOI-10.5281%2Fzenodo.19758536-blue?style=for-the-badge"/>
  <img src="https://img.shields.io/badge/License-CC%20BY--NC%204.0-lightgrey?style=for-the-badge"/>
</p>

<p align="center">
  A 2D finite element simulation of <b>laser powder bed fusion (L-PBF)</b> using FreeFEM++.
  Models a Gaussian laser beam scanning across a line of copper particles sitting on a steel
  substrate, tracking sequential melting, melt pool dynamics, and cumulative heat-affected zone.
</p>
<img width="1008" height="772" alt="lpbffeA" src="https://github.com/user-attachments/assets/ccdc5314-a694-4f6d-951f-1d9211e6d2ee" />

---

## Citation

If you use this code in your research, please cite:

```bibtex
@software{mishra_2026_laserpowder,
  author    = {Mishra, A.},
  title     = {2D Laser Powder Bed Fusion Simulation - FreeFEM++},
  year      = {2026},
  publisher = {Zenodo},
  doi       = {10.5281/zenodo.19758536},
  url       = {https://doi.org/10.5281/zenodo.19758536}
}
```

Plain text citation:

> Mishra, A. (2026). *2D Laser Powder Bed Fusion Simulation - FreeFEM++*. Zenodo. https://doi.org/10.5281/zenodo.19758536

---

## Physics

Laser Powder Bed Fusion (L-PBF) is an additive manufacturing process where a focused laser beam selectively melts metallic powder particles layer by layer to build near-net-shape components. The thermal history of the melt pool governs grain structure, residual stress, porosity, and final part properties.

This simulation models the following coupled phenomena:

- Transient 2D heat conduction with implicit Euler time integration
- Three-material domain: copper particles + steel substrate + air gaps
- Moving Gaussian laser beam as a surface flux (Neumann BC on top edge)
- Convective heat loss on all boundaries (Newton cooling)
- Sequential per-particle melt pool detection as beam scans across
- Cumulative heat-affected zone (HAZ) tracking via latching melt indicator
- Material visualization field for ParaView region identification

---

## Geometry

```
  Laser beam (Gaussian, scans left to right)
       |
       v
  |----o----o----o----o----o----o----o----o----o----o----|  <- TopEdge
  | air|Cu  |air |Cu  |air |Cu  |air |Cu  |air |Cu  |air|  <- particle layer
  |    Ly = 3mm (substrate surface)                      |
  |                                                       |
  |               Steel Substrate                         |  Ly = 3 mm
  |_______________________________________________________|
  0                     Lx = 10 mm
```

- `Np = 10` copper particles equally spaced along the substrate top surface
- Each particle has radius `Rp = 0.5 mm` and sits directly on the substrate (`ycenter = Ly + Rp`)
- Air fills the gaps between particles and above them
- The laser beam hits the top boundary and transfers heat downward through the powder layer

---

## Material Parameters

### Copper Particles (powder)

| Parameter | Symbol | Value | Unit |
|-----------|--------|-------|------|
| Density | rhoP | 8960 | kg/m3 |
| Specific heat | cpP | 385 | J/kg/K |
| Thermal conductivity | kkP | 30 | W/m/K |
| Melting temperature | TmP | 1356 | K |

> Note: conductivity is reduced from bulk (385 W/m/K) to 30 W/m/K to account for powder bed porosity and inter-particle contact resistance.

### Steel Substrate

| Parameter | Symbol | Value | Unit |
|-----------|--------|-------|------|
| Density | rhoS | 8000 | kg/m3 |
| Specific heat | cpS | 500 | J/kg/K |
| Thermal conductivity | kkS | 50 | W/m/K |
| Melting temperature | TmS | 1700 | K |

### Air Gaps

| Parameter | Symbol | Value | Unit |
|-----------|--------|-------|------|
| Density | rhoA | 1.2 | kg/m3 |
| Specific heat | cpA | 1000 | J/kg/K |
| Thermal conductivity | kkA | 0.025 | W/m/K |

### Boundary Conditions

| Parameter | Symbol | Value | Unit |
|-----------|--------|-------|------|
| Convection coefficient | hc | 20 | W/m2/K |
| Ambient temperature | T0 | 300 | K |

---

## Laser Parameters

| Parameter | Symbol | Value | Unit |
|-----------|--------|-------|------|
| Laser power | P | 200 | W |
| Beam radius (1/e2) | r0 | 0.4 | mm |
| Surface absorptivity | absorb | 0.6 | - |
| Scan speed | vbeam | 3 | mm/s |
| Peak flux | Qpeak | absorb * 2P / (pi * r0^2) | W/m2 |

> Absorptivity is set to 0.6 (higher than bulk metal) to represent enhanced absorption in loose powder beds due to multiple scattering between particles.

---

## Governing Equations

### Transient Heat Conduction

```
rho(x) * cp(x) * dT/dt - div(k(x) * grad(T)) = 0        in Omega
```

Where `rho`, `cp`, `k` are spatially varying (different in each material region).

### Boundary Conditions

```
-k * dT/dn = Q_laser(x, t)     on TopEdge   [laser surface flux]
-k * dT/dn = hc * (T - T0)     on all edges [Newton convection]
```

### Gaussian Laser Surface Flux

```
Q(x, t) = absorb * (2P / (pi * r0^2)) * exp(-2 * (x - xc(t))^2 / r0^2)
```

Where `xc(t) = vbeam * t` is the moving beam centre position.

### Melt Pool Detection

```
MeltPool(x, t) = 1    if T(x,t) >= TmP       [currently molten]
MeltPool(x, t) = 0    if T(x,t) <  TmP       [solid]

MeltEver(x)    = 1    if T(x,t') >= TmP for any t' <= t   [latched, never resets]
MeltDepth(x,t) = T(x,t) - TmP                 [positive inside pool, negative outside]
```

---

## Numerical Method

| Aspect | Choice |
|--------|--------|
| Spatial discretisation | Finite Element Method (FEM) |
| Element type | P1 (linear nodal) for all fields, P0 (constant per element) for material properties |
| Time integration | Implicit Euler (unconditionally stable) |
| Matrix assembly | varf + A^-1 * b strategy |
| LHS matrix | Assembled ONCE before time loop (geometry and k, rho, cp are fixed) |
| RHS vector | Rebuilt every timestep (beam position xc changes) |
| Linear solver | Conjugate Gradient (CG) |
| Material assignment | Loop over element centroids, classify as substrate / particle / air |
| Mesh refinement | adaptmesh on particle layer before time loop |

The system matrix `A` is assembled only once since the geometry and material properties do not change during the scan. Only the RHS vector `b` is rebuilt each timestep to reflect the new beam position. This makes even large step counts computationally efficient.

---

## Output Fields

Each `.vtu` file contains the following fields:

| Field | Description | Range |
|-------|-------------|-------|
| `Temperature` | Nodal temperature | 300 to 1356 K |
| `MeltPool` | Instantaneous melt indicator | 0 (solid) or 1 (molten) |
| `MeltEver` | Cumulative melt track (HAZ) | 0 or 1 (latched) |
| `MeltDepth` | T minus TmP, signed distance to liquidus | negative to 0 |
| `gradTx` | Temperature gradient in x-direction | W/m |
| `gradTy` | Temperature gradient in y-direction | W/m |
| `HeatFlux` | Magnitude of thermal gradient vector | W/m2 |
| `Material` | Region indicator: 0=air, 1=particle, 2=substrate | 0 / 1 / 2 |

---

## Repository Structure

```
laser-powder-freefem/
|
|-- laser_powder.edp               # Main FreeFEM++ simulation script
|-- laser_powder_new/
|   |-- laser_powder.pvd           # ParaView collection file
|   |-- result_0004.vtu            # Timestep 4
|   |-- result_0008.vtu            # Timestep 8
|   |-- ...                        # Every 4 steps
|-- README.md
```

---

## How to Run

### Requirements

- FreeFEM++ v4.10 or later: https://freefem.org
- ParaView v5.x or later: https://www.paraview.org
- Output directory `D:\freefem++\` must exist before running

### Step 1 - Run the simulation

```bash
FreeFem++ laser_powder.edp
```

The script will:
1. Build the rectangular domain (substrate + particle layer + air)
2. Refine the mesh in the particle layer region
3. Assign material properties per element by centroid classification
4. Assemble the system matrix once
5. Run timesteps until beam reaches the right edge
6. Save VTU files every 4 steps to `D:\freefem++\laser_powder_new\`
7. Write `laser_powder.pvd` linking all timesteps

Console output per saved step:
```
Step 4/16667    t=0.0008 s   xbeam=0.0024 mm   Particle=1/10   Tmax=892.3 K    Melt=0 nodes
Step 8/16667    t=0.0016 s   xbeam=0.0048 mm   Particle=1/10   Tmax=1187.4 K   Melt=0 nodes
Step 40/16667   t=0.008  s   xbeam=0.024 mm    Particle=1/10   Tmax=1356.0 K   Melt=23 nodes
```

### Step 2 - Open in ParaView

1. `File > Open` > navigate to `D:\freefem++\laser_powder_new\`
2. Change Files of type to `All Files (*.*)`
3. Select `laser_powder.pvd` > OK
4. Choose PVD Reader when prompted > OK
5. Click `Apply`
6. Set colour field to `Material` first to verify geometry
7. Switch to `Temperature` and click `Rescale to Data Range Over All Timesteps`

### Step 3 - Visualize the melt pool

**Option A - Color by MeltPool (instant melt region)**
```
Colour field:   MeltPool
Colour map:     Blue-Red   (0=solid blue, 1=molten red)
Press Play
```

**Option B - Contour at solidus boundary (sharpest result)**
```
Filters > Contour
  Contour By:   MeltDepth
  Value:        0.0
  Apply
  Colour:       White
Press Play
```

**Option C - Full HAZ sintered track**
```
Colour field:   MeltEver
Shows everywhere that was ever molten as beam crosses all 10 particles
```

**Option D - Material layout verification**
```
Colour field:   Material
  0 = air gaps  (blue)
  1 = copper particles  (green)
  2 = steel substrate   (red)
```

Press `Play` to watch sequential particle melting as the beam scans left to right.

---

## What to Look for in Results

### Sequential Melting

Each copper particle melts as the beam centre passes over it. Watch the `MeltPool` field activate and deactivate particle by particle. The air gaps between particles act as thermal insulators, so each particle heats and cools relatively independently.

### Melt Pool Shape

Inside each particle, the melt pool is roughly elliptical. The pool widens near the top (direct laser exposure) and narrows toward the substrate contact zone. Pool depth depends on the ratio of beam power to scan speed.

### Heat Affected Zone (MeltEver)

After all 10 particles are processed, the `MeltEver` field shows the complete sintered track. This is the simulated powder-fused bead. Its continuity between particles depends on whether thermal diffusion bridges the air gap before the beam moves on.

### Temperature Gradient (gradTy)

Steep vertical gradients indicate rapid solidification rates, which correspond to fine microstructure in real L-PBF parts. Large horizontal gradients near the melt pool boundary drive Marangoni convection (not modelled here but identifiable by gradient magnitude).

---

## Common Errors and Fixes

| Error | Cause | Fix |
|-------|-------|-----|
| `End of String` lex error | Non-ASCII character in string | Use plain ASCII only, no dashes, arrows, or special symbols |
| `Compile error: )` in varf | `^` operator in boundary integral | Replace `r0^2` with `r0*r0`, `(x-xc)^2` with `(x-xc)*(x-xc)` |
| Folder not found | `D:\freefem++\` does not exist | Create `D:\freefem++\` manually; mkdir only creates the last level |
| All temperatures stay at 300 K | Power too low or beam too wide | Increase `P` or decrease `r0` |
| Melt never reaches particles | Beam hits top but particles are below | Increase `P`; particles are at `y = Ly` to `y = Ly + 2*Rp` |
| Air overheats | Air conductivity too high | Keep `kkA = 0.025` (real air value) |
| Mesh too coarse in particles | `hmin` too large | Decrease `hmin` in adaptmesh, increase `nbvx` |
| Wrong number of fields in savevtk | `ord` array length mismatch | Count fields in savevtk call, match `int[int] ord` length |

---

## Extending the Model

| Extension | What to change |
|-----------|----------------|
| Different powder material | Update rhoP, cpP, kkP, TmP |
| More particles | Increase `Np`; decrease `Rp` proportionally to keep them touching |
| Partial sintering | Reduce `P` until `Tmax` stays below `TmP` for some particles |
| Raster scan (back and forth) | Reverse `vbeam` sign after first scan completes |
| Multi-layer build | Shift geometry up by `2*Rp` and repeat scan |
| Latent heat of fusion | Add enthalpy smoothing near `TmP` using effective cp |
| Temperature-dependent k | Replace `kkP` with a piecewise function of T |
| Marangoni convection | Couple to incompressible Navier-Stokes in melt pool region |
| 3D model | Replace mesh with mesh3, use int3d, P13d elements |

---

## Author

**akshansh11**
GitHub: https://github.com/akshansh11

---

## License

<p>
<a rel="license" href="http://creativecommons.org/licenses/by-nc/4.0/">
<img alt="Creative Commons Licence" style="border-width:0" src="https://i.creativecommons.org/l/by-nc/4.0/88x31.png"/>
</a>
<br/>
This work is licensed under a
<a rel="license" href="http://creativecommons.org/licenses/by-nc/4.0/">Creative Commons Attribution-NonCommercial 4.0 International License</a>.
</p>

You are free to:

- **Share** - copy and redistribute the material in any medium or format
- **Adapt** - remix, transform, and build upon the material

Under the following terms:

- **Attribution** - You must give appropriate credit to akshansh11 and provide a link to this repository
- **NonCommercial** - You may not use the material for commercial purposes

Copyright 2026 akshansh11. All rights reserved for commercial use.
