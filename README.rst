################################################
Utilities for building on Travis CI and AppVeyor
################################################

A set of scripts to automate builds of macOS and Manylinux1 wheels on the
`Travis CI <https://travis-ci.org/>`_ infrastructure, and also Windows
wheels on the `AppVeyor <https://ci.appveyor.com/>`_ infrastructure.

The Travis CI scripts are designed to build *and test*:

* Dual 32/64-bit architecture macOS wheels built for macOS 10.6+
* 64-bit macOS wheels built for macOS 10.9+
* 64-bit ``manylinuxX_x86_64`` wheels, both narrow and wide Unicode builds, where `X` is any valid Manylinux version, such as `1`, or `2010`
* 32-bit ``manylinuxX_i686`` wheels, both narrow and wide Unicode builds

You can currently build and test against Pythons 2.7, 3.5, 3.6, 3.7.

The small innovation here is that you can test against 32-bit builds, and both
wide and narrow Unicode Python 2 builds, which was not easy on the default
Travis CI configurations.

The AppVeyor setup is designed to build *and test*:

* 64-bit Windows ``win_amd64`` wheels
* 32-bit Windows ``win32`` wheels

You can currently build and test against Pythons 2.7, 3.5, 3.6, 3.7.

*****************
How does it work?
*****************

Multibuild is a series of bash scripts that define bash functions to build and
test wheels.

Configuration is by overriding the default build function, and defining a test
function.

The bash scripts are layered, in the sense that they loaded in the following
sequence:

macOS
=====

These bash scripts get sourced one after the other,
so that functions and variables defined in later scripts can overwrite
functions and variables in earlier scripts::

    multibuild/common_utils.sh
    multibuild/osx_utils.sh
    env_vars.sh
    multibuild/configure_build.sh
    multibuild/library_builders.sh
    config.sh

See ``multibuild/travis_osx_steps.sh`` to review the source order.

The macOS build / test phases run on the macOS VM started by Travis CI.
Therefore any environment variable defined in ``.travis.yml`` or the bash
shell scripts listed above are available for your build and test.

Build options are controlled mainly by the following environment
variables:

* ``MB_PYTHON_VER`` selects the Python version built for, in the format ``major.minor.patch`` for CPython, or ``pypy-major.minor`` for PyPy
* ``MB_PYTHON_OSX_VER`` sets the minimum macOS SDK version targetted. For CPython it may be set to 10.9 or 10.6 (the default). It is currently ignored for PyPy builds.
* ``PLAT`` sets the architecture(s) built, either ``x86_64`` or ``intel`` for 64-bit or 64/32-bit respectively. The default is the same as the Python version selected by ``MB_PYTHON_VER`` and ``MB_PYTHON_OSX_VER``: 64-bit for PyPy or CPython 10.9 builds, and 64/32-bit for CPython 10.6 builds. For normal usage you should not need to set this variable.

Valid combinations of ``MB_PYTHON_VER`` and ``MB_PYTHON_OSX_VER`` for CPython correspond to Python versions available for download at `python.org <https://www.python.org/downloads/mac-osx/>`_.

The ``build_wheel`` function builds the wheel, and the ``install_run``
function installs the wheel and tests it.  Look in ``multibuild/common_utils.sh`` for
default definitions of these functions.  See below for more details.

Manylinux
=========

The build phase is in a Manylinux Docker container, but the test phase is in
a clean Ubuntu 14.04 container.


Build phase
-----------

Specify the Manylinux version to build for with the `MB_ML_VER` environment variable. The default version is `1`.  Versions that are currently valid are:

* `1` (see [PEP 513](https://www.python.org/dev/peps/pep-0513);
* `2010` (see [PEP
  571](https://www.python.org/dev/peps/pep-0571).

At some point `2014` will be a valid version - see [PEP
599](https://www.python.org/dev/peps/pep-0599).

The environment variable specified which Manylinux docker container you are building in.

The `PLAT` environment variable can be one of `x86_64` or `i686`, specifying 64-bit and 32-bit builds, respectively.  The default is 64-bit.

At the time of writing, Manylinux2010 only supports 64-bit
builds, so `MB_ML_VER=2010` and `PLAT=i686` is an invalid
combination, and will generate an error when trying to find the
matching Docker image.

``multibuild/travis_linux_steps.sh`` defines the ``build_wheel`` function,
which starts up the Manylinux1 Docker container to run a wrapper script
``multibuild/docker_build_wrap.sh``, that (within the container) sources the
following bash scripts:

* multibuild/common_utils.sh
* multibuild/manylinux_utils.sh
* env_vars.sh
* multibuild/configure_build.sh
* multibuild/library_builders.sh
* config.sh

See ``docker_build_wrap.sh`` to review the order of script sourcing.

See the definition of ``build_multilinux`` in
``multibuild/travis_linux_steps.sh`` for the environment variables passed from
Travis CI to the Manylinux1 container.

Once in the container, after sourcing the scripts above, the wrapper runs the
real ``build_wheel`` function, which now comes (by default) from
``multibuild/common_utils.sh``.

Test phase
----------

Testing is in an Ubuntu 14.04 Docker container - see
``multibuild/docker_test_wrap.sh``.  ``multibuild/travis_linux_steps.sh``
defines the ``install_run`` function, which starts up the testing Docker
container with a wrapper script ``multibuild/docker_test_wrap.sh``.  The
wrapper script sources the following bash scripts:

* multibuild/common_utils.sh
* config.sh

See ``docker_test_wrap.sh`` for script source order.

See ``install_run`` in ``multibuild/travis_linux_steps.sh`` for the
environment variables passed into the container.

It then (in the container) runs the real ``install_run`` command, which comes
(by default) from ``multibuild/common_utils.sh``.

*********************************
Standard build and test functions
*********************************

The standard build command is ``build_wheel``.  This is a bash function.  By
default the function that is run on macOS, and in the Manylinux container for
the build phase, is defined in ``multibuild/common_utils.sh``.  You can
override the default function in the project ``config.sh`` file (see below).

If you are building a wheel from PyPI, rather than from a source repository,
you can use the ``build_index_wheel`` command, again defined in
``multibuild/common_utils.sh``.

Typically, you can get away with leaving the default ``build_wheel`` /
``build_index_wheel`` functions to do their thing, but you may need to define
a ``pre_build`` function in ``config.sh``.  The default ``build_wheel`` and
``build_index_wheel`` functions will call the ``pre_build`` function, if
defined, before building the wheel, so ``pre_build`` is a good place to build
any required libraries.

The standard test command is the bash function ``install_run``.  The version
run on macOS and in the Linux testing container is also defined in
``multibuild/common_utils.sh``.  Typically, you do not override this function,
but you in that case you will need to define a ``run_tests`` function, to run
your tests, returning a non-zero error code for failure.  The default
``install_run`` implementation calls the ``run_tests`` function, which you
will likely define in ``config.sh``.  See the examples below for examples of
less and more complicated builds, where the complicated builds override more
of the default implementations.

********************
To use these scripts
********************

* Make a repository for building wheels on Travis CI - e.g.
  https://github.com/MacPython/astropy-wheels - or in your case maybe
  ``https://github.com/your-org/your-project-wheels``;

* Add this (here) repository as a submodule::

    git submodule add https://github.com/matthew-brett/multibuild.git

* Add your own project repository as another submodule::

    git submodule add https://github.com/your-org/your-project.git

* Create a ``.travis.yml`` file, something like this::

    env:
        global:
            - REPO_DIR=your-project
            # Commit from your-project that you want to build
            - BUILD_COMMIT=v0.1.0
            # pip dependencies to _build_ your project
            - BUILD_DEPENDS="cython numpy"
            # pip dependencies to _test_ your project.  Include any dependencies
            # that you need, that are also specified in BUILD_DEPENDS, this will be
            # a separate install.
            - TEST_DEPENDS="numpy scipy pytest"
            - UNICODE_WIDTH=32
            - WHEELHOUSE_UPLOADER_USERNAME=travis-worker
            # Following generated with
            # travis encrypt -r your-org/your-project-wheels WHEELHOUSE_UPLOADER_SECRET=<the api key>
            # This is for Rackspace uploads.  Contact Matthew Brett, or the
            # scikit-learn team, for # permission (and the API key) to upload to
            # the Rackspace account used here, or use your own account.
            - secure:
                "MNKyBWOzu7JAUmC0Y+JhPKfytXxY/ADRmUIMEWZV977FLZPgYctqd+lqel2QIFgdHDO1CIdTSymOOFZckM9ICUXg9Ta+8oBjSvAVWO1ahDcToRM2DLq66fKg+NKimd2OfK7x597h/QmUSl4k8XyvyyXgl5jOiLg/EJxNE2r83IA="

    # You will likely prefer "language: generic" for travis configuration,
    # rather than, say "language: python". Multibuild doesn't use
    # Travis-provided Python but rather installs and uses its own, where the
    # Python version is set from the MB_PYTHON_VERSION variable. You can still
    # specify a language here if you need it for some unrelated logic and you
    # can't use Multibuild-provided Python or other software present on a
    # builder.
    language: generic

    # For CPython macOS builds only, the minimum supported macOS version and
    # architectures of any C extensions in the wheel are set with the variable
    # MB_PYTHON_OSX_VER: 10.9 (64-bit only) or 10.6 (64/32-bit dual arch).
    # All PyPy macOS builds are 64-bit only.

    # Required in Linux to invoke `docker` ourselves
    services: docker

    # Host distribution.  This is the distribution from which we run the build
    # and test containers, via docker.
    dist: xenial

    matrix:
      include:
        - os: linux
          env: MB_PYTHON_VERSION=2.7
        - os: linux
          env:
            - MB_PYTHON_VERSION=2.7
            - UNICODE_WIDTH=16
        - os: linux
          env:
            - MB_PYTHON_VERSION=2.7
            - PLAT=i686
        - os: linux
          env:
            - MB_PYTHON_VERSION=2.7
            - PLAT=i686
            - UNICODE_WIDTH=16
        - os: linux
          env:
            - MB_PYTHON_VERSION=3.5
        - os: linux
          env:
            - MB_PYTHON_VERSION=3.5
            - PLAT=i686
        - os: linux
          env:
            - MB_PYTHON_VERSION=3.6
        - os: linux
          env:
            - MB_PYTHON_VERSION=3.6
            - PLAT=i686
        - os: osx
          env:
            - MB_PYTHON_VERSION=2.7
        - os: osx
          env:
            - MB_PYTHON_VERSION=2.7
            - MB_PYTHON_OSX_VER=10.9
        - os: osx
          env:
            - MB_PYTHON_VERSION=3.5
        - os: osx
          env:
            - MB_PYTHON_VERSION=3.6
        - os: osx
          env:
            - MB_PYTHON_VERSION=3.7
            - MB_PYTHON_OSX_VER=10.9
        - os: osx
          language: generic
          env:
            - MB_PYTHON_VERSION=pypy-5.7

    before_install:
        - source multibuild/common_utils.sh
        - source multibuild/travis_steps.sh
        - before_install

    install:
        # Maybe get and clean and patch source
        - clean_code $REPO_DIR $BUILD_COMMIT
        - build_wheel $REPO_DIR $PLAT

    script:
        - install_run $PLAT

    after_success:
        # Upload wheels to Rackspace container
        - pip install wheelhouse-uploader
        # This uploads the wheels to a Rackspace container owned by the
        # scikit-learn team, available at http://wheels.scipy.org.  See above
        # for information on using this account or choosing another.
        - python -m wheelhouse_uploader upload --local-folder
            ${TRAVIS_BUILD_DIR}/wheelhouse/
            --no-update-index
            wheels

  The example above is for a project building from a Git submodule.  If you
  aren't building from a submodule, but want to use ``pip`` to build from a
  source archive on https://pypi.org or similar, replace the first few lines
  of the ``.travis.yml`` file with something like::

    env:
        global:
            # Instead of REPO_DIR, BUILD_COMMIT
            - PROJECT_SPEC="tornado==4.1.1"

  then your ``install`` section could look something like this::

    install:
        - build_index_wheel $PROJECT_SPEC


* Next create a ``config.sh`` for your project, that fills in any steps you
  need to do before building the wheel (such as building required libraries).
  You also need this file to specify how to run your tests::

    # Define custom utilities
    # Test for macOS with [ -n "$IS_OSX" ]

    function pre_build {
        # Any stuff that you need to do before you start building the wheels
        # Runs in the root directory of this repository.
        :
    }

    function run_tests {
        # Runs tests on installed distribution from an empty directory
        python --version
        python -c 'import sys; import yourpackage; sys.exit(yourpackage.test())'
    }

  Optionally you can specify a different location for ``config.sh`` file with
  the ``$CONFIG_PATH`` environment variable.

* Optionally, create an ``env_vars.sh`` file to override the defaults for any
  environment variables used by
  ``configure_build.sh``/``library_builders.sh``. In Linux, the environment
  variables used for the build cannot be set in the ``.travis.yml`` file,
  because the build processing runs in a Docker container, so the only
  environment variables that reach the container are those passed in via the
  ``docker run`` command, or those set in ``env_vars.sh``.

  As for the ``config.sh`` file, you can specify a different location for the
  file by setting the ``$ENV_VARS_PATH`` environment variable.  The path in
  ``$ENV_VARS_PATH`` is relative to the repository root directory.  For
  example, if your repository had a subdirectory ``scripts`` with a file
  ``my_env_vars.sh``, you should set ``ENV_VARS_PATH=scripts/my_env_vars.sh``.

* Make sure your project is set up to build on Travis CI, and you should now
  be ready (to begin the long slow debugging process, probably).

* For the Windows wheels, create an ``appveyor.yml`` file, something like:

  - https://github.com/MacPython/numpy-wheels/blob/master/.appveyor.yml
  - https://github.com/MacPython/astropy-wheels/blob/master/appveyor.yml
  - https://github.com/MacPython/nipy-wheels/blob/master/appveyor.yml
  - https://github.com/MacPython/pytables-wheels/blob/master/appveyor.yml

  Note the Windows test customizations etc are inside ``appveyor.yml``,
  and that ``config.sh`` and ``env_vars.sh`` are only for the
  Linux/Mac builds on Travis CI.

* Make sure your project is set up to build on AppVeyor, and you should now
  be ready (for what could be another round of slow debugging).

If your project depends on NumPy, you will want to build against the earliest
NumPy that your project supports - see `forward, backward NumPy compatibility
<https://stackoverflow.com/questions/17709641/valueerror-numpy-dtype-has-the-wrong-size-try-recompiling/18369312#18369312>`_.
See the `astropy-wheels Travis file
<https://github.com/MacPython/astropy-wheels/blob/master/.travis.yml>`_ for an
example specifying NumPy build and test dependencies.

Here are some simple example projects:

* https://github.com/MacPython/astropy-wheels
* https://github.com/scikit-image/scikit-image-wheels
* https://github.com/MacPython/nipy-wheels
* https://github.com/MacPython/dipy-wheels

Less simple projects where there are some serious build dependencies, and / or
macOS / Linux differences:

* https://github.com/MacPython/matplotlib-wheels
* https://github.com/python-pillow/Pillow-wheels
* https://github.com/MacPython/h5py-wheels

**********************
Multibuild development
**********************

The main multibuild repository is always at
https://github.com/matthew-brett/multibuild

We try to keep the ``master`` branch stable and do testing and development
in the ``devel`` branch.  From time to time we merge ``devel`` into ``master``.

In practice, you can check out the newest commit from ``devel`` that works
for you, then stay at it until you need newer features.
