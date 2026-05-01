# Writing Custom Initial Conditions

LAPS lets you define custom initial conditions by editing **`mhdinit.f90`**. This is typically the only Fortran file you need to modify.

---

## How Initialization Works

When LAPS starts, it calls two subroutines in `mhdinit.f90`:

1. **`background_fields_initialize`** — sets the equilibrium (background) state
2. **`perturbation_initialize`** — adds perturbations on top

The `ifield` and `ipert` parameters in `mhd.input` select which case to use in each subroutine.

---

## The Main Field Array: uu

All fields are stored in a single 4D array:

```fortran
uu(ixmin:ixmax, iymin:iymax, izmin:izmax, 1:8)
```

| Index | Field | Symbol | Units |
|-------|-------|--------|-------|
| 1 | Density | ρ | code density units |
| 2 | Velocity x | Ux | code velocity units |
| 3 | Velocity y | Uy | code velocity units |
| 4 | Velocity z | Uz | code velocity units |
| 5 | Magnetic field x | Bx | code magnetic units |
| 6 | Magnetic field y | By | code magnetic units |
| 7 | Magnetic field z | Bz | code magnetic units |
| 8 | Pressure | P | code pressure units |

**Important:** At initialization, `uu` uses **regular MHD fields** (not conserved quantities like momentum ρu). This is for convenience — the code converts internally.

---

## Spatial Coordinates

The grid coordinates are available as 1D arrays:

```fortran
xgrid(ixmin:ixmax)  ! x coordinates
ygrid(iymin:iymax)  ! y coordinates
zgrid(izmin:izmax)  ! z coordinates
```

The domain is **[0, Lx] × [0, Ly] × [0, Lz]**, defined by `Lx, Ly, Lz` in `mhd.input`.

---

## Index Ranges

The six index variables define which part of the array each MPI process owns:

```fortran
ixmin, ixmax  ! x-index range for this process
iymin, iymax  ! y-index range for this process
izmin, izmax  ! z-index range for this process
```

These are set by the MPI decomposition — each process has its own slice of the 3D grid.

---

## Adding a New Background Field Case

To add `ifield=10` as a new background:

1. Open `mhdinit.f90`
2. Find `subroutine background_fields_initialize`
3. Add a new case in the `ifield` selector:

```fortran
case(10)
    ! Your custom background: e.g., shear flow + uniform B
    do iz = izmin, izmax
    do iy = iymin, iymax
    do ix = ixmin, ixmax
        uu(ix,iy,iz,1) = 1.0          ! uniform density
        uu(ix,iy,iz,2) = tanh(ygrid(iy))  ! shear flow in x
        uu(ix,iy,iz,3) = 0.0
        uu(ix,iy,iz,4) = 0.0
        uu(ix,iy,iz,5) = 1.0          ! uniform Bx
        uu(ix,iy,iz,6) = 0.0
        uu(ix,iy,iz,7) = 0.0
        uu(ix,iy,iz,8) = 0.5          ! uniform pressure
    enddo
    enddo
    enddo
```

4. Set `ifield=10` in `mhd.input`

---

## Adding a New Perturbation Case

To add `ipert=20` as a new perturbation:

1. Open `mhdinit.f90`
2. Find `subroutine perturbation_initialize`
3. Add a new case:

```fortran
case(20)
    ! Custom perturbation: circularly polarized Alfvén wave
    do iz = izmin, izmax
    do iy = iymin, iymax
    do ix = ixmin, ixmax
        kx = 2.0 * pi / Lx
        phase = kx * xgrid(ix)
        uu(ix,iy,iz,3) = uu(ix,iy,iz,3) + db0 * sin(phase)  ! velocity perturbation
        uu(ix,iy,iz,6) = uu(ix,iy,iz,6) + db0 * sin(phase)  ! magnetic perturbation
        uu(ix,iy,iz,4) = uu(ix,iy,iz,4) + db0 * cos(phase)  ! circular polarization
        uu(ix,iy,iz,7) = uu(ix,iy,iz,7) + db0 * cos(phase)
    enddo
    enddo
    enddo
```

4. Set `ipert=20` in `mhd.input`

---

## Adding Custom Namelist Parameters

If your initial condition needs extra parameters (e.g., a shear width):

1. In `mhd.f90`, find the `&pert` namelist definition and add your variable:

```fortran
namelist /pert/ ipert, db0, dv0, nmodex, nmodey, nmodez, drho0, shear_width
```

2. Declare the variable at the top of `mhd.f90`
3. In `mhdinit.f90`, add `use` the module or pass it via a common block
4. Add `shear_width=0.5` (or whatever) to your `mhd.input`

---

## Common Patterns

### Uniform Background
```fortran
uu(ix,iy,iz,1) = 1.0    ! rho
uu(ix,iy,iz,5) = B0      ! Bx
uu(ix,iy,iz,8) = press0  ! P
```

### Alfvén Wave Perturbation (parallel propagation)
```fortran
phase = k * (xgrid(ix) - vA * 0)  ! k·x at t=0
uu(ix,iy,iz,3) = uu(ix,iy,iz,3) + amp * sin(phase)  ! Uy
uu(ix,iy,iz,6) = uu(ix,iy,iz,6) + amp * sin(phase)  ! By
```

### Two Counter-Propagating Waves (collision setup)
```fortran
phase1 = k * xgrid(ix)
phase2 = -k * xgrid(ix)
uu(ix,iy,iz,3) += amp * sin(phase1) + amp * sin(phase2)
uu(ix,iy,iz,6) += amp * sin(phase1) + amp * sin(phase2)
```

---

## Verification Checklist

After writing a custom initial condition:

1. **Compile:** `make clean && make`
2. **Run a short test:** `tmax=0.1, dtrms=0.01`
3. **Check div B at t=0:** Should be ~10⁻¹⁴ or smaller
4. **Check energy at t=0:** Should match your expected initial energy
5. **Check symmetry:** If your IC has symmetry, verify it's preserved at t=0
