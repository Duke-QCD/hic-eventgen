# NERSC

See https://www.nersc.gov/users/getting-started for general information.

## Installation

The models must be installed in a conda environment in the NERSC [global common file system](https://www.nersc.gov/users/storage-and-file-systems/file-systems/global-common-file-system), which is intended for software installation.

See the [conda docs](https://conda.io/docs) for germane details.

Download and install [Miniconda](https://conda.io/miniconda.html) on a NERSC machine:
```bash
wget https://repo.continuum.io/miniconda/Miniconda3-latest-Linux-x86_64.sh
bash Miniconda3-latest-Linux-x86_64.sh
```
Install it somewhere in your home directory (the default location `~/miniconda3` is fine).

The installer might ask if you want to prepend the conda install location to your `PATH`.
This is a good idea, but on NERSC you cannot edit `~/.bashrc` itself (see [NERSC user environment](https://www.nersc.gov/users/software/user-environment/home-directories)).
Add to `~/.bashrc.ext` (or the equivalent file for your shell):
```bash
export PATH="$HOME/miniconda3/bin:$PATH"
```

While the file is open, you will probably need to set your locale language to a UTF-8 encoding, or you'll get unicode errors when installing the freestream model.
```bash
export LANG='en_US.UTF-8'  # or any other UTF-8 language
```

Now create and activate a conda environment with the required libraries:
```bash
conda create -p /global/common/software/<project>/<prefix> numpy scipy cython h5py
source activate /global/common/software/<project>/<prefix>
```
where `<project>` is your NERSC project name and `<prefix>` is your desired name for the environment.
This is the environment where all models will be installed:

- It must be active during installation (so the install script knows where to place files).
- It must be activated in job batch scripts before running events (see below).
- It does not need to be active when running `sbatch` to submit jobs or during any other general tasks on NERSC.

With the environment prepared and active, run the [install](install) script.

## Running jobs

I recommend running events on Edison or Cori Haswell, but not Cori KNL (these codes make poor use of the KNL architecture and so it would be an inefficient way to spend CPU hours).
I have set the compiler flags in the `install` script to target Edison and Cori Haswell.

Before running heavy-ion collision events, peruse the [NERSC documentation on running jobs](http://www.nersc.gov/users/computational-systems/cori/running-jobs), look at their example batch scripts for [Cori](https://www.nersc.gov/users/computational-systems/cori/running-jobs/example-batch-scripts) and [Edison](https://www.nersc.gov/users/computational-systems/edison/running-jobs/example-batch-scripts), and run some generic test jobs.

The [examples](examples) folder contains some sample batch scripts, discussed below.
__Please read the scripts carefully and modify them for your needs.
They will NOT work without some changes!__

### A single parameter point

[examples/simple](examples/simple) shows how to run some events all at the same parameter point.

__Environment configuration:__
The export commands at the beginning activate the conda environment for running events (all batch scripts must do this).
Be sure to replace `<project>` and `<prefix>` in `CONDA_PREFIX`.

__Number of tasks:__
I don't like the way the NERSC examples explicitly give the `-n` option to `srun`, because it depends on both the number of CPUs per node (which is different on Cori and Edison) and the number of nodes (which could be different for every job).
Instead, I set the `--cpus-per-task` sbatch option so that every CPU runs a single task, regardless of the number of CPUs per node or the number of nodes.

__Using run-events with srun:__
`run-events` is treated as an MPI executable via `srun`.
See section "parallel events" in the main readme.

__Filesystem:__
I use Cori scratch for all output files;
in my experience it provides sufficient performance even for large-scale jobs.
The `$CSCRATCH` environment variable points to your user directory on Cori scratch.

__Email notifications:__
If you want email notifications for job status updates, enter your address for the `--mail-user` sbatch option.

### Multiple parameter points

[examples/design](examples/design) demonstrates one method for running a set of parameter design points from input files.
It's similar to the "simple" example, but uses the intermediate script [examples/design-wrapper](examples/design-wrapper) to assign each CPU to a design input file in round-robin fashion.

__Considerations for large-scale jobs:__
Running a design may require many nodes.
For more than, say, 20 nodes, you should broadcast `design-wrapper` to the compute nodes in order to reduce job startup time.
See [this NERSC example](https://www.nersc.gov/users/computational-systems/edison/running-jobs/example-batch-scripts/#toc-anchor-4).

I also recommend placing input files on a high-performance filesystem, e.g. Cori scratch.

__Relative paths to input files:__
In this example, I have set the batch script working directory to `$CSCRATCH/inputfiles` and used relative paths to input files.
Then, `design-wrapper` uses the relative paths in the output filenames.

__Alternative methods:__
There are other strategies for running multiple parameter points, such as [taskfarmer](https://www.nersc.gov/users/data-analytics/workflow-tools/taskfarmer) and [job arrays](https://www.nersc.gov/users/computational-systems/cori/running-jobs/example-batch-scripts#toc-anchor-16).

### Checkpoints (important!)

The "simple" and "design" examples both run events continuously until the job times out;
this maximizes CPU time efficiency since there are never any idle CPUs.
If a job were to instead run a fixed number of events on each CPU, some would finish earlier than others and those CPUs would be idle until the last one finishes.
That usage is still charged, even if the CPUs are idle!

But there's a problem with running events until timeout.
The last event on each CPU is always interrupted, so longer-running events are more likely to be cut off.
And since more central events take longer, this introduces a centrality bias into the event sample.

__More generally: once an event begins, it is part of the cross section, and it must be completed!__

The solution to this is job checkpoints, which are documented in the "checkpoints" section of the main readme.

[examples/checkpoints-gnu-parallel](examples/checkpoints-gnu-parallel) shows how to run a batch of checkpoints using [GNU Parallel](https://www.gnu.org/software/parallel), which is suitable for a __single node__.

[examples/checkpoints-taskfarmer](examples/checkpoints-taskfarmer) shows how to use [taskfarmer](https://www.nersc.gov/users/data-analytics/workflow-tools/taskfarmer), which can run on many nodes and is thus useful if you have too many checkpoints to run on a single node in a reasonable time.
