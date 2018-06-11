# run_PB3D
Runscript for the [PB3D](http://pb3d.github.io/) code, written in Python 3.

Features:
- Automatic support for schedulers such as PBS and SLURM.
- Customization through yaml options file.
- Easy parameter scans through input file modifications in a custom file .
- Complementary script `extract_jobs_info` to extract the eigenvalues for such an input file modification run.

Run
```
./run -h
```
for more information.

# extract_jobs_info
Extracts the Eigenvalues of jobs for an input array
Can be run with either 1 or 3 arguments:
1. `./extract_jobs_info [RUN FOLDER]` for default usage.
2. `./extract_jobs_info [RUN FOLDER] [ARRAY INPUT] [OUTPUT FILE]` where for the array input file can be specified, as well as the output file.
produced there.

# inspect_SLURM_jobs
Inspect all SLURM currently being run by user, with the option to cancel them interactively.

# profile
Runs gprof on gmon.out and visualized with profile.xdot.
