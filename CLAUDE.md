# CLAUDE.md — AI Assistant Guide for gromacs-py

## Project Overview

This is a **computational chemistry research project** for preparing and running molecular dynamics (MD) simulations of the **COVID-19 main protease (3C-like Proteinase, PDB ID: 6W63)** using [GROMACS](https://www.gromacs.org/). It is not an installable Python package — it is an executable scientific workflow delivered as a Jupyter notebook.

The protein studied is the SARS-CoV-2 main protease bound to the broad-spectrum non-covalent inhibitor X77, solved at 2.10 Å resolution by X-ray diffraction from an E. coli BL21(DE3) expression system.

---

## Repository Structure

```
gromacs-py/
├── data/                          # Input data and force field files (~18 MB)
│   ├── charmm36-jul2022.ff/       # CHARMM36 force field parameter files
│   ├── npt.tpr                    # Precomputed NPT binary trajectory (GROMACS)
│   ├── protein.pdb                # Original COVID-19 main protease PDB structure
│   └── protein_clean.pdb          # Cleaned/curated PDB file
├── dummy/                         # Intermediate working directory (gitignored intent)
│   ├── processed.gro              # GROMACS coordinate file (4,665 atoms)
│   ├── topol.top                  # Molecular topology (~1.3 MB)
│   ├── posre.itp                  # Heavy-atom position restraints
│   └── [#backup# files]           # GROMACS auto-backups from reruns
├── gmx_work/                      # Primary simulation output directory (~18 MB)
│   ├── processed.gro              # Final GROMACS coordinate file
│   ├── topol.top                  # Full system topology
│   └── posre.itp                  # Position restraints file
├── notebooks/
│   └── test-gromacs.ipynb         # Main workflow notebook (sole entry point)
├── .vscode/
│   └── settings.json              # VS Code / Jupyter environment settings
└── CLAUDE.md                      # This file
```

---

## Workflow Overview

The simulation pipeline is executed entirely through `notebooks/test-gromacs.ipynb`, organized in two phases:

### Phase 1: System Preparation

1. **PDB curation** — load and validate `protein_clean.pdb`
2. **Topology generation** — run `pdb2gmx` via gmxapi:
   - Force field: CHARMM36-jul2022
   - Water model: TIP3P
   - Outputs: `processed.gro` (4,665 atoms with H added), `topol.top`, `posre.itp`
3. **Validation** — inspect topology composition (pairs, dihedrals, atoms, residues, chains) and sulfur geometry matrix for disulfide bonds

### Phase 2: MD Simulation Execution

1. **Run `gmx mdrun`** on the precomputed NPT ensemble (reads `npt.tpr`)
2. **Outputs**: energy file (`.edr`), trajectory (`.trr`), final coordinates (`.gro`)
3. **Return code inspection** to verify successful completion

---

## Environment Setup

### Required Software

| Dependency | Version / Notes |
|------------|----------------|
| GROMACS    | 2025.4, installed at `/usr/local/gromacs/` |
| gmxapi     | Python GROMACS API (conda installable) |
| Python     | 3.x (modern pathlib / f-string usage) |
| Conda      | Environment named `md_ml` |
| Jupyter    | With `md_ml` kernel registered |

### GROMACS Paths

- Binary: `/usr/local/gromacs/bin/gmx`
- System force fields: `/usr/local/gromacs/share/gromacs/top/`
- The repository bundles its own `charmm36-jul2022.ff/` under `data/` for portability

### Conda Environment

The project uses the `md_ml` conda environment. VS Code and Jupyter are configured to use it automatically via `.vscode/settings.json`. No `requirements.txt` or `environment.yml` is present — recreate with:

```bash
conda create -n md_ml python=3.x
conda install -c conda-forge gmxapi jupyter
```

---

## Key Conventions

### Python Style

- Use `pathlib.Path` for all file paths — avoid raw string concatenation
- Resolve notebook-relative paths with `pathlib.Path(os.getcwd()).parent`
- Use `mkdir(exist_ok=True)` for idempotent directory creation
- Use assertions for early validation of file existence and return codes

### Path Resolution Pattern

```python
import os, pathlib
notebook_dir = pathlib.Path(os.getcwd())
data_dir     = notebook_dir.parent / "data"
work_dir     = notebook_dir.parent / "gmx_work"
```

### GROMACS / gmxapi Usage Pattern

```python
import gmxapi as gmx

cmd = gmx.commandline_operation(
    executable="gmx",
    arguments=["pdb2gmx", "-ff", "charmm36-jul2022", "-water", "tip3p"],
    input_files={"-f": str(pdb_input)},
    output_files={"-o": str(gro_out), "-p": str(top_out), "-i": str(posre_out)},
)
cmd.run()
assert cmd.output.returncode.result() == 0
```

### Return Code Validation

Always check the return code after any gmxapi operation:

```python
rc = cmd.output.returncode.result()
assert rc == 0, f"GROMACS command failed with return code {rc}"
```

### Output File Validation

Verify output file sizes and content after key steps:

```python
assert gro_out.exists() and gro_out.stat().st_size > 0
```

---

## Force Field Notes

- **CHARMM36-jul2022** is used because it provides validated parameters for proteins and lipids and is widely cited in SARS-CoV-2 studies.
- **TIP3P** water model is paired with CHARMM36 as its recommended explicit-solvent model.
- The bundled force field in `data/charmm36-jul2022.ff/` overrides the system force field. Do not modify these files without domain expertise.

---

## Important Files Not to Modify

| File | Reason |
|------|--------|
| `data/npt.tpr` | Binary precomputed GROMACS run input — regenerating requires full parameterization pipeline |
| `data/charmm36-jul2022.ff/` | Force field parameters — edits may corrupt simulation physics |
| `data/protein_clean.pdb` | Curated protein structure — edits require chemical domain knowledge |
| `gmx_work/topol.top` | Generated topology — do not hand-edit; re-run pdb2gmx instead |

---

## No CI/CD or Formal Tests

- There is **no automated test suite**, no `pytest`, and no CI/CD pipeline.
- Correctness is validated by running the notebook top-to-bottom and inspecting:
  - Return codes from each GROMACS command
  - Output file sizes
  - Atom/residue counts in topology
  - Sulfur geometry matrix (for disulfide bond verification)

---

## Development Notes for AI Assistants

1. **Do not fabricate GROMACS command flags** — refer to official GROMACS documentation or existing notebook cells for correct flag names.
2. **Path handling is critical** — all paths should be resolved relative to the notebook directory using `pathlib`. Avoid hardcoded absolute paths (e.g., `/home/biplab/...`).
3. **Binary files**: `*.tpr`, `*.trr`, `*.edr`, `*.gro` are GROMACS binary/text formats. Do not attempt to edit them as text.
4. **Backup files** like `#topol.top.1#` are automatically created by GROMACS on re-run; they are not user-created and can be ignored.
5. **The notebook is the primary artifact** — when adding workflow steps, add them as new cells in `test-gromacs.ipynb`, not as standalone scripts.
6. **Physics correctness matters** — changes to force field selection, water model, or simulation parameters require domain expertise. Flag such changes for human review.
7. **GPU availability** — the notebook checks for GPU-enabled mdrun. Do not assume GPU is always available; fall back gracefully.

---

## Git Information

- **Main development branch**: `master` / `main`
- **Feature branches**: follow pattern `claude/<description>-<id>`
- Only one commit exists as of initial setup (`1352bb1`)
- Remote: configured via local proxy (not a public GitHub remote)
