# LAPS Skill

A **LingTai progressive-disclosure skill** for [LAPS](https://github.com/chenshihelio/LAPS) — a 3D MPI-parallelized pseudo-spectral Hall-MHD Fortran code for MHD simulation.

> **Author of LAPS:** Dr. Chen Shi (cshi1993@ucla.edu)
> **Skill version:** 6.0.0

## What This Is

A self-contained knowledge package that teaches AI agents (and humans) how to install, configure, run, and analyze MHD simulations with LAPS. Covers everything from prerequisites to publication-quality output analysis.

**Progressive disclosure:**
- `SKILL.md` — routing hub: decision tree + quick start + test baseline
- `reference/*.md` — deep-dive docs loaded on demand

## Contents

| Document | What's inside |
|----------|---------------|
| `SKILL.md` | Entry point — decision tree, directory layout, quick start |
| `reference/getting-started.md` | Step-by-step: prerequisites → install → compile → first run |
| `reference/input-parameters.md` | Every `mhd.input` parameter with physics meaning |
| `reference/initial-conditions.md` | How to write custom ICs: the uu array, background fields, perturbations |
| `reference/architecture.md` | Code internals: pseudo-spectral method, RK time integration, parallelization |
| `reference/output-and-analysis.md` | Output formats, Python readers, publication plots |
| `reference/ssprk4-upgrade.md` | SSPRK4 implementation plan |
| `reference/debugging.md` | Common failures, diagnostic thresholds, recovery |
| `reference/research-context.md` | Manuscript status, referee feedback |

## Install in LingTai

```bash
cp -r . /path/to/project/.lingtai/.library_shared/custom/laps
```

Then refresh: `system({"action": "refresh"})`.

## License

MIT
