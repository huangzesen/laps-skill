# LAPS Code Architecture

This document explains how the LAPS Fortran code is organized and how its numerical methods work.

---

## Source File Map

```
LAPS/src_incompressible/
├── mhd.f90              Main driver — time loop, I/O, MPI init
├── mhdinit.f90          Field initialization (background + perturbation)
├── mhdrhs.f90           Right-hand side calculations (forces, advection)
├── rktmod.f90           Runge-Kutta time integration coefficients
├── mhdoutput.f90        Snapshot and diagnostic output
├── mhdrms.f90           RMS diagnostics (energy, divB, divV)
├── parallel.f90         MPI domain decomposition
├── fftw.f90             FFT wrapper using FFTW3
├── dealiasing.f90       Lanczos filtering (anti-aliasing)
├── AEBmod.f90           Expanding box model (optional)
├── restart.f90          Checkpoint/restart functionality
├── makefile             Build system
└── mhd.input            Parameter file (Fortran namelist)
```

### Dependency Chain (compilation order)

```
parallel.f90 → mhdinit.f90 → {fftw.f90, mhdoutput.f90, rktmod.f90, dealiasing.f90,
                                mhdrms.f90, AEBmod.f90} → {mhdrhs.f90, restart.f90} → mhd.f90
```

---

## How a Simulation Runs

The main driver `mhd.f90` orchestrates everything:

```
1. Read mhd.input (namelist parameters)
2. Initialize MPI (parallel.f90)
3. Set up grid and FFT plans (fftw.f90)
4. Initialize fields (mhdinit.f90)
5. Initialize RK coefficients (rktmod.f90)
6. Time loop:
   do istep = 1, nsteps
     do irk = 1, 3                          ← SSPRK3: 3 stages
       transform_uu_real_to_fourier          ← physical → spectral space
       calc_rhs (mhdrhs.f90)                 ← compute forces + advection
       dealias (dealiasing.f90)              ← apply Lanczos filter
       time_advance                          ← RK stage update
     end do
     calc_divB_real, calc_divV_real          ← diagnostic: check divergence
     output diagnostics (mhdrms.f90, mhdoutput.f90)
   end do
7. Finalize MPI
```

---

## Numerical Methods

### Pseudo-Spectral Method

LAPS solves the MHD equations in **Fourier (spectral) space**:

1. Fields live in real space as `uu(ix,iy,iz,1:8)`
2. Each timestep, fields are transformed to Fourier space via FFT
3. Spatial derivatives become **multiplications by ik** in Fourier space:
   - ∂f/∂x → ik_x · f̂
   - ∇²f → -k² · f̂
4. Nonlinear terms (u·∇u, J×B) are computed in real space, then transformed back
5. This is the "pseudo-spectral" approach — linear terms in Fourier space, nonlinear in real space

**Advantages:** Exponential convergence for smooth periodic functions
**Limitation:** Requires periodic boundary conditions

### Time Integration: SSPRK3

LAPS uses **3rd-order Strong-Stability-Preserving Runge-Kutta** (SSPRK3):

```fortran
! Van der Houwen/Wray coefficients
stage 1: u^(1) = u^(n) + (8/15)·dt·RHS(u^(n))
stage 2: u^(2) = u^(1) + (2/15)·dt·RHS(u^(1))
stage 3: u^(n+1) = u^(2) + (1/3)·dt·RHS(u^(2))
```

Each stage calls: FFT → compute RHS → dealias → advance.

The coefficients are in `rktmod.f90`. The time loop is `do irk = 1, 3` in `mhd.f90`.

**Key properties:**
- 3rd-order accurate in time: O(Δt³)
- Strong-stability-preserving: does not violate physical bounds
- Each stage computes one full RHS (expensive but accurate)

### De-aliasing: Lanczos Filter

Nonlinear terms create aliasing — energy leaking into wrong wavenumbers. The Lanczos filter suppresses this:

- Controlled by `afx, afy, afz` in `mhd.input`
- `af = 0.5` → no filtering (maximum resolution)
- `af = 0.495` → gentle filtering (standard choice)
- `af = 0.3` → aggressive filtering (more dissipation)
- Applied in Fourier space after each RHS computation

The filter zeros out high-wavenumber components, preventing aliasing errors from corrupting the solution.

---

## Divergence-Free Constraint (div B = 0)

**How it works:** The Fortran code does **not** use an explicit projection step. Instead, div B ≈ 0 is maintained naturally by the pseudo-spectral method:

- In Fourier space, ∇·B becomes ik·B̂
- The induction equation preserves this structure because:
  - ∇·(∇×E) = 0 identically (curl is divergence-free)
  - The nonlinear term u×B is computed spectrally, preserving this property
- Small numerical errors accumulate over time, but stay at ~10⁻¹⁴ level

**Diagnostic monitoring:** The code computes div B and div V as **diagnostics** (not as correction steps):
- `calc_divB_real` in `mhdrhs.f90` — computes ∇·B for monitoring
- `calc_divV_real` in `mhdrhs.f90` — computes ∇·V for monitoring
- Results written to `rms.dat` at each `dtrms` interval

**Healthy values:**
- div B ~ 10⁻¹⁴ → excellent (spectral method working)
- div B ~ 10⁻¹² → acceptable
- div B > 10⁻¹² → warning: numerical errors accumulating
- div V ~ 0.05 → normal for incompressible MHD

---

## Right-Hand Side: mhdrhs.f90

This is where the physics lives. `calc_rhs` computes:

- **Nonlinear terms:**
  - Advection: (u·∇)u
  - Lorentz force: J×B (where J = ∇×B)
  - Magnetic advection: (u·∇)B - (B·∇)u
- **Linear terms:**
  - Pressure gradient (from incompressibility constraint)
  - Resistive diffusion: η∇²B
  - Viscous diffusion: ν∇²u
- **Hall term** (if enabled): ∇×(J×B/ρ)

All spatial derivatives are computed via FFT (pseudo-spectral method).

---

## Parallelization: parallel.f90

LAPS decomposes the 3D grid across MPI processes:

- **1D decomposition** (`ndim_parallel=1`): each process owns a slab in x
- **2D decomposition** (`ndim_parallel=2`): each process owns a pencil in x-y

FFT requires global communication — data must be transposed between decompositions. This is handled by MPI alltoall operations.

**Scaling:** LAPS scales well up to ~128 cores for 128³ grids. Beyond that, communication overhead grows.

---

## Initialization: mhdinit.f90

The main arrays:

```fortran
complex, allocatable :: uu(:,:,:,:)      ! main field array (spectral space)
complex, allocatable :: fnl(:,:,:,:)     ! nonlinear term storage
complex, allocatable :: fnl_rk(:,:,:,:)  ! RK history for time integration
```

The initialization process:
1. Set background fields in `background_fields_initialize`
2. Add perturbations in `perturbation_initialize`
3. Transform to Fourier space for the time loop

---

## Key Files Summary

| File | Role | Lines | You edit? |
|------|------|-------|-----------|
| `mhd.f90` | Main driver | ~600 | Rarely (for new namelist params) |
| `mhdinit.f90` | Initial conditions | ~1100 | **Yes — this is your main editing target** |
| `mhdrhs.f90` | Physics (forces, advection) | ~700 | Rarely (for new physics) |
| `rktmod.f90` | RK coefficients | ~50 | Only for SSPRK4 upgrade |
| `fftw.f90` | FFT wrapper | ~200 | No |
| `parallel.f90` | MPI decomposition | ~300 | No |
| `dealiasing.f90` | Lanczos filter | ~100 | No |
| `mhdoutput.f90` | Snapshot output | ~100 | No |
| `mhdrms.f90` | RMS diagnostics | ~200 | No |
| `AEBmod.f90` | Expanding box model | ~100 | No |
| `restart.f90` | Checkpoint/restart | ~200 | No |
