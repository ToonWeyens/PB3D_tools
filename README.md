# PB3D_tools
Auxiliary tools for the [PB3D](https://pb3d.github.io/) code
([source](https://github.com/ToonWeyens/PB3D)):
- `run` runs PB3D or POST, locally or through the PBS or SLURM scheduler, and sets up parameter scans.
- `extract_jobs_info` makes a table of the eigenvalues of such a scan.
- `inspect_SLURM_jobs` shows the running SLURM jobs of a user and cancels them on request.
- `analysis/` holds scripts that compare and tabulate the output of finished runs.
- `terpsichore/` holds scripts to run the 3-D stability code TERPSICHORE on PB3D's VMEC equilibria.

## Installation
Put the scripts on the `PATH`, from the folder of this repository:
```bash
echo -e "# PB3D tools\nexport PATH=\$PATH:$(pwd):$(pwd)/analysis:$(pwd)/terpsichore" >> ~/.bashrc
```

The Python scripts start with `python3` and need these packages:

| Scripts | Packages |
|---|---|
| `run` | `pyyaml` |
| `analysis/` | `numpy` and `h5py` (`checkvac_table.py` needs neither, `compare_petsc_mats.py` no `h5py`) |
| `terpsichore/wout2terps.py` | `numpy` and `netCDF4` |

With [uv](https://docs.astral.sh/uv/), `uv sync` in this folder makes `.venv` with all of them
(from `pyproject.toml` and `uv.lock`), and `source .venv/bin/activate` makes it the `python3` the
scripts use. Without uv: `pip install pyyaml numpy h5py netCDF4`.

`run` also needs `rsync`, `mpirun` for runs with one process or more, and `valgrind` for debug runs
(`-d`). `extract_jobs_info` and `inspect_SLURM_jobs` are bash scripts; they run on Linux and macOS.

## run
```
run -g
run PB3D [-o OUT] [-i SCAN] [-c CONFIG] [-v] [-d] [-l LOG] N INPUT EQUILIBRIUM
run POST [-o OUT]           [-c CONFIG] [-v] [-d] [-l LOG] N INPUT PB3D_OUTPUT
```
- `N` is the number of MPI processes; `0` runs the program as a single process without `mpirun`.
- `INPUT` is the input file, `EQUILIBRIUM` the VMEC (`.nc`) or HELENA file.
- `PB3D_OUTPUT` is the `.h5` output file of a PB3D run, or the folder that holds it.
- `-o OUT` names the output folder. For PB3D it must not exist yet; by default it is named after
  the date and time, such as `2026-11-20.21-5-3`. For POST the default is the folder of the PB3D
  output, and POST then writes its output there.
- `-i SCAN` runs one PB3D run per line of the scan file (see below).
- `-v` prints what `run` does, `-d` runs the program under `valgrind`, `-l LOG` writes the
  messages of `run` to a file.

`run` exits with status 1 when a local run returns an exit status that is not zero (PB3D and POST
do that after an error), or when the scheduler refuses a job.

### Configuration file
`run -g` writes `run.yaml.sample`. Edit it and save it as `run.yaml`. `run` reads the file given
with `-c`, by default `run.yaml` in the current folder, and otherwise the `run.yaml` next to the
`run` script.

| Key | Meaning |
|---|---|
| `prog_dir` | folder with the `PB3D` and `POST` executables, for example PB3D's CMake build folder |
| `scheduler` | `None` (run here), `pbs` (submit with `qsub`) or `slurm` (submit with `sbatch`) |
| `valgrind_location` | `valgrind` executable for `-d`; `None` takes the one on the `PATH` |
| `temp_run_prefix` | if set, a job copies its files to a folder under this prefix (a local disk such as `/tmp/`), runs there and copies the results back with `rsync` |
| `extra_PB3D_opts`, `extra_POST_opts` | options added to the command line of PB3D or POST, such as `--no_plots` |
| `max_tot_mem` | memory to ask from the scheduler, in MB (required for PBS) |
| `queue` | PBS queue or SLURM partition (required for PBS) |
| `n_nodes`, `n_cores` | nodes and cores per node to ask from the scheduler |
| `extra_scheduler_script` | script that a job sources first, for example to load modules |
| `wall_time` | wall time to ask from the scheduler, `"HH:MM:SS"` in quotes; `None` asks 12 hours |

Write the wall time in quotes: YAML reads an unquoted `12:00:00` as a number of seconds. `run`
converts such a number back, so both forms work.

`run.yaml` files from before `wall_time` existed still work and ask 12 hours.

### Output folder
```
OUT/
    PB3D, input, <equilibrium>          copies of the executable, the input and the equilibrium
    PB3D.sh / PB3D.slurm / PB3D.pbs     the script that runs PB3D
    PB3D.out                            screen output of PB3D (of the job, with a scheduler)
    PB3D.err                            errors of the job (scheduler only; removed when empty)
    PB3D.status                         exit status of PB3D
    PB3D_out.h5, PB3D_out.txt,          output of PB3D itself
    PB3D_out_EV_R_<level>.txt           eigenvalues, one file per Richardson level
    Plots/, Data/, Scripts/             plot output of PB3D
```
A POST run adds `POST`, its input, `POST.sh` (or `.slurm`, `.pbs`), `POST.out`, `POST.status`
and POST's own output to the same kind of folder.

### Parameter scans
A scan file has one run per line. Each line lists the input variables to change, separated by
commas; lines that start with `#` are ignored:
```
# prim_X and n_mod_X for each run
prim_X = 20, n_mod_X = 20
prim_X = 25, n_mod_X = 25
```
`run PB3D -i scan ...` then makes `OUT/1/`, `OUT/2/`, ... with the layout above, each with its own
input file, and links to `OUT/PB3D` and the equilibrium in `OUT/`. The scan file is copied to
`OUT/array_input`. PB3D's `Examples/array_input` is an example.

In each copy of the input file, `run` replaces every line in which the variable name stands as a
separate word with `    <name> = <value>`. A line with the name twice stops `run` with an error.
A line outside the namelist that holds the name is also replaced; PB3D does not read it.

## extract_jobs_info
```
extract_jobs_info RUN_FOLDER
extract_jobs_info RUN_FOLDER SCAN_FILE OUTPUT_FILE
```
Reads the eigenvalues of a scan made with `run PB3D -i` and writes them to `RUN_FOLDER/EV.dat`
(or `OUTPUT_FILE`), using `RUN_FOLDER/array_input` (or `SCAN_FILE`). The file starts with the scan
lines as comments. Each further row holds the run number, the values the scan set, and the real and
imaginary part of each eigenvalue in the file of the highest Richardson level.

A row shows `ERR` when the run returned an exit status that is not zero (`PB3D.status`) or left a
`PB3D.err` that is not empty, and `..` when the run wrote no eigenvalue file. For runs that failed,
the script prints the end of `PB3D.out`.

## inspect_SLURM_jobs
```
inspect_SLURM_jobs [USERNAME]
```
For each running SLURM job of the user (by default the current one), shows the end of `PB3D.out`
in the job's folder and asks whether to cancel the job, twice. It ends by printing the queue.

## analysis
Python scripts that read the output of finished PB3D runs: `PB3D_out.h5`, PB3D's screen output, or
PETSc matrix dumps. Run a script without arguments to see its usage.

| Script | What it prints |
|---|---|
| `compare_eq.py a.h5 b.h5` | per dataset of the equilibrium and grid groups, max\|a-b\|/max\|a\|, largest first |
| `compare_np.py <np1 dir> <npK dir>...` | radial indices where the integrated perturbation matrices differ between process counts |
| `compare_petsc_mats.py mats_np1.dat mats_np2.dat [block size]` | differences between PETSc binary matrix dumps of two runs |
| `compare_res.py ref.h5 other.h5 [label]` | diagonal of the vacuum response of one run against another, matched by \|m\| |
| `compare_res_series.py <h5 or run dir>...` | convergence of the vacuum response over a series of runs, against the last one |
| `checkvac_table.py <run>...` | one row per run from the `CHECKVAC` lines that `check_vac = .true.` writes with the field-line vacuum (style 1); the axisymmetric vacuum (style 2) writes other checks, which show as `-` |

A run folder can come from `run` (an output folder, or `OUT/1/`, `OUT/2/`, ... of a scan) or from
PB3D's `run_deck.sbatch` (the first `np<N>/` folder in it is read). `compare_np.py` compares runs
with different numbers of processes: the `np<N>/` folders of one `run_deck.sbatch` job, or `run`
output folders made with different `N`.

## terpsichore
Scripts to run TERPSICHORE on PB3D's VMEC equilibria, for benchmarks; see
[`terpsichore/README.md`](terpsichore/README.md).

## Scripts in PB3D itself
PB3D keeps the scripts that build and test it. Its `scripts/snellius/` builds PB3D on the Snellius
supercomputer and runs it there: `run_deck.sbatch` runs one input file at several numbers of
processes in one job and prints the first eigenvalue of each, which the build-and-test pipeline
(`pipeline.sh`) uses. `Documentation/snellius-setup.md` in PB3D describes them. On Snellius, `run`
works with `scheduler: slurm`, `queue: rome` and `extra_scheduler_script: ~/pb3d/env.sh` (the
modules), and with a short `wall_time` such as `"00:15:00"`.
