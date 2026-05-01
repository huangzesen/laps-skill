# Analyzing LAPS Output

LAPS produces three types of output: diagnostic time series (`rms.dat`), field snapshots, and a terminal log (`rec`).

---

## Output Files

### rms.dat — Time Series Diagnostics

Written every `dtrms` time units. Space-separated text file with columns:

```
# time  KE  ME  divB_max  divV_max  ...
0.0   0.012334  0.012452  1.2e-14  0.062
0.1   0.012301  0.012420  2.1e-14  0.058
...
```

**What to monitor:**
- **Energy conservation:** KE + ME should stay roughly constant
- **Divergence control:** div B should stay at ~10⁻¹⁴
- **Dissipation rate:** rate of energy loss indicates numerical vs physical dissipation
- **Instability:** sudden energy growth → numerical instability

### Snapshot Files

Written every `dtout` time units. Contains the full 3D field `uu(ix,iy,iz,1:8)`:

| Index | Field |
|-------|-------|
| 1 | Density (ρ) |
| 2-4 | Velocity (Ux, Uy, Uz) |
| 5-7 | Magnetic field (Bx, By, Bz) |
| 8 | Pressure (P) |

### rec — Terminal Log

MPI initialization info, timestep progress, and error messages.

---

## Python Analysis

### Reading rms.dat

```python
import numpy as np
import matplotlib.pyplot as plt

# Load diagnostics
rms = np.loadtxt('rms.dat')
time = rms[:, 0]
ke = rms[:, 1]      # kinetic energy
me = rms[:, 2]      # magnetic energy
divB = rms[:, 3]    # max div B
```

### Plot Energy Evolution

```python
plt.figure(figsize=(10, 6))
plt.plot(time, ke, label='Kinetic Energy')
plt.plot(time, me, label='Magnetic Energy')
plt.plot(time, ke + me, '--', label='Total Energy')
plt.xlabel('Time')
plt.ylabel('Energy')
plt.legend()
plt.grid(True)
plt.savefig('energy_evolution.png', dpi=150)
```

### Check Divergence Stability

```python
plt.figure(figsize=(8, 5))
plt.semilogy(time, divB)
plt.xlabel('Time')
plt.ylabel('max |∇·B|')
plt.title('Divergence of B (should stay ~10⁻¹⁴)')
plt.grid(True)
plt.savefig('divB_stability.png', dpi=150)
```

### Compute Energy Decay Rate

```python
from scipy.stats import linregress

mask = time > 1.0  # skip initial transient
slope, intercept, r, p, se = linregress(time[mask], np.log(ke[mask]))
print(f"KE decay rate: {slope:.4f} per time unit")
```

### Reading Snapshots

The exact format depends on LAPS version. Typically:

```python
# Read binary snapshot (check your LAPS version for format)
import numpy as np

# Example: reading a Fortran unformatted binary file
with open('snapshot_001.dat', 'rb') as f:
    # Skip Fortran record markers (implementation-dependent)
    data = np.fromfile(f, dtype=np.float64)

# Reshape to 3D field
nx, ny, nz = 32, 32, 32
rho = data[0:nx*ny*nz].reshape(nx, ny, nz)
ux = data[nx*ny*nz:2*nx*ny*nz].reshape(nx, ny, nz)
# ... etc for each field component
```

**Note:** Check the `data_process/` directory for sample Python scripts included with LAPS.

---

## Publication-Quality Plots

For publication figures, typical analyses include:

### 1. Energy Profiles
- Kinetic vs magnetic energy evolution
- Total energy conservation check
- Energy decay rate comparison (SSPRK3 vs SSPRK4)

### 2. Field Morphology
- Cross-sections of Bx, By, Bz at selected times
- Velocity field streamlines
- Density contours

### 3. Fourier Spectra
- Energy spectrum E(k) = sum over shell in k-space
- Compensated spectrum k^α E(k)
- Check for Kolmogorov scaling (k^{-5/3}) or other turbulence scalings

### 4. Diagnostic Time Series
- div B and div V evolution
- Maximum field values
- Mode amplitude evolution

---

## Diagnostic Thresholds

| Metric | Good | Warning | Bad |
|--------|------|---------|-----|
| div B | ~10⁻¹⁴ | 10⁻¹² | > 10⁻¹⁰ |
| div V | ~0.05 | > 0.1 | > 0.5 |
| Energy conservation | < 1% per t-unit | 1-5% | > 5% |
| KE decay (inviscid) | Should be ~0 | -10% over t=5 | > -20% |
