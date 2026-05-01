# Input Parameters (mhd.input)

LAPS reads all configuration from a single file: **`mhd.input`**. This is a Fortran namelist file — each block starts with `&name` and ends with `/`.

---

## Complete Parameter Reference

### &genr — General Settings

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `tmax` | real | 5.0 | Simulation end time (in code units) |
| `dtout` | real | 1.0 | How often to write full field snapshots |
| `dtrms` | real | 0.1 | How often to write RMS diagnostics to rms.dat |

**Tips:**
- Set `dtout` large for long runs (snapshots are big files)
- `dtrms=0.1` gives good time resolution for energy plots

---

### &numerical — Numerical Settings

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `cfl` | real | 0.5 | CFL constant — determines dt from grid spacing |
| `dealias_option` | int | 2 | De-aliasing method: 1=spherical, **2=Lanczos** |
| `afx, afy, afz` | real | 0.495 | Lanczos filter factors (0.5=off, lower=more aggressive) |

**Physics context:**
- The pseudo-spectral method computes derivatives via FFT. Nonlinear terms (u·∇u, J×B) create aliasing — energy leaking into wrong wavenumbers. The Lanczos filter suppresses this.
- `af=0.495` is a gentle filter; `af=0.3` would be much more aggressive (more dissipation)
- CFL condition: `dt = cfl × dx / max_speed`. Smaller `cfl` = more stable but slower.

---

### &prl — Parallelization

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `ndim_parallel` | int | 2 | Parallelization dimensionality: 1 or 2 |

**Tips:**
- `ndim_parallel=2` (pencil decomposition) scales better for large grids
- Number of MPI processes must divide the grid evenly in each parallel dimension
- For 2D parallelization with 32 points in each direction: 4, 9, 16 processes work

---

### &grid — Grid

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `nx, ny, nz` | int | 32 | Number of grid points in each direction |
| `Lx, Ly, Lz` | real | 5.0 | Domain size [0, Lx] × [0, Ly] × [0, Lz] |

**Physics context:**
- The domain is periodic in all three directions
- Grid spacing: dx = Lx/nx, dy = Ly/ny, dz = Lz/nz
- Resolution determines the smallest scales you can resolve: k_max = π/dx
- Common choices: 32³ (testing), 64³ (exploration), 128³ (publication), 256³+ (high-res)

---

### &field — Background Magnetic Field

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `ifield` | int | 3 | Background field case number (selects initial condition) |
| `bx0, by0, bz0` | real | 1.0, 0, 0 | Background magnetic field components |
| `press0` | real | 0.5 | Background pressure |

**Physics context:**
- The background field defines the equilibrium state — a uniform magnetic field plus uniform pressure
- `ifield` selects which subroutine in `mhdinit.f90` to call for the background
- For Alfvén wave studies: set B along one axis (e.g., bx0=1) and propagate waves along it

---

### &pert — Perturbation

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `ipert` | int | 7 | Perturbation case number (selects perturbation type) |
| `db0, dv0` | real | 0.05 | Magnetic and velocity perturbation amplitudes |
| `nmodex, nmodey, nmodez` | int | 8 | Number of Fourier modes in perturbation |
| `drho0` | real | 0.01 | Density perturbation amplitude |

**Physics context:**
- Perturbations are added on top of the background field
- `ipert` selects the perturbation shape (wave mode, polarization)
- `db0` controls magnetic perturbation strength: db0/B0 gives the relative amplitude
- For linear Alfvén wave studies: use small db0 (e.g., 0.01)
- For nonlinear collision studies: use larger db0 (e.g., 0.1–0.5)

---

### &phys — Physics (Dissipation)

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `adiabatic_index` | real | 1.5 | γ — ratio of specific heats |
| `if_resis` | logical | T | Enable magnetic resistivity |
| `resistivity` | real | 1e-5 | Magnetic resistivity η (units: code units) |
| `if_visc` | logical | T | Enable kinematic viscosity |
| `viscosity` | real | 1e-5 | Kinematic viscosity ν |

**Physics context:**
- Resistivity η causes magnetic diffusion: ∂B/∂t = ... + η∇²B
- Viscosity ν causes momentum diffusion: ∂u/∂t = ... + ν∇²u
- Together they provide physical dissipation at small scales
- η = ν = 1e-5 is low dissipation — good for turbulence studies
- Setting `if_resis=F` and `if_visc=F` removes all dissipation (inviscid run)

---

### &AEB — Expanding Box Model (Optional)

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `if_AEB` | logical | F | Enable the expanding box model |
| `radius0` | real | 30.0 | Initial radial distance (solar radii) |
| `Ur0` | real | 1.167 | Expansion velocity |

**Physics context:**
- The expanding box model (EBM) simulates the effect of solar wind expansion on wave evolution
- When `if_AEB=T`, the code evolves in a comoving frame that expands radially
- Used for studying Alfvén waves in the expanding solar wind
- Leave `if_AEB=F` for standard MHD simulations

---

### &Hall — Hall Term (Optional)

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `if_Hall` | logical | F | Enable the Hall term in the induction equation |
| `ion_inertial_length` | real | 0 | Ion inertial length d_i |

**Physics context:**
- The Hall term adds dispersive effects at ion kinetic scales: ∂B/∂t = ∇×(u×B - d_i J×B/ρ)
- `ion_inertial_length` = d_i = c/ω_pi — the scale below which ions and electrons decouple
- Leave `if_Hall=F` for standard MHD (no Hall physics)

---

## Example Configurations

### Standard Alfvén Wave Test (32³)
```
&genr tmax=5.0, dtout=1.0, dtrms=0.1 /
&numerical cfl=0.5, dealias_option=2, afx=0.495, afy=0.495, afz=0.495 /
&prl ndim_parallel=2 /
&grid nx=32, ny=32, nz=32, Lx=5.0, Ly=5.0, Lz=5.0 /
&field ifield=3, bx0=1.0, by0=0.0, bz0=0.0, press0=0.5 /
&pert ipert=7, db0=0.05, dv0=0.05, nmodex=8, nmodey=8, nmodez=8, drho0=0.01 /
&phys adiabatic_index=1.5, if_resis=T, resistivity=1e-5, if_visc=T, viscosity=1e-5 /
&AEB if_AEB=F /
&Hall if_Hall=F /
```

### High-Resolution Run (128³)
Same as above but change `nx=128, ny=128, nz=128` and increase `cfl=0.4` for safety.

### Hall-MHD Run
Same as standard but add `if_Hall=T, ion_inertial_length=0.1` in `&Hall`.

---

## How Parameters Connect to Physics

```
Grid resolution  →  smallest scale you can resolve
Lanczos filter   →  controls aliasing (numerical noise from nonlinear terms)
CFL + grid       →  determines timestep dt
Resistivity η    →  magnetic diffusion scale: L_η = sqrt(η × L / V_A)
Viscosity ν      →  viscous diffusion scale: L_ν = sqrt(ν × L / V_A)
Hall term        →  dispersive effects below ion inertial length d_i
EBM              →  simulates solar wind expansion
```
