# Debugging LAPS

## Common Failures and Solutions

### 1. Segmentation Fault (segfault)

**Symptom:** Program crashes immediately or shortly after start with "Segmentation fault"

**Likely causes:**
- Array dimension mismatch between `mhdinit.f90` and other modules
- FFTW plan creation failure (wrong grid size)
- MPI initialization failure

**Diagnostic:**
```bash
# Check array dimensions match
grep "dimension" LAPS/src_incompressible/mhdinit.f90
grep "dimension" LAPS/src_incompressible/rktmod.f90

# Check if binary exists and is executable
ls -la mhd.exe
file mhd.exe
```

**Solution:**
- Ensure all dimension changes are consistent (e.g., if changing SSPRK3→4, all arrays must agree)
- Run with a single process first: `mpirun -np 1 mhd.exe`

---

### 2. Energy Blow-Up (Numerical Instability)

**Symptom:** Energy grows exponentially in rms.dat; simulation diverges

**Likely causes:**
- `dt` too large (CFL violation)
- Grid resolution too coarse for the physics
- De-aliasing not working properly

**Diagnostic:**
```bash
# Check CFL
grep "cfl" mhd.input
# Check dt in log
tail -20 rec
# Check energy history
head -50 rms.dat
tail -50 rms.dat
```

**Solution:**
- Reduce `cfl` (try 0.3 or 0.2)
- Increase grid resolution (32³ → 64³)
- Check `afx, afy, afz` values (0.495 is standard)
- If using SSPRK4, revert to SSPRK3 first to isolate the issue

---

### 3. div B Too Large (> 10⁻¹²)

**Symptom:** Divergence diagnostic shows divB >> 10⁻¹⁴

**Likely causes:**
- Spectral method numerical errors accumulating
- Grid resolution too coarse
- De-aliasing not working properly

**Diagnostic:**
```bash
grep "divB" rms.dat | head -20
grep "divB" rms.dat | tail -20
```

**Solution:**
- Increase grid resolution (32³ → 64³)
- Check if `dealias_option` is set correctly (2 = Lanczos)
- Verify resistivity is not too small (1e-5 is safe)
- Check that initial condition is compatible with spectral method (smooth, periodic)

---

### 4. Boundary Artifacts

**Symptom:** Anomalous field structures near domain boundaries

**Likely causes:**
- Periodic FFT assumes periodic boundary conditions
- Initial condition not periodic-friendly
- Referee R1 (Feb 2026) flagged this specifically

**Diagnostic:**
```bash
# Check if perturbation is compatible with periodicity
grep "ipert" mhd.input
# Check box size vs perturbation wavelength
grep "Lx" mhd.input
```

**Solution:**
- Ensure perturbation wavelength fits exactly in the box: L/λ = integer
- Smooth initial conditions near boundaries
- Consider larger box (Lx = 10 or 20)

---

### 5. Compilation Errors

**Symptom:** `make` fails

**Common errors:**

| Error | Cause | Fix |
|-------|-------|-----|
| `fftw3.h: No such file` | FFTW path wrong | Fix `-I` path in makefile |
| `mpif90: command not found` | MPI not in PATH | `module load openmpi` or install |
| `Error: Array dimension` | Size mismatch | Check all `dimension()` declarations |
| `-r8: unknown option` | Wrong compiler | Use `-fdefault-real-8` for gfortran |
| `Undefined reference to fftw` | Linker error | Fix `-L` path in makefile |

---

### 6. MPI Communication Errors

**Symptom:** "Fatal error in MPI" or hang/deadlock

**Likely causes:**
- Mismatch between `ndim_parallel` and available cores
- Core count not compatible with domain decomposition

**Solution:**
- For `ndim_parallel = 2`: number of processes must be a perfect square (4, 9, 16, 25, ...)
- For `ndim_parallel = 1`: any number works
- Start debugging with `mpirun -np 1`

---

### 7. Slow Performance

**Symptom:** Simulation much slower than expected

**Diagnostic:**
```bash
# Check how many cores are actually used
mpirun -np 8 mhd.exe > rec &
head -20 rec   # should show MPI decomposition info

# Check if FFTW wisdom exists
ls *.wisdom 2>/dev/null
```

**Solution:**
- Use `ndim_parallel = 2` for better scaling
- Generate FFTW wisdom: pre-compute optimal FFT plans
- Check that `-O3` optimization is enabled in makefile
- Ensure MPI is using fast interconnect (not TCP)

---

## Diagnostic Thresholds

| Metric | Good | Warning | Bad |
|--------|------|---------|-----|
| div B | ~10⁻¹⁴ | 10⁻¹² | > 10⁻¹⁰ |
| div V | ~0.05 | > 0.1 | > 0.5 |
| Energy conservation | < 1% per t-unit | 1-5% | > 5% |
| Kinetic energy decay | -10% over t=5 | -15% | > -20% |

---

## Recovery Procedures

### Simulation crashed mid-run

1. Check `rec` log for error messages
2. Check if restart files were written: `ls restart*.dat`
3. If restart available: set restart flag in mhd.input and relaunch
4. If no restart: reduce `cfl`, increase grid, or simplify physics

### Need to stop and resume

```bash
# Send interrupt signal (writes restart file)
kill -INT $(pgrep mhd.exe)

# Wait for restart file to be written
ls -la restart*.dat

# Resume (modify mhd.input to enable restart)
# Then relaunch with same mpirun command
```

---

## Quick Diagnostic Commands

```bash
# Energy evolution
awk '{print $1, $2, $3}' rms.dat | head -20

# Divergence evolution
awk '{print $1, $4}' rms.dat | head -20

# Check simulation is running
ps aux | grep mhd.exe

# Monitor output
tail -f rec

# Check grid resolution
grep "nx" mhd.input
```
