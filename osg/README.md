# Open Science Grid (OSG)

See https://support.opensciencegrid.org for general information.

## Building

The [makepkg](makepkg) script creates a package `hic-osg.tar.gz` containing all the model code to be distributed to each job.
Run it on OSG submit host such as XSEDE (xd-login.opensciencegrid.org) or Duke CI-connect (duke-login.osgconnect.net)

## Submitting jobs

The script [hic-wrapper](hic-wrapper) is the Condor executable.
It sets environment variables, calls `run-events`, and copies the results file to the final destination via GridFTP (see [data transfer](#data-transfer) below).

Each job runs 10 events.

The script [submit](submit) generates the Condor job files and submits them.
__Please read the script and understand what it's doing.
In particular, check the GridFTP destination and OSG project name.__

`submit` uses a Condor DAG for managing jobs.
The DAG smoothly submits jobs to the queue and throttles the total number of idle jobs, so that many jobs can be submitted at once without overloading the queue.

The script is designed for submitting many jobs for each of a set of input files.
Its usage is

    submit batch_label jobs_per_input_file input_files...

where

- `batch_label` is a human-readable label which also sets the destination folder for all job results files in this batch
- `jobs_per_input_file` is the number of jobs to run for each input file
- `input_files...` are the paths to each input file to run

For example, I have a set of input files for running LHC events in `~/inputs/lhc`.
To run 10,000 events (1000 jobs) for each input file:

    ./condor/submit lhc 1000 inputs/lhc/*

## Data transfer

As currently written, results filed are transferred to a server at Duke University via GridFTP, using the `globus-url-copy` command-line tool in `hic-wrapper`.
However, support for the open source Globus toolkit will end (or has ended) [in January 2018](https://github.com/globus/globus-toolkit/blob/globus_6_branch/support-changes.md), so a new data transfer mechanism will be necessary.
OSG support has a page on [data transfer with Globus](https://support.opensciencegrid.org/support/solutions/articles/5000632397-data-transfer-with-globus), which might be the best solution.

I will not have time to address this myself.
It would be great if someone could figure out a new data transfer mechanism and make the necessary changes to `hic-wrapper` and `submit` (probably remove any references to the grid proxy).
