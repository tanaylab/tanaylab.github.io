TGUtils - Utilities for Tanay Group lab code
============================================

This package contains common utilities used by the Tanay Group Python lab code (for example,
``metacell``). These utilities are generally useful and not associated with any specific project.

Phantom Types
-------------

The vanilla ``np.ndarray``, ``pd.Series`` and ``pd.DataFrame`` types are very generic. They are the
same regardless of the element data type used, and in the case of ``np.ndarray``, the number of
dimensions (array vs. matrix).

To understand the code, it is helpful to keep track of a more detailed data type - whether the
variable is an array or a matrix, and what the element data type is. To facilitate this, ``tgutils``
provides what is known as "Phantom Types". These types can be used in ``mypy`` type declarations,
even though the actual data type of the variables remains the vanilla Numpy/Pandas data type.

See :py:mod:`tgutils.numpy` and :py:mod:`tgutils.pandas` for the list of provided phantom types. To
use them, instead of the vanilla:

.. code-block:: python

    import numpy as np
    import pandas as pd

Write the modified:

.. code-block:: python

    import tgutils.numpy as np
    import tgutils.pandas as pd

This this will provide access to the vanilla symbols using the ``np.`` and/or ``pd.`` prefixes, and
will also provide access to the enhanced functionality described below.

For example, instead of writing:

.. code-block:: python

    def compute_some_integers() -> np.ndarray:
        ...

    def compute_some_floats() -> np.ndarray:
        ...

    def compute_stuff(foo: np.ndarray, bar: np.ndarray) -> ...:
        ...

    def workflow() -> ...:
        foo = compute_some_integers()
        bar = compute_some_floats()
        compute_stuff(foo, bar)

You should write:

.. code-block:: python

    def compute_some_integers() -> np.ArrayInt32:
        ...

    def compute_some_floats() -> np.MatrixFloat64:
        ...

    def compute_stuff(foo: np.ArrayInt32, bar: np.MatrixFloat64) -> ...:
        ...

    def workflow() -> ...:
        foo = compute_some_integers()
        bar = compute_some_floats()
        compute_stuff(foo, bar)

This will allow the reader to understand the exact data types involved. Even better, it will allow
``mypy`` to verify that you actually pass the correct data type to each function invocation.
For example, if you by mistake write ``compute_stuff(bar, foo)`` then ``mypy`` will complain that
the data types do not match - even though, under the covers, both ``foo`` and ``bar`` have exactly
the same data type at run-time: ``np.ndarray``.

To further help with ``mypy`` type checking, the ``tgutils`` package includes a ``stubs`` directory
containing very partial quick-and-dirty type stubs for ``numpy`` and ``pandas`` (ideally, some brave
soul(s) would tackle the very difficult issue of providing proper stubs for these libraries,
allowing for the removal of the ``tgutils`` stubs). Importing :py:func:`tgutils.setup_mypy` module
set ``MYPYPATH`` to this stubs directory, which is also a hack (see the ``metacell`` package for an
example of using this in your ``setup.py`` file).

Type Operations
...............

Control over the data types is also important when performing computations. It affects performance,
memory consumption and even the semantics of some operations. For example, integer elements can
never be ``NaN`` while floating point elements can, boolean elements have their own logic, and
string elements are different from numeric elements.

To help with this, ``tgutils`` provides two functions, ``am`` and ``be``. Both these functions
return the requested data type, but ``am`` is just an assertion while ``be`` is a cast operation.
That is, writing ``ArrayInt32.am(foo)`` will return ``foo`` as an ``ArrayInt32``, or will raise an
error if ``foo`` is not an array of ``int32``; while writing ``ArrayInt32.be(foo)`` will always
return an ``ArrayInt32``, which is either ``foo`` if it is an array of ``int32``, or a copy of
``foo`` whose elements are the conversion of the elements of ``foo`` to ``int32``.

De/serialization
................

The phantom types also provide read and write operations for efficiently storing data on the disk.
That is, writing ``ArrayInt32.read(path)`` will read an array of ``int32`` elements from the
specified path, and ``ArrayInt32.write(foo, path)`` will write an array of ``int32`` elements
into the specified path.

DynaMake
--------

Import ``tgutils.make`` instead of ``dynamake.make``. This will achieve the following:

Using Qsub
..........

The :py:mod:`tgutils.tg_qsub` script deals with submitting jobs to run on the SunGrid cluster in the
Tanay Group lab.

A :py:func:`tgutils.make.tg_require` function allows for collecting context for optimizing the slot
allocation of ``tg_qsub`` for maximizing the cluster utilization and minimizing wait times. This has
no effect unless the collected context values are explicitly used in the ``run_prefix`` and/or
``run_suffix`` action wrapper of some step.

This is a convoluted and sub-optimal mechanism but has significant performance benefits in the
specific environment it was designed for.

Applications
------------

Import ``tgutils.application`` instead of ``dynamake.application``. This will achieve the following:

Resources
.........

By default, the Python process is restricted in the number of simultaneous open files. This
is raised by ``tgutils`` to the maximum allowed by the operating system.

Numpy Errors
............

By default, ``numpy`` ignores several kinds of numeric errors. This is modified by ``tgutils``
to raise an appropriate exception. This increases the robustness of the results.

Numpy Random Number Generation
..............................

By default, ``dynamake`` only handles the Python random number generator. This is extended by
``tgutils`` so that the ``numpy`` random number generator is seeded with the same seeds as the
Python random number generator, even in parallel calls. This seeding ensures results are replicable
(when using the same non-zero seed).

Logging
.......

The default Python logging that prints to ``stderr`` works well for a single application. However,
when running multiple applications in parallel, log messages may get interleaved resulting in
garbled output.

This is solved by ``tgutils`` using the :py:func:`tgutils.application.tg_qsub_logger`, which wraps
the default logger with a :py:class:`tgutils.application.FileLockLoggerAdapter`. This uses a file
lock operation around each emitted log message to ensure it is atomic. The lock file is chosen
to be compatible with the one used by the ``tgutils.tg_qsub`` script, so that log messages
from this script will also be protected.

Parallel
........

When running a large number of very small tasks, it possible to let ``multiprocessing.Pool`` run
each task on the much smaller number of available threads. However, this is less efficient. An
alternative is to use :py:func:`tgutils.application.indexed_range` which will partition the large
range of task indices into equal-sized sub-ranges, one per process. Reporting progress can be
done using the :py:class:`tgutils.application.ParallelCounter` class.

Other Utilities
---------------

Tests
.....

The provided :py:mod:`tgutils.tests` module provides :py:class:`TestWithReset` which properly
resets all the global state for each test, and :py:class:`TestWithFiles` which also creates
a fresh temporary directory for each test. You can create new files using
:py:func:`tgutils.tests.write_file` and verify file contents using
:py:meth:`tgutils.tests.TestWithFiles.expect_file`.

Caching
.......

You can use the :py:class:`tgutils.cache.Cache` class for a lightweight generic cache mechanism.
It uses weak references to hold onto expensive-to-compute data.

YAML
....

You can use :py:func:`tgutils.load_yaml.load_dictionary` for a lightweight verification of loaded
YAML data.
