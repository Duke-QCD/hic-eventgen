# Heavy-ion collisions on the OSG

Workflow for running large-scale event-by-event heavy-ion collision simulations on the Open Science Grid (OSG).
Primarily intended for generating events for Bayesian model-to-data comparison.

## Physics models

The collision model consists of the following stages:

- initial conditions (trento)
- free streaming
- viscous 2+1D hydro (vishnew)
- particlization (frzout)
- hadronic afterburner (UrQMD)

Each model is included as a git submodule in the `models` directory.

## Usage

_Note: I designed this for my own personal use, and this readme is barely more than notes to myself.  Feel free to contact me with questions._

### Building

The workflow is primarily tested on the OSG XSEDE submit host xd-login.opensciencegrid.org, but it should work on other hosts e.g. OSG Connect.

Clone the repository with the `--recursive` option to acquire all submodules.

Run `./makepkg` to build all the models and create the job package `hic-osg.tar.gz`.
This package contains all files common to each job: run script, executables, data tables, etc.

### Input files

To pass parameters to the models, use "input files".
The idea is to create a bunch of input files (e.g. for a set of design points) and then run many jobs for each one.

Input files have a simple `key = value` syntax, like this:

    trento_args = <arguments that will be passed to trento>
    tau_fs = <free streaming time in fm/c>
    vishnew_args = <arguments that will be passed to vishnew>
    Tswitch = <partclization temperature in GeV>

### Submitting jobs

The Python script `models/run-events` executes each job.
It runs 10 events per job, meaning 10 Monte Carlo initial conditions and hydro events plus oversampling.
The number of oversamples is determined adaptively so that each event produces roughly the same total number of particles.

For each event, several observables (particle multiplicities, flow Q-vectors, etc) are computed and saved to the raw binary file `<condor working directory>/hic-osg/results`.
The binary data type is defined in `models/run-events` and must be fully specified when loading the files;
the idea is to cat a set of results files together and then load it using e.g. `numpy.fromfile`.
More discussion of the pros and cons of this format is in the comments of `run-events`.

The shell script `condor/hic-wrapper` is the Condor executable.
It loads necessary environment modules, calls `run-events`, and copies the results file to the final destination via GridFTP (note: this requires a grid certificate).

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
