# Local usage

## Installation

Prerequisites:

- Python __3.5+__ with numpy, scipy, cython, and h5py
- C, C++, and Fortran compilers
- CMake 3.4+
- Boost and HDF5 C++ libraries

Clone the repository with the `--recursive` option to acquire all submodules.

The models must be installed into an active Python [virtual environment](https://docs.python.org/3/library/venv.html) (venv) or a [conda environment](https://conda.io/docs/user-guide/tasks/manage-environments.html) (I suggest using [Miniconda](https://conda.io/miniconda.html)).

If Python and its packages are already installed on your system, the easiest choice is probably a venv:
```bash
python -m venv --system-site-packages --without-pip /path/to/venv
source /path/to/venv/bin/activate
```
Or if using conda:
```bash
conda create -n <env_name> numpy scipy cython h5py
source activate <env_name>
```

After creating and activating the environment, simply run the [install](install) script.

__Possible issue:__
When building the `trento` model inside a conda environment, you may see CMake warnings about "cannot generate a safe runtime search path..."
This can be solved by setting the [CMAKE_PREFIX_PATH](https://cmake.org/cmake/help/latest/variable/CMAKE_PREFIX_PATH.html) environment variable and re-running the `install` script (on most Linux systems the prefix is `/usr`).
Be sure to remove the trento build directory (`models/trento/build`) before re-running.
Example:
```bash
rm -r ../models/trento/build
CMAKE_PREFIX_PATH=/usr ./install
```

## Before running events

The venv or conda environment must be active.
This can of course be accomplished by sourcing the `activate` script as usual, but all that's actually necessary is for `<environment_prefix>/bin` to be in your `PATH`.
To verify this, check if your python executable points to `<environment_prefix>/bin/python` (run `which python`).

The environment variable `XDG_DATA_HOME` must be set to `<environment_prefix>/share` so that `osu-hydro` can find its data files.

In the venv example above, these criteria can be satisfied by running
```bash
export PATH="/path/to/venv/bin:$PATH"
export XDG_DATA_HOME="/path/to/venv/share"
```
Alternatively, use [run-events-wrapper](../models/run-events-wrapper), a simple script that sets the necessary variables and then forwards all arguments to `run-events`.
It is installed to the `bin` directory of the environment, alongside `run-events` itself.
It may be run using its full path:

    /path/to/venv/bin/run-events-wrapper arguments...
