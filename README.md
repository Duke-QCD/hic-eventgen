# Heavy-ion collisions on the OSG

Workflow for running large-scale event-by-event heavy-ion collision simulations on the Open Science Grid (OSG).
Primarily intended for generating events for Bayesian model-to-data comparison.

## Physics models

The collision model consists of the following stages:

- initial conditions (trento)
- free streaming
- viscous 2+1D hydro (osu-hydro)
- particlization (frzout)
- hadronic afterburner (UrQMD)

Each model is included as a git submodule in the `models` directory.

## Usage

I designed this for my personal projects — it is not a general-purpose framework (generalizing it would introduce far too many variables and "moving parts").
However, it has a lot of useful functionality and is an excellent starting point for similar heavy-ion collision projects.
I recommend forking this repository and modifying it as needed.

I realize this readme is brief, and I plan to write more detailed documentation.
Feel free to contact me with questions!

### Building

To submit jobs, build on an OSG submit host, e.g. XSEDE (xd-login.opensciencegrid.org) or Duke CI-connect (duke-login.osgconnect.net).

Clone the repository with the `--recursive` option to acquire all submodules.

Run `./makepkg` to build all the models and create the job package `hic-osg.tar.gz`.
This package contains all files common to each job: run script, executables, data files, etc.

### The run-events script

The Python script `models/run-events` executes complete events and computes observables.
The basic usage is

    run-events [options] results_file

Observables are written to `results_file` in binary format.
The binary data type is defined in `run-events` and must be fully specified when loading the files;
the idea is to cat a set of results files together and then load it using e.g. `numpy.fromfile`.

The most common options are:

- `--nevents` number of events to run
- `--nucleon-width` Gaussian nucleon width (passed to `trento` and used to set the hydro grid resolution)
- `--trento-args` arguments passed to `trento`
  - __must__ include the collision system and cross section
  - __must not__ include the nucleon width or any grid options
- `--tau-fs` free-streaming time
- `--hydro-args` arguments  passed to `osu-hydro`
  - __must not__ include the initial time, freeze-out energy density, or any grid options
- `--Tswitch` particlization temperature

__WARNING__: Options `--trento-args` and `--hydro-args` are passed directly to the respective programs.
Ensure that the restrictions described above are satisfied.

See `run-events --help` for the complete list of options.

### Input files

Options for `run-events` may be specified on the command line or in files.
Files must have one option per line with `key = value` syntax, where the keys are the option names without the `--` prefix.
After creating an input file, use it with the syntax `run-events @path/to/input_file`.

For example, if a file named `config` contains the following:

    nevents = 10
    nucleon-width = 0.6

Then `run-events @config` is equivalent to `run-events --nevents 10 --nucleon-width 0.6`.

Input files are useful for saving logical groups of parameters, e.g. for a set of design points.

### Submitting jobs

The shell script `condor/hic-wrapper` is the Condor executable.
It sets environment variables, calls `run-events`, and copies the results file to the final destination via GridFTP (note: this requires a grid certificate).

Each job runs 10 events.

The shell script `condor/submit` generates the Condor job files and submits them (note: it has the GridFTP destination hard-coded to the Duke file server).
To submit a batch of jobs:

    ./condor/submit batch_label jobs_per_input_file input_files...

where

- `batch_label` is a human-readable label that sets the destination folder for all job results files in this batch
- `jobs_per_input_file` is the number of jobs to run for each input file
- `input_files...` are the paths to each input file to run

For example, I have a set of input files for running LHC events, and I want to run 10,000 events (1000 jobs) for each input file:

    ./condor/submit lhc 1000 inputs/lhc/*

The `submit` script writes a long DAG file and submits it.
The DAG will smoothly submit jobs to the queue and throttle the total number of idle jobs, so that essentially an arbitrary number of jobs can be submitted at once.
