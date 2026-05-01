---
name: laps
description: >
  Self-contained guide to LAPS (Large Plasma Simulation) — a 3D MPI-parallelized
  pseudo-spectral Hall-MHD Fortran code by Dr. Chen Shi. For agents and humans who
  want to run MHD simulations: install LAPS, configure parameters, write initial
  conditions, launch MPI runs, and analyze output. Progressive disclosure — start
  here for routing, drill into reference/ for depth.
version: 6.0.0
tags:
  - physics
  - fortran
  - mpi
  - mhd
  - plasma
  - simulation
  - alfven
  - hall-mhd
  - pseudo-spectral
---

# LAPS — Complete Guide

> **LAPS** = **UCLA** **P**seudo-**S**pectral Hall-MHD code
> Author: Dr. Chen Shi (cshi1993@ucla.edu)
> GitHub: https://github.com/chenshihelio/LAPS
> Papers: [Shi et al. 2020, ApJ 888:68](https://iopscience.iop.org/article/10.3847/1538-4357/ab5fce) · [Shi et al. 2024, Frontiers 11:1412905](https://www.frontiersin.org/articles/10.3389/fspas.2024.1412905/full)

**What LAPS does:** Simulates magnetohydrodynamic (MHD) phenomena — Alfvén wave collisions, solar wind dynamics, plasma turbulence — using incompressible or compressible Hall-MHD, parallelized with MPI, solved via Fourier pseudo-spectral methods on a 3D periodic grid.

**Who this skill is for:** Agents and researchers who want to understand, run, configure, and analyze LAPS simulations — even if they are new to MHD codes.

---

## Quick Decision Tree

```
"What do I need?"
│
├─ 🆕 First time — what is LAPS and how do I set it up?
│  └─ Read: reference/getting-started.md
│     (prerequisites, install, compile, first run — step by step)
│
├─ 🚀 I want to run a simulation
│  ├─ How to configure mhd.input (the parameter file)
│  │  └─ Read: reference/input-parameters.md
│  │     (every parameter explained with physics context)
│  └─ How to write custom initial conditions
│     └─ Read: reference/initial-conditions.md
│        (background fields, perturbations, the uu array)
│
├─ 🔬 I want to understand how LAPS works internally
│  ├─ Code architecture — what each file does
│  │  └─ Read: reference/architecture.md
│  ├─ Numerical methods — pseudo-spectral, RK, dealiasing
│  │  └─ Read: reference/architecture.md § "Numerical Methods"
│  └─ How div B = 0 is maintained (spectral constraint)
│     └─ Read: reference/architecture.md § "Divergence-Free Constraint"
│
├─ 📊 I have output — how do I analyze it?
│  └─ Read: reference/output-and-analysis.md
│     (rms.dat format, snapshots, Python readers, plots)
│
├─ ⬆️ I want to improve accuracy (SSPRK4 upgrade)
│  └─ Read: reference/ssprk4-upgrade.md
│     (coefficients, file-by-file changes, verification)
│
├─ 🐛 Something went wrong
│  └─ Read: reference/debugging.md
│     (segfaults, blow-ups, divergence, slow runs)
│
└─ 📝 What is the research context?
   └─ Read: reference/research-context.md
      (manuscript status, referee feedback, open threads)
```

---

## What's Inside LAPS

```
LAPS/
├── src_incompressible/        ← 3D incompressible Hall-MHD (main version)
│   ├── mhd.f90                ← Main driver: time loop, I/O, MPI init
│   ├── mhdinit.f90            ← YOU EDIT THIS: initial conditions
│   ├── mhdrhs.f90             ← Right-hand side (forces, advection)
│   ├── rktmod.f90             ← Runge-Kutta time integration coefficients
│   ├── mhdoutput.f90          ← Snapshot output
│   ├── mhdrms.f90             ← Energy & divergence diagnostics
│   ├── parallel.f90           ← MPI domain decomposition
│   ├── fftw.f90               ← FFT wrapper (FFTW3)
│   ├── dealiasing.f90         ← Lanczos filtering (anti-aliasing)
│   ├── AEBmod.f90             ← Expanding box model (optional)
│   ├── restart.f90            ← Checkpoint/restart
│   ├── makefile               ← Build system
│   └── mhd.input              ← Parameter file (Fortran namelist)
│
├── src_compressible/          ← 3D compressible version
├── src_incompressible/2D/     ← 2D incompressible version
├── data_process/              ← Data processing utilities
├── README.md                  ← Original README
└── LICENSE.txt                ← GPL v3
```

**For most users:** You only need `src_incompressible/`. The compressible version is a separate codebase with its own physics.

---

## Critical Rules

1. **NEVER modify Fortran code without human approval.** State the proposed change, explain why, and wait.
2. **Report before long simulations.** State expected runtime and resource needs.
3. **If LAPS source is missing:** `git clone https://github.com/chenshihelio/LAPS`
4. **When in doubt, grep first** — search the source before asking questions.

---

## Quick Start (5 commands)

```bash
# 1. Get LAPS
git clone https://github.com/chenshihelio/LAPS

# 2. Install dependencies (macOS)
brew install fftw open-mpi

# 3. Compile
cd LAPS/src_incompressible
make clean && make

# 4. Run standard test
mpirun -np 4 mhd.exe > rec &

# 5. Watch the energy
tail -f rms.dat
```

→ Full walkthrough in `reference/getting-started.md`

---

## Standard Test Baseline

Verify your build is correct with these settings (default `mhd.input`):

| Parameter | Value | Where |
|-----------|-------|-------|
| Grid | 32³ | `&grid` nx/ny/nz |
| Box size | 5.0 | `&grid` Lx/Ly/Lz |
| Perturbation | ipert=7, db0=dv0=0.05 | `&pert` |
| Filter factor | af=0.495 | `&numerical` afx/afy/afz |
| End time | 5.0 | `&genr` tmax |

**Expected at t=5:**
- Kinetic energy: ~0.0111 (from initial ~0.0123, ≈ -10.2%)
- Magnetic energy: ~0.0117 (from initial ~0.0125, ≈ -6.3%)
- div B: ~3×10⁻¹⁴ (excellent)
- div V: ~0.05 (normal)

---

## Reference Documents

| Document | What's inside | Lines |
|----------|--------------|-------|
| `reference/getting-started.md` | Step-by-step: prerequisites → install → compile → first run → understand output | ~150 |
| `reference/input-parameters.md` | Every `mhd.input` parameter with physics meaning, defaults, and tips | ~250 |
| `reference/initial-conditions.md` | How to write custom ICs: the uu array, background fields, perturbations | ~200 |
| `reference/architecture.md` | Code internals: all 11 .f90 files, numerical methods, Helmholtz, parallelization | ~250 |
| `reference/output-and-analysis.md` | Output formats, rms.dat, snapshots, Python readers, publication plots | ~200 |
| `reference/ssprk4-upgrade.md` | SSPRK4 implementation: coefficients, file changes, verification protocol | ~150 |
| `reference/debugging.md` | 7 common failures, diagnostic thresholds, recovery procedures | ~200 |
| `reference/research-context.md` | Manuscript status, referee feedback, open research threads | ~80 |
