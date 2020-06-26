pipdeptree
==========

.. image:: https://travis-ci.org/naiquevin/pipdeptree.svg?branch=master
   :target: https://travis-ci.org/naiquevin/pipdeptree


``pipdeptree`` is a command line utility for displaying the installed
python packages in form of a dependency tree. It works for packages
installed globally on a machine as well as in a virtualenv. Since
``pip freeze`` shows all dependencies as a flat list, finding out
which are the top level packages and which packages do they depend on
requires some effort. It's also tedious to resolve conflicting
dependencies that could get installed because ``pip`` doesn't have
true dependency resolution yet [1]_. ``pipdeptree`` can help here by
identifying conflicting dependencies installed in the environment.

To some extent, ``pipdeptree`` is inspired by the ``lein deps :tree``
command of `Leiningen <http://leiningen.org/>`_.


Installation
------------

.. code-block:: bash

    $ pip install pipdeptree

This will install the latest stable version which is ``1.0.0``. This
version works well for the basic use case but has some limitations.

An improved version ``2.0.0b1`` has been released as well. But as it's
a beta version, pip will not find it by default. To install the latest
beta version specify the ``--pre`` flag.

.. code-block:: bash

    $ sudo pip install --pre pipdeptree

The current stable version is tested with ``2.7``, ``3.4``, ``3.5``
and ``3.6``.

The ``v2beta`` branch has been tested with Python ``3.4``, ``3.5``, ``3.6``, ``3.7``,
``3.8`` as well as ``2.7``.

Python ``2.6`` is way past it's end of life but if you ever find
yourself stuck on a legacy environment, version ``0.9.0`` *might*
work.


Usage and examples
------------------

To give you a brief idea, here is the output of ``pipdeptree``
compared with ``pip freeze``:

.. code-block:: bash

    $ pip freeze
    Flask==0.10.1
    itsdangerous==0.24
    Jinja2==2.11.2
    -e git+git@github.com:naiquevin/lookupy.git@cdbe30c160e1c29802df75e145ea4ad903c05386#egg=Lookupy
    MarkupSafe==0.22
    pipdeptree @ file:///private/tmp/pipdeptree-2.0.0b1-py3-none-any.whl
    Werkzeug==0.11.2

And now see what ``pipdeptree`` outputs,

.. code-block:: bash

    $ pipdeptree
    Warning!!! Possibly conflicting dependencies found:
    * Jinja2==2.11.2
     - MarkupSafe [required: >=0.23, installed: 0.22]
    ------------------------------------------------------------------------
    Flask==0.10.1
      - itsdangerous [required: >=0.21, installed: 0.24]
      - Jinja2 [required: >=2.4, installed: 2.11.2]
        - MarkupSafe [required: >=0.23, installed: 0.22]
      - Werkzeug [required: >=0.7, installed: 0.11.2]
    Lookupy==0.1
    pipdeptree==2.0.0b1
      - pip [required: >=6.0.0, installed: 20.1.1]
    setuptools==47.1.1
    wheel==0.34.2


Is it possible to find out why a particular package is installed?
-----------------------------------------------------------------

`New in ver. 0.5.0`

Yes, there's a ``--reverse`` (or simply ``-r``) flag for this. To find
out which packages depend on a particular package(s), it can be
combined with ``--packages`` option as follows:

.. code-block:: bash

    $ pipdeptree --reverse --packages itsdangerous,MarkupSafe
    Warning!!! Possibly conflicting dependencies found:
    * Jinja2==2.11.2
     - MarkupSafe [required: >=0.23, installed: 0.22]
    ------------------------------------------------------------------------
    itsdangerous==0.24
      - Flask==0.10.1 [requires: itsdangerous>=0.21]
    MarkupSafe==0.22
      - Jinja2==2.11.2 [requires: MarkupSafe>=0.23]
        - Flask==0.10.1 [requires: Jinja2>=2.4]


What's with the warning about conflicting dependencies?
-------------------------------------------------------

As seen in the above output, ``pipdeptree`` by default warns about
possible conflicting dependencies. Any package that's specified as a
dependency of multiple packages with different versions is considered
as a conflicting dependency. Conflicting dependencies are possible due
to pip's `lack of true dependency resolution
<https://github.com/pypa/pip/issues/988>`_ [1]_. The warning is
printed to stderr instead of stdout and it can be completely silenced
by specifying the ``-w silence`` or ``--warn silence`` option. On the
other hand, it can be made mode strict with ``--warn fail``, in which
case the command will not only print the warnings to stderr but also
exit with a non-zero status code. This is useful if you want to fit
this tool into your CI pipeline.

**Note**: The ``--warn`` option is added in version ``0.6.0``. If you
are using an older version, use ``--nowarn`` flag to silence the
warnings.


Warnings about circular dependencies
------------------------------------

In case any of the packages have circular dependencies (eg. package A
depends on package B and package B depends on package A), then
``pipdeptree`` will print warnings about that as well.

.. code-block:: bash

    $ pipdeptree --exclude pip,pipdeptree,setuptools,wheel
    Warning!!! Cyclic dependencies found:
    - CircularDependencyA => CircularDependencyB => CircularDependencyA
    - CircularDependencyB => CircularDependencyA => CircularDependencyB
    ------------------------------------------------------------------------
    wsgiref==0.1.2
    argparse==1.2.1

Similar to the warnings about conflicting dependencies, these too are
printed to stderr and can be controlled using the ``--warn`` option.

In the above example, you can also see ``--exclude`` option which is
the opposite of ``--packages`` ie. these packages will be excluded
from the output.


Using pipdeptree to write requirements.txt file
-----------------------------------------------

If you wish to track only top level packages in your
``requirements.txt`` file, it's possible by grep-ing only the
top-level lines from the output,

.. code-block:: bash

    $ pipdeptree --warn silence | grep -E '^\w+'
    Flask==0.10.1
    gnureadline==8.0.0
    Lookupy==0.1
    pipdeptree==2.0.0b1
    setuptools==47.1.1
    wheel==0.34.2

There is a problem here though - The output doesn't mention anything
about ``Lookupy`` being installed as an *editable* package (refer to
the output of ``pip freeze`` above) and information about its source
is lost. To fix this, ``pipdeptree`` must be run with a ``-f`` or
``--freeze`` flag.

.. code-block:: bash

    $ pipdeptree -f --warn silence | grep -E '^[a-zA-Z0-9\-]+'
    Flask==0.10.1
    gnureadline==8.0.0
    -e git+git@github.com:naiquevin/lookupy.git@cdbe30c160e1c29802df75e145ea4ad903c05386#egg=Lookupy
    pipdeptree @ file:///private/tmp/pipdeptree-2.0.0b1-py3-none-any.whl
    setuptools==47.1.1
    wheel==0.34.2

    $ pipdeptree -f --warn silence | grep -E '^[a-zA-Z0-9\-]+' > requirements.txt

The freeze flag will not prefix child dependencies with hyphens, so
you could dump the entire output of ``pipdeptree -f`` to the
requirements.txt file thus making it human-friendly (due to
indentations) as well as pip-friendly.

.. code-block:: bash

    $ pipdeptree -f | tee locked-requirements.txt
    Flask==0.10.1
      itsdangerous==0.24
      Jinja2==2.11.2
        MarkupSafe==0.23
      Werkzeug==0.11.2
    gnureadline==8.0.0
    -e git+git@github.com:naiquevin/lookupy.git@cdbe30c160e1c29802df75e145ea4ad903c05386#egg=Lookupy
    pipdeptree @ file:///private/tmp/pipdeptree-2.0.0b1-py3-none-any.whl
      pip==20.1.1
    setuptools==47.1.1
    wheel==0.34.2

On confirming that there are no conflicting dependencies, you can even
treat this as a "lock file" where all packages, including the
transient dependencies will be pinned to their currently installed
versions. Note that the ``locked-requirements.txt`` file could end up
with duplicate entries. Although ``pip install`` wouldn't complain
about that, you can avoid duplicate lines (at the cost of losing
indentation) as follows,

.. code-block:: bash

    $ pipdeptree -f | sed 's/ //g' | sort -u > locked-requirements.txt


Using pipdeptree with external tools
------------------------------------

`New in ver. 0.5.0`

It's also possible to have ``pipdeptree`` output json representation
of the dependency tree so that it may be used as input to other
external tools.

.. code-block:: bash

    $ pipdeptree --json

Note that ``--json`` will output a flat list of all packages with
their immediate dependencies. This is not very useful in itself. To
obtain nested json, use ``--json-tree``

`New in ver. 0.11.0`

.. code-block:: bash

    $ pipdeptree --json-tree


Visualizing the dependency graph
--------------------------------

.. image:: https://raw.githubusercontent.com/naiquevin/pipdeptree/master/docs/twine-pdt.png

The dependency graph can also be visualized using `GraphViz
<http://www.graphviz.org/>`_:

.. code-block:: bash

    $ pipdeptree --graph-output dot > dependencies.dot
    $ pipdeptree --graph-output pdf > dependencies.pdf
    $ pipdeptree --graph-output png > dependencies.png
    $ pipdeptree --graph-output svg > dependencies.svg

Note that ``graphviz`` is an optional dependency ie. required only if
you want to use ``--graph-output``.

Since version ``2.0.0b1``, ``--package`` and ``--reverse`` flags are
supported for all output formats ie. text, json, json-tree and graph.

In earlier versions, ``--json``, ``--json-tree`` and
``--graph-output`` options override ``--package`` and ``--reverse``.


Usage
-----

.. code-block:: bash

    usage: pipdeptree [-h] [-v] [-f] [-a] [-l] [-u] [-w [{silence,suppress,fail}]]
                      [-r] [-p PACKAGES] [-e PACKAGES] [-j] [--json-tree]
                      [--graph-output OUTPUT_FORMAT]
    
    Dependency tree of the installed python packages
    
    optional arguments:
      -h, --help            show this help message and exit
      -v, --version         show program's version number and exit
      -f, --freeze          Print names so as to write freeze files
      -a, --all             list all deps at top level
      -l, --local-only      If in a virtualenv that has global access do not show
                            globally installed packages
      -u, --user-only       Only show installations in the user site dir
      -w [{silence,suppress,fail}], --warn [{silence,suppress,fail}]
                            Warning control. "suppress" will show warnings but
                            return 0 whether or not they are present. "silence"
                            will not show warnings at all and always return 0.
                            "fail" will show warnings and return 1 if any are
                            present. The default is "suppress".
      -r, --reverse         Shows the dependency tree in the reverse fashion ie.
                            the sub-dependencies are listed with the list of
                            packages that need them under them.
      -p PACKAGES, --packages PACKAGES
                            Comma separated list of select packages to show in the
                            output. If set, --all will be ignored.
      -e PACKAGES, --exclude PACKAGES
                            Comma separated list of select packages to exclude
                            from the output. If set, --all will be ignored.
      -j, --json            Display dependency tree as json. This will yield "raw"
                            output that may be used by external tools. This option
                            overrides all other options.
      --json-tree           Display dependency tree as json which is nested the
                            same way as the plain text output printed by default.
                            This option overrides all other options (except
                            --json).
      --graph-output OUTPUT_FORMAT
                            Print a dependency graph in the specified output
                            format. Available are all formats supported by
                            GraphViz, e.g.: dot, jpeg, pdf, png, svg

Known issues
------------

1. To work with packages installed inside a virtualenv, ``pipdeptree``
   also needs to be installed in the same virtualenv even if it's
   already installed globally.

2. Due to (1), the output also includes ``pipdeptree`` itself as a
   dependency along with ``pip``, ``setuptools`` and ``wheel`` which
   get installed in the virtualenv by default. To ignore them, use the
   ``--exclude`` option.

3. ``pipdeptree`` relies on the internal API of ``pip``. I fully
   understand that it's a bad idea but it mostly works! On rare
   occasions, it breaks when a new version of ``pip`` is out with
   backward incompatible changes in internal API. So beware if you are
   using this tool in environments in which ``pip`` version is
   unpinned, specially automation or CD/CI pipelines.


Limitations & Alternatives
--------------------------

``pipdeptree`` merely looks at the installed packages in the current
environment using pip, constructs the tree, then outputs it in the
specified format. If you want to generate the dependency tree without
installing the packages, then you need a dependency resolver. You
might want to check alternatives such as `pipgrip
<https://github.com/ddelange/pipgrip>`_ or `poetry
<https://github.com/python-poetry/poetry>`_.

Also, stay tuned for the dependency resolver in upcoming versions of
pip [1]_.


Runing Tests (for contributors)
-------------------------------

There are 2 test suites in this repo:

1. Unit tests that use mock objects. These are configured to run on
   every push to the repo and on every PR thanks to travis.ci

2. End-to-end tests that are run against actual packages installed in
   virtualenvs

Unit tests can be run against all version of python using `tox
<http://tox.readthedocs.org/en/latest/>`_ as follows:

.. code-block:: bash

    $ make test-tox-all

This assumes that you have python versions specified in the
``tox.ini`` file.

If you don't want to install all the versions of python but want to
run tests quickly against ``Python3.6`` only:

.. code-block:: bash

    $ make test

Unit tests are written using ``pytest`` and you can also run the tests
with code coverage as follows,

.. code-block:: bash

    $ make test-cov

On the other hand, end-to-end tests actually create virtualenvs,
install packages and then run tests against them. These tests are more
reliable in the sense that they also test ``pipdeptree`` with the
latest version of ``pip`` and ``setuptools``.

The downside is that when new versions of ``pip`` or ``setuptools``
are released, these need to be updated. At present the process is
manual but I have plans to setup nightly builds for these for faster
feedback.

The end-to-end tests can be run as follows,

.. code-block:: bash

    $ make test-e2e  # starts with a clean virtualenvs

    $ # or

    $ make test-e2e-quick # reuses existing virtualenvs

By default the e2e tests uses python executable ``python3.6``. To use
an alternate version set the environment var ``E2E_PYTHON_EXE``.

.. code-block:: bash

    $ E2E_PYTHON_EXE=python2.7 make test-e2e


Release checklist
-----------------

#. Make sure that tests pass on travis.ci.
[![FOSSA Status](https://app.fossa.com/api/projects/git%2Bgithub.com%2Fsimonpjakobsen%2Fpipdeptree.svg?type=shield)](https://app.fossa.com/projects/git%2Bgithub.com%2Fsimonpjakobsen%2Fpipdeptree?ref=badge_shield)

#. Create a commit with following changes and push it to github
#. Update the `__version__` in the `pipdeptree.py` file.

   #. Add Changelog in `CHANGES.md` file.
   #. Also update `README.md` if required.
#. Create an annotated tag on the above commit and push the tag to
   github
#. Upload new version to PyPI.


License
-------

MIT (See `LICENSE <./LICENSE>`_)

Footnotes
---------

.. [1] Soon we'll have `a dependency resolver in pip itself
       <https://github.com/pypa/pip/issues/6536>`_


## License
[![FOSSA Status](https://app.fossa.com/api/projects/git%2Bgithub.com%2Fsimonpjakobsen%2Fpipdeptree.svg?type=large)](https://app.fossa.com/projects/git%2Bgithub.com%2Fsimonpjakobsen%2Fpipdeptree?ref=badge_large)