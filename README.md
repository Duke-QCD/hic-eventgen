# hic-eventgen

_heavy-ion collision event generator_

## Three-sentence summary

- This is a workflow for running simulations of relativistic heavy-ion collisions.
- The primary intent is generating large quantities of events for use in Bayesian parameter estimation projects.
- It includes scripts and utilities for running on the [Open Science Grid (OSG)](https://www.opensciencegrid.org) and [NERSC](https://www.nersc.gov), and could run on other systems.

## Physics models

The collision model consists of the following stages:

- [trento](https://github.com/Duke-QCD/trento) – initial conditions
- [freestream](https://github.com/Duke-QCD/freestream) – pre-equilibrium
- [OSU hydro](https://github.com/jbernhard/osu-hydro) – viscous 2+1D hydrodynamics
- [frzout](https://github.com/Duke-QCD/frzout) – particlization
- [UrQMD](https://github.com/jbernhard/urqmd-afterburner) – hadronic afterburner

Each is included as a git submodule in the [models](models) directory.

:warning: git submodules have some annoying behavior.
__Use the `--recursive` option when cloning this repository to also clone all submodules.__
I suggest skimming the [section on submodules in the Pro Git book](https://git-scm.com/book/en/v2/Git-Tools-Submodules).

## Installation

hic-eventgen is probably most useful on high-performance computational systems, but it can run locally for testing or generating a few events.

### Computational systems

- [Open Science Grid (OSG)](osg)
- [NERSC](nersc)

### Local

- [Local usage](local)

## Running events

_general information for running on all systems_

The Python script [models/run-events](models/run-events) executes complete events and computes observables.
The basic usage is

    run-events [options] results_file

Observables are written to `results_file` in __binary__ format.
Raw particle data may also be saved to an hdf5 file using the option `--particles <filename>`.
For more information, see [event data format](#event-data-format) and [raw particle data](#raw-particle-data) below.

The most common options are:

- `--nevents` number of events to run (by default, events run continuously until interrupted)
- `--nucleon-width` Gaussian nucleon width (passed to `trento` and used to set the hydro grid resolution)
- `--trento-args` arguments passed to `trento`
  - __must__ include the collision system and cross section
  - __must not__ include the nucleon width or any grid options
- `--tau-fs` free-streaming time
- `--hydro-args` arguments  passed to `osu-hydro`
  - __must not__ include the initial time, freeze-out energy density, or any grid options
- `--Tswitch` particlization temperature

:warning: Options `--trento-args` and `--hydro-args` are passed directly to the respective programs.
Ensure that the restrictions described above are satisfied.
See also the docs for [trento](http://qcd.phy.duke.edu/trento) and [OSU hydro](https://github.com/jbernhard/osu-hydro).

See `run-events --help` for the complete list of options.

Options may also be specified [in files](#input-files).

### The hydro grid

The computational grid for `osu-hydro` is determined adaptively for each event in order to achieve sufficient precision without wasting CPU time.

The grid cell size is set proportionally to the nucleon width (specifically 15% of the width).
So when the nucleon width is small, events run a fine grid to resolve the small-scale structures;
for large nucleons, events run on a coarser (i.e. faster) grid.

The physical extent of the grid is determined by running each event on a coarse grid with ideal hydro and recording the maximum size of the system.
Then, the event is re-run on a grid trimmed to the max size.
This way, central events run on large grids to accommodate their transverse expansion, while peripheral events run on small grids to save CPU time.
Although pre-running each event consumes some time, this strategy is still a net benefit because of all the time saved from running peripheral events on small grids.

### Event data format

Event observables are written in __binary__ format with the data type defined in `run-events` (do a text search for "results = np.empty" to find it in the file).
Many results files may be concatenated together:

    cat /path/to/results/* > events.dat

In Python, read the binary files using [numpy.fromfile](https://docs.scipy.org/doc/numpy/reference/generated/numpy.fromfile.html).
This returns [structured arrays](https://docs.scipy.org/doc/numpy/user/basics.rec.html) from which observables are accessed by their field names:

```python
import numpy as np
events = np.fromfile('events.dat', dtype=<full dtype specification>)
nch = events['dNch_deta']
mean_pT_pion = events['mean_pT']['pion']
```

It's probably easiest to copy the relevant `dtype` code from `run-events`.

I chose this plain binary data format because it's fast and space-efficient.
On the other hand, it's inconvenient to fully specify the dtype when reading files, and organizing many small files can become unwieldy.

A format with metadata, such as HDF5, is not a good choice because each event produces such a small amount of actual data.
The metadata takes up too much space relative to the actual data and reading many small files is too slow.

The best solution would probably be some kind of scalable database (perhaps MongoDB), but I simply haven't had time to get that up and running.
If someone wants to do it, by all means go ahead!

### Raw particle data

Occasionally, it is necessary to save the UrQMD particle data output by each event. This can produce _a lot_ of data, typically on the order of ~1 MB per event.

The raw particle data may be saved to an hdf5 file using `--particles <filename>`. The hdf5 file is easily loaded and inspected in Python. For example,
```python
import h5py

with h5py.File('particles.hdf', 'r') as f:
    for event in f.values():
        for particle in event:
            print(particle.dtype)
```
prints the data format
```python
[('sample', '<i8'), ('ID', '<i8'), ('charge', '<i8'), ('pT', '<f8'), ('ET', '<f8'), ('mT', '<f8'), ('phi', '<f8'), ('y', '<f8'), ('eta', '<f8')]
```
for each particle in the event. Here `sample` denotes the number of the UrQMD oversample, `ID` is the Particle Data Group particle number, `charge` is its electric charge, `pT` the transverse momentum, `ET` the transverse energy, `mT` the transverse mass, `phi` the azimuthal angle, `y` the rapidity, and `eta` the pseudorapidity.


### Input files

Options for `run-events` may be specified on the command line or in files.
Files must have one option per line with `key = value` syntax, where the keys are the option names without the `--` prefix.
After creating an input file, use it with the syntax `run-events @path/to/input_file`.

For example, if a file named `config` contains the following:

    nevents = 10
    nucleon-width = 0.6

Then `run-events @config` is equivalent to `run-events --nevents 10 --nucleon-width 0.6`.

Input files are useful for saving logical groups of parameters, such as for a set of design points.

### Parallel events

`run-events` can be used as an MPI executable for running multiple events in parallel (this is most useful on HPC systems like NERSC).

Option `--rankvar` must be given so that each `run-events` process can determine its rank.
The basic usage is

    mpirun [mpirun_options] run-events --rankvar <rank_env_var> ...

where `<rank_env_var>` is the name of the rank environment variable set by `mpirun` for each process.
For example, Open MPI sets `OMPI_COMM_WORLD_RANK`

    mpirun [mpirun_options] run-events --rankvar OMPI_COMM_WORLD_RANK ...

On SLURM systems (like at NERSC), `srun` sets `SLURM_PROCID`

    srun [srun_options] run-events --rankvar SLURM_PROCID ...

When running with `--rankvar`, output files become folders and each rank creates a file, e.g. `/path/to/results.dat` becomes `/path/to/results/<rank>.dat`.
The formatting of `<rank>` may be controlled by option `--rankfmt`, which must be a [Python format string](https://docs.python.org/3/library/string.html#format-string-syntax).
This is probably most useful for padding rank integers with zeros, e.g. `--rankfmt '{:02d}'`.

When running in parallel, I recommend using the `--logfile` option so that each process writes its own log file, otherwise the output of all processes will intersperse on stdout.

Full example:

    mpirun -n 100 run-events \
      --rankvar OMPI_COMM_WORLD_RANK \
      --rankfmt '{:02d}' \
      --logfile output.log \
      results.dat

This would create results files `results/00.dat`, `01.dat`, ..., `99.dat` and corresponding log files `output/00.log`, ..., `99.log`.

### Checkpoints

Events can be checkpointed and restarted if interrupted.
The basic usage is

    run-events --checkpoint <checkpoint_file_path> ...

Before starting each event, checkpoint data is written to `<checkpoint_file_path>` in Python pickle format (for which I like the extension `.pkl`).
If the event completes successfully, the checkpoint file is deleted.
If the event is interrupted, it can be restarted later by

    run-events checkpoint <checkpoint_file_path>

(Note the differences from the first command: there is no `--` and no other options are accepted.)
When running a checkpoint, the original results and log files are appended to.

Example:

    run-events --checkpoint ckpt.pkl --logfile output.log results.dat

At some point, this process is interrupted:
`results.dat` contains data for any events that have completed, `ckpt.pkl` contains the incomplete event, and `output.log` reflects this status.
Then, sometime later:

    run-events checkpoint ckpt.pkl

This will run the event saved in `ckpt.pkl`, appending to `results.dat` and `output.log`.
Upon completion, `ckpt.pkl` is deleted.
