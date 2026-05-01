# Research Context & Manuscript Status

## Current Manuscript

**Title:** "Solitary Alfvén Waves"
**Journal:** The Astrophysical Journal Letters (AAS)
**Manuscript ID:** AAS71834R1
**Status:** Revision 1 (R1) submitted after referee report (Feb 2026)

---

## Referee R1 Feedback (Feb 2026)

The referee raised three main concerns:

### 1. Boundary Artifacts
- Periodic FFT-based initial conditions can cause edge effects
- Need to verify that artifacts do not affect bulk physics
- **Action:** Monitor boundary region energy; consider smoothing ICs

### 2. Iterative Method Convergence
- Helmholtz projection convergence needs better diagnostic control
- Both large-scale and small-scale convergence need verification
- **Action:** Implement convergence monitoring in Fortran code

### 3. Spatial Resolution
- May need higher resolution to confirm results
- **Action:** Run convergence study (32³ → 64³ → 128³)

---

## Open Research Threads

### Thread 1: SSPRK4 Upgrade
**Status:** Plan complete, awaiting human approval
**Goal:** Reduce numerical energy decay from -10.2% to -5%
**Reference:** `SSPRK4_Implementation_Report.md`

### Thread 2: Helmholtz Solver Improvement
**Status:** Python implementation improved (IMPROVEMENTS.md)
**Goal:** Better convergence control, boundary handling
**Next:** Port improvements to Fortran

### Thread 3: Convergence Study
**Status:** Started (some 128³ runs exist)
**Goal:** Demonstrate numerical convergence with resolution
**Runs needed:** 32³, 64³, 128³, possibly 256³

### Thread 4: Higher Resolution Runs
**Status:** Pending
**Goal:** Production-quality 128³ or 256³ runs for publication
**Compute requirement:** Significant (hours to days per run)

---

## Key Papers

1. **Shi, C. et al. (2020).** "Propagation of Alfvén waves in the expanding solar wind with the fast–slow stream interaction." *ApJ*, 888, 2, 68.
   - Original LAPS paper with expanding box model

2. **Shi, C. et al. (2024).** "LAPS: An MPI-parallelized 3D pseudo-spectral Hall-MHD simulation code incorporating the expanding box model." *Frontiers in Astronomy and Space Sciences*, 11, 1412905.
   - Comprehensive LAPS methodology paper

3. **Ruuth, S. J. & Spiteri, R. J. (2004).** "High-Order Strong-Stability-Preserving Runge-Kutta Methods with Down-Biased Spatial Discretizations." *SIAM J. Numer. Anal.*, 42(3), 974-996.
   - SSPRK4 coefficients

4. **Chorin, A. J. (1968).** "Numerical solution of the Navier-Stokes equations." *Math. Comp.*, 22, 745-762.
   - Helmholtz projection method

---

## Data Management

### Where simulation data lives:
- `analyze_nbs/diagnostics_*/` — diagnostic time series
- `analyze_nbs/collision_small_amp_256/` — 256³ run data
- `analyze_nbs/small_box_small_amp_128/` — 128³ run data
- `publication_plots/` — final figures

### Key data files:
- `rms.dat` — energy and diagnostic time series
- Snapshot files — full 3D field dumps
- `grid.dat` — grid coordinate information
