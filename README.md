# PB3D_tools
Auxiliary tools for easy PB3D usage:
- Run PB3D or POST with `run`.
- Extract jobs info for parameter scan runs with input file modifications with `extract_jobs_info`.
- Inspect all running SLURM jobs for current user with `inspect_SLURM_jobs`.
- Compare and tabulate PB3D output with the scripts in `analysis/`.
- Run TERPSICHORE on PB3D's VMEC equilibria with the scripts in `terpsichore/`.

For easy usage, add directory to `$PATH`:
```
echo -e "# add PB3D run tools \nexport PATH=\$PATH:$(pwd)" >> ~/.bashrc
```

## run
Runscript for the [PB3D](http://pb3d.github.io/) code, written in Python 3.

Features:
- Automatic support for schedulers such as PBS and SLURM.
- Customization through yaml options file.
- Easy parameter scans through input file modifications in a custom file .
- Complementary script `extract_jobs_info` to extract the eigenvalues for such an input file modification run.

Run
```
./run -g
```
to generate a template [yaml](http://yaml.org/start.html) options file, and see
```
./run -h
```
for more information.

## extract_jobs_info
Extracts the Eigenvalues of jobs for an input array
Can be run with either 1 or 3 arguments:
1. `./extract_jobs_info [RUN FOLDER]` for default usage.
2. `./extract_jobs_info [RUN FOLDER] [ARRAY INPUT] [OUTPUT FILE]` where for the array input file can be specified, as well as the output file.
produced there.

## inspect_SLURM_jobs
Inspect all SLURM currently being run by user, with the option to cancel them interactively.

## analysis
Python 3 scripts that read PB3D output (`PB3D_out.h5`, PB3D's standard output or PETSc matrix
dumps) of any run. All but `checkvac_table.py` need `numpy`, and all but `checkvac_table.py` and
`compare_petsc_mats.py` need `h5py`. Each script prints its usage in its docstring.

| Script | What it prints |
|---|---|
| `compare_eq.py a.h5 b.h5` | per dataset of the equilibrium and grid groups, max\|a-b\|/max\|a\|, largest first |
| `compare_np.py <np1 dir> <npK dir>...` | radial indices where the integrated perturbation matrices differ between process counts |
| `compare_petsc_mats.py mats_np1.dat mats_np2.dat [block size]` | differences between PETSc binary matrix dumps of two runs |
| `compare_res.py ref.h5 other.h5 [label]` | diagonal of the vacuum response of one run against another, matched by \|m\| |
| `compare_res_series.py <h5 or run dir>...` | convergence of the vacuum response over a series of runs, against the last one |
| `checkvac_table.py <run>...` | one row per run from the `CHECKVAC` lines that `check_vac = .true.` writes |

## terpsichore
Scripts to run the 3-D stability code TERPSICHORE on PB3D's VMEC equilibria, for benchmarks; see
`terpsichore/README.md`.
