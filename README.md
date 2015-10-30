# Heavy-ion collisions on the OSG

Workflow for running large-scale event-by-event heavy-ion collision simulations on the Open Science Grid (OSG).
Primarily intended for generating events for Bayesian model-to-data comparison.

## Physics models

The collision model is a standard hybrid hydro+cascade, similar to [iEBE](http://inspirehep.net/record/1319339) except for the initial conditions.
Each model resides in its own repository; they are included here as submodules.
See the `models` directory.


## Usage

_Note: I designed this for my own personal use, and this readme is barely more than notes to myself.  Feel free to contact me with questions._

### Building

The workflow is designed to run on the OSG from the XSEDE submit host xd-login.opensciencegrid.org (but could be modified to run elsewhere).

Clone the repository on `xd-login` with the `--recursive` option to acquire all submodules.

Run `./makepkg` to build all the models and create the job package `hic-osg.tar.gz`.
This package contains all files common to each job: run script, executables, data tables, etc.

### Input files

To pass parameters to the models, use "input files".
The idea is to create a bunch of input files (e.g. for a set of design points) and then run many jobs for each one.

Input files have a simple `key = value` syntax, like this:

    trento_args = <arguments that will be passed to trento>
    vishnew_args = <arguments that will be passed to trento>

The arguments are passed directly to the executables `trento` and `vishnew`.
Here is a specific example:

    trento_args = --normalization 120 --reduced-thickness 0 --fluctuation 1.5 --nucleon-width 0.45
    vishnew_args = etas=0.08 etas_slope=0.85 visbulknorm=1.2 edec=0.24

### Submitting jobs

The Python script `models/run-events` executes each job.
It runs 10 events per job, meaning 10 Monte Carlo initial conditions and hydro events plus oversampling.
The number of oversamples is determined adaptively so that each event produces roughly the same total number of particles.

The results for each job are placed in a single HDF5 file `hic-osg/results.hdf` in the job working directory.
The file contains initial conditions and particle data for all events and oversamples.

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

### Data analysis

See my [qm2015 repository](https://github.com/jbernhard/qm2015) for some example data analysis code and results.
