# Faasm Python Environment [![Tests](https://github.com/faasm/faasm-cpython/workflows/Tests/badge.svg?branch=master)](https://github.com/faasm/faasm-cpython/actions)  [![License](https://img.shields.io/github/license/faasm/faasm-cpython.svg)](https://github.com/faasm/faasm-cpython/blob/master/LICENSE.md) 

This build cross-compiles CPython and a number of Python modules to WebAssembly
for use in [Faasm](https://github.com/faasm/faasm).

It also provides a [small Python library](pyfaasm/) which uses `ctypes` to
support calls to the [Faasm host
interface](https://github.com/faasm/faasm/blob/master/docs/host_interface.md).

## Set-up and development

We recommend using Docker for developing this repo. Start the container with:

```
./bin/cli.sh
```

To show the list of available tasks (inside the container):

```bash
inv -l
```

The build uses the [Faasm
toolchain](https://github.com/faasm/faasm-toolchain) to cross-compile both 
CPython and C-extensions to WebAssembly.

## Code changes

The CPython build uses this slightly modified [fork of
CPython](https://github.com/faasm/cpython/tree/faasm).

To see the changes made to CPython, see [this
compare](https://github.com/python/cpython/compare/v3.8.2...faasm:faasm).

A similar (small) list of changes for numpy can be seen
[here](https://github.com/numpy/numpy/compare/v1.19.2...faasm:faasm).

CPython is built statically, some notes on this process can be found 
[here](https://wiki.python.org/moin/BuildStatically). 

Several of the code changes to CPython and numpy were borrowed from 
[pyodide](https://github.com/iodide-project/pyodide).

## Releasing

This repo gets built as a container, `faasm/cpython`. If you want to release a
new version, you can:

- Update the version in `VERSION`, `.github` and `.env`
- Commit to your branch
- Check that the normal build works in CI
- Run `inv git.tag`
- Let the Github action do the rest

## PyFaasm

To build and test this natively using the Faasm emulator (from inside the
development container):

```bash
# Build natively
inv pyfaasm.native

# Run tests
inv pyfaasm.test
```

## Set-up notes

We highly recommend using the containerised approach above. Everything
discuessed below is already set up in the container environment, and these notes
are only useful when debugging or upgrading parts of the build.

### CPython on the build machine/ container

To cross-compile CPython and C-extensions, we need a version of Python on the
build machine that _exactly_ matches the version of CPython we're building.
This is handled with the `install_build_python.sh` script.

This will install Python at `/usr/local/faasm/python3.8`.

When cross-compiling we _have_ to use this Python when running commands and
scripts on the build machine (not any other Python that might be installed).

### Upgrading Pip

Do **not** upgrade Pip in the build machine copy of Python.

The versions of Pip and Python for wasm and the build machine must match 
exactly.

### Building CPython to WebAssembly

You can build CPython by running (with optional `--clean`):

```
inv cpython
```

The result is installed at `third-party/cpython/install/wasm`.

We provide a [Setup.local](third-party/cpython/Modules/Setup.local) file, which
specifies which standard CPython modules will be built statically. 

At the end of the CPython build, it will print out a list of which modules have
been successfully built and which have failed. Note that many of the standard 
library modules will fail in this build, but the ones we need should succeed.

## Cross-compilation set-up

Setuptools and distutils both interrogate the Python system environment during
the build process. This makes it quite difficult to cross-compile libraries, so
we use [crossenv](https://github.com/benfogle/crossenv).

To set up crossenv:

```
./bin/crossenv_setup.sh
```

You can then activate with:

```
. cross_venv/bin/activate
```

From inside the virtual environment, you can inspect the set-up with:

```
python bin/sanity_check.py | less
```

This will display the environment used to install Python modules (including the
relevant cross-compilation variables, e.g. `CC`, `CFLAGS` etc.).

### Changing the crossenv environment

Crossenv picks up the cross-compilation environment from the CPython 
build artifacts. Therefore, to make changes to the cross-compilation 
environment:

- Modify the CPython build (see `tasks.py`)
- Rerun the CPython build (`inv cpython --clean`) 
- Rebuild the crossenv (`./bin/crossenv_setup.sh`) 
- Enter the crossenv and inspect the environment with `bin/sanity_check.py`

## Modules

With the crossenv activated, we can build modules with normal `pip`.

There is a wrapper script that will apply modifications if we know about them.
To run this you must first have the cross-env activated as described above.

```bash
# Install all supported modules
inv libs.install 

# Install experimental modules
inv libs.install --experimental

# Install numpy
inv libs.install --name numpy

# (Attempt) to install arbitrary module
inv libs.install --name <module_name>
```

Libraries will then be installed to 
`cross_venv/cross/lib/python3.8/site-packages`.

### Debugging module builds

You can debug module builds by running `python setup.py install` through your
debugger.

You can also set `DISTUTILS_DEBUG=1` to get distutils to print more info.

## Experimental modules

Some of the modules are experimental, these may require some extra set-up.

### MXNet and Horovod

To install the Python MXNet module we first need to cross-compile the MXNet
shared library:

```
# Update all submodules
cd third-party/mxnet
git submodule update --init
cd ../horovod
git submodule update --init

# Run our MXNet cross-compile (outside crossenv)
cd ../..
inv mxnet
```

Then we can install mxnet and horovod:

```
. ./cross_venv/bin/activate
inv libs.install --name mxnet
inv libs.install --name horovod
```

### Cleaning and uninstalling 

```
# Clean then rebuild
inv mxnet --clean

# Uninstall mxnet
inv mxnet.uninstall
```

## Running in Faasm

See the [Faasm python
docs](https://github.com/faasm/faasm/blob/master/docs/python.md).

## BLAS and LAPACK

Faasm's normal BLAS and LAPACK support using CLAPACK should be picked up by
numpy due to the addition of the [site.cfg](../third-party/numpy/site.cfg).

