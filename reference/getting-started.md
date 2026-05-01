# Getting Started with LAPS

This guide walks you from zero to running your first MHD simulation.

---

## Step 1: Understand What LAPS Is

LAPS (UCLA Pseudo-Spectral) is a **3D Hall-MHD simulation code** written in modern Fortran. It solves the magnetohydrodynamic equations on a periodic 3D grid using:

- **Fourier pseudo-spectral methods** — spatial derivatives via FFT
- **MPI parallelization** — domain decomposition across cores
- **SSPRK3 time integration** — 3rd-order strong-stability-preserving Runge-Kutta
- **Lanczos de-aliasing** — anti-aliasing filter for nonlinear terms

**What you can simulate with LAPS:**
- Alfvén wave propagation and collision
- MHD turbulence
- Solar wind dynamics (with expanding box model)
- Any system governed by incompressible or compressible Hall-MHD on a periodic domain

---

## Step 2: Install Prerequisites

### MPI (Message Passing Interface)

LAPS uses MPI for parallel computation across multiple CPU cores.

**macOS:**
```bash
brew install open-mpi
```

**Linux (Ubuntu/Debian):**
```bash
sudo apt-get install libopenmpi-dev openmpi-bin
```

**HPC clusters:** MPI is usually pre-loaded:
```bash
module load openmpi
```

### FFTW3 (Fastest Fourier Transform in the West)

LAPS uses FFTW3 for fast Fourier transforms — the core of the pseudo-spectral method.

**macOS:**
```bash
brew install fftw
```

**Linux:**
```bash
sudo apt-get install libfftw3-dev
```

**From source:**
```bash
wget www.fftw.org/fftw-3.3.10.tar.gz
tar xzf fftw-3.3.10.tar.gz
cd fftw-3.3.10
./configure --prefix=/usr/local
make && make install
```

### Fortran Compiler

- **gfortran** (GNU, free): `brew install gcc` on macOS, `sudo apt-get install gfortran` on Linux
- **ifort** (Intel, faster on Intel CPUs): requires license

---

## Step 3: Get the Code

```bash
git clone https://github.com/chenshihelio/LAPS
```

LAPS has multiple versions:

| Directory | What it is | When to use |
|-----------|-----------|-------------|
| `src_incompressible/` | 3D incompressible Hall-MHD | **Default for most research** |
| `src_incompressible/2D/` | 2D incompressible | Quick tests, 2D problems |
| `src_compressible/` | 3D compressible | When density variations matter |
| `src_compressible/2D/` | 2D compressible | 2D compressible tests |

---

## Step 4: Compile

```bash
cd LAPS/src_incompressible
```

### Edit the Makefile (if needed)

Two things you might need to change in `makefile`:

1. **FFTW path** — change the `-I` and `-L` flags to point to your FFTW install:
   ```makefile
   # macOS (Homebrew)
   OPTIONS = -O3 -fdefault-real-8 -I/opt/homebrew/include -L/opt/homebrew/lib -lfftw3

   # Linux (system FFTW)
   OPTIONS = -O3 -r8 -I/usr/include -L/usr/lib -lfftw3
   ```

2. **Compiler flag** for double precision:
   - `gfortran`: use `-fdefault-real-8`
   - `ifort`: use `-r8`

### Compile

```bash
make clean && make    # full rebuild
make                  # incremental (if only some files changed)
```

Output binary: **`mhd.exe`**

### Troubleshooting Compilation

| Error | Fix |
|-------|-----|
| `fftw3.h: No such file` | FFTW path wrong in makefile, or FFTW not installed |
| `mpif90: command not found` | MPI not installed or not in PATH |
| `-r8: unknown option` | Use `-fdefault-real-8` for gfortran |
| Undefined reference to `fftw_*` | Linker path wrong — check `-L` in makefile |

---

## Step 5: Run Your First Simulation

### Set Up a Run Directory

```bash
mkdir my_first_run
cp mhd.exe mhd.input my_first_run/
cd my_first_run
```

### Launch

```bash
mpirun -np 4 mhd.exe > rec &
```

- `-np 4`: use 4 MPI processes
- `> rec`: save terminal output to file `rec`
- `&`: run in background

### Monitor Progress

```bash
# Watch the log
tail -f rec

# Watch the energy diagnostics
tail -f rms.dat
```

You should see output like:
```
step=   10, time= 0.050, dt= 0.00500, max(div B)=  1.23E-14, max(div V)=  6.20E-02
```

This tells you: the simulation is at timestep 10, time 0.05, with small div B (good!) and normal div V.

---

## Step 6: Understand the Output

### rms.dat — Time Series

This file records diagnostics at every `dtrms` interval:
- **Time** — simulation time
- **Kinetic energy** (½ρu²) — total kinetic energy
- **Magnetic energy** (½B²) — total magnetic energy
- **div B** — divergence of magnetic field (should stay ~10⁻¹⁴)
- **div V** — divergence of velocity (~0.05 for incompressible)

### Snapshot Files

At every `dtout` interval, LAPS dumps the full 3D field to a snapshot file. The field array `uu` contains 8 components:

| Index | Field | Symbol | Meaning |
|-------|-------|--------|---------|
| 1 | Density | ρ | Mass density |
| 2 | Velocity x | Ux | x-component of velocity |
| 3 | Velocity y | Uy | y-component of velocity |
| 4 | Velocity z | Uz | z-component of velocity |
| 5 | Magnetic field x | Bx | x-component of B |
| 6 | Magnetic field y | By | y-component of B |
| 7 | Magnetic field z | Bz | z-component of B |
| 8 | Pressure | P | Thermal pressure |

### rec — Terminal Log

Contains MPI initialization info, timestep progress, and any error messages.

---

## Step 7: Verify Your Build

Compare your `rms.dat` with the standard test baseline:

| Metric | Expected | Meaning |
|--------|----------|---------|
| KE at t=5 | ~0.0111 | Kinetic energy (from initial ~0.0123) |
| ME at t=5 | ~0.0117 | Magnetic energy (from initial ~0.0125) |
| div B | ~3×10⁻¹⁴ | Divergence constraint (excellent) |
| div V | ~0.05 | Incompressibility (normal) |

If your numbers match, LAPS is working correctly!

---

## Next Steps

- **Configure parameters:** Read `reference/input-parameters.md`
- **Write custom initial conditions:** Read `reference/initial-conditions.md`
- **Understand the code:** Read `reference/architecture.md`
- **Analyze output:** Read `reference/output-and-analysis.md`
