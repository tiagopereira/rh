# Running the code

## Binaries and execution {#binaries-label}

Compilation produces three solver executables: `rh15d_ray_pool`,
`rh15d_ray`, and `rh15d_lteray`.
The LTE solver is a special case for
running only in LTE. The other two are the main programs. They represent
two different modes of job distribution: *normal* and
*pool*.

In the *pool* mode there is a process that works as
*overlord*: its function is to distribute the work to other processes.
The other processes (*drones*) ask the
*overlord* for a work unit. When they finish their unit,
they go back and ask for more, until all tasks are completed. Because of
race conditions and because different columns will run at different
speeds, it is not possible to know which columns a given process will
run beforehand. Due to the *overlord*, `rh15d_ray_pool`
needs to run with two or more processes. The advantage of the
*pool* mode is that the dynamic load allocation ensures the
most efficient use of the resources. With the normal mode it may happen
that some processors will work on columns that take longer to converge
(especially as they are adjacent), and in the end the execution will
have to wait for the process that takes longer. In some cases
(especially with PRD) the *pool* mode can be 2-3 times
faster than the *normal* mode. When one runs with a large
number of processes (> 2000) and each column takes little time to
calculate, the *pool* mode can suffer from communication
bottlenecks and may be slower because a single *overlord*
cannot distribute the tasks fast enough. The only disadvantage of the
pool mode (so far) is that not all output is currently supported with
this mode.

!!! warning
    
    The *normal* mode is deprecated and will be removed in a
    later revision. Use only for single processor runs or if you know what
    you're doing!


In the *normal* mode the jobs (all the atmosphere columns
for which one wants to calculate) are divided by the number of processes
at the start of the execution. There is no communication between
processes, and each process knows from the start all the columns it is
going to run. These columns are adjacent. If the number of columns is
not a multiple of the number of processes, there will be some processes
with larger workloads. There is no minimum number of processes to run,
and `rh15d_ray` can also be run in a single process. Regions of an
atmosphere can take a lot longer to run than others, and the processes
that work on those will take longer to finish. In the
*normal* mode this means that the slowest process will set
the overall running time, and therefore in practice it can take more
than 10x longer than the *pool* mode (and is therefore not
recommended).

As an MPI program, the binaries should be launched with the appropriate
command. Some examples:

    mpirun -np N ./rh15d_ray_pool
    srun ./rh15d_ray_pool       # Use in Betzy or other system with SLURM
    mpiexec ./rh15d_ray_pool    # use in Pleiades
    aprun -B ./rh15d_ray        # use in Hexagon or other Cray

The parallel I/O implementation requires parallel HDF5 1.14 or newer
(`H5Dread_multi`) and MPI-3 shared-memory support. The number of nodes
must not exceed the number of selected x rows, even with the atmosphere
cache disabled.

## The run directory

!!! warning

    Before running, make sure you have the sub-directories `scratch` and
`output` in the run directory.

The run directory contains the configuration files, the binaries, and
the `scratch` and `output` directories. As the names imply, temporary
files will be placed under `scratch` and the final output files in
`output`. No files under `scratch` will be used after the run is
finished (they are not read for re-runs).

The `scratch` directory contains different types of files. Most of them
are binary files write by RH 1.5D to save memory. Example files are the
`background_p*.dat` with the background opacities, files with PRD
weights, and the `rh_p*.log` log files. Each process creates one of
those files, and they will have the suffix `_pN.*`, where `N` is the
process number. The log files have the same format as in RH. The writing
of each process's log file is buffered by line. Because these are
updated often, when running with many processes this can be a drag on
some systems. Therefore, it is possible to run full buffering (meaning
log files are only written when the main program finishes). This option
is not exposed in the configuration files, so one needs to change the
file `parallel.c` in the following part:

``` c
/* _IOFBF for full buffering, _IOLBF for line buffering */
setvbuf(mpi.logfile, NULL, _IOLBF, BUFSIZ_MPILOG);
```

One should replace `_IOLBF` by `_IOFBF` to change from line buffering to
full buffering.

The `output` directory will contain the three output files:
`output_aux.hdf5`, `output_indata.hdf5`, and `output_ray.hdf5`. See
[output](output.md) for more details on the
structure of these files. If doing a re-run, these files must already
exist; they will be updated with the new results. Otherwise, if these
files are already in `output` before the execution, they will be
overwritten. At the start of the execution, the output files are written
with a special a fill value. This means that the disk space for the full
output must be available at the start of the run, and no CPU time will
be wasted if at the end of the run there is not enough disk space. The
files are updated by replacing fill values with computed data.
In pool mode, workers buffer completed columns and all ranks meet for a
coordinated output batch. Each worker contributes at most
`15D_FLUSH_INTERVAL` columns (default 16); the overlord and workers with
no output still participate with empty selections. Each batch is flushed
before computation resumes, with payload and histories flushed before
convergence flags are written. Normal and LTE modes retain independent
per-column writes.

A re-run can recover completed output and recompute missing or
non-converged columns. Columns still buffered in memory when a job stops
are not saved. A larger flush interval reduces the number of output
batches, but increases memory use, synchronization delays, and the amount
of work that may need to be repeated.

## Parallel I/O tuning

The runtime keywords and defaults are listed in [input files](input.md).
Pool output uses collective data transfers by default. Setting
`15D_POOL_COLLECTIVE_WRITE = FALSE` retains the same coordinated batches
but switches their data transfers to independent mode. This is useful
for comparing performance on different filesystems. Legacy writes and
population reads remain independent; a balanced task count alone does
not make their collective call order safe.

### Node-shared atmosphere cache

`15D_USE_NODE_ATMOS_CACHE = TRUE` loads each node's contiguous slab of
the selected HDF5 atmosphere snapshot into MPI shared memory. The pool
dispatcher prefers node-local work, then lets idle workers steal columns
from other nodes. Local reads use memory copies; stolen columns outside
the cache use independent HDF5 file reads. Pool atmosphere reads are
never collective because workers read at different times and the
overlord does not participate in their reads.

The cache is disabled by default. Enable it only when each node has
enough memory for its atmosphere slab in addition to the solver and
output buffers. It stores full-depth columns, before any temperature
cut, and is shared by the ranks on that node. It is not used for
non-HDF5 input or by `rh15d_ray` and `rh15d_lteray`.
`15D_ATMOS_CACHE_CYCLIC` is reserved and currently has no effect.

### Lustre layout and MPI-IO hints

Many supercomputers use Lustre, which distributes file stripes across
object storage targets (OSTs). Tune striping for the workload and the
filesystem rather than assuming that the largest stripe count is best.
Set the layout **before** creating output files, for example with:

    lfs setstripe -s stripe_size -c stripe_count -o stripe_offset directory|filename

Directory settings apply to newly created files; they do not change the
layout of files already present for a re-run. The built-in MPI-IO hint
defaults assume `lfs setstripe -c 24 -S 32M output/`. They are tuning
defaults, not a requirement or a guarantee that the MPI library will
honour them. Override them to match the actual file layout and site
recommendations:

| Environment variable | Default | MPI-IO hint | Meaning |
|:---------------------|:--------|:------------|:--------|
| `RH_LUSTRE_STRIPE_COUNT` | `24` | `striping_factor` | Stripe count; match the Lustre layout. |
| `RH_LUSTRE_STRIPE_SIZE` | `33554432` (32 MiB) | `striping_unit` | Stripe size in bytes. |
| `RH_MPIIO_CB_BUFFER_SIZE` | `268435456` (256 MiB) | `cb_buffer_size` | Collective-buffer size per aggregator. |
| `RH_MPIIO_CB_NODES` | Twice the number of nodes | `cb_nodes` | Requested total number of aggregators. |

These variables are read once during MPI initialization and logged by
rank 0, even when hints are disabled. They are passed to HDF5 only when
`15D_MPIIO_LUSTRE_HINTS = TRUE`. For example:

```bash
export RH_LUSTRE_STRIPE_COUNT=$(lfs getstripe -c output/)
export RH_LUSTRE_STRIPE_SIZE=$(lfs getstripe -S output/)
srun ./rh15d_ray_pool
```

For a re-run, query the existing files if their layout differs from the
directory default. Ensure environment variables reach all MPI ranks.
The application also sets `romio_cb_write=enable`,
`romio_cb_read=enable`, `romio_ds_write=disable`,
`romio_ds_read=disable`, and `cb_config_list=*:2`. These values have no
application-level environment overrides. MPI-library hint files are
implementation-specific and may not override explicitly supplied hints.

HDF5 tuning remains active even with MPI-IO hints disabled: 32 MiB
allocation alignment, 8 MiB metadata blocks, and file locking disabled.
`15D_MPIIO_LUSTRE_HINTS = TRUE` also enables collective metadata
operations on output files; setting it to `FALSE` disables them.
Atmosphere input metadata remains independent in either case. Turning
off the runtime toggles therefore does **not** restore the old I/O
implementation: batched output, consolidated atmosphere reads, and
node-aware scheduling remain in place.

To compare the toggleable optimizations, change one keyword at a time.
A configuration with the collective-transfer, cache, and hint toggles
disabled is:

```text
15D_POOL_COLLECTIVE_WRITE = FALSE
15D_USE_NODE_ATMOS_CACHE  = FALSE
15D_MPIIO_LUSTRE_HINTS    = FALSE
15D_FLUSH_INTERVAL       = 16
```

The directory where you will store the input atmospheres, and
the `scratch` directory can also benefit from Lustre striping. For
`scratch`, because most files there are small, it is recommended to use
a stripe count of 1.

!!! info

    You need a parallel filesystem if you are running RH 1.5D across more
    than one host. Most supercomputers have parallel filesystems, but if you
    are running in a smaller cluster this may not be the case. RH 1.5D will
    always run, but the HDF5 writes will not work and the results will be
    unreadable. NFS is **not** a parallel file system.


## Logs and messages

In addition to the logs per process saved to `scratch`, a much smaller
log will be printed in `stdout`. This log is a smaller summary of what
each process is doing. Here is an example of typical messages:

    Process   1: --- START task   1, (xi,yi) = (  0,156)
    Process 232: --- START task   1, (xi,yi) = (  0,159)
    Process  36: --- START task   1, (xi,yi) = (  0,162)
    Process  12: --- START task   1, (xi,yi) = (  0,171)
    (...)
    Process  12: *** END   task   1 iter, iterations = 121, CONVERGED
    Process   3: *** END   task   1 iter, iterations = 200, NO Convergence
    Process   4: *** SKIP  task   1 (crashed after 81 iterations)
    Process   3: --- START task   2, (xi,yi) = ( 23, 64)
    Process  12: --- START task   2, (xi,yi) = ( 23, 65)
    Process   4: --- START task   2, (xi,yi) = ( 23, 65)
    (...)
    *** Job ending. Total 262144 1-D columns: 262142 converged, 1 did not converge, 1 crashed.
    *** RH finished gracefully.

In this example one can see the three possible outputs for a
single-column calculation: convergence, non-convergence (meaning the
target `ITER_LIMIT` was not met in `N_MAX_ITER` iterations), or a crash
(many reasons). If there are singular matrices or other causes for a
column to crash, RH 1.5D will skip that column and proceed to the next
work unit. Such cases can be re-run with different parameters. In some
cases (e.g. inexistent files) it is not possible to prevent a crash, and
RH 1.5D will finish non-gracefully.

## Reruns and lack of convergence

Dynamic atmospheres often have large gradients in temperature, density,
velocity, etc. that cause problems when solving the non-LTE problem.
This may lead to some (or all) columns not converging, and is dependent
on the input atmosphere, model atoms, and run options. In RH terms,
non-converged or "crashed", represent the same problem. In some cases,
the iterations diverge strongly (crash), while in others they fail to
reach the target limit for convergence in the allocated maximum number
of iterations (non-convergence).

The output for atmosphere columns that did not converged or crashed is
not saved to disk (a fill value is used instead). The recommended
procedure in these cases is to rerun RH with different input options
(e.g. less agressive acceleration, a larger number of maximum
iterations). A special rerun mode is available to save time calculating
again the columns that have already converged. When `15D_RERUN = TRUE`
in `keyword.input`, RH will read the output and run again only for the
columns that did not converge.

The rerun mode requires all the previous output to be under `output/`.
The user has the possibility of changing options in `keyword.input` for
the rerun. Not all options can be changed, because this could lead to
non-sensical results (e.g. changing the input atmosphere or atom files).
Only the following options can be changed in `keyword.input`:

|  |
:-----------------------|
|  `15D_RERUN`
|  `15D_DEPTH_CUT`
|  `15D_TMAX_CUT`
|  `15D_DEPTH_REFINE`
|  `PRD_SWITCH`
|  `NRAYS`
|  `N_MAX_SCATTER`
|  `N_MAX_ITER`
|  `ITER_LIMIT`
|  `NG_DELAY`
|  `NG_ORDER`
|  `NG_PERIOD`
|  `S_INTERPOLATION`
|  `PRD_N_MAX_ITER`
|  `PRD_ITER_LIMIT`
|  `B_STRENGTH_CHAR`


All the other options are locked to the values used for the firs run.
Likewise, it is not possible to change `atoms.input`, the atom files, or
the line list files. When RH 1.5D is first run, nearly all input options
(except the input atmosphere and molecule files) are saved into the
output, and in a rerun these are read from the output and not from the
original files. This also means that a rerun can be performed even if
the original files are no longer available.

## Helper script

There is a Python script called `runtools.py` designed to make it easier
to run RH 1.5D for large projects. It resides in
`rh/python/runtools.py`. It requires [Python](http://www.python.org/)
with the [numpy](http://www.numpy.org/) and [h5py](http://www.h5py.org/)
(or [netCDF4](http://code.google.com/p/netcdf4-python/)) modules. It was
made to run a given RH 1.5D setup over many simulation snapshots,
spanning several atmosphere files. It supports a progressive rerun of a
given problem, and allows the of use different `keyword.input`
parameters for different columns, tackling columns harder to converge.

The first part of `runtools.py` should be modified for a users's need.
It typically contains:

``` python
atmos_dir = '/mydata_dir'
seq_file = 'RH_SEQUENCE'
outsuff = 'output/output_ray_mysim_CaII_PRD_s%03i.hdf5'
mpicmd = 'mpiexec'
bin = './rh15d_ray_pool'
defkey = 'keyword.save'
log = 'rh_running.log'
tm = 40
rerun = True
rerun_opt = [ {'NG_DELAY': 60, 'NG_PERIOD': 40, '15D_DEPTH_REFINE': 'FALSE',
               '15D_ZCUT': 'TRUE', 'N_MAX_ITER': 250, 'PRD_SWITCH': 0.002 },
              {'NG_DELAY': 120, 'NG_PERIOD': 100, '15D_DEPTH_REFINE': 'TRUE',
               'PRD_SWITCH': 0.001 } ]
```

The different options are:



  Name   |       Type  |   Description|
|:-----------|:-------|:---------------------------------------------------
  `atmos_dir` |  string  | Directory where the atmosphere files are kept.
  `seq_file`  |  string  | Location of sequence file. This file contains the names of the atmosphere files to be used (one file per line). The script will then run RH 1.5D for every snapshot in every file listed.
  `outsuff`   |  string  | Template to write the `output_ray.ncdf` files. The `%03i` format will be replaced with the snapshot number.
  `mpicmd`    |  string  | System-dependent command to launch MPI. The script knows that for aprun the -B option should be used. This option also activates system specific routines (e.g. how to kill the run in pleiades).
  `bin`       |  string |  RH 1.5D binary to use.
  `defkey`    |  string |  Default template for `keyword.input`. Because the of the rerun options, `keyword.input` is overwritten for every rerun. This file is used as a template it (i.e., most of its options will be unchanged, unless specified in `rerun_opt`).
  `log`       |  string |  File where to save the main log. Will be overwritten for each new snapshot.
  `tm`        |  int    |  Timeout (in minutes) to kill execution of code, if there is no message written to main log. Used to prevent code from hanging if there are system issues. After killed, program is relaunched. If `tm = 0`, program will never be killed.
  `rerun`     |  bool   |  If `True`, will re-run the program (with different settings) to achieve convergence if any columns failed. Number of reruns is given by size of `rerun_opt`.
  `rerun_opt`  | list   |  Options for re-run. This is a list made of dictionaries. Each dictionary contains the keywords to update `keyword.input`. Only the keywords that differ from the `defkey` file are necessary.


!!! info

    Only the first line of the sequence time is read at a time. The script
    reads the first line, deletes it from the file, and closes the file. It
    then reads the first line again and continues running, until there are
    no more lines in the file. This behaviour enables the file to be worked
    by multiple scripts at the same time, and allows one to dynamically
    change the task list at any time of the run.


!!! info

    The script also includes a tenaciously persistent wait and relaunch
    feature designed to avoid corruption if there are system crashes or
    problems. Besides the `tm` timeout, if there is any problem with the
    execution, the code will wait for some periods and try and relaunch the
    code. For example, if one of the atmosphere files does not exist,
    `runtools.py` will try three times and then proceed to the next file.
